---
title: 'git branch模型最佳实践[译]'
tags: git branch
abbrlink: c1cadd8e
categories:
  - 工程化
date: 2017-12-22 00:15:38
---

----------
【原文链接】：
[A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/)
----------

## git branch大图

下图是git branch 模型的全景图，可以先阅读后续章节，最后再回头查看

<img src="/images/gitflow.png">

<!-- more -->

## 为什么使用git

网上已经有很多关于git与集中代码控制软件的讨论，彻底的比较可以参考[这里](https://git.wiki.kernel.org/index.php/GitSvnComparsion)。作为一个开发者，相比其他代码管理软件，我更倾向于使用git。git彻底地改变了我们对于分支和合并的概念的认识。用过csv/Subersion的人都清楚，分支和合并的工作是很巨大的，而且会花费很长时间。

但对于git而言，分支和合并是极其简单和快速的，他们已经成为开发者日常工作流的一部分。因此分支和合并不再为开发者惧怕。

接下来，我们将讨论git的开发模型。为了能让软件开发流程更加合理， 每个团队成员都应该遵守这个模型提供的开发流程。


## git是分布式的又是集中的
我们一般会在github或者git服务器上建立一个远程库，这个库被认为是中心库（与svn等不同，git是分布式的，技术层面上不存在中心库, 我们把中心库命名为origin， 大家对这都很熟悉

<img src="http://nvie.com/img/centr-decentr@2x.png">

开发者从origin pull代码，并将代码push到origin。因为git是分布式的，每个人除了从origin来push和pull代码， 每个人也可以从对方那push或者pull代码。如图所示， alice david bob clair之间也可以互相pull push代码

## 主要分支
一般而言，每个项目开发中都会有如下两个始终存在的分支：

+ master
+ develop

<img src="http://nvie.com/img/main-branches@2x.png">

其中master分支为稳定分支，用来发布稳定的软件版本。
develop分支为开发分支，用于开发人员开发  

## 扩展分支
除了master和develop分支，我们还可以使用其他扩展分支，来支持更加丰富的需求。如下：

+ 团队成员需要并行开发新功能
+ 追踪某个功能的开发流程
+ 准备发布线上 release版本
+ 快速修复线上 bug

跟主要分支不同的是，扩展分支只有有限的存活时间，当功能完成后，这些分支都将被删除。


为了满足以上需求，我们会用到的扩展分支有：

+ feature 分支 （用于团队成员并行开发新功能）
+ release 分支 (用于发布release版本，并标记tag)
+ hotfix 分支 (用于快速修复线上bug)

下面将会详细介绍这些扩展分支的使用过程。

## feature 分支
可能来自于:

```
develop
```

必须要合并回：

```
develop
```

分支命名：

```
除了master develop release-* hotfix-* 之外的其他名字
（建议使用feature-*）
```

<img src="http://nvie.com/img/fb@2x.png">

feature分支用来为后续release版本开发新的特性或者功能。开始开发新feature时，并不清楚该特性会在何时被合并回去。只要该特性还在开发中，该feature分支就会一直存在。 feature分支最终会被合并进develop分支（如果确定在下一个release版本中增加该功能），或者被舍弃掉（比如失败的新功能实验）


feature分支仅仅存在开发者的本地repo中，不应该被提交到origin

### 新建一个feature分支
当开始开发一个新特性时，从develop分支上切一个feature分支

```
$ git checkout -b myfeature develop
Switched to a new branch "myfeature"
```

### 新特性开发完，合并回develop分支
当明确后续版本将要加入这些新特性时， 将feature分支合并到develop分支。

```
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff myfeature
Updating ea1b82a..05e9557
(Summary of changes)
$ git branch -d myfeature
Deleted branch myfeature (was 05e9557).
$ git push origin develop
```

--no-ff参数表示合并必须要产生一个新的commit操作，而不是使用git默认的“fast-forward”模式。这样做可以避免丢失feature分支存在的历史信息并且可以将feature分支的多次commit合并为develop分支的一次提交。

二者的区别见下图：
<img src="http://nvie.com/img/merge-without-ff@2x.png">


从图中可以看出，在不使用“--no-ff”的情况下，跟踪一个已实现的特性的开发历史将变得十分麻烦（你仅仅只能从git log的信息中推断分支的开发情况）， 而且特性的回滚也将让人头痛。

虽然使用“--no-ff”会增加一些commit，并且合并的速度要稍慢一些，但从长远看绝对是利大于弊的。

## release 分支
可能来自于:

```
develop
```

必须要合并回：

```
develop and master
```

分支命名：

```
release-*
```

release分支是为新的线上版本的发布做准备的。release分支允许在上线前做最后的修改。此外它还允许很小的bug fix和更新一些元数据（如version number、 build date 等）。这些事情交给了release分支处理，develop分支则可以专注地为下一次版本发布继续开发新特性了。

从develop分支上切出release分支的正确时刻，应该是在develop分支对于新版本的功能已经到达一个比较理想的状态，新版本的新增特性的分支都已经验证通过并合并回develop分支上。

新版本的版本号，是在release分支从develop分支切出时确认的，而不是在develop分支上确认后再切出release分支。

### 新建release分支
release分支是从develop分支产生的。比如，当前线上的版本是1.1.5并且我们有一个新版本即将发布。develop分支的功能已经就绪并且我们决定新版本的版本号为1.2（不是1.1.6或者2）。因此我们从develop分支上切出release分支，release分支命名上反映版本号

```
$ git checkout -b release-1.2 develop
Switched to a new branch "release-1.2"
$ ./bump-version.sh 1.2
Files modified successfully, version bumped to 1.2.
$ git commit -a -m "Bumped version number to 1.2"
[release-1.2 74d9424] Bumped version number to 1.2
1 files changed, 1 insertions(+), 1 deletions(-)
```

切到release分支上后，我们使用bump-version.sh来修改所有涉及版本号的内容（假设我们有这个脚本），并将该分支交由qa进行测试。

release分支会存在一段时间，直到qa测试完所有问题并修复掉所有bug，达到上线状态。release分支不再接受大的feature修改，大的feture应该先被合并到develop分支，并在下次发布新版本时，包含进新的release分支中。

### 完成release分支
当release分支达到准上线状态时，就可以进行后续操作了。首先应该讲release分支合并进master分支；其次master分支合并后，应该给master分支打上浅简易懂的tag；最终release分支应该merge回develop分支，因此在release上的新feature的bug fix也会被包含进develop中

```
$ git checkout master
Switched to branch 'master'
$ git merge --no-ff release-1.2
Merge made by recursive.
(Summary of changes)
$ git tag -a 1.2
```

此时发布已经完成，并且已打上了tag作为后续的参考

将release内容，merge回develop分支

```
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff release-1.2
Merge made by recursive.
(Summary of changes)
```

这一步可能会产生conflict， 修复再提交就可以啦。


最后，删除release分支

```
$ git branch -d release-1.2
Deleted branch release-1.2 (was ff452fe).
```


## hotfix 分支
可能来自于:

```
master
```

最终要合并回：

```
develop and master
```

分支命名：

```
hotfix-*
```

hotfix分支和release分支很相似，他们都是用来发布新版本的，尽管hotfix属于非正常发布的。当线上版本出现重大bug并且必须要马上解决时，hotfix分支这时候就派上用场了。严重的线上bug必须要立即解决，从master分支上切出hotfix分支来处理这些bug，并标记上相应的tag.

<img src="http://nvie.com/img/hotfix-branches@2x.png">

hotfix分支保证了团队成员可以继续在develop分支上进行开发，而线上的bug有能得到及时的修复。

### 新建hotfix分支
hotfix分支是从master分支切出的。比如，假设当前线上版本是v1.2，v1.2版本有重大的问题。但develop分支目前正在开发中，还不稳定，因此我们需要从master分支切出hotfix分支来解决问题。

```
$ git checkout -b hotfix-1.2.1 master
Switched to a new branch "hotfix-1.2.1"
$ ./bump-version.sh 1.2.1
Files modified successfully, version bumped to 1.2.1.
$ git commit -a -m "Bumped version number to 1.2.1"
[hotfix-1.2.1 41e61bb] Bumped version number to 1.2.1
1 files changed, 1 insertions(+), 1 deletions(-)

```
(不要忘记修改新的版本号)

修复完bug, 并commit

### 完成hotfix分支
完成hotfix分支后，需要合并回master分支。并且还要合并回develop分支，这样就能保证发布下个版本时，本次版本的bug fix肯定会包含其中。

首先更新master，并打上标记

```
$ git checkout master
Switched to branch 'master'
$ git merge --no-ff hotfix-1.2.1
Merge made by recursive.
(Summary of changes)
$ git tag -a 1.2.1
```

然后将hotfix合并回develop

```
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff hotfix-1.2.1
Merge made by recursive.
(Summary of changes)
```

上面的规则有一个例外。当当前有release分支存在时， 应该将hotfix分支合并回release分支， 这样操作，最终也会导致hotfix做的修改最终合并回develop分支（当release分支完成使命，合并回develop分支时）。如果develop分支也迫切需要这些bug fix并且不能等待release分支合并时，也可以直接将hotfix合并到develop分支上。

最后，删除这个临时分支

```
$ git branch -d hotfix-1.2.1
Deleted branch hotfix-1.2.1 (was abbe5d6).
```

## 总结
文章头部的大图对我们的工程开发是极其有用的，它的组织形式是很优雅和浅简易懂的，能够更好的帮助团队成员理解分支开发和发布流程。

获取高清的pdf,点击[这里](http://nvie.com/files/Git-branching-model.pdf)



-------------

【原文链接】：[A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/)



