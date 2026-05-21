# Genome 013 — struct-offset-gc

## Hypothesis
Struct field offset computation in V8 (struct-types.h, wasm-objects-inl.h) may produce incorrect offsets when struct types have complex field layouts mixing: (1) packed fields (i8, i16), (2) GC reference fields (ref types), (3) i31ref fields, and (4) alignment padding. If the offset computation for a struct.get at one tier (Liftoff) differs from another tier (Turboshaft), or if the offset computed at compile time doesn't match the actual object layout at allocation time, a field access reads from the wrong offset — type confusion between adjacent fields of different types.

## DNA

### Source Facts
- **StructType::field_offset** (`struct-types.h`): Computes the byte offset of each field in a struct. Must account for field sizes and alignment.
- **Field sizes**: i8 = 1 byte, i16 = 2 bytes, i32/f32 = 4 bytes, i64/f64 = 8 bytes, ref types = tagged size (4 or 8 bytes depending on compression), i31 = tagged size.
- **Alignment**: Fields may need alignment padding. GC reference fields typically need pointer-sized alignment.
- **Struct allocation** (`wasm-objects-inl.h`): `WasmStruct::GcSafeFieldType` returns field type from the struct's map.
- **Compression**: With pointer compression, ref fields are 4 bytes instead of 8. The offset computation must match the compressed size.
- **Mixed field layouts**: A struct with [ref, i8, i16, ref, i32] has complex alignment requirements. Packed fields between ref fields could cause alignment issues.

### Key Attack Vectors
1. **Packed field + ref field alignment**: Create a struct with [i8, ref $T]. The i8 field is 1 byte, but the ref field needs tagged-size alignment. If padding is computed differently at different stages, the ref field offset is wrong.
2. **Large struct with many mixed fields**: Create a struct with 50+ fields alternating between packed and ref types. Accumulating offset errors across many fields could cause a significant drift.
3. **Subtype struct extension**: $Parent has [ref, i8]. $Child extends with [i16, ref]. The child's field offsets depend on the parent's layout. If parent padding isn't accounted for in the child's additional fields, offsets are wrong.
4. **Mutable vs immutable fields**: Does mutability affect field layout? If mutable fields have write barriers that assume a specific offset, a wrong offset corrupts the barrier.
5. **Cross-tier differential**: Liftoff computes field offsets differently from Turboshaft. If they disagree, a struct allocated by Liftoff code but accessed by Turboshaft code has wrong field access.

### Guards Expected
- StructType::field_offset should be computed once and reused
- Both Liftoff and Turboshaft should use the same StructType for offset computation
- Alignment should be handled by the type system, not ad-hoc

## Techniques That Work
1. Read `struct-types.h` — StructType::field_offset computation
2. Read `wasm-objects-inl.h` — how struct fields are accessed at runtime
3. Create structs with intentionally complex layouts: [i8, ref, i16, ref, i8, i64, ref]
4. Test field access for each field in both Liftoff and Turboshaft
5. Differential test: allocate struct in one tier, access in another

## DO NOT Try These
- Simple structs with uniform field types (no alignment issues)
- Structs with only ref fields (all same size, no alignment concern)
- Structs with only numeric fields (no tagged pointer compression concern)

## Evolution Plan
- v1: Read struct-types.h StructType field_offset computation. Understand how packed fields, alignment, and GC refs interact. Identify if there are any documented edge cases or TODO comments.
- v2: If v1 finds potential alignment issues, create a struct with worst-case mixed layout and test field access differentials between tiers. If offset computation looks sound, create very large structs (max fields) and test for accumulating drift.
- v3: If v2 finds a differential, minimize to the specific field layout that causes the offset mismatch. If clean, test subtype extension: child struct extending parent with mixed field types.

## Task
Start by reading:
1. `src/wasm/struct-types.h` — StructTypeBase, field_offset, total_fields_size
2. `src/wasm/wasm-objects-inl.h` — WasmStruct field access, RawField, GcSafeFieldType
3. `src/wasm/wasm-objects.cc` — WasmStruct allocation, field validation
4. `src/wasm/baseline/liftoff-compiler.cc` — search for "struct_get" to see Liftoff field access
5. `src/wasm/turboshaft-graph-interface.cc` — search for "struct_get" to see Turboshaft field access

Type invariant to challenge: "StructType::field_offset produces correct offsets for all field layouts, and both Liftoff and Turboshaft use the same offsets."

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc --no-wasm-lazy-compilation`
