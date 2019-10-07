# Paint a vector of points

In this example we look at how sequences are passed into an API. The signature
of `vectorPaint` is assumed to be:

```
point ::= pt(i32,i32)
vectorPaint:(array<point> pts) => returnCode
```

## Export

It is assumed that the implementation of `vectorPaint_` requires a
memory-allocated array; each entry of which consists of two contiguous `i32`
values.

The array is modeled as two values: a pointer to its base and the number of
elements in the array.

The primary pattern with processing arrays is the for-loop pattern, together
with array indexing; which is modeled by the `for` instruction; which iterates a
code fragment over a range of numbers.

```
(@interface datatype @point
  (oneof
    (variant "pt"
      (tuple i32 i32))))

(memory (export "memx") 1)

(func $vectorPaint_ (export ("" "vectorPaint_"))
  (param i32 i32) (result i32)

(@interface func (export "vectorPaint")
  (param $pts (array @point))
  (result @returnCode)
  
  local.get $pts
  array-to-memory #8 "malloc" $pt $ix $ptr
    local.get $pt
    field.get #@point.pt.0
    s32-to-i32
    local.get $ptr
    i32.store {offset 0}
    local.get $pt
    field.get #@point.pt.1
    s32-to-i32
    local.get $ptr
    i32.store {offset 4}
  end
  call $vectorPaint_
  i32-to-enum @returnCode
)
```

The `array-to-memory` block instruction iterates over an array and executes its
block argument for each element of the array. The `array-to-memory` instruction asks the given allocator to
allocate sufficient space in linear memory for the copied out array (given by
multiplying the stride of `#8` by the number of elements in the source array.

In addition, the `array-to-memory` instruction establishes three local variables
that are in scope for the entire operation:

* `$pt` which is the element of the array to map

* `$ptr` the offset within linear memory where the mapped element is located.

* `$ix` the index of the element to map

The `array-to-memory` instruction leaves on the stack the offset within linear
memory where the newly allocated struct is.


## Import

The primary task in passing a vector of values is the construction of an `array`
to be passed to the imported function.

```
(func $vectorPaint_ (import ("" "vectorPaint_"))
  (param i32 i32) (result i32)

(@interface func $vectorPaint (import "" "vectorPaint")
  (param @ptr (array @point))
  (result @returnCode))
  
(@interface implement (import "" "vectorPaint_")
  (param $points i32)
  (param $count i32)
  (result i32)

  local.get $points
  local.get $count
  memory-to-array @point #8 $ix $pt
    local.get $pt
    i32.load {offset 0}
    i32-to-s32
    local.get $pt
    i32.load {offset 4}
    i32-to-s32
    create @point
  end
  call $vectorPaint
  enum-to-i32 @returnCode
)
```

The `memory-to-array` instruction is a higher-order instruction that is used to
create an array from a contiguous region of linear memory. The body of the
instruction is executed once for each element of the array in memory (the two
arguments to the instruction give the memory offset and the count); within the
body of the loop the bound variables `$ix` and `$pt` are the index of the
element and its memory offset respectively.

The two literal operands of `memory-to-array` are the type of elements of the
constructed array and the stride length of the linear memory array.

The body of the instruction should return an element of the resulting array; and
the instruction itself terminates with the array on the stack.

## Adapter

Combining the import and export sequences into an adapter code depends on being
able to fuse the generating loop with the iterating loop.

The initial in-line version gives:

```
(@adapter implement (import "" "vectorPaint_")
  (param $points i32)
  (param $count i32)
  (result i32)

  local.get $points
  local.get $count
  memory-to-array @point #8 $ix $pt
    local.get $pt
    i32.load {offset 0}
    i32-to-s32
    local.get $pt
    i32.load {offset 4}
    i32-to-s32
    create @point
  end
  
  let $pts
    local.get $pts
    array-to-memory #8 "malloc" $pt $ix $ptr
      local.get $pt
      field.get #@point.pt.0
      s32-to-i32
      local.get $ptr
      i32.store {offset 0}
      local.get $pt
      field.get #@point.pt.1
      s32-to-i32
      local.get $ptr
      i32.store {offset 4}
    end
  end
  call $vectorPaint_
  i32-to-enum @returnCode
  enum-to-i32 @returnCode
)
```

The reasoning for the next loop fusion is that the first loop is generating the
same sequence that the second loop is consuming. So, we fuse the loops by
placing the body of the second loop immediately within the first loop -- after
the construction of individual elements; and eliding the construction of the
array itself.

```
(@adapter implement (import "" "vectorPaint_")
  (param $points i32)
  (param $count i32)
  (result i32)

  local.get $count ;; this one is for the eventual call to Mi:$vectorPaint_
  local.get $count
  allocate #8 $arr_ "malloc"
    local.get $points
    local.get $count
    memory.loop #8 $ix memi:$pt_ memx:$ptr_
      local.get $pt_
      i32.load {offset 0}
      i32.store {offset 0}
      i32.load {offset 4}
      i32.store {offset 4}
    end
  end
  call Mi:$vectorPaint_
)
```

Note: The trickiest part of this is actually the handling of the counts. In
particular, the rewrite needs to be able to determine the size of the array
before copying starts.

In some cases, by noticing that the load'n store is effectively a dense copy,
this can be further reduced to:


```
(@adapter implement (import "" "vectorPaint_")
  (param $points i32)
  (param $count i32)
  (result i32)

  local.get $points
  local.get $count
  array.copy #8 memi: memx: mx:malloc
  call Mi:$vectorPaint_
)
```
