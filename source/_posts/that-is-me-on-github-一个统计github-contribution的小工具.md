---
title: 'that_is_me_on_github: 一个统计github contribution的小工具'
tags:
  - that_is_me_on_github
categories:
  - 社区贡献
abbrlink: '8499e970'
date: 2019-06-18 15:33:33
---


## 前言
linus曾说过： “talk is cheap, show me the code”， 一个人的代码能在一定程度上代表他的技术水平，而参与开源项目，既能从中学习并提升自己，又可以在github上留下自己的足迹，供后人凭吊。

而且在知识付费盛行的今天，如何评判一个大佬是否真的在开源社区呼风唤雨，还是徒有虚名，仅仅改了个注释就以contributor自居。统计他的贡献，也能让我们擦亮眼睛，避免盲从被收了智商税。

## that_is_me_on_github
这是一个python实现的cli， 用于统计某个username的github贡献信息，并生成markdown文档。 生成的Markdown中包含该用户的owned repos,  followers, prs 和issues，比较能够全面地分析该用户的贡献。

[项目地址](https://github.com/hustclf/that_is_me_on_github)
[demo](https://github.com/hustclf/that_is_me_on_github/blob/master/demo.md)

<!-- more -->

## 项目背景
我开发这个小工具有以下两个初衷。
1. 我希望在博客上添加一个[社区贡献](https://hustclf.github.io/contributions/)页，用来展示我参与的社区贡献，为博客吸引些人气。

2. 我希望尝试走一个完整开源项目从立项、开发、发布、维护的流程。麻雀虽小、五脏俱全，正好可以通过这个小工具实践下自己的一些想法。

## 项目开发及发布
### 目标
实现一个cli，能够根据提供的github username, 爬取其参与的所有开源项目，并生成markdown。

### 项目管理
- 使用github issues进行issue tracking。
- 使用github project进行版本管理及迭代（类似jira）
- 使用slack stackoverflow email进行使用交流

### 开发框架
- 使用pipenv进行开发
- github提供了api来获取信息，pygithub是对api进行二次封装的python client。
- click是开发python cli的标准框架。通过它可以快速地开发python cli。

### 测试框架
- 使用pytest进行单元测试及集成测试。

### 持续集成
- 使用circleci进行持续集成。

### 代码覆盖
- 使用codecov进行代码覆盖率统计。

### 发布
- pypi包发布：https://pypi.org/project/that-is-me-on-github/
- docker发布：https://hub.docker.com/r/hustclf/that_is_me_on_github

## 后记
还在活跃开发中。
