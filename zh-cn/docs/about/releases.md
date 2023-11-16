---
title: "Redis发布周期"
linkTitle: "发布周期"
weight: 4
description: Redis 的新版本是如何发布的？
aliases:
    - /topics/releases
---

Redis是一种系统软件，也是一种保存用户数据的系统软件类型，因此它是软件堆栈中最关键的部分之一。

出于这个原因，Redis的发布周期是为了确保高度稳定的发布，即使以较慢的周期为代价。

新的版本发布在[Redis GitHub 仓库](http://github.com/redis/redis)中，并且也可通过[下载](/download)获得。公告会通过[Redis 邮件列表](http://groups.google.com/group/redis-db)和[@redisfeed 的 Twitter](https://twitter.com/redisfeed)发送。

## 发布周期

给定的 Redis 版本可以处于三个不同的稳定性级别：

* 不稳定
* 发行候选版
* 稳定版

### 不稳定的树

Redis的不稳定版本位于[Redis GitHub存储库](http://github.com/redis/redis)的`unstable`分支中。

这个分支是源代码树，大部分的新功能都在开发中。`unstable` 分支不被视为适合生产环境使用：它可能包含严重的 bug、不完整的功能，以及潜在的不稳定性。

然而，我们努力确保即使在开发环境中，不稳定的分支也能在大部分时间内可用且没有重大问题。

### 发布候选版

Redis 的新次要和主要版本都是从 `unstable` 分支派生出来的。
派生出的分支的名称就是目标发布版本。

例如，当Redis 6.0作为发布候选版本发布时，`unstable`分支被分叉为`6.0`分支。新的分支是该版本的发布候选版本（RC）。

在发布的时间范围内，可以稳定的修复错误和新功能被提交到不稳定分支，并回溯到候选发布分支。`unstable`分支可能包含不属于当前发布候选的额外工作，并计划在未来的发布中进行。

第一个候选版本（RC1）一旦可以用于开发目的和测试新版本，就会发布。在这个阶段，大多数新功能和新版本带来的变化都已经准备就绪，发布的目的是收集公众的反馈意见。

每隔大约三周，将发布后续的候选版本，主要用于修复错误。这些版本可能还会添加新功能和引入更改，但是其频率和对最终发布候选版本的潜在风险会逐渐降低。

### 稳定树

一旦开发结束，发布候选版本的关键错误报告的频率减少，它就准备好进行最终发布。此时，发布被标记为稳定，并以“0”作为其补丁级版本进行发布。

## 版本控制

稳定版发布自由遵循通常的`主要.次要.补丁`语义化版本控制模式。主要目标是提供关于向后兼容性的明确保证。

### Patch级别版本

主要的修补程序主要包括错误修复，很少引入任何兼容性问题。

从之前的补丁级版本升级通常是安全和无缝的。

只要这些不会产生重大影响或引入与操作相关的问题，就可以添加新功能和配置指令，或更改默认值。

### 次要版本

次要版本通常提供了成熟性和扩展功能。

升级到次要版本之间不会引入任何应用级别的兼容性问题。

次要版本可能包含引入操作相关的不兼容性的新命令和数据类型，包括数据持久性格式和复制协议的更改。

### 主要版本

主要版本引入了新的功能和重大变更。

理想情况下，这些不会引入应用层的兼容性问题。

## 发布计划

每年计划发布一个新的主要版本。

一般来说，每次主要发布之后，会在六个月后发布一个次要版本。

根据需要发布补丁以修复高紧急性问题或者当一个稳定版本积累了足够多的修复来证明其必要性。

**对于敏感事项和安全问题，请联系核心团队，发送电子邮件至 [redis@redis.io](mailto:redis@redis.io)。**

## 支持

作为一项规定，我们通常不支持旧版本，因为我们努力确保Redis API在大部分情况下向后兼容。

升级到较新版本是推荐的方法，通常是简单的。

最新的稳定版本始终得到全面支持和维护。

两个附加版本只接受维护，这意味着只有关键错误和重大安全问题的修复被承诺并发布为补丁。

* 最新稳定版本的上一小版本。
* 上一个稳定主要版本。
 
例如，考虑以下假设的版本：1.2、2.0、2.2、3.0、3.2。

当2.2版本是最新的稳定版时，2.0和1.2版本都得到维护。

一旦3.0.0版本取代2.2成为最新稳定版本，2.0和2.2版本将继续维护，而1.x版本将不再更新。

这个过程将在版本3.2.0重复进行，之后只维护版本2.2和3.0。

上述只是指南而不是石刻的规则，并不会取代常识。
