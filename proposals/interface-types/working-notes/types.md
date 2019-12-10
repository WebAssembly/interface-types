# Interface Types Type Schema

This note lays out the possible type terms that may be constructed with
Interface Types.

### Writing Types

There are two written forms of type terms: an abstract notation and the
webAssembly text format. In addition, there is a binary encoding of type
expressions which denotes the binary format of type terms -- especially within a
webAssembly module.

>Note: the binary representation is the only normative representation of types.

>Note: the rationale for two written forms is that there at least two separate
>kinds of written texts: the WAT format of a webAssembly source and
>independently published specifications of APIs. These have different
>requirements for writing type expressions.

### Denotations vs representations

Generally, the type schema for Interface Types does _not_ indicate any
information as to how values are _represented_. The type schema does establish
constraints about the values denoted -- such as the numerical range of an
integer.

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
method type is written identically to a function type but is enclosed within a
service signature.

### Function Signature

A function signature consists of a sequence of argument types and a sequence of
result types.

The abstract written form of a function signature is:

```
Type .. Type => Type .. Type
```

In the abstract notation parentheses may be used to avoid ambiguity:

```
(Type .. Type) => (Type .. Type)
```

Within a webAssembly module, such as within the signature for an adapter
function, function types are written using the normal wasm style of function
signature:

```
(param type) .. (param type) (result type) .. (result type)
```

### Service Signature

A _service signature_ consists of a set of named function type
signatures. It's purpose is to enable the modeling of objects; specifically it
enables the modeling of access to the functionality of an object.

Although conceptually similar to record types, a service signature has some
crucial modeling differences:

 * A _service signature_ does not itself convey any knowledge about the
   underlying value -- specifically it is opaque. (A service signature tells
   you what you can do with a value, not what it is made of.)
 * A _service signature_ may only consist of named function signatures.
 * A function signature that is within a service signature is not
   _seperable_ from the interface: it is undefined to access a function within
   an interface signature other than by calling it.
 * The Interface Types proposal does not contain any method for _creating
   entities_ with an service signature.
 * Instance field access may modeled in terms of getter and setter functions.
 
Service signatures are written in prefix notation using the `service` keyword:

```
(service
  (method "name" Type) .. (method "name" Type))
```

In the abstract notation, service types are written using a sequence of named
fields enclosed in braces:

```
{ name : Type; .. name : Type }
```

## Aggregate Types

### Record Type

A record is a set of named fields; the record type reflects that: it is a
mapping from names to types.

In abstract notation, record types are written using named fields within square
brackets:

```
[ name : Type; .. name : Type ]
```

In the WAT prefix format, record types are written using the `record` prefix
operator:

```
(record (field "name" Type) .. (field "name" Type))
```


### Tuple Type

A tuple is a cross product of types. It is analogous to a record type where all
the fields have integer names starting with zero. Tuple types are written in the abstract notation as a sequence of types enclosed by braces:

```
( Type , .. , Type )
```

>Note: there is potential for syntactic ambiguity when combining tuple types
>with function types. This may be resolved by using an extra set of parentheses:

```
((string,s32))=>string
```

denotes a function signature that takes one argument: a 2-tuple consisting of a
pair of a `string` and a signed 32 bit integer.

In the WAT prefix format, tuple types are written using the `tuple` prefix
operator:

```
(tuple Type .. Type )
```

Tuple and record types are a convenient technique for denoting composite values;
such as color coordinates and credit card information.

## Collection Types

There are two so-called collection types in the interface types schema: the
array type and the sequence type.

### Array Type

The `array` type denotes a fixed size sequence of elements -- each of which is
of the same type.

The `array` type is a _type constructor_ rather than a type: applying the
`array` type constructor to a type gives the type of the array as a whole. This
is signaled in the abstract notation with angle brackets:

```
array<(s64, string)>
```
denotes an array of tuples, each of which consists of a signed 64 integer and a string.

In the WAT prefix form, we use the `array` prefix operator:

```
(array (tuple s64 string))
```

Note that this is actually syntactic sugar for the more general form:

```
(generic $array (tuple s64 string))
```


### Sequence Type

The `sequence` type constructor is used to denote sequences of elements where
the length is not generally available; and the elements are not indexable.

>Note: a string is better modeled as a sequence of code points rather than an
>array of code points.

In the abstract notation, sequence types are written using the `sequence`
operator:

```
sequence<string>
```
and in the prefix notation, we use the `sequence` prefix operator:

```
(sequence string)
```

## Defining types

New types can be introduced in one of two ways: via an explicit definition of
the type in terms of an _algebraic type definition_ or via a _type import_.

### Algebraic Data Types

Algebraic type definitions define types in terms of combinations of other types;
more specifically, a sum of products of types. Each arm of the sum is, in
effect, a _variant_ of the type. Each element of a product is a _field_ of that
variant.

An algebraic type may either be _quantified_ (i.e., parameterized by one or more type variables) or _monomorphic_.

Algebraic data types are defined in WAT prefix style using a `datatype`
statement:

```
(@interface datatype LocalName 
  (oneof (variant "name" Type..Type) .. (variant "name" Type..Type)))
```

The shorthand form:

```
(@interface datatype LocalName 
  Type)
```

may be used when the type is effectively an alias for a type expression and
there are no variants.

In abstract notation, data types are introduced using a statement of the form:

```
Type ::= Lbl Type .. Type | .. | Lbl Type .. Type
```

where `Lbl` is the name of the variant.

>Note: each arm of an algebraic data type is effectively about a tuple of types.

#### Enumerated Types

Enumerated types are variants (sic) of the regular datatype definition:

```
(@interface datatype $weekday 
  (oneof (variant "monday") .. (variant "sunday")))
```

### Type Imports


