# Canonical ABI

This document proposes a "canonical ABI" that maps interface-typed function
signatures down to core wasm function signatures that can be imported or
exported by core wasm, without the need for custom adapter functions.
While the end goal of interface types is to divorce interface signatures from
the actual underlying representation of each type through custom adapter
functions, there's still a decent amount of remaining work to stabilize adapter
functions and so one goal of defining a "canonical ABI" is to allow us to define
an interface types MVP that can be released in the short term. In addition to
serving as a stepping stone, the canonical ABI will also be valuable in the long
term in providing a simple path for languages' tooling to adopt interface types
by binding to the canonical ABI like any normal C API and then using shared
language-agnostic tooling to wrap the core module into an interface-typed
adapter module.

The canonical ABI defines how interface types values are represented in core
wasm values and in linear memory. Additionally canonical methods of
[lifting](#lifting) and [lowering](#lowering) are provided to interpret these
values as interface types values. Lifting and lowering depends on the
*direction* of the call, whether it's an import or an export. An import in this
case could either be a core WebAssembly module calling an import or a host
defining an import. An export, similarly, could be a host calling a WebAssembly
module or a WebAssembly module defining an export.

With a canonical ABI it's expected that almost all of lifting/lowering
will be handled by code generators for target languages. For example if
you're compiling a Rust project to WebAssembly you'd use a code generator to
generate all the glue necessary to convert from Rust values to core wasm values,
according to the canonical ABI.

An expected example of the canonical ABI will be WASI and the functions that it
supports. The vision for how this might looks is:

1. WASI is first defined in terms of interface types. This means all of its
   functions will be only using types from the interface types specification.

2. When a WebAssembly module, say C or Rust, calls a WASI API then a code
   generator for the appropriate language will be used to have an API, in that
   language, which takes types of that language (e.g. Rust structs, C unions,
   etc) and then converts those values to the canonical ABI (according to this
   document). The WebAssembly module will have a core wasm import of a WASI function
   whose core signature and behavior is defined by this document.

3. On the host side of things, a JS host, for example, would use a code
   generator to generate imported JS glue functions which take and return core
   wasm values and, converting these to and from JS values, call the JS
   implementation of WASI. The JS WASI implementation would thus only deal with
   JS values and wouldn't need to interact with core wasm values or linear
   memory (due to the shared-nothing nature of interface types).

This document doesn't currently specify a binary encoding for what this might
all look like as an adapter module, that's left to a future document for now!

## Interface Types AST

The main thing to "stabilize" for interface types is the grammar for the types
themselves. For the canonical ABI the assumed type grammar of interface types
is:

```
intertype ::= f32 | f64
            | s8 | u8 | s16 | u16 | s32 | u32 | s64 | u64
            | char
            | string
            | (list <intertype>)
            | (flags <name>*)
            | (record (field <name> <id>? <intertype>)*)
            | (variant (case <name> <id>? <intertype>?)*)
            | (handle <resource>)
            | (push-buffer <intertype>)
            | (pull-buffer <intertype>)
```

This grammar includes a few types that have been tentatively discussed, but are
not yet present in the explainer. These types are necessary to complete a
fully interface-typed WASI and will be updated if they change upon addition to
the explainer.

* `string` and `flags` - while the explainer currently defines these as
  specializations of `(list char)` and `record`-of-`bool`, resp.,
  specializations are specifically allowed to have different representations in
  bindings (such as the canonical ABI) than the general type they specialize
  (e.g. `string` is utf-8 while `list char` is utf-32).

* `handle <resource>` - this is intended to be a handle to an abstract
  resource. The resource could be defined by the current module or a foreign
  module. This is intended to encapsulate, for example, a WASI file descriptor.
  This can be used, though, for any object exported from a wasm module or any
  object a wasm module imports from a host. Handles are intended to be
  un-forgeable capabilties where once a handle is given to a module it's seen as
  granting a capability to a module, but the module cannot have access to
  handles that it hasn't explicitly been granted access to. Handles are
  described in more detail in
  [this presentation to the WASI subgroup](https://docs.google.com/presentation/d/1ikwS2Ps-KLXFofuS5VAs6Bn14q4LBEaxMjPfLj61UZE).

* `push-buffer T` and `pull-buffer T` - these types are intended to represent
  abstract views into another wasm instance's memory. This is done in a
  structured way which respects the canonical ABI. The name implies the
  operation that can be applied to that buffer, for example a `push-buffer` can
  have items of type `T` pushed into it, and a `pull-buffer` can have items of
  type `T` removed from it. Buffers have optional capacities (although required
  in the canonical ABI). The main motivation for buffers is POSIX-like `read`
  and `write` APIs in WASI. The `read` function, for example, takes a
  `push-buffer u8` where bytes are pushed into the buffer, and an auxiliary
  channel (the return value of `read`) indicates how many bytes were read.
  Buffers are intended to behave similarly to `handle` where they cannot
  be forged, but unlike `handle` they are only a temporary reference
  which lives for the duration of a function call (and can only show up as
  paramters to functions). This enables efficient binding of APIs like `read`
  and `write` where the canonical ABI is simply passing a pointer/length.

With this grammar of interface types, we can now define what it means to lift
and lower to all these types.

## Core wasm signatures

First let's take a look at the canonical core wasm signature for any particular
interface types signature. Note that the canonical core wasm signatures for
interface types only use the wasm MVP types `i32`, `i64`, `f32`, and `f64` (not
the post-MVP types `externref`, `funcref`, and `v128`).

```
# Returns the core wasm signature for a function which takes `params` as
# arguments (interface types) and returns `results`. The `direction` here is
# either "import" or "export".
#
# Note the usage of a return pointer which is intended to be a temporary
# workaround for the lack of multi-value today. Languages like C and Rust don't
# have a great way of idiomatically working with multivalue returns just yet,
# but this is likely to change in the near future.
def wasm_signature(direction, params, results):
  wasm_params = params.flat_map(|t| flatten(direction, t))
  wasm_results = results.flat_map(|t| flatten(direction, t))

  # If there's more than one result work around the lack of multi-value
  # returns in C/Rust today. For imports the caller allocates a return pointer
  # and for exports the callee returns a location in memory where everything
  # was stored.
  if len(wasm_results) > 1:
    if direction == "import"
      wasm_params.insert(0, i32)
      wasm_results = []
    else:
      wasm_results = [i32]

  return wasm_params, wasm_results

def flatten(direction, interface_type):
  match interface_type
    f32 => [f32]
    f64 => [f64]
    s8 | s16 | s32 | u8 | u16 | u32 => [i32]
    s64 | u64 => [i64]
    char => [i32]

    # List are represented by their pointer/length
    string |
    list $ty => [i32, i32]

    # flags are represented as a contiguous list of `i32` values
    flags $fields => {
      fields = []
      for _ in 0..num_packed_i32s($fields):
        fields.push(i32)
      fields
    }

    # Records have their fields flattened individually
    record $fields => $fields.flat_map(|t| flatten(direction, t))

    # variants are represented with a discriminant plus whatever is the
    # smallest which fits all possible cases.
    variant $cases => [i32] + flatten_cases(direction, $cases)

    # Handles are represented as a single integer unconditionally. The
    # un-forgeability is discussed later.
    handle => [i32]

    # Buffers are represented as a pointer/length when passing to an import
    # and as an integer handle when receiving as part of an export.
    push-buffer | pull-buffer =>
      if direction == "import":
        [i32, i32, i32]
      else:
        [i32]

# Determines the core wasm types used to represent a variant's cases. This
# overlays various cases into the same list of types, so the length of types is
# the length of the longest case, and types are "unified" to a common
# representation to ensure there's enough bits to hold enough values for all
# cases.
def flatten_cases(direction, cases):
  result = []
  for case in cases:
    if case:
      tys = flatten(direction, case)
      for i, ty in enumerate(tys):
        if i < len(result):
          result[i] = unify(result[i], ty)
        else:
          result.push(ty)
  return result

def unify(a, b):
  match (a, b)
    (i64, _) |
    (_, i64) |
    (i32, f64) |
    (f64, i32) => i64,

    (i32, i32) |
    (i32, f32) |
    (f31, i32) => i32,

    (f32, f32) => f32,
    (f64, f64) | (f32, f64) | (f64, i32) => f64,

def num_packed_i32s(fields):
  return align_to(fields.len(), 32) / 32
```

The only difference in exports/imports currently is that `push-buffer` and
`pull-buffer` are represented with a singular `i32` instead of three `i32`
values. This is because if an export is receiving a buffer it's coming from
somewhere else which means it's a handle to a buffer and various "intrinsics",
listed below, will be used to operate on the handle.

## Lifting

The `lift` function here takes four parameters:

* `direction` - one of `"export"` or `"import"` which indicates for which
  purposes the lifting is happening: lifting an export means lifting
  a result, lifting an import means lifting a parameter.
* `src` - this is an instance where the value is being lifted from. This could
  be a WebAssembly instance or the host itself.
* `ty` - the interface type that we're lifting into
* `values` - the core wasm values iterator we have from the function signature
  or results that we're lifting from. Note that due to type-checking and the
  canonical ABI for function signatures it's guaranteed that `values` has the
  right types to lift into `ty`.

This defines how an interface types value is decoded from linear memory and
WebAssembly values. Note that this function is modeled as a generator. This is
used later during the definition of `fuse` to model how lifting/lowering is
interleaved.

```
def lift(direction, src, ty, values):
  match ty
    f32 => yield(values.assert_next(wasm_f32)),
    f64 => yield(values.assert_next(wasm_f64)),

    s8 => {
      val = values.assert_next(wasm_i32) as s32
      assert_or_trap(val >= -128 && val <= 127)
      yield(val as s8)
    }

    s16 => {
      val = values.assert_next(wasm_i32) as s32
      assert_or_trap(val >= -32768 && val <= 32767)
      yield(val as s16)
    }

    u8 => {
      val = values.assert_next(wasm_i32) as u32
      assert_or_trap(val >= 0 && val <= 255)
      yield(val as u8)
    }
    u16 => {
      val = values.assert_next(wasm_i32) as u32
      assert_or_trap(val >= 0 && val <= 65535)
      yield(val as u16)
    }

    s32 => yield(values.assert_next(wasm_i32) as s32),
    u32 => yield(values.assert_next(wasm_i32) as u32),
    s64 => yield(values.assert_next(wasm_i64) as s64),
    u64 => yield(values.assert_next(wasm_i64) as u64),

    char => {
      codept = values.assert_next(wasm_i32) as u32
      # generate a trap if `codept` is out of bounds
      assert_or_trap(codept < 0xD800 || (codpt >= 0xE000 && codept <= 0x10FFFF))
      yield(codept as char)
    }

    # Note that lists are only deallocated during lowering for exports. For
    # imports it's assumed the memory is still owned by the caller WebAssembly
    # module.
    #
    # Also note that `read_utf8_string` here is expected to validate that the
    # input read is indeed valid utf-8. One option is to trap on invalid utf-8,
    # but another option is to match `TextDecoder.decode` on the web with
    # replacement error mode which produces replacement characters.
    #
    # Currently it's expected that `read_utf8_string` will not trap and will
    # instead follow the `TextDecoder.decode` semantics which is what WTF-16
    # languages like JS/Java/C# will want.
    string => {
      ptr = values.assert_next(wasm_i32) as u32
      len = values.assert_next(wasm_i32) as u32
      assert_or_trap(ptr + len <= src.memory.len)

      val = read_utf8_string(src, ptr, len)
      if direction == "export":
        src.canonical_abi_free(ptr, len, 1)
      yield(val as string)
    }

    list<$ty> => {
      ptr = values.assert_next(wasm_i32) as u32
      len = values.assert_next(wasm_i32) as u32

      assert_or_trap(ptr + len * size(direction, $ty) <= src.memory.len)
      yield(list_start<$ty>(len))

      for i in 0..len:
        yield from lift_from_memory(direction, src, $ty, ptr + i * size(direction, $ty))
      if direction == "export":
        src.canonical_abi_free(ptr, len * size(direction, $ty), align($ty))
      yield(list_done<$ty>)
    }

    flags<$fields> => {
      result = {}
      for chunk in $fields.chunks(32):
        bits = values.assert_next(i32)
        for i, field in enumerate(chunk):
          result[field] = bits & (1 << i) != 0
      yield(result as flags<$fields>)
    }

    record<$fields> => {
      yield(record_start<$fields>)
      for $field in $fields:
        yield from lift(direction, src, values)
      yield(record_end<$fields>)
    }

    variant<$cases> => {
      discrim = values.assert_next(i32)
      assert_or_trap(discrim >= 0 && discrim < $cases.len())
      yield(variant_start<$cases>(discrim))
      match $cases[discrim] {
        payload => {
          # Consume all other values that this variant might consume
          payload_types = flatten_cases(direction, $cases)
          payload_values = []
          for ty in payload_types:
            payload_values.push(values.assert_next(ty))

          # If there a payload type for this case then we lift it here.
          # "downcast" the values to the right types for this payload and then
          # do the conversion
          if payload:
            my_types = flatten(direction, payload)
            payload_values.truncate(my_types.len())
            for i, ty in enumerate(my_types):
              payload_values[i] = downcast(payload_values[i], ty)
            yield from lift(direction, src, payload, payload_values)

          yield(variant_end<$cases>)
        }
      }
    }

    # Lifting a handle means that the module has given us an index into its
    # tables of resources. We consult the global TABLES helper to ensure that
    # the `val` given is indeed a valid resource for `src` of type `$resource`
    handle<$resource> => {
      val = values.assert_next(i32)
      yield(TABLES.get($resource, val, src) as handle<$resource>)

      # If we're lifting for an exported function then we're lifting the results
      # of the export after the call. Similar to lists the handle index
      # reference is owned by the callee (which just returned) but is a sort of
      # "temporary" allocation that the callee doesn't need. As a result, like
      # lists, we automatically remove the index from the source's index space.
      if direction == "export":
        TABLES.remove($resource, val, src)
    }

    # Buffers are only lifted from imports, which means that this only applies
    # to the parameters of imported functions. This means that the
    # representation we're dealing with is three i32 values.
    #
    # The first i32 is either 0 meaning that this is a new local buffer within
    # `src`, or it's a nonzero value meaning that it's a handle to a buffer
    # defined in some other module.
    #
    # If the first handle value is 0 then the offset parameter is in bytes from
    # the start of the memory of `src`. The `len` parameter is the length, in
    # units of $ty, of the buffer. The pointer must be well aligned.
    #
    # If the first handle value is not zero then TABLES.get_buffer will validate
    # that the handle is indeed a valid buffer for `src` to reference. The
    # `offset` and `len` parameters are in units of `$ty` relative to within the
    # original buffer. These values are validated to ensure they fit within the
    # original buffer
    push-buffer<$ty> | pull-buffer<$ty> => {
      assert(direction == "import")
      handle = values.assert_next(i32)
      offset = values.assert_next(i32)
      len = values.assert_next(i32)
      if handle == 0:
        assert_or_trap(offset + len * size(direction, $ty) <= src.memory.len)
        yield(Buffer(src, offset, len, ty))
      else:
        buffer = TABLES.get_buffer(handle, src, ty).clone()
        assert_or_trap(offset + len <= buffer.len)
        buffer.offset += offset * size(direction, $ty)
        buffer.len = len
        yield(buffer)
    }
```

There's a few more missing pieces to fill in, namely the in-memory
representation of types due to the representation of `list` as contiguous
elements in-memory. For this we can first define the `size` and `align` of all
types:

```
def size(direction, type):
  match type
    f32 => 4
    f64 => 8
    s8 => 1
    u8 => 1
    s16 => 2
    u16 => 2
    s32 => 4
    u32 => 4
    s64 => 8
    u64 => 8
    char => 4
    string |
    list $ty => size(record([i32, i32]))
    flags $fields => num_packed_i32s($fields) * 4
    record $fields =>
      s = 0
      for field in $fields
        s = align_to(s, align(field))
        s += size(direction, field)
      align_to(s, align(record $fields))
    variant $cases =>
      s = 0
      discrim = variant_discriminant($cases)
      for case in $cases
        if case:
          s = max(s, size(direction, record([discim, case])))
        else:
          s = max(s, size(direction, discrim))
      s
    handle => size(i32)
    push-buffer | pull-buffer =>
      if direction == "import":
        size(record([i32, i32, i32]))
      else:
        size(i32)

def align(type):
  match type
    f32 = 4
    f64 = 8
    s8 = 1
    u8 = 1
    s16 = 2
    u16 = 2
    s32 = 4
    u32 = 4
    s64 = 8
    u64 = 8
    char = 4
    string = 4
    list $ty = 4
    flags $fields = 4
    record $fields =
      a = 1
      for field in $fields
        a = max(a, align(field))
      a
    variant $cases =
      a = align(variant_discriminant($cases))
      for case in $cases
        if case:
          a = max(a, align(case))
      a
    handle => align(i32)
    push-buffer | pull-buffer => align(i32)

def variant_discriminant(cases):
  match cases.len() {
    0 => unreachable,
    n if n <= 1<<8 => i8,
    n if n <= 1<<16 => i16,
    n if n <= 1<<32 => i32,
    _ => i64,
  }
```

With those definitions in mind the way to lift types from memory is then
relatively straightforward:

```
def lift_from_memory(direction, src, ty, ptr):
  match ty
    f32 => yield(read_4_bytes(src, ptr) as f32),
    f64 => yield(read_8_bytes(src, ptr) as f64),

    s8 => yield(read_1_byte(src, ptr) as s8),
    u8 => yield(read_1_byte(src, ptr) as u8),
    s16 => yield(read_2_bytes(src, ptr) as s16),
    u16 => yield(read_2_bytes(src, ptr) as u16),
    s32 => yield(read_4_bytes(src, ptr) as s32),
    u32 => yield(read_4_bytes(src, ptr) as u32),
    s64 => yield(read_8_bytes(src, ptr) as s64),
    u64 => yield(read_8_bytes(src, ptr) as u64),

    # defer validation of the value to `lift`
    char => yield from lift(direction, src, char, [read_4_bytes(src, ptr) as wasm_i32])

    # defer the heavy-lifting to, well, `lift`
    string |
    list<$ty> => {
      ptr = read_4_bytes(src, ptr) as wasm_i32
      len = read_4_bytes(src, ptr + 4) as wasm_i32
      yield from lift(direction, src, list<$ty>, [ptr, len])
    }

    flags<$fields> => {
      values = []
      for i in 0..num_packed_i32s($fields):
        values.push(read_4_bytes(src, ptr + i * 4))
      yield from lift(direction, src, flags<$fields>, values)
    }

    record<$fields> => {
      yield(record_start<$fields>)
      offset = 0
      for $field in $fields:
        offset = align_to(offset, align($field))
        yield from lift_from_memory(direction, src, ptr + offset, $field)
        offset += size(direction, $field)
      yield(record_end<$fields>)
    }

    variant<$cases> => {
      match val {
        $case(payload) => {
          offset = 0
          match variant_discriminant($cases) {
            i8 => {
              discrim = read_1_byte(dst, ptr)
              offset += 1
            }
            i16 => {
              discrim = read_2_bytes(dst, ptr)
              offset += 2
            }
            i32 => {
              discrim = read_4_bytes(dst, ptr)
              offset += 4
            }
            i64 => {
              discrim = read_8_bytes(dst, ptr)
              offset += 8
            }
          }
          assert_or_trap(discrim >= 0 && discrim < $cases.len())
          yield(variant_start<$cases>(discrim))

          # Recursively read the payload, if present for this case. Note that
          # we align the pointer to the whole variant's alignment to ensure that
          # all payloads start at the same address.
          if payload:
            offset = align_to(offset, align(variant $cases))
            yield from lift_from_memory(direction, dst, ptr + offset, payload)

          yield(variant_end<$cases>)
        }
      }
    }

    # defer validation and such to `lift`
    handle<$resource> => {
      val = read_4_bytes(src, ptr)
      yield from lift(direction, src, handle<$resource>, [val as wasm_i32])
    }

    push-buffer<$ty> | pull-buffer<$ty>(val) => {
      assert(direction == "import") # see comments above
      handle = read_4_bytes(src, ptr)
      offset = read_4_bytes(src, ptr + 4)
      len = read_4_bytes(src, ptr + 8)
      yield from lift(direction, src, ty, [handle as wasm_i32, offset as wasm_i32, len as wasm_i32])
    }
```

## Lowering

The `lower` function here takes three parameters:

* `direction` - one of `"export"` or `"import"` which indicates for which
  purposes the lowering is happening: lowering an export means lowering
  a parameter, lowering an import means lowering a result.
* `dst` - this is an instance where the value is being lowered into. This could
  be a WebAssembly instance or the host itself.
* `gen` - a generator representing the interface types values that's being
  lowered. This is expected to be the generator returned from calling `lift`
  above.

This defines how an interface types value is encoded into linear memory and
WebAssembly values.

```
def lower(direction, dst, gen):
  match gen.next()
    f32(val) => [val as wasm_f32]
    f64(val) => [val as wasm_f64]

    # sign-extended
    s8(val) => [val as wasm_i32]
    s16(val) => [val as wasm_i32]

    # zero-extended
    u8(val) => [val as wasm_i32]
    u16(val) => [val as wasm_i32]

    s32(val) => [val as wasm_i32]
    u32(val) => [val as wasm_i32]
    s64(val) => [val as wasm_i64]
    u64(val) => [val as wasm_i64]

    char(val) => [val as wasm_i32]

    # Note that `val` here is guaranteed to be a valid list of unicode scalar
    # values so encoding here should always succeed.
    string(val) => {
      base = dst.canonical_abi_realloc(NULL, 0, 1, utf8_byte_length(val))
      assert_or_trap(base + utf8_byte_length(val) <= dst.memory.len)
      encode_utf8_string(dst, base, val)
      [base as wasm_i32, val.utf8_len() as wasm_i32]
    }

    # List are represented by their pointer/length as 32-bit integers. Note that
    # the `canonical_abi_realloc` can be elided if `dst` is the host itself since
    # the host can retain the raw pointers into the wasm module's memory if it
    # likes.
    list_start<$ty>(len) => {
      base = dst.canonical_abi_realloc(NULL, 0, align($ty), len * size(direction, $ty))
      assert_or_trap(base + len * size(direction, $ty) <= dst.memory.len)
      ptr = base
      for _ in 0..len:
        lower_to_memory(direction, dst, ptr, gen)
        ptr += size(direction, $ty)
      assert_eq(gen.next(), list_done<$ty>) # true by construction
      [base as wasm_i32, val.len() as wasm_i32]
    }

    flags<$fields>(val) => {
      values = []
      for chunk in $fields.chunks(32):
        bits = 0
        for i, field in enumerate(chunk):
          if val[field]:
            bits |= 1 << i
        values.push(bits)
      values
    }

    record_start<$fields> => {
      result = []
      for _ in $fields.len():
        result.append(lower(direction, dst, gen))
      assert_eq(gen.next(), record_end<$fields>) # true by construction
      result
    }

    # variants are represented with a discriminant plus whatever is the
    # smallest which fits all possible cases.
    variant_start<$cases>(discrim) => {
      result = [$cases.index($case) as wasm_i32]

      # Recursively serialize the optional payload
      if $cases[discrim].has_payload:
        result.append(lower(direction, dst, gen))

      assert_eq(gen.next(), variant_end<$cases>) # true by construction

      # Upcast each individual type in the payload to the shared type for
      # each arm of the variant
      payload_types = flatten_cases(direction, $cases)
      for i in 0..result.len() - 1:
        result[i + 1] = upcast(result[i + 1], payload_types[i])

      # If the shared type has more values than this payload generated we
      # insert zero values to pad
      for i in result.len() - 1..payload_types.len()
        result.push(zero_value(payload_types[i]))

      result
    }

    # Lowering a resource means that we're inserting our handle into a fresh
    # slot in the destination's table. This insertion operation allocates a new
    # i32 value which will point to the handle, and note that this also bumps the
    # handle's internal reference count.
    handle<$resource>(val) => [TABLES.insert($resource, val, dst)],

    # Buffers are only ever lowered as parameters of an exported function.
    # Buffers can't be returned from a function so they can never be lowered
    # for import results.
    push-buffer<$ty>(val) | pull-buffer<$ty>(val) => {
      assert(direction == "export")
      [TABLES.insert_buffer(val, dst)]
    }

def lower_to_memory(direction, dst, ptr, gen):
  match gen.next()
    f32(val) => write_4_bytes(dst, ptr, val),
    f64(val) => write_8_bytes(dst, ptr, val),

    s8(val) => write_1_byte(dst, ptr, val),
    u8(val) => write_1_byte(dst, ptr, val),
    s16(val) => write_2_bytes(dst, ptr, val),
    u16(val) => write_2_bytes(dst, ptr, val),
    s32(val) => write_4_bytes(dst, ptr, val),
    u32(val) => write_4_bytes(dst, ptr, val),
    s64(val) => write_8_bytes(dst, ptr, val),
    u64(val) => write_8_bytes(dst, ptr, val),

    char(val) => write_4_bytes(dst, ptr, val),

    string(val) |
    list<$ty>(val) => {
      gen.requeue(val) # put `val` back in the generator to get yielded first in `lower`
      [val_ptr, val_len] = lower(direction, dst, val)
      write_4_bytes(dst, ptr, val_ptr)
      write_4_bytes(dst, ptr + 4, val_len)
    }

    flags<$fields>(val) => {
      gen.requeue(val)
      for i, bits in enumerate(lower(direction, dst, val)):
        write_4_bytes(dst, ptr + i * 4, bits)
    }

    record_start<$fields> => {
      offset = 0
      for $field in $fields:
        offset = align_to(offset, align($field))
        lower_to_memory(direction, dst, ptr + offset, gen)
        offset += size(direction, $field)
      assert_eq(gen.next(), record_end<$fields>) # true by construction
    }

    variant_start<$cases>(discrim) => {
      offset = 0
      match variant_discriminant($cases) {
        i8 => {
          write_1_byte(dst, ptr, discrim)
          offset += 1
        }
        i16 => {
          write_2_bytes(dst, ptr, discrim)
          offset += 2
        }
        i32 => {
          write_4_bytes(dst, ptr, discrim)
          offset += 4
        }
        i64 => {
          write_8_bytes(dst, ptr, discrim)
          offset += 8
        }
      }

      # Recursively write the payload, if present for this case. Note that
      # we align the pointer to the whole variant's alignment to ensure that
      # all payloads start at the same address.
      if $cases[discrim].has_payload:
        offset = align_to(offset, align(variant $cases))
        lower_to_memory(direction, dst, ptr + offset, gen)

      assert_eq(gen.next(), variant_end<$cases>) # true by construction
    }

    handle<$resource>(val) => {
      gen.requeue(val)
      bits = lower(direction, dst, val)[0]
      write_4_bytes(dst, ptr, bits)
    }

    push-buffer<$ty>(val) | pull-buffer<$ty>(val) => {
      gen.requeue(val)
      bits = lower(direction, dst, val)[0]
      write_4_bytes(dst, ptr, bits)
    }
```

## Helper `TABLES` class

In Python-like pseudo-code this is how the `TABLES` referenced above in
lifting/lowering will work for managing indices. Note that the implementation
provided here is intended to be canonical. Specifically these properties are
part of the canonical ABI:

* There is a separate index space per-instance and per-resource type. This means
  that each instance's adaptation effectively has access to a set of tables for
  each handle type, and indices are managed by allocating from these tables.

* Indices to refer to resources are handled in a LIFO fashion. Indexes are
  allocated starting from zero and increasing afterwards as more space is
  required. When an index is removed (because an instance said it no longer
  needs a resource, then that index is queued up as the next to be allocated).

* There can be at most 2^31-1 values of any handle type for a module. This is
  done so indices can fit into a hypothetical `i31ref` in the future.

```
class TABLES:
  instances = Map<(Instance, Resource), Slab>
  buffers = Vec<(Buffer, Instance)>

  # This function will flag that `instance` has access to the `handle` provided.
  #
  # Returns a 32-bit integer which can later be passed to `get` to retrieve the
  # original `handle`. This function always succeeds, unless the table
  # overflows from too many insertions.
  #
  # Note that 1 is added here to never allocate the 0 index.
  def insert(self, resource, handle, instance) -> i32:
    handle.refcnt += 1
    return self.instances[(instance, resource)].push(handle)

  # This function will look up the value `idx` referenced by `instance`. If an
  # entry of type `resource` exists then the original value is returned.
  #
  # If `instance` does not have any valid entry for `idx` then this will trap.
  def get(self, resource, idx, instance) -> Handle:
    assert_or_trap(idx >= 0)
    return self.instances[(instance, resource)].get(idx)

  # Removes access to `instance`'s access to the handle `idx` which should point
  # to `resource`.
  #
  # This function is called from `canonical_abi_drop_$resource`, an intrinsic
  # documented below, and is not called from usual lifting and lowering. The
  # purpose of this is to inform the table here that the instance's access to
  # `idx` is no longer needed and, if possible, to release resources associated
  # with `idx`. This will call the original module's actual destructor function
  # if the reference count for the resource reaches 0.
  #
  # If `instance` does not have any valid entry for `idx` then this will trap.
  # If the entry for `idx` does not have type `$resource` then this will trap.
  def remove(self, resource, idx, instance):
    handle = self.get(resource, idx, instance)
    self.instances[instance].remove(idx)
    handle.refcnt -= 1
    if handle.refcnt == 0:
      handle.src.canonical_abi_drop_$resource(handle.val)

  # Flags that `instance` has access to the `buffer` provided, returning a
  # nonzero index which the receiving module can use to refer to this buffer.
  def insert_buffer(self, buffer, instance) -> idx:
    ret = self.buffers.len()
    self.buffers.push((buffer, instance))
    return ret

  # Attempts to lookup a buffer at `idx` that `instance` should have access to,
  # and the buffer's type should be `ty`. If all of this is true then a buffer
  # is returned, otherwise if a check fails this raises a trap.
  def get_buffer(self, idx, instance, ty) -> Buffer:
    assert_or_trap(idx >= 0 && idx < self.buffers.len())
    buffer, owner = buffers[idx]
    assert_or_trap(owner == instance)
    assert_or_trap(buffer.ty == ty)
    return buffer


class Slab:
  head: i32
  list: Vec<Handle | i32>

  def __init__(self):
    self.head = 0
    self.list = []

  def push(self, handle):
    if self.head == self.list.len():
      self.list.push(self.list.len() + 1)
    ret = self.head
    self.head = self.list[ret]
    self.list[ret] = handle
    return ret

  def get(self, idx):
    # make sure the index is in bounds
    assert_or_trap(idx < self.list.len())
    ret = self.list[idx]
    # make sure this is an allocated slot
    assert_or_trap(ret instanceof Handle)
    return ret

  def remove(self, idx):
    self.get(idx)     # validate that this is an allocated index
    self.list[idx] = self.head
    self.head = idx


class Handle:
  val: i32
  refcnt: i32
  src: Instance


class Buffer:
  src: Instance     # where this buffer is located
  offset: i32       # byte offset in `src`'s memory
  len: i32          # length in units of `self.ty`
  ty: Type          # either push-buffer<T> or pull-buffer<T>
```

## Adapter fusion

The final piece of the pie associated with the canonical ABI is what happens
when adapters are fused together. The below `fuse` function is pseudo-code for
what would happen when `a_instance` called the import with interface-types
signature `sig` and `b_instance` has an export that we're calling:

```
def fuse(signature, instance_a, instance_b, wasm_params):
  # Record a pointer into the list of active buffers, because we'll be resetting
  # the list of active buffers back to this after the function has returned.
  buffers_len = TABLES.buffers.len()

  # deal with return pointer goop
  if uses_retptr(signature, "import"):
    params_to_adapt = wasm_params[1..]
  else
    params_to_adapt = wasm_params

  # Use lifting/lowering to convert all parameters from `instance_a` into
  # `instance_b`. Note that this performs validation of values coming out of
  # `instance_a` so `instance_b` can trust all of its inputs.
  new_params = []
  for param_ty in signature.params:
    gen = lift("import", instance_a, param_ty, params_to_adapt)
    new_params.extend(lower("export", instance_b, gen))
    assert(gen.is_done()) # should be valid due to validation
  assert(params_to_adapt.is_empty()) # should be valid due to validation

  # Call `instance_b` with all of its relative values, and then deal with
  # multiple returns to get the results.
  results = instance_b.exports[signature.name](..new_params)
  if uses_retptr(signature, "export")
    results = read_wasm_results(instance_b, signature, results[0])

  # Use lifting/lowering again, but convert the other way.
  new_results = []
  for result_ty in signature.results:
    gen = lift("export", instance_b, result_ty, results)
    new_results.extend(lower("import", instance_a, val))
    assert(gen.is_done()) # should be valid due to validation
  assert(results.is_empty()) # should be valid due to validation

  # Reset buffers back to what they previously were, discarding all buffer
  # objects that were created or in-use for this function call.
  TABLES.buffers.truncate(buffers_len)

  if uses_retptr(signature, "import"):
    write_wasm_results(instance_a, wasm_params[0], new_results)
    return []
  else
    return new_results

def uses_retptr(signature, direction):
  return len(signature.results.flat_map(|t| flatten(direction, t))) > 1

def read_wasm_results(src, signature, ptr):
  wasm_tys = signature.results.flat_map(|t| flatten("export", t))
  assert_or_trap(ptr % 8 == 0)
  results = []
  for i, ty in wasm_tys:
    bits = read_8_bytes(src, ptr + i * 8)
    results.push(match ty {
      i32 => i32(bits as i32),
      i64 => i64(bits),
      f32 => f32(f32::from_bits(bits as i32)),
      f64 => f64(f64::from_bits(bits)),
    })
  return results

def write_wasm_results(dst, ptr, vals):
  assert_or_trap(ptr % 8 == 0)
  for i, val in vals:
    bits = match val
      i32(v) => i64::from(v),
      i64(v) => v,
      f32(v) => i64::from(v.to_bits())
      f64(v) => i64::from(v.to_bits())
    write_8_bytes(dst, ptr + i * 8, bits)
```

Here `fuse` performs the necessary lifting and lowering to translate all
parameters and results across the call. This `fuse` function is expected to be
implemented by "trusted code" since it refers to `TABLES`.

It's worth pointing out that either `instance_a` or `instance_b` could in theory
be a host with this function as well. A host is more privileged than an
arbitrary wasm module, so much of this can be specialized when dealing with
hosts. For example a host might not actually `lower` the parameters and instead
just work with the raw `lift`-ed values. This means that a host doesn't actually
need to copy a `string` into host memory, it can reference the raw memory
sitting in wasm (assuming no multithreading of course). Similarly if a host
calls an exported function on a wasm module it would likely skip `lift`-ing
parameters and it would go straight to `lower`-ing each one.

It's also worth pointing out that the order of operations here is intended to be
canonical. Lifting and lowering can be side-effectful because of calls to
`canonical_abi_realloc` and `canonical_abi_free`, and the canonical ABI intends
for the ordering of these calls to indeed be canonical. An important property,
however, is that lifting import parameters is guaranteed to be side-effect-free
meaning that host implementations of imported functions can retain all their
raw-in-memory-pointers as necessary.

# Intrinsics

One of the final details of the canonical ABI is that there will be a few
intrinsic functions to operate on the various types. These are primarily
intended for handle-related types and lists.

## Memory Intrinsics

When lowering a list, the canonical ABI needs to allocate space in the
destination core module's linear memory that can be written into. When lifting
a returned list, the canonical ABI needs to free the allocated memory after it
has been read from. When either of these are required, the core module (that's
calling the canonical ABI) must export one or both of the following functions:

```wasm
(module
  (func (export "canonical_abi_realloc") (param i32 i32 i32 i32) (result i32))
  (func (export "canonical_abi_free") (param i32 i32 i32))
)
```

The `canonical_abi_realloc` is provided for use today and to future-proof the
canonical ABI for a future where the canonical ABI is not the only ABI (e.g.
full expressive adapter functions). Realloc takes four arguments: the original
pointer, the original size, the original alignment, and the new desired size. It
returns an `i32` which is a pointer in memory which is valid for the new
desired size of bytes. This function will trap if memory allocation failed and
the first pointer argument can be 0 to indicate that this is effectively a
fresh call for allocated bytes.

The `canonical_abi_free` function takes a pointer/size/alignment and informs the
allocator that the memory is no longer needed.

## Handle Intrinsics

For all types `handle $resource` imported and used by a module the module can
always import these functions:

```wasm
(module
  (import "canonical_abi" "resource_clone_$resource" (func (param i32) (result i32)))
  (import "canonical_abi" "resource_drop_$resource" (func (param i32)))
)
```

It's important to note that the canonical ABI implies reference counting on
handles. The `clone` and `drop` intrinsics map to incrementing and decrementing
the reference count. They are implemented as follows:

```
def resource_clone_$resource(src, val):
  handle = TABLES.get($resource, src, val)
  return TABLES.insert($resource, handle, src)

def resource_drop_$resource(src, val):
  return TABLES.remove($resource, handle, src)
```

Notably `clone` and `drop` will both trap if the index is invalid and the
instance doesn't actually have access to that resource. The `clone` function is
used to create a new index pointing to the same resource in the module's index
space. The `drop` function indicates that the module no longer needs access to
that index. If the reference count reaches 0 at that time then the destructor
for the resource is run (perhaps invoking another module).

If a module defined `$resource` then it can also import these functions:

```wasm
(module
  (import "canonical_abi" "resource_new_$resource" (func (param i32) (result i32)))
  (import "canonical_abi" "resource_get_$resource" (func (param i32) (result i32)))
)
```

which are implemented as:

```
def resource_new_$resource(src, val):
  handle = HANDLE(val=val, refcnt=0, src=src) # refcnt bumped to 1 in insert next
  return TABLES.insert($resource, handle, src)

def resource_get_$resource(src, val):
  return TABLES.get($resource, handle, src).val
```

These functions are used to create a handle wrapper around a resource
(identified as a unique i32) and then to access the private i32 value. These
functions are only available to modules/instances that defined the resource
which means no other instances can have access to the private internals. Note
that `get` will trap if the index passed in is invalid.

Note that hosts may have their own reference count of resources owned by wasm as
well. For example if a wasm resource makes its way all the way to a host then
the host may hold a reference count that, when dropped, may trigger a wasm
destructor. Similarly if a wasm resource is owned by the host and all other wasm
modules with access drop the resource then the resource isn't dropped in the
original module until the host is done with it.

## Buffer Intrinsics

Buffers defined in foreign modules are identified by an `i32` index. To actually
operate on that index the following intrinsics can be used:

```wasm
(module
  (import "canonical_abi" "push_buffer_len" (func (param i32) (result i32)))
  (import "canonical_abi" "push_buffer_push" (func (param i32 i32 i32) (result i32)))
  (import "canonical_abi" "pull_buffer_len" (func (param i32) (result i32)))
  (import "canonical_abi" "pull_buffer_pull" (func (param i32 i32 i32) (result i32)))
)
```

The implementation of these intrinsics is supplied by the table/buffer
management glue.

The `canonical_abi::push_buffer_len` function takes the integer descriptor for
the buffer and returns the integer length of the buffer. This is the maximal
number of items which can be pushed into the buffer.

The `canonical_abi::push_buffer_push` function takes the integer descriptor for
the buffer, a base pointer, and a length. This will use the `deserialize_from_memory`
function to read the corresponding type of element from the buffer provided, up
to the length number of times. The number of items actually pushed into the
buffer will be returned.

The `canonical_abi::pull_buffer_len` is the same as the push variant, only it
returns how many items can be pulled.

The `canonical_abi::pull_buffer_push` is the same as the push version, except
that it uses `write_to_memory` to write to the provided slice of bytes.

All of these functions will call `TABLES.get_buffer` with the first argment as
the provided index and the calling instance as the instance. This means that the
functions will trap if `idx` isn't a valid buffer index.

The `push` and `pull` functions are implemented as:

```
def push(src, idx, ptr, len):
  buffer = TABLES.get_buffer(idx, src)
  ty = match buffer.ty {
    push-buffer<T> => T,
    _ => assert_or_trap(false)
  }
  assert_or_trap(ptr + len * size("export", ty) <= src.memory.len)
  amt = min(len, buffer.len)
  for i in 0..amt:
    gen = lift_from_memory("export", src, ty, ptr)
    lower_to_memory("export", buffer.src, buffer.offset, gen)
    ptr += size("export", ty)
    buffer.offset += size("export", ty)
    buffer.len -= 1
  return amt

def pull(dst, idx, ptr, len):
  buffer = TABLES.get_buffer(idx, dst)
  ty = match buffer.ty {
    pull-buffer<T> => T,
    _ => assert_or_trap(false)
  }
  assert_or_trap(ptr + len * size("import", ty) <= dst.memory.len)
  amt = min(len, buffer.len)
  for i in 0..amt:
    gen = lift_from_memory("import", buffer.src, ty, buffer.offset)
    lower_to_memory("import", dst, ptr, gen)
    ptr += size("import", ty)
    buffer.offset += size("import", ty)
    buffer.len -= 1
  return amt
```

Notably these functions do not trap if `len` is larger than the buffer, but
rather they simply transfer as many elements as possible.
