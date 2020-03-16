# computing/processors/m68k/calling-conventions

There is no single universal standard calling convention for the Motorola 68000.

## Native types

The Motorola 68000 family has the following native types:
- `byte`: a 1-byte value
- `word`: a 2-byte value
- `long`: a 4-byte value

The 68881/68882 FPUs and the 68040's built-in FPU also have the following types:
- `single`: IEEE binary32
- `double`: IEEE binary64
- `extended`: 96-bit extended precision; 15-bit exponent, 16 unused bits, 64-bit mantissa
- `packed`: a packed decimal real format consisting of a 3-digit base-10 exponent and a 17-digit base 10 mantissa.

The ColdFire v4e FPU does not support the `extended` or `packed` formats

## System V

"System V Application Binary Interface Motorola 68000 Supplement" ([ISBN 0-13-877663-6](https://www.worldcat.org/title/system-v-application-binary-interface-motorola-68000-processor-family-supplement/oclc/22002584))
documents the ABI that is used by System V platforms.
However, this book has long been out of print and appears to be nearly impossible
to find. Fortunately, [a partial (large) PDF scan](https://github.com/M680x0/Literature/blob/master/sysv-m68k-abi.pdf)
exists of it on GitHub.

Note that this document is used to describe the ABI on 68020 or later,
but `gcc` uses this ABI when you choose `-march=68000` or `68010`.

Note too that this document predates every m68k-ish CPU made after the
68040.

### Types

| Type              | `sizeof` | alignment | native             |
|-------------------|----------|-----------|--------------------|
| `signed char`     | 1        | 1         | signed byte        |
| `char`            | 1        | 1         | signed byte        |
| `unsigned char`   | 1        | 1         | unsigned byte      |
| `short`           | 2        | 2         | signed word        |
| `signed short`    | 2        | 2         | signed word        |
| `unsigned short`  | 2        | 2         | unsigned word      |
| `int`             | 4        | 4         | signed long        |
| `signed int`      | 4        | 4         | signed long        |
| `long`            | 4        | 4         | signed long        |
| `signed long`     | 4        | 4         | signed long        |
| `enum`            | 4        | 4         | signed long        |
| `unsigned int`    | 4        | 4         | unsigned long      |
| `unsigned long`   | 4        | 4         | unsigned long      |
| `T *`             | 4        | 4         | unsigned long      |
| `T (*)()`         | 4        | 4         | unsigned long      |
| `float`           | 4        | 4         | single-precision   |
| `double`          | 8        | 4 on stack, 8 if in aggregate  | double-precision   |
| `long double`     | 16       | 4 on stack, 8 if in aggregate  | extended-precision |

### Function Calls

- `D0`, `D1`, `A0`, and `A1` are scratch registers.
- `D2`..`D7` and `A2`..`A5` are locals, and must be saved by the callee if used.
- `A6` is the frame pointer, if implemented. (May be aliased to `FP`)
- `A7` is the stack pointer. (Commonly aliased to `SP`)
- If a FPU exists, then `FP0` and `FP1` are scratch registers, and `FP2`..`FP7` are locals.

The stack frame is 4-byte aligned.

All arguments go on the stack.
- Integral arguments are expanded to long-word size and pushed onto the stack.
- Pointers are already long-word size.
- Floating-point arguments occupy 4, 8, or 16 bytes on the stack (`float`, `double`, or `long double` respectively)

Return values depend on the type:
- Integral return values are returned in `D0`
- Pointer return values are returned in `A0`
- Floating point values are returned in `FP0`
- When calling a function that returns a structure or union, the caller allocates space and sets `A0` to that address, and the function returns that address in `A0`.
- Functions that return `void` are not prescribed to put any value in any register.

### gcc-specific notes for `m68k-linux-gnu`

- [gcc will return pointer values in both `A0` and `D0`.](https://github.com/gcc-mirror/gcc/blob/4212a6a3e44f870412d9025eeb323fd4f50a61da/gcc/config/m68k/m68k.c#L5350)
- 64-bit types (`long long int`) became commonplace after the 1990 publication of the _System V 68000 Supplement_. gcc returns 64-bit quantities in `D0` (MSBs) and `D1` (LSBs). I can't find documentation for this.
- gcc does not support `__int128` on this target.
- ColdFire targets that have a FPU (e.g., `-mcpu=548x`) do not support extended-precision floating point; `long double` is a synonym for `double` on these targets. [This was a change in GCC 4.3](https://www.gnu.org/software/gcc/gcc-4.3/changes.html).

## Metrowerks CodeWarrior

Metrowerks was a company that made a compiler called CodeWarrior; Metrowerks was later
absorbed into Motorola, then to Freescale, then to NXP, so "CodeWarrior" may be seen
with those names. It is still possible to get some versions of CodeWarrior targeting
Embedded 68K and ColdFire processors from NXP. The information in this section is based
on looking at:
- CodeWarrior for E68K 3.2 (`mwcce68k` Version 3.0.5 build 55)
- CodeWarrior for Microcontrollers 6.3 (`mwccmcf` Version 6.0 build 38)
- CodeWarrior for ColdFire V7.1 (`mwccmcf` Version 5.2 build 26)
- CodeWarrior MCU v11.1 (`mwccmcf` Version 6.0 build 3026)

### Types

Broadly speaking, the same as the System V convention.

Unlike the System V convention, which says that `enum`s should be a `signed long`,
CodeWarrior adapts the size of an `enum` based on its range of values, unless the
`enumsalwaysint` pragma is enabled.

### Function Calls

Hoo boy.

There are three different calling conventions:
- `standard_abi`: arguments passed on the stack (similar to the SysV convention)
- `compact_abi`: arguments passed on the stack, but with promotion to two-byte quantities instead of four-byte.
- `register_abi`: first arguments passed in registers with the rest on the stack

Register usage is the same for all three ABIs:
- `D0`..`D2`, `A0`, and `A1` are scratch registers.
- `D3`..`D7` and `A2`..`A5` are locals, and must be saved by the callee if used.
- `A6` is the frame pointer, if implemented. (May be aliased to `FP`)
- `A7` is the stack pointer. (Commonly aliased to `SP`)
- If a FPU exists, then `FP0`..`FP2` are scratch registers, and `FP3`..`FP7` are locals.

Return values are the same for all three ABIs, and are identical to the System V convention.
- Integral return values are returned in `D0`
- Pointer return values are returned in `A0`
- Floating point values are returned in `FP0`
- When calling a function that returns a structure or union, the caller allocates space and sets `A0` to that address, and the function returns that address in `A0`.
- Functions that return `void` are not prescribed to put any value in any register.

#### `standard_abi`

The stack frame is 4-byte aligned.

All arguments go on the stack.
- Integral arguments are expanded to long-word size and pushed onto the stack.
- Pointers are already long-word size.
- Floating-point arguments occupy 4, 8, or 16 bytes on the stack (`float`, `double`, or `long double` respectively)

This convention is almost the same as the System V convention, except that `D2` and `FP2` are scratch registers and not locals.

#### `compact_abi`

This is the same as `standard_abi`, except that small integral arguments are expanded to word size (e.g. `char` and `short`
are both passed as two-byte values).

#### `register_abi`

Initial arguments are in registers:
- The first two floating-point arguments are in `FP0` and `FP1`.
- The first two pointer arguments are in `A0` and `A1`.
- The first three integral arguments are in `D0`, `D1`, and `D2`.
- All other arguments (including structures and unions) are on the stack the same way as `standard_abi`.

