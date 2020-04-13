# Interface Types and WebIDL

## Preamble
This note is part of a series that explain some of the major design choices in
the Interface Types specification.

## Introduction

A key problem addressed by Interface Types is the semantic gap between the
expressivitiy of core WebAssembly and the needs of API designers and application
authors. Part of this gap is represented by the difference between the type
system of core WebAssembly and the needs of a typical programmer.

As an example of this, core WebAssembly has no intrinsic type corresponding to
`string`; even though this is an extremely popular type that is critical for
many if not most applications.

Note that this is not an oversight. Neither the MVP of WebAssembly nor the
vision implied by the [GC proposal][GC] have `string` as an intrinsic type.

One of the goals of this proposal is to facilitate interoperability, in part by
defining a type system for WebAssembly that better meets the needs of API
designers.

In particular,

>Interface Types are intended to enable WebAssembly modules to interoperate
>in limited trust situations -- for example where the modules do not share
>memory -- sometimes called 'shared-nothing' module boundaries.

The enriched type system that is defined by the Interface Types proposal has
some of the hallmarks of an IDL (Interface Definition Language). However, it is
not the intention of the proposal to introduce a new IDL; and Interface Types
deliberately lacks many features that would be required for an IDL.

There remain several questions, which this note aims to address:

* Are we intending to support interoperability between different programming
  languages?
* Given the importance of the web platform, what is the relationship between the
  Interface Types and WebIDL?
  
## WebIDL

[Web IDL][] is clearly an important IDL for Web APIs. And, since the web
platform is a crucial domain for WebAssembly, it seems necessary to address the
question of "why not WebIDL?" for describing APIs in Interface Types.

Like WebIDL, Interface Types uses a model of function call rather than a
serialization format to enable interoperability across module
boundaries. However, Interface Types differ in some important areas:

* Interoperability is mediated through special adapter functions, rather than
  special annotations of APIs;
* APIs are strongly statically typed for the most part; reflecting the reality
  that many WebAssembly modules are not written in dynamically typed programming
  languages; and
* Interface Types are not intended to enable the communication of arbitrary
values; rather the scheme is purposefully limited in its range of types to those
that can be canonically encoded and decoded into and out of WebAssembly memory.

The case against WebIDL is comprised of three principal arguments:

* WebIDL contains a number of JavaScript specific details that would not belong
  in the language neutral context of WebAssembly;
* WebIDL represents design choices that are not relevant in the context of
  WebAssembly; and
* WebIDL does not support certain necessary features.

### JavaScript

WebAssembly can be characterized as a vehicle that allows languages _other than_
JavaScript to be executed on the web (specifically, in a browser tab).

On the other hand, WebIDL is designed to be an IDL for JavaScript. This results
in WebIDL having features that may be problematic for other languages.

For example, WebIDL has the concepts of `object` and _Indexed Properties_; where
`object` means the value is a JavaScript object and an _Indexed Property_ is one
which exposes an interface that allow indexed access.

Many languages do not support these concepts; and it may be quite onerous to do
so. However, any feature of an IDL should be useable by any API designer to
create an appropriate API. Features of an IDL that target JavaScript should not
be used by an API designer that wishes to design for many different languages.

There is a fine distinction to be made in making 'avoidance' recommendations and
constructing a completely new IDL.

#### WebIDL Strings and Numbers

As a higher-level IDL, WebIDL includes dynamic restrictions on parameters and
types that go beyond fundamental data types. For example, there are two variants
of floating point numbers: float and unrestricted floats. (With corresponding
equivalents for double precision floating point numbers.) The difference between
a `float` and an `unrestricted float` in WebIDL is one that is not common in
programming languages.

Similarly,  WebIDL  has   three  variants  of  string   type:  the  `DOMString`,
`ByteString` and `USVString` string. Again, the distinction between `DOMString`s
and `USVString`s is not one that is common in programming languages.

### Extended Attributes

Heavy use is made in WebIDL of so-called extended attributes. For example, the
`Unforgeable` extended attribute -- which is intended to mark an operation as
not being overwritable -- has no semantic equivalent in WebAssembly.

