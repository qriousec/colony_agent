# Genome 002 — canonical-hash-collision

## Hypothesis
After the kMaxCanonicalTypes bump from 1M to 2M (21-bit PayloadField), the hash function in `CanonicalEquality` and the recgroup canonicalization pipeline may produce hash collisions or incorrect equality results for type groups containing indices near the 2^20 boundary. Specifically, the `Add(CanonicalValueType)` method in the hash helper (canonical-types.h:275) uses `raw_bit_field()` which includes ALL 32 bits — but the hash combining may lose high-bit information. Additionally, code outside canonical-types.cc that was written when PayloadField was 20 bits may truncate indices.

## DNA

### Source Facts
- **PayloadField is 21 bits** (`value-type.h:247`): `using PayloadField = RefTypeKindField::Next<uint32_t, 21>;`
- **3 reserved bits remain** (`value-type.h:250`): `using ReservedField = PayloadField::Next<uint32_t, 3>;` — These MUST be zero. If any code sets them, comparisons break.
- **kMaxCanonicalTypes = 2M** (`canonical-types.h:28`): `static constexpr size_t kMaxCanonicalTypes = 2 * kV8MaxWasmTypes;`
- **Static assert** (`canonical-types.h:34`): `static_assert(kMaxCanonicalTypes <= (1 << CanonicalValueType::kNumIndexBits));` — confirms 2^21 = 2M fits in 21 bits.
- **Export wrapper cache** (`wasm-export-wrapper-cache.h:66`): `static_assert(kMaxCanonicalTypes < (1u << 21));` — external code also bounds-checks.
- **Hash function** (`canonical-types.h:275-310`): The `CanonicalGroup::Hash` uses:
  - `Add(CanonicalValueType value_type)` which calls `Add(type.raw_bit_field())` — hashes ALL 32 bits.
  - For type indices, `Add(CanonicalTypeIndex index)` maps them through relative offset computation at lines 279-285, shifting by kMaxCanonicalTypes.
  - The hash is a simple multiply-xor combiner via `base::hash_combine`.
- **Equality check** (`canonical-types.h:383-392`): `EqualValueType` uses `is_equal_except_index` (compares bits 0-7) + `EqualTypeIndex` for indexed types.
- **EqualTypeIndex** (`canonical-types.h:277-313`): Maps relative indices by adding kMaxCanonicalTypes. `static_assert(kMaxCanonicalTypes <= kMaxUInt32 / 2)` — overflow guard.
- **Range check at allocation** (`canonical-types.cc:385-397`): When allocating a new canonical index: `if (V8_UNLIKELY(new_index >= kMaxCanonicalTypes))` + `CHECK(PayloadField::is_valid(new_index))` — guards against overflow.
- **Bump was reverted then relanded** (`21471821f74 Revert`, `4a263486400 Reland`) — the first attempt failed, suggesting fragility.
- **OOM details added** (`dda6b46596e`) — canonical types were hitting OOM, suggesting real-world pressure on the limit.

### Key Attack Vectors
1. **Hash collision between groups differing only in bit 20**: If the hash combiner loses the high bit, two different type groups could hash to the same bucket, and `EqualTypeIndex` with relative offsets could accidentally match.
2. **Reserved bits contamination**: If any code path sets bits 29-31 of the bit_field_, the `operator==` (raw bit comparison) would fail even for "equal" types, but `is_equal_except_index` (which masks with `~kIndexBits`) would still check those bits.
3. **Cross-module recgroup matching**: When a large module is decoded, temporary relative indices are computed. If these overflow 21 bits during the shift at line 285, the relative index wraps.
4. **Export wrapper cache key**: If the wrapper cache uses a truncated key, two different canonical types could map to the same wrapper.

### Guards
- `CHECK(PayloadField::is_valid(new_index))` at allocation time prevents invalid indices in new types
- `static_assert` bounds checks are comprehensive in canonical-types.h
- Hash uses full `raw_bit_field()` which includes all 32 bits

## Techniques That Work
1. Create a module with a very large recgroup (close to kV8MaxWasmTypes = 1M types)
2. Create a second module with a similar but slightly different recgroup
3. Check if canonical type indices from the two modules collide
4. Use cross-module imports to exercise canonicalization with high indices
5. Monitor `canonical_supertypes_.size()` growth under repeated module compilation

## DO NOT Try These
- Small type hierarchies (indices will be far below the 20-bit boundary)
- Testing within a single module (indices are relative, collisions happen across modules)
- Fuzzing random type definitions (need targeted high-index pressure)

## Evolution Plan
- v1: Read the hash function implementation in detail (`canonical-types.h` lines 240-320). Map the complete canonicalization pipeline from module decode to canonical index allocation (`canonical-types.cc` AddRecursiveGroup). Identify exactly where the 21-bit index is used in computations that could overflow or truncate.
- v2: If v1 finds a truncation point, create two modules whose recgroups differ only in a way that the truncation makes invisible. If v1 finds the hash is robust, pivot to the relative index shift computation (`kMaxCanonicalTypes` offset) and check for uint32 overflow when groups have many types.
- v3: If v2 identifies a collision, construct a cross-module import that passes a struct of one canonical type where the other canonical type is expected. Verify the cast is not caught because the canonical IDs match after truncation.

## Task
Start by reading these files in order:
1. `src/wasm/canonical-types.h` lines 240-320 (Hash helper class and CanonicalEquality)
2. `src/wasm/canonical-types.cc` lines 30-110 (AddRecursiveGroup — the main canonicalization entry point)
3. `src/wasm/canonical-types.cc` lines 370-400 (index allocation and range checks)
4. `src/wasm/wasm-export-wrapper-cache.h` line 66 (external usage of kMaxCanonicalTypes)
5. Search for any use of `raw_index()` or `ref_index()` that feeds into a computation with limited bit width.

The type invariant to challenge: "Two types with different canonical indices are always distinguishable by the hash function and equality check."

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc`
