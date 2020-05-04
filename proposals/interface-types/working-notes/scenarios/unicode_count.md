# Counting Unicodes

## Preamble
This note is part of a series of notes that gives complete end-to-end examples
of using Interface Types to export and import functions.

## Introduction

Imagine that you have access to a module that can count the number of Unicode
characters in a string. For reasons of portability the type signature for this
function is not expressed in terms of memory slices but in terms of the
Interface Types' concepts of `string` and unsigned 32 bit integers:

```
countCodes:(string)=>u32
```

In this note we show how `countCodes` can be imported as an Interface Types
wrapped function and how we can export such a function so that it can be
imported.

In addition, we show the reasoning that follows when an export and an import are
combined; as would happen, for example, if the importing and exporting modules
were combined in a _shared-nothing static linking_ scenario.

## Importing a Unicode Counter

In order to access this function in a webAssembly module it must be declared in
a special `@interface` section:

```wasm
(@interface func (import "" "countCodes")
  (param string)
  (result u32))
```

This statement mirrors the normal import statement except that it uses types
from the Interface Types schema and involves a special `@interface` section to
distinguish it from regular webAssembly imports. 

The style of `@interface` sections follow the recommendations of the
[Custom Section] proposal. This `func` statement declares that there is an
Interface Type function -- `countCodes` -- that we are importing and that its
type is from strings to unsigned 32 bit integers.

Corresponding to this statement, there is also a statement that reflect that the
core webAssembly module also imports a corresponding `countCodes_` function:

```wasm
(func (import "" "countCodes_")
  (param i32 i32)
  (result i32))
```

Notice that we are using a slightly different name for `countCodes_` here; this
is not technically required but is helpful in distinguishing the different
concepts involved in using Interface Types.

In order to actually access the imported `countCodes_` function, we provide an
_import adapter_ This adapter function must map concepts that are native to core
webAssembly -- memory and integers -- to the more abstract concepts in Interface
Types. In effect, the import adapter _implements_ the core import -- by mapping
it to an import of `countCodes` as an Interface Types function.

The import adapter for `countCodes_` looks like a regular webAssembly function,
with some different instructions and it uses an `@interface` `implement` element
that highlights that this is not a regular webAssembly function:

```
(memory (export "memi" 1)

(func $count_ (import ("" "countCodes_"))
  (param i32 i32) (result i32))

(@interface func $count (import "" "countCodes")
  (param $str string) (result u32))
  
(@interface implement (import "" "countCodes_")
  (param $ptr i32)(param $len i32) (result i32))
  local.get $ptr
  local.get $len
  string.from.utf8 "memi"
  call-import "countCodes"
  i32.from.u32
)
```

The heart of this adapter is the special adapter instruction `string.from.utf8`
which takes a memory slice -- represented as a pair of `i32`s denoting the base
address and the length of the string -- and returns a `string`.

The `string.from.utf8` instruction can optionally reference an exported memory
(`"memi"`); something that will be useful when using Interface Types to realize
shared-nothing linking. This is why we also exported the memory as `"memi"` from
the core webAssembly module -- although this does not imply that we export the
memory from the wrapped Interface Types module.

>Note: we are assuming that the core webAssembly representation of the string
>data is as a region of memory -- with a base address and a length in bytes. The
>string itself is assumed to be encoded as UT8; other notions of string
>representation are possible, including the C-style null terminated string. In
>order to map C-style strings to make them usable for our `"countCodes"`
>Interface Types function, extra calls to the equivalent of `strlen` would have
>to be added to the adapter.

The return from `"countCodes"` is an unsigned 32 bit integer denoted by the
`u32` return type. This is mapped to the regular core webAssembly type `i32`
using the `i32.from.u32` instruction. Of course, this particular instruction is
pretty much a no-operation; however, there are several variants of _integer
coercion instructions_; some of which have a more significant semantics.

## Exporting a Unicode Counter

The other side of the Unicode counter is the actual implementation of the
count. If we assume that that too is implemented in a (different) webAssembly
module then there will be a corresponding export adapter for that module. 

The actual details of counting unicodes is not important for us here; we focus
on the adapter -- how to expose the counting functionality to external modules.

The export adapter is a mirror of the import adapter: its primary task is to map
the incoming `string` into memory so that a core webAssembly module can count
the codepoints. This mapping is achieved with the `utf8.from.string`
instruction:

To implement `countCodes` the incoming `string` must be mapped to the local
linear memory; which, in turn, implies invoking an allocator to find space for
it:

```
(@interface func (export "countCodes")
  (param $str string) (result u32)
  local.get $str
  string.size
  call-export "ex:malloc"
  own (i32)
    call-export "ex:free"
  end
  let (result u32) (local $tgt (owned i32))
    local.get $str
    local.get $tgt
	owned.access
    utf8.from.string "memx"
    call-export "countCodes_imp"
    u32.from.i32
  end
)

(memory (export "memx") 1)

(func $malloc
  (param $sze i32) (result i32)
  ...
)
```

