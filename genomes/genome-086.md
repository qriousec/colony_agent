# Genome 086 — shared-write-barrier-gc

## Hypothesis
Commit 53cc35a6e9a refactored write barrier handling — the WriteBarrierKind for struct.set and array.set is now computed upstream (in the graph builder at turboshaft-graph-interface.cc) and passed as a parameter, rather than computed locally in wasm-lowering-reducer.h. If the upstream graph builder incorrectly computes kNoWriteBarrier for stores into shared Wasm structs/arrays (objects in writable shared space), the GC will miss cross-space references, leading to dangling pointers or incorrect collection. Additionally, 20+ DCHECK(!HeapLayout::InWritableSharedSpace(...)) assertions in src/heap/ code indicate GC paths that don't expect objects in writable shared space — shared Wasm objects can trigger these, causing DCHECK crashes in debug builds and undefined GC behavior in release builds.

## DNA
### Write Barrier Refactor (53cc35a6e9a)
- **Before**: wasm-lowering-reducer.h computed write barrier locally:
  ```cpp
  type->field(field_index).is_ref() ? kFullWriteBarrier : kNoWriteBarrier
  ```
- **After**: write_barrier is a PARAMETER passed from upstream:
  ```cpp
  DCHECK_IMPLIES(write_barrier == kFullWriteBarrier, type->field(field_index).is_ref());
  __ Store(object, value, store_kind, repr, write_barrier, ...);
  ```
- **Key change**: Write barrier decision moved to turboshaft-graph-interface.cc
- Question: Does the graph builder correctly compute write barriers for SHARED struct/array operations?

### GC Shared Space Assumptions (20+ DCHECKs)
- **marking-barrier.cc:110**: `DCHECK(!HeapLayout::InWritableSharedSpace(host))` — regular marking barrier
- **marking-barrier.cc:131**: Same — another marking path
- **heap-write-barrier.cc:117**: `DCHECK(!chunk->InWritableSharedSpace())` — write barrier
- **heap-write-barrier.cc:251**: `DCHECK(!HeapLayout::InWritableSharedSpace(host))` — another write barrier path
- **sweeper.cc:573**: `DCHECK(!HeapLayout::InWritableSharedSpace(host))` — sweeper
- **marking-visitor-inl.h:704**: `DCHECK(!HeapLayout::InWritableSharedSpace(key))` — TC-002 (CONFIRMED BUG)
- **marking-barrier-inl.h:71**: `DCHECK_IMPLIES(HeapLayout::InWritableSharedSpace(host), ...)` — conditional check
- **marking-barrier-inl.h:73**: `DCHECK_IMPLIES(HeapLayout::InWritableSharedSpace(value), ...)` — conditional check

### TC-002 Pattern
TC-002 hit marking-visitor-inl.h:704 when a shared WasmStruct was added to a WeakSet. The DCHECK fired because WasmStruct IS a JSReceiver (passes WeakCollection::Set check) but IS in writable shared space.

### Write Barrier Types
- **kNoWriteBarrier**: Skip write barrier (Smi, RO-space values, young-to-young)
- **kPointerWriteBarrier**: Record slot (old-to-new pointer)
- **kFullWriteBarrier**: Full write barrier with mark checking
- **kMapWriteBarrier**: Map-specific write barrier (used for shared objects: wasm-lowering-reducer.h:449)
- **kEphemeronKeyWriteBarrier**: Ephemeron key write barrier

For shared objects, the WRONG write barrier type means the GC won't correctly track cross-space references.

## Techniques That Work
1. Create shared Wasm module with struct types containing ref fields
2. Use struct.set to write references (local-space objects → shared struct field, or vice versa)
3. Trigger GC after writes: `--expose-gc` and `gc()` calls
4. Force Turboshaft compilation for struct.set (warmup loops or `--wasm-tier-mask-for-testing=2`)
5. Monitor for DCHECK failures in debug build
6. Compare Liftoff (baseline) vs Turboshaft (optimized) — different write barrier computation paths
7. Test cross-space reference patterns: shared struct field → local object, local object field → shared struct

## DO NOT Try These
- WeakMap/WeakSet with shared structs — TC-001/002 already confirmed
- String unsharing at boundary — TC-003/004/005 already confirmed
- Shared struct at class_name() — shared-nonstring-boundary already confirmed on 10+ APIs

## Evolution Plan
- v1: Source trace write barrier computation for shared struct.set in turboshaft-graph-interface.cc. Find where WriteBarrierKind is computed and passed to StructSet operation. Does it account for shared objects? Read the graph builder code for struct.set and array.set. Also scan heap-write-barrier.cc to understand which write barrier path shared objects take.
- v2: Build test: shared struct with ref field. Write a local-space object ref into the shared struct's field (struct.set). Trigger GC. Check if the local object is incorrectly collected (dangling ref). Compare Liftoff vs Turboshaft.
- v3: Systematically trigger OTHER GC DCHECK paths: marking-barrier.cc:110, heap-write-barrier.cc:117/251, sweeper.cc:573. Each requires a different GC operation to hit the specific code path. Map which operations trigger which DCHECKs.

## Task
Start by reading:
1. `src/wasm/turboshaft-graph-interface.cc` — search for `StructSet` and `WriteBarrier`. Find how the write_barrier parameter is computed for struct.set operations. Look for shared type handling.
2. `src/compiler/turboshaft/wasm-lowering-reducer.h:270-310` — StructSet implementation receiving write_barrier param
3. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h:490-500` — StructSet optimization that also passes write_barrier
4. `src/heap/heap-write-barrier.cc` — write barrier dispatch. Which paths handle shared objects? Which have DCHECK(!InWritableSharedSpace)?
5. `src/heap/marking-barrier.cc` — marking barrier paths and their shared space assumptions

The type invariant to challenge: "The GC correctly tracks all references involving shared Wasm objects, including cross-space references from shared struct fields to local-space objects." TC-002 proved this false for ephemeron keys. Find the next violation in write barriers or marking paths.

Use `--experimental-wasm-shared --expose-gc` with `/Users/t/v8/out/fuzzbuild/d8`.
