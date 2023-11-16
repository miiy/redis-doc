`REPLICAOF`命令可以在运行时更改副本的复制设置。

如果Redis服务器已经作为副本工作，命令`REPLICAOF NO ONE`将关闭复制功能，将Redis服务器变为MASTER。以正确的格式`REPLICAOF`（主机名/ IP地址）（端口号）可以将该服务器设置为另一个指定主机名和端口号的服务器的副本。

如果一个服务器已经是某个主服务器的副本，`REPLICAOF` hostname port 命令将停止对旧服务器的复制，并开始对新服务器进行同步，丢弃旧的数据集。

`REPLICAOF` NO ONE的形式将停止复制，将服务器转为主服务器，但不会丢弃复制。因此，如果旧的主服务器停止工作，可以将副本转为主服务器，并设置应用程序在读/写时使用此新主服务器。以后修复另一个Redis服务器后，可重新配置为作为副本工作。

@examples

```
> REPLICAOF NO ONE
"OK"

> REPLICAOF 127.0.0.1 6799
"OK"
```