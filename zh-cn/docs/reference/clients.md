---
title: "处理Redis客户端"
linkTitle: "客户处理"
weight: 5
description: >
    Redis服务器如何管理客户端连接
aliases:
    - /topics/clients
---

这个文档提供了有关Redis如何处理网络层级别的客户端的信息：连接、超时、缓冲区和其他类似的主题都在这里涵盖。

本文档中的信息仅适用于 Redis 版本2.6 或更高版本。

## 接受客户端连接

Redis在配置的TCP端口和启用时会接受客户端连接的Unix套接字。当接受新的客户端连接时，将执行以下操作：

* 由于Redis使用多路复用和非阻塞I/O，将客户端套接字置于非阻塞状态。
* 设置`TCP_NODELAY`选项以确保连接没有延迟。
* 创建一个*可读*的文件事件，以便Redis能够在套接字上有新数据可读时立即收集客户端查询。

客户端初始化之后，Redis 会检查是否已经达到了同时连接客户端的限制（使用 `maxclients` 配置指令进行配置，详细信息请参见本文档的下一节）。

当Redis无法接受新的客户端连接，因为已经达到了最大客户端数限制时，它会尝试向客户端发送错误以使其意识到这种情况，并立即关闭连接。
即使Redis立即关闭连接，错误消息也会传递到客户端，因为新的套接字输出缓冲区通常足够大以容纳错误消息，因此内核会处理错误的传输。

## 客户端请求的顺序是如何服务的？

订单的确定顺序取决于客户端套接字文件描述符的组合和内核报告事件的顺序，因此应将顺序视为未指定的。

然而，当为客户端提供服务时，Redis会执行以下两个操作：

* 每当有新的数据要从客户端套接字中读取时，它只执行一次`read()`系统调用。这确保如果有多个客户端连接，并且其中一些客户端以高速发送查询，其他客户端不会受到惩罚，也不会出现延迟问题。
* 然而，一旦从客户端读取新数据，当前缓冲区中包含的所有查询将按顺序处理。这提高了局部性并且不需要迭代第二次以查看是否有需要一些处理时间的客户端。

## 最大同时连接客户端数

在Redis 2.4中，有一个硬编码的限制，最多可以同时处理的客户端数量。

在Redis 2.6及更高版本中，此限制是动态的：默认情况下设置为10000个客户端，除非在`redis.conf`中通过`maxclients`指令另行规定。

然而，Redis会与内核查询我们可以打开的最大文件描述符数量（检查*软限制*）。如果限制小于我们希望处理的最大客户端数量加32（这是Redis为内部使用保留的文件描述符数量），则最大客户端数量将更新以匹配当前操作系统限制下它*真正能够处理*的客户端数量。

当`maxclients`设置为大于Redis所支持的数字时，在启动时会记录一条消息：

```
$ ./redis-server --maxclients 100000
[41422] 23 Jan 11:28:33.179 # Unable to set the max number of files limit to 100032 (Invalid argument), setting the max clients configuration to 10112.
```

在配置Redis以处理特定数量的客户端时，最好确保操作系统对每个进程的最大文件描述符数也相应设置。

在Linux下，可以使用以下命令在当前会话和系统范围内设置这些限制：

* `ulimit -Sn 100000 ＃ 只有在硬限制足够大的情况下才能起作用。`
* `sysctl -w fs.file-max=100000`

## 输出缓冲区限制

Redis需要为每个客户端处理一个可变长度的输出缓冲区，因为一个命令可能会产生大量需要传输给客户端的数据。

但是，客户端发送更多的命令并产生更多的输出，以比Redis向客户端发送现有输出的速率更快的速度服务是可能的。这在Pub/Sub客户端中尤为常见，特别是当客户端无法以足够快的速度处理新消息时。

两种情况都会导致客户端输出缓冲区增长并消耗更多内存。因此，默认情况下，Redis为不同类型的客户端设置了输出缓冲区大小限制。当达到限制时，客户端连接将被关闭，并在Redis日志文件中记录事件。

Redis使用两种类型的限制：

1. 内存限制：Redis可以使用的最大内存量可以通过配置文件或命令行选项进行设置。一旦达到内存限制，Redis将根据所选的策略进行内存回收。

2. 连接限制：Redis可以同时处理的最大连接数可以通过配置文件或命令行选项进行设置。一旦达到连接限制，Redis将拒绝新的连接请求。

* **硬限制**是一个固定的限制，当达到时，Redis将尽快关闭客户端连接。
* **软限制**则是一个依赖于时间的限制，例如，每10秒钟的32兆字节软限制意味着如果客户端的输出缓冲区在连续的10秒钟内大于32兆字节，连接将被关闭。

不同类型的客户有不同的默认限制：

* **普通客户端** 默认没有限制，这意味着没有任何限制，因为大多数普通客户端使用阻塞实现，发送一个命令并等待完全读取回复后再发送下一个命令，所以在普通客户端中关闭连接始终是不可取的。
* **发布/订阅客户端** 默认的硬限制为32兆字节，软限制为60秒内8兆字节。
* **副本** 默认的硬限制为256兆字节，软限制为60秒内64兆字节。

在运行时，可以使用`CONFIG SET`命令更改限制，也可以使用Redis配置文件`redis.conf`进行永久性更改。有关如何设置限制的更多信息，请参阅Redis发行版中的示例`redis.conf`。

## 查询缓冲区硬限制

每个客户端也受到查询缓冲区限制的限制。这是一个不可配置的硬限制，当客户端的查询缓冲区（也就是我们用来累积来自客户端的命令的缓冲区）达到1GB时，连接将会关闭，并且实际上这只是一个极端限制，以避免在客户端或服务器软件出现错误时导致服务器崩溃。

