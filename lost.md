# LOST Programming Language

The LOST programming language focuses on very low level programming, more a portable assembler than anything else.

The language is meant to be easily compiled, and should support a lot of smaller platforms.

## Project Goals

Create a language and compiler that supports at least the following CPUs:
- [SPU Mark 2](https://ashet.computer/docs/isa.htm)
- [LIMN](https://github.com/limnarch)
- [6502](https://en.wikipedia.org/wiki/MOS_Technology_6502)
- [Z80](https://en.wikipedia.org/wiki/Zilog_Z80)
- [PDP-8](https://en.wikipedia.org/wiki/PDP-8)

The language should be as close to the metal as possible, not having much abstract types. Each operation should map directly to hardware instructions or be very easily abstractable (`x*2` can be done via shifts).

The compiler toolchain created with this project should be a cross-compiler running on modern desktop machines and should be easy to use via command line or text editors.

## WIP: Spec

### `.lost` Files

Source files have an unspecified encoding, but should prefer `UTF-8` encoding. This is deliberately unspecified as LOST is designed as a cross-compiled language, but there might be compiler implementations running on native platforms where it is favourable to write files in the native encoding instead of `UTF-8`.

It is allowed for any compiler to reject files based on their encoding and only accept a certain encoding.

### Compiler Diagnostics

A compiler *must* emit an error when a ill-formed program is detected and *might* emit a warning for certain issues it detects with the code.

### Builtin types

Each builtin type is an unsigned integer except otherwise stated. All integer types are always in two's complement for signed numbers and in machine endianess.

#### `byte`
The smallest adressable unit of the machine. Not necessarily, but very commonly, an 8 bit type.

#### `register`
The register size of the ALU. If a CPU has several ALU sizes, it's the maximum size.

#### `pointer`
The size of a regular *near* pointer. This is the same size as all dereferencable pointers and does not contain segment information.

#### `farpointer`
The full size of the address bus of the system. This is usually the same as `pointer` for non-segmented systems, but might differ for systems with segmentation like the 8086.

#### `integer`
An arbitrary-sized signed big integer that can only be used in constants or at compile time. This is used for constant calculations and can be used in macros.

### Pointer types

#### `*T`
A pointer to a single element of `T`. `T` might be a `layout` type, any runtime integer or another pointer (indireection).

#### `[]T`
A pointer to multiple elements of `T`. `T` might be a `layout` type, any runtime integer or another pointer (indireection).

### Fixed-size arrays

`[N]T` declares an array of `N` elements of type `T`. `T` might be any runtime integer size.

### `layout` types

Layouts are descriptions for sequential memory layouts:

```lost
layout Name
{
  field1: type,
  field2: type
}
```

Each field is layed out in memory one after another, without padding inbetween. If the hardware requires certain alignments to access the fields, it is necessary to insert padding fields.

Layouts can only be used to describe memory areas and thus cannot be used as a variable type, but it's possible to create a pointer to them.

### String literals

String literals will resolve into pointers to a sequence of bytes, but have their own type. There are several flavours of string literals:
```lost
const basic_string:    []byte =  "hello, world!";
const zero_terminated: []byte = z"c strings are fine";
const length_prefixed: []byte = p"pascal strings as well";
const length_prefixed: []byte = P"more pascal for the world";
const byte_sequence:   []byte = b"DeadBeef";
```

When using string literals, the compiler is allowed to deduplicate them and even return pointers to portions of other strings if applicable. This means that `"hello"` and `"hello world"` might have the same address.

#### Basic String 
This string has neither length prefix nor a zero terminator and contains the bytes in the string in *target encoding*.

#### Zero Terminated 
This string has a zero terminator byte.

#### Length Prefixed (short) 
This string is prefixed with a byte sized integer containing the string length.

#### Length Prefixed (long)
This string is prefixed with a register sized integer containing the string length.

#### Byte Sequence
This string only allows hex bytes as contents and will be translated into a raw sequence of bytes without encoding.
Each character pair in the string corresponds to a single byte:
```lost
const dead_sequence = b"DEADBEEF"; # sequence of 4 bytes 0xDE, 0xAD, 0xBE, 0xEF
```

#### Target Encoding
This is a property of the target system and defines, how strings are encoded. For example, CBM computers will use PETSCII whereas IBM machines might use EBCDIC or ASCII.

If a string contains untranslatable symbols, a replacement character fitting for the encoding should be emitted into the string and a *warning* should be generated.