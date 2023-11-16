删除所有现有数据库的所有键，而不仅仅是当前选择的数据库。
该命令永远不会失败。

默认情况下，`FLUSHALL`会同步刷新所有数据库。
从Redis 6.2开始，将**lazyfree-lazy-user-flush**配置指令设置为“yes”将更改默认刷新模式为异步。

可以使用以下修饰符之一来明确指定刷新模式：

* `ASYNC`：异步刷新数据库
* `!SYNC`：同步刷新数据库

注意：异步的`FLUSHALL`命令只会删除在命令调用时存在的键。在异步刷新期间创建的键不会受到影响。

## 行为变更历史

- `>= 6.2.0`：默认刷新行为现在可以通过**lazyfree-lazy-user-flush**配置指令进行配置。