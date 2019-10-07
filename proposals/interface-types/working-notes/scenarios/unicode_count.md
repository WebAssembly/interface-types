# Counting Unicodes

Passing a string and returning the number of unicode code points in it. The
interface type signature for `countCodes` is:

```
countCodes:(string)=>u32
```

## Export

To implement `countCodes` the incoming `string` must be mapped to the local
linear memory; which, in turn, implies invoking an allocator to find space for
it:

```
(@interface func (export "countCodes")
  (param $str string) (result u32)
  local.get $str
  string-to-memory $memx "malloc"
  call "countCodes_"
  i32-to-u23
)

(memory (export "memx") 1)

(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```

## Import

Importing `countCodes` involves reading a `string` out of linear memory.

```
(memory (export "memi" 1)
(func $count_ (import ("" "countCodes_"))
  (param i32 i32) (result i32))

(@interface func $count (import "" "countCodes")
  (param $str string) (result u32))
  
(@interface implement (import "" "countCodes_")
  (param $ptr i32 $len i32) (result i32))
  local.get $ptr
  local.get $len
  memory-to-string "memi"
  call-import "countCodes"
  u32-to-i32
)
```

## Adapter code

After inlining and simple local variable binding elimination, we get a pair of
coercion operators that read a string out of one memory and write it into
another:

```
(@adapter implement (import "" "countCodes_")
  (param $ptr i32 $len i32) (result i32))
  local.get $ptr
  local.get $len
  memory-to-string Mi:"memi"
  string-to-memory Mx:$memx Mx:"malloc"
  call Mx:"countCodes_"
)
```
which, after collapsing coercion operators, becomes:
```
(@adapter implement (import "" "countCodes_")
  (param $ptr i32 $len i32) (result i32))
  local.get $ptr
  local.get $len
  string.copy Mi:"memi" Mx:"memx" Mx:"malloc"
  call Mx:"countCodes_"
)
```

This assumes that `string.copy` combines memory allocation, string copy and
returns the new address and repeats the size of the string.

This also assumes that the `malloc` cannot fail; below we look at exception
handling as a way of partially recovering from this failure. Without explicit
exception handling, a failed `malloc` is required to trap.
