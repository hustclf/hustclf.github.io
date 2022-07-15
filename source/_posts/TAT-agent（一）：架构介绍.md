---
title: TAT agent（一）：架构介绍
tags:
  - TAT
  - 腾讯云自动化助手
categories:
  - 云原生
abbrlink: 5bd51536
date: 2022-01-14 18:51:40
---
## 简介
自动化助手（TencentCloud Automation Tools，TAT）是云服务器 CVM 和轻量应用服务器 Lighthouse 的原生运维部署工具。自动化助手提供了一种自动化的远程操作方式，无需登录及密码，即可批量执行命令（Shell、PowerShell 及 Python 等），完成运行自动化运维脚本、轮询进程、安装/卸载软件、更新应用及安装补丁等任务。

TAT agent 运行在CVM 或 Lighthouse 内部，负责执行具体的任务，并上报执行结果给服务端。
<img src="/images/tat/1.png">
<!-- more -->

TAT agent 同时支持了 Windows( 32bit 64bit) 和 Linux（32bit 64bit），包括但不限于如下操作系统：

- Tencent Linux
- CentOS
- Ubuntu
- Debian
- openSUSE
- SUSE
- CoreOS
- Windows Server


TAT agent 的亮点

- 采用纯 rust 编写，内存占用少，内存安全，性能高
- 一套代码多平台编译，维护方便
- agent 无需外网权限，无需用户开启端口、无需配置 SSH 密钥
- 完整地预加载了环境变量，用户脚本可以无缝迁移
- windows 指定执行用户无需配置用户账号密码
- 无缝升级，agent 升级用户无感知，不影响已有任务运行


## 快速上手
[快速上手文档](https://cloud.tencent.com/document/product/1340/50821)

TAT agent 已开源，欢迎多多贡献：[github](https://github.com/Tencent/tat-agent)

### 架构图
TAT agent 内部架构图
<img src="/images/tat/2.png">


TAT agent 内部主要分两大模块，Main 模块负责主逻辑的执行，如接收 ws server 的 kick，并执行具体的任务。 Bypass 会定时触发、负责执行一些旁路逻辑，如心跳上报、定期kick、自更新检查等。

Main:
- kick_reciever: 接收 kick 并启动 worker来执行工作。kick 来自服务端的 ws server，或者 agent 内部的定时 kick。
- task_worker: task 的执行器，负责执行任务、上报任务状态和输出等。

Bypass:
- ping_sender: 上报agent 心跳。持续上报心跳以保证 agent 处于 Online 状态
- schedule_kick： 定时 kick 触发 agent 主动从 server 端拉取 tasks，当服务端的 ws server 异常导致无法 kick agent 时，agent 定时 kick 可以作为兜底逻辑保证 agent 正常运行
- running_task_nums_checker： 检查当前是否有 task 正在运行。agent 拉取到新版本后，只有在无用户task 运行时才会重新加载自身完成升级。
- update_checker: agent 自更新
- task_timeout_checker: 检查超时的 task，并 kill 掉该 task

## TAT agent 功能介绍
<img src="/images/tat/3.png">

## TAT agent 代码结构
```
├── CHANGELOG.md // agent 各版本更新历史
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── CONTRIBUTING_COMMIT.md
├── Cargo.lock
├── Cargo.toml // agent 版本和依赖相关
├── Cross.toml // 使用 cross 进行跨平台编译
├── Dockerfile // 将 agent 容器化，主要用于测试环境 E2E 测试
├── LICENSE
├── Makefile
├── README-ZH.md
├── README.md
├── install // agent 安装相关脚本
├── src // 源代码
├── target // 产物目录
├── tests // 测试代码
```

