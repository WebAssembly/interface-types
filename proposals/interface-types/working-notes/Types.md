# Interface value types

This proposal defines new interface values, which have corresponding interface value types. Interface value types are a superset of the [core value types](https://webassembly.github.io/spec/core/syntax/types.html#value-types)

Some interface value types are compound by being constructed using one or more interface value types. The abstract syntax is designed to prevent recursive types by not allowing a compound type definition to refer to itself or other definitions. This is so that the set of control flow needed to generate all interface values is simple enough to allow for optimized lazy semantics.

The interface value type syntax is summarized below. Following after that is a section for each type that gives the full syntax, possible values, validation rules, and subtyping relationships.

```
#interface-valtype ::=
  | #valtype
  | s8
  | s16
  | s32
  | s64
  | u8
  | u16
  | u32
  | u64
  | string
  | record ..
  | array ..
```

## `s8`, `s16`, `s32`, `s64`, `u8`, `u16`, `u32`, `u64`

Integer values.

| Type | Values            |
|------|-------------------|
| s8   | [-(2^7), 2^7-1]   |
| s16  | [-(2^15), 2^15-1] |
| s32  | [-(2^31), 2^31-1] |
| s64  | [-(2^63), 2^63-1] |
| u8   | [0, 2^8-1]        |
| u16  | [0, 2^16-1]       |
| u32  | [0, 2^32-1]       |
| u64  | [0, 2^64-1]       |

These types differ from the core value types in that they represent a range of integers instead of a set of bits.

## `string`

A string value is a sequence of [unicode code points](https://www.unicode.org/glossary/#code_point).

## `record`

The `record` interface value type is a compound type defined by the following abstract syntax:

```
#record ::= record $fields: (field #interface-valtype)*
```

A `record` value is an ordered set of `|$fields|` interface values, where each interface value is of the corresponding type in `$fields`. This is sometimes known as a struct or product type.

A `record` type is valid iff:
 * `|$fields| > 0`
 * For every `field` in `$fields`
  * `#interface-valtype` is valid

## `array`

The `array` interface value type is a compound type defined by the following abstract syntax:

```
#array ::= array #interface-valtype
```

An `array` value is a homogenous sequence of interface values with type `$interface-value-type`.

An `array` type is valid iff:
 * `#interface-valtype` is valid
