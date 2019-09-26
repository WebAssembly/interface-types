# Interface Types Proposal

The proposal adds a new set of **interface types** to WebAssembly that describe
high-level values (like strings, sequences, records and variants) without
committing to a single memory representation or sharing scheme. Interface types
can only be used in the interfaces of modules and can only be produced or
consumed by declarative **interface adapters**.

The proposal is semantically layered on top of the WebAssembly [core spec]
(extended with the [multi-value] and [reference types] proposals), and
adds only the ability to adapt the imports and exports of a WebAssembly
module at points which are already host-defined behavior. All adaptations are
specified in a [custom section] and this feature can be polyfilled using the
[JS API].

1. [Motivation](#motivation)
1. [Overview](#overview)
1. [Walkthrough](#walkthrough)
1. [Web IDL integration](#web-idl-integration)
1. [FAQ](#FAQ)


## Motivation

This proposal is motivated by 3 distinct problems:

**Optimizing calls to Web APIs**

With the [reference types] proposal, WebAssembly code can pass around arbitrary
JavaScript values using the [`anyref`] type. By default, JavaScript
values flowing into WebAssembly get boxed into `anyref` values which are then
unboxed whenever flowing back out to JavaScript. These boxed values are
opaque to WebAssembly, but, by importing JavaScript builtin functions like
[`Reflect.construct`], [`Reflect.apply`], [`Reflect.set`] and [`Reflect.get`],
WebAssembly modules are able to perform many fundamental JavaScript operations
without requiring JavaScript glue code, ultimately allowing WebAssembly to call
any method defined in Web IDL by synthesizing appropriate JavaScript values.

However, just removing JS glue code between WebAssembly and Web IDL doesn't
remove all the unnecessary call overhead. For one thing, all the auxiliary
calls to `Reflect` builtins may end up running slower than the highly
JIT-optimized JS glue code. For another, glue code or not, synthesizing
JavaScript values often requires copying WebAssembly values and linear memory
into GC-allocated JS strings and objects that immediately become garbage after
the call. Lastly, calling a statically-typed Web IDL-defined method with
dynamically-typed JavaScript values can add additional runtime overhead.

With the addition of interface types, the Web IDL spec could add a
"WebAssembly binding" section (symmetric to the current [ECMAScript binding]
section) which defines how WebAssembly values (including values of interface
types) can be converted to and from Web IDL values, without going through
JavaScript, even for high-level types like `DOMString` and [Dictionary]. An
optimizing engine can then compile the declarative, statically-typed interface
adapters into efficient stubs that call more-directly into the API's
implementation.

**Enabling "shared-nothing linking" of WebAssembly modules**

While WebAssembly intentionally supports [dynamic linking], in which multiple
instances share the same memory and table, emulating [native dynamic linking],
this type of linking is more fragile:
* Modules must carefully coordinate (at the toolchain and source level) to
  share data and avoid generally clobbering each other.
* Corruption in one module can affect other modules, with certain bugs
  only manifesting with certain combinations of modules.
* Modules are less able to encapsulate, leading to unwanted representation
  dependencies or violations of the Principle of Least Authority.

In contrast, "shared-nothing linking" refers to a linking scheme in which
each module defines and encapsulates its own memory and table. However, there is
currently no host-independent way to implement even basic shared-nothing linking
operations like copying an array of bytes from one module's linear memory to
another.

With interface types available to use at the boundary between two
modules, exports and imports can take, e.g., an abstract sequence of bytes,
or a string, or a sequence of pairs of strings and integers, etc. With interface
adapters to define how these abstract values are to be read from or written into
linear memory, modules can use the wasm engine to take care of copying data
between two modules' linear memories while allowing both modules to
maintain full encapsulation.

**Defining language-neutral interfaces like WASI**

While many [WASI] signatures can be expressed in terms of `i32`s and,
in the future, references to [type imports], there is still a need for WASI
functions to take and return higher-level value types like strings or sequences.
Currently, these values are specified in terms of linear memory. For example,
in [`path_open`], a string is passed as a pair of `i32`s which represent the
offset and length, respectively, of the string in linear memory.

However, specifying this single, fixed representation for data types like
strings will become problematic if:
* WebAssembly gets access to GC memory with the [GC] proposal and the caller
  wants to pass a `ref array u8`;
* the host has a native string type which is not the same as the
  GC `ref array u8` (e.g., a JS/DOM string) which WASI would ideally accept
  without copy; or
* the need arises to allow more than just UTF-8 encoding.

With interface types, WASI functions can simply use the interface type `string`,
allowing abstract string values to be created from linear memory, GC memory and
host-string values, with the source and encoding specified explicitly by the
calling module and extensible over time.


## Overview

This proposal builds on the following existing and separately proposed 
high-level WebAssembly concepts:
* **value type**: the set of types defined by the [core spec][Value Types]
  that can be used to define globals, locals, functions, etc
* **module**: the basic unit of WebAssembly code whose structure is defined in
  the [core spec][Module Syntax] and which may only use value types in its
  functions
* **module type**: currently defined by the [core spec][Old Module Type] as a
  mapping from imports to exports. More recently, there is a [proposal][New Module Type]
  to generalize module types to include import/export names and have text
  format parse rules so that module types can be written separately from
  modules.
* **known section**: sections defined by the [core spec's binary format][Known Section]
  which collectively decode a module
* **custom section**: sections defined by the [core spec's binary format][Custom Section]
  whose contents are uninterpreted by the core spec but can be interpreted by
  other tools or specifications (including this proposal)

This proposal defines the following new high-level concepts:
* **interface type**: a new set of types that describe abstract, high-level values
* **adapter instruction**: a new set of instructions which may produce or consume
  values of both value types and interface types
* **adapter function**: a new set of functions whose signatures may contain
  interface types and whose bodies are composed of adapter instructions
* **interface adapter**: a collection of adapter functions that are applied to a
  module's imports and exports
* **adapted module**: the result of applying an interface adapter to a module,
  which encapsulates the core module's imports and exports, exposing only the
  adapted imports and exports
* **module interface type**: a superset of the core module type which
  classifies an adapted module and thus includes function signatures with
  interface types
* **interface adapter custom section**: the new custom section defined by
  this proposal which encodes an interface adapter which is to be applied
  to the core module defined by the known sections to produce an adapted module
  that shall be executed by the host in place of the core module

Collectively, these concepts and their relationships can be visualized together
in the following diagram:

![Concepts overview diagram](overview.svg)


## Walkthrough

> **Note** The syntax and semantics presented below is still very much in flux.
> Also, there's currently a large number of TODOs, so the walkthrough is
> definitely incomplete.

The walkthrough starts by introducing the `string` type and then using `string`
in the parameters and results of first exports and then imports. Afterwards,
other types and concepts are introduced.
1. [Export returning string (statically allocated)](#export-returning-string-statically-allocated)
1. [Export returning string (dynamically allocated)](#export-returning-string-dynamically-allocated)
1. [Export receiving string](#export-receiving-string)
1. [Strings in imports](#strings-in-imports)
1. [Lifting, lowering and laziness](#lifting-lowering-and-laziness)
1. [Strings with the GC proposal](#strings-with-the-GC-proposal)
1. [Integers](#integers)
1. [TODO](#TODO)

### Export returning string (statically allocated)

Let's start with a WebAssembly module that you can write today (with 
[multi-value]) that returns a string that is stored at a fixed location in
linear memory:

```wasm
(module
  (memory (export "mem") 1)
  (data (i32.const 0) "hello there")
  (func (export "greeting_") (result i32 i32)
    i32.const 0    ;; offset of string in memory
    i32.const 11   ;; length
  )
)
```

If we want this module to be ergonomically callable from JS, we'll need to
wrap this wasm module with a JS module that uses the [JS API] to read the
strings' bytes from the `mem` export to create a JS string value. Tools like
[Embind] and [wasm-bindgen] can be used today to generate this glue
automatically from annotations in C++ and Rust source code, respectively.

With this proposal, however, we can use the `string` interface type as the
result type of our module's export. `string` is defined abstractly as a sequence
of [Unicode code points] and does not imply a [Unicode encoding scheme]
or whether the encoded bytes are stored in linear memory, but we can specify
all this by adding the following to our module:

```wasm
  (@interface func (export "greeting") (result string)
    call-export "greeting_"
    memory-to-string "mem"
  )
```

The statement begins with `@interface` since it is not part of the [core spec]
text format (as recommended by the [Custom Annotation] proposal). This `func`
statement defines two things:
* an export `greeting` in the [adapted module](#Overview) whose result is the
  `string` interface type
* an [adapter function](#Overview) that implements `greeting`

While the set of *instructions* in the body of the adapter function is
distinct from the set of [core wasm instructions][Instructions],
the overall *structure* of the adapter module is the same as a core WebAssembly
function. In particular:
* The adapter function body is a (possibly nested) sequence of instructions.
* Instructions are defined to pop operands and push results onto the stack.
* Validation checks that the types of instructions' signatures line up.
* Instructions can either be written as a linear sequence (as shown above)
  or, equivalently, as [Folded S-expressions].
* The set of instructions is fixed by the specification, but growable over time
  by proposing extensions to the specification.

The specific adapter function shown above uses two adapter instructions:
* The `call-export` instruction calls the `greeting_` export of the core wasm
  module which leaves two `i32` values on the stack.
* The `memory-to-string` instruction pops the two `i32` values, reads and
  decodes UTF-8 bytes from `mem`, and pushes the resulting `string` value.

Having added this adapter function, the `.wat` file now defines an [adapted module](#Overview)
with no imports and whose only export is the (adapted) `greeting` export.
Importantly, the `greeting_` and `mem` core-module exports are not exported
from the adapted module: they are fully encapsulated by the adapted module.
While it is possible for an adapted module to re-export core-module exports,
this must be explicitly specified with an `@interface` statement.

Another note is that, since `greeting_` and `greeting` are in separate export
namespaces (of the core and adapted modules, resp.), they could have the same
name. The trailing underscore is only added for clarity in this walkthrough.

### Export returning string (dynamically allocated)

The previous section's example has the simplifying property that the string is
statically allocated. What happens if `greeting_` wants to `malloc` the
resulting string? For example:

```wasm
(module
  (memory (export "mem") 1)
  (func $malloc (param i32) (result i32) ...)
  (func (export "free") (param i32) ...)
  (func (export "greeting_") (result i32 i32)
    ;; compute allocation size
    call $malloc
    ;; initialize malloc'd memory with string
    ;; return pointer and length
    ;; caller must call "free" when done with the string
  )
)
```

This situation would naturally arise when exporting, e.g., a C++ function that
returns a `std::string` or `std::shared_ptr<char[]>`. In both cases, the
returned C++ object essentially contains a pointer along with the assumption
that the calling C++ code is responsible for calling the object's destructor,
which then frees the memory or drops a reference count, resp. But if that
function is exported from an adapted wasm module, the caller isn't C++ code,
it's the adapter function.

What we need is to be able to call `free` (or some other export corresponding
to a destructor) after the linear memory bytes are read by `memory-to-string`.
Because there is no limitation on the number of `call-export`s in an adapter
function, a first attempt at a solution would be to just call `free` directly:

```wasm
  (@interface func (export "greeting") (result string)
    call-export "greeting_"
    dup
    memory-to-string "mem"
    swap
    call-export "free"
  )
```

Here we use classic stack manipulation operations to duplicate the `i32` pointer
value so that it can be read by both `memory-to-string` and the call to `free`.
Unfortunately, there is a problem: anticipating the [exception handling]
proposal, if an exception is thrown between the `call-export "greeting_"` and
the `call-export "free"`, the memory will be leaked. With more complicated
exported function signatures, there can be many instructions in this range
which can throw.

To address this problem, as well as another problem described [below](#lifting-lowering-and-laziness) 
there is a `defer-call-export` instruction:

```wasm
  (@interface func (export "greeting") (result string)
    call-export "greeting_"
    defer-call-export "free"
    memory-to-string "mem"
  )
```

This instruction requests that the given function (`free`) be called on all
exits from the current frame (normal and exceptional). The arguments for
this deferred call are copied from the top of the stack at the time of the
`defer-call-export` (the number and types of which are determined by the
callee signature, like a normal wasm `call`). Unlike a normal wasm `call`,
however, these arguments aren't popped, but left on the stack, which is
especially convenient for the common case shown here.

### Export receiving string

Now what if instead our module needs to take a string as a parameter? This
can be handled symmetrically using the `string-to-memory` adapter
instruction, as shown in this example:

```wasm
(module
  (memory (export "mem") 1)
  (func (export "malloc") (param i32) (result i32) ...)
  (func (export "free") (param i32) ...)
  (func (export "log_") (param i32 i32) ...)
  (@interface func (export "log") (param $str string)
    arg.get $str
    string-to-memory "mem" "malloc"
    call-export "log_"
  )
)
```

Here, the `string` value is a parameter. Like core WebAssembly, arguments are
not automatically pushed on the stack and must be pushed explicitly, here using
the `arg.get` adapter instruction.

The `string-to-memory` instruction then:
* pops the string off the stack
* computes the number of bytes to encode the string in UTF-8
* calls the given `malloc` exported function to allocate that number of bytes
* UTF-8-encodes the string into `mem` at the offset returned by `malloc`
* pushes both the offset and number of bytes as two `i32`s on the stack

An important technicality is that the `string` interface type is a sequence of
[Unicode code points] while UTF-8 only encodes a sequence of [Unicode scalar values].
Thus `string` allows [surrogate code points] which is imporant later in
[Web IDL Integration](#web-idl-integration). However, this also means that
`string-to-memory` needs to define what happens if a surrogate code point is
encountered while encoding. On the Web, UTF-8 [encoding][WHATWG Encoding] is
defined to replace lone surrogates with the [Replacement Character]. Encoding
the surrogate code point directly ([WTF-8]) is another option. Trapping is a
conservative third option. The `string-to-memory` instruction could allow all 3
options, specified as another immediate, with a specified default in the text
format.

A related question is whether non-UTF-8 encodings should be supported. While
there are [ecosystem benefits](http://utf8everywhere.org/) to allowing only UTF-8,
there may (one day) be practical reasons to support other encodings with either
new instructions or adding a new flag bit to `string-to-memory`.

### Strings in imports

The proposal also allows adapting imports as well. For example, to import a
logging function that takes a string:

```wasm
(module
  (memory (export "mem") 1)
  (func (import "" "log_") (param i32 i32))
  ...
  (@interface func $log (import "" "log") (param $arg string))
  (@interface implement (import "" "log_") (param $ptr i32) (param $len i32)
    arg.get $ptr
    arg.get $len
    memory-to-string "mem"
    call-import $log
  )
)
```

The first `@interface` statement defines an import in the adapted
module and thus has no body (it's an import). Note that the syntax for
`@interface func` imports and exports are symmetric to core wasm `func`
imports and exports.

The second `@interface` statement uses `implement` to indicate that
it is defining an adapter function to implement the *core module* import with
the given name and signature (in the example, `log_ : [i32,i32] → []`).

This example uses the `memory-to-string` instruction that was previously shown
in an export adapter function; now `memory-to-string` is applied to the
*argument* of `call-import` instead of to the *return value* of `call-export`.
Note also that there is no `defer-call-export "free"` instruction in this
example. While it would be *possible* to do so, since the caller of the adapter
is wasm code, it's simpler and potentially more efficient to let the caller
worry about when to free the string.

Using `string` as the return value of an import is symmetric to above,
using the previously-introduced `string-to-memory` instruction.

```wasm
(module
  (memory (export "mem") 1)
  (func (export "" "malloc") (param i32) (result i32) ...)
  (func (import "" "greeting_") (result i32 i32))
  ...
  (@interface func $greeting (import "" "greeting") (result string))
  (@interface implement (import "" "greeting_") (result i32 i32)
    call-import $greeting
    string-to-memory "mem" "malloc"
  )
)
```

### Lifting, lowering and laziness

With `memory-to-string` and `string-to-memory`, we can see a pattern that will
be repeated for all subsequent interface types: there is one set of **lifting
adapter instructions** that "lift" core wasm values into interface-typed values
and another set of **lowering adapter instructions** that "lower" interface-typed
values into core wasm values. Moreover, the validation rules for adapter
instructions ensure that the *only* way to consume an interface-typed value
is with a lowering instruction.

With the 2×2 matrix of {lifting,lowering}×{import,export} covered for `string`,
we can now examine a complete example of [shared-nothing linking](#Motivation).
On the provider side, we have an adapted module that exports a function `get`
which takes and returns a string:

```wasm
(module
  (memory (export "mem1") 1)
  (func (export "mem1Malloc") (param i32) (result i32) ...)
  (func (export "mem1Free") (param i32) ...)
  ...
  (func (export "get_") (param i32 i32) (result i32 i32) ...)
  (@interface func (export "get") (param $key string) (result string)
    arg.get $key
    string-to-memory "mem1" "mem1Malloc"
    call-export "get_"
    defer-call-export "mem1Free"
    memory-to-string "mem1"
  )
)
```

On the client side, we have an adapted module that imports and calls `get`:

```wasm
(module
  (memory (export "mem2") 1)
  (func (export "mem2Malloc") (param i32) (result i32) ...)
  ...
  (func $get_ (import "" "get_") (param i32 i32) (result i32 i32))
  (@interface func $get (import "kv-store" "get") (param string) (result string))
  (@interface implement (import "" "get_") (param $ptr i32) (param $len i32) (result i32 i32)
    arg.get $ptr
    arg.get $len
    memory-to-string "mem2"
    call-import $get
    string-to-memory "mem2" "mem2Malloc"
  )
  (func $randomCode
    ...
    call $get_
    ...
  )
)
```

Looking at this example, an important question is: will the engine be able
to copy directly between the caller's and callee's linear memories for the
parameter and result strings? If `memory-to-string` had the same [eager evaluation]
rules as all core wasm instructions, the answer would be "probaly not", because
any unknown side effects between `memory-to-string` and `string-to-memory` (such
as the calls to `mem1Malloc` and `mem2Malloc`) could force the engine to
conservatively make intermediate copies.

To solve this problem and categorically remove all such intermediate copies,
lifting instructions are specified to be evaluated [lazily]. "Lazily" means 
that `memory-to-string` doesn't read the source linear memory until the point
at which the resulting `string` is lowered by `string-to-memory`. If the
`string` result is never lowered, the source memory is never read. If the
`string` is lowered multiple times, the source memory is read anew each time.

While the implementation of lazy evaluation in a general-purpose programming
language may add runtime overhead (e.g., requiring values to be represented
by [thunks]), in the restricted, declarative setting of adapter functions, a
pair of adapted modules can *always* be [partially evaluated] into a single
core wasm module that contains no adapter instructions, using core wasm
instructions like [`memory.copy`] to copy directly from memory to memory.
This partial evaluation will be formalized as a set of provably-equivalent
rewrite rules which engines can use to *predictably* optimize adapter
function code into equivalent core wasm code (at instantiation time, when
client and provider are known).

For example, the above two adapted modules can be merged and rewritten into the
following core wasm module (using the [multi-memory] proposal and the `let`
blocks introduced by the [function references] proposal):

```wasm
(module
  (memory $mem1 1)
  (memory $mem2 1)
  (func $mem1Malloc (param i32) (result i32) ...)
  (func $mem1Free (param i32) ...)
  (func $mem2Malloc (param i32) (result i32) ...)
  ...
  (func $validateUtf8 (param i32 i32) ... defined by Unicode spec ...)
  ...
  (func $get_ (param i32 i32) (result i32 i32) ...)
  (func $get (param $keySrc i32) (param $keyLen i32) (result i32 i32)
    (call $validateUtf8 (local.get $keySrc) (local.get $keyLen))
    (call $mem1Malloc (local.get $keyLen))
    let (local $keyDst i32) (result i32 i32)
      (memory.copy "mem2" "mem1" (local.get $keyDst) (local.get $keySrc) (local.get $keyLen))

      (call $get_ (local.get $keyDst) (local.get $keyLen))

      let (local $valSrc i32) (local $valLen i32) (result i32 i32)
        (call $validateUtf8 (local.get $keySrc) (local.get $keyLen))
        (call $mem2Malloc (local.get $valLen))
        let (local $valDst i32) (result i32 i32)
          (memory.copy "mem1" "mem2" (local.get $valDst) (local.get $valSrc) (local.get $valLen))

          (call $mem1Free (local.get $valSrc))

          local.get $valDst
          local.get $valLen
        end
      end
    end
  )
  (func $randomCode
    ...
    call $get
    ...
  )
)
```

Some interesting things to notice in this code are:
* For the special case of strings, in addition to copying memory, UTF-8
  validation is required at the interface boundary since the engine has no way
  to know that the source bytes are already valid.
* While the original client and provider modules needed to export memory and
  various functions so that they can be used by the containing adapted module,
  the rewritten module can keep these encapsulated and use module-internal
  references.
* The `defer-call-export` in the original adapted module has been
  rewritten into a plain `call` at the end of the rewritten function scope.
  `defer-call-export` can be thought of as a lazily-evaluated call, with
  evaluation occurring at the end of its (post-rewrite) enclosing scope. Thus,
  `defer-call-export` is naturally aligned with lazy lifting, ensuring that
  memory is still allocated when it is read.


### Strings with the GC proposal

In the preceding [shared-nothing example](#shared-nothing-linking-example),
neither module exposes its linear memory or allocator functions to the outside
world, keeping these encapsulated inside their respective adapted modules. In
fact, with the future [GC] proposal, either module can transparently switch to
GC without the other module noticing. For example, using the strawman GC text
format, and assuming a new lifting instruction, `gc-to-string`, and a new
lowering instruction, `string-to-gc`, the above provider module could be
transparently rewritten to use GC:

```wasm
(module
  (type $Str (array i8))
  (func (export "get_") (param (ref $Str)) (result (ref $Str)) ...)
  ...
  (@interface func (export "get") (param $key string) (result string)
    arg.get $key
    string-to-gc
    call-export "get_"
    gc-to-string
  )
)
```

This example shows how having explicit adapter instructions allows new
representation choices to be added over time. Moreover, the provider module is
not required to make an all-or-nothing choice; individual parameters can use
whichever available representation is best.

### Integers

In addition to `string`, the proposal includes the integer types `u8`, `s8`,
`u16`, `s16`, `u32`, `s32`, `u64` and `s64`. Each of these types represent
subsets of ℤ, the set of all integers, with `uX` types representing ranges 
[0, 2<sup>X</sup>-1] and `sX` types representing ranges
[-2<sup>X-1</sup>,2<sup>X-1</sup>-1]. Since values of these types are proper
integers, not bit sequences like core wasm `i32` and `i64` values, there is no
additional information needed to interpret their value as a number.

As with strings, the integral types have associated lifting and lowering
instructions `lift-int` and `lower-int`, which take the source and destination
type as explicit immediates, as with the [`block`][Block Validation]
and [`select`][Select Validation] instructions. For example:

```wat
(module
  (func (export "compute_") (param i64 i32) (result i32) ...)
  ...
  (@interface func (export "compute") (param s8 u64) (result s64)
    arg.get 0
    lower-int s8 i64
    arg.get 1
    lower-int u64 i32
    call-export "compute_"
    lift-int i32 s64
  )
)
```

Here we can see all the interesting possibilities:
* `lower-int s8 i64` converts a signed integer in the range [-128,127] to a
  64-bit value by sign-extension due to the signedness of the source type.
* `lower-int u64 i32` converts an unsigned value in the range [0, 2<sup>64</sup>-1]
  to a 32-bit value by truncation.
* `lift-int i32 s64` converts a 32-bit value to an integer in the range
  [-2<sup>63</sup>,2<sup>63</sup>-1] by sign-extension due to the signedness of
  the destination type.

> **NOTE** In the future we could consider supporting the more general set of
> (u|s)\<bitwidth\> types, supporting more precise static interface contracts.
> The current set is chosen for it's practical application to C and Web IDL.


### TODO

This rough list of topics is still to be added in subsequent PRs:
* bool
* records
* sequences (esp. considering interactions with defer)
* variants
* closures and interaction with [function references]
* re-importing/exporting core module import/exports
* importing an interface type with [type imports]
* subtyping


## Web IDL integration

One of the primary motivations of this proposal is to allow efficient
calls to Web APIs through their Web IDL interface. The way this is to be
achieved is by extending the Web IDL specification to include a "WebAssembly
binding" section which describes how WebAssembly types (including the new
interface types added by this proposal) are converted to and from Web IDL
types.

Since both Web IDL and WebAssembly are statically-typed, this specification
would start by defining when two Web IDL and WebAssembly types **match**. When
two types match, the specification would define how values of the two types are
converted back and forth.

In particular, going down the list of [Web IDL types]:
* [`any`]: the existing WebAssembly [`anyref`] type is already effectively
  the same as `any` in Web embeddings.
* [Primitive types]: WebAssembly value types already can be efficiently
  converted back and forth.
* [`DOMString`]: since the WebAssembly `string` type is defined as a
  sequence of [Unicode code points] and `DOMString` is defined as a sequence of
  16-bit [code units], conversion would be UTF-16 encoding/decoding, where 
  lone surrogates in a `DOMString` decode to a surrogate code point.
* [`USVString`]: a WebAssembly `string` is a superset of `USVString`.
  Conversion to a `USVString` would follow the same strategy as
  [`DOMString`-to-`USVString` conversion] and map lone surrogates to the
  replacement character.
* [`ByteString`]: as a raw sequence of uninterpreted bytes, this type is probably
  best converted to and from a WebAssembly sequence interface type.
* [`object`], [`symbol`], [Frozen array] types: as JS-specific types, these
  could either be converted to and from `anyref` or be statically typed via
  reference to a [type import].
* [Interface] types: while Web IDL defines interfaces as namespace structures
  containing methods, fields, etc., for the specific purpose of typing Web API
  functions, a Web IDL Interface type just defines an abstract reference type
  used in Web API function signatures. WebAssembly can represent this type with
  either an `anyref` (implying dynamic type checks) or via reference to 
  [type import].
* [Callback], [Dictionary], [Sequence] types: these would be converted to and
  from WebAssembly closure, record, and sequence interface types, respectively.
* [Record] types: WebAssembly does not currently have plans for an "ordered map"
  interface type. This type appears to be infrequently used in APIs, and as long
  as that stays the case and uses remain cold, JS objects could be synthesized
  instead.
* [Enumeration], [Nullable], [Union] types: these would be converted to and
  from WebAssembly variant interface types, by imposing various matching
  requirements on the variant type.
* [Annotated] types: the annotations don't change the representation, but imply
  additional dynamic checks on argument values.
* [`BufferSource`] types: `ArrayBuffer`s could be converted to and from
  WebAssembly sequence types, while views would depend on first-class
  slice/view reference types being added to WebAssembly, which has been
  discussed but is not yet officially proposed.

An important question is: what happens, at instantiation-time, when the Web IDL
and WebAssembly signatures don't match. One option would be to throw an error,
however, this would lead to several practical compatibility hazards:
* Browsers sometimes have slightly incompatible Web IDL signatures (which can
  occur for historical compatibility reasons).
* Sometimes Web IDL specifications are refactored over time (e.g., changing a
  `long` to a `double`), assuming the coercive semantics of JavaScript.
* JavaScript Built-in functions that today have no Web IDL signature might be
  imbued with a Web IDL signature by a future iteration of the spec that is
  subtly incompatible with extant uses that depend on coercieve JS semantics.

To address all these concerns, on WebAssembly/Web IDL signature mismatch,
instantiation would fall back to first converting all WebAssembly values to JS
values and then passing those JS values to the existing, more-coercive Web IDL
ECMAScript binding. To help well-intentioned developers avoid unintended
performance degradation, WebAssembly engines could emit warning diagnostics on
mismatch.


## FAQ

### Will the set of adapter instructions grow to duplicate all of WebAssembly?

No; the criteria for adding an adapter instruction is that the instruction must
solve a problem that couldn't otherwise be solved (without significant
overhead) in core WebAssembly. For example, adapter instructions don't need to
include the myriad of numeric conversion operators. Similarly, the proposal can
leverage existing and planned reference types ([`anyref`], [function references],
[type imports], [GC]) to, e.g., define [abstract data types].

### Why not just add adapter instructions to the core WebAssembly instruction set?

The anticipated optimization strategy described [above](#lifting-lowering-and-laziness)
relies on (1) declarative restrictions on adapter function code and (2) waiting
to compile adapter functions until [instantiation]-time, when both sides of an
import are known to the engine. In contrast, core WebAssembly code is not
declarative and is often compiled/cached before instantiation-time. This
conflict would result in unpredictable and irregular performance and force
engines to make unnecessary heuristic tradeoffs. Additionaly, the semantic
layering helps keep core wasm simple.



[core spec]: https://webassembly.github.io/spec/core
[Module Syntax]: https://webassembly.github.io/spec/core/syntax/modules.html#syntax-module
[Old Module Type]: https://webassembly.github.io/spec/core/valid/modules.html#valid-module
[New Module Type]: https://github.com/WebAssembly/module-types/blob/master/proposals/module-types/Overview.md
[Instructions]: https://webassembly.github.io/spec/core/syntax/instructions.html
[Control Instructions]: https://webassembly.github.io/spec/core/syntax/instructions.html#syntax-instr-control
[Instantiation]: https://webassembly.github.io/spec/core/exec/modules.html#exec-instantiation
[Known Section]: https://webassembly.github.io/spec/core/binary/modules.html#sections
[Custom Section]: https://webassembly.github.io/spec/core/binary/modules.html#custom-section
[Value Types]: https://webassembly.github.io/spec/core/syntax/types.html#value-types
[Folded S-expressions]: https://webassembly.github.io/spec/core/text/instructions.html#folded-instructions

[Dynamic Linking]: https://webassembly.github.io/website/docs/dynamic-linking

[`memory.copy`]: https://github.com/WebAssembly/bulk-memory-operations/blob/master/proposals/bulk-memory-operations/Overview.md#memorycopy-instruction

[Reference Types]: https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md
[`anyref`]: https://webassembly.github.io/reference-types/core/syntax/types.html#syntax-reftype
[Function References]: https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md
[Type Imports]: https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md#imports
[Type Import]: https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md#imports
[GC]: https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md

[Multi-value]: https://github.com/WebAssembly/multi-value/blob/master/proposals/multi-value/Overview.md
[Block Validation]: https://webassembly.github.io/multi-value/core/valid/instructions.html#valid-block
[Select Validation]: https://webassembly.github.io/reference-types/core/valid/instructions.html#valid-select

[Multi-memory]: https://github.com/WebAssembly/multi-memory

[Exception Handling]: https://github.com/WebAssembly/exception-handling/blob/master/proposals/Exceptions.md

[Custom Annotation]: https://github.com/WebAssembly/annotations/blob/master/proposals/annotations/Overview.md

[JS API]: https://webassembly.github.io/spec/js-api/index.html

[`Reflect.construct`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect/construct
[`Reflect.apply`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect/apply
[`Reflect.set`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect/set
[`Reflect.get`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect/get

[Web IDL]: https://heycam.github.io/webidl
[Web IDL types]: https://heycam.github.io/webidl/#idl-types
[Primitive types]: https://heycam.github.io/webidl/#dfn-primitive-type
[ECMAScript Binding]: https://heycam.github.io/webidl/#ecmascript-binding
[Callback]: https://heycam.github.io/webidl/#idl-callback-function
[Dictionary]: https://heycam.github.io/webidl/#idl-dictionaries
[Sequence]: https://heycam.github.io/webidl/#idl-sequence
[Record]: https://heycam.github.io/webidl/#idl-record
[Enumeration]: https://heycam.github.io/webidl/#idl-enumeration
[Interface]: https://heycam.github.io/webidl/#idl-interfaces
[Union]: https://heycam.github.io/webidl/#idl-union
[Nullable]: https://heycam.github.io/webidl/#idl-nullable-type
[Annotated]: https://heycam.github.io/webidl/#idl-annotated-types
[Typed Array View]: https://heycam.github.io/webidl/#dfn-typed-array-type
[Frozen array]: https://heycam.github.io/webidl/#idl-frozen-array
[`BufferSource`]: https://heycam.github.io/webidl/#BufferSource
[`any`]: https://heycam.github.io/webidl/#idl-any
[`object`]: https://heycam.github.io/webidl/#idl-object
[`symbol`]: https://heycam.github.io/webidl/#idl-symbol
[`DOMString`]: https://heycam.github.io/webidl/#idl-DOMString
[`ByteString`]: https://heycam.github.io/webidl/#idl-ByteString
[`USVString`]: https://heycam.github.io/webidl/#idl-USVString
[`DOMString`-to-`USVString` conversion]: https://heycam.github.io/webidl/#dfn-obtain-unicode

[Unicode Code Points]: https://unicode.org/glossary/#code_point
[Unicode Scalar Values]: https://unicode.org/glossary/#unicode_scalar_value
[Unicode Encoding Scheme]: https://unicode.org/glossary/#encoding_scheme
[Code Units]: https://unicode.org/glossary/#code_unit
[Surrogate Code Points]: https://unicode.org/glossary/#surrogate_code_point
[Replacement Character]: https://unicode.org/glossary/#replacement_character
[WHATWG Encoding]: https://encoding.spec.whatwg.org
[WTF-8]: https://simonsapin.github.io/wtf-8/

[WASI]: https://github.com/webassembly/wasi
[`path_open`]: https://github.com/WebAssembly/WASI/blob/master/design/WASI-core.md#path_open

[Embind]: https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html
[wasm-bindgen]: https://github.com/rustwasm/wasm-bindgen

[native dynamic linking]: https://en.wikipedia.org/wiki/Dynamic_linker
[abstract data types]: https://en.wikipedia.org/wiki/Abstract_data_type
[eager evaluation]: https://en.wikipedia.org/wiki/Eager_evaluation
[lazily]: https://en.wikipedia.org/wiki/Lazy_evaluation
[thunks]: https://wiki.haskell.org/Thunk
[partially evaluated]: https://en.wikipedia.org/wiki/Partial_evaluation
