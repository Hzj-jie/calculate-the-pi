# 计算π的值 - 无精度限制的计算 - big_uint的最大公约数

big_uint的最大公约数在big_udec的约分操作中用到，不然big_udec的长度会越来越长导致效率降低。

最基本的最大公约数的算法是辗转相除法，具体细节不再赘述。维基百科上有详细的说明：[wiki/最大公约数](https://zh.wikipedia.org/wiki/最大公因數)

不过如前面提到的，除法的复杂度相当高，一个N位数除以一个M位数的复杂度是O((N - M) * M)，所以还可以考虑用一些方法降低统计上的复杂度。

最好的实现方法是莱墨最大公约数算法，这个算法甚至没有中文的维基百科页面，我也没有找到别的可靠的说明。只能引用英文页面：[wiki/Lehmer's_GCD_algorithm](https://en.wikipedia.org/wiki/Lehmer's_GCD_algorithm)

这个想法很有意思，在公约数计算中，绝大部分情况下，两个数都非常接近，相除的商是1，2，3的几率大约是2/3。所以可以通过使用最高位来猜测商的结果，并且总计算部分值。这和之前提到的除法的试商过程非常接受。

所以在两个数几乎互质的时候，这个算法相当高效。不过实现起来相当复杂，.Net的BigInteger实现了这个算法：[System.Numerics/System/Numerics/BigIntegerBuilder.cs#L1113](https://github.com/microsoft/referencesource/blob/master/System.Numerics/System/Numerics/BigIntegerBuilder.cs#L1113)。

而OpenJDK的BigInteger则没有实现这个算法：[java/math/MutableBigInteger.java#l1871](https://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/math/MutableBigInteger.java#l1871)。

因为是用二进制系统计算，而取模中每一步的计算的目的是“降低两个数的宽度”，所以有几个细节很有意思，

1. 移位操作远比乘除法效率高。这个在乘法和除法的实现中也提到过，如果两个乘数中的一个是2的倍数，或者除数是2的倍数，那么就用移位代替浮点运算。
2. 两个数的基于2的公约数可以通过计算两个数的结尾有多少个二进制的0来判断。
3. 如果一个数是奇数，另一个是偶数，2不会是两个数的公约数。
4. 如果两个数都是奇数，一个减另一个结果变成了一个偶数，可以再用上一条减少两个数的长度。
5. 因为每一轮的运算都是取余，所以当一个的长度是N，另一个是M的时候，一次取模运算可以将长度由N降低到M，而运算量只是O((N - M) * M)，所以当两个数的长度差别很大的时候，或者M很小的时候，一次取模运算会比减法降低更多的宽度。

所以我实现了一个算法，用到上面的几个结论。[service/math/big_uint/big_uint.gcd.vb#L115](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.gcd.vb#L115)

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

不过无论如何，计算最大公约数和除法都是非常耗时的操作，所以big_udec需要谨慎的挑选约分的时机。
