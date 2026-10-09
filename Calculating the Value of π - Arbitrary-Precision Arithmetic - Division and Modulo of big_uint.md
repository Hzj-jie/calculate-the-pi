# Calculating the Value of π - Arbitrary-Precision Arithmetic - Division and Modulo of big_uint

For integers, the result of division consists of a quotient and a remainder. Therefore, division and modulo are essentially the same operation: the former needs to store the quotient, while the latter does not.

Do you remember the term "trial division" (estimating quotient digits)? Each digit of the quotient is determined by trial from the most significant digit to the least significant digit. Trying a candidate value for a given digit always leads to one of three outcomes:

- The divisor multiplied by this digit is greater than the dividend, so this digit should be decreased.
- The divisor multiplied by this digit is less than the dividend, so the true digit might be larger than the current value.
- The divisor multiplied by this digit equals the dividend, so this digit should take this exact value, no further calculation is needed, and all subsequent digits of the quotient are 0.

Of course, textbooks rarely mention that overestimating a quotient digit is actually not a problem, because the remainder can be negative. For example:

```
Divisor     Quotient (Digit)  Divisor × Quotient  Remainder   Left Shift of Divisor
24                                                10000       100
            500               12000               -2000        10
            -90               -2160                 160         1
              6                 144                  16
-----------------------------------------------------------------------------------
            416                                      16
```

(The initial dividend can also be regarded as the initial remainder.)

If we handle this using only positive numbers and a borrow flag, the above calculation can be expressed as:

```
Divisor     Quotient (Digit)  Divisor × Quotient  Remainder   Left Shift of Divisor  Borrow
24                                                10000       100                    0
            500               12000                2000        10                    1
             90                2160                 160         1                    0
              6                 144                  16                              0
-------------------------------------------------------------------------------------------
            416                                      16
```

When there is a borrow, the new remainder is (Divisor × Quotient - Remainder) rather than the usual (Remainder - Divisor × Quotient). However, this approach introduces many edge cases—for example, when there is a borrow, the chosen quotient digit must make its product with the divisor greater than the remainder, whereas when there is no borrow, the product must be less than the remainder. Thus, this approach increases code complexity without reducing time complexity. Because of the subtraction, the time complexity is O(N^3).

Therefore, in the actual implementation, we want to avoid the first case where the product of the divisor and the quotient digit exceeds the remainder. On the other hand, the second case—where the product is less than the remainder—is not a big issue; it merely requires an additional multiplication and subtraction to add to the quotient digit. Let's revisit the example above to illustrate this case:

```
Divisor     Quotient (Digit)  Divisor × Quotient  Remainder   Left Shift of Divisor
24                                                10000                100
            400               9600                  400                 10
             10                240                  160                  1
              5                120                   40                  1
              1                 24                   16
-----------------------------------------------------------------------------------
            416                                      16
```

At the least significant digit, the first trial uses 5, leaving a remainder of 40, which is still greater than the divisor 24. So a second trial of 1 is performed, finally yielding a remainder of 16.

The significance of this approach is that when both the remainder and the divisor are extremely large, it is impossible to estimate the exact quotient digit in one shot, so we use a conservative estimate to guarantee that the product of the divisor and the quotient digit is always less than the remainder.

Consequently, how we estimate the quotient digit is a critical step affecting performance. For example, if there is a 90% probability of getting the exact value on the first try, then E=1.1*N*N*N. If that probability is improved to 95%, then E=1.05*N*N*N.

So first, we must guarantee that the product of the divisor and the quotient digit does not exceed the remainder. Suppose we divide Axxxxx by Bxxxxx, where A and B are each a single decimal digit and xxxxx represents another five digits—or, if there were 50,000 digits, we obviously could not inspect all of them. The quotient digit can only be determined by A and B. Then it is easy to see that this trial quotient digit is always less than or equal to A/(B+1).

With this, the fundamental algorithm is clear—of course, we need to translate our base-10 intuition into base 4294967296.

