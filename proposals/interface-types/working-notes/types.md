# Interface Types Type Schema

This note lays out the possible type terms that may be constructed with
Interface Types.

## Primitive Types

The so-called primitive types are built-in types that do not have an exposed
internal structure.

### Integer Types

There are 8 bit, 16 bit, 32 bit and 64 bit integer types, in both signed and
unsigned variants.

This is somewhat richer than WASM's core integer types; this is to allow API
designers to be more accurate in expressing their intentions.

| | | |
| --- | --- | --- |
| `s8` | -128..127 | Signed 8 bit integer |
| `u8` | 0..255 | Unsigned 8 bit integer |
| `s16` | -32768..32767 | Signed 16it integer |
| `u16` | 0..65535 | Unsigned 16it integer |
| `s32` | -2_147_483_648 .. 2_147_483_647 |Signed 32 bit integer |
| `u32` | 0..4_294_967_295 | Unsigned 32 bit integer |
| `s64` | −9_223_372_036_854_775_808 to +9_223_372_036_854_775_807 | Signed 64 bit integer |
| `u64` | 0..18_446_744_073_709_551_615 | Unsigned 64 bit integer |

Note that we use the `s` prefix to denote signed, and `u` to denote unsigned.

### Float Types

The schema uses the same floating point types as core wasm: `f32` and `f64`.

### String type

| | |
| -- | -- |
| `string` | Sequence of Unicode codepoints

Note that the `string` type does not commit to any particular representation of
string values. Nor is there any commitment to a particular encoding (e.g.,
UTF-8). Such choices are typically embodied in the various coercion instructions
used to manage string values.

## Program Types

There are two main forms of program type: a function type and a method type. The
method type is written identically to a function type but is enclosed within an
interface signature.

### Function Signature



### Interface Signature

## Aggregate Types

### Record Type

### Tuple Type

## Collection Types

### Array Type

### Sequence Type

## Algebraic Data Types

New types can be introduced in one of two ways: via an explicit definition of
the type in terms of an _algebraic type definition_ or via a _type import_.

Algebraic type definitions define types in terms of combinations of other types;
more specifically, a sum of products of types. Each arm of the sum is, in
effect, a _variant_ of the type. Each element of a product is a _field_ of that
variant.

### Discriminated Variants

### Product Type

## Type Imports

