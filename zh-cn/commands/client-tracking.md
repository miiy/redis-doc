该命令启用Redis服务器的跟踪功能，用于[服务器辅助的客户端缓存](/topics/client-side-caching)。

当启用跟踪时，Redis会记住连接请求的键，以便在修改这些键时后续发送失效消息。无效消息会在同一连接中发送（仅在使用RESP3协议时可用），或在不同连接中重定向（在RESP2和Pub/Sub中也可用）。还提供了一种特殊的*广播*模式，其中参与此协议的客户端通过订阅给定键前缀，接收每个通知，而不管他们请求了哪些键。考虑到参数的复杂性，请参考[主要的客户端端缓存文档](/topics/client-side-caching) 获取详细信息。此手册页仅作为此子命令选项的参考。

为了启用追踪功能，请使用以下内容：

    CLIENT TRACKING on ... options ...

特性将在当前连接中一直保持活动状态，直到在某个时刻使用`CLIENT TRACKING off`关闭跟踪。

下面是在启用跟踪时修改命令行行为的选项列表：

* `REDIRECT <id>`：向指定的连接ID发送失效消息。连接必须存在。您可以使用`CLIENT ID`命令获取连接的ID。如果正在重定向的连接被终止，当处于RESP3模式时，启用跟踪的连接将接收到`tracking-redir-broken`推送消息以示条件。
* `BCAST`：启用广播模式的跟踪。在此模式下，无论连接请求的键是什么，都会报告所有指定的前缀的失效消息。当未启用广播模式时，Redis将跟踪使用只读命令获取的键，并仅对这些键报告失效消息。
* `PREFIX <prefix>`：对于广播，注册指定的键前缀，以便仅为以此字符串开头的键提供通知。可以多次使用此选项注册多个前缀。如果未使用此选项启用广播，则Redis将为每个键发送通知。您不能删除单个前缀，但可以通过禁用和重新启用跟踪来删除所有前缀。使用此选项会增加额外的时间复杂性O(N^2)，其中N是跟踪的前缀总数。
* `OPTIN`：当广播模式未激活时，通常不要跟踪只读命令中的键，除非它们紧随`CLIENT CACHING yes`命令之后调用。
* `OPTOUT`：当广播模式未激活时，通常在只读命令中跟踪键，除非它们紧随`CLIENT CACHING no`命令之后调用。
* `NOLOOP`：不要发送有关由该连接本身修改的键的通知。