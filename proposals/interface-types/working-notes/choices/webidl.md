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
>across ownership boundaries.

The enriched type system that is defined by the Interface Types proposal has
some of the hallmarks of an IDL (Interface Definition Language). However, there
remain several questions, which this note aims to address:

* What are the design considerations that lie behind the specification of the
  Interface Types language of types?
* Why not WebIDL?
* Are we intending to support interoperability between different programming
  languages?
* Given the importance of the web platform, what is the relationship between the
  Interface Types and WebIDL?
  
### Supporting Interoperability Across Ownership Boundaries

Interoperating across ownership boundaries has different expectations than when
within a shared domain. In particular, there more limited expectations for
sharing, architecture, and trust.

The term 'ownership boundary' bears further explanation. Informally, we can say
that we are crossing an ownership boundary when, for example, my code is calling
your code and I want to limit the extent that I have to trust you.

By restricting the intended scope to interoperability across ownership
boundaries the design of Interface Types can sidestep the issues that cause
stumbling blocks for the general inter-language interopability.

### Interoperability via Function Calls

Interoperability between different systems is arguably a requirement of all
modern applications. However, one of the differentiating aspects of
interoperability as it applies to WebAssembly is the role of the function
call. Unlike interoperability expressed at the level of web services (say),
using a foreign capability in WebAssembly is most naturally expressed as a
function call.

I.e., invoking capabilities does not naturally imply sending messages,
serializing data, network connections, or even crossing a process boundary: the
invoked capability may even be present in the same thread.

This view of interoperability informs much of the style of the Interface Types
proposal. In particular, this proposal does not include a serialization
format.[^Although there is a binary encoding, its role is to support the
publishing and loading of WebAssembly modules; not to support function
interaction.] Indeed, it is expected that no serialization is required when
invoking a function via Interface Types' adapters.

### Language Specific Module Systems

Interface Types are _not_ intended to replacement language specific module
systems. Nor are Interface Types intended to address the general inter-language
interoperability problem.

Indivual languages will often have their own techniques aimed at allowing
importing and exporting of modules. For example, C/C++ modules would continue to
be linked together into single WebAssembly modules.

## WebIDL

[Web IDL][WebIDL] is clearly an important IDL for Web APIs. And, since the web
platform is a crucial domain for WebAssembly, it seems necessary to address the
question of "why not WebIDL" for describing APIs in Interface Types.

The case against WebIDL is comprised of three principal arguments:

* WebIDL is too strongly aligned to JavaScript;
* WebIDL has features that are not relevant in the context of WebAssembly; and
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

Certain aspects of WebIDLs design seem to be somewhat elaborated. For
example, there are two variants of floating point numbers: float and
unrestricted floats. (With corresponding equivalents for double precision
floating point numbers.) The difference between a `float` and an `unrestricted
float` in WebIDL is one that is not common in programming languages.

Similarly,  WebIDL  has   three  variants  of  string   type:  the  `DOMString`,
`ByteString` and `USVString` string. Again, the distinction between `DOMString`s
and `USVString`s is not one that is common in programming languages.

### Extended Attributes

Heavy use is made in WebIDL of so-called extended attributes. For example, the
`AllowShared` extended attribute is intended to mark a buffer type to be backed
by a `SharedArrayBuffer`.

Apart from the obvious dependence on a specific feature of JavaScript (shared
array buffers), allowing sharing on values contradicts one of the key design
criteria for Interface Types -- namely supporting interoperability in a cross
ownership domain where there is no sharing.

In practice, Web APIs that are specified using WebIDL make heavy use of extended
attributes to convey additional semantics that are very JavaScript centric. Such
usages would have to be prohibited when designing APIs for languages other than
JavaScript.

This again leads to a situation where the 'real' IDL that would be usable for
WebAssembly would be a limited form of a public IDL. This is not an ideal model
for standardization nor for supporting the wider WebAssembly ecosystem.

### Nominal Types

JavaScript does not have a static model of types. On the other hand, many
programming languages do have static types. One of the features of many static type systems is the so-called _nominal type_.

A nominal type differs from a structural type in several ways; but one of the
most fundamental ways is that a nominal type can be used to model entities that
are not directly conveyed by the actual data used to denote them.

For example, one may choose to model a chair using the nominal type `Chair`;
whose values contain a pair of integers: the SKU and price (for example). The
pair of integers is _not equivalent_ to a chair, but represent the information
needed to model chairs in some application or API. 

A different entity (a person say) may also be modeled by a type `Person` which
contains two integers (for example, an indentification number and an index into
a street address table).

A given pair of integers might be either a `Chair` or a `Person` (or a 2D point)
and there is no way of conveying the distinction unless the nominal type is also
taken into consideration.

### Algebraic Type Definitions

The Interface Types language permits the definition of a limited form of
_algebraic type definition_. This is, in part, to regularize concepts such as
nullability and to permit limited forms of polymorphism (allowing a color to be
named by a string, a vector of small integers or a large integer, for example).

WebIDL explicitly prohibits the formation of new types; indeed it uses type
unions to achieve similar polymorphism. However, as we note above, not all
programming languages support dynamic types or type unions; which makes it
problematic to support such polymorphism.

The form of algebraic types envisaged in the Interface Types system is
intentionally limited. For example, we do not permit _recursive_ types to be
defined. This is in keeping with the vision of supporting APIs across ownership
boundaries where complex data is likely to take limited form.[^This design
choice may be revisited in future versions of the Interface Types specification
if supporting structured types such as abstract XML and/or JSON become
important.]

### Working with WebIDL

While the Interface Types system is intended to support interoperability, it is
import to clarify how this relates to WebIDL; especially since access to Web
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

* map the various integers types (`byte`, `octet`, `long` etc.) from WebIDL to their closest analogues in Interface Types (`s8`, `u8`, `s32` etc.).
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
