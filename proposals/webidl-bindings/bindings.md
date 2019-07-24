# WebAssembly Interoperability Bindings

This specification defines an mechanism where WebAssembly modules can specify
access to and provisioning of APIs expressed using wi-IDL.

## Design Concepts

In any actual use of an API there are three parties:

+ Provider

    The provider of an API is the module or other entity that
is responsible for implementing the API.

    For any given API schema there may be multiple providers that offer that API;
however, in a given pairing of client and provider there may only be a single
provider for each API.

+ Client

    The client of an API is a module or other entity that is
using (or consuming) the services provided by the API.

+ Host

    The host is an entity -- such as a browser -- that is provding
the underlying computational substrate that providers and clients use to access
and provision API-based services.

    The host is the entity in a usage scenario that instantiates the client and/or
provider modules and is responsible for establishing the connections between
them.

In order for a client to access an API provided by a provider, both the client
and the provider specify _bindings_ that map imports (exports) into a common
definition language -- the wi-IDL.

The role of the host is to match the bindings offered by the client and provider
and to generate necessary transformations so that the client can access the API
and that the provider can service it.

Note that the host always consumes pairs of bindings: one from the provider and
one from the client.

+ Binding

    A binding is an expression that provides evidence for an
embedder for how an operation expressed in the IDL is to be consumed or
realized. This is effected through the concept of a *binding lambda*.

+ Coercion Operator

    A _coercion operator_ is a function that transforms a WebAssembly value into
    a wi-IDL value _or vice versa_.
    
    Coercion operators are not conventional functions because they map values
    from one logical domain to another. In particular, they typically do not
    have a type signature that can be expressed in either the WebAssembly type
    system or the wi-IDL type schema.
    
    There are generally two main kinds of coercion operators: lifting operators
    and lowering operators. Lifting operators are used to map WebAssembly values
    into wi-IDL values; and lowering operators are used to map wi-IDL values
    into WebAssembly values.
    
    Since a complete binding as presented to a host consists of pairs of
bindings; it is often the case that a lifting coercion operator is paired with a
lowering coercion operator. This results in a net coercion that is neither
lifting nor lowering (although some data transformation may be necessary).

+ Type Closed Expression

    An expression is type closed if it has a valid type -- either as a
    WebAssembly type or as a wi-IDL type.
    
    Coercion operators are not typically type closed; but binding lambdas are.

+ Binding Lambda

    A *BindingLambda* is an expression that encapsulates either access to an
operation or provision of an operation. Binding lambdas have a function
signature; if it is an access binding then the signature is a WASM function
signature; if it is a provision signature then its type signature is a wi-IDL
function signature.

+ Access lambda
  An _access lambda_ represents a specification of how an operation specified in
wi-IDL (typically an imported function) is accessed from WASM. It takes the
form:

    ```
    (access (Name1,...,Namen) BindingExpression)
    ```

    where Namei are identifiers and Expression is an expression involving the
various Name parameters together with appropriate lifting and lowering
operators. The type of this lambda is a WASM function type.

<div class="example">
For example, the lambda:

```
(access ($B $L) ($integer-to-i32 (utf8-count (base-len-as-string $B $L))))
```

denotes an access lambda that accesses a function utf8-count (which presumably counts the number of unicode code points in a string). The sub-expression

```
(base-len-as-string $B $L)
```

represents a coercion expression that maps a string represented as a base
address and byte count (B & L) into the wi-IDL concept of `String`.

The outermost expression, involving the coercion operator integer-to-i32 maps
the wi-IDL concept of integer to the wasm type of i32 -- which is typically a
nop; depending on the actual implementation of utf8-count.
</div>

+ Provision Lambda

The *Provision Lambda* represents the counterpoint to the *Access Lambda*; where
the latter enables a WebAssembly module to access an external API expressed in
terms of the wi-IDL type schema; the Provision Lambda allows a WebAssembly
module to export functionality expressed in terms of wi-IDL types.

