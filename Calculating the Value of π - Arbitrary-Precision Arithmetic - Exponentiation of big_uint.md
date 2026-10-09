# Calculating the Value of π - Arbitrary-Precision Arithmetic - Exponentiation of big_uint

Exponentiation is not actually used when calculating π, as we will see later, but it is still a meaningful feature for `big_uint`.

Exponentiation is simply multiplying a number by itself many times, though we need to consider how to do it efficiently. First, multiplying an A-digit number by a B-digit number produces a result of at most A+B digits. Now consider raising an N-digit number to the M-th power: the result has at most N*M digits, and the number of single-digit multiplications required by repeated multiplication is $\sum_{1}^{M}{x} \times N ^ 2 = (1 + M) \times M / 2 \times N ^ 2$. This is obviously a very inefficient approach, just as nobody would compute A × B by turning it into $\sum_{1}^{B}{A}$.

Therefore, we first need to bound the computational complexity in terms of the number of bits of M rather than the magnitude of M itself, expressing the workload using N and $log_2M$. Simply put, we replace repeated multiplication with a sequence of squarings and multiplications. Although squaring is also multiplication, we can prove that the total number of digit multiplications is significantly reduced. For simplicity, let $M' = log_2M$, and consider the worst-case complexity where every bit of M is 1.

First, let's calculate the number of digit multiplications required for M' squarings—note that this is independent of whether a given bit of M is 1. Squaring a 1-digit number requires 1 digit multiplication and produces a 2-digit result; squaring a 2-digit number requires 4 digit multiplications and produces a 4-digit result; and so on. The total is $1 + 4 + 16 + ... + 2 ^ {2 \times (M' - 1)} = \sum_{0}^{M' - 1}{2 ^ {(x \times 2)}} = 2 ^ {(2 \times M' - 1)} - 1$.

If the initial number has N digits, as in the earlier example, then squaring an N-digit number takes N^2 digit multiplications and produces a 2N-digit result; squaring a 2N-digit number takes 4N^2 multiplications and produces a 4N-digit result; and so on. The formula for the total simply needs to be multiplied by N^2: $N^2 + (2 \times N) ^ 2 + (4 \times N)^2 + ... = N ^ 2 \times (2 ^ {(2 \times M' - 1)}- 1)$

Meanwhile, each time we multiply the squared value into the accumulated result, the number of digits in the accumulated result is at most half the number of digits in the current square, so the number of digit multiplications is $1 \times N + N \times 2 \times N + 3 \times N \times 4 \times N + 7 \times N \times 16 \times N + ... \leq N ^ 2 \times 2 ^ {(2 \times M' - 2)}$

Thus, the grand total is $\leq N^2 \times 2 ^ {(2 \times M' - 2)} \times 3$

Notice that since $2 ^ {M'}$ is on the order of M, this theoretical bound is already smaller than multiplying M times repeatedly. Furthermore, because multiplying an N-digit number by an N-digit number does not always reach the full 2N digits in practice, the actual computational complexity is even lower.

---

For example, consider $11 ^ 5$ using the two approaches above:

$11 \times 11 \times 11 \times 11 \times 11$, where the number of digit multiplications is $2 \times 2 + 3 \times 2 + 4 \times 2 + 5 \times 2 = 28$

versus

$11 \times 11 ^ {2 ^ 2}$, where the number of digit multiplications is $(2 \times 2) + (3 \times 3) + 5 \times 2 = 23$.

---

Let's take a more extreme example, $99 ^ 8$:

$99 \times 99 \times 99 \times 99 \times 99 \times 99 \times 99 \times 99$, where the number of digit multiplications is $2 \times 2 + 4 \times 2 + 6 \times 2 + 8 \times 2 + 10 \times 2 + 12 \times 2 + 14 \times 2 = 112$

versus

${{99 ^ 2 } ^ 2}^2$, where the number of digit multiplications is $2 \times 2 + 4 \times 4 + 8 \times 8 = 84$.

---

And another extreme example, $55 ^ 7$:

$55 \times 55 \times 55 \times 55 \times 55 \times 55 \times 55$, where the number of digit multiplications is $2 \times 2 + 4 \times 2 + 6 \times 2 + 7 \times 2 + 9 \times 2 + 11 \times 2 = 78$

versus

$55 \times 55 ^ 2 \times 55 ^ 4$—noting that when computing the fourth power we reuse the square we already computed—where the number of digit multiplications is $2 \times 2 + 4 \times 4 + 2 \times 4 + 6 \times 7 = 70$

---

In addition, the squaring operation enjoys a memory advantage, as it does not require an extra temporary variable to hold an operand.

What remains are boundary cases, such as $X ^ 0 = 1$ and $0 ^ x = 0 (x > 0)$.

The implementation details can be found at [service/math/big_uint/big_uint.calculation.vb#L369](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.calculation.vb#L369)

```
    Public Function power(ByVal that As big_uint) As big_uint
        If that Is Nothing OrElse that.is_zero() Then
            '0 ^ 0 = 1
            set_one()
            Return Me
        End If
        If is_zero_or_one() OrElse that.is_one() Then
            Return Me
        End If
        that = that.CloneT()
        For i As UInt32 = 1 To that.remove_binary_trailing_zeros()
            power_2()
        Next
        Dim c As big_uint = Nothing
        c = Me.CloneT()
        assert(that.bit_count() > 0)
        Dim last As UInt64 = 0
        For i As UInt64 = 1 To that.bit_count() - uint64_1
            If Not that.getrbit(i) Then
                Continue For
            End If
            While last < i
                c.power_2()
                last += uint64_1
            End While
            multiply(c)
        Next
        Return Me
    End Function
```
