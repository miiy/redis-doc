---
title: Redis 持久化
linkTitle: 持久化
weight: 7
description: Redis如何将数据写入磁盘
aliases: [
    /topics/persistence,
    /topics/persistence.md,
    /docs/manual/persistence,
    /docs/manual/persistence.md
]
---

持久性是指将数据写入耐用存储介质，如固态硬盘(SSD)。Redis 提供了一系列持久性选项，包括：

* **RDB**（Redis数据库）：RDB持久化在指定的时间间隔内对数据集执行时点快照。
* **AOF**（Append Only File）：AOF持久化记录服务器接收到的每个写操作。在服务器启动时，可以重新播放这些操作，重建原始数据集。命令使用与Redis协议相同的格式进行记录。
* **无持久化**: 您可以完全禁用持久化。有时在缓存时使用。
* **RDB + AOF**：您也可以在同一个实例中同时使用AOF和RDB。

如果您不想考虑这些不同持久化策略之间的权衡，您可能想考虑使用[Redis Enterprise的持久化选项](https://docs.redis.com/latest/rs/databases/configure/database-persistence/)，可以通过用户界面预先配置。

要了解有关如何评估您的Redis持久性策略的更多信息，请继续阅读。

## RDB的优势

* RDB 是 Redis 数据的非常紧凑的单文件表示。RDB 文件非常适合于备份。例如，您可能希望每小时存档一次 RDB 文件，保存最近 24 小时的备份，并每天保存一个 RDB 快照备份 30 天。这使得在灾难情况下可以轻松地恢复不同版本的数据集。
* RDB 对于灾难恢复非常好，因为它是一个紧凑的单个文件，可以传输到远程数据中心，或者存储到 Amazon S3（可能经过加密）。
* RDB 最大化了 Redis 的性能，因为 Redis 父进程只需分叉一个子进程来进行持久化存储，而父进程本身不会执行任何磁盘 I/O 或类似操作。
* 与 AOF 相比，RDB 在具有大数据集的情况下可以更快速地进行重启。
* 在副本上，RDB 支持[重启和故障切换后的部分重新同步](https://redis.io/topics/replication#partial-resynchronizations-after-restarts-and-failovers)。

## RDB 的缺点

* 如果您需要在 Redis 停止工作（例如停电后）时尽量减少数据丢失的几率，RDB 不是一个好选择。您可以配置不同的“保存点”，在这些保存点后会生成一个 RDB 快照（例如每隔至少五分钟或对数据集进行 100 次写入之后，您可以有多个保存点）。但是通常，您会每五分钟或更长时间创建一个 RDB 快照，因此，如果 Redis 因任何原因而停止工作并没有正确关闭，您应该准备好丢失最近的几分钟数据。
* 为了使用子进程将数据持久化到磁盘上，RDB 需要经常使用 fork()。如果数据集较大，这个过程可能会耗费一些时间，并且如果数据集非常大且 CPU 性能不佳，可能会导致 Redis 在一些毫秒甚至一秒钟的时间内无法为客户端提供服务。AOF 也需要使用 fork()，但频率较低，并且您可以调整日志重写的频率，而不会对数据的持久性产生任何影响。

## AOF 优势

* 使用AOF Redis更耐用：您可以选择不同的fsync策略：完全不进行fsync，每秒进行fsync，或每次查询都进行fsync。使用默认的每秒进行fsync的策略，写入性能仍然很好。fsync是在后台线程中执行的，当没有进行fsync时，主线程会尽力执行写入操作，因此您最多只会丢失一秒钟的写入数据。
* AOF日志是一个只追加的日志，因此不存在搜索或损坏问题，即使在断电的情况下。即使由于某些原因（例如磁盘空间满了或其他原因）导致日志以半写入的命令结束，redis-check-aof工具也能够轻松修复它。
* 当AOF文件过大时，Redis可以在后台自动进行重写。重写过程完全安全，因为在Redis继续向旧文件追加数据时，会生成一个完全新的文件，其中包含创建当前数据集所需的最小操作集，一旦第二个文件准备就绪，Redis会切换到新文件并开始向其追加数据。
* AOF以易于理解和解析的格式按顺序记录所有操作的日志。您甚至可以轻松导出AOF文件。例如，即使您意外使用了`FLUSHALL`命令清空了所有数据，只要在此期间没有进行日志重写，您仍然可以通过停止服务器，删除最新的命令，然后重新启动Redis来保存数据集。

## AOF 的缺点

* 对于相同的数据集，AOF文件通常比等价的RDB文件要大。
* 取决于确切的fsync策略，AOF可能比RDB慢。一般来说，当fsync设置为"每秒钟"时，性能仍然非常高，在负载高的情况下，禁用fsync后它应该和RDB一样快。然而，在巨大的写入负载情况下，RDB能够提供更多关于最大延迟的保证。

**Redis < 7.0**

* 如果重写期间对数据库进行写入操作，AOF 将使用大量内存（这些写入操作将缓存在内存中，并在重写结束时写入新的 AOF）。
* 在重写期间到达的所有写入命令将写入磁盘两次。
* Redis 可能会在重写结束时冻结对新 AOF 文件的写入和对这些写入命令进行 fsync 操作。
  
好的，那我应该使用什么？

要使用持久性方法的一般指示是，如果您希望获得与PostgreSQL相当的数据安全性。

如果您非常关心数据，但在灾难发生时可以容忍几分钟的数据丢失，您可以仅使用RDB即可。

有许多用户只使用AOF，但我们不鼓励这样做，因为定期进行RDB快照对进行数据库备份、快速重启以及在AOF引擎出现错误时都是一个好的思路。

以下部分将详细介绍两种持久化模型的更多细节。

## 快照功能

默认情况下，Redis将数据集的快照保存在名为`dump.rdb`的二进制文件中。如果数据集中至少有M个变化，您可以配置Redis在每N秒保存一次数据集，也可以手动调用`SAVE`或`BGSAVE`命令。

例如，如果至少有1000个键发生变化，此配置将使Redis每60秒自动转储数据集到磁盘：

    save 60 1000

这种策略被称为快照。

### 如何运作

每当 Redis 需要将数据集转储到硬盘时，都会发生以下情况：

* Redis [分叉](http://linux.die.net/man/2/fork)。现在我们有一个子进程和一个父进程。

* 孩子开始将数据集写入临时的 RDB 文件。

* 当子进程完成写入新的RDB文件后，它会替换掉旧文件。

这种方法使得Redis能够受益于写时复制的语义。

## 仅追加文件

快照不是非常持久的。如果运行Redis的计算机停止运行、电力线路故障或者您意外地 `kill -9` 您的实例，那么写入Redis的最新数据将会丢失。尽管对于某些应用程序而言这可能不是一个大问题，但对于需要完全持久性的用例来说，仅依靠Redis的快照是不可行的选项。

_Apnd只要文件_nom是红s的替rt，充识qo完ly-i又策l。自1.1版本打开。

您可以在配置文件中启用 AOF：


    appendonly yes

从现在开始，每当Redis收到一个改变数据集的命令（例如`SET`），它都会将其追加到AOF中。当你重新启动Redis时，它会重新执行AOF以重建状态。

自从Redis 7.0.0开始，Redis使用了多部分AOF机制。
也就是说，原来的单一AOF文件被拆分成基文件（最多一个）和增量文件（可能有多个）。
基文件表示在AOF重写时的数据的初始（RDB或AOF格式）快照。
增量文件包含自上一个基文件创建以来的增量更改。所有这些文件都被放在一个单独的目录中，并由一个清单文件进行跟踪。

### 日志重写

随着写操作的执行，AOF 文件会越来越大。 例如，如果您将计数器递增 100 次，最终您的数据集中会包含一个包含最终值的关键字，但您的 AOF 文件中会有 100 个条目。其中的 99 个条目并不需要用来重建当前状态。

重写是完全安全的。
当Redis继续将数据附加到旧文件时，
它会生成一个完全新的文件，其中包含创建当前数据集所需的最小操作集，
一旦第二个文件准备好，Redis会切换两个文件，并开始将数据附加到新文件。

所以Redis支持一个有趣的特性：它可以在后台重建AOF而不中断对客户端的服务。每当您发出`BGREWRITEAOF`命令，Redis都会写入重建当前内存中数据集所需的最短命令序列。如果您正在使用Redis 2.2的AOF，您需要定期运行`BGREWRITEAOF`命令。由于Redis 2.4能够自动触发日志重写（请参阅示例配置文件获取更多信息）。

自Redis 7.0.0版本开始，在安排AOF重写时，Redis父进程会打开一个新的增量AOF文件来继续写入。
子进程执行重写逻辑，生成新的基础AOF文件。
Redis将使用临时清单文件来跟踪新生成的基础文件和增量文件。
当它们就绪时，Redis将执行原子替换操作，使临时清单文件生效。
为了避免在AOF重写的重复失败和重试情况下创建许多增量文件的问题，
Redis引入了AOF重写限制机制，以确保失败的AOF重写以越来越慢的速度进行重试。

### 追加文件有多耐用？

您可以配置 Redis 将数据同步到硬盘上的次数。有三个选项：

*`appendfsync always`：每次向AOF追加新命令时都进行`fsync`。非常慢，但非常安全。请注意，命令是在多个客户端的一批命令或者一个流水线执行后追加到AOF的，因此会进行一次写操作和一次`fsync`操作(在发送回复之前)。
*`appendfsync everysec`：每秒进行一次`fsync`。足够快（自版本2.4以来，可能和快照一样快），如果出现灾难，可能会丢失1秒钟的数据。
*`appendfsync no`：永不进行`fsync`，只将您的数据交给操作系统处理。这是一种更快但不太安全的方法。通常，使用这种配置，Linux每30秒会刷新一次数据，但具体取决于内核的精确调整。

建议（也是默认）的策略是每秒执行一次`fsync`。这既快速又相对安全。`always`策略在实践中非常慢，但支持组提交，因此如果有多个并行写入操作，Redis 将尝试执行一个 `fsync` 操作。

### 如果我的AOF文件被截断了，我应该怎么办？

可能是服务器在写入AOF文件时崩溃了，或者存储AOF文件的卷在写入时已满。当发生这种情况时，AOF仍然包含表示给定时点版本的一致数据（默认AOF fsync策略可能会旧到一秒钟），但是AOF中的最后一个命令可能会被截断。
Redis的最新主要版本仍然可以加载AOF文件，只需丢弃文件中最后一个非规范的命令即可。在这种情况下，服务器会发出如下日志：

```
* Reading RDB preamble from AOF file...
* Reading the remaining AOF tail...
# !!! Warning: short read while loading the AOF file !!!
# !!! Truncating the AOF at offset 439 !!!
# AOF loaded anyway because aof-load-truncated is enabled
```

您可以更改默认配置，以强制Redis在这种情况下停止，但默认配置是继续运行，而不考虑文件中的最后一个命令是否格式正确，以确保重新启动后的可用性。

旧版本的Redis可能无法恢复，可能需要执行以下步骤：

将你的AOF文件进行备份。
使用随Redis一同提供的`redis-check-aof`工具修复原始文件：

      $ redis-check-aof --fix <filename>

* 可以选择使用`diff -u`命令检查两个文件的差异。
* 使用修复后的文件重新启动服务器。

### 如果我的AOF文件损坏了，我应该怎么办？

如果AOF文件不仅仅是被截断了，而是中间有无效的字节序列导致损坏，情况会更复杂。Redis在启动时会报错并终止运行：

```
* Reading the remaining AOF tail...
# Bad file format reading the append only file: make a backup of your AOF file, then use ./redis-check-aof --fix <filename>
```

最好的办法是运行`redis-check-aof`工具，最初不带`--fix`选项，然后了解问题，在文件中跳转到给定的偏移量，看是否可以手动修复文件：AOF使用与Redis协议相同的格式，很容易手动修复。否则，可以让工具为我们修复文件，但在这种情况下，从无效部分到文件末尾的所有AOF部分可能会被丢弃，如果损坏发生在文件的初始部分，这将导致大量数据丢失。

### 工作原理

日志重写使用的是快照已经在使用的相同的写时复制技巧。它的工作原理如下：

**Redis >= 7.0**

* Redis [分叉](http://linux.die.net/man/2/fork)，所以现在我们有一个子进程和一个父进程。

* 孩子在临时文件中开始写入新的基本AOF。

* 父进程打开一个新的增量AOF文件来继续写入更新。
  如果重写失败，旧的基本和增量文件（如果有的话），加上这个新打开的增量文件就代表了完整的更新数据集，所以我们是安全的。
  
* 当子进程完成重写基础文件后，父进程会收到一个信号，
然后使用新打开的增量文件和子进程生成的基础文件来构建一个临时清单，
并将其持久化。

- 利润！现在Redis可以原子交换清单文件，以使AOF重写的结果生效。Redis还清理旧的基础文件和任何未使用的增量文件。

**Redis < 7.0**

* Redis [分叉](http://linux.die.net/man/2/fork)，现在我们有一个子进程和一个父进程。

* 孩子开始将新的AOF写入临时文件。

* 父节点在内存缓冲区中累积所有新的更改（同时将新的更改写入旧的追加方式文件，所以如果重写失败，我们是安全的）。

* 当子进程完成对文件的重写后，父进程收到信号，并将内存缓冲区追加到子进程生成的文件末尾。

* 现在Redis自动将新文件重命名为旧文件，并开始将新数据追加到新文件中。

### 如果我目前正在使用dump.rdb快照，我该如何切换到AOF？

在2.0版本及更高版本中，有一个不同的过程来完成此操作，正如您所猜测的那样，自从Redis 2.2版以来，它更简单，并且根本不需要重启。

**Redis >= 2.2**

**备份您的最新dump.rdb文件。**
**将此备份转移到安全的位置。**
**执行以下两个命令：**
**- `redis-cli config set appendonly yes`**
**- `redis-cli config set save ""`**
**确保您的数据库包含相同数量的键。**
**确保写入正确地附加到追加式文件中。**

第一个CONFIG命令启用了追加模式文件持久化。

第二个CONFIG命令用于关闭快照持久化。这是可选的，如果您希望可以同时启用两种持久化方法。

**重要提示：**记得编辑你的redis.conf文件，打开AOF功能，否则当你重启服务器时，配置更改将会丢失，服务器将重新使用旧的配置。

**Redis 2.0**

**请进行以下操作：**
* 备份最新的dump.rdb文件。
* 将此备份转移到安全的位置。
* 停止对数据库的所有写入操作！
* 执行`redis-cli BGREWRITEAOF`命令。这将创建追加写文件（AOF）。
* 当Redis生成完AOF dump后停止服务器。
* 编辑redis.conf文件并启用追加写文件持久化。
* 重新启动服务器。
* 确保您的数据库在切换之前包含相同数量的键。
* 确保写入正确追加到追加写文件中。

## AOF和RDB持久性之间的交互


Redis >= 2.4 确保在进行 RDB 快照操作时不会触发 AOF 重写，或者在 AOF 重写正在进行时允许 `BGSAVE`。这样可以防止两个 Redis 后台进程同时进行大量的磁盘 I/O。

在快照正在进行时，当用户使用 `BGREWRITEAOF` 显式请求日志重写操作时，服务器将以 OK 状态码回复，告知用户该操作已被调度，重写将在快照完成后开始。

在启用AOF和RDB持久化的情况下，如果Redis重新启动，则将使用AOF文件来重建原始数据集，因为它被保证是最完整的。

## 备份 Redis 数据

在开始本部分之前，请确保阅读以下句子：**请务必备份您的数据库**。磁盘可能损坏，云中的实例可能消失等等：没有备份意味着数据被删除的风险很大，可能消失在/dev/null中。

Redis非常友好于数据备份，因为您可以在数据库运行时复制RDB文件：一旦生成，RDB将不会被修改，而且在生成期间会使用临时名称，并且在新快照完成时使用rename（2）原子地重命名为其最终目标。

这意味着在服务器运行时复制RDB文件是完全安全的。这是我们的建议：

* 在您的服务器中创建一个计划任务，每小时在一个目录中创建RDB文件的快照，并在另一个目录中创建每日快照。
* 每次计划任务脚本运行时，请确保调用`find`命令以确保删除过旧的快照：例如，您可以保留最近48小时的每小时快照，并保留一个或两个月的每日快照。请确保使用包含日期和时间信息的名称对快照进行命名。
* 每天至少一次，请确保传输一个RDB快照到您的数据中心之外，或者至少传输到运行Redis实例的物理机之外。

### 备份AOF持久化数据

如果您运行的 Redis 实例仅启用了 AOF 持久化，则仍然可以执行备份操作。
自 Redis 7.0.0 起，AOF 文件被分割为多个文件，这些文件驻留在由 `appenddirname` 配置确定的单个目录中。
在正常操作期间，您只需要将该目录中的文件进行复制/打包即可实现备份。但是，如果在[重写](#log-rewriting)期间执行此操作，可能会导致无效的备份。
为解决此问题，在备份期间必须禁用 AOF 重写：

1. 使用以下命令关闭自动重写：<br/>
   `CONFIG SET` `auto-aof-rewrite-percentage 0`<br/>
   确保在此期间不要手动启动重写（使用 `BGREWRITEAOF`）。
2. 使用以下命令检查当前是否有重写正在进行：<br/>
   `INFO` `persistence`<br/>
   并验证 `aof_rewrite_in_progress` 是否为 0。如果是 1，则需要等待重写完成。
3. 现在可以安全地复制 `appenddirname` 目录下的文件。
4. 完成后，重新启用重写：<br/>
   `CONFIG SET` `auto-aof-rewrite-percentage <prev-value>`

**注意：**如果您想要最小化AOF重写被禁用的时间，您可以在"appenddirname"目录中创建硬链接（在步骤3之后），然后在创建硬链接后重新启用重写（步骤4）。
现在您可以在完成后复制/打包这些硬链接并将它们删除。这是有效的，因为Redis保证它只会在该目录中对文件进行追加操作，或者在必要时完全替换它们，因此内容在任何时刻都应保持一致。


**注意：** 如果您希望在备份过程中处理服务器重启的情况，并确保在重启后不会自动启动重写，您可以将上述第1步也更改为通过`CONFIG REWRITE`来持久化更新的配置。
只需确保在完成后重新启用自动重写（步骤4）并使用另一个`CONFIG REWRITE`来持久化它。

在版本7.0.0之前，可以通过简单地复制AOF文件来备份（就像备份RDB快照一样）。该文件可能缺少最后一部分，但Redis仍然能够加载它（请参阅有关[截断的AOF文件](#what-should-i-do-if-my-aof-gets-truncated)的前几节）。


## 灾难恢复

在Redis的上下文中，灾难恢复基本上与备份相同，再加上能够将这些备份转移到许多不同的外部数据中心。这样，在发生影响Redis运行和生成快照的某些灾难性事件的主要数据中心的情况下，数据将得到保护。

我们将回顾一些成本不太高的最有趣的灾难恢复技术。

* Amazon S3和其他类似的服务是实现灾难恢复系统的好方法。只需以加密形式将每日或每小时的RDB快照传输到S3。您可以使用`gpg -c`（对称加密模式）对数据进行加密。确保将您的密码存储在多个不同的安全位置（例如将副本提供给您组织中最重要的人员）。建议使用多个存储服务以提高数据安全性。
* 使用SCP（SSH的一部分）将您的快照传输到远程服务器。这是一种相当简单和安全的方式：在离您很远的地方获得一个小型VPS，在那里安装ssh，并生成一个没有密码的ssh客户端密钥，然后将其添加到小型VPS的`authorized_keys`文件中。您已经准备好以自动化方式传输备份了。至少在两个不同的提供商处获得两个VPS以获得最佳效果。

重要的是要明白，如果没有正确实施，这个系统很容易失败。至少，确保在传输完成后能够验证文件大小（应该与复制的文件大小相匹配），如果使用 VPS，还可以验证 SHA1 摘要。

如果因某种原因无法进行新鲜备份的传输，您还需要某种独立的警报系统。