# Interface Type Instructions

## Lifting and lowering

### ixx-to-sxx

The numeric lifting and lowering instructions map between WASM's view of numbers and IT's view of numbers.

| | | |
| ----- | ----------- | ---------- |
| `i32-to-s8` | .. `i32` => .. `s8` | Lift least significant 8 bits as signed 8 bit integer |
| `i32-to-s8x` | .. `i32` => .. `s8` | Lift least 8 bits as signed 8 bit integer, error if more than 7 bits significant  |
| `i32-to-u8` | .. `i32` => .. `u8` | Lift least significant 8 bits as unsigned 8 bit integer |
| `i32-to-s16` | .. `i32` => .. `s16` | Lift least significant 16 bits as signed 16 bit integer |
| `i32-to-s16x` | .. `i32` => .. `s16` | Lift least significant 16 bits as signed 16 bit integer, error if more than 15 bits significant  |
| `i32-to-u16` | .. `i32` => .. `u16` | Lift least significant 16 bits as unsigned 16 bit integer |
| `i32-to-s32` | .. `i32` => .. `s32` | Lift i32 to signed 32 bit integer |
| `i32-to-u32` | .. `i32` => .. `u32` | Lift i32 to unsigned 32 bit integer |
| `i32-to-s64` | .. `i32` => .. `s64` | Lift i32 to signed 64 bit integer, with sign extension |
| `i32-to-u64` | .. `i32` => .. `u64` | Lift i32 to unsigned 64 bit integer, zero filled |
| `i64-to-s8` | .. `i64` => .. `s8` | Lift least significant 8 bits as signed 8 bit integer |
| `i64-to-s8x` | .. `i64` => .. `s8` | Lift ls 8 bits as signed 8 bit integer, error if more than 7 bits significant  |
| `i64-to-u8` | .. `i64` => .. `u8` | Lift least significant 8 bits as unsigned 8 bit integer |
| `i64-to-s16` | .. `i64` => .. `s16` | Lift least significant 16 bits as signed 16 bit integer |
| `i64-to-s16x` | .. `i64` => .. `s16` | Lift least significant 16 bits as signed 16 bit integer, error if more than 15 bits significant  |
| `i64-to-u16` | .. `i64` => .. `u16` | Lift least significant 16 bits as unsigned 16 bit integer |
| `i64-to-s32` | .. `i64` => .. `s32` | Lift i64 to signed 32 bit integer |
| `i64-to-s32x` | .. `i64` => .. `s32` | Lift i64 to signed 32 bit integer, error if more than 31 bits significant |
| `i64-to-u32` | .. `i64` => .. `u32` | Lift i64 to unsigned 32 bit integer |
| `i64-to-s64` | .. `i64` => .. `s64` | Lift i64 to signed 64 bit integer |
| `i64-to-u64` | .. `i64` => .. `u64` | Lift i64 to unsigned 64 bit integer |

The small-step semantics of these instructions all follow a simple pattern. For non-erroring variants:

>`ixx.const` N `ixx-to-tyy` --> `tyy.const N'` where `N'` is the result of coercing `N` to type `tyy`

the variants with an `x` suffix might raise an exception:

>`ixx.const N ixx-to-tyyx` --> `tyy.const N'` where `N'` is the result of coercing `N` to type `tyy` and the result of tyy-to-ixx = N

>`ixx.const N ixx-to-tyyx` --> `string.const "invalid coercion" raise`


An arithmetic coercion is safe iff there is an inverse coercion that preserves the value.

>Note: will need to adjust this to allow for a standard error exception


