---
title: 使用operator在kubernetes上部署spark
tags:
  - kubernetes
  - operator
  - spark
  - helm
categories:
  - 大数据
  - kubernetes
abbrlink: 320cc8d6
date: 2019-05-17 11:47:20
---


## 背景
Spark是大数据领域极为流行的处理引擎，它拥有丰富的套件和活跃的社区生态，其结合pyspark极大地提升了数据工程师理解和分析数据的生产力。

Kubernetes是近几年最火的开源项目之一，在经历2018年的快速发展后，kubernetes已经成为容器编排领域的事实标准。 在充分地支持了无状态服务之后，开源社区开始努力解决复杂有状态服务的容器编排，为此提出了operator的概念，用于解决复杂有状态服务的编排。 kubernetes将会全面支持大数据领域的资源编排和管理。


Spark在2.3.0版本支持了kubernetes作为原生集群runtime的功能，相关的讨论在<a href="https://issues.apache.org/jira/browse/SPARK-18278">SPARK-18278</a>

本文主要基于谷歌云的spark-on-k8s-operator项目，实践在kubernetes集群使用operator部署spark，并运行pyspark demo job。

## 准备

1. kubernetes集群
2. helm and tiller
3. git
4. spark镜像 （谷歌的镜像国内无法访问）

### 版本说明
kubernetes 1.14.0
spark 2.4.0


## 使用helm安装spark operator

1. 下载charts
```shell
# git clone https://github.com/helm/charts.git

```

2. install spark operator
```shell
# helm install --name=spark incubator/sparkoperator --namespace spark-operator --set sparkJobNamespace=spark-jobs --set serviceAccounts.spark.name=spark
```
该命令将会在`spark-operator`命名空间下运行一个operator pod， 并且会监听`spark-jobs`命名空间下提交的spark job。


## 运行pyspark-pi

1. 下载spark-on-k8s-operator
```shell
# git clone https://github.com/GoogleCloudPlatform/spark-on-k8s-operator.git
```

2. 运行pyspark-pi
```
# kubectl apply -f examples/spark-py-pi.yaml
```
注意： 由于设定的spark job的namspace为`spark-jobs`，因此需要将spark-py-pi.yaml由`default`改为`spark-jobs`。


## 后记


### Design


### 交流
在slack上有spark-operator相关的channel，可用于交流及答疑
<a href="https://kubernetes.slack.com/messages/CALBDHMTL">Slack</a>


### 参考
<a href="https://issues.apache.org/jira/browse/SPARK-18278">spark 2.3.0开始支持kubernetes</a>
<a href="https://github.com/GoogleCloudPlatform/spark-on-k8s-operator">spark-on-k8s-operator github repo</a>
<a href="https://github.com/helm/charts/tree/master/incubator/sparkoperator">spark operator chart repo</a>
