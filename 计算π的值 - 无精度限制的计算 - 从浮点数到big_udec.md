# 计算π的值 - 无精度限制的计算 - 从浮点数到big_udec

和分数表示不同，浮点数（[IEEE-754](https://standards.ieee.org/standard/754-2019.html)）是有精度问题。简单说来，一个浮点数总是一个二进制数乘以二的若干次方，或者说是一个二进制的科学技术法。比方说 $10 E_{2} 101 = 2 \times 2 ^ 5 = 64$，对小数也同样，比如 $101 E_{2} -10 = 5 \times 2 ^ {-2} = 5 / 4$ 。所以很遗憾，即便是大量使用的java，c，c++，c#等等语言，只要是支持IEEE-754的浮点数，都没法准确的表示十进制的0.1，能表示的只是类似1/16+1/32+1/256+1/512+1/4096+1/8192+1/65536=0.09999084472这样的近似值。

IEEE-754中有关于小数转化的规则，所以当一个浮点数相当接近0.1的时候，误差会被忽略，从而被处理为0.1，比如：[http://cpp.sh/3rcdj](http://cpp.sh/3rcdj)

```cpp
#include <iostream>

int main()
{
    double d = 0.0999999;
    std::cout << d << std::endl;  // 0.0999999
    d = 0.09999999;
    std::cout << d << std::endl;  // 0.1
}
```

当然，使用浮点数的字节数组表示，分析指数部分和有效数部分的做法是可行的。但是这样涉及太多的实现细节，并且让实现和语言相关了，因为并不能保证所有语言中的小数都是双浮点数，比如c#中的Decimal就不是标准的浮点数。所以big_udec使用更为简单的方式来处理浮点数，并引入第一个变量：最大位移次数。

实现的方法很简单，总是假设分母是 $1 E_{2} X$ ，把输入的小数重复乘以2，直到这个小数变成了整数或者达到了最大移位次数，而分母的X就是乘以2的次数。这样就不需要考虑语言实现的细节和误差处理了。[service/math/big_udec/big_udec.vb#L67](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.vb#L67)

```vb.net
    Public Sub replace_by(ByVal v As Double, ByVal max_shift As UInt32)
        assert(v >= 0)
        assert(max_shift > 0)
        d.set_one()
        While v.is_not_integral() AndAlso max_shift > 0
            d.left_shift(uint32_1)
            v *= 2
            max_shift -= uint32_1
        End While
        assert(n.replace_by(v))
    End Sub
```

默认的max_shift是50，因为双浮点数的有效数部分有52位。

同样的方法可以用于任何语言中的“小数”表示，比如c#中的Decimal：[service/math/big_udec/big_udec.vb#L91](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.vb#L91)

```vb.net
    Public Sub replace_by(ByVal v As Decimal, ByVal max_shift As UInt32)
        assert(v >= 0)
        d.set_one()
        While v.is_not_integral() AndAlso max_shift > 0
            d.left_shift(uint32_1)
            v *= 2
            max_shift -= uint32_1
        End While
        assert(n.replace_by(v))
    End Sub
```
