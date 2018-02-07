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
* Incrementalism - Provide a strategy that is polyfillable.

Non-Goals:
* Provide a general purpose managed object solution.
* Cover functionality better carved off Managed Objects in the Anyref proposal.
* No support for anything not expressible in JS/Host objects,
  such as concurrent access.

## Basic Approach

* Assume the separate Anyref proposal will allow direct handling of
  host objects. This will include:
   * Allows host objects in: locals, globals, stack slots, and arguments.
   * Allow multiple Tables to be imported. This will allow indirect indexed
     access to host objects.
   * Extend `WebAssembly.Table` {`element`: elemType} to allow elemType
     to be `anyref'.
* Add a "Host Bindings" section to WebAssembly modules.
   * Describes conversion steps for imports + exports that:
      * Allows unsigned conversion.
      * Allows invocation of constructors.
      * Allows invocation of bound methods.

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
U32 | Treats the next argument as an u32 (must have i32 type) |

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
U32 | Treats the return value an u32 (must have i32 type). |

JavaScript hosts might additionally provide:

Type | Description | Arguments
--- | --- | ---
STRING | The return value is interpreted as the i32 address in linear memory of a pair of i32s. The first is the address of a region to convert, the second its length. The byte region is converted to a string. A FREE_MEM function is invoked prior to return on the i32 address returned (to allow cleanup). | index of FREE_MEM function
ARRAY_BUFFER | The return value is interpreted as the i32 address in linear memory of a pair of i32s. The first is the address of a region to convert, the second its length. The byte region is converted to an ArrayBufferView of the heap. A FREE_MEM function is invoked prior to return on the i32 address returned (to allow cleanup). | index of FREE_MEM function
JSON | The return value is interpreted as the i32 address in linear memory of a pair of i32s. The first is the address of a region to convert, the second its length. The byte region is converted to a string then JSON.parse() is invoked on the result, which is returned. A FREE_MEM function is invoked prior to return on the i32 address returned (to allow cleanup). | index of FREE_MEM function

## Migration / Polyfill

We may be able to polyfill the approach.

Key points:
* Assumes `Anyref' is implemented.
* A JavaScript implementation of decoding the "Host Bindings"
  section will need to:
  * Remove this section from what is handed to WebAssembly.
  * Wrap imported / exported methods in functions which update tables based
    on what is passed in / out.

## Allocation

For raw data like STRING, ARRAY_BUFFER, etc. the index of an ALLOC_MEM
function for incoming, and FREE_MEM function for outgoing data is used to
give the WebAssembly module the opportunity to manage the linear memory.

## Toolchains

Offering convenient access to JavaScript + Web APIs is crucial to the usefulness
of this proposal. Tooling should represent these bindings at a source code
level via attributes. This will allow our LLVM backend to extract binding
information, and generate an appropriate Host Bindings section.

### WebIDL

In order to offer useful bindings for Web APIs, a tool that converts from
WebIDL to a attributes will be required.

It might convert input like this:

```
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

```
webgl_bindings.h
----------------

typedef void* WebGLRenderingContext __attribute__(wasm_anyref);
typedef void* WebGLShader __attribute__(wasm_anyref);
typedef void* DOMString __attribute__(wasm_anyref);
const int WebGLRenderingContext_VERTEX_SHADER = 0x8B31;

extern WebGLShader WebGLRenderingContext_createShader(
   WebGLRenderingContext self, int32 type);
extern void WebGLRenderingContext_shaderSource(
   WebGLRenderingContext self, WebGLShader shader, DOMString source);
extern void WebGLRenderingContext_compileShader(
   WebGlRenderingContext self, WebGLShader shader);
```

### Example

The above bindings could be used to compile a shader:

```
EMSCRIPTEN_KEEPALIVE
WebGLShader createVertexShader(WebGLRenderingContext gl, DOMString code) {
  WebGLShader shader = WebGLRenderingContext_createShader(
      gl, WebGLRenderingContext_VERTEX_SHADER);
  WebGLRenderingContext_shaderSource(gl, shader, code);
  WebGLRenderingContext_compileShader(gl, shader);
  return shader;
}
```

Bindings for each import / export will also be generated.

The import binding for `shaderSource` might be something like:

Field | Value
--- | ---
Import Index | shaderSource(123)
Import Mode | CALL_THIS
Arg0 | PASS_THRU
Arg1 | PASS_THRU
Arg2 | PASS_THRU
Return | PASS_THRU

The export binding for `createVertexShader` might be something like:

Field | Value
--- | ---
Export Index | createVertexShader(456)
Arg0 | PASS_THRU
Arg1 | PASS_THRU
Return | PASS_THRU
