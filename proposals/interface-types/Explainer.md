# Interface Types Proposal

The proposal adds a new set of **interface types** to WebAssembly that describe
high-level values (like strings, sequences, records and variants) without
committing to a single memory representation or sharing scheme. Interface types
can only be used in the interfaces of modules and can only be produced or
consumed by declarative **interface adapters**.

The proposal is semantically layered on top of the WebAssembly [core spec],
adding only the ability to adapt the imports and exports of a WebAssembly
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
section) which defines how WebAssembly values can be converted to and
from Web IDL values, without going through JavaScript, even for high-level
types like `DOMString` and [Dictionary]. An optimizing engine can then compile
the declarative, statically-typed interface adapters into efficient
stubs that call more-directly into the API's implementation.

**Enabling "shared-nothing" linking of WebAssembly modules**

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
host-string values, under the control of the calling module.


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
1. [Lifting and lowering](#lifting-and-lowering)
1. [Strings in imports](#strings-in-imports)
1. [Shared-nothing linking example](#shared-nothing-linking-example)
1. [Strings with the GC proposal](#strings-with-the-GC-proposal)
1. [TODO](#TODO)

### Export returning string (statically allocated)

Let's start with a WebAssembly module that you can write today that returns a
string that is stored at a fixed location in linear memory:

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
completely different from the set of [core wasm instructions][Instructions],
the overall *structure* of the adapter module is the same as a core WebAssembly
function. In particular:
* The adapter function body is a (possibly nested) sequence of instructions.
* Instructions are defined to pop operands and push results onto the stack.
* Validation checks that the types of instructions' signatures line up.
* Instructions can either be written as a linear sequence (as shown above)
  or, equivalently, as [Folded S-expressions].

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
    ;; caller must call "free" when done the string
  )
)
```

This situation would naturally arise in, e.g., a C++ function that returns a
`std::string` or `std::unique_ptr<char[]>`.

What we need is to be able to call `free` right after the bytes are
read from `memory-to-string`. Thus, `memory-to-string` takes an
optional exported function name that it calls after it has read the string:

```wasm
  (@interface func (export "greeting") (result string)
    call "greeting_"
    memory-to-string "mem" "free"
  )
```

Note that the ability to call *any* function also allows reference counting
schemes to call a decrement function, e.g., if a C++ function returned a
`std::shared_ptr<std::string>`.

> **Note** Proper integration with exception handling will likely require
> switching to a different scheme that is exception safe. For example,
> responsibility to call the `free` function could be attached to a 
> [block instruction][Control Instructions] that called `free` when exited
> normally or unwound exceptionally.

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

### Lifting and Lowering

With both `memory-to-string` and `string-to-memory`, we can see a
pattern that will be repeated for all subsequent interface types: there is one
set of **lifting adapter instructions** that "lift" core wasm value types into
interface types and another set of **lowering adapter instructions** that
"lower" interface types into core wasm value types. Indeed, both can be
used in the same adapter function:

```wasm
  (func (export "frob") (param i32 i32) (result i32 i32)
  (@interface func (export "frob") (param $str string) (result string)
    arg.get $str
    string-to-memory "mem" "malloc"
    call-export "frob_"
    memory-to-string "mem" "free"
  )
```

The normal type validation rules ensure that lifting and lowering are used
appropriately and that, e.g., a `string` interface value isn't returned
directly to wasm.

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
in an export adapter functions; now `memory-to-string` is lifting the argument
of a `call-import` instead of lifting the return value of a `call-export`.
Note also that `memory-to-string` has no `free` immediate in this example. While
it would be *possible* to do so, since the caller of the adapter is wasm code,
it's simpler and potentially more efficient to let the caller worry about when
to free the string.

Using `string` as the return value of an import is symmetric to above,
using the previously-introduced `string-to-memory` lowering instruction.

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

### Shared-nothing linking example

With this 2×2 matrix of {lifting,lowering}×{import,export} covered, we can
now see a complete example of how one wasm module can [shared-nothing link](#Motivation)
to another wasm module. In this example, we start with one module providing a
`set` function which takes a `string` key/value pair:

```wasm
(module
  (func (export "set_") (param i32 i32 i32 i32) ...)
  ...
  (@interface func (export "set") (param $key string) (param $val string)
    arg.get $key
    string-to-memory "mem" "malloc"
    arg.get $val
    string-to-memory "mem" "malloc"
    call-export "set_"
  )
)
```

This module can be imported and used by a client module:

```wasm
(module
  (func (import "" "set_") (param i32 i32 i32 i32))
  ...
  (@interface func $set (import "kv-store" "set") (param string string))
  (@interface implement (import "" "set_") (param $keyPtr i32) (param $keyLen i32)
                                           (param $valPtr i32) (param $valLen i32)
    arg.get $keyPtr
    arg.get $keyLen
    memory-to-string "mem"
    arg.get $valPtr
    arg.get $valLen
    memory-to-string "mem"
    call-import $set
  )
)
```

If these adapter functions were compiled naively along with their containing
module, then calling `set` would require temporary, probably garbage-collected,
allocations for each `string` value. However, if the engine waits to compile
adapter functions until the module is [instantiated][Instantiation]—so that it
can see the adapter functions on both sides of an import call and match lifting
with lowering instructions—then passing a string can be implemented without
the intermediate allocation and a direct copy between linear memories.

### Strings with the GC proposal

In preceding [shared-nothing example](#shared-nothing-linking-example),
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
  (func (export "set_") (param (ref $Str)) (param (ref $Str)) ...)
  ...
  (@interface func (export "set") (param $key string) (param $val string)
    arg.get $key
    string-to-gc
    arg.get $val
    string-to-gc
    call-export "set_"
  )
)
```

This example also shows how having explicit adapter instructions allows new
representation choices to be added over time. Moreover, the provider module is
not required to make an all-or-nothing choice; individual parameters can use
whichever available representation is best.

### TODO

This rough list of topics is still to be added in subsequent PRs:
* integers and bool
* records
* sequences
* variants
* closures and interaction with [function references]
* re-importing/exporting core module import/exports
* using core value types in adapter function signatures
* importing an interface types with [type imports]
* adapter functions can contain zero or >1 calls


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



[core spec]: https://webassembly.github.io/spec/core
[Module Syntax]: https://webassembly.github.io/spec/core/syntax/modules.html#syntax-module
[Old Module Type]: https://webassembly.github.io/spec/core/valid/modules.html#valid-module
[New Module Type]: https://github.com/WebAssembly/meetings/blob/master/2019/CG-08-06.md#discuss-new-proposal-that-introduces-types-for-modules-and-instances-and-text-format-thereof-as-initially-discussed-in-design1289
[Instructions]: https://webassembly.github.io/spec/core/syntax/instructions.html
[Control Instructions]: https://webassembly.github.io/spec/core/syntax/instructions.html#syntax-instr-control
[Instantiation]: https://webassembly.github.io/spec/core/exec/modules.html#exec-instantiation
[Known Section]: https://webassembly.github.io/spec/core/binary/modules.html#sections
[Custom Section]: https://webassembly.github.io/spec/core/binary/modules.html#custom-section
[Value Types]: https://webassembly.github.io/spec/core/syntax/types.html#value-types
[Folded S-expressions]: https://webassembly.github.io/spec/core/text/instructions.html#folded-instructions

[Dynamic Linking]: https://webassembly.github.io/website/docs/dynamic-linking

[Reference Types]: https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md
[`anyref`]: https://webassembly.github.io/reference-types/core/syntax/types.html#syntax-reftype
[Function References]: https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md
[Type Imports]: https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md#imports
[Type Import]: https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md#imports
[GC]: https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md

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
