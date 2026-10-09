# 计算π的值 - 无精度限制的计算 - big_uint的乘法

教科书上的乘法是由两个部分组成的，被乘数（虽然其实严格上来说并没有这个定义，就用这个词表示乘法中的第二个数）中的每一位乘以乘数，根据位数移位，再把所有的结果加起来。

在时间复杂度上来说，这是个O(N^2)的算法，但是严格来说需要至少处理(N+N)*M位，N是乘数的长度，M是被乘数的长度。第一个N是用被乘数中的每一位乘以乘数得到的数字，第二个N是将这个数字加入结果。这样显然不是最快的做法，将乘法和加法合并在一次循环中可以很明显的提高性能。

考虑一下一个n+m+1位的数乘以i+j+1位的数，乘数的m+1位是A，被乘数的j+1位是B。A*B的值影响到m+j-1和m+j两位，其中m+j-1位应该直接加在结果上，而m+j位则是进位，在下一位乘法的时候再并入。

```
 {n位}A{m位}
*{i位}B{j位}
------------
...
A*B{m+j-2位}
...
```

同样的思路，把10进制换成4294967296进制就是big_uint的乘法。

```
    Private Sub multiply_uint(ByVal this As big_uint, ByVal that As big_uint)
        v.resize(this.v.size() + that.v.size())
        assert(this.v.size() > 0 AndAlso that.v.size() > 0)
        For i As UInt32 = 0 To this.v.size() - uint32_1
            If this.v.get(i) = 0 Then
                Continue For
            End If
            Dim c As UInt32 = 0
            For j As UInt32 = 0 To that.v.size() - uint32_1
                Dim t As UInt64 = 0
                t = this.v.get(i)
                t *= that.v.get(j)
                c = add(CUInt(t And max_uint32), c, i + j)  ' [1]
                c += CUInt(t >> bit_count_in_uint32)
            Next
#If DEBUG Then
            assert(v.get(i + that.v.size()) = 0)
#End If
            v.set(i + that.v.size(), c)  ' [2]
        Next
        remove_last_blank()
    End Sub
```

这个函数把this和that的乘积存在当前对象中。

[1]就是加法中提到的把前两个值加在第三个值所标志的位置上，并且返回进位的操作。[service/math/big_uint/big_uint.private.vb#L67](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L67)

这里因为是乘法，所以进位可能最多是2（比如说9*2+9）。

[2]则是一个小优化，因为总是第一次达到结果的位置，所以这个位置一定是0，不再需要进行加法，而只用复制进位数就可以了。

对于big_uint相乘的函数，除此以外就是一些边界处理，比如两个中任何一个是0则把结果设置为0，任何一个是1则把结果设置为另一个数。[service/math/big_uint/big_uint.private.vb#L172](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L172)

有一个有意思但是效率很低的算法，把that当作2进制，当一位为1的时候，把this加入结果，然后移位到下一个1的位。这个算法只需要加法和移位运算，其实是处理器中计算乘法的方法（现代处理器会把8*8的结果保存下来，用256进制）。不过big_uint的移位操作相当耗时，所以这个算法比较低效，尽管原理是完全一样的。

```
    Private Sub multiply_bit(ByVal this As big_uint, ByVal that As big_uint)
        this = this.CloneT()
        that = that.CloneT()
        While Not that.is_zero()
            Dim m As UInt32 = 0
            m = that.remove_binary_trailing_zeros()
            assert(that.odd())
            this.left_shift(m)
            add(this)
        End While
    End Sub
```

[service/math/big_uint/big_uint.private.vb#L134](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L134)

之后的除法运算中会大量的使用big_uint*uint的运算，所以乘法还有一个uint的重载。

```
    Public Function multiply(ByVal that As UInt32) As big_uint
        If that = 0 Then
            set_zero()
            Return Me
        End If
        If that = 1 OrElse is_zero() Then
            Return Me
        End If
        If is_one() Then
            replace_by(that)
        End If
        Dim c As UInt32 = 0
        For i As UInt32 = 0 To v.size() - uint32_1
            Dim t As UInt64 = 0
            t = v.get(i)
            t *= that
            t += c
            v.set(i, CUInt(t And max_uint32))
            c = CUInt(t >> bit_count_in_uint32)
        Next
        If c <> 0 Then
            v.push_back(c)
        End If
        remove_last_blank()
        Return Me
    End Function
```

基本操作是一样的，不过不再需要进行不必要的加法，因为结果中的每一位都应该是0，使用赋值即可。
