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

shared_string nondeterministic_choice(shared_string one, shared_string two) {
  return random() > 0.5 ? std::move(one) : std::move(two);
}
```

This function takes two string arguments and returns one of them. Regardless of
the merits of this particular function, it sets up significant challenges should
we want to expose it using Interface Types. Specifically, there are two
resources entering the function, with just one leaving. However, when exposed as
an Interface Type function, all these resources must be created and properly
disposed of within the adapter code itself.

This note focuses on the techniques that enable this to be acheived reliably.

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
  string.size
  call $malloc
  own (i32)
    call $free
  end
  let (local $l owned<i32>)
    local.get $left
    local.get $l
    own.project ;; pick up actual resource
    local.get $left
    string.size
    string-to-memory "memx"
    local.get $right
    string.size
    call $malloc
    own (i32)
      call $free
    end
    let (local $r owned<i32>)
      local.get $right
      local.get $r
      own.project
      local.get $right
      string.size
      string-to-memory "memx"
      
```

