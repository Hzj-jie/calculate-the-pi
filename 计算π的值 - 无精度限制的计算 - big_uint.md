# 计算π的值 - 无精度限制的计算 - big_uint

big_uint是个非常常见的数据结构，简单来说是一个4294967296进制的数字，所有的运算都是基于这两个进制的。所以这个数据结构只有一个可变长的uint32的数组。

```
Partial Public NotInheritable Class big_uint
    ' Little-endian
    Private ReadOnly v As adaptive_array_uint32
...
End Class
```

adaptive_array_uint32等同于一个vector<uint32>，只是简化掉了一些对实现没有什么作用的函数。

big_uint的功能包括几个部分

- 输入输出

常见数据类型，比如uint32 uint64 boolean

字节数组

字符串，支持任意进制

- 运算

加减乘除、取模、次方、开方、阶乘、最大公约数

- 二进制运算

与、或、非运算，奇偶性，移位，判断是否是2的次方

- 比较

剩下还有一些和.Net相关的设计，比如重载ToString，Equals，运算符等。以及为测试设计的函数，返回一个随机big_uint的random函数等。
