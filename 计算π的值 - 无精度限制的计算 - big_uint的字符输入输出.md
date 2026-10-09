# 计算π的值 - 无精度限制的计算 - big_uint的字符输入输出

字符输入输出是一个重要的功能，之后的big_udec的实现也用到big_uint的字符串输入和输出。

除N取余是最基本的进制转换方式，这个转换方式和基数无关，除以任意的N都可以实现N进制的转换。而相对乘以N再加上下一个位就可以实现N进制的输入。这都是最基本的思路，但是这个思路会非常的慢。因为big_uint的位都是4294967296进制的，处理器每次可以处理至少一个4294967296位，所以每次乘法或除法也尽量接近一个uint32时更加高效。

首先第一个设计是digit_count_per_parse的常量数组，每个常量是当选择这个进制的时候，每次处理多少位会使结果更接近一个uint32。

```vb.net
digit_count_per_parse(i) = CByte(System.Math.Floor(bit_count_in_uint32 /
                                                   (System.Math.Log(i) / System.Math.Log(2))))
```

原理很简单，i进制时每个uint32包含的位数不多于 $32 \div (log(i) \div log(2))$ 。比如3进制的时候，一个uint32包含至少20位3进制数；16进制时，一个uint32包含8位16进制数。

这样如果把一个数输出为3进制时，用 $3^{20}=3486784401$ 进制取余数，每个余数再转换为20位3进制数。这样整体的处理效率能大幅提高。

还有个之前在除法的时候提到的一个优化，如果进制数本身是一个只有一位1的二进制数，那么可以把除法和取余变成移位和与运算。不过这个在除法中已经处理了，所以不需要再重复了。

剩下的就是一些边界情况，比如0和1的输出。[service/math/big_uint/big_uint.str.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.str.vb)

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

以及0和1的输入。

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
