# Web Interoperability Interface Definition Language

This specification defines an abstract schema to support a limited form of
interoperability between WebAssembly modules and between WebAssembly modules and
a host environment.

## Introduction & Motivating Principles

This design is intended to support interoperating between systems that do not
necessarily share a common memory or common implementation language. As such the
types and other elements are not likely to be native to any given language
system. Furthermore, we use as building blocks concepts such as integer, float,
function and interface that we expect to be universally applicable in gross form
-- if not in detailed form. I.e., it is not our intention to facilitate all
possible interactions between systems; only those that are most likely to be
‘smooth’.

### Ownership Boundaries

One of the purposes of an IDL of any description is to facilitate access to code
that is ‘foreign’ in some sense. This can be characterized in terms of ownership
boundaries: it is ‘my’ system trying to use ‘your’ system in order to achieve
congruent goals.

If an integration task does not appear to involve such ownership boundaries then
it is likely that richer, higher bandwidth forms of interaction are also
possible that go beyond the scope of this design.

As an example of what this implies, the IDL supports sequences of values: they
can be passed as arguments of functions. However, the IDL does not directly
support *iteration* over sequences, nor are there any specific means by which
sequences can be created. This disitnction between allowing sequences to
transmitted but not necessarily processed is a typical manifestation of the
ownership boundary.

### Resources

Why do we use other people’s code? It is not normally to perform a calculation
that we could do ourselves. A more common scenario is to access a resource that
the other party controls. In the case of accessing web APIs the resources are
often browser related resources (the DOM, web GL etc.) that the client simply
cannot access directly.

This manifests as a requirement to be able to share handles to internal
resources that other parties may reference in operations without being able to
'see inside'.

The wi-IDL models resources as a standard generic type -- `Resource<T>` -- which
is used to denote a handle to a shared resource of type `T`.

### Language Neutrality

This specification does not attempt to solve the ‘language interoperability’
problem in its full generality. There is evidence that suggests this problem is
not actually solvable. However, for the more limited goal of accessing
functionality -- potentially across language boundaries as well as ownership
boundaries -- it is possible to be language agnostic to a large degree:
primarily by strongly limiting the expressive power of the schema.

Thus we do not favor any particular style of language - functional vs object
oriented vs procedural; nor do we favor particular memory strategies - managed
memory vs manually managed linear memory. On the other hand, this also means
that we do not support many common usage patterns - such as shared memory access
to data structures.

## Interoperability Strategy

Interoperability between providers and clients is supported by a two part
strategy: an abstract schema language (wi-IDL) whose primary purpose is to
describe potential interfaces that modules can export and import; together with
a declarative coercion notation that allows individual modules to denote how the
module realizes and/or consumes the schema for individual functions.

> This document focuses on the schema language; a companion document focuses on
> the binding expressions themselves.

The primary components of the schema are interface and data type
specifications. In summary, an interface describes what operations are available
and a data type specification specifies the data that can be communicated in
those operations.

The two concepts are mutually recursive (sic): an interface refers to data and a
data value may refer to entities that have interfaces around them.

### Nominal vs Structural Types

A nominal type is one whose semantics depends on a denotation: on the entities
being denoted by the type. For example, a Person type is typically understood as
meaning or denoting people -- not by any internal structure associated with
Person values.

On the other hand, a structural type is one whose semantics arises from
composition: from the components of its internal representation. For example, a
tuple pair type denotes pairs of values; it is not expected that tuple pairs
would be used to model ‘real world’ entities.

Structural types are important for what one can do with them, nominal types are
important for what they mean.

In this schema, nominal types are modeled in terms of type quantification: a
nominal type is one that is introduced via a quantifier in a quantified
type. Operationally, the difference between a nominal type and a structural type
is that the former is fundamentally opaque whereas a structural type is
transparent: you can see the pieces of a structural type.

### Algebraic Types

An algebraic type is one whose values are defined by composition of two
fundamental operators: tupling and union. Algebraic types are important as a way
of expressing alternate forms of data; two examples being enumerations (such as
boolean) and alternate representations of a point (in terms of cartesian or
polar coordinates).

