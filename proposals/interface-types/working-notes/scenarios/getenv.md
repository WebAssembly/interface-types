# Getenv Function

The `getenv` function builds on `countCodes` by also returning a `string`. Its
interface type signature is:

```
getenv:(string)=>string
```

## Export


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

## Import

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


## Adapter

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
