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

## Importing Unicode Counter

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

The style of `@interface` sections follow the recommendations of the [Custom
Section] proposal. This `func` statement declares that there is an Interface
Type function -- `countCodes` -- that we are importing and that its type is from
strings to unsigned 32 bit integers.

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
  defer-scope
    local.get $ptr
    local.get $len
    memory-to-string "memi"
    call-import "countCodes"
    u32-to-i32
  end
)
```

The heart of this adapter is the special adapter instruction `memory-to-string`
which takes a memory slice -- represented as a pair of `i32`s denoting the base
address and the length of the string -- and returns a `string`. 

The `memory-to-string` instruction can optionally reference an exported memory
(`"memi"`); something that will be useful when using Interface Types to realize
shared-nothing linking. This is why we also exported the memory as `"memi"` from
the core webAssembly module -- although this does not imply that we export the
memory from the wrapped Interface Types module.

>Note: we are assuming that the core webAssembly representation of the string
>data is as a region of memory -- with a base address and a length in
>bytes. Other notions of string representation are possible, including the
>C-style null terminated string. In order to map C-style strings to make them
>usable for our `"countCodes"` Interface Types function, extra calls to the
>equivalent of `strlen` would have to be added to the adapter.

The return from `"countCodes"` is an unsigned 32 bit integer denoted by the
`u32` return type. This is mapped to the regular core webAssembly type `i32`
using the `u32-to-i32` instruction. Of course, this particular instruction is
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
the codepoints. This mapping is achieved with the `string-to-memory`
instruction:

To implement `countCodes` the incoming `string` must be mapped to the local
linear memory; which, in turn, implies invoking an allocator to find space for
it:

```
(@interface func (export "countCodes")
  (param $str string) (result u32)
  local.get $str
  string.size
  call $malloc
  deferred (i32)
    let (local $tgt i32)
      local.get $str
      local.get $tgt
      string-to-memory "memx"
      call $countCodes_impl
      i32-to-u32
    end
  finally
    call $free
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
1. The `string.size` instruction is a special adapter instruction that
   takes a `string` value and returns the number of bytes needed to represent
   the string -- assuming a UTF-8 encoding scheme. This is used to allocate a
   chunk of memory -- with the call to `$malloc` for the string.
1. Since we want to avoid a memory leak, we want to arrange for the allocated
   memory to be freed. We arrange for this to occur at the end of the adapter
   function by using a `deferred` instruction block -- whose `finally` sub-block
   is executed when the memory can no longer be accessed.
   
>   In effect, the address returned by `$malloc` is preserved by the `deferred`
   block and revived when executing the `finally` sub-block.
1. The `let` instruction is part of the function references proposal. We use it
   here in lieu of complex stack manipulation.
1. The `string-to-memory` instruction takes a `string` value, a region of memory
   -- denoted by a base address and a size -- and copies the string into memory.
1. Once the string value is in memory the actual counting of unicodes may take
   place -- denoted by the call to `$countCodes_impl`.
1. The value returned by the `$countCodes_impl` function has to be mapped to the
   Interface Types schema type `u32` -- which is the purpose of the `i32-to-u32`
   instruction.
1. The last operation in the adapter function is the call to `$free` which
   deallocates the memory allocated earlier. As noted above, this is intended to
   be performed when the memory is no longer needed.
   
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
export adapters is to inline the export adapter into the import adapter. For example, we can combine the import and export adapters for `countCodes`:

```
(func $countCodes
  (param $ptr i32)(param $len i32) (result i32))
  defer-scope
    local.get $ptr
    local.get $len
    memory-to-string "memi"
    let (local $str string)
      local.get $str
      string.size
      call $malloc
      deferred (i32)
        let (local $tgt i32)
          local.get $str
          local.get $tgt
          string-to-memory "memx"
          call $countCodes_impl
          i32-to-u32
        end
      finally
        call $free
    end
  end
  u32-to-i32
)
```

The process of simplication involves taking pairs of lifting and lowering
operators -- each typically originally coming from either the import or the
export adapter -- and replacing them with core instructions that implement the
combination.

In the case of the `defer-scope` and `deferred` instructions we eliminate both
and move the instructions within the `finally` sub-block to the end of the
`defer-scope` block:

```
(func $countCodes
  (param $ptr i32)(param $len i32) (result i32))
  local.get $ptr
  local.get $len
  memory-to-string "memi"
    
  let (local $str string)
    local.get $str
    string.size
    call $malloc
    local.tee $def_0
    let (local $tgt i32)
      local.get $str
      local.get $tgt
      string-to-memory "memx"
      call $countCodes_impl
      i32-to-u32
    end
  end
  u32-to-i32
  local.get $def_0
  call $free
)
```

After inlining and simple local variable binding elimination, we get a pair of
coercion operators that read a string out of one memory and write it into
another:

```
  local.get $ptr
  local.get $len
  memory-to-string "memi"
  local.set $str

  local.get $len
  call $malloc
  local.tee $def_0

  local.get $str
  local.get $def_0
  string-to-memory "memx"
```

>Note that we eliminated the `string.size` instruction by replacing it with it's
>_source_ -- the `$len` parameter.

After collapsing the `memory-to-string` and `string-to-memory` pair we get:

```
  local.get $ptr

  local.get $len
  call $malloc
  local.tee $def_0

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

Other pairs of lifting and lowering instructions can be similarly replaced. For
example, the pair:

```
  i32-to-u32
  ...
  u32-to-i32
```
can be simply removed, as their combination is a no-operation.

The final adapter function has no remaining special adapter instructions:

```
(func $countCodes
  (param $ptr i32)(param $len i32) (result i32))
  local.get $ptr

  local.get $len
  call $malloc
  local.tee $def_0

  local.get $len
  memory.copy "memi" "memx"
  call $countCodes_impl

  local.get $def_0
  call $free
)
```
This implementation assumes that the `malloc` cannot fail; in other notes we examine how to account for failures and exceptions.

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

