# Calculating the Value of π - Arbitrary-Precision Arithmetic - Addition and Subtraction of big_udec

Just like `big_uint`, because `big_udec` is an unsigned fraction, subtraction has three overloads:

```vb.net
    Public Function [sub](ByVal that As big_udec, ByRef overflow As Boolean) As big_udec
    Public Function [sub](ByVal that As big_udec) As big_udec
    Public Function assert_sub(ByVal that As big_udec) As big_udec
```

The first returns the overflow status, the second throws an exception on overflow, and the third fails an assertion on overflow.

Just as taught in textbooks, in the result of addition or subtraction, the denominator is the product of the two denominators, and the numerator is the sum or difference of the cross-multiplied terms. Generally speaking, fraction reduction is needed before or after the operation, but as mentioned in the `big_uint` articles, both division and finding the greatest common divisor are very time-consuming operations. As a compromise, only the simplest common factors are considered. First and foremost is naturally 2: as mentioned earlier, dividing by 2 is a very lightweight operation, and checking whether a number is divisible by 2 is equally lightweight. We only need to count how many trailing binary zeros a number has, and division is simply a right shift to remove those trailing binary zeros. [service/math/big_uint/big_uint.bit.vb#L209](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.bit.vb#L209)

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

In addition to 2, I also previously experimented with the prime numbers 3 and 5, but statistically, adding checks for these two primes actually slowed things down. This makes sense: assuming the numerator and denominator are random, the probability that both are divisible by 2 is 25%, whereas the probability that both are divisible by 3 drops to 11.1%, and the probability that both are divisible by 5 is only 4%.

Therefore, during routine operations—not just addition and subtraction, but also multiplication and division—`big_udec` only uses `fast_reduce_fraction` to reduce the lengths of the numerator and denominator. [service/math/big_udec/big_udec.reduce_fraction.vb#L31](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.reduce_fraction.vb#L31)

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

Besides checking for divisibility by 2, this function also handles several simple edge cases, such as when the numerator is 0 or when the numerator and denominator are equal.

Of course, reducing a fraction to its lowest terms is still a meaningful operation. As will be discussed later when calculating π, its fractional representation becomes enormous, and due to the nature of the calculation, reducible common factors will inevitably arise. Therefore, a function to reduce the fraction to its lowest terms using the greatest common divisor is also provided. [service/math/big_udec/big_udec.reduce_fraction.vb#L56](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.reduce_fraction.vb#L56)

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

There is also a mechanism in the implementation to automatically reduce fractions based on computational cost, though it is not currently used. The reason will be explained later when discussing how to calculate π and e.

Finally, addition and subtraction themselves do not require much further elaboration, apart from an optimization when the two fractions share the same denominator. [service/math/big_udec/big_udec.calculation.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.calculation.vb)

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
