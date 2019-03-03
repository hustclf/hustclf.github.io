---
title: 一种使用redis实现频率限制的错误方式及解释
date: 2017-09-18 22:38:36
tags: redis 频率限制 高并发
---
# 一种使用redis实现频率限制的错误方式及解释

redis作为如今最流行的缓存软件广泛应用于互联网业务的方方面面，而频率限制又是很常见的业务场景，如常见的投票、抢购、秒杀等场景。redis由于全部基于内存操作，在读写性能上表现相当给力，因此很多业务中都使用redis来实现频率限制。

网上的教程已经汗牛充栋，我这里仅简单地介绍下原理，然后重点介绍下在高并发场景下，简单地使用redis将不能保证操作的原子性，频率限制会形同虚设。

## 一、一种redis实现频率限制的常见方式

<a href="https://redis.io/commands/INCR#pattern-rate-limiter-1">参考redis官网</a>

常见的有如下的方式来实现，以php为例。

```php
<?php
    // limit = 10 expire = 1， 代表限制1秒内最多10次访问
    function rateLimit($key, $limit = 10, $expire = 1)
    {
        // 获取redis实例
        $redis = \Redis();
        $redis->connnect('127.0.0.1', 6379);

        // 获取当前的计数器的值。命名为op1(操作1)
        $current = redis->get($key);

        // 如果当前计数器的值存在且计数器已经超过限制，返回失败
        if ($current != null && $current >= $limit) {
            return false;
        } else {
            // 获取计数器的存活时间
            $ttl = $redis->ttl($key);
            
            // 开始一个事务 // 命名为op2（操作2）
            $redis->multi(\Redis::MULTI);
            // 将计数器原子加1，若计数器不存在，则创建计数器
            $redis->incr($key);
            
            // 若存活时间小于0，代表计数器之前不存在，是刚刚创建的，则设置一个存活时间
            if ($ttl < 0) {
                $redis->expire($key, $expire);
            }
            
            // 提交事务
            $redis->exec();
            return true;
        }
    }
```

大家看代码加上注释，相信已经能明白实现的原理。这种实现方法，在大多数场景下也运行的非常好，但在高并发场景下就会有很严重的问题。
<!-- more -->
## 二、高并发时存在的坑
单从代码逻辑上来看，是没有丝毫问题的。但必须要注意的是，php（或其他语言）调用redis服务时都是典型的client-server模型，首先要进行3次握手，才能建立连接，之后每一次通信，都是发送一个tcp报文，再封装成ip报文，ip报文在经过n个路由器或交换机，最终才抵达redis服务器（假设php程序和redis不在同一机器上）。其中的网络开销是要远远超过任何一个复杂php操作的执行时间的。

而redis奉行的是极简主义，其单进程单线程模型，保证了每个redis命令单独执行的原子性（如果是事务的话，会将事务里多个命令打包执行，也具有原子性）。但redis无法保证多个命令执行的原子性。 

而从上面代码可知，实现频率限制，通过两步操作来实现的（忽略获取存活时间的一步）。

1. op1： 读取当前计数器的值

2. op2: 若超出频率限制，则返回失败，否则通过事务实现原子性加1

从上面的分析可知，redis是无法保证op1和op2原子性执行的，最终的结果会跟我们设想的大相径庭。


### 2.1 举例分析
#### 目标
我想实现一个1秒钟内限制最多访问10次。
##### 准备
(1) 我们将1秒分成1000份，每一份为1ms， 分别用[t1, t2, t3, t4, ... t1000]来表示

(2) 在t1时刻，来了1000个并发请求，分别用[req1, req2, req3, ... req1000]来表示

(3) 假如php-fpm和redis不在同一机房，请求一次redis花费的时间是100ms

(4) 读取计数器的值和递增计数器的值的操作，分别为[op1, op2]

(5) redis是单个队列顺序执行命令，我使用数组来表示

```
    queue = [
        {
            'req' : req1,  // req代表是哪个请求
            'op'  : op1    // op 代表该请求执行的哪个操作
        },
        {
            'req' : xxx,
            'op'  : xxx,
        }
    ]
```


