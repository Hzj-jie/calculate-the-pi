# Calculating the Value of π - Arbitrary-Precision Arithmetic (`calculate-the-pi`)

English translations of the article series **"计算π的值 - 无精度限制的计算"** (*Calculating the Value of π - Arbitrary-Precision Arithmetic*) by Zijie He, originally published on Zhihu ([Column c_1216677226609082368](https://zhuanlan.zhihu.com/c_1216677226609082368)).

This series documents the design, data structures, and algorithms behind the arbitrary-precision unsigned integer (`big_uint`) and arbitrary-precision unsigned rational/decimal (`big_udec`) implementations in `osi.net`, culminating in multi-million-digit computations of $e$ and $\pi$.

---

## Translated Articles (Chronological Order)

1. [Calculating the Value of π - Arbitrary-Precision Arithmetic - What It Is](<Calculating the Value of π - Arbitrary-Precision Arithmetic - What It Is.md>) / [计算π的值 - 无精度限制的计算 - 是什么](<计算π的值 - 无精度限制的计算 - 是什么.md>) (2020-03-27)
   - *Introduces why hardware-width integers and floats are insufficient for exact mathematical constants like $\pi$ and $e$, and outlines the `big_uint` (unsigned integer) and `big_udec` (unsigned fraction/decimal) types implemented in VB.NET (`osi.net`).*

2. [Calculating the Value of π - Arbitrary-Precision Arithmetic - big_uint](<Calculating the Value of π - Arbitrary-Precision Arithmetic - big_uint.md>) / [计算π的值 - 无精度限制的计算 - big_uint](<计算π的值 - 无精度限制的计算 - big_uint.md>) (2020-03-28)
   - *Summarizes the core data representation of `big_uint` as a little-endian base-$2^{32}$ array (`adaptive_array_uint32`) and lists its supported I/O conversions, arithmetic operations, bitwise operations, and comparisons.*

3. [Calculating the Value of π - Arbitrary-Precision Arithmetic - big_uint.adaptive_array](<Calculating the Value of π - Arbitrary-Precision Arithmetic - big_uint.adaptive_array.md>) / [计算π的值 - 无精度限制的计算 - big_uint.adaptive_array](<计算π的值 - 无精度限制的计算 - big_uint.adaptive_array.md>) (2020-04-04)
   - *Explains why `big_uint` uses a code-generated, specialized `adaptive_array` with aggressive inlining rather than a generic `vector<T>` to avoid .NET generics overhead (~8%) and bounds-checking assertions during intensive arbitrary-precision calculations.*

4. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Addition and Subtraction of big_uint](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Addition and Subtraction of big_uint.md>) / [计算π的值 - 无精度限制的计算 - big_uint的加减法](<计算π的值 - 无精度限制的计算 - big_uint的加减法.md>) (2020-04-05)
   - *Details `big_uint`'s mutable, chainable API design and three-variant error handling convention, then walks through base-$2^{32}$ limb-by-limb addition and subtraction with carry/borrow propagation (`set_and_carry`, `recursive_add`, `set_and_borrow`, `recursive_sub`).*

5. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Multiplication of big_uint](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Multiplication of big_uint.md>) / [计算π的值 - 无精度限制的计算 - big_uint的乘法](<计算π的值 - 无精度限制的计算 - big_uint的乘法.md>) (2020-04-05)
   - *Presents the $O(N^2)$ base-$2^{32}$ multiplication algorithm that fuses limb multiplication and addition in a single pass (`multiply_uint`), compares it against binary shift-and-add multiplication (`multiply_bit`), and shows the specialized `UInt32` multiplication overload.*

6. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Division and Modulo of big_uint](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Division and Modulo of big_uint.md>) / [计算π的值 - 无精度限制的计算 - big_uint的除法和取余](<计算π的值 - 无精度限制的计算 - big_uint的除法和取余.md>) (2020-04-06)
   - *Explains how `big_uint` implements arbitrary-precision integer division and modulo in base $2^{32}$ using a conservative trial-quotient estimate (`n \ (d + 1)`) refined with 64-bit leading-digit extraction (`select_significant_divide_fraction`).*

7. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Exponentiation of big_uint](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Exponentiation of big_uint.md>) / [计算π的值 - 无精度限制的计算 - big_uint的次方](<计算π的值 - 无精度限制的计算 - big_uint的次方.md>) (2020-04-08)
   - *Analyzes the digit-multiplication complexity of naive repeated multiplication versus exponentiation by squaring, and demonstrates how `big_uint.power` uses binary exponentiation to reduce computational and memory overhead.*

8. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Root Extraction of big_uint](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Root Extraction of big_uint.md>) / [计算π的值 - 无精度限制的计算 - big_uint的开方](<计算π的值 - 无精度限制的计算 - big_uint的开方.md>) (2020-04-09)
   - *Describes how `big_uint.extract` computes the integer $N$-th root and remainder of a large unsigned integer by bounding the result's bit length to $\lfloor\text{bits}/N\rfloor + 1$ and testing each bit from most significant to least significant.*

9. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Bitwise Operations of big_uint](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Bitwise Operations of big_uint.md>) / [计算π的值 - 无精度限制的计算 - big_uint的位运算](<计算π的值 - 无精度限制的计算 - big_uint的位运算.md>) (2020-04-09)
   - *Documents the word-by-word implementation of bitwise `AND`, `OR`, `XOR`, and `NOT` operations on `big_uint` along with trailing zero-word cleanup.*

10. [Calculating the Value of π - Arbitrary-Precision Arithmetic - String Input and Output of big_uint](<Calculating the Value of π - Arbitrary-Precision Arithmetic - String Input and Output of big_uint.md>) / [计算π的值 - 无精度限制的计算 - big_uint的字符输入输出](<计算π的值 - 无精度限制的计算 - big_uint的字符输入输出.md>) (2020-04-12)
    - *Details how `big_uint.str` and `big_uint.parse` accelerate arbitrary-base string conversion by grouping digits into `uint32`-sized chunks via `digit_count_per_parse` rather than converting one character digit at a time.*

11. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Greatest Common Divisor of big_uint](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Greatest Common Divisor of big_uint.md>) / [计算π的值 - 无精度限制的计算 - big_uint的最大公约数](<计算π的值 - 无精度限制的计算 - big_uint的最大公约数.md>) (2020-04-20)
    - *Discusses GCD algorithms for `big_uint` (Euclidean, Lehmer's GCD, and binary GCD properties) and presents a hybrid GCD implementation that combines trailing-zero bit shifts, subtraction, and modulo reduction.*

12. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Basic Structure of big_udec](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Basic Structure of big_udec.md>) / [计算π的值 - 无精度限制的计算 - big_udec的基本结构](<计算π的值 - 无精度限制的计算 - big_udec的基本结构.md>) (2020-04-20)
    - *Introduces `big_udec`, which represents arbitrary-precision non-negative numbers as a fraction `n / d` composed of two `big_uint` instances, along with its planned I/O, arithmetic, fraction reduction, and comparison capabilities.*

13. [Calculating the Value of π - Arbitrary-Precision Arithmetic - From Floating-Point Numbers to big_udec](<Calculating the Value of π - Arbitrary-Precision Arithmetic - From Floating-Point Numbers to big_udec.md>) / [计算π的值 - 无精度限制的计算 - 从浮点数到big_udec](<计算π的值 - 无精度限制的计算 - 从浮点数到big_udec.md>) (2020-04-21)
    - *Explains IEEE-754 floating-point representation limitations and shows how `big_udec` converts floating-point values (`Double`, `Decimal`) into fractions in a language-independent manner by repeatedly multiplying by 2 up to `max_shift` times.*

14. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Comparison of big_udec](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Comparison of big_udec.md>) / [计算π的值 - 无精度限制的计算 - big_udec的比较](<计算π的值 - 无精度限制的计算 - big_udec的比较.md>) (2020-04-21)
    - *Describes how `big_udec` implements `is_zero()`, `is_one()`, and `compare()` using fast numerator/denominator magnitude checks before falling back to cross-multiplication (`this.n * that.d` vs. `this.d * that.n`).*

15. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Addition and Subtraction of big_udec](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Addition and Subtraction of big_udec.md>) / [计算π的值 - 无精度限制的计算 - big_udec的加减法](<计算π的值 - 无精度限制的计算 - big_udec的加减法.md>) (2020-04-21)
    - *Details `big_udec` addition, subtraction, and fraction reduction strategies, explaining why routine operations use lightweight power-of-2 bit shifts (`fast_reduce_fraction`) while full GCD reduction is reserved for explicit calls.*

16. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Multiplication, Division, and Exponentiation of big_udec](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Multiplication, Division, and Exponentiation of big_udec.md>) / [计算π的值 - 无精度限制的计算 - big_udec的乘除法次方](<计算π的值 - 无精度限制的计算 - big_udec的乘除法次方.md>) (2020-04-22)
    - *Explains how `big_udec` implements multiplication and division via cross-multiplication with `fast_reduce_fraction` (including division-by-zero handling), and exponentiation by raising the numerator and denominator to the target power independently.*

17. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Root Extraction of big_udec](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Root Extraction of big_udec.md>) / [计算π的值 - 无精度限制的计算 - big_udec的开方](<计算π的值 - 无精度限制的计算 - big_udec的开方.md>) (2020-04-22)
    - *Describes approximating the $n$-th root of a rational `big_udec` by scaling the numerator and denominator with a heuristic power base $p$ so the denominator becomes a perfect $n$-th power and only the numerator requires integer root extraction.*

18. [Calculating the Value of π - Arbitrary-Precision Arithmetic - String Representation of big_udec](<Calculating the Value of π - Arbitrary-Precision Arithmetic - String Representation of big_udec.md>) / [计算π的值 - 无精度限制的计算 - big_udec的字符串表示](<计算π的值 - 无精度限制的计算 - big_udec的字符串表示.md>) (2020-04-24)
    - *Covers exact fractional string serialization/parsing, precision-bounded decimal string conversion controlled by `upure_len`, and the `str_match` method for counting matching leading digits against a reference string.*

19. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Calculating e First](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Calculating e First.md>) / [计算π的值 - 无精度限制的计算 - 先来算e](<计算π的值 - 无精度限制的计算 - 先来算e.md>) (2020-04-24)
    - *Compares the limit definition ${(1 + \frac{1}{x})}^{x}$ against the iterative Taylor series $1 + \sum_{x=1}^{N}{\frac{1}{x!}}$ using `big_udec`, reaching over two million and eight million digits of precision for $e$.*

20. [Calculating the Value of π - Arbitrary-Precision Arithmetic - Calculating the Value of π](<Calculating the Value of π - Arbitrary-Precision Arithmetic - Calculating the Value of π.md>) / [计算π的值 - 无精度限制的计算 - 计算π的值](<计算π的值 - 无精度限制的计算 - 计算π的值.md>) (2020-04-24)
    - *Evaluates several series for computing $\pi$ with `big_udec`—showing why Gregory-Leibniz, Nilakantha, and Wallis converge slowly while the arctangent series and BBP formula achieve over 300,000 and 1.2 million digits of precision respectively after one million iterations.*

---

## Precomputed Digit Files

- **Digits of $e$**: [`e.2m.txt`](e.2m.txt), [`e.8m.txt`](e.8m.txt), [`e.10m.txt`](e.10m.txt)
- **Digits of $\pi$**: [`pi.1m.txt`](pi.1m.txt), [`pi.1.2m.txt`](pi.1.2m.txt), [`pi.1.8m.txt`](pi.1.8m.txt)
