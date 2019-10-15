# Two Argument Integer Function

Calling a two argument integer function, should result in effectively zero code.

## Export

```
(@interface func (export "twizzle")
  (param $a1 s32)(param $a2 s32) (result s32)
  local.get $a1
  s32-to-i32
  local.get $a2
  s32-to-i32
  call $twizzle_
  i32-to-s32
)
```

## Import

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

## Adapter Code

The adapter code, that maps the import of `twozzle_` to its implementation as
`twizzle_` is:

```
(@adapter implement (import "" "twozzle_")
  (param $b1 i32)(param $b2 i32) (result i32)
    local.get $b1
    local.get $b2
    call $Mx:twizzle_
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
        s32-to-i32
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
      s32-to-i32
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
