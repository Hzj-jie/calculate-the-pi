# Calculating the Value of π - Arbitrary-Precision Arithmetic - Basic Structure of big_udec

Finally, we have moved from integers to decimals—after all, π is a decimal.

Of course, calling it a "decimal" here is not entirely accurate; there is more than one way to describe a non-integer. A standard positional decimal is one; a repeating decimal is another, though repeating decimals and fractions are essentially the same thing; and there are also continued fractions. Naturally, there are other representations as well, such as roots, integrals, and so on, which can all represent a non-integer.

For the sake of accuracy and simplicity of representation, `big_udec` chooses fractions to represent non-integers. Precision can be increased without limit simply by increasing the number of digits, and a fraction itself only requires two `big_uint` instances to represent. [service/math/big_udec/big_udec.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_udec/big_udec.vb)

```vb.net
' A positive decimal in range of [0, +inf) by the representation of n / d.
Partial Public NotInheritable Class big_udec
    Private ReadOnly n As big_uint
    Private ReadOnly d As big_uint
```

In addition to input/output, common arithmetic operations, and comparisons, `big_udec` also requires an important fraction reduction operation.

- Input and Output

Common data types, such as `Double` and `Single`

Fractional representations, such as a pair of `big_uint`s, or a pair of `UInt32`s, `Int32`s, etc.

Byte arrays

String decimals, supporting arbitrary bases and arbitrary numbers of decimal places. Of course, detecting repeating decimals could also be considered, though it has not been implemented yet since π is not a repeating decimal.

- Arithmetic Operations

Addition, subtraction, multiplication, division, exponentiation, root extraction, and fraction reduction. Because the greatest common divisor algorithm mentioned earlier is quite time-consuming, `big_udec` is not always kept in its irreducible (lowest-terms) form.

- Comparison

Just like `big_uint`, the remaining parts include .NET-related design elements such as overloading `ToString`, `Equals`, operators, and so on, as well as functions designed for testing, such as a `random` function that returns a random `big_udec`.
