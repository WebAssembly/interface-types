# Interface value types

The set of types and type constructors that are specific to Interface Types
allow API designers to express types of data and function values in a way that
is suitable for module-level descriptions. These types are considerably richer
than those of core WebAssembly, but would not typically be enough for a full
fledged programming language -- that is not the purpose.

The space of types forms a a superset of the
[core value types](https://webassembly.github.io/spec/core/syntax/types.html#value-types);
this is because adapter functions will often use a mixture of core WebAssembly
types and Interface types.

As a matter of policy, Interface types allow designers to express numeric types,
strings, compound types, function types, optional types, arrays and
sequences. Note that these types are intended to represent _immutable_ values;
with some limited exceptions planned for the future.

This document has a structure that mirrors the structure of the core WebAssembly
proposal: we discuss the abstract structure of Interface types, the validation
rules for type expressions and the binary format of type expressions.

## Structure

The Interface types can partitioned into different groups, reflecting the kind of entities involved.

>Note: unlike core WebAssembly types, there is not always a straightforward
>correspondence between Interface types and run-time entities. This is because
>Interface types are _abstract_ and require a _realization_ in terms of core
>WebAssembly entities.

>This realization is not directly part of this specification, although there are
>specific forms and instructions whose purpose is to aid in it.

```
InterfaceValType ::= BasicType
  | StructureType 
  | FunctionType
  | Protocol
  | MacroType
```


### Basic Value Types

```
BasicType ::= IntegralType | StringType
```

#### Integral Types

There are eight integral types, corresponding to signed and unsigned variants of
8 bit, 16 bit, 32 bit and 64 bit integers.

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


```
IntegralType ::= s8 | u8 | s16 | u16 | s32 | u32 | s64 | u64
```

Integral types denote an integer value in a given range. There is no implied
storage layout.

#### String Type

The `string` type denotes a sequence of
[unicode code points](https://www.unicode.org/glossary/#code_point).


```
StringType ::= string
```

>Note: In a future revision, this may be reinterpreted as a macro of a more
>explicit designation of the `sequence` type.

Note that there is no implied representation or encoding implied in this.


### Structured Value Types

Structured types allow the representation of more complex combinations of
values. There are five forms of structured type: arrays, sequences, records,
Protocols and algebraic variants.

```
StructureType ::= ArrayType | SequenceType | RecordType | VariantType
```

#### Arrays

An `array` consists of a finite vector of elements, each of which has the same
type.

```
ArrayType ::= array InterfaceValType
```

Both the type and the representation of each element of the array is assumed to
be identical. Furthermore, arrays are assumed to have a know element
count. (Indexing into arrays is not part of this specification.)

#### Sequences

A sequence consists of an ordered collection of elements; each of which has the
same type.

Sequence elements may not have identical representation; nor is it necessarily
known how many elements there are in a sequence.

```
SequenceType ::= sequence InterfaceValType
```

#### Records

A `record` type is a tuple composition of types. 

```
RecordType ::= record [ vec(InterfaceValType) ]
```

Fields in a record are accessed via their index; for convenience, field indices
may be given a local name:

```
record [ $name string $age u64]
```

#### Variant Types

A `variant` consists of an ordered collection of alternative vectors of types.

```
VariantType ::= oneof [ vec([identifier] vec(InterfaceValType)) ]
```

Variants are accessed by their index in the vector of alternatives. For
convenience, variants may be given a local name:

```
variant [ $id [s64] $name [string] ]
```

A given variant may not have any type arguments; in which case it corresponds to
an enumerated symbol. For example, the boolean type may be viewed as a synonym
for:

```
(type $boolean (variant ($false) ($true)))
```

Additionally, the option type may be viewed as a synonym for:

```
(type ($option #tp) (variant ($nil) ($some #tp)))
```

>Note: this definition is not technically legal; because there is no current
>support for type parameters in WebAssembly.

The representation of a variant is determined by the representation of each of
its alternatives.

### Function Type

A `FunctionType` denotes a function whose signature is expressed as Interface
Types.

```
FunctionType ::= [vec(InterfaceValType)] -> [vec(InterfaceValType)]
```

The definition of a `FunctionType` mirrors that of the core WebAssembly form;
except that the arguments and returns may be interface types.

### Protocols

A Protocol consists of a collection of function signatures. It is intended to
denote a related set of functionality that a given entity may offer.

```
Protocol ::= protocol [vec(identifier FunctionType)]
```

The different _methods_ in a protocol are indexed by integer offset; however,
for convenience, names may be given to individual methods.

### Macro Types

The macro types are convenience expressions for types that could be otherwise
expressed in more primitive terms. However, beyond convenience, they also denote
common scenarios that the Interface Types proposal has explicit support for.

```
MacroType ::= OptionType | EitherType
```

#### Option Type

Option types are used to model nullability in types.

```
OptionType ::= option InterfaceValType
```

The `option` type can be viewed as a use of type variants -- with one variant
being the enumerated symbol `$none` and the other being the single value wrapped
in a `$some` variant.

#### Either Type

The `either` type is used to model situations where one of two values may be
returned by a function. A classic case for this is in modeling error returns in
APIs.

```
eithertype ::= either InterfaceValType InterfaceValType
```


