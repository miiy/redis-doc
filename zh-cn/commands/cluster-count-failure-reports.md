该命令返回指定节点的*故障报告*数量。
故障报告是Redis集群用来将`PFAIL`状态（表示节点不可达）升级为`FAIL`状态（表示集群中大多数主节点在一段时间窗口内达成一致，认为节点不可达）的一种方式。

一些更多的细节：

* 当一个节点在配置的节点超时时间上不可达时，会使用 `PFAIL` 标志另一个节点，这是 Redis 集群的一个基本配置参数。
* 处于 `PFAIL` 状态的节点被包含在心跳包的八卦部分中提供。
* 每当一个节点处理来自其他节点的八卦包时，它会创建（如果需要刷新 TTL）**故障报告**，记住一个给定的节点说另一个给定的节点处于 `PFAIL` 状态。
* 每个故障报告的存活时间是节点超时时间的两倍。
* 如果在某个时间点上一个节点标记另一个节点为 `PFAIL`，并且同时收集了大多数其他主节点关于该节点（包括它自己如果是主节点）的故障报告，那么它将该节点的失败状态从 `PFAIL` 提升为 `FAIL`，并广播一条消息，强制所有可达的节点将该节点标记为 `FAIL`。

该命令返回当前节点的故障报告数量，这些报告尚未过期（即在节点超时时间的两倍内收到）。计数不包括我们询问此计数的节点关于我们所传递的节点ID的信念，计数*仅*包括节点从其他节点收到的故障报告。

这个命令在调试过程中非常有用，当我们认为Redis集群的故障检测器运行不正常时。