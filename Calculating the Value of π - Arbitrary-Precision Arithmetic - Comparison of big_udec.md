# Calculating the Value of π - Arbitrary-Precision Arithmetic - Comparison of big_udec

As a fraction, if the numerator is 0, the fraction itself is 0: [service/math/big_udec/big_udec.vb#L182](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.vb#L182)

```vb.net
    Public Function is_zero() As Boolean
        Return n.is_zero()
    End Function
```

If the numerator and denominator of the fraction are equal, the fraction is 1: [service/math/big_udec/big_udec.vb#L186](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.vb#L186)

```vb.net
    Public Function is_one() As Boolean
        Return n.equal(d)
    End Function
```

Similarly, comparing two fractions comes down to comparing their numerators and denominators: [service/math/big_udec/big_udec.compare.vb#L25](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.compare.vb#L25)

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

Because `big_udec` is not necessarily an irreducible fraction, it may ultimately be necessary to compare the cross-multiplied results.
