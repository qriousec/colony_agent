# Genome 116 — exactness-shared-canonical

## Hypothesis
CanonicalValueType encodes 4 orthogonal modifier bits for indexed ref types: nullable (bit 2), exact (bit 3), shared (bit 4), kind (bits 5-7). The exactness bit was added recently for custom descriptors and is the least battle-tested. When recursive type groups include types differing only in exactness or shared-ness, the canonicalizer must preserve all 4 bits in both hashing (CanonicalHashing.Add) and equality (EqualValueType). If any code path in canonicalization, subtype checking, or type-indexed operations strips or conflates these bits, two semantically different types could get the same canonical ID — causing ref.cast to succeed where it should fail, or vice versa.

## DNA

### Source Facts (file:line)
- **Bit layout**: value-type.h:238-247 — IsNullable(bit 2), IsExact(bit 3), IsShared(bit 4), RefTypeKind(bits 5-7), Payload(bits 8-28)
- **all_bits_without_index()**: value-type.h:995-998 — `bit_field_ & ~kIndexBits` preserves nullable+exact+shared+kind
- **is_equal_except_index()**: value-type.h:1001-1003 — uses all_bits_without_index() for comparison
- **EqualValueType**: canonical-types.h:383-392 — indexed: is_equal_except_index() + EqualTypeIndex(); non-indexed: operator==
- **CanonicalHashing.Add**: canonical-types.h:275-289 — for relative indexed types, hashes all_bits_without_index() + shifted relative index
- **Canonicalize ValueType**: canonical-types.cc:394-398 — CanonicalizeValueType lambda converts type indices
- **CVE-2025-5959**: Nullability was ignored in EqualValueType — fixed
- **Exactness in GetExactness**: wasm-compiler-definitions.cc:21-39 — returns kExactMatchOnly for final types with/without descriptors
- **IsCastToCustomDescriptor**: wasm-gc-typed-optimization-reducer.h:21-26 — guard for descriptor casts

### Gap Analysis
1. **Exactness in hash function**: CanonicalHashing.Add hashes all_bits_without_index(). Since exactness is bit 3, it's included. BUT: are there ANY alternate hashing paths that don't use all_bits_without_index()?
2. **Exactness in subtype checking**: wasm-subtyping.cc IsSubtypeOfImpl — does it handle exact types correctly? exact(T) <: T but T is NOT <: exact(T).
3. **Shared + exact interaction**: Can you create (ref exact shared $struct)? If so, how does the canonicalizer handle it? Is there a test for this combination?
4. **RecGroup with exact/non-exact mixed**: A rec group containing both (ref exact $A) and (ref $A) as field types — do they canonicalize differently?
5. **Populate() method**: value-type.h:363-366 — updates shared and kind bits but NOT exactness. If Populate is called on a type that was created with exact=true, does it preserve exactness?

## Techniques That Work
1. Create recursive type groups where one field is exact and another isn't
2. Create two modules with structurally identical types but different exact annotations
3. Cross-module import/export with exact vs non-exact function signature
4. Test ref.cast between exact and non-exact variants of same base type
5. Use custom descriptors (--experimental-wasm-custom-descriptors) to create exact types

## DO NOT Try These
- Simple nullability-only tests — CVE-2025-5959 was about this, already patched
- Canonical type ID overflow tests — W1 mitigated by kMaxCanonicalTypes
- Descriptor exactness inlining gap — dead surface

## Evolution Plan
- v1: Source audit of exactness bit handling across all canonicalization paths. Read canonical-types.cc, canonical-types.h, wasm-subtyping.cc. Search for "exact" and "is_exact()" to find all sites. Check if Populate() preserves exactness. Check if any comparison path uses operator== on raw bit_field_ (which includes exact bit) vs type-level equality.
- v2: Build test: two rec groups identical except one field is (ref exact $struct) vs (ref $struct). Check if they canonicalize to same or different IDs. If same → bug. Test with cross-module import using exact vs non-exact signature.
- v3: Test shared+exact combination: (ref exact shared $struct). Build module with shared struct where one function returns exact ref and another returns non-exact. Check if ref.cast/ref.test distinguish them correctly.

## Task
1. Read value-type.h:363-366 (Populate method) — check if exactness is preserved.
2. Grep for "is_exact\|kExact\|exact" in canonical-types.cc, canonical-types.h — find all exactness handling.
3. Grep for "is_exact\|exact" in wasm-subtyping.cc — find exactness in subtype checks.
4. Read canonical-types.h:275-320 (CanonicalHashing and CanonicalEquality) — verify exactness in hash+eq.
5. Read wasm-subtyping.cc around IsSubtypeOfImpl — check exact type handling.
6. Build test following v2 plan: two structurally-identical rec groups differing only in exactness.
7. Run with: `--experimental-wasm-custom-descriptors --no-wasm-lazy-compilation`
8. Start with Evolution Plan v1 (source audit of exactness bit handling).