There is a lot going on here, so we will take this slowly:

1. Unlike normal webAssembly functions, the type signature of this export
   adapter function is expressed using types from the Interface Types schema: it
   takes a `string` and returns a `u32`.
1. The `string.size` instruction is a special adapter instruction that takes a
   `string` value and returns the number of bytes needed to represent the string
   -- assuming a UTF-8 encoding scheme. This is used to allocate a chunk of
   memory -- with the call to `"ex:malloc"` for the string.
1. We use a convention of labeling functions we call from the core _exporting_
   module with `"ex:"` to simplify identification. Similarly, where appropriate,
   we would use `"im:"` to reference a function from the core importing
   WebAssembly module.
1. Since we want to avoid a memory leak, we want to arrange for the allocated
   memory to be freed. We arrange for this to occur at the end of the adapter
   function by using an `own` instruction block -- which when the memory can no
   longet be access invokes its sub-block -- which releases the allocated memory
   block.
1. The call to `"ex:malloc"` (and the related call to `"ex:free"`) is using an
   instruction to _call the exported_ function `"ex:malloc"`. This is similar to
   a normal call to a local function -- except that, since the adapter lives
   outside the core WebAssembly module, it must be to a function exported from
   that adapter. However, since we plan to wrap the core WebAssembly module as
   an Interface Types module, we do not need to _re-export_ these
   functions. That way we can use functions defined by the exporting module
   without having to expose them to all users of the module.
1. The `let` instruction is part of the function references proposal. We use it
   here in lieu of complex stack manipulation.
1. The `utf8.from.string` instruction takes a `string` value, a region of memory
   denoted by a base address and a size -- and copies the string into memory. It
   assumes that the target encoding is `utf8`.
1. Once the string value is in memory the actual counting of unicodes may take
   place -- denoted by the call to `$countCodes_impl`.
1. The value returned by the `$countCodes_impl` function has to be mapped to the
   Interface Types schema type `u32` -- which is the purpose of the
   `u32.from.i32` instruction.

## Managing Memory

The export adapter for `countCodes` needs to copy its `string` argument into
memory before the implementation function `$countCodes_impl` can actually
process the string. To do that, a memory block is allocated within the exporting
module's memory (which may be different to the memory used by the importing
module).

Once the `$countCodes_impl` function has completed its work, the memory block
must be freed; otherwise there will be a memory leak.

Memory management within adapters is a little different to that within a core
WebAssembly module: for various reasons, memory allocated within an adapter is
not fully visible within the core module; at the same time, the lifetime of such
memory tends to be very short and under the complete control of the adapter
itself.

>For regular C programs (not necessarily C++), memory management is implemented
>via some form of convention. Typically, the caller of a function assumes the
>obligation of managing memory for any _arguments_ to a call and for any _return
>results_ from the call. This is a convention for C, not enforced by language
>features.

>The situation for WebAssembly is similar; except that any convention being
>followed by any given core WebAssembly module must act in partnership with the
>adapter code -- which is also typically automatically generated by toolchains.

There are several different scenarios involving memory allocated within
adapters. In this case, it happens that the memory allocated for the string
could be deallocated immediately after the call to `$countCodes_impl`.

In other scenarios, the exported function returns an allocated block of memory
that must be released by the adapter. However, this cannot always be done within
the given adapter: if an export adapter is returning a `string` (say) then the
memory for the string must not be released until the string has been safely
consumed -- which may be in an import adapter.

So, the export and import adapters involved in an API must _coordinate_ their
memory management. The memory allocated by an export adapter may be released by
the corresponding import adapter; and vice versa. The added complication is that
the adapters in question will often have been generated by different toolchains.

To resolve this, we use a system of explicit memory _ownership_. The `own`
instruction takes a resource (in this case the address of a memory block) and
registers a block of instructions to execute when the resource is no longer
needed.

The `own` instruction takes a sequence of values from the stack and encapsulates
them within a special structure that also encludes this releasing block of
instructions. The type signature for the instruction reflects this:

```
own: T* --> (owned T*)
```

The special instruction `owned.access` is used to get at the underlying values
that have been so wrapped -- in this case the address of the memory buffer to
place the copy of the string.

As we shall see, releasing the memory is implemented as part of an adapter
fusion process; that is also able to deterministically predict when a given
`owned` resource is no longer needed.

## Shared Nothing Linking

The role of an import adapter is to implement a core webAssembly import by
mapping it to an import of a function in the realm of Interface
Types. Conversely, the role of an export adapter is provide an implementation of
an Interface Types function in terms of core webAssembly.

Normally, import and export adapters are brought together as part of an import
resolution process that the webAssembly host provides when instantiating
webAssembly modules.

However, there is another scenario where import and export adapters are brought
together -- when combining multiple webAssembly modules in a static linking
process. Static linking involves taking two or more webAssembly modules and
combining them by converting imports and exports into local functions.

So-called _shared nothing static linking_ involves linking together modules
where they do not share memory (or anything other than explicitly
imported/exported elements). The result is a single module -- with potentially
further imports and exports -- composed of elements which provably do not need
to trust each other.

