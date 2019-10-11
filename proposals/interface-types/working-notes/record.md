

### Records

Records are collections of named and typed fields. Records can be defined in as
Interface Types; for example this definition encodes a familiar notion of a
credit card:

```wat
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
```

Lifting and lowering a record involves lifting/lowering the individual fields of
the record and then 'packing' or 'unpacking' the record as an ordered
sub-sequence of elements on the stack.

For example, the sequence:

```wat
...
local.get $ccNo
i64-to-u64
local.get $name
local.get $name.len
memory-to-string
i8.local.get $mon
i8_to_u8
i16.local.get $year
i16_to_u16
pack @(record (mon u8) (year u16))
local.get $ccv
pack @cc.cc
...
```
creates a credit card information record from various local variables.

Lowering a record is complimentary, involving an `unpack` operator; together
with individual lowering operators for the fields:

```wat
...
unpack @cc.cc $ccNo $name $expires $ccv
  local.get $ccNo
  u64-to-i64
  local.get $name
  string-to-memory "malloc"
  local.get $expires
  unpack @(record (mon u8) (year u16)) $mon $year
    local.get $mon
    u8-to-i32
    local.get $year
    u16-to-i32
  end
  local.get $ccv
  u16-to-i32
end
```

The result of this unpacking is to leave the fields of the record on the
stack. If the intention were to store the record in memory then this code could
be augmented with memory store instructions.

