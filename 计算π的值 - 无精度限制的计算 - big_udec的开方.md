# 计算π的值 - 无精度限制的计算 - big_udec的开方

其实至此，big_udec已经可以用于计算π和e的值了，不过有些算法依赖于开方，所以也实现以下开方运算。

首先，无理数和有理数是两个互斥的集合，而一个有理数开若干次方后很大几率是一个无理数。而同理，一个无理数无法写成两个有理数的和，所以如big_uint开方操作的“余数”也是不可行的。无论如何也只是逼近而已，而不是准确的等于。

所以退而求其次，big_udec的开次方运算使用一个参数来控制逼近的程度。注意到 $\sqrt[\frac{n}{p}]{\frac{a}{b}} = \frac{a}{b} ^ \frac{p}{n} = (\sqrt[n]{\frac{a}{b}})^p$ 。所以只用考虑big_udec开一个整数次方的情况即可。

容易想到， $\sqrt[n]{\frac{a}{b}} \approx \frac{\sqrt[n]a}{\sqrt[n]b}$ ，而整数的开方已经可以处理了。同样当分子和分母同时乘以一个很大的数的时候，整数开方的“余数”所占的比例会降低，所以 $abs(\frac{\sqrt[n]{a \times C}}{\sqrt[n]{b \times C}} - \sqrt[n]{\frac{a}{b}}) \ll abs(\frac{\sqrt[n]a}{\sqrt[n]b} - \sqrt[n]{\frac{a}{b}})$ ，就是说精度提高了。

再进一步，如果选取一个合适的C可以避免两次开次方操作会非常有意义，一来降低运算量，二来让精度更易控制。一个简单的想法就是选择一个合适的C让分母正好是一个已知数的n次方，所以 $b^{(n-1)} | C$ 。同时也可以考虑把a也带入分母，这样就不会出现 $\sqrt[n]{\frac{b-1}{b}}$ 和 $\sqrt[n]{\frac{1}{b}}$ 精度差别太大的问题。

那么首先 $\sqrt[n]{\frac{a}{b}}=\sqrt[n]{\frac{a^{(n+1)} \times b^{(n-1)}}{a^n \times b^n}}$ ，再对分子和分母分别开方即可得到一个比较合适的近似值了。

不过这还有个问题，如果n很小的时候，这个近似会很糟糕。比方说 $\sqrt[2]{\frac{2}{3}} \approx 0.81649658092$ ，而按照这个做法出来的结果是 $\sqrt[2]{\frac{2 ^ 3 \times 3}{2 ^ 2 \times 3 ^ 2}} = \frac{\sqrt{24}}{6} = \frac{4}{6} \approx 0.66666666667$ ，注意这里已经用整数开方算法去掉了“余数”。所以需要引入另一个参数，次方基数来避免这个问题。

原理上没有什么变化，不过是再额外增加一些次方 $\sqrt[n]{\frac{a}{b}}=\sqrt[n]{\frac{a^{(n \times p +1)} \times b^{(n \times p -1)}}{a^{(n \times p)} \times b^{(n \times p)}}}$ ，通过这个p的值可以控制精度。比如当p=2的时候 $\sqrt[2]{\frac{2}{3}} = \sqrt[2]{\frac{2 ^ 5 \times 3 ^ 3}{2 ^ 4 \times 3 ^ 4}} = \frac{\sqrt{864}}{36} = \frac{29}{36} \approx 0.80555555555$ ，而当p=10的时候 $\sqrt[2]{\frac{2}{3}} = \sqrt[2]{\frac{2 ^ {21} \times 3 ^ {19}}{2 ^ {20} \times 3 ^ {20}}} = \frac{\sqrt{2437438960041984}}{60466176} = \frac{49,370,425}{60466176} \approx 0.81649656496$ 。明显更加接近准确值。

那么根据上面所述的各种条件，这个p的值应该考虑：

- 分子越小，这个值应该越大。因为结果的分母是准确的，而分子是近似值。分子中原分数的分子的次方要高于分母两次。
- 相对的分母越大，这个值应该越大。
- 次方越大，这个值应该越小。如果次方数足够大了，那么 $a^{(n \times p +1)} \times b^{(n \times p -1)}$ 也足够大了，开方以后的余数也足够小了。而且因为最后这个值要被开 $n \times p$ 次方，所以当n足够大的时候，结果总是接近于1，再增大p的值对结果不会有影响。

一个拍脑袋的想法就是把上面这三个规则合并起来： $p = C \times b \div a \div n$ ，这里的C是另一个拍脑袋的常量，现在取的值是40，没有什么必然原因。通过调整这个C的值得到不同的精度。

那么综合上面的结论，得出的开方算法就是：[service/math/big_udec/big_udec.calculation.vb#L220](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.calculation.vb#L220)

```vb.net
    Public Function extract(ByVal that As big_uint,
                            ByVal extract_power_base As UInt32,
                            ByRef divide_by_zero As Boolean) As big_udec
        assert(extract_power_base > 0)
        If that Is Nothing OrElse that.is_zero() Then
            divide_by_zero = True
            set_zero()
            Return Me
        End If
        divide_by_zero = False
        If is_zero() OrElse is_one() OrElse that.is_one() Then
            Return Me
        End If
        Dim p As big_uint = Nothing
        p = Me.n * extract_power_base \ Me.d \ that
        If p.is_zero_or_one() Then
            p = New big_uint(CUInt(2))
        End If
        replace_by(((Me.n ^ (that * p + uint32_1)) * (Me.d ^ (that * p - uint32_1))).assert_extract(that),
                   (Me.n * Me.d) ^ p)
        Return Me
    End Function
```

这里的extract_power_base就是上面说到的C的值。
