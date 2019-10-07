# A suite of adapter scenarios

This suite of examples is intended to demonstrate a spanning set of examples and
transformations that permit the use of interface types to specify imports and
exports.

## Notes
Added let, func.bind, env.get, field.get, string.copy, create instructions.

Changed arg.get to local.get because it is clearer when combined with let.

## Two Argument Integer Function

Calling a two argument integer function, should result in effectively zero code.

### Export

```
(@interface func (export "twizzle")
  (param $a1 s32)(param $a2 s32) (result s32)
  local.get $a1
  s32-to-i23
  local.get $a2
  s32-to-i32
  call "twizzle_"
  i32-to-s32
)
```

### Import

```
(@interface func (import "" "twozzle")
  (param $a1 s32)(param $a2 s32) (result s32)
)
(@interface implement (import "" "twozzle_")
  (param $b1 i32)(param $b2 i32) (result i32)
  local.get $b1
  i32-to-s32
  local.get $b2
  i32-to-s32
  call-import "twozzle"
  s32-to-i32
)

```

### Adapter Code

The adapter code, that maps the import of `twozzle_` to its implementation as
`twizzle_` is:

```
(@adapter implement (import "" "twozzle_")
  (param $b1 i32)(param $b2 i32) (result i32)
    local.get $b1
    local.get $b2
    call Mx:"twizzle_"
)
```

>Note: we adopt the convention that the `Mx:` prefix refers to the exporting
>module and `Mi:` refers to the importing module.

This should be viewed as the result of optimizations over an in-line substitution:
```
(@adapter implement (import "" "twozzle_")
  (param $b1 i32)(param $b2 i32) (result i32)
    local.get $b1
    i32-to-s32
    local.get $b2
    i32-to-s32
    let s32 (local $a2 s32)
      let s32 (local $a1 s32)
        local.get $a1
        s32-to-i23
        local.get $a2
        s32-to-i32
        call Mx:"twizzle_"
        i32-to-s32
      end
    end
    s32-to-i32
)
```

