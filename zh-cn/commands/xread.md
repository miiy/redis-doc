从一个或多个流中读取数据，只返回由调用方报告的最后接收到的ID更大的条目。
此命令有一个选项，如果没有可用的项目，它会阻塞，类似于`BRPOP`或`BZPOPMIN`等其他命令。

请注意，在阅读本页面之前，如果你对流处理不熟悉，
我们建议阅读[我们关于Redis Streams的介绍](/topics/streams-intro)。

## 非阻塞使用方式

如果不使用**BLOCK**选项，则该命令是同步的，并且可以认为与`XRANGE`有一定的关联性：它将返回流中的一系列项目，但即使我们仅考虑同步使用，与`XRANGE`相比，它有两个根本差别：

* 如果我们想从多个键中同时读取数据，可以使用多个流来调用此命令。这是`XREAD`的一个关键特性，特别是在使用**BLOCK**进行阻塞时，能够使用单个连接同时监听多个键是至关重要的功能。
* 虽然`XRANGE`返回指定范围内的数据项，但是`XREAD`更适合从我们迄今为止看到的任何其他数据项以后的第一个数据项开始消费流。因此，我们针对每个流传递给`XREAD`的是我们从该流接收到的最后一个元素的ID。

例如，如果我有两个流`mystream`和`writers`，我想从它们包含的第一个元素开始读取数据，可以像以下示例中的`XREAD`一样调用。

注意：我们在示例中使用了**COUNT**选项，这样每个流的调用将最多返回两个元素。

```
> XREAD COUNT 2 STREAMS mystream writers 0-0 0-0
1) 1) "mystream"
   2) 1) 1) 1526984818136-0
         2) 1) "duration"
            2) "1532"
            3) "event-id"
            4) "5"
            5) "user-id"
            6) "7782813"
      2) 1) 1526999352406-0
         2) 1) "duration"
            2) "812"
            3) "event-id"
            4) "9"
            5) "user-id"
            6) "388234"
2) 1) "writers"
   2) 1) 1) 1526985676425-0
         2) 1) "name"
            2) "Virginia"
            3) "surname"
            4) "Woolf"
      2) 1) 1526985685298-0
         2) 1) "name"
            2) "Jane"
            3) "surname"
            4) "Austen"
```

**STREAMS**选项是强制性的，并且必须是最后一个选项，因为该选项以以下格式获取可变长度的参数：

    STREAMS key_1 key_2 key_3 ... key_N ID_1 ID_2 ID_3 ... ID_N

所以我们从一个键列表开始，之后继续使用所有相关的ID，表示*我们接收到的最后一个流的ID*，这样调用将仅为我们提供来自同一流的更大ID。

例如，在上面的示例中，我们收到的流`mystream`的最后一项的ID是`1526999352406-0`，而流`writers`的ID是`1526985685298-0`。

继续迭代两个流我会调用：

```
> XREAD COUNT 2 STREAMS mystream writers 1526999352406-0 1526985685298-0
1) 1) "mystream"
   2) 1) 1) 1526999626221-0
         2) 1) "duration"
            2) "911"
            3) "event-id"
            4) "7"
            5) "user-id"
            6) "9488232"
2) 1) "writers"
   2) 1) 1) 1526985691746-0
         2) 1) "name"
            2) "Toni"
            3) "surname"
            4) "Morrison"
      2) 1) 1526985712947-0
         2) 1) "name"
            2) "Agatha"
            3) "surname"
            4) "Christie"
```

所以等等。最后，调用将不会返回任何项，而只是一个空数组，然后我们知道没有更多的东西可以从我们的流中获取（我们将不得不重试操作，因此该命令还支持阻止模式）。

## 不完整的ID

使用不完整的ID是有效的，就像对于`XRANGE`有效一样。然而，在这里，如果ID的序列部分缺失，将始终被解释为零，所以命令为：

```
> XREAD COUNT 2 STREAMS mystream writers 0 0
```

is exactly equivalent to

```
> XREAD COUNT 2 STREAMS mystream writers 0-0 0-0
```

## 阻塞数据

在其同步形式中，只要还有更多的项可用，该命令就可以获取新数据。但是，在某个时候，我们将不得不等待数据的生产者使用`XADD`将新条目推送到我们正在消费的流中。为了避免以固定或自适应间隔进行轮询，如果命令无法返回任何数据，它可以阻塞，并根据指定的流和ID自动解除阻塞，直到其中一个请求的键接受数据。

重要: 需要理解这个命令**串播(fans out)**到正在等待相同ID范围的所有客户端，所以每个消费者将获得数据的副本，不同于使用阻塞列表弹出操作时的情况。

为了进行阻塞，需要使用**BLOCK**选项，以及我们希望在超时之前阻塞的毫秒数。通常，Redis的阻塞命令以秒为单位进行超时计算，但此命令以毫秒为单位进行超时计算，即使服务器通常的超时分辨率接近0.1秒。在某些使用情况下，这可以缩短阻塞的时间，如果服务器内部随着时间的推移而改进，超时的分辨率也可能会改进。

当传递了**BLOCK**命令，但至少在一个流中有要返回的数据时，在执行命令时会同步执行，就好像缺少BLOCK选项一样。

这是阻塞调用的一个示例，命令后来返回一个空回复，因为超时时限到期而没有新的数据到达：

```
> XREAD BLOCK 1000 STREAMS mystream 1526999626221-0
(nil)
```

## 特殊的 `$` ID.

在阻塞时，有时我们希望只接收通过`XADD`添加到流中的条目，从我们阻塞的那一刻开始。在这种情况下，我们对已经添加的条目的历史不感兴趣。对于这种用例，我们需要检查流的顶部元素ID，并在`XREAD`命令行中使用这样的ID。这种方法不够清晰，需要调用其他命令，因此可以使用特殊的`$`符号来表示只获取新的内容。

**非常重要**的是要明白，在调用`XREAD`时，应该只在第一次调用时使用`$`符号作为ID。后续的调用中，ID应该是流中上一次报告的项目的ID，否则你可能会错过中间添加的所有条目。

这是一个正常的`XREAD`调用在一个只想消费新条目的消费者的第一次迭代中的样子。

```
> XREAD BLOCK 5000 COUNT 100 STREAMS mystream $
```

一旦我们收到一些回复，下一个电话将是这样的：

```
> XREAD BLOCK 5000 COUNT 100 STREAMS mystream 1526999644174-3
```

依此类推。

## 多个客户端在单个流上被阻塞时如何提供服务

在列表或有序集合上的阻塞列表操作具有*pop*行为。
基本上，元素从列表或有序集合中被移除，以便返回给客户端。在这种情况下，您希望项目以公平的方式被消耗，这取决于在给定键上阻塞的客户端到达的时刻。通常情况下，Redis在这些用例中使用FIFO语义。

然而，请注意，通过流来实现时，这个问题就不存在了：当客户端服务时，流条目不会从流中移除，因此每个等待的客户端都会在 `XADD` 命令提供数据到流中时被服务。

建议阅读[Redis Streams 简介](/topics/streams-intro)，以更好地理解有关流的整体行为和语义。