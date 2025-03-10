---
title: 'Unicode overview'
tags: [DataRepresentation]
---

## Unicode overview

 - [Unicode standard](#unicode-standard)
     - [Unicode planes and BMP](#unicode-planes-and-bmp)
     - [General categories](#general-categories)
 - [Normalization forms](#normalization-forms)
 - [Endianness](#endianness)
 - [BOM (Byte Order Mark)](#bom-byte-order-mark)
 - [UCS-2](#ucs-2)
 - [UTF-32 (UCS-4)](#utf-32-ucs-4)
 - [UTF-16](#utf-16)
     - [Surrogate pairs](#surrogate-pairs)
 - [UTF-8](#utf-8)
     - [Original standard of UTF-8 encoding](#original-standard-of-utf-8-encoding)
     - [UTF-8 byte map](#utf-8-byte-map)
 - [UTF-7](#utf-7)

---

---

### Unicode standard

**Unicode**, formally **The Unicode Standard**, is a text encoding standard designed to support all the writing 
systems of the world. It might be also know under alias **Universal Coded character Set** (with acronym of **UCS**).
The standard includes many common characters, numerals, punctuations and plethora of other symbols (including more
than three thousand emojis).

History of Unicode standard began with version 1.0 released in 1991. In that initial version was introduced just
24 scripts with 7 129 characters. Version 2.0 was introduced in 1996 brought UTF-16 encoding and caped the limit
of supported characters. The first release with more than 100 000 characters included was Version 5.1 from 2008. 
Version 6.0 from 2010 joined the very first batch of over 700 emojis. With Version 8.0 in 2015 was introduced
5 emoji skin tone modifiers. The first crypto imprint of Bitcoin sign currency symbol appeared within Version 10
in 2017. And so the standard was extended by brand-new characters and scripts until the latest release of Version
16.0 from September 2024 that already supported 168 scripts and 154 998 characters.

Basic principles of Unicode standard (Version 16.0) are:
 - _Universality_ — endeavor to support all scripts and graphemes in the world (effort spread across released version);
 - _Efficiency_ — Unicode text is simple to parse and process;
 - _Characters, not glyphs_ — standard encodes characters (semantic items), not glyphs (visual representations);
 - _Semantics_ — each character has well-defined semantics;
 - _Plain text_ — Unicode characters represent plain text;
 - _Logical order_ — the default for memory representation is logical order;
 - _Unification_ — unification of duplicate characters within scripts across languages;
 - _Dynamic_ composition — accented forms can be dynamically composed (dead key principle);
 - _Stability_ — each character is assigned just once and its key properties are immutable;
 - _Convertibility_ — accurate convertibility is guaranteed between the Unicode Standard and other widely accepted
   standards.

Basic terminology:
 - **Character** — is not a preferable term, as it can have multiple meanings.
 - **Code point** — is an atomic unit of information represented by unique number which is given meaning by the Unicode
   standard. Unicode defines the codespace as a sequence of code points in the range from `U+0000`—`U+10FFFF`.
   In this normative notation is always present a two-character prefix `U+` followed by code point itself written
   as hexadecimal number. Minimum of four hexadecimal digits is required (prepending leading zeroes), but for
   five- or six-digit code point no leading zero shall be added.
 - **Text** — is a sequence of code points.
 - **Code unit** — is an atomic unit of storage used to store a part of a code point. A single code unit might
   represent a full code point, or just a part of it. Each encoding has its own code unit, namely 8 bits for UTF-8,
   16 bits for UTF-16, and 32 bits for UTF-32. For example `€` (`EURO SIGN`) with code point `U+20AC` requires
   one UTF-16 code unit (namely `0x20AC`), but three UTF-8 code units (namely `0xE2 0x82 0xAC`). 
 - **Grapheme** — is a sequence of one or multiple code points that are displayed as a single element of writing system.
   Mind that some code points are never part of any grapheme (e.g. zero-width non-joiner, or directional overrides).
 - **Glyph** — is a graphical representation of grapheme or part thereof, usually stored in a font.

A lot of code points are not yet assigned to any grapheme. They are reserved to be assigned for particular use
of any future versions of the Unicode standard. 

Every grapheme is described by a set of properties, e.g. name (`LATIN CAPITAL LETTER A`), code point (`U+0041`),
representative glyph (`A`), general category (`Lu` meaning `Letter, uppercase`), decimal digit value (`NaN`), 
digit value (`NaN`), numeric value (`NaN`), deprecated flag (`false`), few more flags and other properties.


#### Unicode planes and BMP

At the beginning, Unicode was designed for storing of more than 2×10⁹ graphemes (thanks to support of 31-bit encoding).
However, since 2003 was design limited to handle just 1 114 112 graphemes in theory. The real limit of 
representable code points is caped to 1 111 998 code points available for use (after exclusion of surrogates
and non-characters). Still, this might be sufficient space.

Range was divided into 17 planes (called Plane 0, Plane 1, …, Plane 16) with up to 65 536 graphemes (in theory)
in each plane. Within each plane, code points are allocated within named blocks of related characters.
The size of each block is a multiple of 16.

The lowest plane (Plane 0) is often denoted as **BMP** (aka **Basic Multilingual Plane**). Every grapheme of BMP
might be represented using single UTF-16 code point. It is obvious that BMP contains the most commonly used 
graphemes. While graphemes outside BMP requires two UTF-16 code points using so-called **surrogate pairs**.


#### General categories

Major general categories are:
 - `L` (`Letter`)
    - `LC` (`Cased Letter`) is a subset of `L` major general category
 - `M` (`Mark`)
 - `N` (`Number`)
 - `P` (`Punctuation`)
 - `S` (`Symbol`)
 - `Z` (`Separator`)
 - `C` (`Other`)

| Value | Major categories | Category                     | Remarks                                                                                                                       |
|-------|------------------|------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| `Lu`  | `L`, `LC`        | `Letter, uppercase`          |                                                                                                                               |
| `Ll`  | `L`, `LC`        | `Letter, lowercase`          |                                                                                                                               |
| `Lt`  | `L`, `LC`        | `Letter, titlecase`          | Ligatures or digraphs (uppercase followed by a lowercase part).                                                               |
| `Lm`  | `L`              | `Letter, modifier`           | A modifier letter.                                                                                                            |
| `Lo`  | `L`              | `Letter, other`              | An ideograph or a letter in a unicase alphabet.                                                                               |
| `Mn`  | `M`              | `Mark, nonspacing`           |                                                                                                                               |
| `Mc`  | `M`              | `Mark, spacing combining`    |                                                                                                                               |
| `Me`  | `M`              | `Mark, enclosing`            |                                                                                                                               |
| `Nd`  | `N`              | `Number, decimal digit`      | All these, and only these, have Numeric Type = `De`.                                                                          |
| `Nl`  | `N`              | `Number, letter`             | Composed of letters or letterlike symbols (e.g. Roman numerals).                                                              |
| `No`  | `N`              | `Number, other`              | Vulgar fractions, superscript, subscript, vigesimal digits.                                                                   |
| `Pc`  | `P`              | `Punctuation, connector`     | E.g. spacing underscore characters and other tie-spacing chars.                                                               |
| `Pd`  | `P`              | `Punctuation, dash`          |                                                                                                                               |
| `Ps`  | `P`              | `Punctuation, open`          | Opening bracket characters.                                                                                                   |
| `Pe`  | `P`              | `Punctuation, close`         | Closing bracket characters.                                                                                                   |
| `Pi`  | `P`              | `Punctuation, initial quote` | Opening quotation mark (no neutral ?). Like `Ps` of Pe`.                                                                      |
| `Pf`  | `P`              | `Punctuation, final quote`   | Closing quotation mark (no neutral ?). Like `Ps` of Pe`.                                                                      |
| `Po`  | `P`              | `Punctuation, other`         |                                                                                                                               |
| `Sm`  | `S`              | `Symbol, math`               |                                                                                                                               |
| `Sc`  | `S`              | `Symbol, currency`           |                                                                                                                               |
| `Sk`  | `S`              | `Symbol, modifier`           |                                                                                                                               |
| `So`  | `S`              | `Symbol, other`              |                                                                                                                               |
| `Zs`  | `Z`              | `Separator, space`           | Includes the space, but not `TAB`, `CR`, or `LF`.                                                                             |
| `Zl`  | `Z`              | `Separator, line`            | Only `U+2028 LINE SEPARATOR` (`LSEP`). Basic type = Format.                                                                   |
| `Zp`  | `Z`              | `Separator, paragraph`       | Only `U+2029 PARAGRAPH SEPARATOR` (`PSEP`). Basic type = Format.                                                              |
| `Cc`  | `C`              | `Other, control`             | 65 characters. Basic type = Control.                                                                                          |
| `Cf`  | `C`              | `Other, format`              | Soft hyphen, joining CCs (ZWNJ and ZWJ), CCs to support bidirectional text, and language tag characters. Basic type = Format. |
| `Cs`  | `C`              | `Other, surrogate`           | Basic type = Surrogate. Used only in UTC-16.                                                                                  |
| `Co`  | `C`              | `Other, private use`         | Basic type = Private-use.                                                                                                     |
| `Cn`  | `C`              | `Other, not assigned`        | Basic type = Noncharacter or Reserved.                                                                                        |


---

### Normalization forms

Unicode for some characters allows multiple ways of representation while keeping the same meaning and/or
graphical representation. Let us introduce a few important characters to be able to understand the latter
presented examples:
 - Character `á` might be represented either by:
     - composed form with just a single code point of `U+00E1`;
     - decomposed form with two code points: `U+0061` (aka `a`) and `U+0301` (aka `COMBINING ACUTE ACCENT`).
 - Representations of character `ó`:
     - composed form: `U+00F3`;
     - decomposed form: `U+006F` followed by `U+0301`.
 - Character `ở` also has two representations:
     - fully composed form: `U+1EDF` (aka `LATIN SMALL LETTER O WITH HORN AND HOOK ABOVE`) with both tonal mark
       and vowel applied to the base character of `o`;
     - fully decomposed form: `o`, `U+031B` (aka `COMBINING HORN`), `U+0309` (aka `COMBINING HOOK ABOVE`);
     - partially decomposed form: `U+01A1` (aka `LATIN SMALL LETTER O WITH HORN`), `U+0309`;
     - and yet another partially decomposed form: `U+1ECF` (aka `LATIN SMALL LETTER O WITH HOOK ABOVE`), `U+031B`.
 - Unicode has a single character `U+FB03` (aka `LATIN SMALL LIGATURE FFI`) to represent ligature _ffi_.
   Searching and comparison for this character might be tricky (as one might use 3 ASCII characters on input).
 - To show how tricky might be digits, we will use superscript digit `⁵` (aka `SUPERSCRIPT FIVE`) with code point 
   `U+2075` and `②` (aka `CIRCLED DIGIT TWO`) with code point `U+2461`.

Now, we have enough data to understand why Unicode supports four normalization forms (for details see
[Unicode Standard Annex #15](https://unicode.org/reports/tr15)):

| Long name             | Abbr. name | Description                                                    |
|-----------------------|------------|----------------------------------------------------------------|
| Normalization Form D  | NFD        | Canonical decomposition.                                       | 
| Normalization Form C  | NFC        | Canonical decomposition followed by canonical composition.     | 
| Normalization Form KD | NFKD       | Compatibility decomposition.                                   | 
| Normalization Form KC | NFKC       | Compatibility decomposition followed by canonical composition. | 

For the following examples, we might use escaping of Unicode code points used e.g. by Rust programming language.
There is each Unicode character presented as `\u{x}` where `x` is 1—6 hexadecimal characters while all usual
characters will be kept.

> **Example 1:** Let us use have a word `áhój` (truly non-existing word, but simple enough for our purpose).
> Due to having there 2 special characters, we might see it in 4 different forms:
>  - `\u{e1}h\u{f3}j` as fully composed form,
>  - `a\u{301}ho\u{301}j` as fully decomposed form,
>  - and denormalized forms of `\u{e1}ho\u{301}j`, resp. `a\u{301}h\u{f3}j`.
> 
> Mind that those forms cannot be compared to be equal (although they are graphically represented to be
> exactly the same) and space storage is also different. All possible combinations are captured by the
> following table:
> 
> | x                    | NFC(x) or NFKC(x)        | NFD(x) or NFKD(x)            |
> |----------------------|--------------------------|------------------------------|
> | `\u{e1}h\u{f3}j`     | `\u{e1}h\u{f3}j` (no-op) | `a\u{301}ho\u{301}j`         |
> | `\u{e1}ho\u{301}j`   | `\u{e1}h\u{f3}j`         | `a\u{301}ho\u{301}j`         |
> | `a\u{301}h\u{f3}j`   | `\u{e1}h\u{f3}j`         | `a\u{301}ho\u{301}j`         |
> | `a\u{301}ho\u{301}j` | `\u{e1}h\u{f3}j`         | `a\u{301}ho\u{301}j` (no-op) |
> 
> **Summary:** Algorithms for normalization are stable and idempotent. For the given input, 
> compatibility decomposition had no effect.


> **Example 2:** Vietnamese word `Phở` is will show us a different edge case. We have a character that might be
> either composed in a form of one, two, or three Unicode code points. Let us check whether a different composition
> of `ở` grapheme affects output of normalization algorithms:
> 
> | x                     | NFC(x) or NFKC(x)    | NFD(x) or NFKD(x)           |
> |-----------------------|----------------------|-----------------------------|
> | `Ph\u{1edf}`          | `Ph\u{1edf}` (no-op) | `Pho\u{31b}\u{309}`         |
> | `Ph\u{01a1}\u{309}`   | `Ph\u{1edf}`         | `Pho\u{31b}\u{309}`         |
> | `Ph\u{1ecf}\u{31b}`   | `Ph\u{1edf}`         | `Pho\u{31b}\u{309}`         |
> | `Pho\u{309}\u{31b}`   | `Ph\u{1edf}`         | `Pho\u{31b}\u{309}`         |
> | `Pho\u{31b}\u{309}`   | `Ph\u{1edf}`         | `Pho\u{31b}\u{309}` (no-op) |
> 
> **Summary:** Order and composition of grapheme on the input side have no effect on any of normalized forms.


> **Example 3:** The previous two examples presented an idea of composed and decomposed representations
> that has the same graphical representation while having different memory representation.
> Mind that normalized forms C and KC had exactly the same output (same is true for NFD and NFKD).
> 
> Word `②⁵ diﬃcult` is a nice example for presentation of so-called **compatibility equivalent** of three
> specific graphemes (namely circled number two, superscript number five, and `ffi` ligature):
> 
> | x                               | NFC(x) or NFD(x)                | NFKC(x) or NFKD(x)   | 
> |---------------------------------|---------------------------------|----------------------|
> | 25 difficult                    | 25 difficult (no-op)            | 25 difficult (no-op) |
> | \u{2461}\u{2075} di\u{FB03}cult | \u{2461}\u{2075} di\u{FB03}cult | 25 difficult         |
> 
> **Summary:** Both normalization into NFKC and NFKD pre-process the input in the same way, applying
> compatibility decomposition to each grapheme of the input. This caused extraction/simplification of digits
> and substitution/unpacking of ligature.
> 
> **Hint:** No difference between canonical C and D normalization forms is caused by choosing artificial input,
> where is nothing to be handled (simply add one grapheme with diacritic and the difference will be visible
> right away as in the previous examples).

Example of unit test in Rust programming language:

```rust
use unicode_normalization::UnicodeNormalization;

#[cfg(test)]

#[test]
fn test_unicode_normalization() {
    let ahoj: [&str; 5] = [
        "áhój",
        "\u{e1}h\u{f3}j",
        "\u{e1}ho\u{301}j", "a\u{301}h\u{f3}j",
        "a\u{301}ho\u{301}j",
    ];

    ahoj.iter().for_each(|x: &&str| assert_eq!(x.nfc().collect::<String>(), "\u{E1}h\u{F3}j"));
    ahoj.iter().for_each(|x: &&str| assert_eq!(x.nfkc().collect::<String>(), "\u{E1}h\u{F3}j"));
    ahoj.iter().for_each(|x: &&str| assert_eq!(x.nfd().collect::<String>(), "a\u{301}ho\u{301}j"));
    ahoj.iter().for_each(|x: &&str| assert_eq!(x.nfkd().collect::<String>(), "a\u{301}ho\u{301}j"));

    let pho: [&str; 6] = [
        "Phở",
        "Ph\u{1EDF}",
        "Ph\u{01A1}\u{309}", "Ph\u{1ECF}\u{31B}",
        "Pho\u{309}\u{31B}", "Pho\u{31B}\u{309}",
    ];

    pho.iter().for_each(|x: &&str| assert_eq!(x.nfc().collect::<String>(), "Ph\u{1EDF}"));
    pho.iter().for_each(|x: &&str| assert_eq!(x.nfkc().collect::<String>(), "Ph\u{1EDF}"));
    pho.iter().for_each(|x: &&str| assert_eq!(x.nfd().collect::<String>(), "Pho\u{31B}\u{309}"));
    pho.iter().for_each(|x: &&str| assert_eq!(x.nfkd().collect::<String>(), "Pho\u{31B}\u{309}"));

    let difficult: [&str; 2] = ["25 difficult", "\u{2461}\u{2075} di\u{FB03}cult"];

    difficult.iter().for_each(|x: &&str| assert_eq!(x.nfc().collect::<String>(), x.to_string()));
    difficult.iter().for_each(|x: &&str| assert_eq!(x.nfkc().collect::<String>(), "25 difficult"));
    difficult.iter().for_each(|x: &&str| assert_eq!(x.nfd().collect::<String>(), x.to_string()));
    difficult.iter().for_each(|x: &&str| assert_eq!(x.nfkd().collect::<String>(), "25 difficult"));
}
```

The Unicode NFC and NFD use the maximally composed and maximally decomposed forms of each grapheme, but not unify
compatibility equivalent sequences. On the other hand NFKC and NFKD normalization form are like NFC and NFD, but
normalize all compatibility equivalent sequences to some simple representative of their class.

W3 Consortium recommends using NFC for all content. 
[The Unicode Identifier and Pattern Syntax annex](https://www.unicode.org/reports/tr31/)
recommends using NFKC for identifiers in programming languages.
For removal of accents (e.g. due to input from unknown source) might be used NFKD with some further processing
(e.g. stripping of non-ASCII code points).

---

### Endianness

In IT, endianness is the order in which bytes within a word of digital data are:
 - transmitted over a communication medium,
 - addressed in computer memory,
 - or stored in a file.

Let us have a 32-bit integer value `0x04030201` stored in computer memory. **Little-endian** keeps 
the least significant byte on the address with the lowest address and each more significant byte is
stored in memory address with higher address. The following illustration demonstrates how little-endian
stores the given integer's byte in the computer memory:

```text
     Little-endian:
     ...  | a      | a + 1  | a + 2  | a + 3  |  ...
   -------|--------|--------|--------|--------|-------
     ...  |  0x01  |  0x02  |  0x03  |  0x04  |  ...
   -------|--------|--------|--------|--------|-------
```

Little-endian is used by the most widespread CPU architectures nowadays, e.g. Intel x86 and Apple M1.

On the other hand, **bit-endian** starts from the most significant byte on the lowest memory address
and storing each consecutive byte on higher memory addresses:

```text
     Big-endian:
     ...  | a      | a + 1  | a + 2  | a + 3  |  ...
   -------|--------|--------|--------|--------|-------
     ...  |  0x04  |  0x03  |  0x02  |  0x01  |  ...
   -------|--------|--------|--------|--------|-------
```

> One might also face exotic architectures with _mixed-endian_ (aka _middle-endian_) like PDP-11. Then one might
> read value of 0x04030201 in order (0x03, 0x04, 0x01, 0x02) or (0x02, 0x01, 0x04, 0x03). 

One might face endianness also with other kinds of data. For example date is expressed by year, month, and day.
Little-endian date format `day-month-year` is used by the majority of the world. Big-endian date format 
`year-month-day` was adopted by the international standard [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601).
One might also find mixed-endian date formats like `month-day-year` used in the USA or even extremely rare 
format `year-day-month`.


---

### BOM (Byte Order Mark)

In the head of text file might appear the **Bote Order Mark**, a sequence of bytes that hints at the used
encoding form and its byte order (aka endianness). All the valid Unicode BOMs can be found in the following
table:

| BOM                   | Code space      | Encoding                          | Code unit size | Code unit count | Max size  |
|-----------------------|-----------------|-----------------------------------|----------------|-----------------|-----------|
| `0xEF 0xBB 0xBF`      | 21-bit (31-bit) | UTF-8                             | 1 B            | 1—4 (1—6)       | 4 B (6 B) |
| `0xFE 0xFF`           | 21-bit (16-bit) | UTF-16-BE (UCS-2-BE)              | 2 B            | 1—2 (1)         | 2 B (4 B) |
| `0xFF 0xFE`           | 21-bit (16-bit) | UTF-16-LE (UCS-2-LE)              | 2 B            | 1—2 (1)         | 2 B (4 B) |
| `0x00 0x00 0xFE 0xFF` | 32-bit          | UTF-32-BE                         | 4 B            | 1               | 4 B       |
| `0xFF 0xFE 0x00 0x00` | 32-bit          | UTF-32-LE                         | 4 B            | 1               | 4 B       |
| —                     | 7-bit or 8-bit  | ASCII (optionally with code page) | 1 B            | 1               | 1 B       |

As one might see BOM has code point of `U+FEFF` and little endian variant of UTF-16 encoding just project
the different order of bytes. And both variants of UTF-32 encoding obeys the same rule, but width of
BOM is extended to the size of one code unit (4 bytes).


---

### UCS-2

Name of encoding might be translated into 2-byte Universal Character Set. 
This encoding was introduced early, but it is obsolete since 1996 (when it became clear that 65 536
code points was not enough). Code points outside the BMP cannot be represented using UCS-2,
therefore UCS-2 can be found a proper subset of UTF-16.

Advantage of UTC-2 over UTF-16 or UTF-8 is constant length of grapheme, so count of characters
in a string might be easily calculated. If the text contains just code points from BMP, there is
no way to distinguish if UCS-2 or UTF-16 was used.

UCS-2 was used in Windows 95, but it was replaced by UTF-16 already in Windows 2000.

There are little-endian and big-endian variants: UCS-2-LE and UCS-2-BE available.


---

### UTF-32 (UCS-4)

The originally introduced encoding was UCS-4 (4-byte Universal Character Set). As far as I know, UTF-32 should be
an alias of UCS-4 since range of characters by Unicode standard was caped. UTF-32 was originally supposed to
use more code units to represent wider range of code points.

It is a fixed-length encoding capable of representing the full range of Unicode code points. It uses exactly 32 bits
per code point. Due to need of just 21 bits, leading zeroes will always be included. In contrast, all other Unicode
encodings are variable-length encodings. Each 32-bit value in UTF-32 is exactly equal to a represented code point.

The main advantage is that grapheme count and direct indexation is possible as a constant time operation.

This encoding is space-inefficient due to using 4 bytes even for each ASCII character as code points outside the BMP
are relatively rare in most text (except asian languages). As another downside might be also see 11 always-zero 
bits is a prefix of code point.

There are little-endian and big-endian variants: UTF-32-LE and UTF-32-BE available.


---

### UTF-16

UTF-16 is used by the Windows API and by many programming environments such as Delphi, .NET Framework, Java, and Qt.
Variable length character of UTF-16 and fact of code points outside the BMP being rarely used (and rarely
tested) caused many software bugs (also in Windows OS).

Advantage of UTF-16 encoding is that text is more concise (thanks to a majority of code points being encoded by
just 2 bytes). However, counting number of code points require to process the complete text.

There are little-endian and big-endian variants: UTF-16-LE and UTF-16-BE available.


#### Surrogate pairs

BMP has the range of `U+D800`—`U+DFFF` (exactly 2¹¹ code points), which are used as surrogate pairs to encode
2²⁰ code points of higher planes (therefore to represent all the range `U+10000`—`U+10FFFF`).

The block of surrogate code points is divided into two ranges (both of 1024 code points):
 - `U+DC00`—`U+DFFF` are known as **low-surrogate** code points
 - `U+D800`—`U+DBFF` are **high-surrogate** code points

Each surrogate pair is formed by one high-surrogate code point followed by one low-surrogate code point. 
Surrogate pairs can appear only in UTF-16 encoding. Each surrogate pair exist in order to represent code point
greater than `U+FFFF` (aka code point outside the BMP range).

It is worth to mention that surrogate code points are valid only in UTF-16. Other Unicode encodings like UTF-18
or UTF-32 are not supposed to present any surrogate code points.

In UTF-16 encoding a lone surrogate code point is considered invalid. They have to be used in the pairs
as stated above (one high-surrogate and one low-surrogate). Examples of valid surrogates:

| code point | surrogate pair     |
|------------|--------------------|
| `U+10000`  | `U+D800`, `U+DC00` |
| `U+10E6D`  | `U+D803`, `U+DE6D` |
| `U+1D11E`  | `U+D834`, `U+DD1E` |
| `U+10FFFF` | `U+DBFF`, `U+DFFF` |

The encoding algorithm is following:
 - Subtract 0x10000 from a code point outside the BMP.
 - To obtain a high-surrogate, shift right by 10 bits (or divide by 0x400) and then add 0xD800.
 - For a low-surrogate, take the low 10 bits (preferably using AND mask 0x3FF) and then add 0xDC00.

The decoding algorithm (extraction of code point from surrogate pair) is reversal:
 - Take a high-surrogate, subtract from it 0xD800, and shift it left by 10 bits (or multiply by 0x400).
 - Subtract 0xDC00 from a low-surrogate.
 - Add results from the previous two steps together with 0x10000.


---

### UTF-8

UTF-8 is a multibyte encoding (1—4 bytes; for the older 31-bit standard it was 1—6 bytes) able to
encode the whole Unicode code point space. The main advantage is backward compatibility with all
ASCII text. Almost every web page is stored using UTF-8. Another advantage is that UTF-8 is 
**endianless** (thanks to using a single byte as a code unit).

The main drawback comes from being a variable length encoding. There has to be used a linear complexity 
algorithm both to count the number of code points and to index the desired code point (e.g. what is 20th
character of some given text). One has to iterate a lot on UTF-8 encoded text. 

It is possible to ensure that a byte stream is encoded to UTF-8, because marker (a prefix of two or more bits) 
is added to each byte. For more details see section [UTF-8 byte map](#utf-8-byte-map) and illustrations in this
chapter.

The following table presents the current standard of UTF-8 encoding.

| Byte 1      | Byte 2      | Byte 3      | Byte 4      | Code point start | Code point end |
|-------------|-------------|-------------|-------------|------------------|----------------|
| `0bbb aaaa` |             |             |             | `U+0000`         | `U+007F`       |
| `110c ccbb` | `10bb aaaa` |             |             | `U+0080`         | `U+07FF`       |
| `1110 dddd` | `10cc ccbb` | `10bb aaaa` |             | `U+0800`         | `U+FFFF`       |
| `1111 0fee` | `10ee dddd` | `10cc ccbb` | `10bb aaaa` | `U+10000`        | `U+10FFFF`     |

Table legend: `0` and `1` in each Byte column are bit literals, while characters in range `a`—`f` are used to 
present bits of distinct hexadecimal digits required to encode the given code point. Therefore, each letter
`a` denote a single bit from the least significant hexadecimal digit of a code point. On the last row, just
a single letter `f` occurs, representing that the most significant hexadecimal digit is limited just to values
backed by a single bit, so that digit could be either 0 or 1 (as no more hexadecimal digits are representable
by just one bit).

During decoding of UTF-8 stream a quick check of a few bits can tell you some important information:
 - `0xxx xxxx` is simple ASCII character;
 - `10xx xxxx` is so-called **continuation byte**;
 - take number of prefix ones to interpret it as the total number of bytes encoding the current code point
   (e.g. `1110` has three ones, so three bytes are needed to decode the current code point, so two more bytes
   are needed to gather all the bits of the current code point). 


#### Original standard of UTF-8 encoding

The original UTF-8 standard was designed to handle up to 2³¹ code points. The following illustration describes
that original design allowing each UTF-8 code point to be encoded using 1—6 bytes:

```text
 | -------- |
 | 0xxxxxxx |                     →   7 + 0×6 = 7 + 0 = 7b
 | -------- |

 | -------- |      | -------- |
 | 110xxxxx | + 1× | 10xxxxxx |   →   5 + 1×6 = 5 + 6 = 11b
 | -------- |      | -------- |

 | -------- |      | -------- |
 | 1110xxxx | + 2× | 10xxxxxx |   →   4 + 2×6 = 4 + 12 = 16b
 | -------- |      | -------- |

 | -------- |      | -------- |
 | 11110xxx | + 3× | 10xxxxxx |   →   3 + 3×6 = 3 + 18 = 21b
 | -------- |      | -------- |

 | -------- |      | -------- |
 | 111110xx | + 4× | 10xxxxxx |   →   2 + 4×6 = 2 + 24 = 26b
 | -------- |      | -------- |

 | -------- |      | -------- |
 | 1111110x | + 5× | 10xxxxxx |   →   1 + 5×6 = 1 + 30 = 31b
 | -------- |      | -------- |
```


#### UTF-8 byte map

Each byte of UTF-8 stream might be encoded using the following byte map:
 - 0x00—0x7F (128 bytes): ASCII character (compatibility layer)
 - 0x80—0xBF (64 bytes, all with `10` prefix): continuation byte
 - 0xC0—0xDF (32 bytes): first byte of a 2-byte code point
     - _0xC0—x0C1 (2 bytes): invalid bytes_
 - 0xE0—0xEF (16 bytes): first byte of a 3-byte code point
 - 0xF0—0xF7 (8 bytes): first byte of a 4-byte code point 
     - _0xF5—0xF7 (3 bytes): invalid bytes (representing 4-byte code point greater than `U+10FFFF`)_
 - _0xF8—0xFB (4 bytes): first byte of a 5-byte code point (invalid bytes due to over-range)_
 - _0xFC—0xFD (2 bytes): first byte of a 6-byte code point (invalid bytes due to over-range)_
 - _0xFE—oxFE (2 bytes): invalid bytes (never to occur in UTF-8)_

Then there are bytes that are considered invalid in some context:
 - continuation byte (0x80—0xBF) as the first byte of grapheme
 - non-continuation byte (0x00—0x7F or 0xC0—0xFF) before end of grapheme
 - grapheme with the first byte 0xF4 followed by byte from range 0x90—0xFF
   (causing code point overflow over `U+10FFFF`)
 - extracted code points of surrogates (`U+D800`—`U+DFFF`) cannot occur in UTF-8
 - overlong encoding (e.g. ASCII character `A` represented using more than a single byte)


---

### UTF-7

The UTF-7 encoding is similar to UTF-8, except that it uses 7 bits instead of full byte range.
Emails that are not 8-bit clean might be encoded using UTF-7.
