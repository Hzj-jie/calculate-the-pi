# Calculating the Value of π - Arbitrary-Precision Arithmetic - String Input and Output of big_uint

String input and output is an important feature, and the subsequent implementation of `big_udec` also relies on the string input and output of `big_uint`.

Dividing by N and taking the remainder is the most fundamental method for base conversion. This conversion method is independent of the radix—dividing by any N converts a number into base N. Conversely, multiplying by N and adding the next digit implements base-N input. While these are the most basic ideas, doing them one character digit at a time is very slow. Because each internal digit of `big_uint` is in base 4294967296 and the processor can handle at least one base-4294967296 word at a time, performing each multiplication or division with a divisor/multiplier as close as possible to a full `uint32` is much more efficient.

The first design element is a constant array `digit_count_per_parse`, where each entry specifies how many digits of a given base should be grouped together per step so that the chunk value gets as close as possible to a `uint32`.

```vb.net
digit_count_per_parse(i) = CByte(System.Math.Floor(bit_count_in_uint32 /
                                                   (System.Math.Log(i) / System.Math.Log(2))))
```

The principle is simple: in base `i`, the number of digits that can fit into a `uint32` without overflow is at most $32 \div (log(i) \div log(2))$. For example, in base 3, a single `uint32` can hold at least 20 base-3 digits; in base 16, a single `uint32` can hold 8 hexadecimal digits.

Thus, when outputting a number in base 3, we divide and take remainders in base $3^{20}=3486784401$, and then convert each remainder into 20 base-3 digits. This dramatically improves overall processing efficiency.

There is also an optimization mentioned earlier in the division article: if the base itself is a power of 2 (only a single `1` bit in binary), division and modulo can be replaced by bit shifts and bitwise AND operations. However, since this optimization is already handled inside the division implementation, there is no need to duplicate it here.

What remains are boundary cases, such as outputting 0 and 1: [service/math/big_uint/big_uint.str.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.str.vb)

```vb.net
    Public Function str(Optional ByVal base As Byte = default_str_base) As String
        If Not support_base(base) Then
            Return Nothing
        End If
        If is_zero() Then
            Return digit_0
        End If
        If is_one() Then
            Return number_to_char(1)
        End If
        Dim r As StringBuilder = Nothing
        r = New StringBuilder()
        Dim t As big_uint = Nothing
        t = New big_uint(Me)
        Dim dc As Byte = 0
        dc = digit_count_per_parse(base)
        assert(dc > 0)
        Dim bu As big_uint = Nothing
        bu = assert_which.of(base ^ dc).can_cast_to_uint64()
        While Not t.is_zero()
            Dim rm As big_uint = Nothing
            t.assert_divide(bu, rm)
            For i As UInt32 = 0 To dc - uint32_1
                Dim c As UInt32 = 0
                rm.assert_divide(base, c)
                r.Append(number_to_char(assert_which.of(c).can_cast_to_byte()))
                If rm.is_zero() Then
                    If Not t.is_zero() Then
                        r.Append(digit_0, CInt(dc - i - uint32_1))
                    End If
                    Exit For
                End If
            Next
        End While
        Return Convert.ToString(r.reverse())
    End Function
```

And parsing 0 and 1 on input:

```vb.net
    Public Shared Function parse(ByVal s As String,
                                 ByRef r As big_uint,
                                 Optional ByVal base As Byte = default_str_base) As Boolean
        If Not support_base(base) Then
            Return False
        End If
        If s.null_or_whitespace() Then
            r = New big_uint()
            Return True
        End If
        s = s.Trim()
        If s = number_to_char(0) Then
            r = New big_uint()
            Return True
        End If
        If s = number_to_char(1) Then
            r = big_uint.one()
            Return True
        End If
        Dim dc As Byte = 0
        dc = digit_count_per_parse(base)
        assert(dc > 0)
        Dim multiply_t As big_uint = Nothing
        multiply_t = multiply_base(base, dc)
        r = New big_uint()
        For i As Int32 = 0 To strlen_i(s) - 1 Step dc
            Dim u As UInt32 = 0
            Dim j As Int32 = 0
            For j = 0 To dc - 1
                If i + j >= strlen(s) Then
                    r.multiply(multiply_base(base, assert_which.of(j).can_cast_to_byte()))
                    r.add(u)
                    Return True
                End If
                Dim b As Byte = 0
                If char_to_number(s(i + j), b, base) Then
                    u *= base
                    u += b
                Else
                    Return False
                End If
            Next
            r.multiply(multiply_t)
            r.add(u)
        Next
        Return True
    End Function
```
