该命令只返回当前连接的ID。每个连接ID都有一定的保证：

1. 它永远不会重复，因此如果 `客户端 ID` 返回相同的数字，则调用者可以确定基础客户端没有断开并重新连接连接，而是仍然是同一个连接。
2. 该ID单调递增。如果一个连接的ID大于另一个连接的ID，则可以保证第二个连接是在服务器上建立的时间晚于第一个连接。

该命令与`CLIENT UNBLOCK`命令一起使用非常有用，它们与`CLIENT ID`命令一同引入到Redis 5中。请查看`CLIENT UNBLOCK`命令页面以了解涉及这两个命令的模式。

```cli
CLIENT ID
```