### Type Quantification

A quantified type is a way of denoting an arbitrary number of types -- in the
case of universal quantifiers an infinite number and in the case of existential
quantifiers at least one. In addition to this enumeration, quantified types also
allow us to clarify scope; in particular nested scope.

In addition to quantifying types, it is also possible for a type to be
generic. A generic type is like a function on types: it takes a (typically one)
type as argument and returns a new type.

Generic types have two roles in this schema: they underlie the built-in concepts
of Lists and Options; and they can be used to model certain ‘user-defined’ types
-- such as a Permission type which might need to have the type of permission
‘plugged in’.

### Type Equality

Compatibility between types is defined to be based on equality. In particular,
subtyping is not supported by this schema.

The reason for this is that subtypes introduce significant complexity for
validation; furthermore, when combined with binding expressions whose task it is
to raise or lower values to meet the type signature of an operation, subtyping
is unnecessary.

### Collections

There is a single standard collection type in this schema: the sequence. This
allows complex data -- such as a sequence of points each of which is represented
as a pair of float values -- to be communicated as part of an operation.

However, the schema itself does not include any way of manipulating sequences:
for example, there is no notation for indexing into a sequence; or even knowing
how long the sequence is.

The effect is that compound data values may be exchanged as part of an operation
but the values themselves may not be edited in any way.

### Representing Values

In addition to representing interfaces and types, it is also useful and
necessary to be able to represent the values that are communicated across
interfaces.

In effect, a standard set of values that mirrors the types of interfaces
represents an abstract definition of a _value encoding scheme_. Not all
applications require such schemes; and the details of the encoding may vary.

## Conventions

Elements of the wi-IDL can be denoted abstractly -- in terms of concepts and
combinations of concepts -- and concretely -- in terms of written forms. This
specification details two written forms, in addition to the abstract form: a
written form using S-Expression style notation and a binary representation
suitable for embedding in machine-readable artifacts.

### Grammar Notation

Grammar production rules are written in the form:

```
NT(Arg) => Body
```
where *Body* is a sequence of terminals and non-terminals.

Terminal symbols are denoted by quoted strings:

```
'{'
```

A Non Terminal is represented as a name followed optionally by an argument in
parentheses.

The special notation:

```
TNT .. TNT
```

denotes an arbitrary sequence (including 0) of occurrences of `TNT` where `TNT`
may be either a terminal or a non-terminal symbol.

The variant form:

```
TNT Op .. Op TNT
```

denotes an arbitrary sequence of occurrences of `TNT`, separated by `Op`. For
example, the rule:

```
AlgebraicSpec => Name Type '|' .. '|' Name Type
```
matches examples such as:

```
Foo Integer
```

```
Foo Integer | Bar String
```
and

```
Foo Integer | Bar String | Jar Float
```

as well as the empty sequence.

Multiple productions may be applicable to a given non-terminal; these are
represented as multiple rules.

Non terminals may have argument expressions, the constraint on the production is
that all occurrences of a given argument variable must have the same
value. Furthermore, a grammar production rule may have additional semantic
constraints (sometimes known as side-conditions); these are represented by a
predicate enclosed in braces.

For example, the rules

```
FunctionType => TupleType '=>' Type
FunctionType => TupleType '=>' Type 'throws' Type
```

defines the two ways in which a _FunctionType_ can be constructed: with a tuple
of argument types, a result type and an optional exception type.

Similar grammar notations are used to denote the abstract structuring of
concepts and the concrete surface notation; we distinguish between the two by
using a `=>` to denote an abstract production and `::=` to denote a production
in the concrete syntax. In addition, we use the convention that non-terminals of the concrete grammar are prefixed by `W-`, as in:

```
W-Tuple ::= '(' W-Term .. W-Term ')'
```


#### S-Expression Notation

This specification uses a variant of so-called S-Expression notation to
represent human readable expressions. This variant -- called W-Expressions --
has four different bracketing operators -- `()`, `[]`, `{}` and `<>`. These
bracketing operators are distinct from one another but are otherwise equivalent.

