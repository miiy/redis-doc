---
title: Redis集群规范
linkTitle: 集群规格
weight: 9
description: >
    详细说明Redis集群规范
aliases:
  - /topics/cluster-spec
---

欢迎来到**Redis集群规范**。在这里，您将找到关于Redis集群算法和设计原理的信息。这个文档是一个正在进行中的工作，因为它与Redis的实际实现不断同步。

## 设计的主要特点和原理

### Redis 集群目标

Redis Cluster 是 Redis 的分布式实现，设计上按以下重要性排序的目标来实现：

* 高性能和线性可扩展性，最多支持1000个节点。没有代理，使用异步复制，不对值执行合并操作。
* 可接受的写入安全性程度：系统尽力保留来自与大多数主节点连接的客户端的所有写入。通常存在可能丢失已确认写入的小窗口。当客户端处于少数分区时，丢失已确认写入的窗口较大。
* 可用性：Redis Cluster能够在大多数主节点可达的分区中存活，并且对于每个不再可达的主节点至少有一个可达的副本。此外，使用*副本迁移*，不再由任何副本复制的主节点将从由多个副本覆盖的主节点接收一个副本。

本文中描述的内容在Redis 3.0或更高版本中实现。

### 已实现的子集

Redis Cluster 在非分布式版本的 Redis 中实现了所有单键命令。对于所有涉及到的密钥在操作中哈希到相同槽的情况，实现了执行复杂的多键操作，如集合并集和交集。

Redis Cluster实现了一种称为**哈希标签**的概念，可以用来强制将某些键存储在同一个哈希槽中。然而，在手动resharding期间，多键操作可能会暂时无法使用，而单键操作始终可用。

Redis集群不支持多个数据库，类似于独立版本的Redis。我们只支持数据库`0`；不允许使用`SELECT`命令。

## Redis集群协议中的客户端和服务器角色

在 Redis 集群中，节点负责持有数据并承担集群的状态，包括将键映射到正确的节点。
集群节点还能够自动发现其他节点，检测异常节点，并在需要时将副本节点提升为主节点，以便在发生故障时继续运行。

为了执行它们的任务，所有的集群节点都使用TCP总线和一个名为**Redis集群总线**的二进制协议进行连接。
每个节点都使用集群总线与集群中的其他节点相连接。节点使用gossip协议来传播有关集群的信息，以便发现新节点，发送ping数据包以确保所有其他节点正常工作，并发送集群消息以信号特定条件。集群总线还用于在集群中传播发布/订阅消息，并在用户请求时协调手动故障转移（手动故障转移是不由Redis集群故障检测器发起，而是由系统管理员直接发起的故障转移）。

由于集群节点无法代理请求，客户端可能会通过重定向错误`-MOVED`和`-ASK`被重定向到其他节点。
理论上，客户端可以自由地向集群中的所有节点发送请求，必要时进行重定向，因此不需要维护集群的状态。
但是，能够缓存键和节点之间的映射关系的客户端可以以合理的方式提高性能。

### 写作安全

Redis 集群在节点间使用异步复制，并带有 "最后故障转移胜出" 的隐式合并功能。这意味着最后选举出的主节点数据集最终会取代其他所有副本。在分区期间，总是存在着可能丢失写操作的时间窗口。然而，对于连接到大多数主节点的客户端和连接到少数主节点的客户端来说，这些时间窗口是非常不同的。

Redis 集群在与大多数主服务器连接的客户端执行写操作时，会尽力保留这些写入操作，而与少数主服务器执行的写操作可能会丢失。
以下是导致在故障期间丢失大多数分区中已确认写入的场景示例：

1. 一个写操作可能会到达主节点，但是当主节点能够回复给客户端时，通过主节点和副本节点之间使用的异步复制机制可能无法将写操作传播到副本节点。如果主节点在写操作未传播到副本节点的情况下停止运行，且主节点无法访问足够长的时间以使其副本之一被提升为主节点，则该写操作将永远丢失。在主节点完全突然发生故障的情况下，这通常很难观察到，因为主节点通常会在差不多的时间内回复给客户端（确认写操作）和副本节点（传播写操作）。然而，这是一个真实存在的故障模式。

2. 一个理论上可能发生的故障模式是丢失写入的情况。

* 由于网络分区，主节点无法访问。
* 其中一个副本接管了故障转移。
* 一段时间后，主节点可能再次可达。
* 当客户端使用过时的路由表时，可能会将数据写入旧的主节点，直到集群将其转换为新主节点的副本为止。

第二种故障模式不太可能发生，因为主节点无法与其他大多数主节点通信足够的时间以进行故障转移，将不再接受写入，并且在修复分区后，写入仍然被拒绝一小段时间以允许其他节点通知有关配置更改的信息。此故障模式还要求客户端的路由表尚未更新。

针对分区的写入具有更大的时间窗口，容易丢失。例如，当Redis集群中的主节点是少数派，并且至少有一个或多个客户端时，在大多数派进行主节点切换时，所有发送给主节点的写入可能都会丢失，从而导致Redis Cluster丢失了非常数量的写入。

具体而言，如果要进行主节点故障转移，那么对于大多数主节点而言，该主节点必须在至少“NODE_TIMEOUT”时间内不可访问，因此如果在该时间点之前修复了分区，那么不会丢失任何写入操作。如果分区持续时间超过了“NODE_TIMEOUT”，那么在该时间点之前在少数派节点上执行的所有写入操作可能会丢失。然而，Redis集群的少数派一旦与大多数节点失去联系超过“NODE_TIMEOUT”的时间，就会拒绝接受写入操作，因此在此之后将不再可用。因此，在此时之后将不再接受或丢失任何写入操作。

### 可用性

Redis集群在分区的少数方面不可用。在分区的多数方面，假设至少有多数个主节点和每个无法访问的主节点的一个副本，集群在`NODE_TIMEOUT`时间加上一些额外的秒数后又变得可用， 这些时间是为了让副本选举并故障转移其主节点所需的时间（故障转移通常在1或2秒内完成）。

这意味着Redis Cluster旨在经受集群中少数节点的故障，但不适用于需要在大规模网络断裂事件中保持可用性的应用程序。

在由N个主节点组成的集群示例中，只要有一个节点被分区隔离，集群的多数方将保持可用；当有两个节点被分区隔离时，集群的多数方以`1-(1/(N*2-1))`的概率保持可用（在第一个节点故障后，我们剩下总共`N*2-1`个节点，并且只有一个没有副本的主节点发生故障的概率是`1/(N*2-1)`）。

例如，在一个有5个节点和每个节点一个副本的集群中，存在`1/(5*2-1) = 11.11%`的概率，即在两个节点从大多数节点中分区后，集群将不再可用。

由于Redis Cluster的一项名为**replicas migration**的功能，集群的可用性在许多真实场景中得到了提高，因为副本会迁移到孤立的主节点（不再有副本的主节点）。因此，在每个成功的故障事件中，集群可以重新配置副本布局，以更好地抵抗下一次故障。

### 性能

在Redis集群中，节点不会将命令代理给负责特定键的正确节点，而是将客户端重定向到负责特定键空间的正确节点。

最终客户端获得集群的最新表示，以及哪个节点服务于哪个键的子集，因此在正常操作过程中，客户端直接联系正确的节点以发送特定的命令。

因为使用了异步复制，节点在写入时不会等待其他节点的确认（除非使用`WAIT`命令显式请求）。

同时，由于多键命令仅限于“附近”键，数据在重新分片时除外，不会在节点之间移动。

在Redis Cluster中，正常操作处理方式与单个Redis实例的情况完全相同。这意味着在具有N个主节点的Redis Cluster中，您可以期望与单个Redis实例相同性能的N倍增长，因为该设计具有线性扩展性。同时，查询通常在单个往返中执行，因为客户端通常保持与节点的持久连接，所以延迟数据也与单个独立Redis节点的情况相同。

非常高的性能和可伸缩性，同时保持弱但合理的数据安全性和可用性，是Redis Cluster的主要目标。

### 为什么要避免合并操作

Redis集群设计避免了多个节点中相同键值对的冲突版本问题，因为在Redis的数据模型中，这并不总是可取的。Redis中的值通常非常大；经常会看到包含数百万个元素的列表或有序集合。此外，数据类型在语义上也比较复杂。传输和合并这些类型的值可能成为主要瓶颈，或者可能需要应用程序逻辑的非平凡参与、额外的内存来存储元数据等。

在这里没有严格的技术限制。CRDT或同步复制的状态机可以模拟类似于Redis的复杂数据类型。然而，这样的系统的实际运行时行为与Redis Cluster不相似。Redis Cluster的设计是为了满足非集群版本Redis的确切使用案例。

## Redis Cluster 主要组成部分概述

### 密钥分发模型

集群的键空间被分为16384个槽位，有效地将集群大小的上限设置为16384个主节点（然而，建议的最大节点数量约为1000个节点）。

在一个集群中，每个主节点处理一部分的16384个哈希槽位。当集群处于稳定状态时，意味着没有正在进行的集群重新配置（即哈希槽位从一个节点移动到另一个节点）。当集群稳定时，一个哈希槽位将由一个节点服务（然而，该节点可以有一个或多个替代节点，在网络分裂或故障的情况下替代原节点，可以用于读操作的扩展，其中读取旧数据是可接受的）。

