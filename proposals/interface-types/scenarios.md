# A suite of adapter scenarios

This suite of examples is intended to demonstrate a spanning set of examples and
transformations that permit the use of interface types to specify imports and
exports.

## Notes
Added let, func.bind, env.get, field.get, string.copy, create instructions.

Changed arg.get to lcl.get because it is clearer when combined with let.

## Two Argument Integer Function

Calling a two argument integer function, should result in effectively zero code.

### Export

```
(@interface func (export "twizzle")
  (param $a1 s32)(param $a2 s32) (result s32)
  lcl.get $a1
  s32-to-i23
  lcl.get $a2
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
  lcl.get $b1
  i32-to-s32
  lcl.get $b2
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
    lcl.get $b1
    lcl.get $b2
    call Mx:"twizzle_"
)
```

>Note: we adopt the convention that the `Mx:` prefix refers to the exporting
>module and `Mi:` refers to the importing module.

This should be viewed as the result of optimizations over an in-line substitution:
```
(@adapter implement (import "" "twozzle_")
  (param $b1 i32)(param $b2 i32) (result i32)
    lcl.get $b1
    i32-to-s32
    lcl.get $b2
    i32-to-s32
    let s32 (local $a2 s32)
      let s32 (local $a1 s32)
        lcl.get $a1
        s32-to-i23
        lcl.get $a2
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
lcl.get $b1
i32-to-s32
```

so, removing the `let` for `a2`, and replacing `lcl.get $a2` with its defining
subsequence gives:

```
(@adapter implement (import "" "twozzle_")
  (param $b1 i32)(param $b2 i32) (result i32)
    lcl.get $b1
    i32-to-s32
    lcl.get $b2
    i32-to-s32
    let s32 (local $a2 s32)
      lcl.get $b1
      i32-to-s32
      s32-to-i23
      lcl.get $a2
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
    lcl.get $b2
    i32-to-s32
    let s32 (local $a2 s32)
      lcl.get $b1
      lcl.get $a2
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
    lcl.get $b1
    lcl.get $b2
    call Mx:"twizzle_"
    i32-to-s32
    s32-to-i32
)
```

with the final removal of the redundant coercion pair at the end:

```
(@adapter implement (import "" "twozzle_")
  (param $b1 i32)(param $b2 i32) (result i32)
    lcl.get $b1
    lcl.get $b2
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
  lcl.get $str
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
  lcl.get $ptr
  lcl.get $len
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
  lcl.get $ptr
  lcl.get $len
  memory-to-string Mi:"memi"
  string-to-memory Mx:$memx Mx:"malloc"
  call Mx:"countCodes_"
)
```
which, after collapsing coercion operators, becomes:
```
(@adapter implement (import "" "countCodes_")
  (param $ptr i32 $len i32) (result i32))
  lcl.get $ptr
  lcl.get $len
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
  lcl.get $str
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
  lcl.get $ptr
  lcl.get $len
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
  lcl.get $ptr
  lcl.get $len
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
  lcl.get $u
  string-to-memory "memx" "malloc"
  
  lcl.get $cb
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
  lcl.get $status
  i32-to-enum @statusCode
  lcl.get $text
  lcl.get $len
  memory-to-string "memx"
  lcl.get $ecb
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
  lcl.get $url
  lcl.get $len
  memory-to-string "memi"
  lcl.get $callb
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
  lcl.get $status
  enum-to-i32 @statusCode
  lcl.get $text
  string-to-memory "memi" "malloc"
  lcl.get $callbk
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
  lcl.get $url
  lcl.get $len
  memory-to-string "memi"
  lcl.get $callb
  func.ref $cbEntry
  func.bind (func (param i32 i32 i32) (result i32))
  
  let ($u string) ($cb (ref (func (param i32 i32 i32) (result i32))))
  lcl.get $u
  string-to-memory "memx" "malloc"
  
  lcl.get $cb
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
  lcl.get $url
  lcl.get $len
  string.copy Mi:"memi" Mx:"memx" Mx:"malloc"

  lcl.get $callb
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
  lcl.get $status

  lcl.get $text
  lcl.get $len
  string.copy Mx:"memx" Mi:"memi" Mi:"malloc"

  lcl.get $callbk
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
  lcl.get $url
  lcl.get $len
  string.copy Mi:"memi" Mx:"memx" Mx:"malloc"

  lcl.get $callb
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

(memory (export $mem1) 1)

(func $payWithCard_ (export ("" "payWithCard_"))
  (param i64 i32 i32 i16 i16 eqref) (result i32)

(@interface func (export "payWithCard")
  (param $card @cc)
  (param $session (resource @connection))
  (result boolean)
  lcl.get $card
  field.get #cc.ccNo   << access ccNo

  lcl.get $card
  field.get #cc.name
  string-to-memory $mem1 "malloc"
  
  lcl.get $card
  field.get #cc.expires.mon

  lcl.get $card
  field.get #cc.expires.year

  lcl.get $cc
  field.get #cc.ccv
  
  lcl.get $session
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
  lcl.get $cc
  i64.load #cc.ccNo
  i64-to-u64

  lcl.get $cc
  i32.load #cc.name.ptr

  lcl.get $cc
  i32.load #cc.name.len
  memory-to-string "mem2"
  
  lcl.get $cc
  i16.load #cc.expires.mon

  lcl.get $cc
  i16.load #cc.expires.year

  create (record (mon u16) (year u16))
  
  lcl.get $cc
  i16.load #cc.ccv

  create @cc
  
  lcl.get $conn
  
  call $payWithCard
  enum-to-i32 boolean
)
```

### Adapter

Combining the import, exports and distributing the arguments, renaming `cc` for
clarity, we initially get:

