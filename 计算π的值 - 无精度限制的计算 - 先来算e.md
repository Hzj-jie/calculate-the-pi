# 计算π的值 - 无精度限制的计算 - 先来算e

e的算法逼近的比较快，所以先来算e的值，碰到的问题少一些。

首先计算出来的e的值的结果是一个分数，当然big_udec本身就是一个分数。再通过str_match函数和已知的e的值进行比较判断精度。

计算e的方法主要有两种：

${(1 + \frac{1}{x})}^{x}$ ，当x的值增大的时候精度提高。

$1 + \sum_{x=1}^{N}{\frac{1}{x!}}$ ，当N的值增大的时候精度提高。

第一种做法在选择了x之后精度就确定了，而第二种方法可以通过迭代提高精度。

先看看第一种做法：

```vb.net
    <test>
    Private Shared Sub calculate_e_power_1000()
        Dim s As big_udec = Nothing
        s = big_udec.one()
        s.add(big_udec.fraction(CUInt(1), CUInt(1000)))
        s.power(CUInt(1000))
        assertion.equal(s.str(), "2.7169239322358924573830881219475771889643150188365728037223547748")
        '                         2.71828182845904523536028747135266249775724709369995957
    End Sub
```

当x为1000的时候，精度只能达到2位；100000的时候可以达到4位。

而第二种做法因为是迭代式的，所以经过若干轮后可以输出中间值和结论：

```vb.net
    <command_line_specified>
    <test>
    Private Shared Sub calculate_e_factorial_max_uint32_progressively()
        Dim s As big_udec = Nothing
        s = big_udec.one()
        Dim c As big_udec = Nothing
        c = big_udec.one()
        For i As UInt32 = 1 To max_uint32 - uint32_1
            c.assert_divide(New big_udec(i))
            s.add(c)

            If (i Mod 100000) = 0 Then
                c.reduce_fraction()
                s.reduce_fraction()
                raise_error(error_type.warning, "@ ", i, " -> ", s.fractional_str(), " : c -> ", c.fractional_str())
            End If
        Next
    End Sub
```

[service/tests/math/big_udec/big_udec_e_test.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/math/big_udec/big_udec_e_test.vb)

在我的机器上大约4小时15分钟可以算400000轮，精度已经超过了前两百万位。[service/resources/math/codegen/e-2m.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/resources/math/codegen/e-2m.txt)

大约5天计算了1600000轮，按照之前的规律估计精度结果超过了八百万位。

这几个big_udec的分数表示相当的大，没法贴在这里，移步github。[e.2m.txt](https://github.com/Hzj-jie/calculate-the-pi/blob/master/e.2m.txt) 大约4M，[e.8m.txt](https://github.com/Hzj-jie/calculate-the-pi/blob/master/e.8m.txt) 大约18M。这估计是互联网上能找到的最接近e的分数表示。
