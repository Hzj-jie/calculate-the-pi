# Calculating the Value of π - Arbitrary-Precision Arithmetic - From Floating-Point Numbers to big_udec

Unlike fractional representations, floating-point numbers ([IEEE-754](https://standards.ieee.org/standard/754-2019.html)) suffer from precision issues. Simply put, a floating-point number is always a binary number multiplied by a power of two—in other words, binary scientific notation. For example, $10 E_{2} 101 = 2 \times 2 ^ 5 = 64$, and the same applies to fractions, such as $101 E_{2} -10 = 5 \times 2 ^ {-2} = 5 / 4$. Unfortunately, even in widely used languages like Java, C, C++, and C#, any floating-point type that follows IEEE-754 cannot accurately represent the decimal value 0.1; it can only represent an approximation like 1/16+1/32+1/256+1/512+1/4096+1/8192+1/65536=0.09999084472.

IEEE-754 includes rules for decimal conversion, so when a floating-point number is sufficiently close to 0.1, the error is ignored and it is treated as 0.1, for example: [http://cpp.sh/3rcdj](http://cpp.sh/3rcdj)

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

Of course, using the byte-array representation of a floating-point number to parse its exponent and significand (mantissa) is feasible. However, doing so involves too many implementation details and ties the implementation to specific languages, because there is no guarantee that decimal types in all languages are IEEE-754 double-precision floats—for instance, `Decimal` in C# is not a standard binary floating-point number. Therefore, `big_udec` uses a simpler approach to handle floating-point numbers and introduces its first parameter: the maximum number of shifts (`max_shift`).

The implementation is straightforward: it always assumes the denominator is $1 E_{2} X$, and repeatedly multiplies the input floating-point number by 2 until the number becomes an integer or reaches the maximum shift count, while `X` in the denominator is the number of times it was multiplied by 2. This avoids having to deal with language-specific implementation details and rounding-error handling. [service/math/big_udec/big_udec.vb#L67](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.vb#L67)

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

The default `max_shift` is 50, because the significand of a double-precision floating-point number has 52 bits.

The same method can be applied to any "decimal" representation in any language, such as `Decimal` in C#: [service/math/big_udec/big_udec.vb#L91](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.vb#L91)

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
