# Host Bindings Proposal for WebAssembly

## Motivation

WebAssembly currently in practice relies on a substantial amount of support
from JavaScript + Web APIs to be useful on the Web.
Interoperability with JavaScript and Web APIs will help make WebAssembly
in practice more "of the Web" while improving performance and ergonomics.
Bindings for non-Web hosts embeddings are also relevant.

## Goals / Non-Goals

Goals:
* Ergonomics - Allow WebAssembly modules to create,
  pass around, call, and manipulate JavaScript + DOM objects.
* Speed - Allow JS/DOM or other host calls to be well optimized.
* Platform Consistency - Allow WebIDL to be used
  to annotate Wasm imports / exports (via a tool).
* Incrementalism - Provide a strategy that is polyfillable (maybe partial).

Non-Goals:
* Provide a general purpose managed object solution.
* No support for anything not expressible in JS/Host objects,
  such as concurrent access.

## Basic Approach

* Extend `WebAssembly.Table`, allowing variants that support tables
  of particular kinds of JavaScript objects.
   * {`element`: elemType} extended to allow elemType to reference particular
     prototypes. E.x.: WebGLRenderingContext
   * Throw `TypeError` (as now) if the wrong type is stored in a Table.
* Allow multiple Tables to be imported. Table 0 will remain the indirect
  function table.
* Add a "Host Bindings" section to WebAssembly modules.
   * Describes conversion steps for imports + exports that:
      * Allows incoming objects of various types to be directed to a Table slot
        and an i32 slot index be passed to the WebAssembly function.
      * Allows outgoing objects of various types be expressed as i32 indices
        into a particular Table object.
      * Allows the WebAssembly module to manage allocation of Table slots.

## Details

### A subsection will list a series of IMPORT bindings:

#### Structure of an Import Binding

Field | Description
--- | ---
Import index | Function with extra conversions
Function Mode | Style of function call: function / new / method invocation.
n x Argument Bindings | List of description of how each argument is handled.
Return Type | Description of how the return type is handled.

#### Import Function Modes

Mode | Description
--- | ---
CALL_FUNCTION | Treated as a normal function call.
CALL_NEW | Calls function as if called as: new func(...). Return type must be OBJECT_HANDLE.
CALL_THIS | Treats the first argument as a 'this' reference to bind the call to.

#### Import Argument Binding Types

A series of binding conversion operations for each import argument
(calling from Wasm-to-Host), taken from:

Type | Description | Arguments
--- | --- | ---
PASS_THRU | Leaves the argument in the current type. |
OBJECT_HANDLE | Converts the next argument from an i32 to object by getting the object out of a table slot (arg must be i32) | object table index
U32 | Treats the next argument as an u32 (must have i32 type) |
U64_PAIR | Pass the next argument (must be i64) as a pair of u32 (low then high), treating the value as unsigned. |

JavaScript hosts might additionally provide:

Type | Description | Arguments
--- | --- | ---
STRING | Converts the next two arguments from a pair of i32s to a utf8 string. It treats the first as an address in linear memory of the string bytes, and the second as a length. |
ARRAY_BUFFER | Converts the next two arguments from a pair of i32s to an ArrayBufferView. It treats the first as an address in linear memory of the array view bytes, and the second as a length. |
JSON | Converts the next two arguments from a pair of i32s to JSON. It treats the first as an address in linear memory of the json bytes, and the second as a length. Parses this region as if it has been passed to JSON.parse(). |
STRING_IMMEDIATE | Encodes a string in this section, passed as an argument, but does not consume an actual function argument. |

#### Import Return Value Binding Types

A binding conversion for the return type (Host-to-Wasm), taken from:

Type | Description | Arguments
--- | --- | ---
PASS_THRU | Leaves the return value in the current type |
OBJECT_HANDLE | Stores the import return value to an i32 location specified by an outgoing argument (last one). Requires one unconsumed argument above that must be an i32. | object table index

JavaScript hosts might additionally provide:

Type | Description | Arguments
--- | --- | ---
STRING | Calls a passed ALLOC_MEM function to reserve destination space, copies in the bytes, and provides the address of the i32 allocation as the return value. | index of an allocation function
ARRAY_BUFFER | Calls a passed ALLOC_MEM function to reserve destination space, copies in the bytes, and provides the address of the i32 allocation as the return value. Assumes one additional argument is available to provide the index of the allocation function. | index of allocation function
JSON | Calls a passed ALLOC_MEM function to reserve destination space, copies in the bytes, and provides the address of the i32 allocation as the return value.  Assumes one additional argument is available to provide the index of the allocation function. | index of allocation function

