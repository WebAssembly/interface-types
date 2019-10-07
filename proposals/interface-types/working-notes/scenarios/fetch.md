# Fetch Url

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

## Export

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

## Import

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

## Adapter

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