用于将键映射到哈希槽的基本算法如下（有关此规则的哈希标签异常，请阅读下一段）。

    HASH_SLOT = CRC16(key) mod 16384

CRC16被指定为：

* 名称: XMODEM（也称为ZMODEM或CRC-16/ACORN）
* 宽度: 16位
* 多项式: 1021（实际上是x^16 + x^12 + x^5 + 1）
* 初始化: 0000
* 反转输入字节: 否
* 反转输出CRC: 否
* 输出CRC的异或常数: 0000
* "123456789"的输出结果: 31C3

在CRC16的输出结果中使用了16位中的14位（这就是为什么上述公式中有一个模16384的运算）。

在我们的测试中，CRC16在将不同类型的键均匀分布到16384个插槽中表现得非常好。

**注意**：本文档附录A中提供了CRC16算法的参考实现。

### 哈希标签

计算哈希槽的过程中有一个例外，用于实现**哈希标签**。哈希标签是一种确保多个键分配在同一个哈希槽中的方法。这被用于在Redis Cluster中实现多键操作。

为了实现哈希标签，在某些情况下，将键的哈希槽以稍微不同的方式计算。
如果键包含 "{...}" 格式，只有位于 `{` 和 `}` 之间的子字符串会被用于计算哈希槽。然而，由于可能存在多个 `{` 或 `}`，算法通过以下规则明确定义：



* 如果键包含"{"字符。
* 并且如果在"{"右边有一个"}"字符。
* 并且如果在第一个"{"和第一个"}"之间存在一个或多个字符。

然后，不再对键进行哈希处理，只对第一个 `{` 和随后的第一个 `}` 之间的内容进行哈希处理。

# Title
## Subtitle
### Sub-Subtitle

This is a paragraph of text.

**Bold Text**

*Italic Text*

[Link](www.example.com)

- List Item 1
- List Item 2
- List Item 3

1. Numbered Item 1
2. Numbered Item 2
3. Numbered Item 3

> Blockquote

`Inline code`

```
Code block
```

| Column 1 | Column 2 | Column 3 |
|----------|----------|----------|
|   Item   |   Item   |   Item   |
|   Item   |   Item   |   Item   |

* 两个键`{user1000} .following`和`{user1000} .followers`将哈希到相同的哈希槽，因为只有子串`user1000`将被哈希，以计算哈希槽。
* 对于键`foo{} {bar}`，整个键将被哈希，因为`{`的第一次出现在右边没有字符的`}`后面。
* 对于键`foo{{bar}} zap`，子串`{bar`将被哈希，因为它是第一个`{`和`}`右边的第一个出现之间的子串。
* 对于键`foo{bar} {zap}`，子串`bar`将被哈希，因为算法在第一个有效或无效（不含字节）的`{`和`}`匹配处停止。
* 从算法中可以看出，如果键以`{}`开头，则保证作为整体被哈希。在使用二进制数据作为键名时非常有用。

在Ruby和C语言中，下面是`HASH_SLOT`函数的实现，增加了哈希标签例外处理。


# Ruby示例代码：

```ruby
# 定义一个类
class Person
  attr_accessor :name, :age
  
  def initialize(name, age)
    @name = name
    @age = age
  end
  
  def say_hello
    puts "Hello, my name is #{@name}!"
  end
  
  def say_age
    puts "I'm #{@age} years old."
  end
end

# 创建一个对象
person = Person.new("Alice", 25)

# 调用对象的方法
person.say_hello
person.say_age
```

请注意，上述示例代码是用Ruby编写的。

    def HASH_SLOT(key)
        s = key.index "{"
        if s
            e = key.index "}",s+1
            if e && e != s+1
                key = key[s+1..e-1]
            end
        end
        crc16(key) % 16384
    end

# C示例代码：

```c
#include <stdio.h>

int main() {
    printf("Hello, world!\n");
    return 0;
}
```

    unsigned int HASH_SLOT(char *key, int keylen) {
        int s, e; /* start-end indexes of { and } */

        /* Search the first occurrence of '{'. */
        for (s = 0; s < keylen; s++)
            if (key[s] == '{') break;

        /* No '{' ? Hash the whole key. This is the base case. */
        if (s == keylen) return crc16(key,keylen) & 16383;

        /* '{' found? Check if we have the corresponding '}'. */
        for (e = s+1; e < keylen; e++)
            if (key[e] == '}') break;

        /* No '}' or nothing between {} ? Hash the whole key. */
        if (e == keylen || e == s+1) return crc16(key,keylen) & 16383;

        /* If we are here there is both a { and a } on its right. Hash
         * what is in the middle between { and }. */
        return crc16(key+s+1,e-s-1) & 16383;
    }

### 集群节点属性

每个节点在群集中具有唯一的名称。该节点名称是一个160位随机数的十六进制表示，第一次启动节点时获得（通常使用/dev/urandom）。
节点将其ID保存在节点配置文件中，并永久使用相同的ID，或者至少在节点配置文件未被系统管理员删除或通过“CLUSTER RESET”命令请求执行*硬重置*之前。

节点ID用于在整个集群中标识每个节点。
一个给定的节点有可能改变其IP地址，而无需改变节点ID。
该集群也能够通过运行在集群总线上的流言协议来检测IP/端口的变化并重新配置。

节点ID不仅是与每个节点相关联的唯一信息，而且始终保持全局一致。每个节点还有以下一组相关信息。一些信息是关于此特定节点的集群配置详细信息，最终在整个集群中保持一致。而其他一些信息，例如上次节点被ping的时间，则只在每个节点本地有效。

每个节点在群集中维护以下有关其它节点的信息：节点的 ID、IP 和端口，一组标志，如果被标记为“replica”则为节点的主节点，节点最后一次收到 ping 信号和 pong 信号的时间，节点当前的*配置纪元*（稍后在本规范中解释），连接状态以及服务的哈希槽集合。

