# Genome 084 — gc-optimizer-shared-elim

## Hypothesis
When the wasm-gc-typed-optimization-reducer encounters shared types, it skips type assertions entirely (line 181-184: "TODO: Extend this for shared types"). If the optimizer's type narrowing (via TypeSnapshotTable) and cast elimination (via IsSubtypeOf checks at graph nodes) don't correctly account for the shared bit when computing subtype relationships, it could incorrectly eliminate casts between shared and non-shared variants of the same structural type, enabling type confusion where a non-shared object is accessed as shared (or vice versa).

## DNA
### Critical Gap: Shared Type Assertion Skip
- **wasm-gc-typed-optimization-reducer.h:181-184**:
  ```cpp
  if (type.is_shared()) {
    // TODO(mliedtke): Extend this for shared types.
    return;
  }
  ```
  This means `--wasm-assert-types` flag does NOT validate shared type assertions. If the optimizer infers a wrong type for shared objects, assertions won't catch it.

### Type Narrowing Mechanism
The reducer uses a TypeSnapshotTable to track inferred types at each program point:
- After `ref.cast (ref $T)`, the optimizer narrows the value's type to `(ref $T)`
- After `ref.test (ref $T)` in a true branch, same narrowing
- At loop headers and merge points, types are WIDENED to the union

### Key Question: Does IsSubtypeOf Handle Shared Correctly?
- **wasm-subtyping.cc**: IsSubtypeOf checks include shared bit
- **value-type.h:242**: IsSharedField is bit 4 in the bit field
- If `(ref $T)` and `(ref shared $T)` are correctly NOT subtypes of each other, the optimizer should not eliminate casts between them
- BUT: does the TypeSnapshotTable correctly PRESERVE the shared bit through narrowing/widening?

### Type Narrowing at Merge Points
At phi nodes (merge points), the optimizer computes the union (LUB) of incoming types:
- If branch 1 provides `(ref $T)` and branch 2 provides `(ref shared $T)`, what is the merged type?
- Correct: `(ref any)` or similar supertype that encompasses both
- Incorrect: Either `(ref $T)` or `(ref shared $T)` — would enable cast elimination that shouldn't happen

### Recent Hardening Commits
- **c047ab74b85**: "Verify consistent hierarchy in turboshaft wasm type casts / checks" — suggests previous inconsistencies
- **08d948d0d0a**: "Add type assertions to type optimizer" — runtime validation of optimizer's type inference
- **535e378dbef**: "Introduce unconditional Wasm trap operation" — trap handling changes

### AnyConvertExtern with Shared Types
- **wasm-gc-typed-optimization-reducer.h:518**: `AnyConvertExtern(object, is_shared)` — handles shared externref conversion
- Does the optimizer correctly narrow the result type to include/exclude the shared bit?

## Techniques That Work
1. Build module with BOTH shared and non-shared variants of same struct type
2. Create code paths where shared and non-shared refs merge at phi nodes
3. Use ref.cast/ref.test to narrow types, then check if optimizer eliminates casts incorrectly
4. Force Turboshaft compilation with warmup or `--wasm-tier-mask-for-testing=2`
5. Compare behavior with `--no-wasm-opt` (disables optimization) vs normal
6. Use `--wasm-assert-types` to check if assertions catch optimizer errors (but note: shared types skip assertions!)
7. Create tier differential tests: Liftoff (no optimization) vs Turboshaft (optimized)

## DO NOT Try These
- Custom descriptor exactness — confirmed DEAD by 3+ workers
- Struct/array field offset computation — confirmed DEFENDED
- Simple shared struct operations without optimization interaction
- ref.cast/ref.test on basic types without merge/narrowing complexity

## Evolution Plan
- v1: Source trace the type narrowing path for shared types. Read wasm-gc-typed-optimization-reducer.h to find how TypeSnapshotTable handles shared types. Trace: (1) How does ReduceWasmTypeCast narrow types? (2) How does the snapshot table compute type unions at merge points? (3) Does wasm-subtyping.cc LUB computation include the shared bit? Look for TypeCheckAlwaysSucceeds/TypeCheckAlwaysFails logic.
- v2: Build test with shared/non-shared type merge. Create a Wasm function where:
  - Branch 1: creates non-shared struct ref
  - Branch 2: receives shared struct ref
  - Merge point: phi node with both types
  - After merge: ref.cast to specific type
  - Check if cast is incorrectly eliminated
- v3: Test loop-carried shared type narrowing. Create a loop where iteration 0 uses non-shared and iteration 1+ uses shared. Does the loop header type correctly widen to include both? Does a ref.cast inside the loop get incorrectly eliminated after the optimizer infers the type from iteration 0?

## Task
Start by reading:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — full file. Focus on:
   - Line 176-200: AssertType (shared skip at 181)
   - ReduceWasmTypeCast: how does it decide to eliminate casts?
   - ReduceWasmTypeCheck: how does it decide checks always succeed/fail?
   - Type widening at merge/phi nodes
2. `src/wasm/wasm-subtyping.cc` — LUB (least upper bound) computation:
   - How does CommonAncestor/LUB handle shared vs non-shared?
   - Is `(ref $T)` vs `(ref shared $T)` correctly handled?
3. `src/compiler/turboshaft/snapshot-table-opindex.h` — how types are stored/merged at snapshot points

The type invariant to challenge: "The optimizer correctly distinguishes shared and non-shared type variants during type narrowing and cast elimination." The TODO at line 181 suggests this may not be true.

Use `--experimental-wasm-shared` with `/Users/t/v8/out/fuzzbuild/d8`.
