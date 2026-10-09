# Calculating the Value of π - Arbitrary-Precision Arithmetic - big_uint

`big_uint` is a very common data structure. Simply put, it represents a number in base 4,294,967,296, and all operations are built on top of this radix and binary representation. Consequently, the data structure contains only a single variable-length array of `uint32`.

```
Partial Public NotInheritable Class big_uint
    ' Little-endian
    Private ReadOnly v As adaptive_array_uint32
...
End Class
```

`adaptive_array_uint32` is equivalent to a `vector<uint32>`, except that some functions unnecessary for this implementation have been stripped out for simplicity.

The functionality of `big_uint` is divided into several parts:

- Input and Output

Common primitive data types, such as `uint32`, `uint64`, and `boolean`

Byte arrays

Strings, supporting arbitrary bases

- Arithmetic Operations

Addition, subtraction, multiplication, division, modulo, exponentiation, roots, factorial, and greatest common divisor (GCD)

- Bitwise Operations

Bitwise AND, OR, and NOT, parity checks, bit shifting, and checking whether a number is a power of 2

- Comparisons

In addition, there are several .NET-specific design elements, such as overloading `ToString`, `Equals`, and operators, as well as helper functions designed for testing, like a `random` function that returns a random `big_uint`.
