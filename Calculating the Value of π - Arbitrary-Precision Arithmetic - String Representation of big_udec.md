# Calculating the Value of π - Arbitrary-Precision Arithmetic - String Representation of big_udec

big_udec has two representations: fractional and decimal. The fractional representation is very easy to understand—simply numerator / denominator. This is an exact representation; the fractional output of a big_udec can be used to reconstruct an identical big_udec.

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

The decimal representation, on the other hand, is inexact. As mentioned earlier, there is no detection for repeating decimals, so the precision of the decimal output is controlled by the decimal places parameter upure_len.

The basic idea is to divide the numerator by the denominator: the quotient is the value of the current digit, and the remainder multiplied by the base width becomes the numerator for the next round of division. $\frac{A}{B} = \frac{(n \times B)}{B} + \frac{(A - n \times B)}{B}$, where n is the current digit, and (A - n * B) * str_base is the dividend for the next round. Since (A - n * B) < B, every division after the first adds only one digit to the result, while the quotient of the first division is the integer part. The process terminates when the number of digits reaches the upure_len parameter or when the remainder becomes 0—the former means the required precision has been reached, and the latter means all subsequent digits are 0.

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

Parsing from a decimal input follows a very simple idea: since the length of the decimal input is always known, we just need to move the decimal point all the way to the end and set the denominator to $1 \times (base) ^ N$. For example, $0.0003 = \frac {3}{10000}$.

Because decimal output has limited precision, a big_udec reconstructed from a decimal output may differ from the original big_udec.

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

Sometimes we don't need to output the decimal string, but only want to compare how many digits match a known value—for example, comparing the value of π represented by big_udec against a known high-precision decimal representation. For this purpose, there is also a str_match function for comparison. Its basic implementation is the same as decimal output, except termination is not controlled by the upure_len parameter, but rather stops when it reaches the end of the input string or when a digit no longer matches. [service/math/big_udec/big_udec.strmatch.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.strmatch.vb)

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
