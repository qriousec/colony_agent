# Genome 014 — cross-module-exact

## Hypothesis
When a cross-module function import or table operation involves exact type requirements (custom descriptors), the type checking at the JSToWasmObject boundary (wasm-objects.cc:3546-3596) has an ASYMMETRY: WasmExportedFunction uses `IsCanonicalSubtype(real_type_index, expected)` which is exactness-aware (via CanonicalValueType), but WasmJSFunction and WasmCapiFunction use `MatchesSignature(canonical_index)` which is index-only and does NOT check exactness. If a table or global with exact function type is populated via `WebAssembly.Function` (WasmJSFunction path) whose signature is a proper subtype, the MatchesSignature path may accept it while the WasmExportedFunction path would reject — or vice versa, depending on how MatchesSignature handles subtyping. This inconsistency at the boundary between two code paths performing the same logical check (is this function compatible with the expected type?) is a classic type confusion vector.

Secondary factor: The `IsCanonicalSubtype` API was recently changed (48c35511792) to take `CanonicalValueType` instead of `CanonicalTypeIndex` to handle exactness. Callers that construct `expected` from table metadata must preserve exactness. If `WasmTableObject::canonical_type()` strips exactness, the check would be more permissive than intended.

## DNA

### Source Facts (with file:line references)

**IsCanonicalSubtype API (canonical-types.cc:322-336)**:
```cpp
bool TypeCanonicalizer::IsCanonicalSubtype(CanonicalTypeIndex sub_index,
                                           CanonicalValueType super_type) {
  DCHECK(super_type.has_index());
  if (sub_index == super_type.ref_index()) return true;  // equality = pass
  if (super_type.is_exact()) return false;  // exact + not equal = fail
  return IsCanonicalSubtype_Locked(sub_index, super_type.ref_index());
}
```
- Takes CanonicalValueType for super, which carries exactness.
- If super is exact, ONLY equality passes. Subtypes rejected.
- If super is non-exact, walks the supertype chain.

**JSToWasmObject (wasm-objects.cc:3546-3596)**:
- WasmExportedFunction path (line 3554): `IsCanonicalSubtype(real_type_index, expected)` — EXACTNESS CHECKED
- WasmJSFunction path (line 3566): `MatchesSignature(canonical_index)` — INDEX ONLY, NO EXACTNESS
- WasmCapiFunction path (line 3575): `MatchesSignature(canonical_index)` — INDEX ONLY, NO EXACTNESS
- WasmStruct/Array path (line 3587): `IsCanonicalSubtype(actual_type, expected)` — EXACTNESS CHECKED

**WasmTableObject::IsValidElement (wasm-objects.cc:535-542)**:
- Uses `IsCanonicalSubtype(sig_id, table_type)` — EXACTNESS CHECKED for table operations.

**module-instantiate.cc:745-746**:
- `IsCanonicalSubtype(data->internal()->sig()->index(), expected_type)` — import checking uses full CanonicalValueType.

**Custom descriptor subtyping fix (48c35511792)**:
- Changed IsCanonicalSubtype to take CanonicalValueType instead of CanonicalTypeIndex
- Fixed ProcessBranchOnTarget for custom descriptors
- Fixed JSToWasmObject missing exact type support
- Fixed ref.get_desc returning wrong exactness

**kMaxCanonicalTypes bumped to 2M (4a263486400)**:
- Uses 1 of 4 unused bits in ValueType for more type indices
- `static_assert(kMaxCanonicalTypes <= (1 << CanonicalValueType::kNumIndexBits))`

### Related CVEs
- CVE-2024-2887: Canonical type confusion via TypeCheckAlwaysSucceeds
- CVE-2024-6100: 20-bit truncation of canonical type IDs
- CVE-2024-8194: Type confusion in cross-module contexts

### Guards Known
1. `IsCanonicalSubtype` with CanonicalValueType handles exactness (canonical-types.cc:329)
2. Validation-time type hierarchy checks (function-body-decoder-impl.h)
3. `WasmTypeCheckConfig::is_valid()` added in c047ab74b85

## Techniques That Work
1. Create two modules sharing types via canonical type indices
2. Module A exports a function with type signature using struct subtypes
3. Module B imports it into a table with exact type requirement
4. Test via `WebAssembly.Function` (WasmJSFunction path) vs Wasm-exported function (WasmExportedFunction path)
5. Use `--experimental-wasm-js-interop` for custom descriptor / exact type support
6. Grep for all callers of `MatchesSignature` to find which skip exactness
7. Check how `WasmTableObject::canonical_type()` constructs its CanonicalValueType

## DO NOT Try These
- Generic ref.cast tests without cross-module context (already tested by desc-exactness, custom-desc-cast)
- Cont type boundary tests (dead surface, confirmed by wasm-to-js-indexed)
- Memory cache reload tests for WasmFX (covered by wasmfx-cont-leak)

## Evolution Plan
- v1: **Source audit of MatchesSignature vs IsCanonicalSubtype asymmetry**. Read `WasmJSFunctionData::MatchesSignature` implementation. Determine if it does equality check or subtype check. Read how `WasmTableObject::canonical_type()` constructs the CanonicalValueType and whether exactness is preserved. Map all callers of MatchesSignature and compare with IsCanonicalSubtype callers at the same logical check point.
- v2: **Cross-module table + exact type test**. Build module A with struct type $Parent (with custom descriptor, exact). Build module B with $Child extends $Parent. Export table from A with `(table (ref exact $Parent))`. From JS, create `WebAssembly.Function` with $Child signature and attempt to store it in the table. Compare behavior between WasmJSFunction path and WasmExportedFunction path.
- v3: **call_indirect dispatch + exact type**. If v2 shows the table accepts the wrong type, test whether `call_indirect` through that table entry causes a type confusion at call time. The call dispatch may trust the table's declared type and skip runtime checks, leading to signature mismatch exploitation.

## Task
Start with v1 (source audit). Read these files in order:
1. `src/wasm/wasm-js.cc` — search for `MatchesSignature` implementation for WasmJSFunction
2. `src/wasm/wasm-objects.cc` — read JSToWasmObject function fully (lines 3500-3650)
3. `src/wasm/wasm-objects-inl.h` — check how WasmTableObject stores and retrieves canonical_type
4. `src/wasm/canonical-types.cc` — read IsCanonicalSubtype (lines 322-336)
5. `src/wasm/canonical-types.h` — read CanonicalValueType definition, check kNumIndexBits and exactness bit layout

Your type invariant to challenge: "All code paths in JSToWasmObject check type compatibility with the same precision, including exactness." The asymmetry between IsCanonicalSubtype (exactness-aware) and MatchesSignature (index-only) suggests this invariant may not hold.

Post your findings to #colony-workers with tag `cross-module,exactness,asymmetry`. Include specific file:line references for every assertion.