A provision lambda specifies how a particular function implemented as
WebAssembly code can be interpreted in terms of implementing a schema defined
API.

+ Principal Function Call

When constructing an access lambda, or when constructing a provision lambda
there is often a function call within the appropriate binding expression that
represents the main or principal call. In the case of an access lambda, this
principal call is normally a call to a wi-IDL function; whereas in the case of
a provision lambda, the principal function call is typically to a webAssembly
function that has been exported.

However, for a variety of reasons, there may be multiple calls expressed in an
acess or provision lambda. For example, any time a `string` value is
communicated (either as the result of a principal call or as an argument to a
principal call) there may be additional calls to functions that allocate the
string value in linear memory and/or calls to functions that lift a block of
linear memory into a string value.

From a technical perspective, there is actually very little that distinguishes
such ancillary calls from the principal function call. Furthermore, there may be
use cases (such as when testing a webAssembly module) where there is no actual
principal function invoked.


## Abstract Syntax

This section defines the legal forms of interoperability bindings. In parallel
we also show the 'W-expression' form of each term.

>There are many cases where it is important to understand whether a given
>expression refers to an IDL concept or whether it refers to a WebAssembly
>concept. As an aid to clarity, we mark identifiers that refer to a WebAssembly
>concept with a prefix `$`. Identifiers that refer to IDL concepts are not
>marked; and identifiers that refer to both WebAssembly and to IDL concepts (such
>as binding operators) will be marked if the _result_ of the operator refers to a
>WebAssembly element.

### Binding Lambda

A _BindingLambda_ is a function that defines how a wasm function may access or
provide an external API function.

```
BindingLambda => AccessLambda
BindingLambda => ProvisionLambda
```

### Access Lambda

An _AccessLambda_ is a function that defines how an imported function should be
realized as an EB Function.

```
AccessLambda => 'access' Tuple '->' AccessExpression
```

```
W-AccessLambda ::= '(' 'access' W-Tuple W-AccessExpression ')'
```

For example, the access lambda 


#### Access Lambda Type

An access lambda has a type signature that is expressed as a WASM function type.

```
E |= AccessLambda : functype
```

### Provision Lambda

A _ProvisionLambda_ is a function that defines how an exported EB function is
realized term terms of exported wasm elements.

```
ProvisionLambda => 'provide' Tuple '=>' ProvisionExpression
```

or, as a W-Expression form:

```
W-ProvisionLambda ::= '(' 'provide' W-Tuple W-ProvisionExpression ')'
```

#### Provision Lambda Type

A provision lambda has a type signature that is expressed as a wi-IDL schema
type:

```
E |= ProvisionLambda : FunctionType
```

The type signature of a provision lambda is a schema type. It typically uses
lowering operators -- together with a call to an exported WASM function.

<div class="example">

For example, supposing that the utf8-count function alluded to above were
implemented as an exported wasm function -- `$utf8-count`, then there may be a
provisioning lambda that supports it:

```
(provide (T)
   ($i32-as-integer
      ($_cp_count ^ (string-to-base-ptr T $Buffer $Size))))
```

Note that the string-to-base-ptr operator returns a two-tuple: the address in
memory of where the string data is and the size, in bytes, of the resulting
string. This tuple is unpacked into two values -- using the `^` operator --
which are used as two arguments to the `$_cp_count` principal function call.
</div>

### Binding Expression

A _binding expression_ defines the legal forms that may be used in bindings:

```
BindingExpression => Variable
BindingExpression => Literal
BindingExpression => OperatorCall
BindingExpression => AccessLambda
BindingExpression => ProvisionLambda
BindingExpression => TryCatch
BindingExpression => Throw
```

>Note that we use the term `OperatorCall` rather than `FunctionCall` because
>some of the operators are not functions in the normal sense of the word.

### Variable

