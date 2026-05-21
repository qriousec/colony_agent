# Genome 127 — nullability-exactness-subtype

## Hypothesis
Two specific code gaps in V8's Wasm subtyping system combine for type confusion:

1. **EquivalentIndices (wasm-subtyping.cc:14-17)** compares ONLY canonical type IDs. Two types `(ref $T)` and `(ref null $T)` share the same canonical index, so EquivalentIndices returns true — losing the nullability difference checked at the higher-level EquivalentTypes (line 485). If any code path calls EquivalentIndices directly instead of EquivalentTypes, nullable and non-nullable refs become equivalent.

2. **IsCanonicalSubtype (canonical-types.cc:322-336)** returns true when `sub_index == super_index` (line 326) without checking exactness. This means `(ref $T)` passes as subtype of `(ref exact $T)` because they share canonical index. A non-exact reference passes an exact-type guard.

Combined: a `(ref null $T)` could pass through an exact non-nullable cast `(ref exact $T)`, enabling null dereference when fields are accessed on the "guaranteed non-null, exact" value.

## DNA
- **wasm-subtyping.cc:14-17**: `EquivalentIndices()` — compares `module->canonical_type_id(index1) == module->canonical_type_id(index2)`. Ignores nullability entirely.
- **wasm-subtyping.cc:481-507**: `EquivalentTypes()` — checks nullability at line 485: `if (type1.nullability() != type2.nullability()) return false;`. Then calls `EquivalentIndices` at line 489. If any caller uses EquivalentIndices instead of EquivalentTypes, the nullability check is bypassed.
- **canonical-types.cc:322-336**: `IsCanonicalSubtype()` fast path: `if (sub_index == super_type.ref_index()) return true;` (line 326). The exact check at line 329 (`if (super_type.is_exact()) return false;`) happens AFTER the equality check. So for same-index types, exactness is ignored.
- **wasm-subtyping.cc:404-407**: `if (supertype.is_exact() && !subtype.is_exact()) return false;` — exactness check in IsSubtypeOfImpl for HeapType. But line 475-476: `if (sub_index == super_index) return true` — early return IGNORES exactness.
- **wasm-subtyping.cc:454-459**: Nullability is checked at ValueType level, then dropped when HeapTypes are extracted for IsSubtypeOfImpl.
- **CVE-2025-5959**: Nullability was previously ignored in EqualValueType — closely related pattern.
- **value-type.h:247**: PayloadField is 21 bits, kMaxCanonicalTypes = 2M. Canonical IDs encode type index only, not nullability or exactness.
- **Parent genome 124 (canonical-type-collision)**: 2 iterations, 0 posts. Unknown what was tested. Planned v1=nullability, v2=exactness, v3=hash collision.

## Techniques That Work
1. Create a type $T (struct with fields), then use both `(ref $T)` and `(ref null $T)` in the same module
2. Create function that takes `(ref $T)` (non-nullable) — should trap on null
3. Call with null through a path where EquivalentIndices is used for type checking
4. Create `(ref exact $T)` cast — should only accept exactly $T, not subtypes
5. Try passing a subtype through a path where sub_index == super_index bypasses the exact check
6. Use Turboshaft tier-up so the optimized code uses the IsCanonicalSubtype fast path

## DO NOT Try These
- Shared types (--experimental-wasm-shared, BANNED)
- Simple ref.cast null checks — these are implemented at the Cast operation level, not via subtyping
- Module-level validation errors — the validator may catch obvious type mismatches

## Evolution Plan
- v1: **EquivalentIndices caller audit**. Search the codebase for ALL callers of EquivalentIndices (vs EquivalentTypes). For each caller, check if it relies on nullability being checked elsewhere or if it's vulnerable. Build test: two function signatures differing only in nullability of a ref parameter — do they canonicalize to the same signature? If so, call_indirect with wrong nullability.
- v2: **IsCanonicalSubtype exactness bypass**. The fast path at canonical-types.cc:326 returns true for same index before checking exact at line 329. Build test: create `(ref exact $Parent)` type guard. Pass a `$Child` instance through `ref.cast (ref $Parent)` first (succeed), then access it as `(ref exact $Parent)`. The canonical subtype check sees same-index and returns true without verifying exactness. If `$Child` has extra fields, this could allow reading fields beyond `$Parent`'s layout.
- v3: **Combined nullability + exactness at JS boundary**. The Turboshaft wrapper's FromJS/ToJS uses canonical types. Test: JS passes null to Wasm function expecting `(ref exact $T)`. If the JS-to-Wasm wrapper uses IsCanonicalSubtype for validation (same-index fast path), null could pass through. Then Wasm code accesses fields on null → crash or controlled read.

## Task
Start by reading these files in order:
1. `src/wasm/wasm-subtyping.cc` — Full file. Focus on:
   - EquivalentIndices (line 14-17) — who calls this?
   - EquivalentTypes (line 481-507) — proper nullability check
   - IsSubtypeOfImpl for HeapType (line 404-476) — exactness handling, especially line 475 early return
2. `src/wasm/canonical-types.cc` — IsCanonicalSubtype (line 322-348), fast path
3. Run: `grep -rn 'EquivalentIndices' /Users/t/v8/src/` to find ALL callers

Then build test 1: Two function types differing only in nullability of a parameter:
- Type A: (ref $S) -> i32
- Type B: (ref null $S) -> i32
- Do they get the same canonical signature?
- Use call_indirect with table containing function of type A, call with type B signature
- Pass null through the call → does it reach the function body?

Run with: `/Users/t/v8/out/fuzzbuild/d8 test.js`
No --experimental-wasm-shared flag.
