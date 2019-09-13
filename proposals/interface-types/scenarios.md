# A suite of adapter scenarios

## Notes
Added let, func.bind, env.get, string.copy instructions.

Changed arg.get to lcl.get because it is clearer when combined with let.

## Two Argument Integer Function

Calling a two argument integer function, should result in effectively zero code.

### Export

```
(@interface func (export "twizzle")
  (param $a1 int32)(param $a2 int32) (result int32)
  lcl.get $a1
  lcl.get $a2
  call "twizzle_"
)
```

### Import

```
(@interface func (import "" "twozzle")
  (param $a1 int32)(param $a2 int32) (result int32)
)
(@interface implement (import "" "twozzle_")
  (param $b1 int32)(param $b2 int32) (result int32)
  lcl.get $b1
  lcl.get $b2
  call-import "twozzle"
)

```

### Adapter Code

```
(@adapter M2:"twozzle" as M1:"twizzle"
  (param $a1 int32)(param $a2 int32) (result int32)
    lcl.get $a1
    lcl.get $a2
    call M1:"twizzle_"
)
```

This should be viewed as the result of optimizations over an in-line substitution:
```
(@adapter M2:"twozzle" as M1:"twizzle"
  (param $b1 int32)(param $b2 int32) (result int32)
    lcl.get $b1
    lcl.get $b2
    let $a1 $a2
    lcl.get $a1
    lcl.get $a2
    call M1:"twizzle_"
)
```
The `let` pseudo instruction pops elements off the stack and gives them names.

Inlining the above example, eliminating the combination `lcl.get $b2`; `let $a2` by rewriting `$b2` with `$a2`:
```
(@adapter M2:"twozzle" as M1:"twizzle"
  (param $b1 int32)(param $a2 int32) (result int32)
    lcl.get $b1
    let $a1
    lcl.get $a1
    lcl.get $a2
    call M1:"twizzle_"
)
```
and again for `$b1/$a1` give the result:
```
(@adapter M2:"twozzle" as M1:"twizzle"
  (param $a1 int32)(param $a2 int32) (result int32)
    lcl.get $a1
    lcl.get $a2
    call M1:"twizzle_"
)
```

Below, we will assume that this transformation is applied automatically; except
where we need to show what happens more clearly.

## Counting Unicodes

Passing a string and returning the number of unicode code points in it.

### Export

```
(memory (export "mem1") 1)
(@interface func (export "count")
  (param $str string) (result i32)
  lcl.get $str
  string-to-memory "mem1" "malloc"
  call "count_"
)
(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```

### Import

```
(memory (export "mem2" 1)
(func $count_ (import ("" "count_"))
  (param i32 i32) (result i32))
(@interface func $count (import "" "count")
  (param $str string) (result i32))
(@interface implement (import "" "count_")
  (param $ptr i32 $len i32) (result i32))
  lcl.get $ptr
  lcl.get $len
  memory-to-string "mem2"
  call-import $count
)
```

### Adapter code

Note that the type of the adapter is in terms of wasm core types.

```
(@adapter M1:"count" as M2:"count"
  (param $ptr i32 $len i32) (result i32))
  lcl.get $ptr
  lcl.get $len
  memory-to-string M2:"mem2"
  string-to-memory M1:"mem1" M1:"malloc"
  call M2:"count_"
)
```
which, after collapsing coercion operators, becomes:
```
(@adapter M1:"count" as M2:"count"
  (param $ptr i32 $len i32) (result i32))
  lcl.get $ptr
  lcl.get $len
  string.copy M2:"mem2" M1:"mem1" M1:"malloc"
  call M2:"count_"
)
```

This assumes that `string.copy` combines memory allocation, string copy and
returns the new address and repeats the size of the string.

## Getenv Function

The `getenv` function builds on `count` by also returning a `string`.

### Export


```
(memory (export "mem1") 1)
(@interface func (export "getenv")
  (param $str string) (result string)
  lcl.get $str
  string-to-memory "mem1" "malloc"
  call "getenv_"
  memory-to-string "mem1"
)
(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```

### Import

The importer must also export `malloc` so that the returned string can be
stored.

```
(memory (export "mem2" 1)
(func $getenv_ (import ("" "getenv_"))
  (param i32 i32) (result i32 i32))
(@interface func $getenv (import "" "getenv")
  (param $str string) (result string))
(@interface implement (import "" "getenv_")
  (param $ptr i32 $len i32) (result i32))
  lcl.get $ptr
  lcl.get $len
  memory-to-string "mem2"
  call-import $getenv
  string-to-memory "mem2" "malloc"
)

(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```


