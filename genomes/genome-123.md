# Genome 123 — gc-optimizer-cast-elim

## Hypothesis
When a recursive type hierarchy creates deep subtype chains, the GC type optimization reducer (wasm-gc-typed-optimization-reducer.h) may incorrectly infer TypeCheckAlwaysSucceeds at loop merge points where the TypeSnapshotTable merges type knowledge from back-edges, causing ref.cast to be eliminated. Combined with the recent IsSameTypeHierarchy simplification (commit c047ab74b85, module_ parameter removed), cross-hierarchy type confusion may occur when the optimizer eliminates ref.cast for types that appear to be in the same hierarchy but aren't, allowing a parent struct to be treated as a child struct with extra fields.

## DNA
- **wasm-gc-typed-optimization-reducer.h:270-280**: The ReduceWasmTypeCast method checks `IsHeapSubtypeOf(type.heap_type(), cast_op.config.to.heap_type(), module_)`. If the inferred `type` is wrong (too narrow), this returns true and the cast is eliminated.
- **wasm-gc-typed-optimization-reducer.h:336-345**: Same pattern in ReduceWasmTypeCheck for ref.test.
- **wasm-gc-typed-optimization-reducer.cc:388-427**: ProcessPhi() — On FIRST loop header evaluation (line 393-399), only forward edge type is used. On revisit, union of all phi inputs computed. **ATTACK**: If backedge refines to less specific type, union weakens from `ref exact T` → `ref T`, losing exactness.
- **wasm-gc-typed-optimization-reducer.cc:564-571**: Unreachable predecessor check — `if (!reachable[i]) continue;` skips unreachable predecessors in type merge. If a predecessor becomes reachable after optimization, type union is incomplete.
- **wasm-gc-typed-optimization-reducer.h:307-308**: Intersection type truncation — `wasm::Intersection(type, type_check.config.to, module_).type`. If intersection computation is wrong (wasm-subtyping.cc:626-655), a check that should always-fail passes.
- **Commit c047ab74b85**: Removed `module_` parameter from `IsSameTypeHierarchy` check. Before: `IsSameTypeHierarchy(type, target, module_)`. After: `IsSameTypeHierarchy(type, target)`. Could weaken cross-hierarchy validation.
- **Commit 16dfe88b9d8**: WasmTypeCastOp was missing `CanDependOnChecks()`, allowing the revectorizer to reorder it past array.len. Fix added CanDependOnChecks. But WasmTypeCheckOp still uses only `AssumesConsistentHeap()`.
- **Commit 08d948d0d0a**: Added type assertions to type optimizer — suggests assertions were missing before.
- **wasm-gc-typed-optimization-reducer.h:181-183**: Type assertions SKIP shared types entirely: `if (type.is_shared()) { /* TODO */ return; }`. This means shared type mismatches are never caught by assertions.
- **CVE-2024-2887**: Type confusion via TypeCheckAlwaysSucceeds returning true when it shouldn't.
- **TypeSnapshotTable**: Tracks known types per SSA value. At loop merge points (phi nodes), types from multiple edges are merged. Key: first pass uses only forward edge (potentially too specific), revisit widens via union.

## Techniques That Work
1. Create deep type hierarchies: parent → child → grandchild with different field counts
2. Use loops where the loop variable alternates between parent and child instances
3. Force Turboshaft tier-up (call function many times)
4. Place ref.cast inside the loop, casting to child type
5. If the optimizer incorrectly infers the loop variable is always child type, the cast is eliminated
6. Then pass a parent instance through the loop — the eliminated cast allows field OOB access

## DO NOT Try These
- Shared types (require --experimental-wasm-shared, BANNED)
- Simple linear casts without loops — the optimizer won't have merge points to confuse
- Module-level type errors — the validator catches these

## Evolution Plan
- v1: Source analysis of ProcessPhi() at wasm-gc-typed-optimization-reducer.cc:388-427. Map: (a) first-pass forward-edge-only type assignment, (b) revisit union behavior, (c) exactness loss at phi merge. Build test: loop with forward edge carrying `ref exact $child` and back-edge carrying `ref $parent`. Check if union drops exactness, then cast to `ref exact $child` is eliminated allowing parent struct access.
- v2: Exploit the intersection computation. Create types where `Intersection(ref $A, ref $B)` should be uninhabited but the computation at wasm-subtyping.cc:626-655 incorrectly returns a valid type. This would make `TypeCheckAlwaysFails` return false when it should return true, keeping a cast that should be eliminated or vice versa. Also test: unreachable predecessor becoming reachable after unrolling — does the type union update correctly?
- v3: Target WasmTypeCheckOp effects asymmetry. WasmTypeCheckOp uses `AssumesConsistentHeap` (no CanDependOnChecks). With `--experimental-wasm-revectorize`, try to get ref.test reordered past a null check by the revectorizer. Also: test custom descriptor bypass — create a type with descriptor and non-exact flag to skip optimization at line 276, then exploit unoptimized cast path.

## Task
Start by reading these files in order:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — full file, focus on ReduceWasmTypeCast (line ~260), ReduceWasmTypeCheck (line ~330), and the TypeSnapshotTable/ProcessTypeCast/ProcessTypeCheck methods
2. `src/compiler/turboshaft/operations.h` — search for WasmTypeCastOp and WasmTypeCheckOp, compare their OpEffects
3. `src/wasm/wasm-subtyping.cc` — IsSameTypeHierarchy implementation, IsHeapSubtypeOf

Then build a test module with:
- A parent struct type with 1 field (i32)
- A child struct type extending parent with 2 fields (i32, i64)
- A loop that receives a ref to parent, casts to child, and reads the extra field
- Feed parent instances through the loop and check if the cast is properly enforced at Turboshaft tier

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc test.js`
No --experimental-wasm-shared flag.
