# Genome 006 — load-elim-desc

## Hypothesis
WasmLoadElimination (`wasm-load-elimination-reducer.h`) recently added `ref.get_desc` support (commit `3bb690ad095`). When load elimination caches a `ref.get_desc` result and the same SSA value flows through a control flow merge where one path narrows the type (via `ref.test`/`ref.cast`) but the other doesn't, the cached descriptor load may be reused in the non-narrowed path. This produces a wrong descriptor value if the value in the non-narrowed path has a different descriptor than in the narrowed path. The `struct.get` / `ref.get_desc` load elimination does not account for custom descriptor identity — it matches by type index + field index, but two objects of the same nominal type can have different descriptors.

## DNA

### Source Facts
- **ref.get_desc support** (`3bb690ad095`): "Support ref.get_desc in WasmLoadElimination" — recently added.
- **WasmLoadElimination** (`wasm-load-elimination-reducer.h`): 1108 lines. Caches loads from struct fields, array elements, and (now) descriptor loads.
- **Load elimination key**: Typically keyed on (object, field_index, type_index). For ref.get_desc, the key should be (object) since descriptors are per-instance.
- **Control flow merges**: At phi points, cached loads from different paths are merged. If the merge doesn't invalidate descriptor caches when the value could have different descriptors, the wrong descriptor is returned.
- **Custom descriptor model**: Two instances of the same nominal type can have different descriptors. The descriptor is stored in the object's map (or derived from it). `ref.get_desc` extracts it.
- **WasmGC typed optimization interaction**: The type analyzer refines types at branches on `ref.test`. In the true branch, the type is narrowed to the checked type. Load elimination may cache a descriptor load in this narrowed branch and incorrectly propagate it to the merge point.

### Key Files
- `src/compiler/turboshaft/wasm-load-elimination-reducer.h` — main load elimination logic
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — type inference that feeds load elimination
- `src/wasm/turboshaft-graph-interface.cc` — how ref.get_desc is lowered to IR

### Attack Vectors
1. **Same SSA value, different descriptors**: Create a value of type `$DescType`. Load its descriptor (ref.get_desc). Branch on some condition. In one branch, cast to a more specific descriptor variant. After merge, load descriptor again. If load elimination reuses the first load, the descriptor may be wrong.
2. **Struct field caching with descriptor types**: struct.get on a descriptor-typed field. If the field holds objects with different descriptors but same nominal type, load elimination may cache the field load and return a stale value after mutation.
3. **GC safepoint invalidation**: If a GC can move objects or update maps, does load elimination properly invalidate descriptor caches across safepoints?

### Guards Expected
- Load elimination should invalidate caches at calls, GC points, and stores
- ref.get_desc cache should be keyed on the specific object, not just the type
- Merge snapshots should compute a conservative union of cached loads

## Techniques That Work
1. Read the load elimination reducer to understand:
   - How ref.get_desc loads are keyed and cached
   - How caches are merged at control flow joins
   - What operations invalidate the cache (calls, stores, GC)
2. Construct a test module:
   - Allocate two structs of same type but different descriptors (struct.new_desc with different descriptor values)
   - Store one in a local, load its descriptor
   - Branch: in true path, load descriptor of the other struct
   - After merge, load descriptor of first struct again
   - Check if load elimination returns stale value
3. Three-tier differential: Liftoff (no load elimination), Turboshaft without optimization, Turboshaft with optimization

## DO NOT Try These
- Load elimination for non-descriptor struct fields (well-tested)
- Testing without custom descriptors enabled
- Testing without optimization (load elimination only runs in optimized tier)

## Evolution Plan
- v1: Read `wasm-load-elimination-reducer.h` in full. Identify how ref.get_desc loads are represented, keyed, cached, and merged. Find the commit `3bb690ad095` diff to see exactly what was added.
- v2: If v1 finds that ref.get_desc caching doesn't account for descriptor identity (i.e., it caches based on object identity but doesn't invalidate when the object's descriptor could differ through aliasing), construct a test module that triggers stale descriptor reads after control flow merge.
- v3: If v2 finds a differential, minimize and identify the exact cache key/merge logic that's wrong. If v2 is clean, pivot to testing struct.get on fields that contain descriptor-typed references — does load elimination correctly handle field loads where the stored value's descriptor may have changed?

## Task
Start by reading these files in order:
1. Run `git -C /Users/t/v8 show 3bb690ad095` to see the ref.get_desc load elimination diff
2. `src/compiler/turboshaft/wasm-load-elimination-reducer.h` — search for "desc" to find all descriptor-related logic
3. Search for how load caches are merged at control flow joins (look for "Merge" or "Snapshot" or "phi")
4. `src/wasm/turboshaft-graph-interface.cc` — search for "get_desc" to see how the instruction is lowered

The type invariant to challenge: "WasmLoadElimination correctly handles ref.get_desc loads across control flow merges where the same object could have different descriptors in different paths."

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc --experimental-wasm-custom-descriptors --no-wasm-lazy-compilation`