### Adapter

```
(@adapter M1:"getenv" as M2:"getenv"
  (param $ptr i32 $len i32) (result i32 i32))
  lcl.get $ptr
  lcl.get $len
  memory-to-string M2:"mem2"
  string-to-memory M1:"mem1" M1:"malloc"
  call M1:"getenv_"
  memory-to-string M1:"mem1"
  string-to-memory M2:"mem2" M2:"malloc"
)
```

which, after collapsing coercions, becomes

```
(@adapter M1:"getenv" as M2:"getenv"
  (param $ptr i32 $len i32) (result i32 i32))
  lcl.get $ptr
  lcl.get $len
  string.copy M2:"mem2" M1:"mem1" M1:"malloc"
  call M1:"getenv_"
  string.copy M1:"mem1" M2:"mem2" M2:"malloc"
)
```

## Fetch Url

The `fetch` function includes a callback which is invoked whenever 'something
happens' on the session. 

It also introduces two new enumeration types: `ReturnCode` and `StatusCode`.

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

(memory (export "mem1") 1)

(@interface func (export "fetch")
  (param $u string 
         $cb (func (param $status statusCode $text string) (result returnCode)))
  (result string)
  lcl.get $u
  string-to-memory "mem1" "malloc"
  
  lcl.get $cb
  func.bind (env $ecb (param $status statusCode $text string) (result returnCode))
    (func
      (param $status i32 $txt i32 $len i32)
      (result i32)
     lcl.get $status
     int23-to-enum @statusCode
     lcl.get $txt
     lcl.get $len
     memory-to-string "mem1"
     env.get $ecb
     call-indirect
     enum-to-int32 @returnCode
    )
   call $fetch_
   int32-to-enum @returnCode
)

(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)

