# Pay by Credit Card

In order to process a payment with a credit card, various items of information
are required: credit card details, amount, merchant bank, and so on. This
example illustrates the use of nominal types and record types in adapter code.

>Note that we have substantially simplified the scenario in order to construct
>an illustrative example.

The signature of `payWithCard` as an interface type is:

```
cc ::= cc{
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
export, and passed by reference to the import.

```
(@interface datatype @cc 
  (record "cc"
    (ccNo u64)
    (name string)
    (expires (record
      (mon u8)
      (year u16)
      )
    )
    (ccv u16)
  )
)

(@interface typealias @connection eqref)

(memory (export $memx) 1)

(func $payWithCard_ (export ("" "payWithCard_"))
  (param i64 i32 i32 i32 i32 i32 eqref) (result i32)

(@interface func (export "payWithCard")
  (param $card @cc)
  (param $session (resource @connection))
  (result boolean)

  local.get $card
  field.get #cc.ccNo   ;; access ccNo
  u64-to-i64

  local.get $card
  field.get #cc.name
  string-to-memory $mem1 "malloc"
  
  local.get $card
  field.get #cc.expires.mon

  local.get $card
  field.get #cc.expires.year

  local.get $card
  field.get #cc.ccv
  
  local.get $session
  resource-to-eqref @connection
  call $payWithCard_
  i32-to-enum boolean
)

(func (export "malloc")
  (param $sze i32) (result i32)
  ...
)
```

## Import

In this example, we assume that the credit details are passed to the import by
reference; i.e., the actual argument representing the credit card information is
a pointer to a block of memory.

```
(func $payWithCard_ (import ("" "payWithCard_"))
  (param i32 eqref) (result i32)

(@interface func $payWithCard (import "" "payWithCard")
  (param @cc (resource @connection))
  (result boolean))
  
(@interface implement (import "" "payWithCard_")
  (param $cc i32)
  (param $conn eqref)
  (result i32)

  local.get $cc
  i64.load {offset #cc.ccNo}
  i64-to-u64

  local.get $cc
  i32.load {offset #cc.name.ptr}

  local.get $cc
  i32.load {offset #cc.name.len}
  memory-to-string "memi"
  
  local.get $cc
  i16.load_u {offset #cc.expires.mon}

  local.get $cc
  i16.load_u {offset #cc.expires.year}

  create (record (mon u16) (year u16))
  
  local.get $cc
  i16.load_u {offset #cc.ccv}

  create @cc
  
  local.get $conn
  eqref-to-resource @connection
  
  call $payWithCard
  enum-to-i32 boolean
)
```

## Adapter

Combining the import, exports and distributing the arguments, renaming `cc` for
clarity, we initially get:

```
(@adapter implement (import "" "payWithCard_")
  (param $cc i32)
  (param $conn eqref)
  (result i32)
  
  local.get $cc
  i64.load {offset #cc.ccNo}
  i64-to-u64

  local.get $cc
  i32.load {offset #cc.name.ptr}

  local.get $cc
  i32.load {offset #cc.name.len}
  memory-to-string "memi"
  
  local.get $cc
  i16.load_u {offset #cc.expires.mon}

  local.get $cc
  i16.load_u {offset #cc.expires.year}

  create (record (mon u16) (year u16))
  
  local.get $cc
  i16.load_u {offset #cc.ccv}

  create @cc
  
  local.get $conn
  eqref-to-resource @connection

  let $session (resource @connection)
  let $card @cc

  local.get $card
  field.get #cc.ccNo   ;; access ccNo
  u64-to-i64

  local.get $card
  field.get #cc.name
  string-to-memory $mem1 "malloc"
  
  local.get $card
  field.get #cc.expires.mon
  u16-to-i32

  local.get $card
  field.get #cc.expires.year
  u16-to-i32

  local.get $card
  field.get #cc.ccv
  u16-to-i32
  
  local.get $session

  resource-to-eqref @connection
  call $payWithCard_
  i32-to-enum boolean
  end
  end
  enum-to-i32 boolean
)
```

With some assumptions (such as no aliasing, no writing to locals, no
re-entrancy), we can propagate and inline the definitions of intermediates. This
amounts to 'regular' inlining where we recurse into records and treat the fields
of the record in an analogous fashion to arguments to the call.

```
(@adapter implement (import "" "payWithCard_")
  (param $cc i32)
  (param $conn eqref)
  (result i32)
  
  local.get $cc
  i64.load {offset #cc.ccNo}

  local.get $cc
  i32.load {offset #cc.name.ptr}

  local.get $cc
  i32.load {offset #cc.name.len}
  string.copy Mi:"mem2" Mx:"memx" "malloc"
  
  local.get $cc
  i16.load_u {offset #cc.expires.mon}

  local.get $cc
  i16.load_u {offset #cc.expires.year}

  local.get $cc
  i16.load_u {offset #cc.ccv}
  
  local.get $conn
  call $payWithCard_
)
```

There would be different sequences for the adapter if the underlying ABI were different --
for example, if structured data were passed as a pointer to a local copy for
example.