In practice, Web APIs that are specified using WebIDL make heavy use of extended
attributes to convey additional semantics that are very JavaScript centric. Such
usages would have to be prohibited when designing APIs for languages other than
JavaScript.

This again leads to a situation where the 'real' IDL that would be usable for
WebAssembly would be a limited form of a public IDL. This is not an ideal model
for standardization nor for supporting the wider WebAssembly ecosystem.

### Algebraic Type Definitions

The Interface Types language permits the definition of a limited form of
_algebraic types_ -- i.e., there is support for records (tuples) and variants
(sums). This is, in part, to regularize concepts such as nullability and to
permit limited forms of polymorphism (allowing a color to be named by a string,
a vector of small integers or a large integer, for example).

Although there are mechanisms within WebIDL for supporting variants, the form of
these are not consistent with sum types in general. In particular, variant types
in WebIDL are not associated with explicit discriminators.

Without there being a mandatory tag associated with each case in a variant it
becomes impossible to consistently and automatically discriminate between the
variants without the ability to inspect the data itself. This is straightforward
in JavaScript but unreliable in other languages.

On the other hand, modifying Web IDL to support such tags would be a necessarily
breaking change in the design of Web IDL -- and all the APIs that use variants.

The form of algebraic types envisaged in the Interface Types system is
intentionally limited. For example, we do not permit _recursive_ types to be
defined. This is in keeping with the vision of supporting APIs across ownership
boundaries where complex data is likely to take limited
form.[^This design choice may be revisited in future versions of the Interface Types specification if supporting structured types such as abstract XML and/or JSON become important.]

### Working with WebIDL

While the Interface Types system is intended to support interoperability, it is
important to clarify how this relates to WebIDL; especially since access to Web
APIs is a major use case.

There are two potential strategies for WebIDL: it would be possible to construct
an analogous binding to the [EcmaScript WebIDL Bindings] for JS. I.e., this
specification would show how different WebIDL values should be realized from
Interface Types values.

Alternately, WebIDL could be viewed as an alternate to core WebAssembly itself:
import and export adapters could be constructed that allow Interface Types to
work with WebIDL APIs. In this scenarion, the host embedder would be augmented
to be able to combine export adapters targeting WebIDL and import adapters used
by WebAssembly modules.

Typically, such adapters would be automatically generated for most of the APIs
an embedder offers.

This is a slight extension to the inter-WebAssembly shared-nothing linking approach.

As an example of the former approach, a WebAssembly Binding for Interface Types
would:

* map the various integers types (`byte`, `octet`, `long` etc.) from WebIDL to
  their closest analogues in Interface Types (`s8`, `u8`, `s32` etc.).
* map the various string types (`DOMString`, `USVString`) to `string`.

  > Note that this involves losing a distinction available within WebIDL.
* map the byte string type (`ByteString`) to an array of `s8` integers.

* map dictionary types to their analogs as Interface Types records

Note that this does not imply that the semantics of Interface Types and WebIDL
are completely aligned: any such binding specification would likely ignore those
aspects of WebIDL that are not accounted for in the Interface Types language.

#### Modifying WebIDL itself

One of the strategies that we considered was modifying WebIDL itself. After all,
as we note, the Interface Types language has obvious overlap with WebIDL.

However, for some of the same reasons outlined here, changing WebIDL is
considered to be infeasible: WebIDL has an orientation to JavaScript that would
be difficult to carry forward in such a refactoring.

## Summary

The intented purpose of Interface Types requires a language that allows API
designers and application authors to express their intentions in a way that is
language neutral. 

Although it is not our intention to support arbitrary language interoperability,
we do wish to support it for the limited scenarios where ownership boundaries
are involved.

We chose not to use WebIDL as the IDL for Interface Types because WebIDL is not
language agnostic and because it would be too disruptive of the Web community to
change WebIDL to fit the requirements.


[Explainer]: https://github.com/WebAssembly/interface-types/blob/explainer/proposals/interface-types/Explainer.md

[GC]: https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md

[Web IDL]: https://heycam.github.io/webidl

[EcmaScript WebIDL Bindings]: https://heycam.github.io/webidl/#ecmascript-binding