在`CLUSTER NODES`文档中详细描述了所有节点字段的[解释](https://redis.io/commands/cluster-nodes)。

`CLUSTER NODES` 命令可以发送到集群中的任何节点，并根据查询节点本地视图提供集群的状态和每个节点的信息。

以下是发送给小型三节点集群中的主节点的`CLUSTER NODES`命令的示例输出。

    $ redis-cli cluster nodes
    d1861060fe6a534d42d8a19aeb36600e18785e04 127.0.0.1:6379 myself - 0 1318428930 1 connected 0-1364
    3886e65cc906bfd9b1f7e7bde468726a052d1dae 127.0.0.1:6380 master - 1318428930 1318428931 2 connected 1365-2729
    d289c575dcbc4bdd2931585fd4339089e461a27d 127.0.0.1:6381 master - 1318428931 1318428931 3 connected 2730-4095

在上面的列表中，不同的字段按顺序排列：节点ID、地址:端口、标志、上次发送的ping、上次接收的pong、配置时代、连接状态、插槽。关于上述字段的详细信息将在我们讨论Redis Cluster的特定部分时介绍。

### 集群巴士

每个Redis Cluster节点都有一个额外的TCP端口，用于接收来自其他Redis Cluster节点的连接。此端口将通过将数据端口加上10000来派生，或者可以通过cluster-port配置指定。


如果Redis节点在6379端口上监听客户端连接，并且在redis.conf中没有添加cluster-port参数，则Cluster总线端口16379将被打开。

如果Redis节点在端口6379上监听客户端连接，
并在redis.conf中设置cluster-port 20000，
那么集群总线端口20000将被打开。

节点到节点的通信完全使用集群总线和集群总线协议进行，该协议是一个由不同类型和大小的帧组成的二进制协议。集群总线二进制协议并没有公开文档，因为它不适用于外部软件设备使用该协议与Redis集群节点交互。但您可以通过阅读Redis集群源代码中的`cluster.h`和`cluster.c`文件来获取有关集群总线协议的更多详细信息。

### 集群拓扑结构

Redis集群是一个全网状结构，每个节点通过TCP连接与其他节点相互连接。

在一个包含N个节点的集群中，每个节点都有N-1个出站TCP连接和N-1个入站连接。

这些TCP连接始终保持活动状态，并且不是根据需求创建的。
当节点在集群总线上期望在ping的应答中得到pong时，在等待足够长的时间将节点标记为不可达之前，它将尝试通过从头开始重新连接来刷新与节点的连接。

在Redis集群中，节点之间形成一个完全网状结构，**节点使用一种八卦协议和配置更新机制来避免在正常情况下节点之间交换过多的消息**，因此消息交换的数量不会呈指数增长。

### 节点握手

节点始终在群集总线端口上接受连接，即使在收到ping时仍会回复，即使该ping节点不受信任。
然而，如果发送节点不被视为群集的一部分，则接收节点将丢弃所有其他数据包。

一个节点只会以两种方式接受另一个节点作为群集的一部分：

1. 静态配置：在节点的配置文件中明确指定群集中的其他节点。
2. 动态加入：节点通过与集群通信，请求加入群集，等待其他节点的认可。

* 如果一个节点以`MEET`信息（`CLUSTER MEET`命令）介绍自己。meet信息和`PING`信息完全相同，但会强制接收方将该节点纳入集群中。只有当系统管理员通过以下命令请求时，节点才会将`MEET`消息发送给其他节点：

    CLUSTER MEET ip port

* 如果一个已经受信任的节点向另一个节点传递信息，该节点也将将其注册为集群的一部分。因此，如果A知道B，B知道C，最终B将向A发送有关C的传闻信息。当这种情况发生时，A将将C注册为网络的一部分，并尝试与C建立连接。

这意味着只要我们在任何连通图中加入节点，它们最终会自动形成一个完全连通的图。这意味着集群能够自动发现其他节点，但前提是存在系统管理员所强制的信任关系。

此机制使集群更加强大，但在IP地址更改或其他与网络相关的事件发生后，防止不同的Redis集群意外混合。

## 重定向与重分片

### MOVED 重定向

一个Redis客户端可以自由地向集群中的每个节点发送查询，包括副本节点。节点将分析查询，并在查询可接受的情况下（即，查询中只提到了一个键，或者提到的多个键都属于同一个哈希槽），它将查找负责该键或键所属的哈希槽的节点。

如果哈希槽由节点提供服务，则查询将简单处理，否则节点将检查其内部哈希槽到节点的映射，并以MOVED错误回复给客户端，如以下示例所示：

    GET x
    -MOVED 3999 127.0.0.1:6381

错误包括键的哈希槽（3999）以及可以执行查询的实例的端点：端口。
客户端需要重新向指定节点的端点地址和端口重新发起查询。
端点可以是IP地址、主机名，也可以为空（例如“-MOVED 3999 :6380”）。
空的端点表示服务器节点具有未知的端点，客户端应该将下一个请求发送到与当前请求相同的端点，但使用提供的端口。

注意，即使客户端在重新发出查询之前等待了很长时间，同时集群配置发生了变化，目标节点将再次回复一个MOVED错误，如果哈希槽3999现在由另一个节点提供服务。如果联系的节点没有更新的信息，同样会发生这种情况。

尽管从集群节点的视角来看，节点是通过ID来识别的，但我们试图简化与客户端的接口，只暴露哈希槽与由endpoint:port对标识的Redis节点之间的映射关系。

客户端不需要，但应该尝试记住哈希槽3999由127.0.0.1:6381提供服务。这样一旦有新的命令需要发出，它可以计算目标键的哈希槽，并有更大的机会选择正确的节点。

另一种选择是在接收到MOVED重定向时，使用`CLUSTER SHARDS`或弃用的`CLUSTER SLOTS`命令刷新整个客户端集群布局。遇到重定向时，很可能重新配置了多个槽位而不仅仅是一个，所以尽快更新客户端配置通常是最佳策略。

请注意，当集群处于稳定状态（配置没有进行任何改变）时，最终所有的客户端都将获得一个哈希槽位到节点的映射，使得集群高效运行，客户端可以直接与正确的节点通信，无需重定向、代理或其他单点故障实体的介入。

一个客户端还必须能够处理后面在本文档中描述的-ASK重定向，否则它就不是一个完整的Redis集群客户端。

### 实时重新配置

Redis Cluster支持在群集运行时添加和移除节点的功能。添加或移除节点被抽象为相同的操作：将一个哈希槽从一个节点移动到另一个节点。这意味着可以使用相同的基本机制来重新平衡群集、添加或移除节点等操作。

* 要将新节点添加到集群中，将一个空节点添加到集群，并将一些哈希槽从现有节点移动到新节点。
* 要将节点从集群中移除，将分配给该节点的哈希槽移动到其他现有节点。
* 要重新平衡集群，将给定的一组哈希槽在节点之间移动。

实现的核心是能够在哈希槽之间移动。从实际角度来看，哈希槽只是一组键，所以Redis集群在重新分片期间的真正操作就是将键从一个实例移动到另一个实例。移动一个哈希槽意味着移动所有散列到该哈希槽的键。

为了理解这个工作原理，我们需要展示 `CLUSTER` 子命令，它们用于在 Redis 集群节点中操作槽位转换表。

以下子命令可用（还有其他不适用于此情况的命令）：

* `CLUSTER ADDSLOTS` slot1 [slot2] ... [slotN]
* `CLUSTER DELSLOTS` slot1 [slot2] ... [slotN]
* `CLUSTER ADDSLOTSRANGE` start-slot1 end-slot1 [start-slot2 end-slot2] ... [start-slotN end-slotN]
* `CLUSTER DELSLOTSRANGE` start-slot1 end-slot1 [start-slot2 end-slot2] ... [start-slotN end-slotN]
* `CLUSTER SETSLOT` slot 节点 node
* `CLUSTER SETSLOT` slot 迁移中 node
* `CLUSTER SETSLOT` slot 导入中 node

第一个命令`ADDSLOTS`、`DELSLOTS`、`ADDSLOTSRANGE`和`DELSLOTSRANGE`，仅用于将插槽分配（或删除）给Redis节点。分配插槽意味着告诉给定的主节点负责存储和提供指定的哈希插槽的内容。

在分配散列槽之后，它们将通过集群中的八卦协议进行传播，
如稍后在*配置传播*部分中所指定的那样。

`ADDSLOTS` 和 `ADDSLOTSRANGE` 命令通常在从头创建新集群时使用，以便将每个主节点分配到所有可用的 16384 个哈希槽的子集中。

`DELSLOTS` 和 `DELSLOTSRANGE` 主要用于手动修改集群配置或调试任务：实际上很少使用。

`SETSLOT`子命令用于为特定的节点ID分配一个插槽，如果使用`SETSLOT <slot> NODE`形式。否则，插槽可以在两个特殊状态`MIGRATING`和`IMPORTING`中设置。这两个特殊状态用于将哈希插槽从一个节点迁移到另一个节点。

* 当一个槽位被设置为MIGRATING时，节点将接受所有关于该哈希槽的查询，但仅当所查询的键存在时，否则该查询将通过"-ASK"重定向转发给迁移目标节点。 
* 当一个槽位被设置为IMPORTING时，节点将接受所有关于该哈希槽的查询，但仅当请求之前有一个"ASKING"命令时。如果客户端没有给出"ASKING"命令，查询将被重定向到真正的哈希槽所有者，就像通常情况下会发生的"-MOVED"重定向错误一样。

让我们通过哈希槽迁移的例子来更清楚地说明。
假设我们有两个 Redis 主节点，分别称为 A 和 B。
我们希望将哈希槽 8 从 A 迁移到 B，因此我们发出如下命令：

* We send B: CLUSTER SETSLOT 8 IMPORTING A
* We send A: CLUSTER SETSLOT 8 MIGRATING B

所有其他节点在每次被查询到属于哈希槽8的键时，将继续指向节点“A”，因此发生的情况是：

* 所有关于现有键的查询都由“A”处理。
* 所有关于“A”中不存在的键的查询都由“B”处理，因为“A”会将客户重定向到“B”。

这样我们就不再在"A"中创建新的键值了。
同时在reshardings和Redis Cluster配置中使用的`redis-cli`将会将哈希槽8中的现有键值从A迁移到B。
这是通过以下命令来执行的：

    CLUSTER GETKEYSINSLOT slot count

上述命令将返回指定哈希槽中的“count”个键。
对于返回的键，`redis-cli`将向节点"A"发送一个`MIGRATE`命令，该命令将以原子方式将指定的键从A迁移到B（两个实例在迁移键的时间内（通常很短）都被锁定，以防止竞态条件）。这是`MIGRATE`的工作原理：

    MIGRATE target_host target_port "" target_database id timeout KEYS key1 key2 ...

`MIGRATE`将连接到目标实例，发送一个序列化版本的键，一旦接收到OK代码，就会删除其自己数据集中的旧键。对于外部客户端来说，在任意给定的时间点，一个键要么存在于A中，要么存在于B中。

在Redis Cluster中，除了0之外，不需要指定其他数据库，但是`MIGRATE`是一个通用命令，可用于不涉及Redis Cluster的其他任务。
即使移动包含长列表等复杂键的情况，`MIGRATE`也经过优化，以尽可能快速，但是在Redis Cluster中，如果存在大键，则重新配置集群在应用程序对数据库使用有延迟限制时被认为是不明智的过程。

迁移过程结束后，将向参与迁移的两个节点发送`SETSLOT <slot> NODE <node-id>`命令，以将槽设置回正常状态。通常还会将同样的命令发送给其他所有节点，以避免等待新配置在整个集群中自然传播。

### ASK 重定向

在前一节中，我们简要讨论了ASK重定向。为什么我们不能简单地使用MOVED重定向呢？因为MOVED表示哈希槽被永久地由另一个节点提供，并且下一个查询应该尝试指定的节点。而ASK只意味着将下一个查询发送到指定的节点。

这是必需的，因为关于哈希槽 8 的下一个查询可能涉及的键仍然在 A 中，因此我们希望客户端始终首先尝试 A，然后再尝试 B（如果需要的话）。由于这仅在可用的 16384 个哈希槽中发生一次，对集群的性能影响是可以接受的。

我们需要强制客户端的行为，以确保在尝试节点A之后仅尝试节点B，当客户端在发送查询之前发送ASKING命令时，只有设置为IMPORTING的插槽节点B 才会接受查询。

基本上，ASKING命令在客户端上设置了一个一次性标志，强制一个节点为一个IMPORTING插槽提供查询。

从客户端的角度来看，ASK重定向的完整语义如下所示：

- 如果收到ASK重定向，请仅发送被重定向到指定节点的查询，但继续发送后续查询到旧节点。
- 用ASKING命令开始重定向的查询。
- 还不要更新本地客户端表，将哈希槽8映射到B。

一旦哈希槽8的迁移完成，A将发送MOVED消息，客户端可以永久地将哈希槽8映射到新的端点和端口对。请注意，如果一个有错误的客户端提前进行了映射，这不是一个问题，因为在发出查询之前它不会发送ASKING命令，所以B将使用MOVED重定向错误将客户端重定向到A。

在“CLUSTER SETSLOT”命令文档中，插槽迁移的解释基本相同，只是措辞不同（为了文件的冗余性）。

### 客户端连接和重定向处理

为了提高效率，Redis 集群客户端会维护一个当前槽位配置的映射。
然而，该配置不要求是最新的。
当错误地访问节点导致重定向时，客户端可以相应地更新其内部槽位映射。

客户通常在两种不同的情况下需要获取插槽完整列表和映射节点地址:

* 在启动时，用于填充初始插槽配置
* 当客户端收到`MOVED`重定向时

请注意，客户端可以通过更新表中的移动槽位来处理`MOVED`重定向；但通常这种方法效率不高，因为经常会一次性修改多个槽位的配置。例如，如果一个副本被晋升为主节点，旧主节点所服务的所有槽位都将被重新映射。最简单的方法是通过从头开始获取槽位到节点的完整映射来响应`MOVED`重定向。

客户端可以发出`CLUSTER SLOTS`命令，以检索一组槽范围和关联的主节点和副本节点，这些节点用于服务指定的范围。

以下是`CLUSTER SLOTS`的输出示例：

```
1) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 7000
      3) "4b8357a53cce4e1b643d663182b6fdb4277ef78c"
   4) 1) "127.0.0.1"
      2) (integer) 7001
      3) "68dbeed200e7f2e7f8ca46f9a0e10d7e357683f2"
2) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 7003
      3) "6d8f366890955e1be1ef089d15ef500beeeb7f08"
   4) 1) "127.0.0.1"
      2) (integer) 7004
      3) "6943e0f66eeb5fa6f2bff001ed836a36f0d43a4d"
3) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 7002
      3) "19dcad9657987180f4262ac325f989b9d3dcb22d"
   4) 1) "127.0.0.1"
      2) (integer) 7005
      3) "0f8c69db274e6f1a1e4f8a504378e4aa4d52da28"
```

```
127.0.0.1:7000> cluster slots
1) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 7001
   4) 1) "127.0.0.1"
      2) (integer) 7004
2) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 7000
   4) 1) "127.0.0.1"
      2) (integer) 7003
3) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 7002
   4) 1) "127.0.0.1"
      2) (integer) 7005
```

返回数组的每个元素的前两个子元素是范围的起始和结束槽位。额外的元素表示地址-端口对。第一个地址-端口对是负责槽位的主节点，其他额外的地址-端口对是负责同一槽位的副本节点。副本节点只在没有错误情况下列出（即，当它们的FAIL标志未设置时）。

上述输出中的第一个元素显示，从5461到10922（包括起始和结束）的插槽由127.0.0.1:7001提供服务，可以通过与127.0.0.1:7004的副本建立联系来扩展只读负载。

如果集群配置错误，`CLUSTER SLOTS` 不能保证返回完全覆盖 16384 个 slot 的范围，因此客户端应该初始化 slots 配置映射，填充目标节点为 NULL 对象，并在用户尝试执行属于未分配 slot 的键的命令时报错。

在向调用者返回错误之前，当发现一个槽位未分配时，客户端应该尝试重新获取槽位的配置，以检查集群是否已正确配置。

### 多键操作

使用哈希标签，客户端可以自由使用多键操作。
例如，以下操作是有效的：

    MSET {user:1000}.name Angela {user:1000}.surname White

当哈希槽正在进行resharding时，多键操作可能变得不可用。

更具体地说，即使在resharding过程中，针对所有已存在且仍然哈希到同一槽（无论是源节点还是目标节点）的键的多键操作仍然可用。

对于不存在或在resharding过程中分割在源节点和目标节点之间的键的操作，将生成一个"-TRYAGAIN"错误。
客户端可以在一段时间后尝试操作，或者报告错误。

一旦指定哈希槽的迁移完成，该哈希槽上的所有多键操作将再次可用。

### 使用副本节点进行读取扩展

通常情况下，副本节点会将客户端重定向到与给定命令所涉及的哈希槽对应的权威主节点，然而客户端可以通过使用"READONLY"命令来扩展读操作的规模。

`READONLY` 声明一个 Redis 集群副本节点，该客户端允许读取可能过期的数据，并且不对运行写入查询感兴趣。

在只读模式下，如果操作涉及到由副本的主节点不提供服务的键，则群集只会向客户端发送重定向。这可能发生的原因有：

1. 客户端发送了一个关于哈希槽位在此副本的主节点上从未被处理的命令。
2. 集群已重新配置（例如重新分片），该副本不再能为给定哈希槽位提供服务。

当发生这种情况时，客户端应根据前面的部分解释更新其哈希槽映射。

连接的只读状态可以通过`READWRITE`命令清除。

## 容错性

### 心跳和八卦信息

Redis集群节点不断交换ping和pong数据包。这两种数据包具有相同的结构，并且都携带重要的配置信息。唯一的实际区别是消息类型字段。我们将ping和pong数据包的总和称为*心跳数据包*。

通常节点发送ping数据包，这将触发接收方回复pong数据包。然而，这不一定真实。节点也有可能只发送pong数据包，以向其他节点发送关于其配置的信息，而无需触发回复。例如，为了尽快广播新配置，这是很有用的。

通常，一个节点每秒会随机ping若干个节点，这样每个节点发送（和接收）的ping数据包数量是一个恒定的量，而与集群中节点的数量无关。

然而，每个节点都确保对于超过`NODE_TIMEOUT`时间一半的未发送ping或未收到pong的其他节点进行ping。在`NODE_TIMEOUT`时间结束之前，节点还尝试重新连接到另一个节点的TCP链接，以确保节点不被认为是无法访问的，仅仅因为当前的TCP连接存在问题。

全球交换的消息数量可能相当可观，如果将`NODE_TIMEOUT`设置为小值并且节点的数量（N）非常大，因为每个节点都会尝试在每半个`NODE_TIMEOUT`时间内对每个没有最新信息的其他节点进行ping操作。

例如，在一个具有100个节点且节点超时设置为60秒的集群中，每个节点将尝试每30秒发送99个ping，总ping数为每秒3.3次。乘以100个节点，这意味着总集群每秒发送330个ping。

有办法降低消息数量，但目前报告中Redis Cluster故障检测所使用的带宽没有出现任何问题，因此现在使用的是明显和直接的设计。请注意，即使在上述示例中，每秒交换的330个数据包均均匀地分配给100个不同节点，因此每个节点接收的流量是可接受的。

### 心跳包内容

Ping和Pong数据包包含一个对所有类型的数据包（例如用于请求故障转移投票的数据包）都通用的头部，以及一个特殊的八卦部分，该部分只属于Ping和Pong数据包。

常见的标题包含以下信息：

* Node ID，一个160位的伪随机字符串，由Redis集群节点第一次创建时分配，并在其整个生命周期中保持不变。
* 发送节点的`currentEpoch`和`configEpoch`字段，用于挂载Redis集群使用的分布式算法（在后续章节中详细解释）。如果节点是副本，则`configEpoch`是其主节点的最后已知`configEpoch`。
* 节点标志，指示节点是副本、主节点还是其他单比特节点信息。
* 发送节点服务的哈希槽位的位图，或者如果节点是副本，则为其主节点服务的槽位的位图。
* 发送者TCP基本端口，该端口由Redis用于接受客户端命令。
* 集群端口，该端口由Redis用于节点间通信。
* 发送者视角下的集群状态（down或ok）。
* 如果发送节点是副本，则为其主节点的主节点ID。

Ping和pong数据包还包含一个八卦部分。这个部分向接收方提供了发送方节点对群集中其他节点的看法。八卦部分仅包含发送方已知节点集合中的几个随机节点的信息。群集中在八卦部分中提到的节点数量与群集大小成比例。

对于在流言中添加的每个节点，报告以下字段：



以下是所有数据的附注格式:
* 节点ID.
* 节点的IP和端口.
* 节点标志.

**八卦**（Gossip）章节允许接收节点从发送节点的角度获取有关其他节点状态的信息。这对于故障检测以及发现集群中的其他节点都很有用。

### 失败检测

Redis集群故障检测用于识别主节点或副本节点无法通过大多数节点访问，并通过将副本升级为主节点来作出响应。当无法进行副本升级时，集群会进入错误状态，停止接收客户端的查询请求。

如前所述，每个节点都会使用与其他已知节点关联的标志列表。用于故障检测的有两个标志，分别称为`PFAIL`和`FAIL`。`PFAIL`意味着*可能故障*，是一种未确认的故障类型。`FAIL`表示节点正在失败，并且此情况已在固定时间内得到大多数主节点的确认。

**PFAIL标志：**

当一个节点在超过`NODE_TIMEOUT`的时间内无法访问时，将使用`PFAIL`标记另一个节点。无论节点类型是主节点还是副本节点，都可以将另一个节点标记为`PFAIL`。

对于 Redis 集群节点的不可达概念是，我们有一个**活动的 ping**（我们发送但尚未收到回复的 ping），其挂起时间超过了 `NODE_TIMEOUT`。为了使该机制正常工作，`NODE_TIMEOUT` 必须与网络往返时间相比较长。为了在正常操作期间提供可靠性，节点将在半个 `NODE_TIMEOUT` 已过去但尚未收到 ping 的回复时尝试重新连接到集群中的其他节点。该机制确保连接保持活动，因此通常不会在节点之间出现错误的连接导致错误的故障报告。

**失败标志：**

`PFAIL` 标志仅仅是每个节点对其他节点的本地信息，但它还不足以触发副本提升。要考虑节点宕机，`PFAIL` 条件需要升级为 `FAIL` 条件。

根据本文档中的节点心跳部分的提要，每个节点向包括一些随机已知节点状态在内的每个其他节点发送流言消息。每个节点最终会接收到每个其他节点的一组节点标志。通过这种方式，每个节点都有一种机制来通知其他节点它们检测到的故障条件。

当满足以下条件集时，将"PFAIL"条件提升为"FAIL"条件：

* 有一个节点，我们称之为A，有另一个被标记为`PFAIL`的节点B。
* 节点A通过短信广播机制，从集群中大部分主节点的角度收集到了有关节点B状态的信息。
* 大部分主节点在`NODE_TIMEOUT * FAIL_REPORT_VALIDITY_MULT`时间内发出了`PFAIL`或`FAIL`的信号。（在当前实现中，有效因子设置为2，因此这只是`NODE_TIMEOUT`时间的两倍）。

如果以上所有条件都为真，则节点A将：

- 将节点标记为“FAIL”。
- 发送一个“FAIL”消息（与心跳消息中的“FAIL”条件相对）给所有可达节点。

`FAIL`消息会强制每个接收节点将节点标记为`FAIL`状态，无论它们是否已经将节点标记为`PFAIL`状态。

请注意，* FAIL 标志通常是单向的 *。也就是说，一个节点可以从 `PFAIL` 转变为 `FAIL`，但是 `FAIL` 标志只能在以下情况下被清除：

* 节点已经可达且是一个副本。在这种情况下，"FAIL"标志可以清除，因为副本不会发生故障转移。
* 节点已经可达且是一个不提供任何槽位的主节点。在这种情况下，"FAIL"标志可以清除，因为没有槽位的主节点实际上不参与集群，并且在等待配置以加入集群。
* 节点已经可达且是一个主节点，但是在没有任何可检测到的副本晋升的情况下已经过去了很长时间（N倍的"NODE_TIMEOUT"）。在这种情况下，最好让它重新加入集群并继续运行。

有一点值得注意的是，尽管“PFAIL” -> “FAIL”的转换使用了一种形式的一致性，但使用的一致性是薄弱的：

1. 节点在一段时间内收集其他节点的视图，所以即使大多数主节点需要“同意”，实际上这只是我们在不同时间从不同节点收集到的状态，我们不确定，也不需要在给定的时刻大多数主节点都同意。然而，我们会丢弃旧的故障报告，所以故障是由大多数主节点在一段时间窗口内发出的信号。
2. 尽管每个发现“FAIL”条件的节点都会使用“FAIL”消息将该条件强加给集群中的其他节点，但无法确保消息会到达所有节点。例如，一个节点可能会检测到“FAIL”条件，但由于分区无法与任何其他节点建立联系。

然而，Redis集群故障检测有一个存活性要求：最终所有节点应该对给定节点的状态达成一致。这里有两种可能的情况是由于脑裂条件引起的。一种情况是一些节点的少数认为节点处于“FAIL”状态，另一种情况是少数节点认为节点不处于“FAIL”状态。无论哪种情况，最终集群都会对给定节点的状态有一个统一的视图：

**第1种案例**：如果大多数主节点将一个节点标记为`FAIL`，由于故障检测和其产生的*链式效应*，其他所有节点最终也会将主节点标记为`FAIL`，因为在指定的时间窗口内会报告足够多的故障。

**案例2**：当只有少数主节点将节点标记为“FAIL”时，副本提升将不会发生（因为它使用了一种更正式的算法，以确保每个人最终都会了解到提升），并且每个节点将根据上述的“FAIL”状态清除规则清除“FAIL”状态（即在经过N次“NODE_TIMEOUT”的时间后不再进行提升）。

**`FAIL`标志仅用作触发运行算法中的安全部分**来进行副本升级。理论上，副本可以独立操作，并在其主服务器无法连接时开始副本升级，并等待大多数主服务器拒绝提供确认，如果实际上大多数主服务器是可达的。然而，`PFAIL -> FAIL`状态的增加复杂性，弱一致性以及`FAIL`消息强制在集群中可达部分中以最短的时间传播状态，具有实际优势。由于这些机制，如果集群处于错误状态，通常所有节点将在大约同一时间停止接受写入。从使用Redis Cluster的应用程序的角度来看，这是一个理想的特性。此外，通过本地问题无法连接到其主服务器的副本发起的错误选举尝试（否则大多数其他主服务器可以连接到主服务器）将被避免。

## 配置处理、传播和故障转移

### 当前时期的集群

Redis Cluster 使用了类似于 Raft 算法的 "term" 概念。在 Redis Cluster 中，"term" 被称为 "epoch"，用于对事件进行增量版本控制。当多个节点提供冲突信息时，另一个节点可以了解哪个状态是最新的。

`currentEpoch` 是一个64位无符号数。

在创建Redis集群节点时，包括副本节点和主节点，将`currentEpoch`设置为0。

每当从另一个节点接收到数据包时，如果发送者的时代（集群总线消息头的一部分）大于本地节点的时代，则更新`currentEpoch`为发送者的时代。

由于这些语义，最终所有节点将同意集群中最大的`currentEpoch`。

这些信息在集群状态发生变化时使用，当一个节点寻求达成一致以执行某些操作时。

目前，这种情况只发生在副本晋升过程中，如下一部分所描述的。基本上，时代是群集的逻辑时钟，并指示给定的信息优于较小时代的信息。

### 配置纪元

每个主节点都会在 ping 和 pong 数据包中广告其 `configEpoch`，以及一个位图表示它所服务的槽集合。

当创建新节点时，主节点中的`configEpoch`设置为零。

在副本选举期间创建了一个新的`configEpoch`。尝试取代失败的主节点的副本会增加它们的epoch，并尝试从大多数主节点获取授权。当副本获得授权时，会创建一个新的唯一的`configEpoch`，然后将副本转换为使用新的`configEpoch`的主节点。

正如后面的部分所解释的，`configEpoch`在不同节点声明不同配置时，有助于解决冲突（这种情况可能是由于网络分区和节点故障引起的）。

复制节点在 ping 和 pong 数据包中也会广播 `configEpoch` 字段，但是对于复制节点而言，该字段表示最后一次它们交换数据包时主节点的 `configEpoch`。这使得其他实例能够检测到复制节点存在旧的配置需要更新（主节点将不会给予具有旧配置的复制节点选票）。

每当某个已知节点的`configEpoch`发生变化时，所有收到该信息的节点都会将其永久存储在nodes.conf文件中。`currentEpoch`的值也是如此。在节点继续操作之前，更新这两个变量时，它们被保证保存并进行`fsync`到磁盘上。

使用简单的算法在故障转移期间生成的 `configEpoch` 值保证是新的、递增的和唯一的。

### 选举和晋升副本

复制品选举和提升由复制节点处理，借助主节点为复制品投票以提升。
当至少一个具备成为主节点先决条件的复制品从其角度上看到主节点处于“FAIL”状态时，会发生复制品选举。

为了将副本升级为主节点，需要启动一次选举并获胜。只有在主节点处于“FAIL”状态时，所有属于同一个主节点的副本才能开始一次选举，但只有一个副本将赢得选举并晋升为主节点。

当满足以下条件时，复制开始选举：



* 复制品的主节点处于`FAIL`状态。
* 主节点正在服务一个非零数量的槽位。
* 为了确保提升的复制品的数据相对较新鲜，副本的复制连接与主节点的断开时间不超过给定的时间。这个时间可由用户配置。

为了被选举，副本的第一步是增加其`currentEpoch`计数器，并向主实例请求投票。

复制品通过向群集的每个主节点广播一个 `FAILOVER_AUTH_REQUEST` 数据包来请求投票。然后等待最长时间为 `NODE_TIMEOUT` 的两倍的时间，等待回复 (但至少为2秒)。

一旦主服务器投票支持一个指定的副本，并通过`FAILOVER_AUTH_ACK`积极响应，则在`节点超时*2`的时间内，它将不能再为同一主服务器的其他副本投票。在此期间，它将无法回复同一主服务器的其他授权请求。这不是为了保证安全性，但有助于防止多个副本在大致相同的时间内（即使`configEpoch`不同）被选中，通常这是不希望的。

一个副本会丢弃任何在投票请求发送时其纪元小于`currentEpoch`的`AUTH_ACK`回复，以确保它不会计算以前选举中的选票。

一旦副本从大多数主节点收到ACK，它就赢得选举。
否则，如果在两倍 `NODE_TIMEOUT` 的时间内没有达到多数，选举将被中止，并在 `NODE_TIMEOUT * 4` 之后再次尝试新的选举（至少为4秒）。

### 复制等级

当主服务器处于 `FAIL` 状态时，备份服务器会在一段时间后尝试进行选举。该延迟的计算方式如下：

    DELAY = 500 milliseconds + random delay between 0 and 500 milliseconds +
            REPLICA_RANK * 1000 milliseconds.

固定延迟确保我们等待“FAIL”状态在集群中传播，否则副本可能在主节点仍然不知道“FAIL”状态并拒绝授予选票时尝试竞选。

随机延迟用于使副本的时钟不同步，从而使它们不太可能同时发起选举。

`REPLICA_RANK` 是指该副本根据其从主服务器处理的复制数据量而确定的等级。
当主服务器发生故障时，副本之间会交换消息以确定（尽最大努力）等级：
复制偏移量最新的副本排在第0位，第二新的排在第1位，以此类推。
通过这种方式，最新的副本尝试在其他副本之前当选。

等级顺序没有严格执行；如果高级别副本未能当选，其他副本将很快尝试。

一旦复制品赢得选举，它将获得一个新的唯一且增量的`configEpoch`，该值高于任何其他现有主节点。它开始在ping和pong数据包中广播自己作为主节点，提供一组已服务的槽位，其中`configEpoch`将胜过过去的值。

为了加快其他节点的重新配置速度，将pong数据包广播到集群的所有节点。当前无法访问的节点将在它们接收到来自另一个节点的ping或pong数据包时重新配置，或者如果检测到通过心跳数据包发布的信息已过时，则会从另一个节点接收到一个“UPDATE”数据包。

其他节点将检测到有一个新的主节点在提供与旧的主节点相同的槽位服务，并且它的`configEpoch`更大，它们将升级它们的配置。旧的主节点的副本（或者如果它重新加入集群的故障转移主节点）将不仅升级配置，还将重新配置以从新的主节点复制数据。后续章节将解释重新加入集群的节点如何进行配置。

### 大师回复复制品投票请求

在前一节中，我们讨论了副本如何竞选。本节将解释在请求为特定副本投票的主节点的视角下会发生什么情况。

大师收到来自副本的`FAILOVER_AUTH_REQUEST`请求形式的投票请求。

要授予投票权，需要满足以下条件：

1. 在给定的时期内，主节点只投票一次，并且拒绝对较早的时期进行投票：每个主节点都有一个lastVoteEpoch字段，只要授权请求包中的currentEpoch不大于lastVoteEpoch，主节点就会拒绝再次投票。当主节点对投票请求做出肯定回复时，lastVoteEpoch相应更新，并安全地存储在磁盘上。
2. 主节点仅为标记为“FAIL”的副本进行投票。
3. 当授权请求中的currentEpoch小于主节点的currentEpoch时，这些请求会被忽略。因此，主节点的回复中的currentEpoch将始终与授权请求相同。如果同一个副本再次请求投票，并递增currentEpoch，可以确保从主节点得到的旧延迟回复不会被接受用于新的投票。

由于未使用规则3引起的问题示例：

主 `currentEpoch` 为 5，lastVoteEpoch 为 1（这可能发生在几次失败的选举之后）

* 复制品 `currentEpoch` 为3。
* 复制品尝试使用时期4（3+1）进行选举，主节点以`currentEpoch`为5的ok进行回复，但是回复被延迟。
* 复制品稍后会再次尝试进行选举，使用时期5（4+1），延迟的回复到达复制品，并被接受为有效。

4. 如果已经有一个主节点的副本获得了选举投票，在经过 `NODE_TIMEOUT * 2` 的时间之前，其他副本不会再投票给相同的主节点副本。严格来说，这并非必需，因为不可能在同一时期内有两个副本赢得选举。然而，在实际中，这可以确保当一个副本被选举时，它有足够的时间通知其他副本，并避免另一个副本赢得新的选举，从而进行不必要的第二次故障切换。

5. 主节点不会通过任何方式选择最佳副本。如果副本的主节点处于“FAIL”状态，并且该主节点在当前任期内没有投票，则会授予积极的投票。最佳副本是最有可能在其他副本之前启动选举并赢得选举的副本，因为由于在上一节中所解释的*较高级别*，它通常能够更早地开始投票过程。

6. 当一个主节点拒绝为给定的副本投票时，没有负面反应，该请求被简单地忽略。

7. 主节点不会为发送具有小于主表中由副本声明的插槽的任何`configEpoch`的副本投票。请记住，副本发送其主节点的`configEpoch`和为其主节点服务的插槽的位图。这意味着请求投票的副本必须具有对其希望故障转移的插槽进行配置的新版本或相等的配置，以使其大于或等于授予投票的主节点的配置。

### 配置时期在分区期间的实际用例

该部分示例了如何使用时代概念来使副本晋升过程更加抗分区。

* 主节点无法无限期访问。主节点有三个副本A、B、C。
* 副本A赢得选举，并晋升为主节点。
* 网络分区导致大部分集群无法访问A。
* 副本B赢得选举并晋升为主节点。
* 分区导致大部分集群无法访问B。
* 之前的分区已被修复，A再次可用。

这时B已经停止运行，A恢复可用并成为主节点（实际上，“更新”消息会迅速重新配置A，但在这里我们假设所有“更新”消息都丢失了）。同时，副本C将尝试选举，以接管B的故障转移工作。以下是具体过程：

1. C 将尝试竞选并成功，因为对于大部分从节点来说，其主节点的配置版本实际上较低。它将获得一个新的增量 `configEpoch`。
2. A 将无法声称其哈希槽位的主节点身份，因为其他节点已经与更高的配置版本（B 的版本）相关联，而不是 A 发布的版本。
3. 因此，所有节点将升级它们的表格，将哈希槽位分配给 C，并且集群将继续运行。

正如你将在接下来的章节中看到的那样，一个重新加入集群的陈旧节点通常会尽快收到有关配置更改的通知，因为一旦它将任何其他节点ping了一下，接收者就会检测到它有陈旧的信息，并发送一个`UPDATE`消息。

### 哈希槽配置传播

Redis集群的一个重要部分是用于传播有关哪个集群节点正在为给定的哈希槽提供服务的机制。这对于启动新的集群以及在将副本升级为提供其故障主节点槽的能力都非常重要。

相同机制允许由于不确定的时间而分区的节点以明智的方式重新加入集群。

哈希槽配置有两种传播方式：

1. 心跳信息。发送ping或pong数据包的发送者始终会添加关于它（或它的主节点，如果它是一个副本）所服务的哈希槽集合的信息。
2. `UPDATE`信息。由于每个心跳数据包都包含有关发送者的`configEpoch`和所服务的哈希槽集合的信息，如果心跳数据包的接收者发现发送者的信息过时，它将发送一个带有新信息的数据包，强制过时的节点更新其信息。

当接收到心跳或`UPDATE`消息时，接收者使用一些简单的规则来更新其表格，将散列槽映射到节点。当创建新的Redis集群节点时，其本地散列槽表只是简单地初始化为`NULL`条目，这样每个散列槽就不会与任何节点绑定或关联。看起来类似于以下格式：

```
0 -> NULL
1 -> NULL
2 -> NULL
...
16383 -> NULL
```

节点为了更新哈希槽表而遵循的第一个规则如下：

**规则1**：如果散列槽位未分配（设置为`NULL`），并且已知的节点宣称该槽位，我将修改我的散列槽表并将宣称的散列槽与其关联。

所以，如果我们从节点A收到一个心跳信号，声称使用配置纪元值为3来服务哈希槽1和2，那么表格将被修改为：

```
0 -> NULL
1 -> A [3]
2 -> A [3]
...
16383 -> NULL
```

在创建新的集群时，系统管理员需要手动分配每个主节点所服务的槽位（可以使用`CLUSTER ADDSLOTS`命令，通过redis-cli命令行工具或其他方式），这些信息会快速地在整个集群中传播。

## 然而，这个规则还不够。我们知道哈希槽位映射在以下两个事件期间可能会发生改变：

1. 在故障转移期间，副本替代其主节点。
2. 插槽从一个节点重新分片到另一个节点。

## 现在让我们专注于故障转移。当一个副本接管其主节点时，它会获得一个配置纪元，确保配置纪元大于其主节点（更一般地大于之前生成的任何其他配置纪元）。例如，一个名为B的副本可能以纪元4接管A。它将开始发送心跳包（第一次进行全集群广播），并且由于以下第二个规则，接收者将更新其散列槽表：

**规则2**：如果哈希槽已经被分配，并且已知的节点正在使用比当前与槽相关的主节点的`configEpoch`更大的`configEpoch`进行广告，我将把哈希槽重新绑定到新节点上。

所以在收到来自B的消息，声称要服务哈希槽1和2，配置时代为4后，接收方将按照以下方式更新它们的表格：

```
0 -> NULL
1 -> B [4]
2 -> B [4]
...
16383 -> NULL
```

存活性属性：由于第二个规则的存在，最终集群中的所有节点都将同意一个槽位的所有者是在宣传它的节点中具有最大`configEpoch`的节点。

在Redis Cluster中，这个机制被称为**最后一个故障切换成功**。

在resharding过程中也会发生同样的情况。当导入一个哈希槽的节点完成导入操作时，它的配置纪元会增加，以确保更改能够在整个集群中传播。

### 更新消息，更详细了解

有了前面的部分了解，更容易理解更新消息是如何工作的。节点A可能在一段时间后重新加入集群。它将发送心跳包，声称它使用配置纪元为3来服务哈希槽1和2。所有收到更新信息的接收器将看到同样的哈希槽与配置纪元较高的节点B相关联。因此，它们会向A发送一个`UPDATE`消息，包含槽的新配置。A会根据上面的**规则2**更新其配置。

### 节点如何重新加入集群

当节点重新加入集群时，使用相同的基本机制。
继续以上例子，节点A将被通知哈希槽1和2现在由B提供服务。假设这两个是A唯一提供服务的哈希槽，A提供的哈希槽数量将降为0！因此A将重新配置为新主节点的副本。

实际遵循的规则比这稍微复杂一些。一般情况下，可能会出现在 A 重新加入之后经过很长时间，而在此期间，原本由 A 服务的哈希槽可能由多个节点服务，例如哈希槽 1 由 B 服务，哈希槽 2 由 C 服务。

实际的**Redis 集群节点角色切换规则**是：**主节点将更改其配置以复制（成为）窃取了其最后一个哈希槽的节点**。

在重新配置过程中，最终提供的哈希槽数将降至零，并且节点将相应地重新配置。请注意，在基本情况下，这仅意味着旧的主节点将成为取代它的副本的副本。然而，在一般情况下，该规则涵盖了所有可能的情况。

副本完全相同：它们重新配置以复制上一个哈希槽位被其前任主节点窃取的节点。

### 副本迁移

Redis Cluster 在系统中实现了名为*副本迁移*的概念，以提高系统的可用性。这个想法是，在一个带有主副本设置的集群中，如果副本和主节点之间的映射是固定的，那么如果单个节点有多个独立故障，随着时间的推移，可用性将受到限制。

例如，在一个集群中，每个主节点都有一个副本，只要主节点或副本中的任何一个发生故障，集群就可以继续运行，但如果它们同时发生故障，则无法继续运行。然而，有一类故障是由硬件或软件问题引起的单个节点的独立故障，它们会随着时间的推移而累积。例如：

* 主节点A有一个副本A1。
* 主节点A发生故障。A1被提升为新的主节点。
* 三个小时后，A1独立发生故障（与A的故障无关）。由于节点A仍然宕机，没有其他副本可用于提升。集群无法继续正常操作。

如果主节点和副本之间的映射是固定的，使集群对上述情况更加抵抗的唯一方法是向每个主节点添加副本，然而这是昂贵的，因为需要执行更多的Redis实例、更多的内存等等。

另一种方法是在集群中创建不对称，并允许集群布局随时间自动变化。例如，集群可以有三个主节点 A、B、C。A 和 B 分别有一个副本 A1 和 B1。然而，主节点 C 不同，有两个副本：C1 和 C2。

**复制品迁移**是指自动重新配置复制品以迁移到没有覆盖范围（没有正常工作的复制品）的主服务的过程。通过复制品迁移，上述情况会变成以下情况：

* 主节点A失败，A1被晋升为新的主节点。
* C2作为A1的副本进行迁移，但没有其他副本可支持。
* 三小时后，A1也失败。
* C2被晋升为新的主节点，取代A1。
* 集群可以继续进行操作。

### 复制品迁移算法

迁移算法不使用任何形式的协议，因为Redis Cluster中的副本布局不是需要与配置时代一致或版本化的集群配置的一部分。相反，它使用一种算法来避免在没有备份主节点的情况下进行大规模副本迁移。该算法保证在集群配置稳定后，每个主节点都将至少有一个副本进行备份。

以下是算法的工作方式。首先，我们需要定义在此上下文中什么是*好的副本*：一个好的副本是指从给定节点的角度来看，处于`FAIL`状态之外的副本。

算法的执行在每个检测到至少有一个没有好的副本的主节点的副本中触发。然而，在所有检测到此条件的副本中，只有一个子集应该起作用。除非不同的副本在某一时刻对其他节点的故障状态具有稍微不同的观点，否则这个子集实际上通常是一个单独的副本。

**执行副本**是指在主节点中具有最多附加副本的副本，即不处于失败状态并且具有最小节点ID的副本。

所以举个例子，如果有10个主节点每个节点有1个副本，还有2个主节点每个节点有5个副本，那么将尝试迁移的副本是在这2个具有5个副本的主节点中具有最低节点ID的副本。由于没有使用协议，当集群配置不稳定时，可能会发生竞争条件，多个副本都认为自己是非故障的具有更低节点ID的副本（实践中这种情况不太可能发生）。如果发生这种情况，结果是多个副本迁移到同一个主节点，这是无害的。如果竞争以一种方式发生，这将导致让这个放弃的主节点没有副本，一旦集群再次稳定，算法将被重新执行，并将副本迁移回原始主节点。

最终，每个主节点最少会有一个副本支持。然而，正常情况是，一个主节点上的单个副本会从具有多个副本的主节点迁移至孤立的主节点。

该算法由一个用户可配置参数`cluster-migration-barrier`控制，此参数表示主节点在副本迁移之前必须保留的可用副本数。例如，如果将此参数设置为2，则副本只能在其主节点保留两个正常工作的副本时尝试进行迁移。

### configEpoch 冲突解决算法

当通过故障转移期间的副本晋升创建新的`configEpoch`值时，它们被保证是唯一的。

然而，有两个明显的事件会以不安全的方式创建新的configEpoch值，只是简单地对本地节点的`currentEpoch`进行增加，并希望在同一时间没有冲突。这两个事件都是由系统管理员触发的：

1. `CLUSTER FAILOVER`命令使用`TAKEOVER`选项可以手动将一个副本节点提升为主节点，而不需要大多数主节点可用。这在多数据中心设置中非常有用。
2. 为了性能原因，在集群重新平衡时，槽迁移也会在本地节点生成新的配置纪元，而无需达成一致。

具体来说，在手动重新分片过程中，当一个哈希槽从节点A迁移到节点B时，重新分片程序将强制B将其配置升级到集群中找到的最大时间戳加1（除非该节点已经是具有最大配置时间戳的节点），而无需其他节点的同意。
通常，实际的重新分片涉及移动几百个哈希槽（尤其是在小集群中）。在重新分片过程中，对于每个移动的哈希槽，需要达成一致才能生成新的配置时间戳是低效的。此外，为了存储新的配置，每次都需要在集群的每个节点上进行fsync操作。相反，通过执行的方式，我们只需要在第一个哈希槽被移动时生成一个新的配置时间戳，这在生产环境中更加高效。

由于上述两种情况，可能会（虽然不太可能）导致多个节点具有相同的配置纪元。如果系统管理员执行了一次重分片操作，并且同时发生了一次故障转移（加上很多不幸的事情），如果它们传播得不够快，可能会导致`currentEpoch`冲突。

此外，软件错误和文件系统损坏也可能导致多个节点具有相同的配置纪元。

当服务于不同哈希槽的主节点具有相同的 `configEpoch` 时，不存在问题。重要的是，在副本故障转移主节点时，它们的配置时代是唯一的。

以上所说，手动干预或重新分片可能以不同的方式更改群集配置。Redis Cluster 主要的活性属性要求槽配置始终收敛，因此在任何情况下，我们确实希望所有主节点都具有不同的 `configEpoch`。

为了执行这个操作，**一个冲突解决算法**被用于当两个节点最终拥有相同的`configEpoch`时。

* 如果一个主节点检测到另一个主节点正在使用相同的 `configEpoch` 进行广告。
* 并且如果该节点的节点 ID 在与其他节点声明相同的 `configEpoch` 的节点之间按字典顺序较小。
* 那么它将将其 `currentEpoch` 增加 1，并将其用作新的 `configEpoch`。

如果存在具有相同`configEpoch`的一组节点，则除了具有最大节点ID的节点之外的所有节点都将向前移动，从而保证最终每个节点都将选择一个独特的`configEpoch`，而不管发生了什么。

该机制还保证在创建新的集群后，所有节点都会从一个不同的`configEpoch`开始（即使这实际上不会被使用），因为`redis-cli`在启动时确保使用`CLUSTER SET-CONFIG-EPOCH`。然而，如果由于某些原因节点配置错误，则它会自动更新到一个不同的配置时期。

### 节点重置

节点可以进行软件复位（无需重新启动）以便在不同的作用或不同的集群中重复使用。这在正常操作、测试和云环境中非常有用，因为一个给定的节点可以重新配置以加入不同的节点集来扩大或创建一个新的集群。

在 Redis 集群中，使用 `CLUSTER RESET` 命令来重置节点。该命令提供了两种变体：

* `集群重置 SOFT`
* `集群重置 HARD`

要执行重置操作，命令必须直接发送给节点。如果未提供重置类型，则执行软重置。

以下是复位执行的操作列表：

1. 关闭设备并断开电源。
2. 拆下外部连接和附件。
3. 按住重置按钮。
4. 等待片刻后松开重置按钮。
5. 重新连接外部连接和附件。
6. 打开设备并重新启动。

1. 软重置和硬重置：如果节点是副本，则将其转换为主节点，并丢弃其数据集。如果节点是主节点并且包含键，则重置操作将被中止。
2. 软重置和硬重置：所有槽位都被释放，并且手动故障转移状态被重置。
3. 软重置和硬重置：节点表中的所有其他节点都被移除，因此节点不再了解任何其他节点。
4. 仅硬重置：`currentEpoch`、`configEpoch`和`lastVoteEpoch`被设置为0。
5. 仅硬重置：节点ID被更改为一个新的随机ID。

带有非空数据集的主节点不能被重置（因为通常希望将数据重新分片到其他节点上）。但是，在特殊情况下当此操作是适当的时候（例如当群集完全被销毁以创建新群集时），在进行重置之前必须执行`FLUSHALL`命令。

### 从集群中移除节点

对于现有的集群，通过将所有数据重新分片到其他节点（如果是主节点）并关闭该节点，实际上可以将节点从集群中移除。然而，其他节点仍会记住它的节点ID和地址，并尝试连接它。

由于这个原因，在移除节点的同时我们也想要将其在所有其他节点的表中的条目删除。这可以通过使用`CLUSTER FORGET <node-id>`命令来实现。

命令执行两个操作：

1. 它从节点表中删除具有指定节点ID的节点。
2. 它设置了一个60秒的禁止，防止具有相同节点ID的节点重新添加。

第二个操作是必要的，因为Redis Cluster使用gossip来自动发现节点，所以从节点A中删除节点X可能导致节点B再次向节点A传递有关节点X的消息。由于60秒的禁止期限，Redis Cluster管理工具有60秒的时间将节点从所有节点中移除，以防止由于自动发现而重新添加节点。

有关详细信息，请参阅`CLUSTER FORGET`文档。

## 发布/订阅

在Redis集群中，客户端可以订阅每个节点，还可以向其他节点发布消息。集群会确保所发布的消息按照需要进行转发。

客户端可以向任何节点发送SUBSCRIBE命令，也可以向任何节点发送PUBLISH命令。
它会将每个发布的消息广播给所有其他节点。

Redis 7.0及更高版本支持分片发布/订阅功能，其中分片通道由相同算法分配到插槽中。必须将分片消息发送到拥有散列至的插槽的节点。集群确保发布的分片消息被转发到分片中的所有节点，因此客户端可以通过连接到负责该插槽的主节点或其任何副本来订阅分片通道。

## 附录

### 附录A：ANSI C中的CRC16参考实现

```c
#include <stdio.h>
#include <stdint.h>

#define CRC16_POLYNOMIAL 0xA001

uint16_t crc16(uint8_t *data, size_t len) {
    uint16_t crc = 0xFFFF;

    for (size_t i = 0; i < len; i++) {
        crc ^= data[i];
        for (int j = 0; j < 8; j++) {
            if (crc & 0x0001) {
                crc = (crc >> 1) ^ CRC16_POLYNOMIAL;
            } else {
                crc >>= 1;
            }
        }
    }

    return crc;
}

int main() {
    uint8_t data[] = {0x01, 0x02, 0x03, 0x04};
    size_t len = sizeof(data) / sizeof(data[0]);
    uint16_t crc = crc16(data, len);
    printf("CRC16: 0x%04X\n", crc);

    return 0;
}
```


    /*
     * Copyright 2001-2010 Georges Menie (www.menie.org)
     * Copyright 2010 Salvatore Sanfilippo (adapted to Redis coding style)
     * All rights reserved.
     * Redistribution and use in source and binary forms, with or without
     * modification, are permitted provided that the following conditions are met:
     *
     *     * Redistributions of source code must retain the above copyright
     *       notice, this list of conditions and the following disclaimer.
     *     * Redistributions in binary form must reproduce the above copyright
     *       notice, this list of conditions and the following disclaimer in the
     *       documentation and/or other materials provided with the distribution.
     *     * Neither the name of the University of California, Berkeley nor the
     *       names of its contributors may be used to endorse or promote products
     *       derived from this software without specific prior written permission.
     *
     * THIS SOFTWARE IS PROVIDED BY THE REGENTS AND CONTRIBUTORS ``AS IS'' AND ANY
     * EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
     * WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
     * DISCLAIMED. IN NO EVENT SHALL THE REGENTS AND CONTRIBUTORS BE LIABLE FOR ANY
     * DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
     * (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
     * LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
     * ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
     * (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
     * SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
     */

    /* CRC16 implementation according to CCITT standards.
     *
     * Note by @antirez: this is actually the XMODEM CRC 16 algorithm, using the
     * following parameters:
     *
     * Name                       : "XMODEM", also known as "ZMODEM", "CRC-16/ACORN"
     * Width                      : 16 bit
     * Poly                       : 1021 (That is actually x^16 + x^12 + x^5 + 1)
     * Initialization             : 0000
     * Reflect Input byte         : False
     * Reflect Output CRC         : False
     * Xor constant to output CRC : 0000
     * Output for "123456789"     : 31C3
     */

    static const uint16_t crc16tab[256]= {
        0x0000,0x1021,0x2042,0x3063,0x4084,0x50a5,0x60c6,0x70e7,
        0x8108,0x9129,0xa14a,0xb16b,0xc18c,0xd1ad,0xe1ce,0xf1ef,
        0x1231,0x0210,0x3273,0x2252,0x52b5,0x4294,0x72f7,0x62d6,
        0x9339,0x8318,0xb37b,0xa35a,0xd3bd,0xc39c,0xf3ff,0xe3de,
        0x2462,0x3443,0x0420,0x1401,0x64e6,0x74c7,0x44a4,0x5485,
        0xa56a,0xb54b,0x8528,0x9509,0xe5ee,0xf5cf,0xc5ac,0xd58d,
        0x3653,0x2672,0x1611,0x0630,0x76d7,0x66f6,0x5695,0x46b4,
        0xb75b,0xa77a,0x9719,0x8738,0xf7df,0xe7fe,0xd79d,0xc7bc,
        0x48c4,0x58e5,0x6886,0x78a7,0x0840,0x1861,0x2802,0x3823,
        0xc9cc,0xd9ed,0xe98e,0xf9af,0x8948,0x9969,0xa90a,0xb92b,
        0x5af5,0x4ad4,0x7ab7,0x6a96,0x1a71,0x0a50,0x3a33,0x2a12,
        0xdbfd,0xcbdc,0xfbbf,0xeb9e,0x9b79,0x8b58,0xbb3b,0xab1a,
        0x6ca6,0x7c87,0x4ce4,0x5cc5,0x2c22,0x3c03,0x0c60,0x1c41,
        0xedae,0xfd8f,0xcdec,0xddcd,0xad2a,0xbd0b,0x8d68,0x9d49,
        0x7e97,0x6eb6,0x5ed5,0x4ef4,0x3e13,0x2e32,0x1e51,0x0e70,
        0xff9f,0xefbe,0xdfdd,0xcffc,0xbf1b,0xaf3a,0x9f59,0x8f78,
        0x9188,0x81a9,0xb1ca,0xa1eb,0xd10c,0xc12d,0xf14e,0xe16f,
        0x1080,0x00a1,0x30c2,0x20e3,0x5004,0x4025,0x7046,0x6067,
        0x83b9,0x9398,0xa3fb,0xb3da,0xc33d,0xd31c,0xe37f,0xf35e,
        0x02b1,0x1290,0x22f3,0x32d2,0x4235,0x5214,0x6277,0x7256,
        0xb5ea,0xa5cb,0x95a8,0x8589,0xf56e,0xe54f,0xd52c,0xc50d,
        0x34e2,0x24c3,0x14a0,0x0481,0x7466,0x6447,0x5424,0x4405,
        0xa7db,0xb7fa,0x8799,0x97b8,0xe75f,0xf77e,0xc71d,0xd73c,
        0x26d3,0x36f2,0x0691,0x16b0,0x6657,0x7676,0x4615,0x5634,
        0xd94c,0xc96d,0xf90e,0xe92f,0x99c8,0x89e9,0xb98a,0xa9ab,
        0x5844,0x4865,0x7806,0x6827,0x18c0,0x08e1,0x3882,0x28a3,
        0xcb7d,0xdb5c,0xeb3f,0xfb1e,0x8bf9,0x9bd8,0xabbb,0xbb9a,
        0x4a75,0x5a54,0x6a37,0x7a16,0x0af1,0x1ad0,0x2ab3,0x3a92,
        0xfd2e,0xed0f,0xdd6c,0xcd4d,0xbdaa,0xad8b,0x9de8,0x8dc9,
        0x7c26,0x6c07,0x5c64,0x4c45,0x3ca2,0x2c83,0x1ce0,0x0cc1,
        0xef1f,0xff3e,0xcf5d,0xdf7c,0xaf9b,0xbfba,0x8fd9,0x9ff8,
        0x6e17,0x7e36,0x4e55,0x5e74,0x2e93,0x3eb2,0x0ed1,0x1ef0
    };

    uint16_t crc16(const char *buf, int len) {
        int counter;
        uint16_t crc = 0;
        for (counter = 0; counter < len; counter++)
                crc = (crc<<8) ^ crc16tab[((crc>>8) ^ *buf++)&0x00FF];
        return crc;
    }