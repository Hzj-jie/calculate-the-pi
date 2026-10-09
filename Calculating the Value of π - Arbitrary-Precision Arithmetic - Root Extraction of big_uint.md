# Calculating the Value of π - Arbitrary-Precision Arithmetic - Root Extraction of big_uint

Likewise, root extraction is not strictly required for calculating π, but it is a feature that `big_uint` ought to support. Root extraction here does not just mean square roots, but arbitrary N-th roots. Moreover, given our precision requirements and current stage of implementation, the result of root extraction is not a floating-point or decimal number, but two integers: the integer root and the remainder. For example:

$\sqrt[5]{130} = 2 .. 88$, $\sqrt[3]{28} = 3 .. 1$

Root extraction can be done using Newton's method, but that requires decimals or fractions, which have not been implemented yet. Instead, we use a simple bitwise trial method to compute the root, which is very similar to division:

- First, estimate the number of bits in the binary result. As discussed in the exponentiation article, the M-th power of an N-digit number never exceeds N×M digits. This rule holds regardless of the base; for example, $(11_{2}) ^ 5 = (11110011_{2})$ or $(10_{2}) ^ 7 = (10000000_2)$. Therefore, the number of bits in the root does not exceed N / M + 1.
- Next, try each bit from the most significant bit down to the least significant bit. When setting a bit to 1, if the M-th power of the candidate root exceeds the radicand, set that bit back to 0; otherwise, keep it as 1.
- Finally, subtract the M-th power of the resulting root from the original radicand to obtain the remainder.

Let's walk through a slightly more involved example, finding $\sqrt[4]{935}$:

$(935_{10}) = (1110100111_2)$

Taking the 4th root, the result has at most 10 / 4 + 1 = 3 bits.

- $(100_2)^4=256 < 935$, so the first bit is 1.
- $(110_2)^4 = 1296 > 935$, so the second bit is 0.
- $(101_2)^4=625<935$, so the third bit is 1.

Thus, the root is $(101_2) = (5_{10})$, and the remainder is 935 - 625 = 310.

What remains are the boundary cases. For instance, when taking the 0th root, the radicand is only allowed to be 1 (since any number to the 0th power is 1); otherwise, a `divide_by_zero` error flag is set. When the root degree is 1, the function simply returns immediately. [service/math/big_uint/big_uint.private.vb#L210](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L210)

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
