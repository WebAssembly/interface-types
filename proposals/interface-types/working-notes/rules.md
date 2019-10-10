# Sketch of Interface Types transformation Rules

## Lifting operators

i32-to-s32

memory-to-string 

memory-to-array Instr* end

pack @record

## Lowering operators

s32-to-i32

string-to-memory <Allocator>

array-to-memory size <Allocator> Instr* end

unpack @record 

## Combination Rules

i32-to-s32 s32-to-i32 ==> empty

memory-to-string string-to-memory <Allocator> ==> string.copy <Allocator>

memory-to-array InstrA end array-to-memory <Size> <Allocator> InstrB end ==>
  memory.loop <Size> InstrA InstrB end

