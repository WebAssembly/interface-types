# Choosing Strings

## Preamble
This note is part of a series of notes that gives complete end-to-end examples
of using Interface Types to export and import functions.

## Introduction

Keeping track of memory that is allocated in order to allow string processing
can be challenging. This example illustrates an extreme case that involves
non-determinism.

This scenario is based on the idea of using run-time information in order to
control the flow of information. In particular, consider the C++ chooser
function:

```C++
typedef std::shared_ptr<std::string> shared_string;

shared_string nondeterministic_choice(shared_string&& one, shared_string&& two) {
  return random() > 0.5 ? one : two;
}
```

This function takes two string arguments and returns one of them. Regardless of
the merits of this particular function, it sets up significant challenges should
we want to expose it using Interface Types. Specifically, there are two
resources entering the function, with just one leaving. However, when exposed as
an Interface Type function, all these resources must be created and properly
disposed of within the adapter code itself.

This note focuses on the techniques that enable this to be achieved reliably.

## Exporting the Chooser

The Interface Type function signature for our chooser is simple: the function
takes two `string`s and returns one:

```wasm
(@interface func (export "chooser")
  (param $left string)
  (param $right string)
  (result string)
  ...
)
```

One of the specific challenges in this scenario is the handling of shared
pointers. We 'require' that the core `nondeterministic_choice` function honor
the semantics of proper reference counting the input arguments and the returned
string.

However, there are actually _two_ schemes in play in this scenario: the
Interface Types management of resources and the C++ implementation of
`shared_ptr`. Effectively, in addition to lifting and lowering the `string`
values we must also lift and lower the ownership between Interface Types and
core WASM.

In this scenario we take a somewhat simplified view of C++'s implementation of
shared pointers: a shared pointer is implemented as a pair consisting of a
reference count and a raw pointer to the shared resource.

>Note: In practice, the C++ implementation of shared pointers is somewhat more
>complex; for reasons that are not important here.

This results in _two_ memory allocations for the shared resource: one for the
resource itself and one for the pointer structure -- which contains a reference
count and a raw pointer to the resource.


```wasm
(@interface func (export "chooser")
  (param $left string)
  (param $right string)
  (result string)

  local.get $left
  dup
  string.size
  call-export "malloc"
  string-to-memory            ;; leave the destination on the stack
  call-export "shared_builder"
  let (local $l i32)
  
    local.get $right
    dup
    string.size
    call-export "malloc"
    string-to-memory
    call-export "shared_builder" ;; build a shared_ptr structure
    let (local $r i32)
      ;; set up the call to the core chooser 
      local.get $p1
      local.get $p2
      call-export "chooser_"  ;; call the chooser itself
      call-export "shared_moveout" ;; actual value
      own (i32)               ;; own the result
        call-export "free"    ;; will eventually call to free string
      end
      memory-to-string
      
      local.get $p2           ;; the shared_ptr structures
      call-export "shared_release"
    end
    local.get $p1
    call-export "shared_release"
  end
end
```

The returned value from `chooser_` is wrapped up as an `own`ed allocation
and lifted to a `string`.

>Note we do not need to `own` the string memory of the arguments because we have
>asserted that both the arguments to `chooser_` will be the _last_ references to
>the strings. However, only one of the C++ strings will be deallocated -- the
>other is returned to us. We _do_ need to `own` the return result, however.

In addition to the Interface Types management, because we are using a
non-trivial C++ structure, we have to invoke the appropriate constructors,
access functions and destructors of our `shared_ptr` structure.

This is achieved through the calls to `shared_builder`, `shared_moveout` and
`shared_release` functions exported by the core WebAssembly module.



