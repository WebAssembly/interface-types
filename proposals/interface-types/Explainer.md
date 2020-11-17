# Interface Types Proposal

The proposal adds a new set of **interface types** to WebAssembly that describe
high-level values. The proposal is semantically layered on top of the
WebAssembly [core spec] and can be implemented in terms of an unmodified core
wasm engine.

This proposal assumes the [multi-value] and [module linking] proposals as well
as the [`let`] instruction of the [function references] proposal.

1. [Motivation](#motivation)
2. [Additional Requirements](#additional-requirements)
3. [Proposal](#proposal)
   1. [Interface Types](#interface-types)
      1. [Interface Values Are Computed Lazily](#interface-values-are-computed-lazily)
      2. [Interface Values Have Destructors](#interface-values-have-destructors)
      3. [Interface Values Are Consumed At Most Once](#interface-values-are-consumed-at-most-once)
      4. [Interface Values Only Flow Forward](#interface-values-only-flow-forward)
   2. [Adapters](#adapters)
      1. [Adapter Functions](#adapter-functions)
      2. [Adapter Modules](#adapter-modules)
   3. [Lifting and Lowering Instructions](#lifting-and-lowering-instructions)
      1. [Integers](#lifting-and-lowering-integers)
      2. [Characters](#lifting-and-lowering-characters)
      3. [Lists](#lifting-and-lowering-lists)
      4. [Records](#lifting-and-lowering-records)
      5. [Variants](#lifting-and-lowering-variants)
4. [An End-to-End Example](#an-end-to-end-example)
5. [Adapter Fusion](#adapter-fusion)
6. [Use Cases Revisited](#use-cases-revisited)
7. [FAQ](#FAQ)
8. [TODO](#TODO)


## Motivation

As a compiler target, WebAssembly provides only low-level types that aim to be
as close to the underlying machine as possible, allowing each source language
to implement its own high-level types efficiently in terms of the low-level
types. However, when modules from multiple languages need to communicate with
each other or the host, there remains the question of how to exchange
high-level values.
 
To help motivate the proposed solution, we consider 4 use cases. After the
proposal is introduced, the use cases are [revisited below](#use-cases-revisited).

### Defining Language-Neutral Interfaces Like WASI

While many [WASI] signatures can be expressed in terms of `i32`s and,
in the future, references to [type imports], there is still a need
for WASI functions to take and return compound value types like strings or
lists. Currently, these values are specified in terms of linear memory. For
example, in [`path_open`], a string is passed as a pair of `i32`s which
represent the offset and length, respectively, of the string in linear memory.

However, specifying a single, fixed representation of data types like strings
will become problematic when:
* with the [GC] proposal, the caller wants to pass a `ref array u8`;
* the host has an opaque native string type which WASI would ideally accept
  without copy into and out of wasm linear or GC memory;
* more than one string encoding is supported.

Another problem with passing `i32` offsets into linear memory is that the
calling convention implicitly requires the callee to have access the caller's
memory. Thus, all WASI client modules currently must export their memory which
is then accessed by the WASI implementation in various ad hoc ways.

Ideally, WASI APIs would be expressed in terms of representation-agnostic
high-level value types and the caller wouldn't need to export their memory.

### Optimizing Calls to Web APIs

A long-standing source of friction between WebAssembly and the rest of the
Web platform (and the original motivating use case for this proposal) is that
calling Web APIs from WebAssembly usually requires thunking through JavaScript
which hurts performance and adds extra complexity for developers. Technically,
since they are exposed as JavaScript functions, Web IDL-defined functions can
be directly imported and called by WebAssembly through the [JS API]. However,
Web IDL functions usually take and produce high-level Web IDL types like
[`DOMString`] and [Sequence] while WebAssembly can only pass numbers. With the
newly-added [`externref`], WebAssembly can import JavaScript functions that
create JavaScript values that can be passed to the Web IDL functions via the
[ECMAScript Binding], but the performance of this approach may be even worse
than the JIT-optimized glue code.

Ideally, Web API calls would be statically type checked and passed Web IDL
values created directly by wasm.

### Creating Maximally-Reusable Modules

To fully realize the potential of WebAssembly, it should be possible for a
WebAssembly module author to target an output profile that maximizes the number
of clients that can reuse their module (where a "client" can be another
WebAssembly module, a native language runtime that embeds WebAssembly or some
other kind of host system). As part of this maximum-reuse profile, the language
and toolchain that a module author uses should be kept an encapsulated
module implementation detail allowing module clients to independently choose
their own language and toolchain. This in turns allows for the emergence of a
single, unified WebAssembly ecosystem of maximally-reusable modules. In
contrast, the path of least resistance, and thus the default outcome if no
effort is made to the contrary, is for the WebAssembly ecosystem to be
fragmented along the lines of language and toolchain.

It's important to scope this use case to avoid aiming for automatic seamless
integration between arbitrary languages, as this has generally been shown to be
an intractable problem. Instead, we can observe the general success of
**shared-nothing** architectures in allowing unrelated languages to
interoperate. Popular examples include Unix pipes connecting separate processes
and HTTP APIs connecting separate (micro)services. A shared-nothing
architecture partitions a whole application into multiple isolated units that
encapsulate their mutable state; shared mutable state is either banned or
significantly restricted. When multiple languages are used, then, it's natural
to put separate languages into separate isolated units. In raw WebAssembly
terms, a natural shared-nothing unit is a module which only imports and exports
functions and immutable values, but not memories, tables or mutable globals.

While existing shared-nothing architectures tend to incur communication
overhead due to system calls, context switches and extra copying, WebAssembly
can leverage its lightweight sandbox and use synchronous function calls to
avoid these sources of overhead (leaving asynchronous communication to be
provided by the host via API). Thus, instead of pipe or channel APIs,
shared-nothing WebAssembly modules can use function imports to directly call
across shared-nothing boundaries. In OS terms, this is analogous to the
[synchronous IPC] used for performance in some microkernels. However, this
approach has an obvious challenge in WebAssembly today: without a shared linear
memory, how can a function call pass values larger than the fixed-size core
wasm types (`i32`, `i64`, etc)?

One possible solution would be to leverage the upcoming [GC] proposal to pass
immutable GC objects across shared-nothing boundaries. However, this would add
an unnecessary dependency on GC when the clients and producers did not
otherwise require GC. Moreover, such an approach would require needless extra
copying and garbage when both sides used linear memory or mutable GC memory.
Lastly, while the GC proposal has types like `struct` and `array` that are
higher-level than linear memory, the GC proposal still seeks to be as close to
an assembly language as safety, portability and performance allow. Thus,
language and toolchain details are still likely to bubble up into a module's
public interface. Ideally, shared-nothing modules would be able to define their
function imports and exports using expressive high-level value types without
requiring GC or incurring extra copies or garbage.


## Additional Requirements

To properly address the above motivating use cases, there are some additional
requirements which impact the set of available solutions:
* A solution should allow a wide variety of value representations, avoiding
  the need for intermediate (de)serialization steps. Ideally, this would
  include even lazily-generated data structures (e.g., via iterators,
  generators or comprehensions).
* A solution should not force the exclusive use of linear or [GC] memory.
* A solution should allow efficient conversion to and from opaque host values
  that exist outside of any wasm memory when the host values are compatible.
* A solution should not require O(n)-sized engine-internal allocations of
  temporary values.
* A solution should not have subtle performance cliffs or depend on too
  much compiler magic.
* A solution should allow efficient and robust backwards-compatible evolution
  of API signatures.
* The proposal should still allow for the use of "shared-everything" linking
  (analogous to [native dynamic linking]) as a low-level mechanism for
  factoring common library code out of shared-nothing modules.


## Proposal

The proposal introduces first the new types, then the new modules and
functions that can contain the new types, then lastly the new instructions that
can be used in the new functions for producing and consuming the new types.

### Interface Types

To complement core wasm's set of *low-level* value types (`i32`, `i64`, ...),
this proposal defines a new set of *high-level* value types called **interface
types**. In the core spec, core value types are formally defined by the
[`valtype`] grammar. For interface types, the analogous set is `intertype`,
defined by the following grammar:
```
intertype ::= f32 | f64
            | s8 | u8 | s16 | u16 | s32 | u32 | s64 | u64
            | char
            | (list <intertype>)
            | (record (field <name> <id>? <intertype>)*)
            | (variant (case <name> <id>? <intertype>?)*)
```
`f32` and `f64` are the same types that appear in `valtype` and thus `valtype`
and `intertype` intersect at these two types. `char` is defined to be a
[Unicode scalar value]  (i.e., a non-[surrogate]  [code point]).

Unlike core wasm's `valtype`, which, starting with [function references],
allows cyclic type definitions, `intertype` type definitions are required to be
acyclic. This restriction follows from the fact that interface types are meant
to be used at inter-language boundaries to copy values and matches the
restrictions of other IPC/RPC schema languages.

Additionally, while core wasm's `valtype` contains reference values which refer
to mutable state, `intertype` compound values are transitively immutable. One
important consequence of this distinction is that, while `valtype` reference
subtyping is non-coerceive and constrained by low-level memory layout
considerations, `intertype` subtyping can be highly coercive, thereby allowing
flexible, backwards-compatible evolution of interface-typed APIs. Since copying
is inherent in the use of interface types, coercions can be folded into the
copy. A summary of the allowed coercions is given by the following table:

| Type             | Coercion allowed |
| ---------------- | ---------------- |
| `f32`, `f64`     | `f32` to `f64`   |
| `s8`, `u8`, `s16`, `u16`, `s32`, `u32`, `s64`, `u64` | if the source range is included in destination range |
| `char`           | none |
| `list`           | if the list's element type can be coerced |
| `record`         | if the record's fields' types can be coerced, matching fields by name regardless of order and allowing source fields to be ignored |
| `variant`        | if the variant's cases' types can be coerced, matching cases by name regardless of order and allowing destination cases to be ignored |

While these types are sufficiently general to capture a number of
more-specialized types that often arise in interfaces, it can still be
beneficial to have the specialized types explicitly represented in API
signatures. For example, this allows specialized source-language types to be
used in automatically-generated source-level API declarations. Thus, the
proposal contains several text-format and binary-format [abbreviations]. In the
notation below, the types on the left-hand side of the `≡` are expanded during
parsing/decoding into the abstract syntax of the right-hand side. Thus,
validation and execution are purely defined in terms of the general type; the
abbreviations only exist in the concrete formats.
```
                                      string ≡ (list char)
                        (tuple <intertype>*) ≡ (record ("𝒊" <intertype>)*) for 𝒊=0,1,...
                             (flags <name>*) ≡ (record (field <name> bool)*)
                                        bool ≡ (variant (case "false") (case "true"))
                              (enum <name>*) ≡ (variant (case <name>)*)
                        (option <intertype>) ≡ (variant (case "none") (case "some" <intertype>))
                        (union <intertype>*) ≡ (variant (case "𝒊" <intertype>)*) for 𝒊=0,1,...
(expected <intertype>? (error <intertype>)?) ≡ (variant (case "ok" <intertype>?) (case "error" <intertype>?))
```
Thus far, interface types are fairly normal and what you might expect to see in
a functional language or protocol schema. However, in support of the particular
use cases and requirements of this proposal, interface types have some atypical
properties described next.

#### Interface Values Are Computed Lazily

One of the additional requirements listed [above](#additional-requirements) is
to avoid intermediate O(n) copies (in time or memory use) of interface values.
This requirement may seem to be at odds with the above introduction of a new
set of immutable `intertype` values since a standard [eager][Eager Evaluation]
interpretation of `intertype` would imply the need to make a temporary copy at
the point where the `intertype` value was produced. While one could hope that
the copy could be optimized away by a sufficiently-smart compiler, such
optimizations would almost never apply in practice due to the typical
interleaving of arbitrary core wasm execution between producing and consuming
instructions.

To reliably satisfy the performance requirements, `intertype` values are
instead given a [lazy][Lazy Evaluation] interpretation. Thus, `intertype`
values have the form `(instruction operands*)` where `instruction` is one of
the producer instructions introduced [below](#lifting-and-lowering-instructions)
and `operands*` is a list of core wasm values. When an `intertype` value is
consumed by one of the consumer instructions (also introduced [below](#lifting-and-lowering-instructions)),
only *then* is the `instruction` applied to its `operands*`. Thus, with no
possible intervening side effects, a compiler will never have to create an
intermediate O(n) copy.

As an example, a lazy `string` value might have the form:
```wasm
(list.lift string $liftChar (i32.const 100) (i32.const 20))
```
where [`list.lift`](#lifting-and-lowering-lists) is an instruction that
produces a list given a set of immediates and operands that will be
explained below. When this lazy `string` value is consumed (e.g., by
[`list.lower`](#lifting-and-lowering-lists)), only *then* will the
`list.lift` instruction be executed to produce the abstract sequence of
characters.

Laziness is described in more detail below in conjunction with individual
lifting and lowering instructions.

#### Interface Values Have Destructors

While lazy evaluation solves the problem of intermediate copies, it creates a
new problem when the lazy value refers to dynamic allocations that must be
released after being read. The solution is to give all lazy values an
additional, optional "destructor" function that is called when the lazy value
is popped (either by being explicitly consumed or implicitly popped as part
of a control flow instruction). Thus, the form of a lazy value becomes
`(instruction operands* destructor?)`. When `destructor` is called, it is also
passed `operands*`, allowing the `operands*` to serve as the destructor's
"closure state". From a C++ or Rust perspective, interface values are like
locally-scoped objects with destructors.

Destructors are described in more detail [below](#lifting-and-lowering-instructions)
in conjunction with individual lifting and lowering instructions.

#### Interface Values Are Consumed At Most Once

Given these laziness and ownership properties, interface types are restricted
to be [affine], meaning that their values can't be duplicated or consumed
multiple times. This restriction solves two problems:
1. Copying a lazy interface value should not make an O(n) copy, but having
   two lazy values referring to the same underlying dynamic allocation will
   lead to double-free when two destructors are called.
2. The instructions executed lazily for interface values can have arbitrary
   side effects and thus the ability to consume an interface value multiple
   times can lead to surprising behaviors on both the producer and consumer
   ends, reducing overall ecosystem robustness.

Affine typing is described in more detail more [below](#adapter-functions)
in conjunction with adapter functions.

#### Interface Values Only Flow Forward

Interface types have one last restriction, above and beyond [affinity](#interface-values-are-consumed-at-most-once):
interface types cannot be used as the `param` of a loop block signature. This
restriction ensures that interface values only flow "forward" which allows an
implementation to easily snapshot the `operands*` of an interface value into
function locals, instead of needing to pass around the `operands*` as a
first-class tuple.

The use of this property is discussed more [below](#adapter-fusion) in the
context of a simple compilation scheme for erasing laziness at compile-time.


### Adapters

This proposal is layered on top of the core wasm spec and thus interface types
can't be used directly in core modules. Instead, the proposal defines a new
kind of module that *is* able to use interface types and may also *nest* core
modules. This new kind of module is called an **adapter module** and it
contains **adapter functions** that are allowed to use interface types and
instructions. Adapter modules embed core modules using the concepts introduced
by the [module linking] proposal.

#### Adapter Functions

Adapter functions are structured the same as core functions, with three main
differences:
* Adapter functions are a different kind of definition than core functions so:
   * In the text format, adapter functions start with `adapter_func` instead of
     `func`.
   * In the binary format, adapter functions go into a new adapter function
     section and populate a new adapter function index space.
* In place of core wasm's [`valtype`], adapter functions allow `adaptertype`.
* In place of core wasm's [`instr`], adapter functions allow `adapterinstr`.

where `adaptertype` and `adapterinstr` are defined as:
```
adaptertype ::= valtype | intertype
adapterinstr ::= instr | ... new instructions introduced below
```
Thus, adapter function are allowed to contain a mix of both core and interface
types and instructions and can be considered a *superset* of core functions.

The validation rules for core functions are also extended to validate the new
interface types and instructions. An important constraint that must be
preserved by all rules is the affinity of interface types mentioned [above](#interface-values-are-consumed-at-most-once).
This is achieved by the following validation rules:
* Interface types are not allowed in locals (function- or `let`-scoped), since
  locals can be read multiple times.
* Function parameters are changed from being implicit locals to describing
  the initial contents of the operand stack, just like block parameters.
  Consequently, `(param)` declarations in functions lose their identifier.

Other than affine typing restrictions, interface types may be used with the
[parametric instructions] and [control instructions]. For example, the
following adapter function uses interface types in conjunction with the core
`if`, `return` and `drop`.
```wasm
(adapter_func $return_one_of (param string string i32) (result string)
  i32.eqz
  (if (param string string) (result string)
    (then return)
    (else drop return))
)
```

##### `rotate`

Since locals are used not just for duplicating values, but also for reordering
the stack, adapter functions introduce a new `rotate` instruction for moving
a value to the top of the stack:
```
rotate $n : [ Tn+1 ... T0 ] -> [ Tn ... T0 Tn+1 ]
```
`rotate` is a pure restriction over what can be achieved with `let`, but could
be a valuable addition to core wasm due to its significantly smaller encoding,
reduced validation cost and improved static use-def information (for a
first-tier-compiler that doesn't build [SSA]). (See also [design/#1381](https://github.com/WebAssembly/design/issues/1381).)

##### `call_adapter`

Since adapter functions occupy a distinct index space from core functions,
they cannot be called or indirectly referenced with `call`, `call_indirect`,
`ref.func`, `call_ref`, etc. Instead, a new `call_adapter` instruction is
added for calling adapter functions:
```
call_adapter $f : [ P* ] -> [ R* ]
where:
 - $f : (adapter_func (param P*) (result R*))
 - the index of $f is less than the index of the containing adapter function
```
The core instructions for indirectly calling functions are *not* replicated for
adapter functions. The reason for only allowing direct, non-recursive adapter
calls is so that, at instantiation time, the wasm engine can have a complete,
precise callgraph of all adapter function calls, allowing the engine to
predictably compile lazy evaluation into eager evaluation at instantiation-time.
This is discussed more [below](#adapter-fusion).

#### Adapter Modules

Like adapter functions, adapter modules have the same structure as core modules
in both their text and binary format. The main differences are:
* Adapter modules are a different kind of definition than core  modules so:
  * In the text format, adapter modules start with `adapter_module` instead of
    `module`.
  * In the binary format, an adapter module [preamble] sets a bit indicating that
    this is an adapter module (backwards-compatibly reframing the 32-bit `version`
    field as two 16-bit `version` and `kind` fields).
* Adapter modules may not contain `func`, `memory`, `table`, `global`, `elem`
  or `data` definitions. Only `type`, `import`, `export`, `module`, `instance`
  and `alias` definitions are allowed.
* Adapter modules may additionally contain `adapter_func`, `adapter_module`
  and `adapter_instance` definitions.

For example, a trivial example of an adapter module is:
```wasm
(adapter_module
  (import "duplicate" (adapter_func $dup (param string) (result string string)))
  (import "print" (adapter_func $print (param string)))
  (adapter_func (export "print_twice") (param string)
    call_adapter $dup
    call_adapter $print
    call_adapter $print
  )
)
```
While this example shows that it's possible to write a pure adapter module,
the main point of an adapter module is to *adapt* the imports or exports of a
*core* module. To do this, adapter modules use and extend the concepts added by
the [module linking] proposal to *import* and *nest* core modules and then
*locally instantiate* these core modules, passing in adapter functions as
imports and calling exports from adapter functions.

For example, the following adapter module adapts a core module export (with the
`...` to be filled in with the new instructions introduced
[below](#lifting-and-lowering-instructions)):
```wasm
(adapter_module
  (module $CORE
    (func $transform (export "transform") (param i32 i32) (result i32 i32)
      ;; core code
    )
    (memory $memory (export "memory") 1)
  )
  (instance $core (instantiate $CORE))
  (adapter_func (export "transform") (param (list u8)) (result (list u8))
    ...
    call $core.$transform
    ...
  )
)
```
Imports can also be adapted, although doing so requires a bit more plumbing due
to the acyclic nature of instantiation:
```wasm
(adapter_module
  (import "print" (adapter_func $print (param string)))
  (adapter_module $ADAPTER
    (import "print" (adapter_func $originalPrint (param string)))
    (adapter_func $print (export "print") (param i32 i32)
      ...
      call_adapter $originalPrint
    )
  )
  (adapter_instance $adapter (instantiate $ADAPTER (adapter_func $print)))
  (module $CORE
    (import "print" (func $print (param i32 i32) (result i32 i32)))
    (func $run (export "run")
      ;; core code
    )
  )
  (instance $core (instantiate $CORE (adapter_func $adapter.$print)))
  (adapter_func (export "run")
    call $core.$run
  )
)
```
Here, the inner `$ADAPTER` module is instantiated first so that its export
(`$adapter.$print`) can be used to instantiate the inner `$CORE` module. Notice
that the core `instantiate` instruction has been extended to additionally
accept adapter functions.

Just as modules have a *module type*, adapter modules have an *adapter-module
type*. For example, the type of the preceding adapter module is:
```wasm
(adapter_module
  (import "print" (adapter_func (param string)))
  (export "run" (adapter_func))
)
```
Similarly, just as module instances have an *instance type*, adapter modules'
instances have an *adapter-instance type*:
```wasm
(adapter_instance
  (export "run" (adapter_func))
)
```

In the above example, because the `run` adapter function does no actual
adaptation, the underlying `$core.$run` function could just as well have
been exported directly:
```wasm
(export "run" (func $core.$run))
```
According to [module linking], the dotted notation `$core.$run` is syntactic
sugar for an [alias definition]:
```wasm
(alias $core_run (func $core $run))
(export "run" (func $core_run))
```
With this change to the above adapter module, the adapter-module type would
become:
```wasm
(adapter_module
  (import "print" (adapter_func (param string)))
  (export "run" (func))
)
```
This example demonstrates how adapter-module types can contain a *mix* of both
core and adapter definitions. Similarly, core memories, tables and globals can
be imported and exported.


### Lifting and Lowering Instructions

Along with each new interface type, the proposal adds new instructions for
producing and consuming that interface type. These instructions fall into two
categories:
* **Lifting instructions** produce interface types, going from lower-level
  types to higher-level types.
* **Lowering instructions** consume interface types, going from higher-level
  types to lower-level types.

By having the conversion between `valtype` and `intertype` explicit and
programmable, this proposal allows a wide variety of low-level representations
to be lifted and lowered without any intermediate serialization step. This
includes even data structures that are lazily- or dynamically-generated from
constructs such as iterators, generators or comprehensions.

#### Lifting and Lowering Integers

The proposal includes a matrix of integral conversion instructions which allow
converting between the 2 sign-less core integer types and the 8
explicitly-signed interface integer types:
```
ct ::= i32 | i64
it ::= u8 | s8 | u16 | s16 | u32 | s32 | u64 | s64

<it>.lift_<ct> : [ ct ] -> [ it ]

<ct>.lower_<it> : [ it ] -> [ ct ]
where:
  - bitwidth(ct) >= bitwidth(it)
```
For lifting, if `bitwidth(ct)` is greater than `bitwidth(it)`, then only the
least-significant `bitwidth(it)` bits of the `ct` value are used. For lowering,
the bitwidth validation-time restriction ensures that out-of-range errors
cannot occur. In both cases, the signedness of `ct` is implied by the sign of
`it`.

Since there can be no dynamic allocation read lazily by integer lifting, there
is no [destructor](#interface-values-have-destructors) immediate.

As an example usage, the following adapter module converts the implicitly-signed
`i32` into the explicitly-unsigned `u32`:
```wasm
(adapter_module $ADAPTER
  (module $CORE
    (func $get_num (export "get_num") (result i32)
      (i32.const 0xffffffff)
    )
  )
  (instance $core (instantiate $CORE))
  (adapter_func (export "get_num") (result u32)
    (u32.lift_i32 (call $core.get_num))
  )
)
```
Were `$CORE.$get_num` to be called directly by the [JS API], the `i32` value
`0xffffffff` would be interpreted by [`ToJSValue`] as the signed value -1.
With the adapter module, however, `get_num` explicitly returns a `u32` and
thus `ToJSValue` would know to interpret the bits as 2<sup>32</sup>-1.


#### Lifting and Lowering Characters

A `char` is strictly defined to be a [Unicode Scalar Value]  (USV), which
effectively means a positive integer value in either the range [0, 0xD7FF] or
[0xE000, 0x10FFFF], inclusive. Thus, lifting and lowering simply map between
`i32` and `char`:
```
char.lift : [ i32 ] -> [ char ]
char.lower : [ char ] -> [ i32 ]
```
If the `i32` passed to `char.lift` is outside the USV range, the instruction
traps. Thus, uses of `char.lower` can statically assume that the produced `i32`
is a valid, in-range USV. Even though interface types are generally lazy, the
`char.lift` range check is performed eagerly.

Since there can be no dynamic allocations read lazily by `char.lift`, the
instruction has no [destructor](#interface-values-have-destructors) immediate.

While it may initially appear that these instructions are implicitly assuming
a 4-byte ([UTF-32]) encoding, that is not the case. Rather, `char.lift` is to
be used *after* a code point has been decoded from linear memory and
`char.lower` is to be used *before* encoding the code point into linear memory.
Thus the containing adapter function is in full control of the encoding used.

More interesting than how individual characters are passed between modules,
though, is how whole *strings* of characters are passed. Doing this requires
passing a `(list char)` and using the list lifting and lowering instructions
which are introduced next.

#### Lifting and Lowering Lists

Since lists contain interface-typed elements, list lifting and lowering
instructions must recursively define how to lift and lower their elements.
Lifting is performed by the `list.lift` instruction:
```
list.lift $List $done $liftElem $destructor? : [ T:<valtype>* ] -> [ $List ]
where:
  - $List refers to a (list E:<intertype>)
  - $done : [ T* ] -> [ done:i32, U* ]
  - $liftElem : [ U* ] -> [ E, T* ]
  - $destructor : [ T* ] -> [], if present
```
To produce a `$List`, `list.lift` will repeatedly perform the following:
* Call `$done`, to determine whether the iteration is complete.
* Call `$liftElem`, to lift the next element to append to the list.

The `T*` and `U*` tuples define the *loop-carried state* of the iteration
and can be used to hold indices, pointers or general data structure state.
The loop-carried state is passed as follows:
* The initial `T*` operands passed to `list.lift` are passed to the first call
  to `$done`.
* The `U*` results of `$done` are passed to `$liftElem` (if `$done`
  doesn't return a nonzero `done`).
* The `T*` results of `$liftElem` are fed into the next call to `$done`.

With this design, `list.lift` allows a wide variety of list representations to
be lifted directly without any preparatory serialization:
* a homogeneous array, with `$liftElem` incrementing an index or pointer;
* a linked list, with `$liftElem` following a pointer to the next node;
* a `string`, with an arbitrary variable-length encoding of its `char` elements;
* an ordered tree, with `$liftElem` pushing and popping a stack of parents;
* a list comprehension, with `$liftElem` calling into arbitrary user-defined code.

List lowering is somewhat simpler since `$lowerElem` is simply called once for
each element of the incoming list.
```
list.lower $List $lowerElem : [ T:<valtype>*, $List ] -> [ T* ]
where
  - $List refers to an (list E:<intertype>)
  - $lowerElem : [ E, T* ] -> [ T* ]
```
As with `list.lift`, `T*` describes the loop-carried state with the `T*` values
being threaded through each invocation of `$lowerElem`. By having `T*` in both
the inputs *and* outputs of `list.lower`, an adapter function can lower into a
wide variety of list implementations including those with up-front allocation
and incremental (re)allocation during lowering.

Since [interface values are computed lazily](#interface-values-are-computed-lazily),
`list.lift` is evaluated lazily. This means that, when `list.lift` is executed
in program order, it semantically produces a tuple containing the `list.lift`
instruction (with its immediates) and a copy of the `T*` operands. It is only
when a `list` is consumed by `list.lower` that `list.lift` will start calling
`$liftElem`. Thus, any linear memory allocations read by `$liftElem` must be
kept valid until `list.lower` completes. If needed, `list.lift`'s `$destructor`
immediate can be used to promptly release dynamic allocations once the `list`
is popped.

As an example usage, a `(list s32)` can be lifted from a dynamically-allocated
contiguous array as follows:
```wasm
(adapter_func $done (param i32 i32) (result i32 i32 i32)
  (let (local $ptr i32) (local $end i32)
    (return (i32.eq (local.get $ptr) (local.get $end))
            (local.get $ptr)
            (local.get $end)))
)
(adapter_func $liftElem (param i32 i32) (result s32 i32 i32)
  (let (local $ptr i32) (local $end i32)
    (return (s32.lift_i32 (i32.load (local.get $ptr)))
            (i32.add (local.get $ptr) (i32.const 4))
            (local.get $end)))
)
(adapter_func $free (param i32 i32)
  (let (local $ptr i32) (local $end)
    (call $libc.$free (local.get $ptr)))
)
(adapter_func $liftArray (param i32 i32) (result (list s32))
  (let (local $ptr i32) (local $end i32)
    (return (list.lift (list s32) $done $liftElem $free (local.get $ptr) (local.get $end))))
)
```
and lowered into a linked list of `i32`s as follows:
```wasm
(adapter_func $lowerElem (param s32 i32) (result i32)
  (let (param s32) (local $prevNext i32) (result i32)
    i32.lower_s32
    (call $core.$malloc (i32.const 8))
    (let (local $srcVal i32) (local $dst i32)
      (i32.store (local.get $dst) (local.get $srcVal))
      (i32.store (local.get $prevNext) (local.get $dst))
      (i32.add (local.get $dst) (i32.const 4))))
)
(adapter_func $lowerLinkedList (param (list s32)) (result i32)
  (call $core.$malloc (i32.const 4))
  (let (param (list s32)) (local $container i32)
    (list.lower (list s32) $lowerElem (local.get $container))
    (i32.store (i32.const 0))
    (return (local.get $container)))
)
```
If `$liftArray` is called in adapter module A to produce a `(list s32)` value
that is passed to `$lowerLinkedList` in adapter module B, the net result will
be to convert from A's array representation into B's linked-list representation
and then free A's memory.

##### Optimization: Element Count

While the above lifting and lowering instructions allow a wide variety of list
representations, this generality comes at the cost of performance. To see why,
consider what would happen if we wanted to instead lower the above `(list s32)`
into a contiguous array. Because we don't know the size up-front, we'll need
some sort of dynamic reallocation strategy. While the cost of a geometric
reallocation strategy is [amortized O(n)], in practical terms, it will still be
slower than if we could have simply allocated the required memory up-front.
However, requiring *all* lists to supply a length up-front would be problematic
for various list representations like generators, iterators and strings.

Thus, the following two instructions are added to give list producers the
*option* of supplying an up-front element count:
```
list.lift_count $List $liftElem $destructor? : [ T*, count:i32 ] -> [ $List ]
where:
  - $List refers to an (list E:<intertype>)
  - $liftElem : [ T* ] -> [ E, T* ]
  - $destructor : [ T* ] -> [], if present
```
Because `count` defines the number of iterations up-front, the `$done` function
immediate isn't necessary and the entire instruction gets much simpler. For
example, the previous `$liftArray` adapter function could be rewritten more
succinctly as:
```wasm
(adapter_func $liftElem (param i32) (result s32 i32)
  (let (local $ptr i32)
    (return (s32.lift_i32 (i32.load (local.get $ptr)))
            (i32.add (local.get $ptr) (i32.const 4))))
)
(adapter_func $liftArray (param i32 i32) (result (list s32))
  (let (local $ptr i32) (local $end i32)
    (return (list.lift (list s32) $liftElem
              (local.get $ptr)
              (i32.shr_u (i32.sub (local.get $end) (local.get $ptr)) (i32.const 2)))))
)
```
On the consumer end, `list` consumers cannot depend on `list.lift_count`. Rather,
consumers can query whether a count is present with `list.has_count`:
```
list.has_count : [ (list T) ] -> [ (list T), maybe_count:i32, condition:i32 ]
```
If `condition` is nonzero, then `maybe_count` contains the `count` supplied to
`list.lift_count`. Note that, by passing the `(list T)` through unmodified,
`list.has_count` does not "consume" the list from the perspective of the
[affine type rules](#interface-values-are-consumed-at-most-once).

Using `list.has_count`, a `(list s32)` can be more-efficiently lowered into a
contiguous array as follows:
```wasm
(adapter_func $lowerElem (param s32 i32) (result i32)
  (let (param s32) (local $dst i32)
    (i32.store (local.get $dst) (i32.lower_s32))
    (return (i32.add (local.get $dst) (i32.const 4))))
)
(adapter_func $lowerArray (param (list s32)) (result i32 i32)
  list.has_count
  if (param (list s32) i32)
    (let (param (list s32)) (local $count i32)
      (call $core.$malloc (i32.mul (local.get $count) (i32.const 4)))
      (let (param (list s32)) (local $dst i32)
        (list.lower (list s32) $lowerElem (local.get $dst))
        (return (local.get $dst) (local.get $count))))
  else
    drop  ;; pop maybe_count
    ...   ;; use a dynamic reallocation strategy with list.lower
  end
)
```
Here, the then-branch takes advantage of the `i32` `$count` produced by
`list.has_count` to pre-allocate the `$dst` array. Without a count, the
else-branch must use a dynamic reallocation strategy (which is elided here for
brevity, but shown in full in the [end-to-end example below](#an-end-to-end-example)).

##### Optimization: Canonical Representation

There is one remaining performance-critical case that needs to be addressed:
strings. Recalling [above](#interface-types), `string` is an abbreviation for
`(list char)` and thus, to lift or lower a `string`, adapter functions can use
a combination of `list.lift`+`char.lift` or `list.lower`+`char.lower`. This
handles the general case of differing producer/consumer string encodings quite
well with the net result being a direct string transcoding. However, in the
common case where the producer and consumer have the *same* representation, the
transcoding will be suboptimal because strings in most cases will not be able
to use `list.lift_count` (whose unit is number-of-USVs). Thus, strings will
need to use a dynamic reallocation strategy despite the fact that the
destination's allocation size could have been simply taken from the source's
allocation size.

To motivate the solution, let's first consider another desirable optimization:
when a `list` is passed to a host API (c.f. the [Web API](#optimizing-calls-to-web-apis)
and [WASI](#defining-language-neutral-interfaces-like-wasi) use cases), we'd like
to define a single, canonical representation of the `list` in linear memory to
which the host can align its own internal representation such that, when the
canonical representation was used, zero copying was necessary. In theory,
a sufficiently-smart compiler could pattern-match the `$liftElem` code of a
`list.lift` to detect when a zero-copy-compatible representation was used, but
such an optimization would be very brittle.

To achieve both these optimizations, one last set of optional instructions are
added:
```
list.lift_canon $List <memidx>? $destructor? : [ T*, offset:i32, byte_length:i32 ] -> [ $List ]
where:
  - $List is a (list E), and E is a scalar type  (float, integral or char)
  - $destructor : [ T* ] -> [], if present

list.is_canon : [ (list E) ] -> [ (list E), maybe_byte_length:i32, condition:i32 ]
  - $List is a (list E), and E is a scalar type  (float, integral or char)

list.lower_canon <memidx>? : [ offset:i32, (list E) ] -> []
```
These instructions work as follows:
* `list.lift_canon` produces a list from the given `<memidx>` (defaulting
  to memory 0) at the given `offset`, reading `byte_length` bytes, calling
  the given `$destructor` when the list is popped.
* `list.is_canon` queries whether the given `(list E)` was lifted by
  `list.lift_canon` and, if so, returns the original `byte_length`.
* `list.lower_canon` writes the given list into the given `<memidx>`
  (defaulting to memory 0) at the given `offset`.

An example usage of these instructions is given [below](#an-end-to-end-example).

As part of the specification of these instructions, a canonical representation
of `$List` is precisely specified. While in theory a canonical representation
could be defined for *all* kinds of lists, this raises interesting questions in
the case of compound element types (lists of lists, variants and records).
Thus, the proposal starts by conservatively only allowing the "scalar" types
(numbers and characters), reserving the option to allow the compound types at a
later time.

While the canonical representation of all the numeric types are obvious,
due to their fixed-power-of-2 sizes, `char` requires the proposal to choose
an arbitrary character encoding. To match the core wasm spec's [choice of UTF-8][core-wasm-utf8],
and the more general trend of ["UTF-8 Everywhere"][UTF8-Everywhere], this
proposal also chooses UTF-8. By choosing a canonical string encoding happy path
while providing a graceful fallback to efficient transcoding, Interface Types
provides a gentle pressure to eventually converge without performance cliffs in
the meantime.

It should be noted that, while many languages are permanently tied to
[potentially ill-formed UTF-16], a common optimization among these languages'
runtimes is to represent strings as a union of either a full two-byte
[WTF-16] code unit or a one-byte representation when the string doesn't
contain any code points outside the one-byte-representable range. While this
one-byte representation today is commonly [Latin-1], which is not
UTF-8-compatible, pure 7-bit [ASCII], which *is* UTF-8-compatible, could be
instead be used, allowing the use of `list.lift_canon` for all strings
containing only ASCII.

#### Lifting and Lowering Records

Like lists, records are compound types that contain interface-typed fields that
are recursively lifted and lowered. The signature of `record.lift` is:
```
record.lift $Record $liftFields $destructor? : [ T:<valtype>* ] -> [ $Record ]
where:
  - $Record refers to a (record (field <name> <id>? F:<intertype>)*)
  - $liftFields : [ T* ] -> [ F* ]
  - $destructor : [ T* ] -> [], if present
```
As with `list.lift`, the `T*` tuple captures the core wasm state passed into
`$liftFields` to lift the individual fields and `$destructor` when the lifted
record is popped. Record lowering is symmetric:
```
record.lower $Record $lowerFields : [ T:<valtype>*, $Record ] -> [ U:<valtype>* ]
where
  - $Record refers to a (record (field <name> <id>? F:<intertype>)*)
  - $lowerFields : [ T*, F* ] -> [ U* ]
```
Having separate `T*` inputs and `U*` outputs gives `record.lower` maximum
flexibility to allocate before or during `$lowerFields`.

As with `list`, `record` has a lazy interpretation and thus `record.lift` is
evaluated lazily. Thus, any linear memory allocations read by `$liftFields`
must be kept valid until `record.lower`, using `$destructor` to free the
allocation thereafter, if necessary.

As an example, given the following type definition of a record type:
```wasm
(type $Coord (record (field "x" s32) (field "y" s32)))
```
a `$Coord` value can be lifted from a C `struct` containing two `i32`s in
linear memory by the following adapter functions:
```wasm
(adapter_func $liftFields (param i32) (result s32 s32)
  (let (local $ptr i32) (result s32 s32)
    (s32.lift_i32 (i32.load (local.get $ptr)))
    (s32.lift_i32 (i32.load offset=4 (local.get $ptr))))
)
(adapter_func $liftCoord (param i32) (result $Coord)
  record.lift $Coord $liftFields
)
```
and lowered into a different linear memory representation, for example, as two
`i64`s in reverse order:
```wasm
(adapter_func $lowerFields (param i32 s32 s32)
  rotate 2  ;; move the i32 to the top of the stack
  (let (param s32 s32) (local $ptr i32)
    (i64.store (local.get $ptr) (i64.lower_s32))
    (i64.store offset=8 (local.get $ptr) (i64.lower_s32)))
)
(adapter_func $lowerCoord (param i32 $Coord)
  record.lower $Coord $lowerFields
)
```
When passing the result of `$liftCoord` to `$lowerCoord`, the net effect will
be to perform a copy that swaps and sign-extends.

#### Lifting and Lowering Variants

Similar to the other compound types, variants recursively lift and lower their
cases' payloads' interface types. The signature of `variant.lift` is:
```
variant.lift $Variant $case $liftCase? $destructor? : [ T:<valtype>* ] -> [ $Variant ]
where
  - $Variant refers to a (variant (case <name> <id>? C:<intertype>?)*)
  - $liftCase : [ T* ] -> [ C[$case] ] iff C[$case] is present
  - $destructor : [ T* ] -> [], if present
```
By lifting only a single case, the responsibility for branching to select the
case is left up to the containing adapter function which can use core control
instructions like `br_if` and `br_table`.

In contrast, the `variant.lower` instruction takes control of the dynamic
branching itself by taking one lowering function per case:
```
variant.lower $Variant $lowerCase* : [ T:<valtype>*, $Variant ] -> [ U:<valtype>* ]
where
  - $Variant refers to a (variant (case <name> <id>? C:<intertype>?)*)
  - for each case: $lowerCase : [ T*, C? ] -> [ U* ]
```
Having separate `T*` inputs and `U*` outputs gives `variant.lower` maximum
flexibility to allocate before or during `$lowerCase`.

As with `list` and `record`, `variant` has a lazy interpretation and thus
`variant.lift` is evaluated lazily. Thus, any linear memory allocations read by
`$liftCase` must be kept valid until `variant.lower`, using `$destructor` to
optionally free the allocation when the variant value is popped.

As an example, given the following type definition of a variant type:
```wasm
(type $MaybeAge (variant (case "has_age" $hasAge u8) (case "no_age" $noAge)))
```
a `$MaybeAge` value can be lifted from a representation that is either null
or a pointer to an allocated object that must be freed after its contents
are read:
```wasm
(adapter_func $free (param i32)
  call $core.$free
)
(adapter_func $liftHasAge (param i32) (result u8)
  i32.load
  u8.lift_i32
)
(adapter_func $liftMaybeAge (param i32) (result $MaybeAge)
  (let (local $ptr i32)
    (if (result $MaybeAge) (i32.eqz (local.get $ptr))
      (then (variant.lift $MaybeAge $hasAge $liftHasAge $free (local.get $ptr)))
      (else (variant.lift $MaybeAge $noAge))))
)
```
and lowered into a packed-word encoding that uses a sentinel value of `-1` to
represent the `no_age` case:
```wasm
(adapter_func $lowerHasAge (param u8) (result i32)
  i32.lower_u8
)
(func $lowerNoAge (result i32)
  i32.const -1
)
(adapter_func $lowerMaybeAge (param $MaybeAge) (result i32)
  variant.lower $MaybeAge $lowerHasAge $lowerNoAge
)
```
When passing the result of `$liftMaybeAge` to `$lowerMaybeAge`, the net effect
will be to convert from the null-or-object encoding to the packed encoding and
then free the object in the non-null case.


## An End-to-End Example

Having introduced the building blocks, we can now look at a more-complete
example in which module `$A` exports a function returning a `(list u8)` that is
imported and called by module `$B`. This example also demonstrates the ability
to share `libc` *code* between `$A` and `$B` without sharing `libc` *state*.

`$A` is fairly straightforward, importing `libc` as module so that a private
`$libc` instance can be created and imported by the nested core module.
(This follows the general pattern of the [dynamic linking example][shared-everything-example].)
```wasm
(adapter_module $A
  (type $Libc (instance
    (export "memory" $memory (memory 1))
    (export "free" $free (func (param i32)))
  ))
  (import "libc" (module $LIBC (export $Libc)))
  (instance $libc (instantiate $LIBC))
  (alias (memory $libc $memory))  ;; make $libc.$memory the default memory

  (module $CORE_A
    (import "libc" (instance (type $Libc)))
    (func $getBytes (export "get_bytes") (result i32 i32)
      ;; core code, returning (ptr, length) of allocated std::vector
    )
  )
  (instance $core (instantiate $CORE_A (instance $libc)))

  (adapter_func $freeVector (param i32 i32)
    drop  ;; drop the byte length
    call $core.$free
  )
  (adapter_func $getBytes (export "get_bytes") (result (list u8))
    call $core.$getBytes
    list.lift_canon (list u8) $liftByte $freeVector
  )
)
```
Since `$core.$getBytes` returns ownership of the internal buffer of a
`std::vector`, a destructor (`$freeVector`) is used to free the buffer when the
`list` is popped. Without the destructor, `get_bytes` would have no way to know
when it could release the `std::vector` memory returned by `$core.$getBytes`.

`$B` is a bit more complicated due to list lowering having to handle both the
optimized and general case and import adapters requiring extra plumbing to work
around the lack of cyclic imports.
```wasm
(adapter_module $B
  (type $Libc (instance
    (export "memory" $memory (memory 1))
    (export "malloc" $malloc (func (param i32) (result i32)))
    (export "free" $free (func (param i32)))
    (export "realloc" $realloc (func (param i32) (param i32) (result i32)))
  ))
  (import "libc" (module $LIBC (export $Libc)))
  (instance $libc (instantiate $LIBC))

  (import "./A.wasm" (adapter_module $A
    (import "libc" (module (export $Libc)))
    (adapter_func (export "get_bytes") $getBytes (result (list u8)))
  ))
  (adapter_instance $a (instantiate $A (module $LIBC)))

  (adapter_module $ADAPTER
    (import "get_bytes" (adapter_func $originalGetBytes (result (list u8))))

    (import "libc" (instance $libc (type $Libc)))
    (alias (memory $libc $memory))  ;; make $libc.$memory the default memory

    (adapter_func $growingLowerByte (param u8 i32 i32 i32) (result i32 i32 i32)
      (let (param u8) (result i32 i32 i32)
        (local $dst i32) (local $length i32) (local $capacity i32)
        (if (i32.eq (local.get $length) (local.get $capacity))
          (then
            (local.set $capacity (i32.mul (local.get $capacity) (i32.const 2)))
            (local.set $dstPtr (call $libc.$realloc (local.get $dstPtr) (local.get $capacity)))))
        (i32.store8 (local.get $dstPtr) (i32.lower_u8))
        (i32.add (local.get $dstPtr) (i32.const 1))
        (i32.add (local.get $length) (i32.const 1))
        (local.get $cap))
    )
    (adapter_func $getBytes (export "get_bytes") (result i32 i32)
      (list.is_canon (call_adapter $originalGetBytes))
      if (param (list u8) i32) (result i32 i32)
        (let (result i32 i32) (local $byteLength i32)
          (call $libc.$malloc (local.get $byteLength))
          (let (result i32 i32) (local $dstPtr i32)
            (list.lower_canon (list u8) (local.get $dstPtr))
            local.get $dstPtr
            local.get $byteLength))
      else
        drop  ;; pop maybe_count
        (list.lower (list u8) $growingLowerByte
          (call $core.$malloc (i32.const 8)) (i32.const 0) (i32.const 8))
        drop  ;; pop capacity, leave [ptr, length] pushed
      end
    )
  )
  (adapter_instance $adapter (instantiate $ADAPTER (instance $libc) (adapter_function $a.$getBytes)))

  (module $CORE_B
    (import "libc" (instance (type $Libc)))
    (import "get_bytes" (func (result i32 i32)))
    ;; core code
  )
  (instance $core (instantiate $CORE_B (instance $libc) (adapter_func $adapter.$getBytes)))
)
```
Another thing that this example illustrates is the complementary use of
"shared-everything" and "shared-nothing" linking described in the 
[Additional Requirements section](#additional-requirements). In particular,
`$LIBC` is a shared-everything library that factors out the common code from
the two shared-nothing modules `$A` and `$B`. Note that only the stateless
code of `$LIBC` is shared; `$A` and `$B` get their own private `$LIBC`
instances. Thus, from all external appearances, `$A` and `$B` produce
shared-nothing instances despite using shared-everything linking as an
implementation detail.


## Adapter Fusion

While the lazy evaluation scheme for lifting instructions and interface values
decribed above avoids intermediate O(n) copies, we might worry that it comes at
the cost of extra runtime implementation complexity and performance overhead.
For example, to represent its lazy values, [Haskell uses thunks] and a runtime
implementation oriented around thunks.

However, due to the combination of:
* [affine typing restrictions](#interface-values-are-consumed-at-most-once),
  which ensure that interface values are consumed at most once,
* [loop-parameter restrictions](#interface-values-only-flow-forward), which
  ensure that interface values only flow forward, and
* [adapter calling restrictions](#adapter-functions), which ensure that
  all adapter function calls are direct and non-recursive,

the implementation of adapter function laziness can be greatly simplified:
at instantiation time, when imports are known, and thus the complete
adapter function callgraph is known, it is possible to compile all adapter
functions and interface types down to *core functions* and *core types*. With
this compilation scheme, called "adapter fusion", both runtime complexity and
performance overhead are avoided.

A high-level sketch of the adapter fusion algorithm is as follows. This
process is described in terms of a single adapter function (henceforth called
the "root") that has been imported by a core instance (via `adapter_function`
operand to `instantiate`). Thus, this algorithm is expected to be performed
repeatedly, once per imported adapter function.
1. Adapter functions are recursively inlined into the root until no
   `call_adapter` instructions remain.
2. For each operand of each lifting instruction, synthesize a new `local`
   of the operand's type in the root. (Note: lifting instructions only
   have core operand types.)
3. Give each use of a lifting instruction a unique integer id.
4. Rewrite each lifting instruction in-place into:
   1. for each operand, a `local.set` into the `local` synthesized by step 2.
   2. an `i32.const` that pushes the unique integer id assigned by step 3.
5. Rewrite all interface types to `i32` (which, due to step 4, is now the
   core type of all lifting instructions).
6. Compute the [Reaching Definitions] for all lowering instructions. (Due to
   validation, lowering instructions will be transitively reached by zero or
   more type-compatible lifting instructions.)
7. Rewrite each lowering instruction in-place into a `br_table` that switches
   on all reaching `i32` lifting instruction identifiers (pushed by step 4.2 and
   discovered by step 6), with one case per lifting instruction containing:
   1. for each operand of the lifting instruction's function immediate (i.e.,
      `$liftElem`/`$liftFields`/`$liftCase`), a `local.get` from the local
      synthesized by step 2 and initialized by step 4.1.
   2. an inline fusion of the lifting instruction's function immediate
      and the lowering instruction's function immediate produced by recursively
      invoking this algorithm on the composition of the two.
8. At each point where an interface value is popped (lowering instructions,
   `drop` instructions and control flow instructions), emit a `br_table` that
   switches on all reaching `i32` lifting instruction identifiers, where each
   case body contains a call to the corresponding lifting instruction
   `$destructor`, passing the closure state captured by step 4.2.

Thus, the algorithm erases almost all the dynamism of laziness by aggressively
inlining. Even the one remaining source of dynamism (the `br_table`s) will
often be optimized away when a lowering instruction is reached by a single
lifting instruction (as is the case in the example below). For normal code,
this compilation strategy would risk significant code bloat, but the role of
adapter code is to be a thin layer between core modules. If code bloat does
become a problem, there is a spectrum of less-aggressively-specializing
compilation strategies available. In the limit, no inlining need be performed;
lazy values can be boxed into tuples holding function references.

As a demonstration, the adapter modules `$A` and `$B` shown in the 
[last section](#an-end-to-end-example) would be fused into the `$fused_root`
function shown below. The containing module would also be created according to
similarly-automatic fusion rules. Note that the nested modules `$CORE_A` and
`$CORE_B` are identical to those originally nested in `$A` and `$B`; the fusion
algorithm only operates on *adapter modules* and leaves core modules untouched.
```wasm
(module
  (type $Libc (instance
    (export "memory" $memory (memory 1))
    (export "malloc" $malloc (func (param i32) (result i32)))
    (export "free" $free (func (param i32)))
    (export "realloc" $realloc (func (param i32) (result i32)))
  ))
  (import "libc" (module $LIBC (export $Libc)))
  (instance $libc_a (instantiate $LIBC))
  (module $CORE_A
    (import "libc" (instance (type $Libc)))
    (func (export "get_bytes") (result i32 i32)
      ;; core code
    )
  )
  (instance $core_a (instantiate $CORE_A (instance $libc_a))
  (module $GLUE
    (import "" (func $core_a_get_bytes (result i32 i32)))
    (import "libc_a" (instance $libc_a (type $Libc)))
    (import "libc_b" (instance $libc_b (type $Libc)))

    (func $fused_root (export "get_bytes") (result i32 i32)
      (local $list_lift_ptr i32)  ;; added by step 2 for list.lift_canon
      (local $list_lift_len i32)  ;; added by step 2 for list.lift_canon

      ;; $A.$getBytes:
      call $core_a_get_bytes
      local.set $list_lift_len    ;; rewritten from list.lift_canon by step 4.1
      local.set $list_lift_ptr    ;; rewritten from list.lift_canon by step 4.1
      ;; i32.const added by step 4.2 eliminated as dead code

      ;; $B.$getBytes_:
      ;; list.is_canon is a constant expression because only one reaching lift
      local.get $list_lift_len
      ;; `if` now has a constant expression, so the `if` and `else` are removed
      let (result i32 i32) (local $byteLength )
        (call $libc_b.$malloc (local.get $byteLength))
        let (result i32 i32) (local $dstPtr i32)

          ;; emitted by step 7.2 for matching list.lift_canon/lower_canon
          (memory.copy $libc_a.$memory $libc_b.$memory  ;; using multi-memory proposal
            (local.get $dstPtr)
            (local.get $list_lift_ptr)
            (local.get $byteLength))

          ;; emitted by step 8: inline $A.$freeVector
          (call $libc_a.$free (local.get $list_lift_ptr))

          ;; end of $B.$getBytes then-branch
          local.get $dstPtr
          local.get $list_lift_len
        end
      end
    )
  )
  (instance $libc_b (instantiate $LIBC))
  (instance $glue (instantiate $GLUE (instance $libc_a) (instance $libc_b) (func $core_a.$getBytes)))
  (module $CORE_B
    (import "libc" (instance (type $Libc)))
    (import "get_bytes" (func (result i32 i32)))
    ;; core code
  )
  (instance $core_b (instantiate $CORE_B (instance $libc_b) (func $glue.$fused_root)))
)
```
As demonstrated by this example, fusion is able to produce highly optimized core
wasm code when both modules use the "canonical" representation of lists.
However, even non-canonical lists will produce a single fused loop that
simultaneously iterates over the source and destination, allowing direct copy
without any intermediate buffer. In the case of nested lists, the nested lifting
and lowering loops will also be fused so that there is never an intermediate
copy.

In summary, adapter fusion converts multiple adapter modules which each
respectively encapsulate their own memory into a single fused core module that
contains all the memories (using [multi-memory]) and fused functions that
directly copy between memories. Thus, fusion can serve as an implementation
technique for Interface Types, wrapping the existing core wasm spec's
[`module_instantiate`] procedure with a new `adapter_instantiate` procedure that
performs adapter fusion and then feeds the resulting core module into
`module_instantiate`. Moreover, if imports are known at build time (e.g., by
Webpack), then fusion can occur at build time, ultimately requiring no native
wasm engine support.


## Use Cases Revisited

We now consider how this proposal can be used to address the use cases given
[above](#motivation).

### Defining language-neutral interfaces like WASI (revisited)

With this proposal, a WASI interface can be defined purely in terms of an
[adapter-module type](#adapter-modules). From this type, written as a
`.wit`/[`.witx`] file, each source-language's toolchain can automatically
generate source-language declarations and adapter functions.

For example, the signature of [`fd_pwrite`] could be defined as:
```wasm
(adapter_instance
  (type $FD (export "fd"))
  (type $Errno (enum "acces" "badf" "busy" ...))
  (adapter_func $fd_pwrite (export "fd_pwrite")
    (param $fd (ref $FD))
    (param $iovs (list u8))
    (param $offset u64)
    (result (expected u32 $Errno)))
)
```
and from this `.wit` type, different source-language declarations and adapter
functions could be generated. For example, a C++ declaration generator might
emit something like:
```C++
template <class T> class handle { int table_index_; ... };
enum class errno { acces, badf, busy, ... };

std::expected<uint32_t, errno>
fd_pwrite(handle<FD> fd,
          const std::vector<uint8_t>& iovs,
          uint64_t offset);
```
Based on these signatures, the generated adapter function would be:
```wasm
(adapter_func $lowerOk (param u32) (result i32 i32)
  i32.const 0
  i32.lower_u32
)
(adapter_func $lowerError (param $Errno) (result i32 i32)
  i32.const 1
  call_adapter $lowerErrno  ;; $lowerErrno omitted for brevity
)
(adapter_func $fd_pwrite_adapter (param i32 i32 i64) (result i32 i32)
  (let (result i32 i32) (local $fd i32) (local $iovs i32) (local $offset i64)
    (call_adapter $fd_pwrite
      (table.get $fds (local.get $fd))
      (list.lift_canon $liftByte (i32.load (local.get $iovs)) (i32.load offset=4 (local.get $iovs)))
      (u64.lift_i64 (local.get $offset)))
    variant.lower (expected u32 $Errno) $lowerOk $lowerError)
)
```
where we assume `std::vector`'s first two words are the pointer and length and
`std::expected` is represented as a boolean `i32` and a payload `i32` that can
be returned via [multi-value]. Calls in C++ to `fd_pwrite` are thus compiled
into calls to the imported `$fd_pwrite_adapter`. At instantiation-time, the
wasm engine can compile `$fd_pwrite_adapter` into an efficient trampoline
which converts directly from the C++ linear memory representation of
`fd_pwrite`'s arguments into the callee's representation.

### Optimizing calls to Web APIs (revisited)

Using interface types, a wasm [adapter module](#adapter-modules) can directly
call Web APIs, passing compatible high-level values. The process starts when an
adapter module is instantiated by either the [JS API] or [ESM-integration]. In
either case, the [*instantiate a WebAssembly module*] spec routine is invoked,
passing in a set of imported JS values. This routine specifies how the incoming
JS values are coerced to match the module's declared import types. With this
proposal, these rules would naturally be extended to consider interface-typed
imports:
* if the imported JS value is a built-in function created by Web IDL
  (e.g., for an [operation], [getter] or [setter]) *and* its Web IDL signature
  "matches" the interface-typed import signature (in the sense described below),
  then an [adapter function](#adapter-functions) is synthesized from the two
  signatures that converts between interface values and Web IDL values;
* otherwise, the interface values are converted to JS values (as described
  [below](#embedding-webassembly-in-language-runtimes-revisited)) and the
  Web IDL function is called as a JS function through the [ECMAScript Binding].

An important requirement for these two coercion paths is that there should be
little to no observable difference between the two paths, allowing the former
path to be considered an optimization of the latter path. Ensuring 100%
semantic equivalence may not be possible due to JS semantic corner cases (e.g.,
NaN canonicalization of `unrestricted double` or prototype-chain walking for
missing `dictionary` properties). However, in non-pathological scenarios, this
equivalence must be maintained to ensure that JS virtualization of Web APIs
still works effectively (so that a JS function can polyfill, fix, attenuate,
censor, virtualize, etc. a Web IDL function).

Given that, the following Web IDL/interface types would match:
* [`any`] matches `externref`
* [`void`] matches the empty function result type
* [numeric types] match numeric interface types
* [`USVString`] matches `string` (these types are isomorphic)
* [`DOMString`] matches `string` (using the lossy [`DOMString`-to-`USVString`] conversion)
* [`ByteString`] matches `string` (using the lossy [`TextDecoder.decode`] conversion)
* [Dictionary] matches `record`
* [Sequence] and [Frozen array] match `list`
* [`boolean`] matches the `bool` variant abbreviation
* [Enumeration] types match the `enum` variant abbreviation
* [Nullable] types match the `option` variant abbreviation
* [Union] types match the `union` variant abbreviation
* [Interface] types match [TODO](#TODO)
* [`ArrayBufferView`] types match [TODO](#TODO)
* [Callback] types match [`funcref`]
* [`object`], [`symbol`], [`Promise`] match `externref`
* [`ArrayBuffer`] matches `externref` (rarely used outside of [`BufferSource`])
* [Record] matches `externref` (rarely used; not actually a `record`)
* [Annotated] types recursively match the [inner type]  (the annotations add callee-side runtime checks)

As shown by this list, `externref` provides an escape hatch for
less-frequently-used or JS/Web-specific types. When `externref` is used, an
adapter module can import helper functions for producing and consuming
`externref`s. In the future, with [type imports], the `externref`s could be
replaced by `(ref $Import)`s which could eliminate dynamic type checks in
adapter functions at instantiation-time.

### Creating Maximally-Reusable Modules (revisited)

To create a maximally-reusable module, a developer would produce an adapter
module that exclusively uses interface types in its signature to create a
shared-nothing interface. The value semantics of interface types provide the
client language significant flexibility in how to coerce to and from its
native language values.

For example, the [JS API] would be extended to provide the following coercions
(in [`ToJSValue`] and [`ToWebAssemblyValue`]):
* Numeric types other than `s64` and `u64` convert to and from JS number values.
* `s64` and `u64` types convert to and from JS BigInt values.
* `string` converts to and from JS string values (using the [`DOMString`-to-`USVString`] 
  conversion for the "from" direction).
* If the JS [records and tuples] proposal progresses, interface `record` and
  `list` types could convert to and from these new value types.
* The `tuple` record abbreviation could also produce a JS tuple value.
* The `bool` variant abbreviation converts to and from JS boolean values.
* The `enum` variant abbreviation converts to and from JS string values based on
  the `enum`'s case labels.
* The `option` variant abbreviation converts the JS `null` or `undefined` to
  the `none` case and any other JS value to the `some` case.
* The `result` variant abbreviation, when used as the sole result of a function
  call would produce an exception for the `error` case, and return the payload
  of the `ok` case.
* The `union` variant abbreviation when converted to JS would convert the
  payload directly. Converting from a JS value to a `union` is highly ambiguous
  but could perhaps use the same ad hoc resolution scheme Web IDL's
  [`union` ECMASCript Binding][ES Union].
* The general `variant` interface type does not have a canonical mapping to JS.
  TypeScript initially supported [tagged unions] using an object with a special
  `kind` property as the discriminant, but this feature was later generalized
  to [User-Defined Type Guards]. For `variant`, coercing to and from an object
  of the form `{kind:'name', value:<payload>}` could potentially work.

Note that the same coercions can be applied both when WebAssembly is embedded
in a native JavaScript engine and when the client is JavaScript running as
WebAssembly (on a host that does not include a JavaScript engine). In a similar
manner, other high-level languages can define their own coercion semantics
between interface values.


## FAQ

### How does Interface Types interact with ESM-integration?

This proposal has no direct impact on [ESM-integration] and adapter modules
should Just Work when imported via `import` or `<script type='module'>`. The
reason is that ESM-integration is responsible for selecting which wasm module
to compile and which JS import values to supply to the [*instantiate a WebAssembly module*]
procedure whereas this proposal changes what happens *during* these respective
steps.

With [ESM-integration] and Interface Types, WebAssembly is one step closer to
enabling WebAssembly modules to have full access to the Web platform without
the need for JS "glue code". The remaining capability needed to achieve this
goal is something like the [get-originals] proposal which reflects JS and Web
IDL APIs as built-in modules that can be directly imported.


## TODO

* "outparams" to capture the `read(buffer)` / typed-array-view use cases ([#68](https://github.com/WebAssembly/interface-types/issues/68))
* lifting and lowering between interface types and opaque reference types, allowing zero-copy when used on both sides
* types for describing handles to resources that address resource lifetime (see also [#87](https://github.com/WebAssembly/interface-types/issues/87))
* optional/default-valued record fields or function parameters
* how precisely calls to host functions work
* ability to import/export interface *values* (as opposed to adapter functions), thereby allowing adapter modules to import configuration data and [JSON modules]
* address what happens when a trap, exception or event unwinds into an adapter function (e.g., lockdown semantics)


[Core Spec]: https://webassembly.github.io/spec/core
[JS API]: https://webassembly.github.io/spec/js-api/index.html
[`ToJSValue`]: https://webassembly.github.io/spec/js-api/index.html#tojsvalue
[`ToWebAssemblyValue`]: https://webassembly.github.io/spec/js-api/index.html#towebassemblyvalue
[*instantiate a WebAssembly module*]: https://webassembly.github.io/spec/js-api/index.html#instantiate-a-webassembly-module
[Web API]: https://webassembly.github.io/spec/web-api/index.html
[C API]: https://github.com/WebAssembly/wasm-c-api
[Abbreviations]: https://webassembly.github.io/reference-types/core/text/conventions.html#abbreviations
[`name`]: https://webassembly.github.io/spec/core/text/values.html#names
[`valtype`]: https://webassembly.github.io/spec/core/syntax/types.html#syntax-valtype
[`instr`]: https://webassembly.github.io/spec/core/syntax/instructions.html#syntax-instr
[`hostfunc`]: https://webassembly.github.io/spec/core/exec/runtime.html#syntax-hostfunc
[`module_instantiate`]: https://webassembly.github.io/spec/core/appendix/embedding.html#mathrm-module-instantiate-xref-exec-runtime-syntax-store-mathit-store-xref-syntax-modules-syntax-module-mathit-module-xref-exec-runtime-syntax-externval-mathit-externval-ast-xref-exec-runtime-syntax-store-mathit-store-xref-exec-runtime-syntax-moduleinst-mathit-moduleinst-xref-appendix-embedding-embed-error-mathit-error
[Parametric Instructions]: https://webassembly.github.io/spec/core/syntax/instructions.html#parametric-instructions
[Control Instructions]: https://webassembly.github.io/spec/core/syntax/instructions.html#control-instructions
[Preamble]: https://webassembly.github.io/spec/core/binary/modules.html#binary-version
[core-wasm-utf8]: https://webassembly.github.io/spec/core/binary/values.html#binary-utf8

[`externref`]: https://webassembly.github.io/reference-types/core/syntax/types.html#syntax-reftype
[`funcref`]: https://webassembly.github.io/reference-types/core/syntax/types.html#syntax-reftype

[Function References]: https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md
[`let`]: https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md#local-bindings

[Type Imports]: https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md#imports

[Multi-value]: https://github.com/webassembly/multi-value
[Multi-memory]: https://github.com/webassembly/multi-memory

[Module Linking]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Explainer.md
[Alias Definition]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Explainer.md#instance-imports-and-aliases
[Shared-everything Dynamic Linking]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Explainer.md#shared-everything-dynamic-linking
[Shared-everything-example]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Example-SharedEverythingDynamicLinking.md

[GC]: https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md

[WASI]: https://github.com/webassembly/wasi
[`path_open`]: https://github.com/WebAssembly/WASI/blob/master/phases/snapshot/docs.md#-path_openfd-fd-dirflags-lookupflags-path-string-oflags-oflags-fs_rights_base-rights-fs_rights_inherting-rights-fdflags-fdflags---errno-fd
[`fd_pwrite`]: https://github.com/WebAssembly/WASI/blob/master/phases/snapshot/docs.md#-fd_pwritefd-fd-iovs-ciovec_array-offset-filesize---errno-size
[`.witx`]: https://github.com/WebAssembly/WASI/blob/master/docs/witx.md

[ESM-integration]: https://github.com/WebAssembly/esm-integration/tree/master/proposals/esm-integration

[get-originals]: https://github.com/domenic/get-originals/

[Records and Tuples]: https://github.com/tc39/proposal-record-tuple
[JSON Modules]: https://github.com/tc39/proposal-json-modules

[Operation]: https://heycam.github.io/webidl/#dfn-create-operation-function
[Setter]: https://heycam.github.io/webidl/#dfn-attribute-setter
[Getter]: https://heycam.github.io/webidl/#dfn-attribute-getter
[ECMAScript Binding]: https://heycam.github.io/webidl/#ecmascript-binding
[Numeric Types]: https://heycam.github.io/webidl/#dfn-numeric-type
[Dictionary]: https://heycam.github.io/webidl/#idl-dictionaries
[Callback]: https://heycam.github.io/webidl/#idl-callback-function
[Sequence]: https://heycam.github.io/webidl/#idl-sequence
[Record]: https://heycam.github.io/webidl/#idl-record
[Enumeration]: https://heycam.github.io/webidl/#idl-enumeration
[Interface]: https://heycam.github.io/webidl/#idl-interfaces
[Union]: https://heycam.github.io/webidl/#idl-union
[Nullable]: https://heycam.github.io/webidl/#idl-nullable-type
[Annotated]: https://heycam.github.io/webidl/#idl-annotated-types
[Inner Type]: https://heycam.github.io/webidl/#annotated-types-inner-type
[Typed Array View]: https://heycam.github.io/webidl/#dfn-typed-array-type
[Frozen array]: https://heycam.github.io/webidl/#idl-frozen-array
[ES Union]: https://heycam.github.io/webidl/#es-union
[`boolean`]: https://heycam.github.io/webidl/#idl-boolean
[`ArrayBufferView`]: https://heycam.github.io/webidl/#ArrayBufferView
[`ArrayBuffer`]: https://heycam.github.io/webidl/#idl-ArrayBuffer
[`BufferSource`]: https://heycam.github.io/webidl/#BufferSource
[`any`]: https://heycam.github.io/webidl/#idl-any
[`void`]: https://heycam.github.io/webidl/#idl-void
[`object`]: https://heycam.github.io/webidl/#idl-object
[`symbol`]: https://heycam.github.io/webidl/#idl-symbol
[`Promise`]: https://heycam.github.io/webidl/#idl-promise
[`DOMString`]: https://heycam.github.io/webidl/#idl-DOMString
[`ByteString`]: https://heycam.github.io/webidl/#idl-ByteString
[`USVString`]: https://heycam.github.io/webidl/#idl-USVString
[`DOMString`-to-`USVString`]: https://infra.spec.whatwg.org/#javascript-string-convert

[`TextDecoder.decode`]: https://encoding.spec.whatwg.org/#dom-textdecoder-decode
[Unicode Scalar Value]: https://unicode.org/glossary/#unicode_scalar_value
[Code Point]: https://unicode.org/glossary/#code_point
[Surrogate]: https://unicode.org/glossary/#surrogate_code_point
[UTF8-Everywhere]: http://utf8everywhere.org/
[Potentially ill-formed UTF-16]: http://simonsapin.github.io/wtf-8/#ill-formed-utf-16
[WTF-16]: http://simonsapin.github.io/wtf-8/#wtf-16

[Tagged Unions]: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-0.html#tagged-union-types 
[User-Defined Type Guards]: https://www.typescriptlang.org/docs/handbook/advanced-types.html#user-defined-type-guards
[Haskell uses thunks]: https://en.wikibooks.org/wiki/Haskell/Laziness
[Native Dynamic Linking]: https://en.wikipedia.org/wiki/Dynamic_linker
[Affine]: https://en.wikipedia.org/wiki/Substructural_type_system#Affine_type_systems
[Eager Evaluation]: https://en.wikipedia.org/wiki/Eager_evaluation
[Lazy Evaluation]: https://en.wikipedia.org/wiki/Lazy_evaluation
[SSA]: https://en.wikipedia.org/wiki/Static_single_assignment_form
[Reaching Definitions]: https://en.wikipedia.org/wiki/Reaching_definition
[UTF-8]: https://en.wikipedia.org/wiki/UTF-8
[UTF-32]: https://en.wikipedia.org/wiki/UTF-32
[Latin-1]: https://en.wikipedia.org/wiki/ISO/IEC_8859-1
[ASCII]: https://en.wikipedia.org/wiki/ASCII
[Amortized O(n)]: https://en.wikipedia.org/wiki/Dynamic_array#Geometric_expansion_and_amortized_cost
[Synchronous IPC]: https://en.wikipedia.org/wiki/Microkernel#Inter-process_communication
