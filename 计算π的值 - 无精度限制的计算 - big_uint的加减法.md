# 计算π的值 - 无精度限制的计算 - big_uint的加减法

这恐怕是绝大多数理解big_uint人都知道的一个部分，加法和减法是比较基础的big_uint的操作。当然作为永远只表示正数的"u"int，减法的API会有些差别。

首先还是先说明一下big_uint的API设计的基本原则，

- big_uint是一个类/引用而不是结构/值，这是.net的一个特定设定，和实现不是很相关。
- big_uint本身是可变的，所有的公用运算函数都是修改当前的值本身，并且返回自身。当然受限于.net的设定，所有运算符都是静态函数，并且总是返回一个新的实例。这样就可以串联多个运算而不需要多次复制，比如

```
big_uint b = big_uint.one().add(100).multiply(100).power(2);
```

- 对于可能产生运算错误的运算函数，总有三个版本，1）通过传出参数返回错误，2）在错误时抛出异常，3）在错误时断言失败并强制退出程序。比如说减法（触发溢出错误）、除法（除法除零错误）。

```
Public Function [sub](ByVal that As big_uint, ByRef overflow As Boolean) As big_uint
Public Function [sub](ByVal that As big_uint) As big_uint ' throws OverflowException
Public Function assert_sub(ByVal that As big_uint) As big_uint ' assert no overflow exception
```

先看相对简单的加法。从低位开始，按位相加，记录进位，如果到了最高位还有进位，则把最后一个进位放在最高位。这是最基本的数学课本上介绍的方法。不过在高精度计算中，这里的“位”是4294967296进制的“位”。所以在每一“位”的加法的时候，需要使用一个uint64。其实这也很好理解，十进制加法的时候，每次也是会算出两位来，一位是当前位的结果，另一位则是进位。所以uint64的低32位是一“位”，高32位是进位。

实现的细节上还有些考量，加法的操作只用做到两个big_uint中比较少的位数，剩下的只需要把更多的位数复制到结果中，再把进位加上去就好了。另外，如果两个数中的一个是0，结果是另一个数。[service/math/big_uint/big_uint.calculation.vb#L14](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.calculation.vb#L14)

```
    Public Function add(ByVal that As big_uint) As big_uint
        If that Is Nothing OrElse that.is_zero() Then
            Return Me
        End If
        If is_zero() Then
            assert(replace_by(that))
            Return Me
        End If
        assert(v.size() > 0 AndAlso that.v.size() > 0)
        Dim i As UInt32 = 0
        Dim c As UInt32 = 0
        For i = 0 To min(v.size(), that.v.size()) - uint32_1
            c = add(that.v.get(i), c, i)  ' [1]
#If DEBUG Then
            assert(c = 0 OrElse c = 1)
#End If
        Next

        If v.size() < that.v.size() Then
            v.resize(that.v.size())
            memcpy(v.data(), i, that.v.data(), i, that.v.size() - i)
        End If
        recursive_add(c, i)  ' [2]
        Return Me
    End Function
```

这里[1]是把第i位赋值为that.v.get(i)+v.get(i)+c的低32位，并返回高32位。而因为加法的进位最多为1，所以虽然是三个数相加，但最多进位仍然为1。不过因为add(uint, uint, uint)函数还会用在乘法中，所以其进位可能会最多是2。[service/math/big_uint/big_uint.private.vb#L67](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L67)

[2]是把c加到第i位上，如果仍有进位则再加入高位，直到没有进位位置。[service/math/big_uint/big_uint.private.vb#L73](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L73)

减法也是从低位到高位按位相减，记录退位，如果到了最高位仍然需要退位，则是溢出错误。因为减法中的每一“位”总是+a-b-c，所以退位最多是1。

同样实现上还有些考量，因为减法中即便被减数的某一位是0，也需要用4294967296减去减数的对应位，所以减法总是做到减数的位数。如果退位不为0，之后再处理退位。另外，如果减数是0，则直接返回被减数。

```
    Public Function [sub](ByVal that As big_uint, ByRef overflow As Boolean) As big_uint
        If that Is Nothing OrElse that.is_zero() Then
            overflow = False
            Return Me
        End If
        If that.v.size() > v.size() Then
            v.resize(that.v.size())
        End If
        assert(v.size() > 0 AndAlso that.v.size() > 0)
        Dim i As UInt32 = 0
        Dim c As UInt32 = 0
        For i = 0 To that.v.size() - uint32_1
            c = [sub](that.v.get(i), c, i)  ' [1]
        Next
        While i < v.size()
            c = [sub](c, i)  ' [2]
            If c = 0 Then
                Exit While
            End If
            i += uint32_1
        End While
        overflow = (c = uint32_1)
        remove_extra_blank()
        Return Me
    End Function
```

这里[1]是把第i位设置为v.get(i)-that.v.get(i)-c的低32位，如果其和为负数，则返回退位为1，否则返回退位为0。[service/math/big_uint/big_uint.private.vb#L35](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L35)

[2]是把第i位设置为v.get(i)-c的低32位，如果其和为负数，则返回退位为1，否则返回退位为0。[service/math/big_uint/big_uint.private.vb#L42](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L42)

在最新的实现中，[2]的函数被并入了和加法一致的[recursive_sub](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.private.vb#L41)函数。这样加法和减法的辅助函数就都是对称的了，这两对函数同样在除法中会有作用。

```
    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Function set_and_borrow(ByVal t As Int64, ByVal p As UInt32) As UInt32
        v.set(p, CUInt(t And max_uint32))
        Return If(t < 0, uint32_1, uint32_0)
    End Function

    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Sub sub_assertions(ByVal c As UInt32, ByVal p As UInt32)
        'this assert is too costly
#If DEBUG Then
        assert(p < v.size())
        assert(c = uint32_0 OrElse c = uint32_1)
#End If
    End Sub

    'sub d at position p with carry-over as c
    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Function [sub](ByVal d As UInt32, ByVal c As UInt32, ByVal p As UInt32) As UInt32
        sub_assertions(c, p)
        Return set_and_borrow(CLng(v.get(p)) - d - c, p)
    End Function

    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Function recursive_sub(ByVal c As UInt32, ByVal p As UInt32) As Boolean
        If c = 0 Then
            Return False
        End If
        While p < v.size()
            sub_assertions(c, p)
            c = set_and_borrow(CLng(v.get(p)) - c, p)
            If c = 0 Then
                Exit While
            End If
            p += uint32_1
        End While
        Return (c = uint32_1)
    End Function

    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Sub add_assertions(ByVal p As UInt32)
        'this assert is too costly
#If DEBUG Then
        assert(p < v.size())
#End If
    End Sub

    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Function set_and_carry(ByVal t As UInt64, ByVal p As UInt32) As UInt32
        v.set(p, CUInt(t And max_uint32))
        t >>= bit_count_in_uint32
#If DEBUG Then
        assert(t <= 2)
#End If
        Return CUInt(t)
    End Function

    'add d to the pos as p with carry-over as c
    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Function add(ByVal d As UInt32, ByVal c As UInt32, ByVal p As UInt32) As UInt32
        add_assertions(p)
        Return set_and_carry(CULng(v.get(p)) + c + d, p)
    End Function

    <MethodImpl(method_impl_options.aggressive_inlining)>
    Private Sub recursive_add(ByVal d As UInt32, ByVal p As UInt32)
        If d = 0 Then
            Return
        End If
        While p < v.size()
            add_assertions(p)
            d = set_and_carry(CULng(v.get(p)) + d, p)
            If d = 0 Then
                Return
            End If
            p += uint32_1
        End While
        v.push_back(d)
    End Sub
```
