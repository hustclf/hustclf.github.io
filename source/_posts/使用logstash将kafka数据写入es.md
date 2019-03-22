---
title: 使用logstash将kafka数据写入elasticsearch
tags:
  - logstash
  - kafka
  - elasticsearch
categories:
  - 大数据
abbrlink: aaa3e6e1
date: 2019-02-15 01:39:01
---

----------
## 目标
将kafka中的日志写入elasticsearch，并要求支持以下功能：

1. 根据日志时间建立index，便于后续对index的管理
2. 使用log_id作为document_id，保持写入es的幂等性
3. 对建立的多个index, 设置相同的alias （业务方使用alias进行query，对具体index不感知）
4. 由于日志时间为epoch_seconds，无法被es自动引用为时间，需要进行字段mapping

## 软件版本及日志格式
#### 软件版本
kafka: 2.1.0

elasticsearch: 6.6.0

kakfa-connect-elasticsearch: v5.1.1

logstash: 6.6.0

<!-- more -->

#### 日志格式
日志是以json格式存储在topic中，topic_name为logs

```
{
	"log_id": 'log_xxxxxxx',
	"timestamp": 1550167937,
	"user_id": "xxxx"
}
```

## 计算方案选型
### kafka-connect-elasticsearch
kafka-connect-elasticseach是confluent（kafka团队的母公司）提供的kafka connect的一个plugin， 用于将kakfa的数据导入es

相关地址及参考链接如下：

<a href="https://github.com/confluentinc/kafka-connect-elasticsearch">github repo</a>

<a href="https://docs.confluent.io/current/connect/kafka-connect-elasticsearch/index.html">document</a>

### logstash
logstash 提供kafka input plugin 和 elasticsearch output plugin, 也支持将kafka数据导入es

<a href="https://www.elastic.co/guide/en/logstash/current/index.html">document</a>

### 对比
kafka-connect-elasticsearch是基于kafka connect api实现的plugin，功能强在kafka端，目前该插件对elasticsearch的支持有限，而且迭代也较慢，而且目前部署对devops也不友好，不满足我们需要的功能

logstash是elasticsearch较早推出的收集及分发数据的组件，功能强在elasticsearch端，功能强大且较成熟，并且支持所需的所有功能。

因此，决定使用logstash来满足我们的需求

## 实现
### 任务分拆
1. 支持按照日志创建index名称： 在output中指定index的命名方式
2. 支持使用log_id作为es的document_id
3. 支持设置别名
4. 支持将日志时间戳mapping 为document的时间戳

要实现上述目标，只需要提供两个配置文件：

1 pipeline.yaml: 用于定义input fliter output等信息

2 index-template.json: 用于定义index模板，将index名与alias动态关联，并且创建mapping

### pipeline.yaml
```
# configuration for transformming events from kafka to elasicsearch
# Kafka -> Logstash -> Elasticsearch pipeline.
input {
  kafka {
    bootstrap_servers => “localhost:9092"
    topics => ["logs"]
    group_id => "logstash-logs"
    auto_offset_reset => "earliest"
    codec => json
  }
}

filter {
  date {
    match => ["timestamp", "UNIX"]
    target => "@timestamp"
    timezone => "Asia/Shanghai"
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "logs-%{+YYYY.MM}"
    document_type => "log"
    document_id => "%{log_id}"
    template => "conf/index-template.json"
    template_name => "logs"
    template_overwrite => true
  }
}
```
解释：

- input: 配置了kafka相关的信息
- fliter: 由于index命名时所用的时间为logstash的元数据 `@timestamp`, 因此需要在`fliter`中使用`date` 插件，用日志时间覆写原始的`@timestamp` （原始的`@timestamp`是日志摄入时间，而非日志时间）
- output: 配置了elasticsearch的信息，以及使用log_id作为document_id， index的命名、及index使用的模板路径`config/index-template.json`

### index-template.json
```
{
    "index_patterns" : ["logs-*"],
    "settings": {
         "number_of_shards": 1,
     },
     "mappings": {
         "log": {
             "properties": {
                 "timestamp": {
                     "type": "date",
                     "format": "epoch_second"
                 },
             },
             "dynamic_date_formats": [
                 "date_time",
                 "date_time_no_millis"
             ]
         }
     },
    "aliases" : {
        "logs" : {}
    }
}
```

解释：

- index_patterns: 确定了该index模板作用的index范围
- mappings: 配置使用日志的`timestamp`字段，作为es中document的日期字段
- alias: 为新增的index，创建统一的别名


## 运行
下载logstash（current version = 6.6.0）, 并将上述两个文件置于`path/config`目录下, 然后运行如下命令:

```
$ bin/logstash -f config/pipeline.yaml
```

## 部署
logstash已经提供了官方镜像， 因此可以使用docker-compose或者kubernetes来部署logstash，此处不再赘述