- Left-shift the divisor so that the divisor and the remainder (dividend) have the same number of digits.
- Repeat the step below until all left-shifted positions of the divisor have been shifted back to the right.

[service/math/big_uint/big_uint.divide.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.divide.vb)

```
    Private Shared Sub divide_uint(ByVal that As big_uint,
                                   ByVal remainder As big_uint,
                                   ByVal result As big_uint)
        Dim i As UInt32 = 0
        i = remainder.uint32_size() - that.uint32_size()
        If Not result Is Nothing Then
            result.v.resize(i + uint32_1)
        End If
        that.left_shift(CULng(i) << bit_count_in_uint32_shift)
        While True
            While True
                Dim n As UInt64 = 0
                Dim d As UInt64 = 0
                select_significant_divide_fraction(remainder, that, n, d)
                If n < d Then
                    Exit While
                End If
                If n = d Then
                    Dim cmp As Int32 = 0
                    cmp = that.compare(remainder)
                    If cmp <= 0 Then
                        remainder.assert_sub(that)
                        If Not result Is Nothing Then
                            result.recursive_add(uint32_1, i)
                        End If
                        If cmp = 0 Then
                            that.right_shift(CULng(i) << bit_count_in_uint32_shift)
                            Return
                        End If
                    End If
                    Exit While
                End If
                n \= (d + uint64_1)
                Dim t32 As UInt32 = 0
#If DEBUG Then
                t32 = assert_which.of(n).can_cast_to_uint32()
#Else
                t32 = CUInt(n)
#End If
                If Not result Is Nothing Then
                    result.recursive_add(t32, i)
                End If
                remainder.assert_sub(that * t32)
                If remainder.uint32_size() < that.uint32_size() Then
                    Exit While
                End If
            End While

            If i = 0 Then
                Return
            End If

            that.right_shift(CULng(bit_count_in_uint32))
            i -= uint32_1
        End While
    End Sub
```

There are also several boundary cases to handle: for example, if the dividend initially has fewer digits than the divisor, the dividend is directly returned as the remainder; if the divisor is a power of 2, division can be avoided altogether by using bit shifts and bitwise AND operations.

[service/math/big_uint/big_uint.calculation.vb#L110](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.calculation.vb#L110)

Because each digit of `big_uint` is a `uint` (`UInt32`) while a 64-bit processor natively supports `ulong` (`UInt64`) division, there is an optimization in `select_significant_divide_fraction`:

```
    Private Shared Sub select_significant_divide_fraction(ByVal remainder As big_uint,
                                                          ByVal that As big_uint,
                                                          ByRef n As UInt64,
                                                          ByRef d As UInt64)
#If DEBUG Then
        assert(remainder.uint32_size() >= that.uint32_size())
#End If
        If remainder.uint32_size() = that.uint32_size() Then
            If remainder.uint32_size() > 1 Then
                n = remainder.highest_uint64()
                d = that.highest_uint64()
                Return
            End If
            n = remainder.highest_uint32()
            d = that.highest_uint32()
            Return
        End If
        n = remainder.highest_uint64()
        d = that.highest_uint32()
    End Sub
```

If the divisor and the remainder have the same number of digits, and both have more than one digit, we extract the top two digits (`UInt64`) from both at the same time, which effectively improves the accuracy of estimating the quotient digit.

Additionally, just like multiplication, there is also a `divide_bit` implementation that replaces division with bit shifts, additions, and subtractions, which mirrors how division is implemented in hardware processors: [service/math/big_uint/big_uint.divide.vb#L56](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.divide.vb#L56)

The implementation of `divide(uint, uint&, bool&)` handles dividing by a single `uint` ([service/math/big_uint/big_uint.divide.vb#L13](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.divide.vb#L13)), which is equivalent to dividing a multi-digit number by a single-digit number. Its principle is even simpler and will not be elaborated further.

Modulo is simply the division operation with quotient accumulation omitted, so it needs no further explanation either.