The `let` pseudo instruction pops elements off the stack and gives them names;
and is part of the [function reference
proposal](https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md#local-bindings).

The overall goal of the rewriting of adapters is to eliminate references to
interface types and their values. This reflects the intention to construct
executable code that implements the import of functions whose types are
specified using interface types.

The first step in 'optimising' this sequence is to remove the `let`
instructions, where possible, and replacing instances of references to them with
the sub-sequences that gave the bound variables their value.

For example, in

```
let s32 (local $a1 s32)
```
the sub-sequence that results in the value for `$a1` is:

```
local.get $b1
i32-to-s32
```

so, removing the `let` for `a2`, and replacing `local.get $a2` with its defining
subsequence gives:

```
(@adapter implement (import "" "twozzle_")
  (param $b1 i32)(param $b2 i32) (result i32)
    local.get $b1
    i32-to-s32
    local.get $b2
    i32-to-s32
    let s32 (local $a2 s32)
      local.get $b1
      i32-to-s32
      s32-to-i23
      local.get $a2
      s32-to-i32
      call Mx:"twizzle_"
      i32-to-s32
    end
    s32-to-i32
)

```
and, removing the redundant pair:

```
i32-to-s32
s32-to-i32
```
gives:

```
(@adapter implement (import "" "twozzle_")
  (param $b1 i32)(param $b2 i32) (result i32)
    local.get $b2
    i32-to-s32
    let s32 (local $a2 s32)
      local.get $b1
      local.get $a2
      s32-to-i32
      call Mx:"twizzle_"
      i32-to-s32
    end
    s32-to-i32
)
```

Repeating this for the second `let` gives:

```
(@adapter implement (import "" "twozzle_")
  (param $b1 i32)(param $b2 i32) (result i32)
    local.get $b1
    local.get $b2
    call Mx:"twizzle_"
    i32-to-s32
    s32-to-i32
)
```

with the final removal of the redundant coercion pair at the end:

```
(@adapter implement (import "" "twozzle_")
  (param $b1 i32)(param $b2 i32) (result i32)
    local.get $b1
    local.get $b2
    call Mx:"twizzle_"
)

```

Below, we will assume that similar transformations are applied automatically;
except where we need to show what happens more clearly.

## Counting Unicodes

Passing a string and returning the number of unicode code points in it. The
interface type signature for `countCodes` is:

```
countCodes:(string)=>u32
```

### Export

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

### Import

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

### Adapter code

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

## Getenv Function

The `getenv` function builds on `countCodes` by also returning a `string`. Its
interface type signature is:

```
getenv:(string)=>string
```

### Export


```
(@interface func (export "getenv")
  (param $str string) (result string)
  local.get $str
  string-to-memory "memx" "malloc"
  call "getenv_"
  memory-to-string "memx"
)

(memory (export "memx") 1)

(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```

### Import

The importer must also export a version of `malloc` so that the returned string can be
stored.

```
(func $getenv_ (import ("" "getenv_"))
  (param i32 i32) (result i32 i32))

(@interface func $getenv (import "" "getenv")
  (param $str string) (result string))

(@interface implement (import "" "getenv_")
  (param $ptr i32) (param $len i32) (result i32))
  local.get $ptr
  local.get $len
  memory-to-string "memi"
  call-import $getenv
  string-to-memory "memi" "malloc"
)

(memory (export "memi" 1)

(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```


### Adapter

After collapsing coercions and inlining variable bindings:

```
(@adapter implement (import "" "getenv_")
  (param $ptr i32) (param $len i32) (result i32 i32))
  local.get $ptr
  local.get $len
  string.copy Mi:"memi" Mx:memx Mx:"malloc"
  call Mx:"getenv_"
  string.copy Mx:"memx" Mi:"memi" Mi:"malloc"
)
```

## Fetch Url

The `fetch` function includes a callback which is invoked whenever 'something
happens' on the session. 

It also introduces two new enumeration types: `returnCode` and `statusCode`.

The signature of `fetch` as an interface type is:

```
fetch:(string,(statusCode,string)=>returnCode)=>returnCode
```

The callback requires an inline function definition, which is then bound to a
new closure.

The `func.bind` adapter instruction takes two literal arguments: a type
specification of the _capture list_ (which is structured in a way that is
analogous to the param section of a function) and a literal function definition.

The `func.bind` instruction also 'pops off' the stack as many values as are
specified in the capture signature; these are bound into a closure which is
pushed onto the stack.

### Export

```
(@interface datatype @returnCode 
  (oneof
    (enum "ok")
    (enum "bad")
  )
)

(@interface datatype @statusCode 
  (oneof
    (enum "fail")
    (enum "havedata")
    (enum "eof")
  )
)

(memory (export "memx") 1)

(@interface func (export "fetch")
  (param $u string)
  (param $cb (ref (func (param $status @statusCode) (param $text string) (result @returnCode))))
  (result string)
  local.get $u
  string-to-memory "memx" "malloc"
  
  local.get $cb
  func.ref $callBack
  func.bind (func (param @statusCode string) (result @returnCode)
  call $fetch_
  i32-to-enum @returnCode
)

(func $callBack
  (param $ecb (ref (func (param statusCode string) (result returnCode))))
  (param $status i32)
  (param $text i32)
  (param $len i32)
  (result i32)
  local.get $status
  i32-to-enum @statusCode
  local.get $text
  local.get $len
  memory-to-string "memx"
  local.get $ecb
  call-indirect
  enum-to-i32 @returnCode
)

(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```

### Import

Importing `fetch` implies exporting the function that implements the callback.

```
(memory (export "memi" 1)

(func $fetch_ (import ("" "fetch_"))
  (param i32 i32 (ref (func (param i32 i32)(result i32)))) (result i32))

(@interface func $fetch (import "" "fetch")
  (param string (ref (func (param @statusCode string)) (result @returnCode)))
  (result @returnCode))
  
(@interface implement (import "" "fetch_")
  (param $url i32) (param $len i32) (param $callb (ref (func (param i32 i32)(result i32)))) (result i32)
  local.get $url
  local.get $len
  memory-to-string "memi"
  local.get $callb
  func.ref $cbEntry
  func.bind (func (param i32 i32 i32) (result i32))
  call-import $fetch
  enum-to-i32 @returnCode
)

(func $cbEntry
  (param $callBk (ref (func (param i32 i32 i32) (result i32))))
  (param $status @statusCode)
  (param $text string)
  (result @returnCode)
  local.get $status
  enum-to-i32 @statusCode
  local.get $text
  string-to-memory "memi" "malloc"
  local.get $callbk
  call-indirect
  i32-to-enum @returnCode
)

(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```

### Adapter

Combining the export and import functions leads, in the first instance, to:

```
(@adapter implement (import "" "fetch_")
  (param $url i32) (param $len i32) (param $callb (ref (func (param i32 i32)(result i32)))) (result i32)
  local.get $url
  local.get $len
  memory-to-string "memi"
  local.get $callb
  func.ref $cbEntry
  func.bind (func (param i32 i32 i32) (result i32))
  
  let ($u string) ($cb (ref (func (param i32 i32 i32) (result i32))))
  local.get $u
  string-to-memory "memx" "malloc"
  
  local.get $cb
  func.ref Mx:$callBack
  func.bind (func (param @statusCode string) (result @returnCode)
  call Mx:$fetch_
  i32-to-enum @returnCode
  enum-to-i32 @returnCode
)
```

Some reordering, (which is hard to sanction if we allow side-affecting
instructions), gives:

```
(@adapter implement (import "" "fetch_")
  (param $url i32) (param $len i32) (param $callb (ref (func (param i32 i32)(result i32)))) (result i32)
  local.get $url
  local.get $len
  string.copy Mi:"memi" Mx:"memx" Mx:"malloc"

  local.get $callb
  func.ref Mi:$cbEntry
  func.bind (func (param i32 i32 i32) (result i32))
  func.ref Mx:$callBack
  func.bind (func (param @statusCode string) (result @returnCode)
  call Mx:$fetch_
)

```
The sequence of a `func.ref` followed by a `func.bind` amounts to a
partial function application and, like regular function application, can be
inlined. 

In particular, we specialize `Mx:$callBack` with the constant `Mi:$cbEntry`
replacing the bound variable `$ecb`. After inlining the now constant function
call, we get

```
(@adapter $callBackx func
  (param $callBk (ref (func (param i32 i32 i32) (result i32))))
  (param $status i32)
  (param $text i32)
  (param $len i32)
  (result i32)
  local.get $status

  local.get $text
  local.get $len
  string.copy Mx:"memx" Mi:"memi" Mi:"malloc"

  local.get $callbk
  call-indirect
)
```

and the original implementation of the adapter becomes:

```
(@adapter implement (import "" "fetch_")
  (param $url i32)
  (param $len i32)
  (param $callb (ref (func (param i32 i32)(result i32))))
  (result i32)
  local.get $url
  local.get $len
  string.copy Mi:"memi" Mx:"memx" Mx:"malloc"

  local.get $callb
  func.ref Mx:$callBackx
  func.bind (func (param i32 i32 i32) (result i32))
  call Mx:$fetch_
)
```

Which is reasonable code.

>Note that we are not able to specialize `Mx:$callBackx` further because the
>function value passed to `func.bind` is not a known function -- it is part of
>the `fetch` API.

## Pay by Credit Card

In order to process a payment with a credit card, various items of information
are required: credit card details, amount, merchant bank, and so on. This
example illustrates the use of nominal types and record types in adapter code.

>Note that we have substantially simplified the scenario in order to construct
>an illustrative example.

The signature of `payWithCard` as an interface type is:

```
cc ::= cc{
  ccNo : u64;
  name : string;
  expires : {
    mon : u8;
    year : u16
  }
  ccv : u16
}
payWithCard:(card:cc, amount:s64, session:resource<connection>) => boolean
```

### Export

In order to handle the incoming information, a record value must be passed --
including a `string` component -- and a `session` that denotes the bank
connection must be paired with an existing connection resource. The connection
information is realized as an `eqref` (an `anyref` that supports equality).

In this scenario, we are assuming that the credit card information is passed by
value (each field in a separate argument) to the internal implementation of the
export, and passed by reference to the import.

```
(@interface datatype @cc 
  (record "cc"
    (ccNo u64)
    (name string)
    (expires (record
      (mon u8)
      (year u16)
      )
    )
    (ccv u16)
  )
)

(@interface typealias @connection eqref)

(memory (export $memx) 1)

(func $payWithCard_ (export ("" "payWithCard_"))
  (param i64 i32 i32 i32 i32 i32 eqref) (result i32)

(@interface func (export "payWithCard")
  (param $card @cc)
  (param $session (resource @connection))
  (result boolean)

  local.get $card
  field.get #cc.ccNo   ;; access ccNo
  u64-to-i64

  local.get $card
  field.get #cc.name
  string-to-memory $mem1 "malloc"
  
  local.get $card
  field.get #cc.expires.mon

  local.get $card
  field.get #cc.expires.year

  local.get $card
  field.get #cc.ccv
  
  local.get $session
  resource-to-eqref @connection
  call $payWithCard_
  i32-to-enum boolean
)

(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```

### Import

In this example, we assume that the credit details are passed to the import by
reference; i.e., the actual argument representing the credit card information is
a pointer to a block of memory.

```
(func $payWithCard_ (import ("" "payWithCard_"))
  (param i32 eqref) (result i32)

(@interface func $payWithCard (import "" "payWithCard")
  (param @cc (resource @connection))
  (result boolean))
  
(@interface implement (import "" "payWithCard_")
  (param $cc i32)
  (param $conn eqref)
  (result i32)

  local.get $cc
  i64.load {offset #cc.ccNo}
  i64-to-u64

  local.get $cc
  i32.load {offset #cc.name.ptr}

  local.get $cc
  i32.load {offset #cc.name.len}
  memory-to-string "memi"
  
  local.get $cc
  i16.load_u {offset #cc.expires.mon}

  local.get $cc
  i16.load_u {offset #cc.expires.year}

  create (record (mon u16) (year u16))
  
  local.get $cc
  i16.load_u {offset #cc.ccv}

  create @cc
  
  local.get $conn
  eqref-to-resource @connection
  
  call $payWithCard
  enum-to-i32 boolean
)
```

### Adapter

Combining the import, exports and distributing the arguments, renaming `cc` for
clarity, we initially get:

```
(@adapter implement (import "" "payWithCard_")
  (param $cc i32)
  (param $conn eqref)
  (result i32)
  
  local.get $cc
  i64.load {offset #cc.ccNo}
  i64-to-u64

  local.get $cc
  i32.load {offset #cc.name.ptr}

  local.get $cc
  i32.load {offset #cc.name.len}
  memory-to-string "memi"
  
  local.get $cc
  i16.load_u {offset #cc.expires.mon}

  local.get $cc
  i16.load_u {offset #cc.expires.year}

  create (record (mon u16) (year u16))
  
  local.get $cc
  i16.load_u {offset #cc.ccv}

  create @cc
  
  local.get $conn
  eqref-to-resource @connection

  let $session (resource @connection)
  let $card @cc

  local.get $card
  field.get #cc.ccNo   ;; access ccNo
  u64-to-i64

  local.get $card
  field.get #cc.name
  string-to-memory $mem1 "malloc"
  
  local.get $card
  field.get #cc.expires.mon
  u16-to-i32

  local.get $card
  field.get #cc.expires.year
  u16-to-i32

  local.get $card
  field.get #cc.ccv
  u16-to-i32
  
  local.get $session

  resource-to-eqref @connection
  call $payWithCard_
  i32-to-enum boolean
  end
  end
  enum-to-i32 boolean
)
```

With some assumptions (such as no aliasing, no writing to locals, no
re-entrancy), we can propagate and inline the definitions of intermediates. This
amounts to 'regular' inlining where we recurse into records and treat the fields
of the record in an analogous fashion to arguments to the call.

```
(@adapter implement (import "" "payWithCard_")
  (param $cc i32)
  (param $conn eqref)
  (result i32)
  
  local.get $cc
  i64.load {offset #cc.ccNo}

  local.get $cc
  i32.load {offset #cc.name.ptr}

  local.get $cc
  i32.load {offset #cc.name.len}
  string.copy Mi:"mem2" Mx:"memx" "malloc"
  
  local.get $cc
  i16.load_u {offset #cc.expires.mon}

  local.get $cc
  i16.load_u {offset #cc.expires.year}

  local.get $cc
  i16.load_u {offset #cc.ccv}
  
  local.get $conn
  call $payWithCard_
)
```

There would be different sequences for the adapter if the underlying ABI were different --
for example, if structured data were passed as a pointer to a local copy for
example.

## Paint a vector of points

In this example we look at how sequences are passed into an API. The signature
of `vectorPaint` is assumed to be:

```
point ::= pt(i32,i32)
vectorPaint:(array<point> pts) => returnCode
```

### Export

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


### Import

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

### Adapter

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


## Directory Listing

In this example, we look as an API to generate a list of files in a directory. 

The interface type signature for this function is:

```
listing:(string)=>sequence<string>
```

Since the caller does not know how many file names will be returned, it has to
protect itself from potentially abusive situations. That in turn means that the
allocation of the string sequence in the return value may fail.

In order to avoid memory leaks, we protect the adapter with exception handling
-- whose purpose it to clean up should an allocation failure occur

In addition, memory allocated by the internal implementation of `$listing_`
needs to be released after the successful call.

### Export

```
(memory (export "memx") 1)

(@interface func (export "listing")
  (param $dir string)
  (result (sequence string))
  
  local.get $dir
  string-to-memory "memx" "malloc"
  call $it_opendir_
  
  iterator.start
    sequence.start string
    iterator.while $it_loop
      call $it_readdir_
      dup
      eqz
      br_if $it_loop
      memory-to-string "memx"
      sequence.append
    end
    iterator.close
      call $it_closedir
      sequence.complete
    end
  end
)
...

```

The `sequence.start`, `sequence.append` and `sequence.complete` instructions are
used to signal the creation of a sequence of values.

In this particular example, the export adapter is not simply exporting an
individual function but is packaging a combination of three functions that,
together, implement the desired interface. This is an example of a situation
where the C/C++ language is not itself capable of realizing a concept available
in the interface type schema.

The `$it_opendir_`, `$it_readdir_` and `$it_closedir_` functions are intended to
denote variants of the standard posix functions that have been slightly tailored
to better fit the scenario.

The `iterator.start`, `iterator.while` and `iterator.close` instructions model
the equivalent of a `while` loop. The body of the `iterator.start` consists of
three subsections: the initialization phase, an `iterator.while` instruction
which embodies the main part of the iteration and the `iterator.close` whose
body contains instructions that must be performed at the end of the loop.

The `iterator.while` instruction repeats its internal block until specifically
broken out of; it is effectively equivalent to the normal wasm `loop`
instruction.

### Import

Consuming a sequence 

```

```
