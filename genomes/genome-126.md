# Genome 126 — load-elim-struct-gc

## Hypothesis
The WasmLoadElimination reducer caches struct/array field loads across operations. The recent fix 2778b6ea720 corrected load elimination for array atomic rmw operations, proving gaps existed in how load elimination tracks invalidation. If a struct field load is cached by the reducer, and then a GC-triggering operation (allocation, function call) occurs, the cached value may become stale if the GC moves objects or if an intervening store to the same field (via a different alias) is not properly tracked. For GC reference fields, this means the eliminated load could return a reference to a moved/collected object, or a reference of the wrong type if the field was overwritten.

## DNA
- **wasm-load-elimination-reducer.h**: The reducer maintains a map of known field values. When a StructGet is encountered, it checks if the value is already known. If so, it returns the cached value without emitting a load.
- **Commit 2778b6ea720**: Fixed load elimination for array atomic rmw — the reducer wasn't properly handling atomic read-modify-write operations on arrays. This means the invalidation logic had blind spots.
- **Commit 3bb690ad095**: Added ref.get_desc support to WasmLoadElimination — new operations being added to the reducer increase complexity and potential for bugs.
- **Commit 535e378dbef**: Introduced unconditional Wasm trap operation — changes to control flow affect load elimination's analysis of what operations can intervene.
- **GC interaction**: V8's garbage collector can move objects (compacting GC). The load elimination must invalidate cached values across any operation that could trigger GC. In Turboshaft, this is tracked via OpEffects (CanAllocate, CanCallAnything).
- **Aliasing**: Two different OpIndex values might reference the same struct object (through different paths). If the reducer caches a load from one alias, and a store occurs through the other alias, the cached value is stale.

## Techniques That Work
1. Create a struct with multiple fields including ref-typed fields
2. Store a ref to struct A in a field, then trigger GC, then load the field — check if the loaded ref is still valid
3. Use two references to the same struct (aliasing through function parameters and global variables) — store through one alias, load through the other
4. Combine struct.set and struct.get with intervening allocations that trigger GC
5. Use array operations mixed with struct operations to confuse the load elimination's tracking
6. Force Turboshaft tier-up to engage the load elimination reducer

## DO NOT Try These
- Shared types (--experimental-wasm-shared, BANNED)
- Liftoff-only testing — load elimination only runs in Turboshaft
- Simple sequential loads without aliasing or GC — the reducer handles these correctly

## Evolution Plan
- v1: Source analysis of wasm-load-elimination-reducer.h. Map: (a) how struct field loads are cached (key: object OpIndex + field index?), (b) how stores invalidate cached values (same object? all objects?), (c) how GC-triggering operations invalidate the cache (CanAllocate?), (d) how aliasing is handled (same object through different SSA values). Identify specific invalidation gaps.
- v2: Build aliasing tests. Two function parameters that are actually the same struct. Store through param A, load through param B. Does the reducer correctly see these as the same object? What about globals vs locals? What about struct.get on a value loaded from an array (indirect alias)?
- v3: If v2 is clean, test GC pressure scenarios. Rapidly allocate many structs to trigger compacting GC between struct.set and struct.get. Use concurrent marking (--stress-concurrent-allocation) to increase the window. Check if the load elimination's invalidation across allocations is complete.

## Task
Start by reading:
1. `src/compiler/turboshaft/wasm-load-elimination-reducer.h` — full file. Focus on: how the ScopedHashMap/LoadEliminationTable works, what operations invalidate cached loads, how struct.set/struct.get are tracked.
2. `git -C /Users/t/v8 show 2778b6ea720` — the array atomic rmw fix. What was the invalidation gap?
3. `src/compiler/turboshaft/operations.h` — search for StructGetOp, StructSetOp, ArrayGetOp, ArraySetOp. Check their OpEffects.

Then build a test:
- A struct type with 2 fields: {i32, ref $self}
- A function that takes two params of the struct type (same object passed twice)
- Store ref value through param 0, then allocate (trigger potential GC), then load from param 1's same field
- Compare stored vs loaded values
- Force Turboshaft tier-up

Run with: `/Users/t/v8/out/fuzzbuild/d8 test.js`
No --experimental-wasm-shared flag.
