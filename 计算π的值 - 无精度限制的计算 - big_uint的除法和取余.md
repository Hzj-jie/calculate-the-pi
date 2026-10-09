# 计算π的值 - 无精度限制的计算 - big_uint的除法和取余

对于整数，除法的结果是商和余数。所以除法和取余算是同一个运算，前者需要把结果存下来，而后者不需要。

还记得“试商”这个名词吗？商的每一位是由高位到低位“试”出来的，在某一位上试取了某个值以后总归会有三种结果。

- 除数乘以此位大于了被除数，则应该将此位减小。
- 除数乘以此位小于被除数，则此位可能比当前值大。
- 除数乘以此位等于被除数，则此位应该取此值，并且之后的运算都不必了，后面的商的位数都是0了。

当然书上并没有说到，其实商大了并没有什么问题，因为余数可以是负数。比方说

```
除数        商（位）        除数×商        余数        除数的左移位
24                                          10000       100
            500             12000           -2000        10
            -90             -2160             160         1
              6               144              16
----------------------------------------------------
            416                                16
```

（最初的被除数也可以认为是余数。）

如果只用正数和退位来处理的话，上式可以表示为

```
除数        商（位）        除数×商        余数        除数的左移位        退位
24                                          10000       100                 0
            500             12000            2000        10                 1
             90              2160             160         1                 0
              6               144              16                           0
----------------------------------------------------
            416                                16
```

当有退位时，新的余数是（除数×商-余数）而不是常见的（余数-除数×商）。不过这种做法有很多边界情况，比方说在有退位时，取商的值要使之与除数的乘积大于余数，没有退位时这个乘积要小于余数。所以这个做法其实在并没有降低时间复杂度的情况下，增加了代码复杂性。因为有减法，所以时间复杂度是O(N^3)。

所以在实现时要避免第一种，除数和商的位的乘积大于余数的情况。但是第二种情况，乘积小于余数的时候并没有很大的问题，只是需要再进行一次乘法和减法来补充商的位。再用上面的例子来讨论一下这种情况。

```
除数        商（位）        除数×商        余数        除数的左移位
24                                          10000                100
            400             9600              400                 10
             10              240              160                  1
              5              120               40                  1
              1               24               16
----------------------------------------------------
            416                                16
```

在最低位时第一次尝试了5，剩下的余数40仍然大于除数24，所以第二次尝试1，最终得到余数是16。

这种情况的意义在于，当余数和除数都极大的时候，没法最准确的估计出商的位，所以用保守的方式保证除数和商的位的乘积总小于余数。

那么如何估计商的位则是影响性能的相当重要的一步。比如说90%的几率保证可以得到准确的值，则E=1.1*N*N*N。而如果这个几率提高到95%，E=1.05*N*N*N。

所以首先保证除数和商的位的乘积不超过余数。假设Axxxxx除以Bxxxxx，这里A和B各是一个十进制数位，xxxxx是另五位，但是假设是50000位的话，自然不能都看一遍。这个商的位只能由A和B决定。那么很容易的想到，这个商总是小于或等于A/(B+1)。

这样最基本的运算方式就已经明确了，当然要把十进制思维方式转换成4294967296进制。

- 把除数往左移位，使除数和余数（被除数）的位数相同。
- 重复下面这步，直到余数的左移位被全部右移了。

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

还有些边界处理，比如开始时被除数位数如果小于除数，则直接将被除数作为余数返回；如果除数是2的次方，则可以通过移位和与操作避免除法运算。

[service/math/big_uint/big_uint.calculation.vb#L110](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.calculation.vb#L110)

因为big_uint的每一位都是uint，而64位处理器可以支持ulong的除法，所以这里有个优化，在select_significant_divide_fraction中。

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

如果除数和余数的位数一样，并且都多于一位，则同时取高两位，这样可以有效的提高取商的位的精度。

另外和乘法一样，同样也有个divide_bit的实现，用移位和加减法取代除法，这样也是在处理器中实现的方法。[service/math/big_uint/big_uint.divide.vb#L56](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.divide.vb#L56)

divide(uint, uint&, bool&)的实现则是除以一个uint的计算（[service/math/big_uint/big_uint.divide.vb#L13](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.divide.vb#L13)），相当于一个数除以一个一位数的操作，原理更加简单，不再赘述。

取余运算则是忽略累加结果的操作，亦不再赘述。