| | | |
| ----- | ----------- | ---------- |
| `s8-to-i32` | .. `s8` => .. `i32` | Map signed 8 bit to `i32` |
| `u8-to-i32` | .. `u8` => .. `i32` | Map unsigned 8 bit to `i32` |
| `s16-to-i32` | .. `s16` => .. `i32` | Map signed 16 bit to `i32` |
| `u16-to-i32` | .. `u16` => .. `i32` | Map unsigned 16 bit to `i32` |
| `s32-to-i32` | .. `s32` => .. `i32` | Map signed 32 bit to `i32` |
| `u32-to-i32` | .. `u32` => .. `i32` | Map unsigned 32 bit to `i32` |
| `s64-to-i32` | .. `s64` => .. `i32` | Map signed 64 bit to `i32` |
| `s64-to-i32x` | .. `s64` => .. `i32` | Map signed 64 bit to `i32`, error if overflow |
| `u64-to-i32` | .. `u64` => .. `i32` | Map unsigned 64 bit to `i32` |
| `u64-to-i32x` | .. `u64` => .. `i32` | Map unsigned 64 bit to `i32`, error if overflow |
| `s8-to-i64` | .. `s8` => .. `i64` | Map signed 8 bit to `i64` |
| `u8-to-i64` | .. `u8` => .. `i64` | Map unsigned 8 bit to `i64` |
| `s16-to-i64` | .. `s16` => .. `i64` | Map signed 16 bit to `i64` |
| `u16-to-i64` | .. `u16` => .. `i64` | Map unsigned 16 bit to `i64` |
| `s32-to-i64` | .. `s32` => .. `i64` | Map signed 32 bit to `i64` |
| `u32-to-i64` | .. `u32` => .. `i64` | Map unsigned 32 bit to `i64` |
| `s64-to-i64` | .. `s64` => .. `i64` | Map signed 64 bit to `i64` |
| `u64-to-i64` | .. `u64` => .. `i64` | Map unsigned 64 bit to `i64` |


### pack and unpack

These instructions construct and deconstruct records into their constituent parts.

| | | |
| --- | ---- | ------ |
| `pack` &lt;typeref> | .. F1 .. Fn => .. R | Remove top n elements from stack as fields in record |
| `unpack` &lt;typeref> | .. R => .. F1 .. Fn | Remove top element and replace with fields in order

Note that the stack order of fields in `pack` and `unpack` is the same, with the
first field being the deepest on the stack.

### ixx-to-enum

These instructions refer to a type definition that is an enumeration type.

| | | |
| ----- | ----------- | ---------- |
| `enum-to-i32` &lt;Type> | .. &lt;Enum> => .. `i32` | Map enumeration to `i32` |
| `i32-to-enum` &lt;Type> | .. `i32` => .. &lt;Enum> | Map `i32` to enumeration |

Note that enumeration types are considered equivalent up reordering. We likely
need to rely on this to give a canonical value to each enumeration value.

### memory-to-string

The memory string instructions assume that the non-interface type representation
of a string is as a contiguous sequence of unicode characters in a memory
region.

| | | |
| ----- | ----------- | ---------- |
| `memory-to-string` &lt;Mem> | .. `i32` `i32`=> .. `string` | Memory buffer (base count) to `string |
| `string-to-memory` &lt;Mem> &lt;Malloc> | .. `string` => .. `i32` `i32` | Copy a string value into memory &lt;Mem> using &lt;Malloc> to allocate within that memory. |

Note that a memory-based string is assumed to represented as a pair of `i32`
values: the first is the offset in the memory of the first byte and the second
is the number of bytes in the string -- not the number of unicode characters.

These instructions also reference a memory index literal -- which defaults to 0
-- which indicates which memory the string is held in.

### Memory and Arrays

| | |
| -- | -- |
| `memory-to-array` | Lift a region of memory as contents of an array |
| `array-to-memory` | Lower an array by mapping contents to a region of memory |
| `array.count` | Report the number of elements in the array |

#### memory-to-array

The `memory-to-array` instruction is used to lift a block of memory --
containing the elements of the array in a contiguous region -- into an `array`
of the appropriate element type.

This is a block instruction, with a complex set of productions defining its semantics:


```
i32.const p i32.const c memory-to-array sz <type> instr* end -->
  i32.const p i32.const c array<type>.const [] memory-to-array-loop sz instr* end