```
(@adapter M1:"payWithCard" as M2:"payWithCard"
  (param $cc i32 $conn eqref)(result i32)
  
  lcl.get $cc
  i64.load mem2:#cc.ccNo

  lcl.get $cc
  i32.load mem2:#cc.name.ptr

  lcl.get $cc
  i32.load mem2:#cc.name.len
  memory-to-string "mem2"
  
  lcl.get $cc
  i16.load mem2:#cc.expires.mon

  lcl.get $cc
  i16.load mem2:#cc.expires.year

  create (record (mon int16) (year int16))
  
  lcl.get $cc
  i16.load mem2:#cc.ccv

  create @cc
  
  lcl.get $conn
  let $session
  let $card

  lcl.get $card
  field.get #cc.ccNo   << access ccNo

  lcl.get $card
  field.get #cc.name
  string-to-memory $mem1 "malloc"
  
  lcl.get $card
  field.get #cc.expires.mon

  lcl.get $card
  field.get #cc.expires.year

  lcl.get $cc
  field.get #cc.ccv
  
  lcl.get $session
  call $payWithCard_
  int-to-enum boolean
  enum-to-i32 boolean
)
```

With some assumptions (such as no aliasing, no writing to locals, no
re-entrancy), we can propagate and inline the definitions of intermediates:

```
(@adapter M1:"payWithCard" as M2:"payWithCard"
  (param $cc i32 $conn eqref)(result i32)
  lcl.get $cc
  i64.load mem2:#cc.ccNo

  lcl.get $cc
  i32.load mem2:#cc.name.ptr

  lcl.get $cc
  i32.load mem2:#cc.name.len
  string.copy mem2 mem1 "malloc"
  
  lcl.get $cc
  i16.load mem2:#cc.expires.mon

  lcl.get $cc
  i16.load mem2:#cc.expires.year
  
  lcl.get $cc
  i16.load mem2:#cc.ccv
  
  lcl.get $conn
  call $payWithCard_
)
```

## Paint a vector of points

In this example we look at how sequences are passed into an API. The signature
of `vectorPaint` is assumed to be:

```
point ::= pt(i32,i32)
vectorPaint:(sequence<point> pts) => returnCode
```

### Export

It is assumed that the implementation of `vectorPaint_` requires a
memory-allocated array; each entry of which consists of two contiguous `i32`
values.

The array is modeled as two values: a pointer to its base and the number of
elements in the array.

The primary pattern with processing sequences is the iterator pattern; which is
modeled by the `for` instruction; which iterates a code fragment over a
sequence.

```
(@interface datatype @point
  (oneof
    (variant "pt"
      (tuple i32 i32))))

(memory (export "mem1") 1)

(func $vectorPaint_ (export ("" "vectorPaint_"))
  (param i32 i32) (result i32)

(@interface func (export "vectorPaint")
  (param $pts (sequence @point))
  (result @returnCode)
  
  lcl.get $pts
  sequence.count
  allocate scale:8 "malloc"
  let array
  lcl.get $pts
  (for $ix $pt ; for ix,pt in pts
    lcl.get $pt
    field.get #0
    lcl.get $array
    lcl.get $ix
    i32.index.store #0
    lcl.get $pt
    field.get #1
    lcl.get $array
    lcl.get $ix
    i32.index.store #4
  )
  lcl.get $array
  call $vectorPaint_
  i32-to-enum @returnCode
)
```

### Import

The primary task in passing a vector of values is the construction of a `sequence`. 

```
(func $vectorPaint_ (import ("" "vectorPaint_"))
  (param i32 i32) (result i32)

(@interface func $vectorPaint (import "" "vectorPaint")
  (param @ptr (sequence @point))
  (result @returnCode))
  
(@interface implement (import "" "vectorPaint_")
  (param $points i32 $count i32)(result i32)
  lcl.get $points
  let $ptr
  sequence.start @point
  lcl.get $count
  loop-for $vector
    lcl.get $ptr
    i32.load #0
    lcl.get $ptr
    i32.load #1
    create @point
    sequence.append
    lcl.inc $ptr #sizeof(point)
    lcl.decr $mx
    br_if $vector
  end
  sequence.complete
  call $vectorPaint
  enum-to-i32 @returnCode
)
```

The `sequence.start`, `sequence.append` and `sequence.complete` instructions are
used to manage the generation of sequences. The `loop-for` pseudo instruction
facilitates the adaptation process, but is a slight generalization of the core
wasm `loop` pattern.

### Adapter

Combining the import and export sequences into an adapter code depends on being
able to fuse the generating loop with the iterating loop.

The initial in-line version gives:

```
(@adapter M1:"vectorPaint" as M2:"vectorPaint"
  (param $points i32 $count i32)(result i32)
  lcl.get $points
  let $ptr
  sequence.start @point
  lcl.get $count
  loop-for $vector
    lcl.get $ptr
    i32.load #0
    lcl.get $ptr
    i32.load #1
    create @point
    sequence.append
    lcl.inc $ptr #sizeof(point)
    lcl.decr $mx
    br_if $vector
  end
  sequence.complete
  
  let $pts
  lcl.get $pts
  sequence.count
  allocate scale:8 "malloc"
  let array
  lcl.get $pts
  (for $ix $pt ; for ix,pt in pts
    lcl.get $pt
    field.get #0
    lcl.get $array
    lcl.get $ix
    i32.index.store #0
    lcl.get $pt
    field.get #1
    lcl.get $array
    lcl.get $ix
    i32.index.store #4
  )
  lcl.get $array
  call $vectorPaint_
  i32-to-enum @returnCode
  enum-to-i32 @returnCode
)
```


## Colors

Packed values

## Directory Listing

Error recovery