### A subsection will list a series of EXPORT bindings:

#### Structure of an Export binding

Field | Description
--- | ---
Export index | Function with extra conversions
Return Type | Description of how the return type is handled.
n x Argument Bindings | List of description of how each argument is handled.

#### Export Argument Binding Types

A series of binding conversion operations for each import argument (going from
Host-to-Wasm), taken from:

Type | Description | Arguments
--- | --- | ---
PASS_THRU | Leaves the argument in the current type.
OBJECT_HANDLE | The incoming argument is placed in a slot selected by the NEXT_SLOT i32 global. This slot number is passed as an i32 in its place. The NEXT_SLOT i32 global is incremented. The index of the NEXT_SLOT global is provided in this section. | object table index, index of NEXT_SLOT i32 global
U64_PAIR | Decode the next argument as a pair of u32s (low then high), treating the value as an i64. |

JavaScript hosts might additionally provide:

Type | Description | Arguments
--- | --- | ---
STRING | A provided ALLOC_MEM function is called to reserve linear memory for the string. String bytes are copied to the allocated memory and the address and size are passed as a pair of i32 arguments to the function in their place.  The index of the ALLOC_MEM function is provided in this section. | index of ALLOC_MEM function
ARRAY_BUFFER | A provided ALLOC_MEM function is called to reserve linear memory for the array buffer view data. Buffer bytes are copied to the allocated memory and the address and size are passed as a pair of i32 arguments to the function in their place.  The index of the ALLOC_MEM function is provided in this section. | index of ALLOC_MEM function
JSON | A provided ALLOC_MEM function is called to reserve linear memory for the string resulting from JSON.stringify(). Buffer bytes are copied to the allocated memory and the address and size are passed as a pair of i32 arguments to the function in their place.  The index of the ALLOC_MEM function is provided in this section. | index of ALLOC_MEM function

#### Export Return Value Binding Types

A binding conversion for the return type (Wasm-to-Host), taken from:

Type | Description | Arguments
--- | --- | ---
PASS_THRU | Leaves the return type in the current type. |
OBJECT_HANDLE | Converts the return value from an i32 to object by getting the object out of a corresponding table slot (arg must be i32). | object table index
U32 | Treats the return value an u32 (must have i32 type). |
U64_PAIR | Treat the return value (must be i64) as a u64 and return as a pair of u32 (low then high). |

JavaScript hosts might additionally provide:

Type | Description | Arguments
--- | --- | ---
STRING | The return value is interpreted as the i32 address in linear memory of a pair of i32s. The first is the address of a region to convert, the second its length. The byte region is converted to a string. A FREE_MEM function is invoked prior to return on the i32 address returned (to allow cleanup). | index of FREE_MEM function
ARRAY_BUFFER | The return value is interpreted as the i32 address in linear memory of a pair of i32s. The first is the address of a region to convert, the second its length. The byte region is converted to an ArrayBufferView of the heap. A FREE_MEM function is invoked prior to return on the i32 address returned (to allow cleanup). | index of FREE_MEM function
JSON | The return value is interpreted as the i32 address in linear memory of a pair of i32s. The first is the address of a region to convert, the second its length. The byte region is converted to a string then JSON.parse() is invoked on the result, which is returned. A FREE_MEM function is invoked prior to return on the i32 address returned (to allow cleanup). | index of FREE_MEM function

## Migration / Polyfill

We may be able to polyfill the approach, though this will likely require
some amount of module bytes filtering.

Key points:
* WebAssembly.Table will need to be wrapped to pretend it can support
  more element types.
* A JavaScript implementation of decoding the "Host Bindings" section will need to:
  * Remove this section from what is handed to WebAssembly.
  * Wrap imported / exported methods in functions which update tables based
    on what is passed in / out.

## Allocation

Exports have the property that they need to be able to allocate Table slots
for incoming objects or linear memory for raw data.

For raw data like STRING, ARRAY_BUFFER, etc. the index of an ALLOC_MEM
function for incoming, and FREE_MEM function for outgoing data is used to
give the WebAssembly module the opportunity to manage the linear memory.

For objects, we especially want a cheap calling convention. Rather than provide
a single slot alloc/free, we provide a NEXT_SLOT global to hold the i32 index
of a pre-reserved location for the next incoming object of a given type.
A reservation function can then be called inside exports to commit the
reservation and get the next one. Since the code for the common case might
be small, this allows toolchain inlining of that path inside the WebAssembly
module. NEXT_SLOT is incremented after use. This potentially allows multiple
arguments of the same type to share a single slot and reservation function.
However, that does require the allocator to provide a NEXT_SLOT with contiguous
slots up to the maximum number of arguments of the same type in the program.

