将在使用`EVAL`执行的后续脚本中设置调试模式。Redis包含一个完整的Lua调试器，代号为LDB，可以用来简化编写复杂脚本的任务。在调试模式下，Redis充当远程调试服务器，客户端（例如`redis-cli`）可以逐步执行脚本，设置断点，检查变量等等 - 有关LDB的更多信息，请参阅[Redis Lua调试器](/topics/ldb)页面。

**重要提示:** 避免在 Redis 生产服务器上调试 Lua 脚本。请使用开发服务器代替。

LDB 可以在异步模式或同步模式下启用。在异步模式下，服务器创建了一个分叉的调试会话，不会阻塞，并且在会话完成后将回滚所有对数据的更改，因此可以使用相同的初始状态重新开始调试。而同步调试模式则在调试会话活动期间阻塞服务器，并保留所有对数据集的更改。

* `YES`。启用Lua脚本的非阻塞异步调试（更改将被丢弃）。
* `!SYNC`。启用Lua脚本的阻塞同步调试（更改将保存到数据）。
* `NO`。禁用脚本调试模式。

有关"EVAL"脚本的更多信息，请参阅[EVAL脚本简介](/topics/eval-intro)。