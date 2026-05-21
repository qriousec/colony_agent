# Genome 001 — loop-phi-narrow

## Hypothesis
When a loop's phi node merges a specific subtype (from the forward edge) with a value whose type is `wasm::ValueType()` (unknown/default) from the back edge, the `ProcessPhi` function in `wasm-gc-typed-optimization-reducer.cc:388-427` returns early at line 402 without updating the type, preserving the narrow forward-edge type from the first loop header evaluation. This causes downstream `ref.cast` operations to be eliminated by `REDUCE_INPUT_GRAPH(WasmTypeCast)` at `wasm-gc-typed-optimization-reducer.h:251-321` because `IsHeapSubtypeOf` succeeds on the incorrectly narrow type, allowing a value of type A (e.g., a parent struct) to be treated as type B (a child struct with more fields).

## DNA

### Source Facts
- **Loop phi first evaluation** (`wasm-gc-typed-optimization-reducer.cc:393-399`): On first visit to a loop header, phi type is set to forward-edge type only. The back edge is not yet evaluated.
- **Loop phi revisit** (`wasm-gc-typed-optimization-reducer.cc:401-418`): Computes union of all inputs. BUT at line 402: `if (union_type == wasm::ValueType()) return;` — if ANY input has unknown type, the function returns WITHOUT updating the phi's type. The narrow type from first evaluation persists.
- **GetTypeForPhiInput** (`wasm-gc-typed-optimization-reducer.cc:375-386`): For loop phis (back edge, input_index==1), it calls `types_table_.Get(input)` which returns whatever is in the snapshot table. If the back-edge value was never given a type (e.g., it's a phi in an inner block that wasn't processed for type info), it returns `wasm::ValueType()` (the default).
- **Type check elimination** (`wasm-gc-typed-optimization-reducer.h:274-280`): If `IsHeapSubtypeOf(inferred_type, cast_target)` AND not a custom descriptor cast, the cast is removed entirely.
- **TODO at line 408-410**: "Ideally, we'd skip unreachable predecessors here completely, as we might loosen the known type due to an unreachable predecessor." — Confirms the analysis knows the type union logic is imperfect.
- **Uninhabited skip** (lines 406-411): `is_uninhabited()` (bottom) values are skipped in the union. But `wasm::ValueType()` (void/default) is NOT uninhabited — it's the default "no information" value and triggers the early return.

### Guards Identified
- `is_first_loop_header_evaluation_` flag gates first vs revisit behavior
- `wasm::ValueType() == union_type` check causes early return (the vulnerability path)
- `is_uninhabited()` check for skipping bottom types (does NOT catch default ValueType)
- The loop is revisited after back-edge block is processed (but the back-edge input's type depends on whether source analysis reached it)

### Known CVE Patterns
- CVE-2024-2887: Type check elimination via incorrect type inference (different mechanism but same reducer)
- The `TypeSnapshotTable` approach with block-level snapshots is susceptible to ordering issues in loops

## Techniques That Work
1. Create a type hierarchy: `$Parent <: struct` and `$Child <: $Parent` with extra fields in `$Child`
2. Create a loop where:
   - Forward edge: value is known to be `$Child` (via allocation or cast)
   - Back edge: value comes from a code path where type analysis yields `ValueType()` (unknown)
3. After the loop, perform `ref.cast $Child` which should be eliminated if the phi type stays narrow
4. Actually pass a `$Parent` value on the back edge path
5. Access `$Child`-specific fields → type confusion (read beyond `$Parent`'s allocated size)

### Triggering `ValueType()` on back edge
- A value that arrives through multiple blocks where type analysis doesn't propagate (e.g., exception handler paths, blocks where the type analysis gives up)
- A value that is a phi of incompatible types that the analysis marks as unknown
- Check if `GetTypeForPhiInput` can return `ValueType()` for specific graph shapes

## DO NOT Try These
- Simple loops where both paths have known types (union will be computed correctly)
- Loops with `ref.cast` on the back edge (this will refine the type, not leave it unknown)
- Testing with only numeric types (this reducer only handles reference types)

## Evolution Plan
- v1: Map how `ValueType()` propagates through the snapshot table. Find a Wasm bytecode pattern that causes a loop back-edge phi input to have `ValueType()` type. Read `GetTypeForPhiInput` and `CreateMergeSnapshot` carefully.
- v2: If v1 identifies a pattern, construct a module where the loop phi retains the narrow forward-edge type. Verify with `--trace-wasm-typer` that the phi type is incorrectly narrow.
- v3: If v2 confirms narrow phi, add a downstream `ref.cast` that gets eliminated. Then pass a wider type on the back edge to trigger type confusion. Test with `--no-wasm-lazy-compilation` to force TurboShaft compilation.

## Task
Start by reading these files in order:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` lines 375-427 (phi processing)
2. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` lines 530-582 (CreateMergeSnapshot)
3. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` lines 584-630 (RefineTypeKnowledge)
4. `src/compiler/turboshaft/snapshot-table-opindex.h` (how `Get` returns default values)

The type invariant to challenge: "After loop revisit, the phi node's type is the union of all reachable input types."

The specific attack: Find a graph shape where `GetTypeForPhiInput` returns `ValueType()` for the back-edge input, causing `ProcessPhi` to return early (line 402) and preserving the narrow first-evaluation type.

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc --no-wasm-lazy-compilation --trace-wasm-typer`
