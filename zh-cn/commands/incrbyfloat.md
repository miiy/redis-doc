使用指定的增量值对存储在 `key` 中的浮点数表示的字符串进行增加操作。通过使用负的 `increment` 值，可以实现对键存储的值进行递减操作（根据加法的明显属性）。
如果键不存在，则在执行操作之前将其设置为 `0`。
如果发生以下任一条件，则返回错误：

* 关键字包含错误类型的值（不是字符串）。
* 当前关键字内容或指定的增量不能解析为双精度浮点数。

如果命令成功，新的递增值将存储为键的新值（替换旧值），并以字符串形式返回给调用者。

字符串键中已包含的值和增量参数都可以选择以指数表示法提供，但在增量之后计算得到的值始终以相同的格式存储，即一个整数后面（如果需要）跟着一个点，并且紧接着是表示小数部分的可变数量的数字。尾部的零总是会被移除。

该输出的精度在小数点后固定为17位，而与计算的内部精度无关。

@examples

```cli
SET mykey 10.50
INCRBYFLOAT mykey 0.1
INCRBYFLOAT mykey -5
SET mykey 5.0e3
INCRBYFLOAT mykey 2.0e2
```

## 实施细节

命令总是在复制链和追加日志文件中以`SET`操作的形式传播，以确保底层浮点数计算实现的差异不会导致不一致性。