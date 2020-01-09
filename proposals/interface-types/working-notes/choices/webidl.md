# Interface Types and WebIDL

## Preamble
This note is part of a series that explain some of the major design choices in
the Interface Types specification.

## Introduction

A key motivation for Interface Types is the semantic gap between the
expressivitiy of core webAssembly and the needs of API designers and high-level
programmers. Part of this gap is represented by the difference between the type
system of core webAssembly and the needs of a typical programmer.

As an example of this, core webAssembly has no intrinsic type corresponding to
`string`; even though this is arguably the most important type in many if not
most applications.[^The reasons for this lack are beyond the scope of this
note.] Note that neither the MVP of webAssembly nor the vision implied by the
[GC proposal][GC] have `string` as an intrinsic type.

A core part of this proposal is the design of an IDL (Interoperable Definition
Language); one goal of which is to address the semantic gap. However, in
designing an IDL there remain several questions, which this note aims to
address:

* What are the design considerations that lie behind the specification of the
  Interface Types IDL?
* Why design a new IDL, why not select one of the available IDLs?
* Why not WebIDL?
* Are we intending to support interoperability between different programming languages?
* Given the importance of the web platform, what is the relationship between the Interface Types IDL and WebIDL?

## Intended Scope of Interface Types

As noted in the [Explainer], there are three primary motivations for the
Interface Types proposal: optimizing calls to Web APIs, enabling various forms
of module linking and supporting the wider API ecosystem by enabling third party
APIs (such as WASI) to be expressed in an interoperable way.

These can be summarized as 

>Interface Types are intended to enable webAssembly modules to interoperate
>across ownership boundaries.

### Ownership Boundaries

The term 'ownership boundary' bears further explanation. Informally, we can say
that we are crossing an ownership boundary when, for example, my code is calling
your code and I want to limit the extent that I have to trust you.

More formally, a function call crosses an ownership boundary when it is
important that the only information communicated to the function involves data
that is explicitly passed as arguments to the call, and that there is no
possibility that the callee can side-effect the caller's state (by, for example,
overwriting areas of linear memory).

This limiting of trust is a vital enabler for the API ecosystem: if
functionalities can be published and consumed with limited trust it makes it
more likely that those same functionalities will be published and consumed.

It is a goal of the Interface Types proposal that this limited trust can be
supported by proving that an imported module cannot side-effect the importing
module (and vice versa). This proof taking the form of a simple validation step
performed during module instantiation.

### Interoperability via Function Calls

Interoperability between different systems is a hallmark of many if not most
modern applications. However, one of the differentiating aspects of
interoperability as it applies to webAssembly is the role of the function
call. Unlike interoperability expressed at the level of web services (say),
using a foreign capability in webAssembly is most naturally expressed as a
function call.

I.e., invoking capabilities does not naturally imply sending messages,
serializing data, network connections, or even crossing a process boundary: the
invoked capability may even be present in the same thread.

This context informs much of the style of how interoperability is expressed in
the Interface Types proposal. In particular, this proposal does not include a
serialization format.[^Although there is a binary encoding, its role is to
support the publishing and loading of webAssembly modules; not to support
function interaction.] Indeed, it is expected that no serialization is required
when invoking a function via Interface Types' adapters.

## WebIDL

[Web IDL][WebIDL] is clearly an important IDL for Web APIs. And, since the web
platform is a crucial domain for webAssembly, it seems necessary to address the
question of "why not WebIDL" for describing APIs in Interface Types.

The case against WebIDL is comprised of three principal arguments:

* WebIDL is too strongly aligned to JavaScript;
* WebIDL has features that are not relevant in the context of webAssembly; and
* WebIDL does not support certain necessary features.

### Javascript

WebAssembly can be characterized as a vehicle that allows languages _other than_
Javascript to be executed on the web (specifically, in a browser tab).

On the other hand, WebIDL is designed to be an IDL for Javascript. This results
in WebIDL having features that may be problematic for other languages.

For example, WebIDL has the concepts of `object` and _Indexed Properties_; where
`object` means the value is a Javascript object and an _Indexed Property_ is one
which exposes an interface that allow indexed access.

Many languages do not support these concepts; and it may be quite onerous to do
so. However, any feature of an IDL should be useable by any API designer to
create an appropriate API. Features of an IDL that target Javascript sould not
be used by an API designer that wishes to design for many different languages.

There is a fine distinction to be made in making 'avoidance' recommendations and
constructing a completely new IDL.

#### WebIDL Strings

Certain aspects of WebIDLs design seem to be somewhat overelaborated. For
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

Apart from the obvious dependence on a specific feature of Javascript (shared
array buffers), allowing sharing on values contradicts one of the key design
criteria for Interface Types -- namely supporting interoperability in a cross
ownership domain where there is no sharing.

In practice, Web APIs that are specified using WebIDL make heavy use of extended
attributes to convey additional semantics that are very Javascript centric. Such
usages would have to be prohibited when designing APIs for languages other than
Javascript.

### Nominal Types

Javascript does not have a static model of types. On the other hand, many
programming languages do have static types. One of the features of many static type systems is the so-called _nominal type_.

A nominal type differs from a structural type in several ways; but one of the
most fundamental ways is that a nominal type can be used to model entities that
are not directly conveyed by the actual data.

For example, one may choose to model a chair using the nominal type `Chair`;
whose values contain a pair of integers: the SKU and price (for example). The
pair of integers is _not equivalent_ to a chair, but represent the information
needed to model chairs in some application or API. 

A different entity (a person say) may also be modeled by a type `Person` which
contains two integers also (for example, the age of the person and an index into
a street address table).

A given pair of integers might be either a `Chair` or a `Person` (or a 2D point)
and there is no way of conveying the distinction unless the nominal type is also
taken into consideration.

### Algebraic Type Definitions





### Working with WebIDL




[Explainer]: https://github.com/WebAssembly/interface-types/blob/explainer/proposals/interface-types/Explainer.md

[GC]: https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md

[Web IDL]: https://heycam.github.io/webidl
