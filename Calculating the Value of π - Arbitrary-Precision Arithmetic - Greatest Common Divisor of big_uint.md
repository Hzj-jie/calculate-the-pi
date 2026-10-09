# Calculating the Value of π - Arbitrary-Precision Arithmetic - Greatest Common Divisor of big_uint

The greatest common divisor (GCD) of `big_uint` is used in the fraction reduction operations of `big_udec`; without it, the length of `big_udec` would grow increasingly large and degrade efficiency.

The most basic algorithm for finding the greatest common divisor is the Euclidean algorithm, the details of which will not be repeated here. Wikipedia has a detailed explanation: [wiki/Greatest Common Divisor](https://zh.wikipedia.org/wiki/最大公因數)

However, as mentioned earlier, the complexity of division is quite high—dividing an N-digit number by an M-digit number has a complexity of O((N - M) * M)—so we can also consider methods to reduce the statistical complexity.

The best implementation method is Lehmer's GCD algorithm. This algorithm does not even have a Chinese Wikipedia page, nor have I found other reliable explanations in Chinese, so I can only cite the English page: [wiki/Lehmer's_GCD_algorithm](https://en.wikipedia.org/wiki/Lehmer's_GCD_algorithm)

The idea is very interesting. In GCD calculations, in the vast majority of cases, the two numbers are very close to each other, and the probability that the quotient of their division is 1, 2, or 3 is about 2/3. Therefore, we can use the most significant digits to guess the quotient and only compute partial values. This is very similar to the trial quotient process in division mentioned earlier.

Thus, when two numbers are almost coprime, this algorithm is quite efficient. However, it is fairly complex to implement. .NET's `BigInteger` implements this algorithm: [System.Numerics/System/Numerics/BigIntegerBuilder.cs#L1113](https://github.com/microsoft/referencesource/blob/master/System.Numerics/System/Numerics/BigIntegerBuilder.cs#L1113).

On the other hand, OpenJDK's `BigInteger` does not implement this algorithm: [java/math/MutableBigInteger.java#l1871](https://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/math/MutableBigInteger.java#l1871).

Because calculations are performed in a binary system, and the goal of each step in the modulo operation is to "reduce the bit width of the two numbers," there are several interesting details (cf. Stein's binary GCD algorithm):

1. Bit-shift operations are far more efficient than multiplication and division. As also mentioned in the implementation of multiplication and division, if one of the two multipliers is a multiple of 2, or the divisor is a multiple of 2, bit shifting is used instead of floating-point operations.
2. The power-of-2 common divisor of two numbers can be determined by counting how many trailing binary zeros both numbers have.
3. If one number is odd and the other is even, 2 cannot be a common divisor of the two numbers.
4. If both numbers are odd, subtracting one from the other yields an even number, allowing the previous rule to be applied again to reduce the length of the two numbers.
5. Since each round of calculation finds the remainder, when one number has length N and the other has length M, a single modulo operation can reduce the length from N to M with a computational cost of only O((N - M) * M). Therefore, when the lengths of the two numbers differ significantly, or when M is very small, a single modulo operation reduces the width much more effectively than subtraction.

So I implemented a hybrid algorithm that leverages the conclusions above. [service/math/big_uint/big_uint.gcd.vb#L115](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.gcd.vb#L115)

```
    <MethodImpl(math_debug.aggressive_inlining)>
    Private Shared Function gcd_hybrid_loop(ByVal a As big_uint, ByVal b As big_uint) As big_uint
        While True
#If DEBUG Then
            assert(Not a.is_zero())
            assert(Not b.is_zero())
#End If

            Dim c As big_uint = Nothing
            If small_number_gcd(a, b, c) Then
                Return c
            End If

            Dim cmp As Int32 = 0
            cmp = a.compare(b)

            If cmp < 0 Then
                root.connector.swap(a, b)
            End If

            If cmp = 0 Then
                Return a
            End If

            If a.remove_trailing_binary_zeros() > 0 Then
                Continue While
            End If

            If b.remove_trailing_binary_zeros() > 0 Then
                Continue While
            End If

            If a.uint32_size() - b.uint32_size() >= 2 OrElse b.uint32_size() <= 2 Then
                c = a.assert_modulus(b)
                If c.is_zero() Then
                    Return b
                End If
                a = b
                b = c
                Continue While
            End If

            a.assert_sub(b)
        End While
        assert(False)
        Return Nothing
    End Function

    <MethodImpl(math_debug.aggressive_inlining)>
    Private Shared Function gcd_hybrid(ByVal a As big_uint, ByVal b As big_uint) As big_uint
        Dim shift As UInt32 = 0
        shift = shifting(a, b)
        Return gcd_hybrid_loop(a, b).left_shift(shift)
    End Function
```

Regardless, computing the greatest common divisor and performing division are both very time-consuming operations, so `big_udec` needs to carefully choose when to reduce fractions.