```

### Import

Importing `fetch` implies exporting the function that implements the callback.

```
(memory (export "mem2" 1)

(func $fetch_ (import ("" "fetch_"))
  (param i32 i32 (func (param i32 i32)(result i32))) (result i32))

(@interface func $fetch (import "" "fetch")
  (param string (func (param @statusCode string) (result @returnCode)))
  (result @returnCode))
  
(@interface implement (import "" fetch_")
  (param $url i32 $len i32 $callb (func (param i32 i32)(result i32))) (result i32)
  lcl.get $url
  lcl.get $len
  memory-to-string "mem2"
  lcl.get $callb
  func.bind (env $callbk (func (param i32 i32 i32) (result i32)))
    (func
      (param $status statusCode $text string) (result returnCode)
      lcl.get $status
      enum-to-int32 @statusCode
      lcl.get $text
      string-to-memory "mem2" "malloc"
      env.get $callbk
      call-indirect
      int32-to-enum @returnCode
    )
    
   call $fetch
   enum-to-int32 @returnCode
)

(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```

### Adapter

Combining the export and import functions leads, in the first instance, to:

```
(@adapter M1:"fetch" as M2:"fetch"
  (param $url i32 $len i32 $callb (func (param i32 i32)(result i32))) (result i32)
  lcl.get $url
  lcl.get $len
  memory-to-string "mem2"
  lcl.get $callb
  func.bind (env $callbk (func (param i32 i32 i32) (result i32)))
    (func
      (param $status statusCode $text string) (result returnCode)
      lcl.get $status
      enum-to-int32 @statusCode
      lcl.get $text
      string-to-memory "mem2" "malloc"
      env.get $callbk
      call-indirect
      int32-to-enum @returnCode
    )
    
  let $cb
  let $u

  lcl.get $u
  string-to-memory "mem1" "malloc"
  lcl.get $cb
  func.bind (env $ecb (param $status statusCode $text string) (result returnCode))
    (func
      (param $status i32 $txt i32 $len i32)
      (result i32)
     lcl.get $status
     int23-to-enum @statusCode
     lcl.get $txt
     lcl.get $len
     memory-to-string "mem1"
     env.get $ecb
     call-indirect
     enum-to-int32 @returnCode
    )
   call $fetch_
   int32-to-enum @returnCode
   enum-to-int32 @returnCode
)
```

Some reordering, (which is hard to sanction if we allow side-affecting
instructions), gives:

```
(@adapter M1:"fetch" as M2:"fetch"
  (param $url i32 $len i32 $callb (func (param i32 i32)(result i32))) (result i32)
  lcl.get $url
  lcl.get $len
  memory-to-string "mem2"
  let $u

  lcl.get $u
  string-to-memory "mem1" "malloc"
  
  lcl.get $callb
  func.bind (env $callbk (func (param i32 i32 i32) (result i32)))
    (func
      (param $status statusCode $text string) (result returnCode)
      lcl.get $status
      enum-to-int32 @statusCode
      lcl.get $text
      string-to-memory "mem2" "malloc"
      env.get $callbk
      call-indirect
      int32-to-enum @returnCode
    )
    
  let $cb
  lcl.get $cb
  func.bind (env $ecb (param $status statusCode $text string) (result returnCode))
    (func
      (param $status i32 $txt i32 $len i32)
      (result i32)
     lcl.get $status
     int23-to-enum @statusCode
     lcl.get $txt
     lcl.get $len
     memory-to-string "mem1"
     env.get $ecb
     call-indirect
     enum-to-int32 @returnCode
    )
   call $fetch_
   int32-to-enum @returnCode
   enum-to-int32 @returnCode
)

```

Simplifying low-hanging sequences for clarity:

```
(@adapter M1:"fetch" as M2:"fetch"
  (param $url i32 $len i32 $callb (func (param i32 i32)(result i32))) (result i32)
  lcl.get $url
  lcl.get $len
  string.copy M2:"mem2" M1:"mem1" "malloc"
  
  lcl.get $callb
  func.bind (env $callbk (func (param i32 i32 i32) (result i32)))
    (func
      (param $status statusCode $text string) (result returnCode)
      lcl.get $status
      enum-to-int32 @statusCode
      lcl.get $text
      string-to-memory "mem2" "malloc"
      env.get $callbk
      call-indirect
      int32-to-enum @returnCode
    )
    
  func.bind (env $ecb (param $status statusCode $text string) (result returnCode))
    (func
      (param $status i32 $txt i32 $len i32)
      (result i32)
     lcl.get $status
     int23-to-enum @statusCode
     lcl.get $txt
     lcl.get $len
     memory-to-string "mem1"
     env.get $ecb
     call-indirect
     enum-to-int32 @returnCode
    )
   call $fetch_
)

```

The double binds in a row can be combined by inlining the call to `env.get $ecb`:

```
(@adapter M1:"fetch" as M2:"fetch"
  (param $url i32 $len i32 $callb (func (param i32 i32)(result i32))) (result i32)
  lcl.get $url
  lcl.get $len
  string.copy M2:"mem2" M1:"mem1" "malloc"
  
  lcl.get $callb
    
  func.bind (env $callbk (func (param i32 i32 i32) (result i32)))
    (func
      (param $status i32 $txt i32 $len i32)
      (result i32)
     lcl.get $status
     int23-to-enum @statusCode
     lcl.get $txt
     lcl.get $len
     memory-to-string "mem1"
     
     let $status $text

     lcl.get $status
     enum-to-int32 @statusCode
     lcl.get $text
     string-to-memory "mem2" "malloc"
     env.get $callbk
     call-indirect
     int32-to-enum @returnCode
     enum-to-int32 @returnCode
    )
   call $fetch_
)

```
More reordering and simplification:

```
(@adapter M1:"fetch" as M2:"fetch"
  (param $url i32 $len i32 $callb (func (param i32 i32)(result i32))) (result i32)
  lcl.get $url
  lcl.get $len
  string.copy M2:"mem2" M1:"mem1" "malloc"
  
  lcl.get $callb
    
  func.bind (env $callbk (func (param i32 i32 i32) (result i32)))
    (func
      (param $status i32 $txt i32 $len i32)
      (result i32)
     lcl.get $status

     lcl.get $txt
     lcl.get $len
     string.copy M1:"mem1" M2:"mem2" "malloc" 

     env.get $callbk
     call-indirect
    )
   call $fetch_
)

```

Which is reasonable code.

## Pay by Credit Card

In order to process a payment with a credit card, various items of information
are required: credit card details, amount, merchant bank, and so on. This
example illustrates the use of nominal types and record types in adapter code.

>Note that we have aggressively simplified the scenario in order to construct an
>illustrative example.

## Paint a vector of points

## Colors

## Directory Listing
