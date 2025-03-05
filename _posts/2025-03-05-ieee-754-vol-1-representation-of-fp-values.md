---
title: 'IEEE 754 vol. 1: Representations of floating-point values'
tags: [DataRepresentation]
---

## IEEE 754 vol. 1: Representations of floating-point values

 - [Supported data types](#supported-data-types)
 - [Representation of binary floating point values](#representation-of-binary-floating-point-values)
 - [Kind of floating-point values](#kind-of-floating-point-values)
 - [Normalized and denormalized numbers](#normalized-and-denormalized-numbers)

---

---

### Supported data types

IEEE 754 is a technical standard for floating-point arithmetic. It defines representation of several data types
of floating-point numbers. Mind that most FPUs on the current CPUs is based on that standard.

The most commonly used floating-point data types are those called `binary` in the standard (with data type width
in bits as a suffix). Those representations are optimized for calculations, quick comparison, etc.  

| common name         | std name  | width | sign bit | exponent | bias  | mantissa | valid digits | integer range | minimum denormalized number | minimum normalized number | maximum      |
|---------------------|-----------|-------|----------|----------|-------|----------|--------------|---------------|-----------------------------|---------------------------|--------------|
| half-precision      | binary16  | 16b   | 1b       | 5b       | 15    | 10+1b    | ~3.31        | ±2048         | 6 × 10⁻⁸                    | 6.1 × 10⁻⁴                | 6.5 × 10⁴    |
| single-precision    | binary32  | 32b   | 1b       | 8b       | 127   | 23+1b    | ~7.22        | ±1.6 × 10⁷    | 1.4 × 10⁻⁴⁵                 | 1.1 × 10⁻³⁸               | 3.4 × 10³⁸   |
| double-precision    | binary64  | 64b   | 1b       | 11b      | 1023  | 53+1b    | ~15.95       | ±9 × 10¹⁵     | 5 × 10⁻³²⁴                  | 2.2 × 10⁻³⁰⁸              | 1.7 × 10³⁰⁸  |
| extended-precision  | binary80  | 80b   | 1b       | 15b      | 16383 | 64b      | ~19.26       | ±1.8 × 10¹⁹   | 3.6 × 10⁻⁴⁹⁵¹               |                           | 1.1 × 10⁴⁹³² |
| quadruple-precision | binary128 | 128b  | 1b       | 15b      | 16383 | 112+1b   | ~34.01       | ±1 × 10³⁴     | 3.6 × 10⁻⁴⁹⁵¹               | 3.3 × 10⁻⁴⁹³²             | 1.1 × 10⁴⁹³² |
| -                   | bfloat16  | 16b   | 1b       | 8b       | -     | 7+1b     | ~2.40        | ±256          | 9.2 × 10⁻⁴¹                 | 1.1 × 10⁻³⁸               | 3.3 × 10³⁸   |

Depending on your programming language, selection of supported data types is usually limited. Many programming 
languages supports single-precision (`binary32`) and double-precision (`binary64`) data types.

Less frequently might be seen support for extended-precision (`binary80`) data type, which is native data type 
of x87 FPU (or at least it was done that way on x86 architecture of company Intel). This data type is supported
e.g. by Object Pascal programming language used by Delphi where it is called `Extended` or by C language
since standard C99 where it is known  `long double`.

There is one extraordinary point about `binary80` I would like to mention. Mantissa does not include one extra 
virtual/hidden/implicit bit for normalized numbers. All mantissa bits are explicitly stored in FPU register 
80 bits wide.

In the last row of the previous table is type of `bfloat16` that was included in one of the later revisions
of IEEE 754. It was designed to be used for AI, because it uses more bits for exponent (increasing overall range
of representable value) but also fewer bits for mantissa (decreasing precision and number of representable digits).

The standard also defines decimal types of different width. Those are stored using decimal format either
BID (aka Binary Integer Decimal) encoding or DPD (aka Densely Packed Decimal) encoding. Decimal types are
might be used e.g. for financial calculations as it supports also a different rounding algorithms etc.
All those decimal types are listed in the following table:

| std name   | width | sign bit | exponent range | mantissa digits |
|------------|-------|----------|----------------|-----------------|
| decimal32  | 32b   | 1b       | -95 … +96      | 7               |
| decimal64  | 64b   | 1b       | -383 … +384    | 16              |
| decimal128 | 128b  | 1b       | -6143 … +6144  | 34              |


---

### Representation of binary floating point values



```
   ---------------------------
   | s |   e   |      m      |
   ---------------------------
```

Where `s` stands for **sign bit** (yes, MSB is always a sign bit),
`e` stands for **exponent** (stored using Offset binary),
and `m` stands for mantissa (aka significand).

The pair of `s` and `m` might be seen as stored using 
Sign-magnitude representation.


---

### Kind of floating-point values

Floating-point value can be in any of the following kinds:

| Conditions                      | Kind of value          |
|---------------------------------|------------------------|
| 0 < e < max(e)                  | normalized number      |
| e == 0 && m != 0                | denormalized number    |
| e == 0 && m == 0 && s == 0      | positive zero (+0)     |
| e == 0 && m == 0 && s != 0      | negative zero (-0)     |
| e == max(e) && m == 0 && s == 0 | positive infinity (+∞) |
| e == max(e) && m == 0 && s != 0 | negative infinity (-∞) |
| e = max(e) && m != 0            | not-a-number (NaN)     |

The majority of representable values are so-called normalized numbers.
Values close to zero are denormalized numbers.
Floating-point types have a separate representation for positive and negative zeroes.
Although, many programming languages might hide sign of zero from both developers and users.
Positive and negative infinities might a result of certain calculations.
And last but not least, are so-called NaNs, that are used to denote an invalid result of calculation.

The following axis present whole range of representable floating-point value kinds:

```
  ^
  | 
  | not-a-numbers
  |-----
  | positive infinity
  |-----
  | 
  | positive normalized numbers
  |
  |-----
  |
  | positive denormalized numbers
  |-----
  | +0 (aka positive zero)
  |-----
  | -0 (aka negative zero)
  |-----
  |
  | negative denormalized numbers
  |-----
  |
  | negative normalized numbers
  |
  |-----
  | negative infinity
  |-----
  | not-a-numbers
  |
  v
```


---

### Normalized and denormalized numbers

Normalized number or simply _normal_ floating-point value has no leading zeroes in mantissa.
All the zeroes that might appear during calculation are removed by shifting the exponent.
Normalized numbers are calculated using the following formula:

```
     sign_bit                         (exponent - bias)
 (-1)           ×   1.mantissa   ×   2
```

Denormalized number (sometimes called _denormal_) is for IEEE 754 always a _subnormal_ number.
Subnormal number is any non-zero number with magnitude smaller than the smallest positive normal number.
Using other words, it is a floating-point value with leading zeroes in mantissa, so exponent cannot be
shifted anymore to finish the process of value normalization.
Denormalized numbers are calculated using the following formula:

```
     sign_bit                         (1 - bias)
 (-1)           ×  0.mantissa   ×   2
```