For freeing slots, no explicit mechanism is provided. But the assumption is
that global containing a pending free slot can be set prior to function return,
which will get released on the next allocation.
NOTE: This does have the side-effect of holding a reference to the last
returned value of each type until the module is re-entered through a path
that triggers the actual free.

## Toolchains

Offering convenient access to JavaScript + Web APIs is crucial to the usefulness
of this proposal. Tooling should represent these bindings at a source code
level via attributes. This will allow our LLVM backend to extract binding
information, and generate an appropriate Host Bindings section.

### WebIDL

In order to offer useful bindings for Web APIs, a tool that converts from
WebIDL to a attributes will be required.

It might convert input like this:

```webidl
webgl.idl
---------

[Exposed=(Window,Worker),
 Func="mozilla::dom::OffscreenCanvas::PrefEnabledOnWorkerThread"]
interface WebGLRenderingContext {
  const GLenum VERTEX_SHADER = 0x8B31;
  WebGLShader createShader(GLenum type);
  void shaderSource(WebGLShader shader, DOMString source);
  void compileShader(WebGLShader shader);
}
```

To something like this:

```c
webgl_bindings.h
----------------

typedef int32 WebGLRenderingContext
   __attribute__(wasmjsdom("object_handle:WebGLRenderingContext"));
typedef int32 WebGLShader
   __attribute__(wasmjsdom("object_handle:WebGLShader"));
const int WebGLRenderingContext_VERTEX_SHADER = 0x8B31;

extern void WebGLRenderingContext_createShader(
   WebGLRenderingContext self, int32 type, WebGLShader result);
extern void WebGLRenderingContext_shaderSource(
   WebGLRenderingContext self, WebGLShader shader,
   const char* str, int32 length);
extern void WebGLRenderingContext_compileShadershaderSource(
   WebGlRenderingContext self, WebGLShader shader);
```

### Example

The above bindings could be used to compile a shader:

```c
typedef struct { char* str; int32 len; } DOMString
   __attribute__(wasmjsdom("utf8_string:"));

WebGLShader _drop_slot_WebGLShader;
WebGLShader _alloc_shader_slot() { // allocs and updates _drop_slot... }

WebGLRenderingContext _next_slot_WebGLRenderingContext;
void _reserve_slot_WebGLRenderingContext() { // update next slot... }

EMSCRIPTEN_KEEPALIVE
WebGLShader createVertexShader(WebGLRenderingContext gl, DOMString code) {
  _reserve_slot_WebGLRenderingContext();
  WebGLShader shader = _alloc_shader_slot();
  WebGLRenderingContext_createShader(
      gl, WebGLRenderingContext_VERTEX_SHADER, shader);
  WebGLRenderingContext_shaderSource(gl, shader, code.str, code.len);
  free(code.str);
  WebGLRenderingContext_compileShader(gl, shader);
  _drop_slot_WebGLShader = shader;
  return shader;
}
```

Internally this becomes:

```c
int32 _drop_slot_WebGLShader;
int32 _alloc_shader_slot() { ... }

int32 createVertexShader(int32 gl, int32 code_str, int32 code_len) {
  _reserve_slot_WebGLRenderingContext();
  int32 shader = _alloc_shader_slot();
  WebGLRenderingContext_createShader(gl, 0x8B31, shader);
  WebGLRenderingContext_shaderSource(gl, shader, code_str, code_len);
  free(code_str);
  WebGLRenderingContext_compileShader(gl, shader);
  _drop_slot_WebGLShader = shader;
  return shader;
}

int32 _alloc_mem(int32 size) { return malloc(size); }
```

Bindings for each import / export will also be generated.

The import binding for `shaderSource` might be something like:

Field | Value | Arguments
--- | --- | ---
Import Index | shaderSource(123) |
Import Mode | CALL_THIS |
Arg0 | OBJECT_HANDLE | Table(5):WebGLRenderContext
Arg1 | OBJECT_HANDLE | Table(7):WebGLShader
Arg2 | STRING |
Return | OBJECT_HANDLE | Table(7):WebGLShader

The export binding for `createVertexShader` might be something like:

Field | Value | Arguments
--- | --- | ---
Export Index | createVertexShader(456) |
Arg0 | OBJECT_HANDLE | Table(5):WebGLRenderContext, _next_slot_WebGLRenderingContext
Arg1 | STRING | _alloc_mem(678)
Return | OBJECT_HANDLE | Table(7):WebGLShader
