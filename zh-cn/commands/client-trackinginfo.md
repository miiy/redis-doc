该命令返回有关当前客户端连接使用[服务器辅助客户端端缓存](/topics/client-side-caching)功能的信息。

以下是跟踪信息部分及其各自的值列表：

* **标志**：连接使用的跟踪标志列表。标志及其含义如下：
  * `off`：连接未使用服务器辅助的客户端缓存。
  * `on`：连接启用了服务器辅助的客户端缓存。
  * `bcast`：客户端使用广播模式。
  * `optin`：默认情况下，客户端不缓存键。
  * `optout`：默认情况下，客户端缓存键。
  * `caching-yes`：下一个命令将缓存键（仅与 `optin` 一起存在）。
  * `caching-no`：下一个命令将不缓存键（仅与 `optout` 一起存在）。
  * `noloop`：客户端不会接收到自己修改的键的通知。
  * `broken_redirect`：重定向使用的客户端 ID 不再有效。
* **重定向**：通知重定向时使用的客户端 ID，如果没有则为 -1。
* **前缀**：通过这些键前缀向客户端发送通知的列表。