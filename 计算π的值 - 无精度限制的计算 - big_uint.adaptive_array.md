# 计算π的值 - 无精度限制的计算 - big_uint.adaptive_array

感觉有点不值一提，不过还是稍微说一下。[service/math/big_uint/big_uint.adaptive_array.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/math/big_uint/big_uint.adaptive_array.vb)是一个和vector类似的数据结构，是一个可以自增大小的数组。[root/formation/vector.vb](https://github.com/Hzj-jie/osi.net/blob/master/root/formation/vector.vb)的实现也是基于同样的数据结构。adaptive_array本身由[root/codegen/adaptive_array](https://github.com/Hzj-jie/osi.net/tree/master/root/codegen/adaptive_array)生成的，这是一个很早的实现了，否则应该会用root/codegen/precompile。不过这个自然也不是很重要，这个数据结构相对稳定，已经很少有改动了。最近的改动都是为了提高big_uint的性能。

至于直接使用adaptive_array而不是vector的原因，主要是因为性能。

首先vector本身是模板，而模板在.net中是有性能损失的，具体可以看[root/tests/connector/generic_perf.vb](https://github.com/Hzj-jie/osi.net/blob/master/root/tests/connector/generic_perf.vb)，里面有一些常见的基于模板的操作，运行这个测试可以得到一些对比值，大约读写generic_class(Of Int32)中的数值比读写int_class要慢8%左右。这个8%会受到.net版本的影响，不过可以放心的认为模板的效率略低于无模板的类型。

其次，绝大多数的vector的操作会基于安全性的考虑增加一些断言。比如[get和set](https://github.com/Hzj-jie/osi.net/blob/master/root/formation/vector.vb#L57)会保证操作的数据不会超出边界，并且会对数据进行复制而不是引用。

而无精度限制的计算对性能要求非常的高，比方说在一台很差的电脑上（很好的电脑要拿来写代码）用了大约2分30秒计算出了e的前81655位，如果性能降低10%，则需要多15秒。所以通过codegen生成一个为了big_uint特制的big_uint.adaptive_array.vb相当的合理。

除此以外，似乎也没什么好说的了，大量的aggressive_inlining也是基于性能的考量。
