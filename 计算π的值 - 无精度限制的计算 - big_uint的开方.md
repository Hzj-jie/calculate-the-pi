# 计算π的值 - 无精度限制的计算 - big_uint的开方

同样，开方运算也不是必须的，只是big_uint应该支持的一个功能。这里的开方不是单指开平方，而是任意次方。并且因为精度的要求和实现的阶段，开方的结果也不是一个小数，而是两个整数，一个是根，另一个是余数。比如说

$\sqrt[5]{130} = 2 .. 88$ ， $\sqrt[3]{28} = 3 .. 1$

开方可以用到牛顿法，不过那需要用到小数或者分数，暂时还没有实现到。所以用个简单的尝试法来计算开方，和除法很接近。

- 首先估计一下二进制的结果的位数。如次方的时候所说，N位数的M次方不会超过N×M位。这个规则和进制无关，比如说 $(11_{2}) ^ 5 = (11110011_{2})$ 或者 $(10_{2}) ^ 7 = (10000000_2)$ 。所以结果的位数不超过N / M + 1。
- 再从高位开始尝试每一个二进制位的值。把一位设为1的时候如果结果大于开方数，则取0，否则取1。
- 最后把原开方数减去根的次方就得到余数了。

找个稍微复杂点的例子，求 $\sqrt[4]{935}$

$(935_{10}) = (1110100111_2)$

开4次方，结果不超过10 / 4 + 1 = 3位。

- $(100_2)^4=256 < 935$ 第一位取1。
- $(110_2)^4 = 1296 > 935$ 第二位取0。
- $(101_2)^4=625<935$ 第三位取1。

所以结果是 $(101_2) = (5_{10})$ ，余数是935-625=310。

再余下的就是一些边界情况。比如开0次方时只许被开方数是1，任何数的0次方都是1，否则设置divide_by_zero错误；根次是1的时候，直接返回。[service/math/big_uint/big_uint.private.vb#L210](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L210)

```
    Private Sub extract(ByVal that As big_uint, ByRef remainder As big_uint, ByRef divide_by_zero As Boolean)
        If that Is Nothing OrElse that.is_zero() Then
            If is_one() Then
                divide_by_zero = False
                remainder = zero()
            Else
                divide_by_zero = True
            End If
            Return
        End If
        divide_by_zero = False
        If that.is_one() Then
            remainder = zero()
            Return
        End If
        If is_zero() OrElse is_one() Then
            remainder = zero()
            Return
        End If
        Dim r As big_uint = Nothing
        r = New big_uint(bit_count())
        r.divide(that, remainder)
        Dim l As UInt64 = 0
        If remainder.is_zero() Then
            l = r.as_uint64()
        Else
            l = r.as_uint64() + uint64_1
        End If
        assert(l > 0)
        r.set_zero()
        r.set_bit_count(l)
        For i As UInt64 = 0 To l - uint64_1
            r.setrbit(l - i - uint64_1, True)
            If r ^ that > Me Then
                r.setrbit(l - i - uint64_1, False)
            End If
#If DEBUG Then
            assert(r ^ that <= Me)
#End If
        Next
        remainder = New big_uint(Me - r ^ that)
        assert(replace_by(r))
    End Sub
```