## 客户驱逐

Redis 被设计用来处理大量的客户端连接。
客户端连接往往会消耗内存，当连接数量很多时，聚合内存消耗可能会非常高，导致数据逐出或内存不足错误。
可以在一定程度上通过 [输出缓冲区限制](#output-buffer-limits) 来减轻这些情况，但 Redis 允许我们通过更健壮的配置来限制所有客户端连接使用的聚合内存。


这个机制被称为**客户端驱逐**，实质上是一种安全机制，一旦所有客户端的内存使用量总和超过阈值，将断开客户端的连接。
该机制首先会尝试断开使用内存最多的客户端的连接。
它会断开最少数量的客户端连接，以使总连接数回到`maxmemory-clients`阈值以下。

`maxmemory-clients` 定义了所有连接到 Redis 的客户端的最大内存使用量的总和。
聚合将考虑客户端连接使用的所有内存：[查询缓冲区](#query-buffer-hard-limit)、输出缓冲区和其他中间缓冲区。

请注意，复制品和主连接不受客户端驱逐机制影响。因此，这些连接永远不会被驱逐。

`maxmemory-clients` 可以通过配置文件 (`redis.conf`) 或使用 `CONFIG SET` 命令进行永久设置。
此设置可以是 0（表示无限制），以字节为单位的大小（可能带有 `mb`/`gb` 后缀），
或者可以使用 `%` 后缀表示 `maxmemory` 的百分比（例如将其设置为 `10%` 将意味着将占用 `maxmemory` 配置的 10%）。

默认设置为0，表示默认情况下客户端逐出被关闭。
然而，对于任何大规模的生产部署，强烈建议配置一些非零的 `maxmemory-clients` 值。
例如，`5%` 这样的值可以作为一个好的起点。

有可能标记一个特定的客户端连接，将其排除在客户端逐出机制之外。
这对控制路径连接非常有用。
例如，如果您有一个应用程序通过`INFO`命令监视服务器并在出现问题时提醒您，您可能希望确保该连接不会被逐出。
您可以使用以下命令（来自相关客户端的连接）来实现这一点：

`CLIENT NO-EVICT` `on`

你可以通过以下方式恢复到原来的状态：

`CLIENT NO-EVICT` `off`
`CLIENT NO-EVICT` `off`

有关更多信息和示例，请参考默认的`redis.conf`文件中的`maxmemory-clients`部分。

从Redis 7.0开始，客户端驱逐功能可用。

## 客户端超时

默认情况下，Redis最新的版本不会在客户端无操作多秒钟后关闭与客户端的连接：连接将一直保持开启。

然而，如果您不喜欢这种行为，您可以配置一个超时时间，这样只要客户端闲置超过指定的秒数，客户端连接就会被关闭。

您可以通过 `redis.conf` 或者简单地使用 `CONFIG SET timeout <value>` 来配置此限制。

请注意，超时仅适用于普通客户端，并且**不适用于Pub/Sub客户端**，因为Pub/Sub连接是一种*推送样式*的连接，因此空闲的客户端是正常情况。

即使默认情况下连接不受超时限制，但在两种情况下设置超时是有意义的：

* 任务关键应用程序，当客户端软件中的错误会使 Redis 服务器产生大量空闲连接，导致服务中断时。
* 作为调试机制，以便在客户端软件中的错误饱和服务器使其产生大量空闲连接，无法与服务器进行交互时仍能连接到服务器。

超时不是非常精确的：Redis避免设置计时器事件或运行O（N）的算法来检查空闲的客户端，因此检查是分批进行的。这意味着尽管超时设置为10秒，但如果同时连接了许多客户端，例如在12秒后，客户端连接可能会关闭。

## CLIENT命令

Redis `CLIENT` 命令允许您检查每个已连接的客户端的状态，杀死特定的客户端，并为连接命名。如果您在大规模使用 Redis，这是一个非常强大的调试工具。

`CLIENT LIST`命令用于获取连接的客户端列表及其状态：

```
redis 127.0.0.1:6379> client list
addr=127.0.0.1:52555 fd=5 name= age=855 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client
addr=127.0.0.1:52787 fd=6 name= age=6 idle=5 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=ping
```

在上面的示例中，有两个客户端连接到Redis服务器。让我们来看一下返回的一些数据代表什么：

* **addr**：客户端地址，即与Redis服务器连接时使用的客户端IP和远程端口号。
* **fd**：客户端套接字文件描述符号。
* **name**：客户端名称，由`CLIENT SETNAME`设置。
* **age**：连接存在的秒数。
* **idle**：连接空闲的秒数。
* **flags**：客户端类型（N表示普通客户端，请查看[完整的标志列表](https://redis.io/commands/client-list)）。
* **omem**：客户端用于输出缓冲区的内存使用量。
* **cmd**：最后执行的命令。

请参阅[`CLIENT LIST`](https://redis.io/commands/client-list)文档以获取完整的字段列表和其用途。

在获取客户端列表后，您可以使用`CLIENT KILL`命令关闭客户端的连接，并将客户端地址指定为其参数。

以下命令`CLIENT SETNAME`和`CLIENT GETNAME`可用于设置和获取连接名称。从Redis 4.0版本开始，客户端名称会显示在`SLOWLOG`输出中，以帮助识别导致延迟问题的客户端。

## TCP保活

从版本3.2开始，Redis默认启用TCP keepalive（`SO_KEEPALIVE`套接字选项），并设置为约300秒。该选项对于检测死亡对等体（即使它们看起来已连接，也无法访问的客户端）非常有用。此外，如果在客户端和服务器之间存在需要看到一些流量以保持连接打开的网络设备，该选项将防止意外的连接关闭事件发生。