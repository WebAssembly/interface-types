# Pay by Credit Card

In order to process a payment with a credit card, various items of information
are required: credit card details, amount, merchant bank, and so on. This
example illustrates the use of nominal types and record types in adapter code.

>Note that we have substantially simplified the scenario in order to construct
>an illustrative example.

The signature of `payWithCard` as an interface type is:

```
cc ::= {
  ccNo : u64;
  name : string;
  expires : {
    mon : u8;
    year : u16
  }
  ccv : u16
}
payWithCard:(card:cc, amount:s64, session:resource<connection>) => boolean
```

## Export

In order to handle the incoming information, a record value must be passed --
including a `string` component -- and a `session` that denotes the bank
connection must be paired with an existing connection resource. The connection
information is realized as an `eqref` (an `anyref` that supports equality).

In this scenario, we are assuming that the credit card information is passed by
value (each field in a separate argument) to the internal implementation of the
export, and passed by memory reference to the import.

Lowering a record from interface types into its constituent components is
handled by the `unpack` instruction.

```
(@interface datatype $ccExpiry
  (record
    (field "mon" u8)
    (field "year" u16)
  )
)
(@interface datatype $cc 
  (record
    (field "ccNo" u64)
    (field "name" string)
    (field "expires" (type $ccExpiry))
    (field "ccv" u16)
  )
)

(@interface typealias $connection eqref)

(memory (export $memx) 1)

(func $payWithCard_ (export ("" "payWithCard_"))
  (param i64 i32 i32 i32 i32 i32 i64 eqref) (result i32)

(@interface func (export "payWithCard")
  (param $card (type $cc))
  (param $amount s64)
  (param $session (resource (type $connection)))
  (result boolean)

  local.get $card
  unpack (type $cc) $ccNo $name $expires $ccv
    local.get $ccNo   ;; access ccNo
    u64-to-i64

    local.get $name
    string-to-memory "mem1" "malloc"
  
    local.get $expires
    unpack (type $ccExpiry) $mon $year
      local.get $mon
      local.get $year
    end
    
    local.get $ccv
  end
  
  local.get $amount
  s64-to-i64
  
  local.get $session
  resource-to-eqref (type $connection)
  call $payWithCard_
  i32-to-enum boolean
)

(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```

An `unpack type $x $y .. end` instruction sequence is equivalent to:

```wat
unpack <type>
let $y
  let $x
    ..
  end
end
```

I.e., the effect of the `unpack` instruction is to take a record off the stack
and replace it with the fields of the records, in field order.

If local variables are specified in the instruction then this is viewed as
syntactic sugar for unpacking the record and then binding the local variables to
the fields of the record -- using the appropriate `let` instructions.

## Import

In this example, we assume that the credit details are passed to the import by
reference; i.e., the actual argument representing the credit card information is
a pointer to a block of memory.

Constructing a record from fields involves the use of a `pack` instruction;
which is the complement to the `unpack` instruction used above.

```
(func $payWithCard_ (import ("" "payWithCard_"))
  (param i32 s64 eqref) (result i32)

(@interface func $payWithCard (import "" "payWithCard")
  (param (type $cc) s64 (resource $connection))
  (result boolean))
  
(@interface implement (import "" "payWithCard_")
  (param $cc i32)
  (param $amnt i64)
  (param $conn eqref)
  (result i32)

  local.get $cc
  i64.load {offset #cc_ccNo}
  i64-to-u64

  local.get $cc
  i32.load {offset #cc_name_ptr}

  local.get $cc
  i32.load {offset #cc_name_len}
  memory-to-string "memi"
  
  local.get $cc
  i16.load_u {offset #cc_expires_mon}

  local.get $cc
  i16.load_u {offset #cc_expires_year}

  pack (type $ccExpiry)
  
  local.get $cc
  i16.load_u {offset #cc_ccv}

  pack (type $cc)
  
  local.get $amnt
  i64-to-s64
  
  local.get $conn
  eqref-to-resource (type $connection)
  
  call-import $payWithCard
  enum-to-i32 boolean
)
```

The constants of the form `#cc_name_ptr` refer to offsets within the credit card
structure as laid out in memory.

## Adapter

Combining the import, exports and distributing the arguments, renaming `cc` for
clarity, we initially get:

```
(func $Mi:payWithCard
  (param $cc i32)
  (param $amnt i64)
  (param $conn eqref)
  (result i32)
  
  local.get $cc
  i64.load {offset #cc_ccNo}
  i64-to-u64

  local.get $cc
  i32.load {offset #cc_name_ptr}

  local.get $cc
  i32.load {offset #cc_name_len}
  memory-to-string "memi"
  
  local.get $cc
  i16.load_u {offset #cc_expires_mon}

  local.get $cc
  i16.load_u {offset #cc_expires_year}
  
  pack (type $ccExpiry)
  
  local.get $cc
  i16.load_u {offset #cc_ccv}

  pack (type $cc)
  
  local.get $amnt
  i64-to-s64
  
  local.get $conn
  eqref-to-resource (type $connection)
  
  let $session (resource (type $connection))
  let $card (type $cc)
    unpack (type $cc) $ccNo $name $expires $ccv
      local.get $ccNo   ;; access ccNo
      u64-to-i64

      local.get $name
      string-to-memory "mem1" "malloc"

      local.get $expires
      unpack (type $ccExpiry) $mon $year
        local.get $mon
        local.get $year
      end
    
      local.get $ccv
    end
  
    local.get $session
    resource-to-eqref (type $connection)
    call $payWithCard_
    i32-to-enum boolean
  end
  end
  enum-to-i32 boolean
)
```

With some assumptions (such as no aliasing, no writing to locals, no
re-entrancy), we can propagate and inline the definitions of intermediates. This
amounts to 'regular' inlining where we recurse into records and match up the
different packed fields with their unpacked counterparts.

```
(func $Mi:payWithCard
  (param (type $cc) i32)
  (param $amnt i64)
  (param $conn eqref)
  (result i32)
  
  local.get $cc
  i64.load {offset #cc_ccNo}

  local.get $cc
  i32.load {offset #cc_name.ptr}

  local.get $cc
  i32.load {offset #cc_name.len}
  string.copy "memi" "memx" "malloc"
  
  local.get $cc
  i16.load_u {offset #cc_expires.mon}

  local.get $cc
  i16.load_u {offset #cc_expires.year}
  
  local.get $cc
  i16.load_u {offset #cc_ccv}
  
  local.get $amnt

  local.get $conn
  call $Mx:payWithCard_
)
```

There would be different sequences for the adapter if the underlying ABI were different --
for example, if structured data were passed as a pointer to a local copy for
example.
