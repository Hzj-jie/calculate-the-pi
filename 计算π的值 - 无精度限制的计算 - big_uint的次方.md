# 计算π的值 - 无精度限制的计算 - big_uint的次方

次方运算在计算π的时候并不会用到，这个在后面会说到，不过作为big_uint的一个功能还是有意义的。

次方就是把一个数乘以自身很多次而已，不过得考虑一下怎么做效率比较高。首先A位数乘以B位数的结果不超过A+B位数。那么考虑一下一个N位数的M次方，结果最多N*M位，需要的位乘法数量是 $\sum_{1}^{M}{x} \times N ^ 2 = (1 + M) \times M / 2 \times N ^ 2$ 。这显然是个非常低效的做法，就像谁也不会把A × B转成 $\sum_{1}^{B}{A}$ 来计算。

所以首先得把计算量控制在M的位数而不是M的值本身可表示的范围内，需要用N和 $log_2M$ 来表示计算量。简单说来就是用若干的平方和乘法取代乘法。虽然平方也是乘法，不过可以证明总的位乘法数量可以显著降低。为了简化表示，设 $M' = log_2M$ 。并且取复杂度最高的情况，即M的每一位都是1。

首先算一下M'次平方所需要的位乘法的数量，注意这个和M在这一位是不是1无关。 一位数的平方需要的位乘法是一次，结果是二位数；二位数是四次，结果是四位数；以此类推。总和是$1 + 4 + 16 + ... + 2 ^ {2 \times (M' - 1)} = \sum_{0}^{M' - 1}{2 ^ {(x \times 2)}} = 2 ^ {(2 \times M' - 1)} - 1$。

如果起始位数是N，就如之前的例子，那么N位数的平方的位乘法是N^2，结果是2N位数；2N位数是4N^2次，结果是4N位数；以此类推。总和的公式只用再乘以N^2即可。 $N^2 + (2 \times N) ^ 2 + (4 \times N)^2 + ... = N ^ 2 \times (2 ^ {(2 \times M' - 1)}- 1)$

而每次把结果乘回结果的时候，每次原值的位数不会超过平方位数的一半，所以位乘法数量是 $1 \times N + N \times 2 \times N + 3 \times N \times 4 \times N + 7 \times N \times 16 \times N + ... \leq N ^ 2 \times 2 ^ {(2 \times M' - 2)}$

所以总和是 $\leq N^2 \times 2 ^ {(2 \times M' - 2)} \times 3$

注意到 $2 ^ {M'}$ 要小于M，所以这个值在理论上已经小于了重复乘以M次的做法。而因为实际上N位乘以N位不会达到2N位，所以实际计算的时候复杂度要更低一些。

---

举个例子吧， $11 ^ 5$ ，用上面两种做法就是：

$11 \times 11 \times 11 \times 11 \times 11$ ，位乘法的次数是 $2 \times 2 + 3 \times 2 + 4 \times 2 + 5 \times 2 = 28$

和

$11 \times 11 ^ {2 ^ 2}$ ，位乘法的次数是 $(2 \times 2) + (3 \times 3) + 5 \times 2 = 23$ 。

---

再举个极端点的例子， $99 ^ 8$ ：

$99 \times 99 \times 99 \times 99 \times 99 \times 99 \times 99 \times 99$ ，位乘法的次数是 $2 \times 2 + 4 \times 2 + 6 \times 2 + 8 \times 2 + 10 \times 2 + 12 \times 2 + 14 \times 2 = 112$

和

${{99 ^ 2 } ^ 2}^2$ ，位乘法的次数是 $2 \times 2 + 4 \times 4 + 8 \times 8 = 84$ 。

---

另一种极端的例子， $55 ^ 7$ ：

$55 \times 55 \times 55 \times 55 \times 55 \times 55 \times 55$ ，位乘法的次数是 $2 \times 2 + 4 \times 2 + 6 \times 2 + 7 \times 2 + 9 \times 2 + 11 \times 2 = 78$

和

$55 \times 55 ^ 2 \times 55 ^ 4$ ，注意到计算四次方的时候不用再算平方，位乘法的次数是 $2 \times 2 + 4 \times 4 + 2 \times 4 + 6 \times 7 = 70$

---

另外平方操作有一些内存优惠，不需要一个临时变量保存中间结果。

再剩下的就是一些边界情况，比如说 $X ^ 0 = 1$ ，$0 ^ x = 0 (x > 0)$ 。

实现的细节在[service/math/big_uint/big_uint.calculation.vb#L369](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.calculation.vb#L369)

```
    Public Function power(ByVal that As big_uint) As big_uint
        If that Is Nothing OrElse that.is_zero() Then
            '0 ^ 0 = 1
            set_one()
            Return Me
        End If
        If is_zero_or_one() OrElse that.is_one() Then
            Return Me
        End If
        that = that.CloneT()
        For i As UInt32 = 1 To that.remove_binary_trailing_zeros()
            power_2()
        Next
        Dim c As big_uint = Nothing
        c = Me.CloneT()
        assert(that.bit_count() > 0)
        Dim last As UInt64 = 0
        For i As UInt64 = 1 To that.bit_count() - uint64_1
            If Not that.getrbit(i) Then
                Continue For
            End If
            While last < i
                c.power_2()
                last += uint64_1
            End While
            multiply(c)
        Next
        Return Me
    End Function
```
