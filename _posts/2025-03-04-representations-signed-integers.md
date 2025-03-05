---
title: 'Representations of signed integers'
tags: [DataRepresentation]
---

## Representations of signed integers

 - [Overview](#overview)
 - [Sign-magnitude](#sign-magnitude)
 - [One's complement](#ones-complement)
 - [Two's complement](#twos-complement)
 - [Offset binary](#offset-binary)
 - [Zig-Zag](#zig-zag)

---

---

### Overview

We will look together on a few possibilities how to encode signed integer data types.
Let us begin with an overview captured by the following table, where the leftmost column
holds binary representation and each other column shows appropriate meaning of that particular
signed integer encoding algorithm.

| Binary value | Sign-magnitude | One's Complement | Two's Complement | Offset binary | Zig-Zag |
|--------------|----------------|------------------|------------------|---------------|---------|
| `0000`       | `+0`           | `+0`             | `0`              | `-8`          | `0`     | 
| `0001`       | `+1`           | `+1`             | `+1`             | `-7`          | `-1`    | 
| `0010`       | `+2`           | `+2`             | `+2`             | `-6`          | `+1`    | 
| `0011`       | `+3`           | `+3`             | `+3`             | `-5`          | `-2`    | 
| `0100`       | `+4`           | `+4`             | `+4`             | `-4`          | `+2`    | 
| `0101`       | `+5`           | `+5`             | `+5`             | `-3`          | `-3`    | 
| `0110`       | `+6`           | `+6`             | `+6`             | `-2`          | `+3`    | 
| `0111`       | `+7`           | `+7`             | `+7`             | `-1`          | `-4`    |
| `1000`       | `-0`           | `-7`             | `-8`             | `0`           | `+4`    | 
| `1001`       | `-1`           | `-6`             | `-7`             | `+1`          | `-5`    | 
| `1010`       | `-2`           | `-5`             | `-6`             | `+2`          | `+5`    | 
| `1011`       | `-3`           | `-4`             | `-5`             | `+3`          | `-6`    | 
| `1100`       | `-4`           | `-3`             | `-4`             | `+4`          | `+6`    | 
| `1101`       | `-5`           | `-2`             | `-3`             | `+5`          | `-7`    | 
| `1110`       | `-6`           | `-1`             | `-2`             | `+6`          | `+7`    | 
| `1111`       | `-7`           | `-0`             | `-1`             | `+7`          | `-8`    |

The previous table presents the main approaches how to represent signed integers.
To understand the principles, it is enough to use just a nibble (aka 4 bits long
data type).


---

### Sign-magnitude

```
            ^
         *  |
          \ |
           \|
            *
            |  *
            | /
            |/
         ---*--->
```

As one might see, the MSB (Most Significant Bit) is used as so-called **sign bit**. 
All the bits but MSB represent unsigned **magnitude** of the value.
To convert between negative and positive value one might negate just a single bit.
For comparison of just magnitude, one might mask sign bit and compare just magnitude.

This representation was used by some early binary computers, e.g. IBM 7090.
Sign-magnitude is still the most common way of representing the significand (aka mantissa) of floating-point values.

**Highlights:**
 - 👍 symmetric
 - 👍 simple positive ←→ negative conversion (negation of MSB)

**Lowlights:**
 - 👎 both positive and negative zero 


---

### One's Complement

```
            ^
            *
           /|
          / |
         *  |
            |  *
            | /
            |/
         ---*--->
```

Conversion between positive and negative value is done by bitwise NOT operation (aka complement).
Like sign-magnitude representation, one's complement has two representations of zero (the one with all bits
cleared and the one with all bits set).

This representation is not used much these days.

**Highlights:**
 - 👍 simple positive ←→ negative conversion (bit-wise negation) 

**Lowlights:**
 - 👎 both positive and negative zero 
 - 👎 need to do an end-around carry


---

### Two's Complement

```
            ^
            o
           /|
          / |
         *  |
            |  *
            | /
            |/
         ---*--->
```

Negation is done by inverting all bits (using bitwise NOT operation) and then adding one to the result.
This last step is the reason of having just a single representation of zero.

This is the most often used native representation of signed integers in the currently used CPU architectures.

**Highlights:**
 - 👍 just one representation of zero value
 - 👍 negative value is always indicated by active MSB

**Lowlights:**
 - 👎 not so straightforward/intuitive positive ←→ negative conversion (needs two steps)


---

### Offset binary

```
            ^
            |
            |  *
            | /
            |/
            /   
           /|  
          / | 
         *------>
```

The offset binary representation is also known as **excess-K** or **biased**.

Signed number is represented by the bit pattern corresponding to the unsigned number plus **K**,
with K being known as **offset** or **biasing value** (or simple **bias**).
The size of offset might be designed as needed. Often, the middle value is used.

Offset binary (aka biased representations) are now primarily used for the exponent of floating-point numbers.
For single precision (32b wide) data type of IEEE-754 standard is using 8-bit excess-127 encoded exponent field;
while double precision (64b wide) data type is using 11-bit excess-1023.

**Highlights:**
 - 👍 quick comparison (enabling quick sifting of mantissa)

**Lowlights:**
 - 👎 not designed for calculations


---

### Zig-Zag

```
            ^
            |
            |  *
         *  |
            | *
          * |   
            |* 
           *| 
         ---*--->
```

Positive values **p** are encoded as even numbers (using formula 2 × **p**), 
while negative values **n** are encoded as odd numbers (using formula 2 × |**n**| - 1).
The encoding might be seen as "zig-zag"-ing between positive and negative numbers.

This representation of signed integers is used by Protocol Buffers' data types `sint32` and `sint64`.
The reason is to simply reduce a number of bits required to encode both small positive and negative values. 

**Highlights:**
 - 👍 positive → even and negative → odd
 - 👍 designed for encoding of small magnitude values (both positive and negative)

**Lowlights:**
 - 👎 not designed for calculations
