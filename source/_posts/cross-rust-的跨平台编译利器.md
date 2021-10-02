---
title: 'cross: rust 的跨平台编译利器'
tags:
  - rust
  - cross
  - 跨平台编译
categories:
  - rust
abbrlink: 3d1fe733
date: 2021-10-02 18:27:56
---

## 1. cross 介绍
cross 是 `0` 配置的 Rust 跨平台编译工具，它有着使用简单、功能强大的特点，极大地方便了 rust 跨平台项目的构建、测试和发布。
##  2. cargo 跨平台编译及问题
### 2.1 cargo 跨平台编译
cargo 本身支持跨平台编译，一般使用有如下命令：
- 查看 target 列表
  `rustup target list`

- 安装新的 target，以 `i686-unknown-linux-gnu` 为例
  `rustup target add i686-unknown-linux-gnu`
- 编译 linux 32 gpu 的产物
  `cargo build --release --target i686-unknown-linux-gnu`

<!-- more -->

### 2.2 问题
以上编译产物是非静态的二进制，运行时会对动态环境有依赖，如依赖 libc.so、依赖 libssl.so 等，以下是对某个 rust 编写的 agent 使用 `x86_64-unknown-linux-gnu` 编译产物的依赖：
```
# ldd agent
        linux-vdso.so.1 =>  (0x00007fffa0b6f000)
        /$LIB/libonion.so => /lib64/libonion.so (0x00007f6ce530f000)
        libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007f6ce3819000)
        librt.so.1 => /lib64/librt.so.1 (0x00007f6ce3611000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f6ce33f5000)
        libm.so.6 => /lib64/libm.so.6 (0x00007f6ce30f3000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007f6ce2eef000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f6ce2b21000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f6ce51f6000)
```

而对于在运行客户环境的 agent，要尽量减少外部的依赖，因此 agent 需要编译为纯静态的二进制， 即编译target 为 `x86_64-unknown-linux-musl`, 此时使用 cargo 进行编译就不方便了。

如果使用 musl 的方式编译，需要依赖 musl libc 和 musl-gcc。
参考： https://rustwiki.org/zh-CN/edition-guide/rust-2018/platform-and-target-support/musl-support-for-fully-static-binaries.html#musl-%E6%94%AF%E6%8C%81%E5%AE%8C%E5%85%A8%E9%9D%99%E6%80%81%E4%BA%8C%E8%BF%9B%E5%88%B6%E6%96%87%E4%BB%B6

如果要支持多平台的产物编译，除了要安装对应的musl target，还需要各个平台的 musl libc 和 musl-gcc 版本，搭建跨平台编译环境就很不方便了。

##  3. cross 进行跨平台编译
于是 `cross` 就应运而生了，它通过容器技术屏蔽了各个环境编译依赖的细节，实现了 `0` 配置的跨平台编译体验，而且保持了同 cargo 一致的体验。

演示如下：
### 3.1 安装 cross
`crago install cross`
之后就可以使用 `cross` 替换到 `cargo` 命令了。
### 3.2 编译 `x86_64-unknown-linux-musl`
`cross build --release --target x86_64-unknown-linux-musl`

cross 会根据 target 拉取对应的容器镜像，然后在容器镜像内部进行编译， 编译的产物也会挂载在本机的 `target/x86_64-unknown-linux-musl` 目录下。

有了 `cross`，就不用再纠结于各个 target 的复杂依赖，可以很清爽的进行跨平台编译了。

## 4. cross 实现原理
那么 `cross` 是如何实现的呢？
主要有两点：
- 使用容器镜像屏蔽了各平台的依赖细节
- 内部调用 `cargo` 保证了 cross 与 cargo 命令完全一致的使用体验
### 4.1 使用容器镜像屏蔽了各平台的依赖细节
以 `target/x86_64-unknown-linux-musl` 为例， 执行 cross build 时，cross 会拉取该 target 对应的镜像，并在镜像中执行 cargo build 的命令，该 target 的镜像的 dockerfile 里封装了依赖的细节。

Dockerfile.x86_64-unknown-linux-musl

``` 
# 基础镜像
FROM ubuntu:20.04

# 安装基本的lib，如 gcc、make git 等
COPY common.sh lib.sh /
RUN /common.sh

# 安装 cmake
COPY cmake.sh /
RUN /cmake.sh

# 安装 xargo
COPY xargo.sh /
RUN /xargo.sh

# 编译 musl-gcc
COPY musl.sh /
RUN /musl.sh TARGET=x86_64-linux-musl

# 设置 target 和 链接器
ENV CARGO_TARGET_X86_64_UNKNOWN_LINUX_MUSL_LINKER=x86_64-linux-musl-gcc \
    CC_x86_64_unknown_linux_musl=x86_64-linux-musl-gcc \
    CXX_x86_64_unknown_linux_musl=x86_64-linux-musl-g++
```

感兴趣的可以看下各脚本的实现细节：
[dockerfile 地址](https://github.com/rust-embedded/cross/blob/master/docker/Dockerfile.x86_64-unknown-linux-musl)

### 4.2 内部调用 `cargo` 保证了 cross 与 cargo 命令完全一致的使用体验
cross 内部在起来对应target 的容器后，内部调用 cargo 以达到同 cargo 的一致体验。参考：
[调用代码](https://github.com/rust-embedded/cross/blob/2b2a01220c73ae517129019d436d382412d1928b/src/main.rs#L389)

## 5. 结语
`cross` 作为一款 `0` 配置的 Rust 跨平台编译利器，极大地便捷了日常开发和测试、流水线跨平台和发布等工作，
此外 cross 也支持 target 自定义镜像，方便用户将自定义的静态链接库等很方便地集成进去。 
