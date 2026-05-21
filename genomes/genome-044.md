# Genome 044 — wasm-load-elim-gc

## Hypothesis
WasmLoadElimination (W15) in Turboshaft eliminates redundant loads of struct/array fields when the optimizer proves no intervening store modifies the field. But GC safepoints can move objects in memory, invalidating cached load results. If the load eliminator doesn't properly invalidate cached values across GC safepoints, eliminated loads return stale pointers to pre-GC locations.

The WasmLoadElimination reducer is at `src/compiler/turboshaft/wasm-load-elimination-reducer.h`. It must handle:
1. **GC safepoints**: Any operation that can trigger GC (allocation, call, trap) must invalidate cached loads for objects that might move
2. **Write barriers**: Stores to struct fields must invalidate cached loads for that field
3. **Aliasing**: Two struct references might alias the same object — stores through one must invalidate loads through the other
4. **Immutable fields**: Fields declared `mut` must be invalidated on stores; immutable fields can be cached across stores but NOT across GC safepoints if the struct itself might move

Multi-factor: Wasm struct/array field load + GC safepoint (allocation or call) + load elimination optimization + Turboshaft pipeline.

## DNA

### Key Source File
```
src/compiler/turboshaft/wasm-load-elimination-reducer.h — Main reducer
```

### Grep Commands
```bash
# Load elimination reducer
wc -l src/compiler/turboshaft/wasm-load-elimination-reducer.h
head -50 src/compiler/turboshaft/wasm-load-elimination-reducer.h

# How GC safepoints interact with load elimination
grep -rn "safepoint\|Safepoint\|GC\|gc_point\|invalidate\|Invalidate\|kill\|Kill" src/compiler/turboshaft/wasm-load-elimination-reducer.h | head -20

# What operations trigger invalidation
grep -rn "REDUCE.*StructGet\|REDUCE.*StructSet\|REDUCE.*ArrayGet\|REDUCE.*ArraySet" src/compiler/turboshaft/wasm-load-elimination-reducer.h | head -10

# Call handling (calls can trigger GC)
grep -rn "REDUCE.*Call\|REDUCE.*Allocate" src/compiler/turboshaft/wasm-load-elimination-reducer.h | head -10

# How load results are cached
grep -rn "cache\|Cache\|snapshot\|Snapshot\|table\|Table\|state\|State" src/compiler/turboshaft/wasm-load-elimination-reducer.h | head -20

# WasmGC struct.get/set lowering
grep -rn "StructGet\|StructSet\|ArrayGet\|ArraySet" src/compiler/turboshaft/wasm-lowering-reducer.h | head -20
```

### Attack Pattern
1. Create a struct with a ref field
2. Store value A into field
3. Read field → result_1 (optimizer caches this)
4. Trigger GC (via allocation or explicit gc() call)
5. Read field again → result_2 (should re-read but might use cached result_1)
6. If load elimination is too aggressive, result_2 is a stale pointer

The key question: does the load eliminator invalidate its cache at GC safepoints?

### Precedent
- TurboFan had similar load elimination bugs in the past
- The Wasm GC spec says struct.get of a mutable field returns the current value, which can change if the struct is accessed from another thread (shared) or if the implementation moves objects (GC)

## Techniques That Work
1. Read the reducer source to understand the invalidation model
2. Build module: struct with mutable ref field, load-GC-load pattern
3. Run with `--turboshaft-wasm --no-liftoff` to force Turboshaft
4. Compare with `--liftoff --no-wasm-tier-up` (Liftoff doesn't do load elimination)
5. Use `%WasmTierUpFunction()` to force specific function to Turboshaft
6. Add GC pressure between loads: `(struct.new $type)` allocates and can trigger GC
7. Test with immutable fields vs mutable fields — different caching rules

## DO NOT Try These
- Simple struct.get without intervening operations — no optimization opportunity
- Loads across function boundaries — load elimination is typically intra-function
- Non-GC types (i32, f64) — these don't move with GC

## Evolution Plan
- v1: **Source audit of invalidation logic**. Read wasm-load-elimination-reducer.h. Map: what operations invalidate the load cache? Are GC safepoints (allocations, calls) among them? How are struct field loads cached (by struct type + field index, or by actual struct reference)?
- v2: **Build test module**. Function that reads struct field, allocates (GC trigger), reads same field again. Compare results between Liftoff and Turboshaft. If Turboshaft returns the same value but Liftoff returns different (because GC moved the struct), that's a load elimination bug.
- v3: **Aliasing confusion**. Two references to the same struct (via parameter + global). Store through global, read through parameter. Does load elimination know they alias? Test with identical struct types vs subtype relationships.

## Task
Start with v1. Read the reducer source:
1. `src/compiler/turboshaft/wasm-load-elimination-reducer.h` — Full file. Understand the caching model.
2. Look for how operations are classified: which operations "kill" cached loads? Specifically check: struct.new (allocation = GC), calls (can trigger GC), struct.set (store).
3. Check if there's a "may_alias" analysis — how does the reducer know if two struct references could point to the same object?
4. Look for the immutable field optimization — are immutable fields cached more aggressively?

Post with tags `load-elimination,gc,struct,turboshaft,w15,source-audit`.
