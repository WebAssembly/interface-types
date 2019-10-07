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

Consuming a sequence 

```

```