A _Variable_ refers to either an abstract IDL value (as typically identified in
a binding lambda) or to an imported or an exported webAssembly function.

As a convention, we prefix webAssembly functions and values with a `$` character.

```
Variable => Name
Variable => $ Name
```

### Literal

A _Literal_ is an expression whose value is constant. 

```
Literal => Term
```

Whether a given literal expression is valid depends on it's type schema as well
as it's type. For example, _NumericLiteral_ values that represent webAssembly
values depend on the specific webAssembly type (`i32` vs `f64` for example) and
then whether the literal value 'fits into' the webAssembly type. On the other
hand, all finite integer values are legitimate instances of the wi-IDL Integer
type.

Similarly, _String_ literals are only permitted where they denote wi-IDL
`String` values.

### Throw

There are two variants of `throw` expression:

```
Throw => 'throw' Expression 
Throw => '$throw' ExceptionIndex Expression
```
As a W-Expression, this is:

```
W-Throw ::= '(' 'throw' W-Term ')'
W-Throw ::= '(' '$throw' W-Term ')'
```

The `$throw` form is used to denote the situation where a webAssembly exception
is indicated; the `throw` form is used to denote a wi-IDL exception.

The `ExceptionIndex` argument refers to the index of the exception type in the
enclosing webAssembly module.

### TryCatch

A _TryCatch_ expression denotes a scope in which exceptions that are thrown in a
sub-scope are handled.

```
TryCatch => 'try' Expression 'catch' ProvisionLambda
TryCatch => '$try' Expression '$catch' AccessLambda
```

The `$try` form is used in cases where the overall form represents an access to
a wi-IDL function.

Coercion operators can fail, for a variety of reasons. In addition, when
accessing an operation the operation itself may fail. One use of _TryCatch_
expressions is to be able to handle potentially failing coercion operators and
map failure of a coercion to another form of expression.

Other uses of _TryCatch_ forms are to enable coercion of exceptions themselves;
for example to map wi-IDL exceptions to webAssembly exceptions and vice-versa.

Note that most coercion operators have duels; for example, duel to
`i32-as-integer` is the `integer-to-i32` operator. Generally, coercion
exceptions arise because of a combination of two such operators. Where an
exception is thrown during coercion involving such a pair, the exception is
thrown _as though_ it were thrown from the reducing operator.

The precise situations where coercion operators throw exceptions are detailed in
the descriptions of the individual operators.

## Bindings WebAssembly Section

A bindings section defines the bindings of imported functions or the exported
functions of a module. Both of these take the form of a special section in the
webAssembly module.

### Import Bindings

The _ImportBindings_ section is a special webAssembly section that defines the
bindings for the imports from a module.

```
ImportBinding => 'import-binding' ModuleName FunctionName AccessLambda
```

An _ImportBinding_ is a specification of how an individual imported function
should be seen as accessing a functionality (sic) represented as a wi-IDL
function.

The W-Expression production for _W-ImportBinding_ is

```
W-ImportBinding ::= '(' 'import-binding' ModuleName FunctionName W-AccessLambda ')'
```
<div class="example">

For example, to access an imported function that counts the number of code
points in a string, the binding signature of the corresponding export may look
like:

```
{ codePointCount : (string) => integer 
    ...
)
```
The import, together with the bindings looks like:

```
(module
  (import "unicode-lib" "codePointCount" (func (param i32 i32) (result i32)))
  ...
  (import-binding "unicode-lib" "codePointCount"
    (access ($Base $Len) (integer-to-i32 
      (codePointCount (base-len-as-string $Base $Len)))))
  ...
)
```
</div>

### Export Bindings

The _ExportBindings_ special section defines the bindings for functions that are
exported by the module. Like imports, exports from a webAssembly module are
managed on a per function basis.

```
ExportBinding => 'export-binding' FunctionName ProvisionLambda
```

The W-Expression form of an export binding is given by:

```
W-ExportBinding ::= '(' 'export-binding' FunctionName W-ProvisionLambda ')'
```

