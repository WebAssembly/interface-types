# Interface instructions

This proposal defines new instructions that operate over interface value types. These instructions have the same form as core instructions; they push and pop values on an execution stack and support single pass validation.

The new interface instructions are summarized below.

```
#interface-instr ::=
  | sNN.from_iMM
  | iMM.from_sNN

  | uNN.from_iMM
  | iMM.from_uNN

  | string.size ..
  | string.lift_memory ..
  | string.lower_memory ..

  | record.lift ..
  | record.lower ..

  | array.count
  | array.lift ..
  | array.lower ..

  | call_export ..
  | call_adapter ..

where:
  NN is one of [8, 16, 32, 64]
  MM is one of [32, 64]
```

In addition the following instructions are included from the core instructions:

```
#interface-instr ::=
  | ..
  | <numeric instructions>
  | <memory instructions>
  | <table instructions>
  | drop
  | unreachable
  | nop
  | call ..
  | call_indirect ..
```

Refer to the definition of instruction classes [here](https://webassembly.github.io/spec/core/syntax/instructions.html).

And finally, the following instruction is included from the typed function references proposal:

```
#interface-instr ::=
  | ..
  | let ..
```

## Failure

The current definitions of instructions handles error conditions by 'failing'. This is currently defined to be a core wasm trap.

This is a temporary solution and may be replaced in the future with something more comprehensive.

## `sNN.from_iMM`

This instruction has type `iMM` -> `sNN`.

The output is the value of `iMM` as a two's complement signed integer.

If this cannot be represented in the desired type, then this instruction `fail`s.

## `iMM.from_sNN`

This instruction has type `[sNN] -> [iMM]`

The output is the value of `sNN` as a two's complement signed integer.

If this cannot be represented in the desired type, then this instruction `fail`s.

## `uNN.from_iMM`

This instruction has type `iMM` -> `uNN`.

The output is the value of `iMM` as an unsigned integer.

If this cannot be represented in the desired type, then this instruction `fail`s.

## `iMM.from_uNN`

This instruction has type `[uNN] -> [iMM]`

The output is the value of `uNN` as an unsigned integer.

If this cannot be represented in the desired type, then this instruction `fail`s.

## `string.size`

The full syntax of this instruction is:
```
string.size #encoding
```

The type of this instruction is:
```
[string] -> [i32]
```

The output is the number of bytes needed to represent the input string in `#encoding`.

The syntax for supported encodings is:
```
#encoding ::=
  | utf8
```

Currently, only UTF8 is supported as an encoding. UTF16 may be added in the future.

## `string.lift_memory`

The full syntax of this instruction is:
```
string.lift_memory #memidx #encoding
```

The type of this instruction is:
```
[$base: i32, $len: i32] -> [string]
```

The output is a sequence of unicode code points taken by interpreting `store.memory[$memidx].bytes[$base .. $base + $len]` with `#encoding`.

This instructions `fail`s if the range `$base .. $base + $len` is out-of-bounds, or the byte sequence is not valid under `#encoding`.

## `string.lower_memory`

The full syntax of this instruction is:
```
string.lower_memory #memidx #encoding
```

The type of this instruction is:
```
[$base: i32, string] -> []
```

The instruction will encode the string with `#encoding` into the memory specified by `#memidx` at the offset `$base`.

## `record.lift`

The full syntax of this instruction is:
```
record.lift #typeidx
```

The type of this instruction is:
```
[$a, $b, ..] -> [$type]

Where:
  * $type is a type of the form (record $fields) taken from #typeidx
  * $a, $b, .. are the #interface-valtype's for each field in $fields
```

The output is a record interface value with fields given by the inputs.

## `record.lower`

The full syntax of this instruction is:
```
record.lower #typeidx
```

The type of this instruction is:
```
[$type] -> [$fields]

Where:
  * $type is a valid type of the form (record $fields) taken from #typeidx
```

The outputs are the fields of the input `record`.

## `array.count`

The full syntax of this instruction is:
```
array.count #typeidx
```

The type of this instruction is:
```
[$type] -> [i32]

Where:
  * $type is a type of the form (array ..) taken from #typeidx
```

The output of this instruction is the number of interface values contained in
the input array.

## `array.lift`

This is a structured instruction with the following syntax:
```
array.lift #typeidx $stride: #s32
  interface-instr*
end
```

The type of this instruction is:
```
[$base: i32, $count: i32] -> [$type]

Where:
  * $type is a type of the form (array $inner) taken from #typeidx
  * #interface-instr* is valid with type [i32] -> [$inner]
```

This instruction will execute the following steps:
```
  1. Let $offsets = take([$base, $base + $stride, $base + $stride * 2, ..], $count)
  2. Map #interface-instr* upon $offsets to yield the elements of the array
```

## `array.lower`

This is a structured instruction with the following syntax:
```
array.lower #typeidx $stride: #s32
  interface-instr*
end
```

The type of this instruction is:
```
[$base: i32, $type] -> []

Where:
  * $type is a valid type of the form (array $inner) taken from #typeidx
  * #interface-instr* is valid with type [i32, $inner] -> []
```

This instruction will execute the following steps:
```
  1. Let $elems be the elements of the array
  2. Let $count be the number of elements in the array
  3. Let $offsets = take([$base, $base + $stride, $base + $stride * 2, ..], $count)
  4. Let $params = zip($offsets, $elems)
  5. Map #interface-instr* upon $params
```

## `call_export`

The full syntax of this instruction is:
```
call_export #name
```

The type of this instruction is:
```
[$a, $b, ..] -> [$x, $y, ..]

Where:
  * #name refers to an export of a valid core wasm function
  * $a, $b, .. are the parameters of the core wasm function
  * $x, $y, .. are the results of the core wasm function
```

The output of this function is the result of calling the core wasm function
given by export name.

## `call_adapter`

The full syntax of this instruction is:
```
call_adapter #interface-funcidx
```

The type of this instruction is:
```
[$a, $b, ..] -> [$x, $y, ..]

Where:
  * #interface-funcidx refers to a valid interface-functype
  * $a, $b, .. are the parameters of the interface-functype
  * $x, $y, .. are the results of the interface-functype
```

Calling an interface function is effectively the same as inlining the callee interface function in the caller's scope. Interface value params flow directly into a called function and the resulting interface values flow directly back into the caller.

`call_adapter` may not be used recursively. This is statically prevented by only allowing a `call-adapter` to a strictly lesser interface-funcidx.
