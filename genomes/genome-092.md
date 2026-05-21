# Genome 092 — loop-type-widening

## Hypothesis
The wasm-gc-typed-optimization-reducer uses a TypeSnapshotTable (line 128) to track inferred types at each program point. At loop backedges, types from the loop body must be WIDENED back to the loop header. If the widening computation (LUB — least upper bound) doesn't account for all type combinations seen across iterations — especially when ref.cast/ref.test inside the loop narrows types that the optimizer propagates past the backedge — the loop header type becomes too narrow. Subsequent iterations may then have type casts eliminated based on the narrow header type, when the actual runtime type is wider.

## DNA
### TypeSnapshotTable Architecture
- **wasm-gc-typed-optimization-reducer.h:128**: `TypeSnapshotTable types_table_` — tracks known types at each program point
- **wasm-gc-typed-optimization-reducer.h:70-72**: TypeSnapshotTable definition and key/value types
- The table maps SSA values to their inferred (narrowed) types
- At loop headers, types from all predecessors (including the backedge) must be merged

### Loop Type Widening Mechanism
- When the optimizer processes a loop, it first processes the header with initial types
- It then processes the loop body, which may narrow types via ref.cast/ref.test/br_on_cast
- At the backedge, the narrowed types flow back to the loop header
- The header types must be WIDENED to accommodate both the initial types AND the backedge types
- This widening uses LUB (least upper bound) in the type hierarchy

### Potential Failure Modes
1. **Insufficient widening**: If LUB doesn't account for all type narrowings in the body, the header type is too narrow → casts eliminated incorrectly
2. **Fixed-point non-convergence**: If the optimizer doesn't iterate to a fixed point, transient narrow types persist
3. **Shared type interaction**: Since AssertType SKIPS shared types (line 181-184), type errors for shared types in loops won't be caught by assertions
4. **Type hierarchy depth**: Deep type hierarchies (struct A <: B <: C <: D) with casts at different levels may stress the LUB computation

### Key Code Locations
- **wasm-gc-typed-optimization-reducer.h:128**: TypeSnapshotTable instance
- **wasm-gc-typed-optimization-reducer.h:251**: REDUCE_INPUT_GRAPH(WasmTypeCast) — where casts are eliminated
- **wasm-gc-typed-optimization-reducer.h:323**: REDUCE_INPUT_GRAPH(WasmTypeCheck) — where checks are eliminated
- **wasm-gc-typed-optimization-reducer.h:434**: REDUCE_INPUT_GRAPH(WasmTypeAnnotation) — type annotations
- **turboshaft-graph-interface.cc**: Loop header phi creation and type merging

### Related Patterns
- **AssertType skip for shared** (line 181-184): If shared types are used in loops, assertions won't catch widening errors
- **br_on_cast narrowing** (genome-094): br_on_cast inside loops creates complex narrowing patterns that must be widened correctly

## Techniques That Work
1. Create deep type hierarchy: struct A <: B <: C <: D (4+ levels)
2. Create loop that casts from wide type (ref A) to narrow type (ref D) via ref.cast
3. Use the narrowed value inside the loop (struct.get specific fields)
4. On each iteration, pass DIFFERENT concrete types (sometimes B, sometimes C, sometimes D)
5. Compare Liftoff (no type optimization) vs Turboshaft (with TypeSnapshotTable)
6. Force Turboshaft with `--no-liftoff` or warmup iterations
7. Check for incorrect cast elimination by verifying ref.cast still traps when it should
8. Use `--trace-turbo` to inspect optimizer's type inference at loop headers

## DO NOT Try These
- Simple loops without type narrowing — no interesting type interaction
- Non-loop type cast elimination — covered by other workers
- Shared struct field access — dead in single-isolate mode
- Linear (non-loop) br_on_cast chains — covered by genome-094

## Evolution Plan
- v1: Source trace TypeSnapshotTable loop handling. Read wasm-gc-typed-optimization-reducer.h for how loop headers are processed. How does Seal/SealAndPatch work for loop phis? How are types merged at loop headers? Read turboshaft's loop analysis (loop-finder or similar) to understand how loop backedges are identified and processed. Find the LUB computation used for type widening.
- v2: Build test with 4-level type hierarchy and loop. Create `(type $A (sub (struct (field i32))))`, `(type $B (sub $A (struct (field i32) (field i32))))`, etc. Loop that receives ref $A, casts to $B, uses field, then receives a DIFFERENT type next iteration. Check if Turboshaft incorrectly eliminates the cast after first iteration.
- v3: Test with polymorphic loop — array of structs with mixed types, loop processes each element with ref.cast. The optimizer may narrow the type based on the first element and eliminate casts for subsequent elements. Test with `--no-liftoff` to force Turboshaft optimization.

## Task
Start by reading:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — FULL FILE. Focus on:
   - TypeSnapshotTable definition (line 70-72, 128)
   - How loop headers are handled (search for "loop", "backedge", "phi", "Seal")
   - REDUCE_INPUT_GRAPH(WasmTypeCast) at line 251 — cast elimination logic
   - How types are merged/widened at control flow joins
2. `src/compiler/turboshaft/loop-finder.h` or similar — how loops are identified in Turboshaft
3. `src/compiler/turboshaft/type-inference-reducer.h` — generic type inference, may have loop widening logic
4. `src/wasm/wasm-subtyping.h` — LUB (CommonAncestor) computation for Wasm GC types
5. `src/compiler/turboshaft/snapshot-table.h` — the underlying SnapshotTable mechanism. How does it handle loop backedges? Is there a widening step?

The type invariant to challenge: "At loop headers, the TypeSnapshotTable correctly widens types to the LUB of all incoming types, ensuring no cast is eliminated that could trap at runtime." If widening is incomplete, casts are eliminated incorrectly → type confusion.

Use `--no-liftoff --experimental-wasm-gc` with `/Users/t/v8/out/fuzzbuild/d8`. Add `--trace-turbo` for optimizer IR dumps.