i32.const p i32.const 0 array<type>.const a memory-to-array-loop <type> sz instr* end --> array<type>.const a
i32.const p i32.const c array<type>.const a memory-to-array-loop <type> sz instr* end where c>0 -->
  label{ i32.const (p+sz) i32.const (c-1) array<type>.const a memory-to-array-loop sz instr* end} array<type>.const a i32.const p i32.const c instr* append-to-array end
  
array<type>.const a <type>.const b append-to-array --> array<type>.const a' where all i in 0..a.count a'[i]=a[i] and a'[a.count] = b 
```

The interior instructions of the `memory-to-array` block are executed with the
memory offset of the current array element and a count of the number of
remaining elements in the array on the top of the stack.

#### array-to-memory

The `array-to-memory` instruction maps an array to a region of memory. It must
be the case that the region of memory given to the instruction is large enough
to accomodate the contents of the array.

Like the `memory-to-array` instruction, this is a block instruction:

```
i32.const p i32.const e i32.const e <array>.A array-to-memory sz instr* end --> epsilon
i32.const p i32.const i i32.const e <array>.A array-to-memory sz instr* end where i<e -->
  label{ i32.const (p+sz) i32.const i32.const (i+1) i32.const e array-to-memory sz instr* end } i32.const p <array>[i] instr* end
```

#### array.count

The `array.count` instruction returns the number of elements in an array. 

>val:array&lt;t> `array.count` --> `i32.const K` where there are `K` elements in the array.

### Sequences

Sequences are lifted and lowered via _iterators_. 

| | |
| -- | -- |
| `iterator.start` sequence&lt;type> | Start sequence processing |
| `iterator.while` label | Loop block for each element of the sequence |
| `iterator.close` | Terminate processing sequence |
| `sequence.start` &lt;type> | Initiate construction of a sequence |
| `sequence.append` | Append element to sequence |
| `sequence.complete` | Finish creation of sequence |

## Invoking

There are two primary instructions for invoking Interface Types functions:
`call-function` and `invoke-method`. The latter models invoking a method on an
object, or in Interface Type terms, invoking a method on an entity that has an
interface.

>Note. There are two technical differences between calling a function and
>invoking a method: a method is effectively a function with a distinguished
>first argument that is also an inseperable element of a set of methods.

| | |
| -- | -- |
| `call-function` Fn | Call Interface Types function |
| `invoke-method` Nm | Invoke method Nm |


### Call Function

The `call-function` instruction calls a function whose signature is expressed
using the Interface Types schema.

>Note that if the called function raises an exception then we require (in the
>binary format) a count which indicates which handler is the designated handler
>for exceptions from the function.

### Invoke Method

The `invoke-method` instruction invokes a method on an object whose type is
expressed in terms of an interface (not to be confused with Interface Types ...).

## Control flow

### Case Handling

The `case` instruction is used to map different variants of a type -- expressed
as an _algebraic data type_. Case instructions may either be involved lifting
code or in lowering code.

```
val case B1 .. Bk end --> vali Bi where Bi= block ti=>tr instr* end and val:ti
```

Here, `val` is assumed to be of an algebraic type; which means that its value is
one of a fixed (k) set of variants. Let the actual variant of val have type `ti`
where `i` is the ith variant and block Bi has multi-value signature ti=>tr. 

The `case` instruction demultiplexes the variation of types into a particular
variant type; and executes a block of instructions that assumes that a value of
that type is on the stack.

It is a validation error for the number of block choices to be different to the
number of variants in the type of val; each sub-block should correspond to
exactly one variant of the type of `val`.

#### Vary Type

The `vary` instruction lifts a value of a given type and _converts_ it into a
variant of an algebraic type:

```
val:t vary i <type> --> val_i_:<type> 
```
where `i` is an index into the algebraic type &lt;type> whose variant type is `t`.




### Let variable definitions

The `let` instruction is used to introduce local names for values that are on
the stack. It is a block instruction, the local name is in scope for the
contents of the block instruction.

The `let` instruction is part of the 'function reference types' proposal but is
reproduced here for completeness.

### Deferred execution

In order to prevent memory leaks, especially when returning memory-allocated
values from a call, it is necessary to be able to free allocated
memory. However, the order of execution of a malloc/free pair does not always
line up with the structure of an adapter. For example, allocated memory may need
to remain valid until the caller of the export adapter has finished its work.

To support this we have two instructions: the `defer-scope` instruction
establishes a context -- and a scope -- within which a `deferred` block may be
executed. Any `deferred` blocks entered within the scope are not actually
executed until the last instruction within the `defere-scope` instruction has
been executed.

```
defer-scope t=>t instr* end
```
and

```
deferred t_k_ instr* end
```

The semantics of `deferred` are that the instructions in its block are
executed at the end of containing `defer-scope` instruction. In addition, the top _k_ elements of
the stack are preserved and made available within the block when it is actually executed.

There are two common use cases for deferred execution: when returning a memory
allocated value to its caller and for ensuring that allocations are properly
undone in the case of exception handling.


```
i32.const Sz
call $malloc
deferred (i32) ;; we want to keep the pointer to the allocated block
  call $free   ;; the top of the stack contains the address of the allocated block
