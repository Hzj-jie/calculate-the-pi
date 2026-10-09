# 计算π的值 - 无精度限制的计算 - big_udec的基本结构

终于从整数到了小数了，毕竟π是个小数。

当然这里说“小数”是不准确的，描述一个非整数的方式不止一种。常说的小数是一个；循环小数算另一个，不过循环小数和分数是一回事；还有个连分数。当然还有些，比如开方，积分等等都可以表示一个“小数”。

基于准确性和描述的简略性，big_udec选用分数表示一个小数。只需要增加位数就可以无限制的提高精度，而分数本身也只需要两个big_uint就可以表示。[service/math/big_udec/big_udec.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.vb)

```vb.net
' A positive decimal in range of [0, +inf) by the representation of n / d.
Partial Public NotInheritable Class big_udec
    Private ReadOnly n As big_uint
    Private ReadOnly d As big_uint
```

big_udec需要的功能除了输入输出、常见的运算、比较以外，还有一个重要的约分操作。

- 输入输出

常见的数据类型，比如double single

分数表示，比如一对big_uint或者一对uint32，int32等

字节数组

字符串小数，支持任意进制和任意小数位数。当然也可以考虑检测循环小数，不过还没有实现，π不是循环小数。

- 运算

加减乘数、次方、开方，还有约分。因为之前提到的最大公约数算法相当耗时，所以big_udec并不总是最简分数。

- 比较

也如big_uint，剩下还有一些和.Net相关的设计，比如重载ToString，Equals，运算符等。以及为测试设计的函数，返回一个随机big_udec的random函数等。
