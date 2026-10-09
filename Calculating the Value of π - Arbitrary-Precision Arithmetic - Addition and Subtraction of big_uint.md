# Calculating the Value of π - Arbitrary-Precision Arithmetic - Addition and Subtraction of big_uint

This is probably a part familiar to almost anyone who understands `big_uint`: addition and subtraction are among the most fundamental `big_uint` operations. Of course, as an unsigned integer ("u"int) that only represents non-negative numbers, the subtraction API has some differences.

First, let's outline the basic design principles of the `big_uint` API:

- `big_uint` is a class (reference type) rather than a structure (value type). This is a .NET-specific design choice and is not particularly tied to the core algorithm.
- `big_uint` itself is mutable: all public arithmetic methods modify the current instance in place and return `Me` (`this`). However, due to .NET constraints, all overloaded operators are static functions and always return a new instance. Returning `Me` from instance methods allows chaining multiple operations without unnecessary copying, for example:

```
big_uint b = big_uint.one().add(100).multiply(100).power(2);
```

- For arithmetic functions that may encounter runtime errors—such as subtraction (which can trigger an overflow/underflow error) or division (which can trigger a division-by-zero error)—there are always three variants: 1) returning the error status via an output parameter, 2) throwing an exception on error, and 3) asserting no error and terminating the program if an error occurs.

```
Public Function [sub](ByVal that As big_uint, ByRef overflow As Boolean) As big_uint
Public Function [sub](ByVal that As big_uint) As big_uint ' throws OverflowException
Public Function assert_sub(ByVal that As big_uint) As big_uint ' assert no overflow exception
```

Let's start with the relatively simple addition operation. Starting from the least significant digit, we add digit by digit while tracking the carry; if a carry remains after reaching the most significant digit, it is appended as a new most significant digit. This is the exact method taught in elementary math textbooks. In arbitrary-precision arithmetic, however, each "digit" (or limb) is in base 4,294,967,296. Therefore, adding each "digit" requires a `uint64`. This is easy to understand: in decimal addition, adding two single digits can also produce a two-digit result—one for the current position and one for the carry. Similarly, in a `uint64`, the lower 32 bits form the current "digit" and the upper 32 bits form the carry.

