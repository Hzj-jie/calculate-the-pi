# Calculating the Value of π - Arbitrary-Precision Arithmetic - Multiplication of big_uint

Textbook multiplication consists of two parts: multiplying the multiplier by each digit of the multiplicand (strictly speaking, the order doesn't matter, so we use "multiplicand" here to refer to the second number), shifting each partial product according to its digit position, and summing all the partial products together.

In terms of time complexity, this is an $O(N^2)$ algorithm, or more precisely, it processes at least $(N + N) \times M$ digits, where $N$ is the length of the multiplier and $M$ is the length of the multiplicand. The first $N$ comes from multiplying the multiplier by a single digit of the multiplicand, and the second $N$ comes from adding that partial product into the running result. Clearly, doing these as separate passes is not the fastest approach—combining multiplication and addition into a single loop significantly improves performance.

Consider multiplying an $(n+m+1)$-digit number by an $(i+j+1)$-digit number, where the $(m+1)$-th digit of the multiplier is $A$ and the $(j+1)$-th digit of the multiplicand is $B$. The product $A \times B$ affects positions $m+j-1$ and $m+j$: the lower part at position $m+j-1$ is added directly to the result, while the upper part at position $m+j$ becomes the carry to be folded into the next digit's multiplication.

```
 {n digits}A{m digits}
*{i digits}B{j digits}
----------------------
...
A*B{m+j-2 digits}
...
```

Following the exact same logic, replacing base 10 with base 4,294,967,296 gives us `big_uint` multiplication.

```
    Private Sub multiply_uint(ByVal this As big_uint, ByVal that As big_uint)
        v.resize(this.v.size() + that.v.size())
        assert(this.v.size() > 0 AndAlso that.v.size() > 0)
        For i As UInt32 = 0 To this.v.size() - uint32_1
            If this.v.get(i) = 0 Then
                Continue For
            End If
            Dim c As UInt32 = 0
            For j As UInt32 = 0 To that.v.size() - uint32_1
                Dim t As UInt64 = 0
                t = this.v.get(i)
                t *= that.v.get(j)
                c = add(CUInt(t And max_uint32), c, i + j)  ' [1]
                c += CUInt(t >> bit_count_in_uint32)
            Next
#If DEBUG Then
            assert(v.get(i + that.v.size()) = 0)
#End If
            v.set(i + that.v.size(), c)  ' [2]
        Next
        remove_last_blank()
    End Sub
```

This function stores the product of `this` and `that` in the current object.

`[1]` is the helper mentioned in the addition article: it adds the first two arguments into the digit position specified by the third argument and returns the resulting carry. [service/math/big_uint/big_uint.private.vb#L67](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L67)

Because this is multiplication, the combined carry can be at most 2 (analogous to $9 \times 2 + 9$ in decimal).

`[2]` is a small optimization: since each outer loop iteration reaches position `i + that.v.size()` for the very first time, that slot in the result is guaranteed to be 0. Thus, we can simply assign the carry `c` directly instead of performing an addition.

Beyond this core routine, multiplying two `big_uint` instances only requires handling a few boundary cases—for example, if either operand is 0, the result is set to 0; if either operand is 1, the result is set to the other operand. [service/math/big_uint/big_uint.private.vb#L172](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L172)

There is also an interesting but much less efficient algorithm: treat `that` as a binary number, and whenever a bit is 1, add `this` to the result after shifting `this` to match the position of that 1-bit. This algorithm requires only addition and bit-shifting, which is actually how hardware processors perform multiplication under the hood (modern processors precompute $8 \times 8$ products and operate in base 256). However, bit-shifting a `big_uint` is fairly expensive, making this algorithm quite slow despite being based on the exact same principle.

```
    Private Sub multiply_bit(ByVal this As big_uint, ByVal that As big_uint)
        this = this.CloneT()
        that = that.CloneT()
        While Not that.is_zero()
            Dim m As UInt32 = 0
            m = that.remove_binary_trailing_zeros()
            assert(that.odd())
            this.left_shift(m)
            add(this)
        End While
    End Sub
```

[service/math/big_uint/big_uint.private.vb#L134](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L134)

Since the upcoming division algorithm heavily relies on `big_uint * uint` operations, multiplication also provides an overload for `UInt32`.

```
    Public Function multiply(ByVal that As UInt32) As big_uint
        If that = 0 Then
            set_zero()
            Return Me
        End If
        If that = 1 OrElse is_zero() Then
            Return Me
        End If
        If is_one() Then
            replace_by(that)
        End If
        Dim c As UInt32 = 0
        For i As UInt32 = 0 To v.size() - uint32_1
            Dim t As UInt64 = 0
            t = v.get(i)
            t *= that
            t += c
            v.set(i, CUInt(t And max_uint32))
            c = CUInt(t >> bit_count_in_uint32)
        Next
        If c <> 0 Then
            v.push_back(c)
        End If
        remove_last_blank()
        Return Me
    End Function
```

The basic operation is identical, except that no extra addition into an existing result array is needed—each digit can simply be assigned in place.
