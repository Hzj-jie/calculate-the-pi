# 计算π的值 - 无精度限制的计算 - big_uint的位运算

位运算相当简单，就简单记录一下。与、或和异或运算[service/math/big_uint/big_uint.bit.vb#L65](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.bit.vb#L65)

```vb.net
    'the bitwise operators are following the rules
    '1. if that is longer than me, ignore the overlength bits
    '2. if that is shorter than me, treat all the bits left as 0
    Private Function bit_wise_operation(ByVal that As big_uint, ByVal op As bit_wise_operator) As big_uint
        If that Is Nothing Then
            Return Me
        End If
        For i As UInt32 = 0 To v.size() - uint32_1
            Dim t As UInt32 = 0
            If i < that.v.size() Then
                t = that.v.get(i)
            Else
                t = 0
            End If
            Select Case op
                Case bit_wise_operator.and
                    v.set(i, v.get(i) And t)
                Case bit_wise_operator.or
                    v.set(i, v.get(i) Or t)
                Case bit_wise_operator.xor
                    v.set(i, v.get(i) Xor t)
                Case Else
                    assert(False)
            End Select
        Next
        If op = bit_wise_operator.and OrElse
           op = bit_wise_operator.xor Then
            remove_extra_blank()
        End If
        Return Me
    End Function
```

非运算[service/math/big_uint/big_uint.bit.vb#L94](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.bit.vb#L94)

```vb.net
    Public Function [not]() As big_uint
        If Not is_zero() Then
            assert(v.size() > 0)
            For i As UInt32 = 0 To v.size() - uint32_1
                v.set(i, Not v.get(i))
            Next
        End If
        Return Me
    End Function
```
