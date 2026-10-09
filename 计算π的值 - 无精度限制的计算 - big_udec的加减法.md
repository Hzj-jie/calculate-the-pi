# 计算π的值 - 无精度限制的计算 - big_udec的加减法

和big_uint一样，因为是无符号的分数，所以减法也有三个重载：

```vb.net
    Public Function [sub](ByVal that As big_udec, ByRef overflow As Boolean) As big_udec
    Public Function [sub](ByVal that As big_udec) As big_udec
    Public Function assert_sub(ByVal that As big_udec) As big_udec
```

第一个返回溢出状态，第二个在溢出时抛出异常，第三个在溢出时断言失败。

和教科书上说的一样，加减法的结果中分母就是分母相乘，分子是交叉相乘的和或差。一般来说之前或者之后需要约分，不过如big_uint中提到的，除法和求最大公约数都是非常耗时的操作，所以退而求其次，只考虑几个最简单的公约数。首当其冲的自然是2，如前所提到的，除2是个非常轻量的操作，而判断一个数是否可以被2整除也是很轻量的。只需要考虑一个数的结尾有几个二进制的0即可，而除法也只是通过移位去掉末尾的二进制0。[service/math/big_uint/big_uint.bit.vb#L209](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.bit.vb#L209)

```vb.net
    Public Function trailing_binary_zero_count() As UInt32
        assert(Not is_zero())
        Dim i As UInt32 = 0
        While i < uint32_size()
            If v.get(i) <> 0 Then
                Return (i << bit_count_in_uint32_shift) + v.get(i).trailing_binary_zero_count()
            End If
            i += uint32_1
        End While
        assert(False)
        Return uint32_0
    End Function

    Public Function remove_trailing_binary_zeros() As UInt32
        Dim r As UInt32 = 0
        r = trailing_binary_zero_count()
        assert_right_shift(r)
        Return r
    End Function
```

除此以外，之前还尝试过3和5两个质数，不过统计意义上增加这两个质数的检查反倒会降低速度。这也是很合理的，假设分子和分母是随机的，那么两者都可以被2整除的几率为25%，而都可以被3整除的几率则降低到了11.1%，都可以被5整除的几率则为4%。

所以在常规的操作中，不光是加减法，还是乘除的时候，big_udec都只使用fast_reduce_fraction操作来降低分子和分母的长度。[service/math/big_udec/big_udec.reduce_fraction.vb#L31](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.reduce_fraction.vb#L31)

```vb.net
    Private Shared Function fast_reduce_fraction(ByVal n As big_uint, ByVal d As big_uint) As Boolean
        assert(Not n Is Nothing)
        assert(Not d Is Nothing)

        If n.is_zero() Then
            d.set_one()
            Return True
        End If
        If n.is_one() OrElse d.is_one() Then
            Return True
        End If
        If n.equal(d) Then
            n.set_one()
            d.set_one()
            Return True
        End If

        Dim m As UInt32 = 0
        m = min(n.trailing_binary_zero_count(), d.trailing_binary_zero_count())
        n.right_shift(m)
        d.right_shift(m)

        Return False
    End Function
```

这里除了考虑整除2以外，还有就是一些非常简单的边缘情况，比如说分子是0的时候，分子分母相等的时候。

当然约分至最简分数也是有意义的操作，之后在计算π的时候会提到，其分数表示是非常巨大的，而因为计算方式的原因，其中显然会存在可以约分的情况。所以也提供一个使用最大公约数约分至最简分数的功能。[service/math/big_udec/big_udec.reduce_fraction.vb#L56](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.reduce_fraction.vb#L56)

```vb.net
        Using code_block
            Dim b As big_uint = Nothing
            b = big_uint.gcd(n, d)
            Dim c As big_uint = Nothing
            n.assert_divide(b, c)
            assert(c.is_zero())
            d.assert_divide(b, c)
            assert(c.is_zero())
        End Using
```

实现中还有个根据计算量自动约分的设计，不过暂时没有用到。原因之后在讨论如何计算π和e的时候会再说明。

最后，加法和减法本身没有太多需要赘述的内容，只有一个两分数分母相同时的优化而已。[service/math/big_udec/big_udec.calculation.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.calculation.vb)

```vb.net
        If Me.d.equal(that.d) Then
            replace_by(Me.n + that.n, Me.d)
#If REDUCE_FRACTION_WHEN_CALCULATING Then
            reduce_fraction()
#End If
        Else
#If REDUCE_FRACTION_WHEN_CALCULATING Then
            Dim g As big_uint = Nothing
            g = big_uint.gcd(Me.d, that.d)
            Dim c As big_uint = Nothing
            Me.d.assert_divide(g, c)
            assert(c.is_zero())
            replace_by((that.d \ g) * Me.n + that.n * Me.d, that.d * Me.d)
#Else
            replace_by(that.d * Me.n + that.n * Me.d, that.d * Me.d)
#End If
        End If
```
