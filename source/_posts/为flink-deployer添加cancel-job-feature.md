---
title: 为flink-deployer添加cancel job feature
tags:
  - flink
  - flink-deployer
abbrlink: ebbd7fcc
categories:
  - 社区贡献
date: 2019-04-07 22:50:39
---


# 为flink-deployer添加cancel job feature
kafkacat是一款go实现的用于部署flink jobs的cli, 它内部集成了flink rest api, 支持对flink job的部署，更新等。

目前支持的功能有：

- Listing jobs
- Deploying a new job
- Updating an existing job
- Querying Flink queryable state

我也在使用flink-deployer，用于集成ci/cd pipeline中，支持自动部署flink job到kubernetes集群。 但日常开发中，有时候会用到取消job的功能，但目前flink-deployer还不支持，但维护者[支持其他contributor贡献该特性](https://github.com/ing-bank/flink-deployer/issues/26)，因此我
打算贡献该特性，顺便实战入门下golang。

<!-- more -->

## 调研flink rest api
flink rest api中提供了terminate flink job的api（ [链接]
(https://ci.apache.org/projects/flink/flink-docs-stable/monitoring/rest_api.html#jobs-jobid-1) ）, 并且支持两种取消job的模式：

- cancel

- stop

两种模式中，相比cancel， stop更加柔和些，相关比较(<a href="https://ci.apache.org/projects/flink/flink-docs-stable/ops/cli.html">参考</a>)。


## pr地址
[pr](https://github.com/ing-bank/flink-deployer/pull/37)