There are a few additional implementation details to consider. The digit-by-digit addition only needs to iterate up to the length of the shorter `big_uint`; for the remaining higher digits of the longer operand, we simply copy them into the result and propagate the final carry. Furthermore, if either operand is 0, the result is simply the other operand. [service/math/big_uint/big_uint.calculation.vb#L14](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.calculation.vb#L14)

```
    Public Function add(ByVal that As big_uint) As big_uint
        If that Is Nothing OrElse that.is_zero() Then
            Return Me
        End If
        If is_zero() Then
            assert(replace_by(that))
            Return Me
        End If
        assert(v.size() > 0 AndAlso that.v.size() > 0)
        Dim i As UInt32 = 0
        Dim c As UInt32 = 0
        For i = 0 To min(v.size(), that.v.size()) - uint32_1
            c = add(that.v.get(i), c, i)  ' [1]
#If DEBUG Then
            assert(c = 0 OrElse c = 1)
#End If
        Next

        If v.size() < that.v.size() Then
            v.resize(that.v.size())
            memcpy(v.data(), i, that.v.data(), i, that.v.size() - i)
        End If
        recursive_add(c, i)  ' [2]
        Return Me
    End Function
```

Here, `[1]` sets the $i$-th digit to the lower 32 bits of `that.v.get(i) + v.get(i) + c` and returns the upper 32 bits as the carry. Because the carry in addition is at most 1, even when summing three values (`v.get(i)`, `that.v.get(i)`, and `c`), the resulting carry is still at most 1. However, because `add(uint, uint, uint)` is also reused in multiplication, its carry can be as large as 2 in general. [service/math/big_uint/big_uint.private.vb#L67](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L67)

`[2]` adds the carry `c` to the $i$-th digit, propagating any resulting carry into higher digits until no carry remains. [service/math/big_uint/big_uint.private.vb#L73](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L73)

Subtraction likewise proceeds from the least significant digit to the most significant digit, subtracting digit by digit and tracking the borrow. If a borrow is still required past the most significant digit, an overflow (underflow) error has occurred. Since each digit step in subtraction computes $+a - b - c$, the borrow is at most 1.

There are similar implementation considerations for subtraction. Even if a digit of the minuend is 0, we still need to subtract the corresponding digit of the subtrahend from 4,294,967,296, so the primary loop always runs across all digits of the subtrahend. If a non-zero borrow remains afterward, it is propagated through the remaining higher digits of the minuend. Also, if the subtrahend is 0, the minuend is returned immediately.

```
    Public Function [sub](ByVal that As big_uint, ByRef overflow As Boolean) As big_uint
        If that Is Nothing OrElse that.is_zero() Then
            overflow = False
            Return Me
        End If
        If that.v.size() > v.size() Then
            v.resize(that.v.size())
        End If
        assert(v.size() > 0 AndAlso that.v.size() > 0)
        Dim i As UInt32 = 0
        Dim c As UInt32 = 0
        For i = 0 To that.v.size() - uint32_1
            c = [sub](that.v.get(i), c, i)  ' [1]
        Next
        While i < v.size()
            c = [sub](c, i)  ' [2]
            If c = 0 Then
                Exit While
            End If
            i += uint32_1
        End While
        overflow = (c = uint32_1)
        remove_extra_blank()
        Return Me
    End Function
```

Here, `[1]` sets the $i$-th digit to the lower 32 bits of `v.get(i) - that.v.get(i) - c`, returning a borrow of 1 if the difference is negative, or 0 otherwise. [service/math/big_uint/big_uint.private.vb#L35](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L35)

`[2]` sets the $i$-th digit to the lower 32 bits of `v.get(i) - c`, returning a borrow of 1 if the result is negative, or 0 otherwise. [service/math/big_uint/big_uint.private.vb#L42](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L42)

In the latest implementation, the loop around `[2]` has been consolidated into a [recursive_sub](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L41) helper function mirroring `recursive_add`. This makes the helper functions for addition and subtraction completely symmetric, and both pairs of helpers are also reused in division.

```
    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Function set_and_borrow(ByVal t As Int64, ByVal p As UInt32) As UInt32
        v.set(p, CUInt(t And max_uint32))
        Return If(t < 0, uint32_1, uint32_0)
    End Function

    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Sub sub_assertions(ByVal c As UInt32, ByVal p As UInt32)
        'this assert is too costly
#If DEBUG Then
        assert(p < v.size())
        assert(c = uint32_0 OrElse c = uint32_1)
#End If
    End Sub

    'sub d at position p with carry-over as c
    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Function [sub](ByVal d As UInt32, ByVal c As UInt32, ByVal p As UInt32) As UInt32
        sub_assertions(c, p)
        Return set_and_borrow(CLng(v.get(p)) - d - c, p)
    End Function

    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Function recursive_sub(ByVal c As UInt32, ByVal p As UInt32) As Boolean
        If c = 0 Then
            Return False
        End If
        While p < v.size()
            sub_assertions(c, p)
            c = set_and_borrow(CLng(v.get(p)) - c, p)
            If c = 0 Then
                Exit While
            End If
            p += uint32_1
        End While
        Return (c = uint32_1)
    End Function

    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Sub add_assertions(ByVal p As UInt32)
        'this assert is too costly
#If DEBUG Then
        assert(p < v.size())
#End If
    End Sub

    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Function set_and_carry(ByVal t As UInt64, ByVal p As UInt32) As UInt32
        v.set(p, CUInt(t And max_uint32))
        t >>= bit_count_in_uint32
#If DEBUG Then
        assert(t <= 2)
#End If
        Return CUInt(t)
    End Function

    'add d to the pos as p with carry-over as c
    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Function add(ByVal d As UInt32, ByVal c As UInt32, ByVal p As UInt32) As UInt32
        add_assertions(p)
        Return set_and_carry(CULng(v.get(p)) + c + d, p)
    End Function

    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Sub recursive_add(ByVal d As UInt32, ByVal p As UInt32)
        If d = 0 Then
            Return
        End If
        While p < v.size()
            add_assertions(p)
            d = set_and_carry(CULng(v.get(p)) + d, p)
            If d = 0 Then
                Return
            End If
            p += uint32_1
        End While
        v.push_back(d)
    End Sub
```
