关于在该手册和命令中使用的 "Slave" 一词的注解：从 Redis 5 版本开始，除了为了向后兼容之外，Redis 项目不再使用 "Slave" 一词。请使用新的命令 `CLUSTER REPLICAS`。命令 `CLUSTER SLAVES` 将继续为向后兼容而工作。

该命令提供一个副本节点列表，这些副本节点从指定的主节点进行复制。列表以与`CLUSTER NODES`相同的格式提供（请参考其文档以获取格式的规范）。

如果指定的节点在节点接收到命令的节点表中未知或者不是主节点，则该命令将失败。

请注意，如果给定的主节点添加、移动或删除了副本，并且我们在尚未收到配置更新的节点上使用 `CLUSTER SLAVES`，它可能显示过时的信息。但是，最终（如果没有网络分区的情况下，大约几秒钟内），所有节点将就与给定主节点关联的节点集达成一致。