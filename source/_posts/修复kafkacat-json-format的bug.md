---
title: 修复kafkacat json format的bug
tags: 
  - kafkacat
abbrlink: 66a1e560
categories:
  - 社区贡献
date: 2019-01-19 11:16:29
---


# 修复kafkacat json format的bug
kafkacat是一款c语言实现的kafka client cli, 不同于kafka官方包里的cli，使用比较繁琐，并且依赖jvm才能运行， kafkacat是一个小巧的cmd， 依赖一个基础c library: [librdkafka](https://github.com/edenhill/librdkafka),能够很方便地对kafka进行一些基本操作，如list、producer、consumer等。

<!-- more -->

kafkacat的作者是[edenhill](https://github.com/edenhill)， 同时也是librdkafka的作者，现在应该是kafka的母公司，confluent的一名工程师

## 如何发现该bug？
kafka在v0.10之后的版本，每条数据除了key 和message之外，还会携带一条timestamp， 代表该条日志的时间.( [参考](https://stackoverflow.com/questions/39514167/retrieve-timestamp-based-data-from-kafka) )

我在使用kafka streams开发应用时，发现我的输出的topic总是会被kafka清理掉， 我产生的日志使用event timestamp作为执行的timestamp，数据是1月以前的，因此我怀疑输出的topic的日志，timestamp使用的是event timestamp，因此被kafka认为是过期日志，清理掉了。

kafkacat是支持消费时指定固定的格式的，通过指定 `-J` 参数，可以打印json格式的message,其中会携带该message的所有源数据，包括： partition, key, value, timestamp等。

因此我使用kafkacat进行消费时，发现了该bug， 按照json格式消费时，展示timestamp字段不正确，因此我提了相关issue， 并通过阅读源码，修改了该bug。


## issue地址
[issue](https://github.com/edenhill/kafkacat/issues/167)

## pr地址
[pr](https://github.com/edenhill/kafkacat/pull/166)