The process of static linking involves combining import adapters and export
adapters -- specifically by inlining calls to the corresponding export
adapters. In addition to inlining, other simplications may follow.

One major result of this inlining and simplication is that the pairs of
import/export adapters are replaced with generated adapters which implement
bridges between the vistigial modules within the combined module.

The two important characteristics of these adapters are (a) they are synthesized
from the import and export adapters and (b) comprise solely of core webAssembly
instructions. This, in turn, supports the minimal trust model implied by shared
nothing linking: there is provably no possibility of a function in one vestigial
module side-effecting resources in another.

## Combining Import and Export Adapters

The first step in constructing an adapter from the corresponding import and
export adapters is to inline the export adapter into the import adapter. For
example, we can combine the import and export adapters for `countCodes`:

```
(func $countCodes
  (param $ptr i32)(param $len i32) (result i32)
    local.get $ptr
    local.get $len
    string.from.utf8 "memi"
    let (result i32)(local $str string)
      local.get $str
      string.size
      call-export "ex:malloc"
      own (i32)
        call-export "ex:free"
      end
      let (result u32) (local $tgt (owned i32))
        local.get $str
        local.get $tgt
	    owned.access
        utf8.from.string "memx"
        call-export "ex:countCodes_impl"
        u32.from.i32
      end
    end
    i32.from.u32
)
```

Our first task is to identify the correct positioning of the memory management
functions -- specifically, where we can safely invoke the `call free` function
to release the memory allocated for the string.

One straightfoward technique for this is to add additional local variables to
the adapter function -- that are used to hold the references collected by the
`own` instruction itself. 

At the same time, we can unwrap the various uses of `owned.access`, and invoke
the wrapped code at the end of the combined adapter:

```
(func $countCodes
  (param $ptr i32)(param $len i32) (result i32)
  (local $m1 i32)
    local.get $ptr
    local.get $len
    string.from.utf8 "memi"
    let (result i32)(local $str string)
      local.get $str
      string.size
      call-export "ex:malloc"
	  local.tee $m1
      let (result (u32) (local $tgt i32)
        local.get $str
        local.get $tgt
        utf8.from.string "memx"
        call-export "ex:countCodes_impl"
        u32.from.i32
      end
    end
    i32.from.u32
    local.get $m1
    call-export "ex:free"
)
```

The process of simplication involves taking pairs of lifting and lowering
operators -- each typically originally coming from either the import or the
export adapter -- and replacing them with core instructions that implement the
combination.

For example, the instructions:

```
...
u32.from.i32
i32.from.u32
...
```

cancel each other out, and can be completely eliminated from the combined code.

After inlining and simple local variable binding elimination, we get a pair of
coercion operators that read a string out of one memory and write it into
another:

```
  local.get $ptr
  local.get $len
  string.from.utf8 "memi"
  local.set $str

  local.get $len
  call-export "ex:malloc"
  local.tee $m1

  local.get $str
  local.get $def_0
  utf8.from.string "memx"
```

>Note that we eliminated the `string.size` instruction by replacing it with it's
>_source_ -- the `$len` parameter.

After collapsing the `string.from.utf8` and `utf8.from.string` pair we get:

```
  local.get $ptr

  local.get $len
  call-export "ex:malloc"
  local.tee $m1

  local.get $len
  memory.copy "memi" "memx"
```

I.e., we replace the combination with a memory copy from one memory space (the
importing memory `"memi"`) to another (the exporting memory `"memx"`). This, of
course, relies on both the [bulk memory][Bulk-memory] and the [multiple
memory][Multi-memory] proposals.

We are guaranteed to be able to perform this replacement because every import
adapter must be paired with a corresponding export adapter -- of the same
type. This includes the corresponding operators for handling the `string`
argument to the call.

The final adapter function has no remaining special adapter instructions:


```
(func $countCodes
  (param $ptr i32)(param $len i32) (result i32)
  (local $m1 i32)
  local.get $ptr

  local.get $len
  call-export "ex:malloc"
  local.tee $m1

  local.get $len
  memory.copy "memi" "memx"
  call-export "ex:countCodes_impl"

  local.get $m1
  call-export "ex:free"
)
```

This implementation assumes that the `malloc` cannot fail; in other notes we
examine how to account for failures and exceptions.

## Summary

The `countCodes` function is fairly simple (one might not even bother importing
it); however, it also illustrates many of the aspects needed to import and
export functions.

We have also seen how combining and simplifiying import with export adapters
allows a form of linking of webAssembly modules that is both static and is
provably minimal in its assumptions of shared semantics.

[core spec]: https://webassembly.github.io/spec/core

[Custom Section]: https://webassembly.github.io/spec/core/binary/modules.html#custom-section

[Bulk-memory]: https://github.com/WebAssembly/bulk-memory-operations/blob/master/proposals/bulk-memory-operations/Overview.md

[Multi-memory]: https://github.com/WebAssembly/multi-memory
