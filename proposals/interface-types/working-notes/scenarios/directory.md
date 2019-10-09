# Directory Listing

In this example, we look as an API to generate a list of files in a directory. 

The interface type signature for this function is:

```
listing:(string)=>sequence<string>
```

Since the caller does not know how many file names will be returned, it has to
protect itself from potentially abusive situations. That in turn means that the
allocation of the string sequence in the return value may fail.

In order to avoid memory leaks, we protect the adapter with exception handling
-- whose purpose it to clean up should an allocation failure occur

In addition, memory allocated by the internal implementation of `$listing_`
needs to be released after the successful call.

## Export

```
(memory (export "memx") 1)

(@interface func (export "listing")
  (param $dir string)
  (result (sequence string))

  local.get $dir
  string-to-memory "memx" "malloc"
  call $it_opendir_
  
  iterator.start
    sequence.start string
    iterator.while $it_loop
      call $it_readdir_
      dup
      eqz
      br_if $it_loop
      memory-to-string "memx"
      sequence.append
    end
    iterator.close
      call $it_closedir
      sequence.complete
    end
  end
)
...
```

The `sequence.start`, `sequence.append` and `sequence.complete` instructions are
used to signal the creation of a sequence of values.

In this particular example, the export adapter is not simply exporting an
individual function but is packaging a combination of three functions that,
together, implement the desired interface. This is an example of a situation
where the C/C++ language is not itself capable of realizing a concept available
in the interface type schema.

The `$it_opendir_`, `$it_readdir_` and `$it_closedir_` functions are intended to
denote variants of the standard posix functions that have been slightly tailored
to better fit the scenario.

The `iterator.start`, `iterator.while` and `iterator.close` instructions model
the equivalent of a `while` loop. The body of the `iterator.start` consists of
three subsections: the initialization phase, an `iterator.while` instruction
which embodies the main part of the iteration and the `iterator.close` whose
body contains instructions that must be performed at the end of the loop.

The `iterator.while` instruction repeats its internal block until specifically
broken out of; it is effectively equivalent to the normal wasm `loop`
instruction.

## Import

Consuming a sequence involves running an iterator over the sequence, calling a
local allocator function for each found element. This is driven by the
`sequence.loop` control instruction which repeats its body once for each element
of the sequence.

In this example, we assume that the local `$startList` and `$appendToList`
functions can be used to allocate a list structure to collect the strings from
the directory listing. In addition, the `$mkStr` function takes two `i32`
numbers and creates a local pair that can hold a string value.

```
(func $listing_ (import ("" "listing_"))
  (param i32 i32) (result i32)

(@interface func $listing (import "" "listing")
  (param @url string)
  (result (sequence string)))

(@interface implement (import "" "listing_")
  (param $text i32)
  (param $count i32)
  (result i32)

  local.get $text
  local.get $count
  memory-to-string

  call $listing

  call $startList
  let $list

  sequence.loop
    string-to-memory "memi" "malloc"
    call $mkStr
    local.get $list
    call $appendToList
  end

  local.get $list
)
```

## Adapter

The usual first step...

```
(@adapter implement (import "" "listing_")
  (param $text i32)
  (param $count i32)
  (result i32)

  local.get $text
  local.get $count
  memory-to-string

  let $dir
    local.get $dir
    string-to-memory "memx" "malloc"
    call $it_opendir_

    iterator.start
      sequence.start string
      iterator.while $it_loop
        call $it_readdir_
        dup
        eqz
        br_if $it_loop
        memory-to-string "memx"
        sequence.append
      end
      iterator.close
        call $it_closedir
        sequence.complete
      end
    end
  end

  call $startList
  let $list

  sequence.loop
    string-to-memory "memi" "malloc"
    call $mkStr
    local.get $list
    call $appendToList
  end

  local.get $list
)
```

After minor cleanup, and replacement of the `sequence.append` instruction with
the body of the `sequence.loop`:

```
(@adapter implement (import "" "listing_")
  (param $text i32)
  (param $count i32)
  (result i32)

  local.get $text
  local.get $count
  string.copy "memi" "memx" "malloc"

  local.get $count
  call mx:$it_opendir_

  call $startList
  let $list

  iterator.start
    iterator.while $it_loop
      call mx:$it_readdir_
      dup
      eqz
      br_if $it_loop
      memory-to-string "memx"
      string-to-memory "memi" "malloc"
      call $mkStr
      local.get $list
      call $appendToList
    end
    iterator.close
      call mx:$it_closedir
    end
  end
  local.get $list
)
```

After cleanup and replacement of `iterator.*` instructions by their normal wasm
counterparts:


```
(@adapter implement (import "" "listing_")
  (param $text i32)
  (param $count i32)
  (result i32)

  local.get $text
  local.get $count
  string.copy "memi" "memx" "malloc"

  local.get $count
  call mx:$it_opendir_

  call $startList
  let $list

  loop $it_loop
    call mx:$it_readdir_
    dup
    eqz
    br_if $it_loop
    string.copy "memx" "memi" "malloc"
    call $mkStr
    local.get $list
    call $appendToList
  end
  call mx:$it_closedir
  local.get $list
)
```
