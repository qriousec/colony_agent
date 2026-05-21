# Genome 016 — wle-struct-invalidation

## Hypothesis
WasmLoadElimination (WLE) has had two confirmed bugs where new operations with can_write effects were not handled (ArrayAtomicRMW in 2778b6ea720, WasmFXArgBuffer in bae619a077d), causing CHECK failures. The WLE also relies on an unverified assumption (line 664): "We rely on having no raw Store operations operating on Wasm objects at this point." If StructAtomicRMW (shared struct concurrent modifications) or another path introduces a mismatch between what the WLE tracks and what actually modifies struct fields, stale values of the wrong type could be served from the WLE cache. Specifically: if shared struct field A (ref type) is modified by thread 2 via struct.atomic.set, and the WLE on thread 1 serves the cached old value (which might now be a freed/reused reference), this is a type confusion.

Multi-factor: WLE struct field caching + shared struct concurrent modification + struct.atomic.set/get interaction.

## DNA

### Source Facts

**WLE ProcessStructGet (wasm-load-elimination-reducer.h)**:
- Caches struct field loads by `{base, field_index, type_index}` key
- Returns cached value on subsequent StructGet for same base+field

**WLE ProcessStructSet (wasm-load-elimination-reducer.h:~795)**:
- Calls `memory_.Invalidate(set)` to invalidate specific field in cache
- Only invalidates the EXACT field that was set

**WLE ProcessAtomicRMW (wasm-load-elimination-reducer.h:~807)**:
- Calls `memory_.Invalidate(op)` for StructAtomicRMW
- Added properly, handles struct atomic operations

**WLE ProcessCall (wasm-load-elimination-reducer.h:892-923)**:
- Invalidates all non-aliasing inputs
- Calls `memory_.InvalidateMaybeAliasing()` for all potentially aliasing memory
- Safe: any call could modify any struct field

**WLE kStore break case (line 664)**:
- Comment: "We rely on having no raw Store operations on Wasm objects at this point in the pipeline."
- TODO: "Is there any way to DCHECK that?"
- If violated, struct fields could be modified without WLE invalidation

**Confirmed bug (2778b6ea720)**:
- ArrayAtomicRMW was missing from WLE handler → CHECK crash
- Fix: added break case (correct since WLE doesn't track array elements)

**Confirmed bug (bae619a077d)**:
- WasmFXArgBuffer was missing → CHECK crash
- Fix: added break case (correct since arg buffer doesn't affect struct fields)

**Recent WLE additions**:
- `3bb690ad095`: ref.get_desc added to WLE as load-like operation
- `2778b6ea720`: ArrayAtomicRMW break case
- `bae619a077d`: WasmFXArgBuffer break case
- `53cc35a6e9a`: Improved write-barrier treatment

### Attack Angles
1. StructAtomicRMW interaction with WLE cache for ref-typed fields
2. Raw Store assumption violation — grep for Store operations on Wasm objects
3. ref.get_desc caching across descriptor mutations
4. Shared struct field access patterns that bypass WLE invalidation

## Techniques That Work
1. Build module with shared struct type, ref-typed mutable field
2. Thread 1: struct.get field → some_operation → struct.get same field (WLE should cache)
3. Thread 2: struct.atomic.set on same field with different ref type (subtype)
4. Check if thread 1's second struct.get returns the stale cached value
5. Use `--turboshaft-wasm-load-elimination` flag
6. Use `--no-liftoff` to force Turboshaft compilation
7. Use `--experimental-wasm-shared` for shared structs

## DO NOT Try These
- Array element load elimination (WLE doesn't track array elements)
- Memory cache reload after WasmFX (dead surface)
- Struct field offset computation (dead surface)

## Evolution Plan
- v1: **Source audit of WLE invalidation completeness**. Read WLE ProcessBlock fully. Map every operation with can_write effects and verify its handler is correct. Check whether StructAtomicRMW invalidation covers all cases (especially ref-typed fields). Check the kStore assumption by grepping for Store operations on Wasm objects in the pipeline stage where WLE runs.
- v2: **Shared struct concurrent modification test**. Build a module with two functions: reader (struct.get, call builtin, struct.get) and writer (struct.atomic.set). Run them concurrently. Check if the WLE caches across the call for the reader, and whether the writer's modification is visible.
- v3: **ref.get_desc cache test**. Build module with custom descriptor types. Test whether ref.get_desc is properly cached/invalidated across operations that could change type information.

## Task
Start with v1 (source audit). Read:
1. `src/compiler/turboshaft/wasm-load-elimination-reducer.h` — full file, focus on ProcessBlock switch and Invalidate methods
2. `src/compiler/turboshaft/operations.h` — search for operations with can_write effects involving Wasm objects
3. Check the pipeline stage ordering: which passes run before/after WLE? Can any pass introduce raw Stores to Wasm objects?

Your type invariant to challenge: "WLE correctly invalidates cached struct field values for ALL operations that could modify those fields." The two confirmed bugs (ArrayAtomicRMW, WasmFXArgBuffer) show this invariant was violated twice already.

Post to #colony-workers with tags `wle,invalidation,struct,atomic`.
