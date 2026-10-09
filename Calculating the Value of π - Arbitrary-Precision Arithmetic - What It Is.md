# Calculating the Value of π - Arbitrary-Precision Arithmetic - What It Is

Computer arithmetic is inherently constrained by data width. For example, a common unsigned integer is 32 bits wide, ranging from 0 to 4,294,967,295. A floating-point number is typically 64 bits wide, spanning ±1.79769313486231570E+308, but offers only 15 significant decimal digits. Such precision and range are more than sufficient for everyday computations, yet they fall short in scenarios such as exact mathematical calculations.

Therefore, we need to consider how to implement efficient storage structures and algorithms for arbitrary-precision arithmetic. Of course, this is not literally unlimited precision—no matter what, it cannot exceed physical memory limits. However, since memory capacity far exceeds the required precision, memory can be treated as effectively infinite here.

A practical way to verify that the implementation works is to compute constants such as π and e.

Both π and e are infinite non-repeating decimals (while this might not be universally settled in every sense, they are non-repeating within all known ranges). Thus, what we ultimately need to implement is a decimal/fractional type rather than just a typical `BigInteger`.

A very simple idea is to combine two `BigInteger` instances into a fraction, though many details are needed to make that fraction practical. Still, this simple idea forms the foundation of the implementation. With a slight variation, the implementation starts with `big_uint` (an unsigned integer), followed by `big_udec` (an unsigned decimal/fraction). As for signed values, they are simply these two types paired with a sign bit.

The implementation is language-agnostic; out of personal preference, it is written in [http://vb.net](http://vb.net).

big_uint: [service/math/big_uint](https://github.com/Hzj-jie/osi.net/tree/master/service/math/big_uint)

big_udec: [service/math/big_udec](https://github.com/Hzj-jie/osi.net/tree/master/service/math/big_udec)

`big_uint` implements addition, subtraction, multiplication, division, exponentiation, roots, factorials, and greatest common divisor (GCD) operations, as well as comparison and bitwise operations.

`big_dec` implements addition, subtraction, multiplication, division, exponentiation, roots, and reciprocal operations, as well as comparison operations.