Similarly, the 'splat' operator -- written as a prefix `^` -- whose role is to unpack a W-Expression into a sequence of elements. For example, 

```
(1 2 3 4) == (1 ^(2 3) 4)
```

even though,

```
(1 2 3 4) != (1 (2 3) 4)
```

>Note: The reason for having four different bracketing operators is to support
>readability without requiring complex parsing. This may be adjusted in the
>future.

## wi-IDL Schema

The wi-IDL schema language is a language for describing types.

A type is a term that denotes a collection of values. Specifically, there exists
a unique meta-function mapping values to types (all values have a unique type).

The wi-IDL type schema defines the values that a wi-IDL signature can reference.

There are several different kinds of type:

```
Type => FunctionType
Type => QuantifiedType
Type => TypeInterface
Type => TupleType
Type => NominalType
```

### Function Type

A function type denotes the types of functions.

```
FunctionType => TupleType '=>' Type
```

In the concrete W-Expression form, function types are written:

```
W-FunctionType ::= '(' 'func' W-TupleType W-Type ')'
```

For example, the type of a function that returns a string from a pair of
integers has type -- as an W-Expression term:

```
(func (integer integer) string)
```

A function that returns more than one value is indicated by its return type
being a _TupleType_.

>Discussion Point: webAssembly is described in terms of a stack machine. In that
>context, all functions simply return a tuple of values. However, in order to
>support interoperability with systems that are not inherently stack machines,
>this specification assumes all functions return a single value -- which may be
>a tuple. In particular, so-called void functions are assumed to return the
>empty tuple.

A function that may raise an exception can declare it with a `throws` clause in
its type signature:

```
FunctionType => TupleType '=>' Type 'throws' Type
```
or in W-Expression form:

```
W-FunctionType ::= '(' 'func' W-TupleType W-Type W-Type ')'
```

The type of an exception is not distinguished from other types. I.e., there is
no privileged type that is used for exceptions.

### Quantified Type {#quantified-type}

There are two forms of quantifier: universal and existential. 

```
UniversalType => 'all' TypeVar .. TypeVar '.' Type
```

and

```
ExistentialType => 'exist' TypeVar .. TypeVar '.' Type
```

Or, as W-Expressions:

```
W-UniversalType ::= '(' 'all' '(' W-TypeVar .. W-TypeVar ')' W-Type ')'
W-ExistentialType ::= '(' 'exist' '(' W-TypeVar .. W-TypeVar ')' W-Type ')'
```

Where more than one quantifier is applied to a type, they are left-concatenated; for example:

```
all X Y . (X Y)
```

is equivalent to

```
all X . all Y . (X Y)
```

There are two forms of _TypeVar_, corresponding to whether the type variable
denotes a type or a type function:

```
TypeVar => Name
TypeVar => Name '/' Integer
```

```
W-TypeVar ::= W-Identifier
W-TypeVar ::= '(' W-Identifier '/' W-Integer ')'
```

The second form is used to denote a generic type; for example `Sequence/1`
denotes the sequence type where the `/1` denotes the fact that `Sequence` takes
a single type argument.

>Question: Should type hiding be supported? I.e., if a quantified form introduces
>a type variable that is already in scope does it have the effect of occluding
>the outer name. Convenience and composability says yes, security says no.

#### Interface {#interface}

An interface type is a collection of type signatures, as an unordered set:

```
TypeInterface => '{' Signature ..  Signature '}'
```

```
W-TypeInterface ::= '{' W-Signature .. W-Signature '}'
```

There are two forms of _Signature_; one denotes the type of a field: i.e., it’s an
association of a type to a name:

```
Signature => Name ':' Type
```

```
W-Signature ::= '(' W-Identifier W-Type ')'
```

The second form of _Signature_ denotes a nominal type:

```
Signature => 'type' NominalType
```

```
W-Signature ::= '(' 'type' W-NominalType ')'
```

This form is used to identify so-called type imports. 

Signatures are also used to identify nominal types; in which case it may be
associated with an algebraic type combination:

```
Signature => 'type' NominalType '=' AlgebraicSpec
Signature => 'all' TypeVar .. TypeVar '.' NominalType '=' AlgebraicSpec
```

```
W-Signature ::= '(' 'type' W-NominalType W-AlgebraicSpec ')'
W-Signature ::= '(' 'type' '(' 'all' '(' W-TypeVar .. W-TypeVar ')' W-NominalType W-AlgebraicSpec ')' ')'
```

For example, the standard `Option/1` type can be defined as the _Signature_:

```
(type (all t <Option t> [none (some t)]))
```

#### Algebraic Specification

An _AlgebraicSpec_ is a specification of the possible values of a type. This
most closely mirrors the concept of _discriminated union_. An _AlgebraicSpec_
consists of a collection of alternate forms of a type; each of which is labeled
by a discriminator.

```
AlgebraicSpec => Name Type '|' .. '|' Name Type
```

If the _Type_ is the empty tuple then it may be omitted -- this corresponds to
the concept of enumeration types.

```
W-AlgebraicSpec ::= '[' W-Variant .. W-Variant ']'
W-Variant ::= W-Identifier | '(' W-Identifier W-Type ')'
```

<div class="example">
The legal values for standard Boolean type may be defined as an enumeration:

```
True | False
```
which can be elaborated into a full _Signature_ for the `Boolean` type:

```
type Boolean = True | False
```

</div>

>If the _Type_ associated with one of the arms of an _AlgebraicSpec_ is a
>_TypeInterface_ then the variant counts as the description of a record type; if
>the _Type_ is a _TupleType_ then the variant counts as a form of labeled tuple.

#### Tuple Type {#tuple-type}

A TupleType is a linear sequence of types. It denotes the cross product of its
component types:

```
TupleType => '(' Type ..  Type ')'
```

```
W-TupleType ::= '(' W-Type .. W-Type ')'
```

There are two special cases of _TupleType_ that warrant further discussion: the
unary tuple and the zero-ary tuple.

A unary tuple type, such as

```
(Integer)
```

is _not_ equivalent to its element type.

The zero tuple type:

```
()
```
is used to designate void.

>NOTE: Due to the ubiquity of tuples, the W-Expression form of a tuple type has
>a special status: it is not marked with a special keyword as the first element
>of its written form.

#### Nominal Type {#nominal-type}

A nominal type expression is used to identify types by name:

```
NominalType => Name
NominalType => Name '<' Type ..  Type '>'
```

```
W-NominalType ::= W-Identifier
W-NominalType ::= '<' W-Identifier W-Type .. W-Type '>'
```

Any _Name_ appearing in a _NominalType_ must either be a standard name or it must
have appeared in scope; either as a _Signature_ or as a bound _TypeVar_.

## Standard Types {#standard-types}

Standard types are nominal types whose scope encompasses all type expressions;
i.e., it is as though any given type expression were wrapped in an all
quantification that mentions all the standard types.

### Numeric Types{#numeric-types}

This includes `Integer` and `Float`.

Note that we specifically do not elaborate the numeric types -- for example into
signed vs unsigned, or 8bit vs 64bit. Such elaboration can be accounted for by
appropriate operators.

### Boolean Type {#boolean-type}

The `Boolean` type is standard, but is defined as though by an TypeDefinition
signature:

<div class="example">
```
(type Boolean [true false])
```
</div>

### String Type {#string-type}

The `String` type denotes immutable sequences of characters. Note that, like the
numeric types, this is an abstract type and does not details relating to
encoding. This is left to the appropriate lifting and lowering operators.

### Sequence/1 Type {#sequence-type}

The `Sequence/1` type is used to denote sequences of values. Its type parameter
refers to the types of elements in the sequence.

### Reference/1 Type {#reference-type}

The `Reference/1` type is used to denote values that are ‘passed by reference’
(aka resources). The type argument of the Reference is the type of the
underlying resource -- often expressed as an Interface where that makes sense.

### Option/1 Type {#option-type}

