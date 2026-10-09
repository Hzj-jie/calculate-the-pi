# 计算π的值 - 无精度限制的计算 - big_udec的乘除法次方

因为big_udec也支持0，所以除法也可能会有除0错误，所以也有三个重载：

```vb.net
    Public Function divide(ByVal that As big_udec, ByRef divide_by_zero As Boolean) As big_udec
    Public Function divide(ByVal that As big_udec) As big_udec
    Public Function assert_divide(ByVal that As big_udec) As big_udec
```

除此以外乘除运算其实并没有太多可说的，不过就是交叉相乘罢了。和加减法一样，使用fast_reduce_fraction为两个分数的分子和分母分别约分。[service/math/big_udec/big_udec.calculation.vb#L120](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.calculation.vb#L120)

```vb.net
    Public Function multiply(ByVal that As big_udec) As big_udec
        If that Is Nothing OrElse that.is_zero() Then
            set_zero()
            Return Me
        End If
        If is_zero() Then
            Return Me
        End If

        Dim n1 As big_uint = Nothing
        Dim n2 As big_uint = Nothing
        Dim d1 As big_uint = Nothing
        Dim d2 As big_uint = Nothing
        n1 = Me.n.CloneT()
        n2 = that.n.CloneT()
        d1 = Me.d.CloneT()
        d2 = that.d.CloneT()
#If REDUCE_FRACTION_WHEN_CALCULATING Then
        reduce_fraction(n1, d2)
        reduce_fraction(n2, d1)
#Else
        fast_reduce_fraction(n1, d2)
        fast_reduce_fraction(n2, d1)
#End If
        replace_by(n1.multiply(n2), d1.multiply(d2))

#If Not REDUCE_FRACTION_WHEN_CALCULATING Then
        increase_fraction_dirty_rate()
#End If
        Return Me
    End Function

    Public Function divide(ByVal that As big_udec, ByRef divide_by_zero As Boolean) As big_udec
        If that Is Nothing OrElse that.is_zero() Then
            divide_by_zero = True
            Return Me
        End If
        divide_by_zero = False
        If is_zero() Then
            Return Me
        End If

        Dim n1 As big_uint = Nothing
        Dim n2 As big_uint = Nothing
        Dim d1 As big_uint = Nothing
        Dim d2 As big_uint = Nothing
        n1 = Me.n.CloneT()
        n2 = that.n.CloneT()
        d1 = Me.d.CloneT()
        d2 = that.d.CloneT()
#If REDUCE_FRACTION_WHEN_CALCULATING Then
        reduce_fraction(n1, n2)
        reduce_fraction(d1, d2)
#Else
        fast_reduce_fraction(n1, n2)
        fast_reduce_fraction(d1, d2)
#End If
        replace_by(n1.multiply(d2), d1.multiply(n2))

#If Not REDUCE_FRACTION_WHEN_CALCULATING Then
        increase_fraction_dirty_rate()
#End If
        Return Me
    End Function
```

次方运算也相当简单，就是把分子和分母各计算次方即可。[service/math/big_udec/big_udec.calculation.vb#L203](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.calculation.vb#L203)

```vb.net
    Public Function power(ByVal that As big_uint) As big_udec
        If that Is Nothing OrElse that.is_zero() Then
            set_zero()
            Return Me
        End If
        If that.is_one() Then
            Return Me
        End If

        replace_by(Me.n ^ that, Me.d ^ that)
        Return Me
    End Function
```
