# Genome 124 — canonical-type-collision

## Hypothesis
The canonical type system (canonical-types.cc) uses hash-based deduplication to assign shared canonical IDs to structurally identical types across modules. After the kMaxCanonicalTypes bump to 2M (commit 4a263486400, reverted once for "legacy code bugs"), two distinct recursive type groups that hash-collide could be falsely equated by the equality check in CanonicalizeGroup. If two types with different field layouts share one canonical ID, ref.cast using that ID would accept objects of the wrong type — a parent struct with N fields passes a cast meant for a child struct with N+M fields, enabling OOB field access.

## DNA
- **canonical-types.h:28**: `kMaxCanonicalTypes = 2 * kV8MaxWasmTypes = 2'000'000`
- **value-type.h:247**: `PayloadField = RefTypeKindField::Next<uint32_t, 21>` — 21 bits, max 2^21 = 2,097,152. Only 97K headroom above 2M.
- **canonical-types.h:34**: `static_assert(kMaxCanonicalTypes <= (1 << CanonicalValueType::kNumIndexBits))` — tight fit.
- **Commit 21471821f74**: Revert reason: "need to migrate legacy code first, otherwise this causes bugs" (Bug 452635472). Reland 4a263486400 (Dec 2025) claims legacy code was removed.
- **canonical-types.cc:444-452**: Hash function uses `CanonicalHashing` with relative indices. Line 258-261: descriptor indices hashed with only 11-bit shift. Comment: "collisions in hashing are OK" — but collisions lead to equality comparison which may have gaps.
- **canonical-types.cc:386-391**: Overflow check happens AFTER computing `new_index = canonical_recgroup_start.index + (...)`. If this computation overflows before the check, truncation could map distinct types to same index.
- **CRITICAL: wasm-subtyping.cc:14-17**: `EquivalentIndices()` compares ONLY canonical type IDs: `module->canonical_type_id(index1) == module->canonical_type_id(index2)`. **Nullability info is LOST** at this level — two types with different nullability but same canonical index are treated as equivalent. This violates the nullability check at line 485.
- **CRITICAL: value-type.h:383-392 (via canonical-types.h EqualValueType)**: Exactness flag IS part of `is_equal_except_index()` comparison (line 1001-1003), but in hash canonicalization, exactness mismatches may not be caught if hash collision sends us to wrong bucket.
- **wasm-subtyping.cc:404-407**: Exact type subtyping: `if (supertype.is_exact() && !subtype.is_exact()) return false;`. But line 475-476: `if (sub_index == super_index) return true` — ignores exactness! A `ref T` would be subtype of `ref exact T` if they share canonical index.
- **CVE-2024-2887**: Used type ID manipulation for type confusion. CVE-2024-6100: 20-bit HeapTypeField overflow. CVE-2024-8194: canonical type confusion. CVE-2025-5959: nullability ignored in equality.
- **wasm-export-wrapper-cache.h:66**: `static_assert(kMaxCanonicalTypes < (1u << 21))` — tight constraint.

## Techniques That Work
1. Create two modules with large recursive type groups that are structurally similar but differ in one field type (e.g., i32 vs i64 in a deeply nested ref field)
2. Engineer hash collisions by manipulating the recursive structure
3. Use cross-module imports to bring both types into one instance
4. Cast between the collision pair
5. For PayloadField overflow: create close to 1M types in a single module, then instantiate twice to push canonical count toward 2M

## DO NOT Try These
- Shared types (--experimental-wasm-shared, BANNED)
- Simple non-recursive types — these canonicalize trivially without hash collisions
- Type groups that differ in obvious ways (different number of types) — equality check catches these immediately

## Evolution Plan
- v1: **Nullability loss in EquivalentIndices** — the most promising angle. Read wasm-subtyping.cc:14-17 (EquivalentIndices) and line 481-507 (EquivalentTypes). Verify that nullability checked at line 485 is bypassed by canonical ID comparison at line 17. Build test: create `(ref $T)` and `(ref null $T)` in same module, verify they get same canonical ID. Then use ref.cast from nullable to non-nullable — does the optimizer see them as equivalent and eliminate the cast? If cast is eliminated, null passes through → null dereference on field access.
- v2: **Exactness bypass via canonical index equality**. Read wasm-subtyping.cc:475-476 (`if (sub_index == super_index) return true`). This ignores exactness! Create `ref $T` and `ref exact $T` with same canonical index. Test if ref.cast to `ref exact $T` is eliminated because sub_index == super_index → always-succeeds. Then pass a subtype instance through the "exact" cast → access fields that only exist on the exact type.
- v3: **Hash collision engineering**. If v1/v2 are clean, focus on canonical-types.cc hash function. Create two structurally different recursive groups that hash to the same bucket. Check if the equality comparator in FindCanonicalGroup (line 61-74) correctly distinguishes them or falsely merges them. Especially target: groups differing only in descriptor relationships (11-bit shift at line 258-261).

## Task
Start by reading:
1. `src/wasm/canonical-types.cc` — full file. Focus on hash computation (look for `base::hash_combine` or similar), the `CanonicalizeGroup` method, and how recursive type references within a group are handled during comparison.
2. `src/wasm/canonical-types.h` — the TypeCanonicalizer class interface, kMaxCanonicalTypes, segment storage.
3. `src/wasm/value-type.h:710-725` — the ValueTypeBase constructor with index, the CHECK at line 720.

Then build a test:
- Module A with rec group: struct $S0 { i32 }, struct $S1 extends $S0 { ref $S0 }
- Module B with rec group: struct $S0 { i32 }, struct $S1 extends $S0 { ref $S1 } (self-referencing)
- These groups are structurally different but similar
- Export functions from both, import into a third module, attempt cross-type casts

Run with: `/Users/t/v8/out/fuzzbuild/d8 test.js`
No --experimental-wasm-shared flag.