The `Option/1` type is used to denote optional or nullable values. Although a standard type, it can be defined using a TypeDefinition signature:

<div class="example">
```
(type (all (t) <Option t> [none (some t)]))
```
</div>

## Value Representations {#value-representation}

In this section we define the forms of values that may be represented and
communicated via APIs defined using the wi-IDL.

All values are associated with a single type that is inferrable by inspection of
the value itself.

```
Term => Tuple
Term => Record
Term => Variant
Term => String
Term => Number
```


### Tuples

A _tuple_ is a combination of values, enclosed in parentheses. 

```
Tuple => '(' Term .. Term ')'
```

or, in concrete W-Expression form:

```
W-Tuple ::= '(' W-Term .. W-Term ')'
```
where a _Term_ is either a value or a 'splatted' value:

```
Term => Value
Term => '^' Value
```

```
W-Term ::= W-Value
W-Term ::= '^' W-Value
```

The interpretation of the `^` operator is that the component elements of its
argument are 'opened out' into the tuple. For example,

```
( "alpha" ^ ("beta" "gamma") "delta")
```
is equivalent to

```
( "alpha" "beta" "gamma" "delta")

```

>Note: Of course, this is most useful in cases where the argument to the `^` is
>not a literal tuple but another expression whose value is a tuple.

The type of a tuple is a _TupleType_, whose type components correspond to the
types of the components of the tuple itself.

### Records

A _record_ is a combination of values, enclosed in braces, where each value is
labeled with an identifier.

```
Record => '{' Field .. Field '}'
Field => Name ':' Term
```
or, as a concrete form in W-Expressions:

```
W-Record ::= '{' W-Field .. W-Field '}'
W-Field ::= '(' W-Identifier W-Value ')'
```

The type of a record is a _TypeInterface_; where the named elements of the record correspond to named elements of the _TypeInterface_.

>Note: a future version of this specification may include the ability to include
>type signatures to a record; signifying locally introduced types.

### Variants

Where a value is one of a set of variants -- as defined by a _AlgebraicSpec_ --
the value is written as the appropriate label followed by its argument type. If
the argument type is the empty tuple then it may be omitted.

```
VariantTerm => Name Tuple
VariantTerm => Name
```

or, in concrete W-Expression form:

```
W-VariantTerm ::= '(' W-Identifier W-Term .. W-Term ')'
W-VariantTerm ::= W-Identifier
```

### Strings

A `String` value is written as a sequence of character references expressed as
UTF-8 codes; enclosed in double quotes:

```
String => '"' CharRef .. CharRef '"'
CharRef => UTF8-Char
```

```
W-String ::= '"' W-CharRef .. W-CharRef '"'
W-CharRef ::= UTF8-Char
```

### Numbers

There are two forms of number supported by this specification: integer values
and floating point values.

```
NumericLiteral => Integer
NumericLiteral => Float
```

```
W-Number ::= Integer
W-Number ::= Float
```

Integers are a non-empty sequence of decimal digits optionally preceded by a
minus character:

```
Integer => Digit .. Digit
Integer => '-' Digit .. Digit
```

```
Float => Sign Mantissae Exponent

Sign => '-'
Sign =>

Mantissae => Digit .. Digit '.' Digit .. Digit
Exponent => 'e' Integer
```

>Note: may need to extend this to permit infinities and explicit NaN values.

## Binary Encoding {#bindary-encoding}

This section describes the binary encoding of wi-IDL.

### Notation

The binary encoding is presented in the form of productions of the form:

```
NT(Arg) => Body
```

where *Body* is a sequence of terminals and non-terminals.

The terminals are either single characters (ASCII) or byte codes -- represented
as hex sequences.

A Non Terminal is represented as a name enclosed in angle brackets followed
optionally by an argument in parentheses and/or a repeat count -- a number or
argument enclosed in braces.

Multiple productions may be applicable to a given non-terminal; these are
represented as multiple rules.

Non terminals may have argument expressions, the constraint on the production is
that all occurrences of a given argument variable must have the same value.

<div class="example">
For example, in the production:

```
FunctionType => 0xft TupleType Type
```
the term `Type` refers to the non terminal `Type`.
</div>

<div class="example">
Similary, the production:
```
String => u32(C) CodePoint{C}
```

denotes the fact that a `String` is represented by a count `C` -- which
satisfies the `u32` non-terminal -- followed by `C` CodePoints.
</div>

For convenience, in the presentation, where a literal string is required in the
encoding it is listed using the string's characters spaced out.

<div class="example">
For example, the encoding:

```
0x06 S t r i n g
```
is a convenience form of the sequence of bytes:

```
0x06 0x53 0x74 0x72 0x69 0x6e 0x67
```
</div>

Note: all of the type names defined in this document use only ASCII characters
in their name.

Where a terminal or non-terminal is repeated, its argument may take the form of
_Identifier*_. For example, in

<div class="example">

```
TupleType(T*) => 0xtt u32(C) Type{C}(T*)
```

the _Type_ non terminal has both a count and an argument expression. In this
case, the argument _T_ refers to the argument of the _Type_ non-terminal, and
_T*_ refers to the vector of occurrences of _Type_.

### Standard Scheme

All of the encodings follow a common scheme, consisting of a single lead-in byte
followed by type specific content. Any collections are preceded by a length
(encoded as an LEB) so that a streaming parser knows exactly how much input to
consume.

#### Encoding Integers

Apart from representing integer values themselves, the encoding of other
elements of the IDL also requires the representation of integers; for example,
in representing a `String` value, its length is also required in the encoding.

Integers are encoded using the LEB128 variable length integer encoding. Although
the LEB format is able to represent both signed and unsigned integers, this
specification only uses unsigned integers.

```
7bit(0) => 0x00
7bit(1) => 0x01
...
7bit(127) => 0x7f

8bit(0) => 0x80
8bit(1) => 0x81
...
8bit(127) => 0xff

u32(Ix) => 7bit(Ix)
u32(Ix<<7+Lx) => 8bit(Lx) u32(Ix)
```

#### Encoding Vectors 

A vector of entities is encoded as a length integer (itself encoded as LEB128)
following length entries of the vector -- each of which is of the expected type.

#### Encoding Strings

A string is encoded as a vector of Codepoints represented as sequences of UTF8
bytes.

```
String(C*) => u32(L) CodePoint{L}(C*)
```

### Encoding Types

There are several different kinds of type represented in the encoding:

```
Type => FunctionType
Type => TupleType
Type => QuantifiedType
Type => TypeInterface
Type => NominalType
```

#### Function Type

Function types come in two flavors: exception throwing or not. This is reflected
in two encodings for function types:

```
FunctionType => 0xft TupleType Type
FunctionType => 0xfe TupleType Type Type
```


#### Tuple Type

A tuple type consists of a sequence of types:

```
TupleType => 0xtt i32(A) Type{A}
```

#### Quantified Type

Quantified types come in two forms, universal and existential. They are used to
introduce scoping of names.

```
QuantifiedType => UniversalType
QuantifiedType => ExistentialType

UniversalType => 0xut TypeVar Type
ExistentialType => 0xet TypeVar Type

TypeVar => 0xtv i32(ix)
TypeVar => 0xtk i32(Ar) i32(ix)

```

The bound type variable in a _QuantifiedType_ may either be a simple type or it
may be a type function variable. In the latter case, it is associated with an
arity -- the expected number of argument types to the type function.

Each type variable also has a _type index_. This is a number that identifies the
type within the body of the quantified type. The type index should be a positive
integer -- negative indices are reserved for standard or predefined types.

#### Type Interface

#### Nominal Type

#### Integer Type

#### Float Type

#### String Type

#### Boolean Type

#### Sequence/1 Type

#### Reference/1 Type

#### Option/1 Type

### Encoding Values

#### Tuple Value Encoding

#### Record Value Encoding

#### Variant Encoding

#### Integer Encoding

#### Floating Point Encoding

#### String Encoding

#### Encoding Sequences

### Examples

Note: This section is non-normative.