#### t1时刻
在t1时刻有1000个并发请求过来，最终会有1000个op1操作，会加入到redis执行的队列中，其结果为：
```
    queue = [
        {
            'req': req1,
            'op' : op1,
        },
        {
            'req': req2,
            'op' : op1,
        },
        {
            'req': req3,
            'op' : op1,
        },
        ...
        {
            'req': req1000,
            'op' : op1,
        },
    ]
```

redis执行完这些命令，并且会在t100时刻将结果返回。

#### t100时刻
 此时所有的请求都会拿到结果，每个请求都会知道此时计数器的值为1，它们都会来执行op2，并且返回true。

#### t100-t1000时间段
 每个请求都会执行op2，并且将op2的命令发给redis处理，此时redis执行的队列如下。

```
    queue = [
        {
            'req': req1,
            'op' : op2,
        },
        {
            'req': req2,
            'op' : op2,
        },
        {
            'req': req3,
            'op' : op2,
        },
        ...
        {
            'req': req1000,
            'op' : op2,
        },
    ]


// 注：由于网络环境的变化，req到达redis的顺序并不一定是req1,req2....req1000, 此处为举例，忽略细节
```

#### t1000时刻
此时正好过了一秒，可以看下我们最终的结果。

(1) 1000请求都通过了频率限制，将有权限执行后续操作

(2) 计数器累加到了1000


哈哈，我们为了提供系统安全而设计的频率限制，在高并发场景下形同虚设，惊不惊喜！

### 2.2 原因
(1) 高并发场景下，频率限制的两步操作在redis执行时是非原子性的

(2) 请求redis带来的时间开销是要远远大于普通代码执行的。

## 三、实验
可以使用swoole多进程来模拟并发请求，执行上面的代码，下面是我使用100个并发程序，运行得到的结果。（为了使效果更直观，php和redis位于不同的机房，请求的时间开销为130ms左右）

### 3.1 目标 
频率限制10分钟内最多操作10次
### 基本环境
(1) swoole模拟100个并发访问
(2) php和redis跨机房部署、延迟130ms

### 3.2 代码
```php
<?php
    $workNum = 100;

    $stdout = '';
    for ($i = 0; $i < $workNum; $i++) {
        $process       = new swoole_process('rateLimit', $stdout);
        $pid           = $process->start();
        $workers[$pid] = $process;
    }

    function rateLimit(swoole_process $worker)
    {
        $redis = new \Redis();
        $redis->connect('xxx.xxx.xxx.xxx', xxx);
        $redis->auth('i dont know');

        $limit  = 10;
        $expire = 600;
        $key    = 'testbyclf';

        $current = $redis->get($key);
        if ($current != null && $current >= $limit) {
                echo "permission rejected\n";
        } else {
                $ttl = $redis->ttl($key);

                $redis->multi(\Redis::MULTI);
                $redis->incr($key);

                if ($ttl < 0) {
                        $redis->expire($key, $expire);
                }

                $redis->exec();
                echo "permission passed\n";
        }

        $worker->exit();
    }
```
### 3.3 结果
#### swoole输出
![swoole.png](/images/swoole.png)
#### redis输出
![redis.png](/images/redis.png)
## 四、总结
(1) 并发请求访问共享资源时，要保证原子操作

(2) 要区分对待访问服务和普通代码执行之间，时间开销的巨大差距

## 五、后续
#### 既然在高并发场景时，使用简单的redis命令来实现有诸多的弊病，那是否有其他方案来解决呢？
其实，只需要保证op1和op2操作的原子性，就能解决高并发时的问题。目前的方案有两种：

(1) 使用lua脚本在redis中实现频率限制的功能。redis是能够保证lua脚本执行的原子性的，并且还将多次网络开销变成一次网络开销，速度上会大大提高

(2) redis 4.0以上版本支持了模块功能，目前官方已经有成熟的频率限制的第三方模块，可以通过<a href="https://redis.io/modules">redis官网-module</a>了解详情。
