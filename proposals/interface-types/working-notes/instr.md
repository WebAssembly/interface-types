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
| `i64-to-s64` | .. `i64` => .. `s64` | Lift i64 to signed 64 bit integer, with sign extension |
| `i64-to-u64` | .. `i64` => .. `u64` | Lift i64 to unsigned 64 bit integer, zero filled |


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
| `s64-to-i32x` | .. `u64` => .. `i32` | Map signed 64 bit to `i32`, error if overflow |
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

#### Allocate and defer

The `allocate` instruction is a block instruction which may include a defer;
`allocate` with a defer block sets up a continuation that will be entered when
the allocated memory needs to be released.

```
i32.const &lt;sz allocate &lt;malloc> &lt;mem> instr-a* defer instr-d* end -->
  label{ i32.const &lt;ptr> i32.const &lt;sz> instr-d* } instr-a* end
```

>Note: still to show how this survives rewriting when paired with corresponding
>lifting instructions.

#### Release

The `release` instruction is a marker instruction that denotes a call to a
corresponding `free` function.

```
i32.const &lt;ptr> &lt;sz> release --> epsilon
```

#### memory-to-array

This is a block instruction, with two productions:

```
i32.const p i32.const 0 memory-to-array sz instr* end --> epsilon

i32.const p i32.const c memory-to-array sz instr* end where c>0 -->
  label{ i32.const (p+sz) i32.const (c-1) memory-to-array sz instr* end} i32.const p i32.const c instr* end
```

The interior instructions of the `memory-to-array` block are executed with the
memory offset of the current array element and a count of the number of
remaining elements in the array on the top of the stack.

#### array-to-memory

Like the `memory-to-array` instruction, this is a block instruction:

```
i32.const p i32.const e i32.const e &lt;array>.A array-to-memory sz instr* end --> epsilon
i32.const p i32.const i i32.const e &lt;array>.A array-to-memory sz instr* end where i<e -->
  label{ i32.const (p+sz) i32.const i32.const (i+1) i32.const e array-to-memory sz instr* end } i32.const p &lt;array>[i] instr* end
```

### Sequences

## Invoking

### Call-import

### Invoke Interface

## Control flow

### Let variable definitions

### Deferred execution

### Exceptions