end
```

The `defer-scope` instruction sets up a defer context in the stack, whose role
is to collect deferred code. When the body of the `defer-scope` reduces to value
instructions then execute any deferred instructions:

```
defer-scope instr* end --> deferred{epsilon} instr* end

deferred{instr*} valn end --> instr* valn
```

The `deferred` instruction (which must be in the scope of a `deferred` context) appends instructions to that context

```
deferred{ d-instr* } ;L; val(n) deferred t(n) f* end ;R; end  --> deferred{ d-instr* val(n) f* } ;L; val(n) ;R; end
```

>Question: Do we want to allow the `deferred` instruction to push back an arbitrary number of defer contexts (ala br)?
>Question: If we mark functions as throwing, should we also mark functions as deferring?

### Exceptions

The exception handling of Interface Types adapters is separate from, but
somewhat overlaid on, the exception types of core wasm.

There are four elements of the architecture of exceptions in Interface Types:
exception values, function signatures and instructions that raise and catch
exceptions.

#### Exception Values

Exceptions are values whose type is expressed within the Interface Types schema.

>Note: given the need to model different kinds of _exceptional_ cases, it is
>likely that the type of an exception is expressed in terms of an algebraic data
>type definition. This is not required by this proposal; it is quite possible
>for exceptions to be modeled as integer values.

#### Exception Signature

An exception signature is a function or method signature that indicates what
exceptions may be raised by the function. A function that may raise exceptions
has a different signature to one that does not:

```
foo: (string)=>string raises fooException
```

There is no restriction on the type of exceptions a function may raise, except
that a given function may only raise exceptions of that declared type. 

In affect, a `raises` annotation on a function type signals that the function
may either return a regular value or an exceptional value.

#### Raising exceptions

The `raise` instruction is used to signal that an exception is to be thrown. If
the `raise` instruction occurs in the syntactic scope of a `catch-exception`
block of the correct type then it amounts to a break to the handler part of that
`catch-exception` block. If the exception is not in an appropriate syntactic
scope then it must be the case that the adapter function is annotated with the
type of the exception.

```
val:t; raise -->
```

>Note: the `raise` instruction embeds a count which is a _scope count_: the
>number of handle contexts to unravel before finding the appropriate handler for
>the exception. That handler must be declared to handle exceptions of the type _t_.

#### Handling exceptions

The `handle` block instruction is used to mark a scope where exceptions may be
raised and handled. The `handle` instruction establishes a `handle` context
which is used by the `raise` instruction:

```
handle instr* onexception t instr_t* end --> handle{ instr_t* } instr* end
handle{instr_t*} val(n) end --> val(n)
```

There are actually two potential handle scopes: if a function is declared to
throw an exception of some type then that also establishes a handle scope for
exceptions of that type.

>Note: need more detail here.
