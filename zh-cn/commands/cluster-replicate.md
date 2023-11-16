命令将一个节点重新配置为指定主节点的副本。
如果接收命令的节点是一个"空的主节点"，作为命令的副作用，节点角色将从主节点更改为副本。

一旦一个节点被转变为另一个主节点的副本，就没有必要通知其他集群节点进行更改：节点之间交换的心跳包会自动传播新的配置。

副本将始终接受命令，假设：

1. 在其节点表中存在指定的节点ID。
2. 指定的节点ID没有标识我们发送命令到的实例。
3. 指定的节点ID是主节点。

如果接收命令的节点尚未是副本节点，而是主节点，只有在满足以下附加条件的情况下，命令才会成功，并将该节点转换为副本节点：

1. 节点没有为任何散列插槽提供服务。
2. 节点为空，键空间中没有存储任何键。

如果命令成功，新的副本将立即尝试联系其主服务器以进行复制。