# 计算π的值 - 无精度限制的计算 - big_udec的比较

作为一个分数，如果分子是0，这个分数就是0：[service/math/big_udec/big_udec.vb#L182](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.vb#L182)

```vb.net
    Public Function is_zero() As Boolean
        Return n.is_zero()
    End Function
```

如果分数的分子和分母相同，就是1：[service/math/big_udec/big_udec.vb#L186](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.vb#L186)

```vb.net
    Public Function is_one() As Boolean
        Return n.equal(d)
    End Function
```

同理，比较两个分数就是分别比较分子和分母：[service/math/big_udec/big_udec.compare.vb#L25](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.compare.vb#L25)

```vb.net
    Public Shared Function compare(ByVal this As big_udec, ByVal that As big_udec) As Int32
        Dim c As Int32 = 0
        c = object_compare(this, that)
        If c <> object_compare_undetermined Then
            Return c
        End If

        assert(Not this Is Nothing)
        assert(Not that Is Nothing)

        If this.is_zero() AndAlso that.is_zero() Then
            Return 0
        End If
        If this.is_zero() Then
            Return -1
        End If
        If that.is_zero() Then
            Return 1
        End If

        If this.d > that.d AndAlso this.n <= that.n Then
            Return -1
        End If
        If this.d < that.d AndAlso this.n >= that.n Then
            Return 1
        End If
        If this.d >= that.d AndAlso this.n < that.n Then
            Return -1
        End If
        If this.d <= that.d AndAlso this.n > that.n Then
            Return 1
        End If
        If this.d = that.d AndAlso this.n = that.n Then
            Return 0
        End If

        Return (this.n * that.d).compare(this.d * that.n)
    End Function
```

因为big_udec不一定是最简分数，所以可能最终需要比较交叉相乘的结果。
