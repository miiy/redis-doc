当只使用`key`参数调用时，从存储在`key`的有序集合中返回一个随机元素。

如果提供的`count`参数是正数，则返回一个包含**不同元素**的数组。
数组的长度是`count`或排序集合的基数 (`ZCARD`)，以较小的值为准。

如果使用负数的 `count` 进行调用，行为将会改变，允许该命令**多次返回相同的元素**。
在这种情况下，返回元素的数量是指定 `count` 的绝对值。

选择 `WITHSCORES` 修饰符可改变响应，使其包括从有序集合中随机选择的元素的相应分数。

具体的使用示例如下：


```cli
ZADD dadi 1 uno 2 due 3 tre 4 quattro 5 cinque 6 sei
ZRANDMEMBER dadi
ZRANDMEMBER dadi
ZRANDMEMBER dadi -5 WITHSCORES
```

## 当传递了count时的行为规范

当`count`参数为正值时，该命令的行为如下：

* 不会返回重复的元素。
* 如果`count`大于有序集合的基数，命令只会返回完整的有序集合，不会再添加其他元素。
* 回复中元素的顺序并不是真正的随机，所以需要客户端自行打乱它们。

当`count`是一个负值时，行为如下所示：

* 可能存在重复元素。
* 总是返回恰好`count`个元素，如果有序集合为空（不存在的键），则返回一个空数组。
* 回复中元素的顺序是真正的随机的。