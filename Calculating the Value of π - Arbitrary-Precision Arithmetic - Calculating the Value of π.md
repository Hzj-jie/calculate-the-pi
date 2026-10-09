# Calculating the Value of π - Arbitrary-Precision Arithmetic - Calculating the Value of π

Calculating the value of π is slightly more complicated, mainly because there are many calculation methods available, so we need to pick a relatively efficient approach. Common methods can be found on Wikipedia: [wiki/Approximations_of_%CF%80](https://en.wikipedia.org/wiki/Approximations_of_π), so I won't repeat them here.

To calculate the value of π efficiently, several points need to be considered:

1. Just like calculating the value of e, an iterative approach is generally better, as it allows computation in stages by simply outputting intermediate results.
2. A monotonically increasing iterative method should be better than an oscillating one; simply put, the precision improves with every iteration.

Therefore, the Gregory-Leibniz series is relatively inefficient:

$\pi =4\sum _{n=0}^{\infty }{\cfrac {(-1)^{n}}{2n+1}}=4\left({\frac {1}{1}}-{\frac {1}{3}}+{\frac {1}{5}}-{\frac {1}{7}}+-\cdots \right)\!$

Although the calculation is relatively fast, after 67,108,865 iterations, the precision still only reaches 3 decimal places (3.1416).

```vb.net
    <command_line_specified>
    <test>
    Private Shared Sub calculate_pi_sequence_of_numbers_67108865()
        Dim s As big_udec = Nothing
        s = big_udec.zero()
        For i As UInt32 = 1 To 67108865 Step 2
            Dim c As big_udec = Nothing
            c = big_udec.fraction(CUInt(4), i)
            If (i >> 1).even() Then
                s.add(c)
            Else
                s.assert_sub(c)
            End If
        Next
        assertion.equal(s.fractional_str(), "3.141600282926117160615236325805874484799183595558135389722686711")
    End Sub
```

Similarly, the Nilakantha series is also relatively inefficient:

$\pi =3+{\frac {4}{2\times 3\times 4}}-{\frac {4}{4\times 5\times 6}}+{\frac {4}{6\times 7\times 8}}-{\frac {4}{8\times 9\times 10}}+\cdots $

After 20,002 iterations, the precision only reaches 10 decimal places (3.1415926539).

```vb.net
    <test>
    Private Shared Sub calculate_pi_nilakantha_20002()
        Dim s As big_udec = Nothing
        s = New big_udec(uint32_3, uint32_1)
        For i As UInt32 = 2 To 20002 Step 2
            Dim d As big_uint = Nothing
            d = New big_uint(i).
                    multiply(i + uint32_1).
                    multiply(i + uint32_2)
            Dim c As big_udec = Nothing
            c = big_udec.fraction(CUInt(4), d)
            If (i >> 1).odd() Then
                s.add(c)
            Else
                s.assert_sub(c)
            End If
        Next
        assertion.equal(s.str(), "3.1415926535900430885195015124988771459935110088383081739079144538")
        '                         3.1415926535897932384626433832795028841971693993751058209749445923078164062862089986280348253421170679
    End Sub
```

Although the Wallis product is monotonic, because the numbers multiplied in each step are relatively large, the precision is clearly not very high.

${\frac {\pi }{2}}={\Big (}{\frac {2}{1}}\cdot {\frac {2}{3}}{\Big )}\cdot {\Big (}{\frac {4}{3}}\cdot {\frac {4}{5}}{\Big )}\cdot {\Big (}{\frac {6}{5}}\cdot {\frac {6}{7}}{\Big )}\cdot {\Big (}{\frac {8}{7}}\cdot {\frac {8}{9}}{\Big )}\cdots$

After 2,520,002 iterations, it still only reaches 9 digits of precision.

```vb.net
    <command_line_specified>
    <test>
    Private Shared Sub calculate_pi_wallis_product_max_uint32_progressively()
        Dim s As big_udec = Nothing
        s = big_udec.one()
        s.multiply(big_udec.fraction(CUInt(2), CUInt(1)))
        For i As UInt32 = 2 To max_uint32 - CUInt(2) Step 2
            s.multiply(big_udec.fraction(i, i - uint32_1))
            s.multiply(big_udec.fraction(i, i + uint32_1))

            If (i Mod 900000) = 2 Then
                s.reduce_fraction()
                raise_error(error_type.warning, "@ ", i, " -> ", s.fractional_str())
            End If
        Next
    End Sub
```

Therefore, using the arctan and BBP methods is much better:

```vb.net
    <command_line_specified>
    <test>
    Private Shared Sub calculate_pi_arctangent()
        Dim s As big_udec = Nothing
        s = big_udec.fraction(2, 1)
        Dim c As big_udec = Nothing
        c = big_udec.fraction(2, 3)
        For i As UInt32 = 2 To max_uint32 - uint32_1
            s.add(c)
            c.multiply(big_udec.fraction(New big_uint(i), New big_uint(i).left_shift(1).add(uint32_1)))

            If (i Mod 100000) = 0 Then
                c.reduce_fraction()
                s.reduce_fraction()
                raise_error(error_type.warning, "@ ", i, " -> ", s.fractional_str(), " : c -> ", c.fractional_str())
            End If
        Next
    End Sub

    <command_line_specified>
    <test>
    Private Shared Sub bbp()
        Dim s As big_udec = Nothing
        s = big_udec.zero()
        For i As UInt32 = 0 To max_uint32 - uint32_1
            Dim c As big_udec = Nothing
            c = big_udec.fraction(4, 8 * i + 1)
            c.assert_sub(big_udec.fraction(2, 8 * i + 4))
            c.assert_sub(big_udec.fraction(1, 8 * i + 5))
            c.assert_sub(big_udec.fraction(1, 8 * i + 6))
            c.multiply(big_udec.fraction(big_uint.one(), big_uint.one().left_shift(CULng(4) * i)))
            s.add(c)

            If i <> 0 AndAlso (i Mod 100000) = 0 Then
                c.reduce_fraction()
                s.reduce_fraction()
                raise_error(error_type.warning, "@ ", i, " -> ", s.fractional_str(), " : c -> ", c.fractional_str())
            End If
        Next
    End Sub
```

The former reaches 301,033 digits of precision after one million iterations, improving precision by an average of 0.3 digits per iteration. The latter is even better, reaching 1,204,135 digits of precision after one million iterations, improving precision by an average of 1.2 digits per iteration. It took four days of computation on my machine to reach this precision.

The output is also very large, so please check it out on GitHub. [pi.1m.txt](https://github.com/Hzj-jie/calculate-the-pi/blob/master/pi.1m.txt) is about 6.5 MB, and [pi.1.2m.txt](https://github.com/Hzj-jie/calculate-the-pi/blob/master/pi.1.2m.txt) is about 7.27 MB.

All of the algorithms above can be found in [service/tests/math/big_udec/big_udec_pi_test.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/math/big_udec/big_udec_pi_test.vb).

Currently, 10M digits of π have been collected in [service/resources/math/codegen/pi-10m.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/resources/math/codegen/pi-10m.txt); unfortunately, we have not yet reached this precision. According to current estimates, using the BBP algorithm would require 8.4M iterations, which is estimated to take two to three months. Nevertheless, the number above is the closest known fractional representation of π.
