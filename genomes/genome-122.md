# Genome 122 — deser-canonical-mapping

## Hypothesis
Wasm code caching serializes compiled code to disk and deserializes it later. During serialization (wasm-serialization.cc:561-566), canonical sig IDs embedded in code relocations are replaced with module-local type IDs via CanonicalSigIdToModuleLocalTypeId. During deserialization (lines 1040-1051), module-local IDs are converted back to canonical IDs via canonical_sig_id(). The mapping function at lines 606-628 builds a reverse map from canonical → module-local using DCHECK-guarded lookup. In release builds, if the canonical ID doesn't exist in the map (e.g., because the module was compiled with a different type configuration than when it was serialized), the lookup produces undefined behavior. For modules with complex GC type hierarchies, type canonicalization changes between V8 versions could cause the round-trip to silently corrupt signature references.

## DNA

### Source Facts (file:line)
- **Serialization reloc**: wasm-serialization.cc:561-566 — replaces canonical sig ID with module-local ID
- **CanonicalSigIdToModuleLocalTypeId**: wasm-serialization.cc:606-628 — builds canonical→module-local map
- **Deserialization reloc**: wasm-serialization.cc:1040-1051 — converts module-local back to canonical
- **canonical_sig_id()**: wasm-module.h:960-962 — looks up canonical ID for module-local type
- **WASM_CANONICAL_SIG_ID reloc**: reloc-info.cc:276-285 — stores/reads bare uint32_t
- **isorecursive_canonical_type_ids**: wasm-module.h:806-809 — per-module canonical ID storage
- **WriteHeader**: wasm-serialization.cc:406-437 — serializes module metadata
- **ReadHeader**: wasm-serialization.cc:902-921 — deserializes and validates metadata

### Gap Analysis
1. **DCHECK-guarded lookup**: At wasm-serialization.cc:627, the lookup uses `it->second` after a DCHECK. In release builds, if the canonical ID isn't in the map, this is UB (accessing end iterator).
2. **Version mismatch**: If serialized code is loaded by a different V8 version where type canonicalization rules changed, canonical IDs may differ. The mapping would either fail silently or produce wrong module-local IDs.
3. **GC types in relocations**: Are GC struct/array type IDs embedded in code as WASM_CANONICAL_SIG_ID relocations? If yes, the signature-centric mapping may not handle them correctly (sigs vs struct/array types have different canonicalization).
4. **Multiple canonical IDs per module-local type**: If a module-local type gets different canonical IDs in different compilation runs (e.g., due to rec group ordering), the serialized→deserialized mapping breaks.
5. **Shared types**: Do shared types have separate canonical IDs from their non-shared counterparts? If the serializer doesn't account for this, shared/non-shared could be conflated during round-trip.

### Attack Strategy
1. Create a Wasm module with complex GC type hierarchy (rec groups, subtypes, shared types)
2. Compile and serialize the code (d8 --serialize-wasm or WebAssembly.Module API)
3. Modify the module slightly (reorder types, change shared/exact annotations)
4. Deserialize the old code with the modified module
5. If the mapping is incorrect, calls use wrong signatures → type confusion on parameters/returns

## Techniques That Work
1. Use d8's wasm code caching: compile module, serialize to disk, deserialize
2. Create modules with many types to exercise the mapping function
3. Test with shared types to check shared/non-shared canonical ID separation
4. Use `--wasm-serialization` flag if available
5. Compare canonical IDs before/after serialization round-trip
6. Stress test with large rec groups and subtype hierarchies

## DO NOT Try These
- Testing without serialization — need the round-trip to trigger the mapping
- Testing simple modules with one or two types — need complex hierarchies for mapping ambiguity
- Testing WASM_CALL_TAG relocations — these are different from WASM_CANONICAL_SIG_ID

## Evolution Plan
- v1: Source audit of the serialization/deserialization type mapping. Read wasm-serialization.cc fully. Map all relocation types that involve type IDs. Check if GC types (not just sigs) appear as WASM_CANONICAL_SIG_ID. Check WriteHeader/ReadHeader for type metadata validation.
- v2: Build test: module with rec group containing shared struct types. Serialize the compiled code. Deserialize and run. Check if type IDs are correctly restored by comparing call behavior before/after round-trip.
- v3: If v2 finds discrepancy: try to trigger the DCHECK-guarded lookup failure in release. Create scenario where canonical ID doesn't exist in the module-local map. Check if deserialized code uses wrong signature for indirect calls.

## Task
1. Read wasm-serialization.cc:406-440 (WriteHeader) — what module metadata is serialized?
2. Read wasm-serialization.cc:600-630 (CanonicalSigIdToModuleLocalTypeId) — full function with map construction.
3. Read wasm-serialization.cc:1035-1055 (deserialization reloc handling) — how canonical IDs are restored.
4. Search for "WASM_CANONICAL_SIG_ID" across the codebase — find all places this reloc type is created.
5. Check if WebAssembly.Module serialization API exists in d8 (search d8.cc or wasm-js.cc for "serialize").
6. Build module with rec group: (type $A (shared (struct (field i32)))), (type $B (shared (struct (field i64) (field (ref $A))))).
7. Serialize and deserialize, then call functions using $A and $B signatures.
8. Run with: `--experimental-wasm-shared --no-wasm-lazy-compilation`
9. Start with Evolution Plan v1 (source audit of serialization type mapping).
