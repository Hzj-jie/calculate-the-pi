# Calculating the Value of π - Arbitrary-Precision Arithmetic - Calculating e First

Algorithms for e converge relatively quickly, so let's calculate the value of e first, which will run into fewer issues.

First, the calculated result for the value of e is a fraction—as big_udec itself is a fraction. We then compare it against the known value of e using the str_match function to determine the precision.

There are two main methods to calculate e:

${(1 + \frac{1}{x})}^{x}$, where precision increases as the value of x grows.

$1 + \sum_{x=1}^{N}{\frac{1}{x!}}$, where precision increases as the value of N grows.

In the first approach, the precision is fixed once x is chosen, whereas the second approach can progressively improve precision through iteration.

Let's look at the first approach first:

```vb.net
    <test>
    Private Shared Sub calculate_e_power_1000()
        Dim s As big_udec = Nothing
        s = big_udec.one()
        s.add(big_udec.fraction(CUInt(1), CUInt(1000)))
        s.power(CUInt(1000))
        assertion.equal(s.str(), "2.7169239322358924573830881219475771889643150188365728037223547748")
        '                         2.71828182845904523536028747135266249775724709369995957
    End Sub
```

When x is 1000, the precision only reaches 2 decimal places; when x is 100000, it reaches 4 decimal places.

Since the second approach is iterative, we can output intermediate values and results after a certain number of rounds:

```vb.net
    <command_line_specified>
    <test>
    Private Shared Sub calculate_e_factorial_max_uint32_progressively()
        Dim s As big_udec = Nothing
        s = big_udec.one()
        Dim c As big_udec = Nothing
        c = big_udec.one()
        For i As UInt32 = 1 To max_uint32 - uint32_1
            c.assert_divide(New big_udec(i))
            s.add(c)

            If (i Mod 100000) = 0 Then
                c.reduce_fraction()
                s.reduce_fraction()
                raise_error(error_type.warning, "@ ", i, " -> ", s.fractional_str(), " : c -> ", c.fractional_str())
            End If
        Next
    End Sub
```

[service/tests/math/big_udec/big_udec_e_test.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/math/big_udec/big_udec_e_test.vb)

On my machine, it takes about 4 hours and 15 minutes to compute 400,000 rounds, and the precision already exceeds the first two million digits. [service/resources/math/codegen/e-2m.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/resources/math/codegen/e-2m.txt)

In about 5 days, it computed 1,600,000 rounds; following the previous pattern, the estimated precision exceeds eight million digits.

The fractional representations of these big_udec values are quite large and cannot be pasted here, so please check them out on GitHub. [e.2m.txt](https://github.com/Hzj-jie/calculate-the-pi/blob/master/e.2m.txt) is about 4 MB, and [e.8m.txt](https://github.com/Hzj-jie/calculate-the-pi/blob/master/e.8m.txt) is about 18 MB. These are probably the closest fractional representations of e that can be found on the Internet.