<div class="example">

The corresponding module that exports the code point counter alluded to above
may resemble:

```
(module
  ...
  (export "codePointCount" (func $_cp_count))
  ...
  (func $_cp_count (return i32) (param $b i32 $l i32)
    ...
  )
  ...
  (export-binding "codePointCount"
    (provide (S) 
      (i32-as-integer 
        ($_cp_count ^ (string-to-base-ptr S $buffer $buffLen)))))
  ...
)
```

</div>

## Coercion Operators

The coercion operators are used to lift webAssembly values into wi-IDL values
and vice versa.

The general form of a coercion operator takes the form of a function call; but as
previously noted, coercion operators are not conventional functions.

### Lifting Operators

#### i32-as-integer

The `i32-as-integer` operator maps a webAssembly `i32` value into the wi-IDL
space of `Integer`. 

Note that this does not necessarily imply any _transformation_ of the bits
representing the integer value.

```
i32-as-integer : (i32) => Integer
```

#### i64-as-integer

The `i64-as-integer` operator maps a webAssembly `i64` value into the wi-IDL
space of `Integer`. 

Note that this does not necessarily imply any _transformation_ of the bits
representing the integer value.

```
i64-as-integer : (i64) => Integer
```

#### base-len-as-string

The `base-len-as-string` operator maps a utf8 encoded sequence of bytes into the wi-IDL space of `String`

```
base-len-as-string: (i32 i32) => String
```

### Lowering Operators

Lowering operators are the complement of the lifting operators; generally they
are used to map values in the wi-IDL space into the webAssembly space.

#### integer-to-i32

The `integer-to-i32` operator maps a wi-IDL `Integer` value into the webAssembly
space of 32-bit integers.

```
Integer-to-i32 : (Integer) -> i32
```

Note that this operator _may_ involve a _truncation_ of values if the original
source of the `Integer` value had a higher precision than 32 bits.

In particular, the identity

```
(integer-to-i32 (i32-as-integer X)) == X
```
is satisfied, whereas the identity

```
(integer-to-i32 (i64-as-integer X)) == X
```
is not, in general.

#### integer-to-i64

The `integer-to-i64` operator maps a wi-IDL `Integer` value into the webAssembly
space of 64-bit integers.

```
Integer-to-i64 : (Integer) -> i64
```

Note that this operator _may_ involve a _truncation_ of values if the original
source of the `Integer` value had a higher precision than 64 bits.

Note that this operator _may_ involve _sign extension_ if the original `Integer`
source had less than 64 bits of precision.

#### unsigned-integer-to-i64

The `unsigned-integer-to-i64` operator maps a wi-IDL `Integer` value into the webAssembly
space of 64-bit integers.

```
unsigned-integer-to-i64 : (Integer) -> i64
```

Note that this operator _may_ involve a _truncation_ of values if the original
source of the `Integer` value had a higher precision than 64 bits.

If the original `Integer` source has _fewer_ than 64 bits of precision, then the
output is zero-filled to the left.


#### string-to-base-ptr

The `string-to-base-ptr` maps a wi-IDL `String` value into a block of webAssembly linear memory -- as a sequence of utf-8 encoded bytes.

```
string-to-base-ptr : (String i32 i32) => (i32 i32) $throws i32 
```

The three arguments to `string-to-base-ptr` are the `String` value itself, the
base address of a block of linear memory -- together with the size of the
buffer. The returned result is the address of the utf8-encoded sequence of bytes
together with its length -- as a 2-tuple.

Note that although in many cases the returned address of the string may be
within the allocation buffer, it is not guaranteed. For example, if the
corresponding lifting operator were a `base-len-as-string` operator that was
accessing the same linear memory then it is possible that no copying actually
takes place.

>Question: Should an exception be thrown anyway if the length of the string is
>larger than that allocated for, even if the allocation buffer is not used?

