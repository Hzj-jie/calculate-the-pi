# 计算π的值 - 无精度限制的计算 - big_udec的字符串表示

big_udec有两种表示方式，分数和小数。分数表示非常容易理解，分子/分母就好了。这是一种精确的表示方式，一个big_udec的分数输出可以用以产生一个完全一致的big_udec。

```vb.net
    Public Function fractional_str(Optional ByVal str_base As Byte = constants.str_base) As String
        Return strcat(n.str(str_base), " ", character.left_slash, " ", d.str(str_base))
    End Function

    Public Shared Function parse_fraction(ByVal s As String,
                                          ByRef o As big_udec,
                                          Optional ByVal base As Byte = constants.str_base) As Boolean
        If Not support_base(base) Then
            Return False
        End If
        If String.IsNullOrEmpty(s) Then
            o = big_udec.zero()
            Return True
        End If
        s = s.Trim()
        If s.StartsWith(character.left_slash) OrElse s.EndsWith(character.left_slash) Then
            Return False
        End If
        Dim i As Int32 = 0
        i = s.IndexOf(character.left_slash)
        assert(i > 0 OrElse i < s.Length() - 1 OrElse i = npos)
        Dim n As big_uint = Nothing
        Dim d As big_uint = Nothing
        If i = npos Then
            If Not big_uint.parse(s, n, base) Then
                Return False
            End If
            d = big_uint.one()
        Else
            If Not big_uint.parse(s.Substring(0, i - 1), n, base) Then
                Return False
            End If
            If Not big_uint.parse(s.Substring(i + 1), d, base) Then
                Return False
            End If
        End If
        assert(Not n Is Nothing)
        assert(Not d Is Nothing)
        If d.is_zero() Then
            Return False
        End If
        o = New big_udec(n, d).fast_reduce_fraction()
        Return True
    End Function
```

小数表示则不精确，如之前提到的没有循环小数的检测，所以小数的输出通过小数位数的参数upure_len控制精度。

基本的思路就是用分子除以分母，结果是当前位的值，余数乘以每位的宽度是下一轮除法的分子。 $\frac{A}{B} = \frac{(n \times B)}{B} + \frac{(A - n \times B)}{B}$ ，这里n就是当前的位，(A - n * B) * str_base是下一轮的被除数。因为(A - n * B) < B，所以除了第一次的除法，之后每次只为结果增加一位。而第一次的商是整数部分。当位数达到upure_len参数时或者余数为0的时候结束，前者达到精度要求，后者表示之后都是0了。

```vb.net
        Public Overrides Function ToString() As String
            If this.is_zero() Then
                Return character._0
            End If
            If this.is_one() Then
                Return character._1
            End If

            Dim o As StringBuilder = Nothing
            o = New StringBuilder()
            Dim n As big_uint = Nothing
            n = this.n.CloneT()
            For i As UInt32 = 0 To upure_len
                Dim r As big_uint = Nothing
                n.assert_divide(this.d, r)
                o.Append(n.str(str_base))
                If i = 0 Then
                    o.Append(character.dot)
                End If
                If r.is_zero() Then
                    Exit For
                End If
                n = r
                n.multiply(str_base)
            Next
            o.trim_end(character.zero)
            o.trim_end(character.dot)
            If o.Length() = 0 Then
                Return character._0
            End If
            Return o.ToString()
        End Function
```

而从小数的输入则是一个很简单的思路，因为小数输入的长度总是已知的，只需要把小数点往后移到最后，并且设分母为 $1 \times (base) ^ N$ 即可。比如 $0.0003 = \frac {3}{10000}$ 。

因为小数的输出有精度控制，所以再通过小数输出生成的big_udec和之前的big_udec可能是不同的。

```vb.net
    Public Shared Function parse_fraction(ByVal s As String,
                                          ByRef o As big_udec,
                                          Optional ByVal base As Byte = constants.str_base) As Boolean
        If Not support_base(base) Then
            Return False
        End If
        If String.IsNullOrEmpty(s) Then
            o = big_udec.zero()
            Return True
        End If
        s = s.Trim()
        If s.StartsWith(character.left_slash) OrElse s.EndsWith(character.left_slash) Then
            Return False
        End If
        Dim i As Int32 = 0
        i = s.IndexOf(character.left_slash)
        assert(i > 0 OrElse i < s.Length() - 1 OrElse i = npos)
        Dim n As big_uint = Nothing
        Dim d As big_uint = Nothing
        If i = npos Then
            If Not big_uint.parse(s, n, base) Then
                Return False
            End If
            d = big_uint.one()
        Else
            If Not big_uint.parse(s.Substring(0, i - 1), n, base) Then
                Return False
            End If
            If Not big_uint.parse(s.Substring(i + 1), d, base) Then
                Return False
            End If
        End If
        assert(Not n Is Nothing)
        assert(Not d Is Nothing)
        If d.is_zero() Then
            Return False
        End If
        o = New big_udec(n, d).fast_reduce_fraction()
        Return True
    End Function
```

[service/math/big_udec/big_udec.str.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.str.vb)

有时候不需要把小数输出出来，而只是比较和已知的值匹配多少位，比如说用big_udec表示的π的值和已知的高精度小数表示进行对比。所以还有个str_match函数用以进行对比。基本的实现和小数输出一致，不过运算的结束不以upure_len参数控制，而是达到输入字符串的结尾或者某一位不再匹配为止。[service/math/big_udec/big_udec.strmatch.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.strmatch.vb)

```vb.net
    Public Function str_match(ByVal s As String, Optional ByVal str_base As Byte = constants.str_base) As UInt32
        Dim i As UInt32 = 0
        Dim n As big_uint = Nothing
        n = Me.n.CloneT()
        While True
            Dim r As big_uint = Nothing
            n.assert_divide(d, r)
            Dim c As String = Nothing
            c = n.str(str_base)
            If Not s.strsame(i, c, 0, c.strlen()) Then
                Return i
            End If
            i += c.strlen()
            If i = c.strlen() Then
                If Not s.strsame(i, character.dot, 0, character.dot.ToString().strlen()) Then
                    Return i
                End If
                i += character.dot.ToString().strlen()
            End If
            n = r
            n.multiply(str_base)
        End While
        assert(False)
        Return 0
    End Function
```
