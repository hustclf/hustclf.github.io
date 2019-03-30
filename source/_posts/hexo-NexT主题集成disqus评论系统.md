---
title: hexo NexT主题集成disqus评论系统
tags:
  - hexo
  - next
  - 评论系统
copyright: ture
abbrlink: '8838407'
categories:
  - 博客
date: 2019-03-27 12:58:25
---


# hexo NexT主题集成disqus评论系统

## 为何需要评论系统
一个良好的博客系统，绝不仅仅是个人的琐碎日记本，把自己只言片语的感悟或者一知半解的知识散播到互联网上，这是很不负责的行为。 因为你的博文会被很多同领域的同学看到，如果博文的观点是错的，那将以讹传讹，不仅无丝毫借鉴意义，反而浪费他人的宝贵时间。那如何发现自己博文的不足甚至谬误呢？  一个良好的评论系统能帮到你。

<!-- more -->

## 评论系统的作用
- 广交朋友，多多交流
- 改进博文写作水平，修正错误的观点
- 正反馈作者，提升写作热情


## 评论系统选择
NexT主题提供了许多第三方评论系统的集成<a href="https://theme-next.iissnan.com/third-party-services.html">参考</a>，大致有如下几种：

- disqus
- Facebook Comments：国内技术同学多数没有Facebook的账号，不方便，pass
- HyperComments: 不太清楚
- 多说：不再维护
- 网易云跟帖： 不再维护
- gitment、gitalk: 调研github api, 以issue和comment的方式comment, 慎用
- Valine： 过于简单

我个人对评论系统的要求如下：

- 集成简单
- 评论内容不能丢
- 支持邮件提醒，以便能及时回复

经过多次踩坑，我最终选择了disqus作为评论系统，disqus能支持目前我需要的所有功能，并且还有较强大的统计功能（目前用不到)。

有些同学会担心disqus会被墙掉，但目前我的使用而言，没有发现被墙的情况。而且即使被墙了，我觉得也无所谓，不会翻墙的技术同学也不在我的博客受众范围内。

## 如何集成disqus
### <a href="https://disqus.com/profile/signup/">注册</a>
<img src="/images/disqus/signup.png">

### 登陆，点击 `GET STARTED` 开始创建站点，之后就可以点击右上角的 Admin 进入后台管理。
<img src="/images/disqus/login.png">

### 点击 `I want to install Disqus on my site`
<img src="/images/disqus/disqus_intent.png">

### 按照表单填写信息，记住 Website Name 这条属性。
<img src="/images/disqus/create.png">

### 接下来按照指引填写信息，完成第三步 3.Configure Disqus 后点击最下面 Complete Setup 完成创建。【中间会有一个嵌入代码的案例，不是 Next 主题的可以参考下】
<img src="/images/disqus/settings.png">

### 接下来配置主题下面的 config.yml 文件。
大于等于5.1.1版本，将 disqus 下的 enable 设定为 true，同时提供 shortname。 count 用于指定是否显示评论数量。
```
disqus:
  enable: true
  shortname:
  count: true

```
小于5.1.1 版本，设定 disqus_shortname 的值即可。
```
disqus_shortname: shortname
```



