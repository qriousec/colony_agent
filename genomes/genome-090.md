# Genome 090 — shared-gc-marking-sweep

## Hypothesis
TC-002 confirmed that DCHECK(!HeapLayout::InWritableSharedSpace(key)) at marking-visitor-inl.h:704 fires when shared Wasm objects are used as WeakCollection ephemeron keys. The same pattern — GC code assuming objects are NOT in writable shared space — exists in marking-barrier.cc:110,131 (regular marking barrier) and sweeper.cc:573 (sweeper). These code paths are reached through DIFFERENT operations than WeakCollection: regular property writes trigger marking barriers, and garbage collection sweeping processes all objects. If shared Wasm structs/arrays with ref fields trigger these paths, DCHECK crashes occur in debug builds and undefined GC behavior (corrupted marking state, missed references) occurs in release builds.

## DNA
### TC-002 Pattern (CONFIRMED)
- **marking-visitor-inl.h:704**: `DCHECK(!HeapLayout::InWritableSharedSpace(key))`
- Triggered by: WeakSet.add(sharedWasmStruct) → GC cycle → marking visitor processes ephemeron
- Impact: DCHECK crash (debug), potential object mishandling (release)

### Untested Sibling Paths
1. **marking-barrier.cc:110**: `DCHECK(!HeapLayout::InWritableSharedSpace(host))`
   - Context: Regular (non-shared) marking barrier
   - Trigger: When GC marks objects and encounters a host object in shared space
   - How to reach: Write a ref to a shared Wasm struct field during GC marking

2. **marking-barrier.cc:131**: `DCHECK(!HeapLayout::InWritableSharedSpace(host))`
   - Context: Another marking barrier path
   - Trigger: Similar to above, different code path

3. **sweeper.cc:573**: `DCHECK(!HeapLayout::InWritableSharedSpace(host))`
   - Context: Sweeper processes dead objects
   - Trigger: When sweeper encounters a shared Wasm object during sweeping phase

4. **heap-write-barrier.cc:117**: `DCHECK(!chunk->InWritableSharedSpace())`
   - Context: Write barrier for pointer slots
   - Trigger: Writing a pointer to/from shared space without using shared write barrier

5. **heap-write-barrier.cc:251**: `DCHECK(!HeapLayout::InWritableSharedSpace(host))`
   - Context: GenerationalBarrier
   - Trigger: When GC generational tracking encounters shared objects

### Key Insight
V8 has SEPARATE write barrier and marking paths for shared and non-shared objects. The shared paths are in heap-write-barrier.cc:269 (`DCHECK(InWritableSharedSpace(host))`) and marking-barrier-inl.h:71-73. If a shared Wasm object is processed through the NON-SHARED path, it hits the DCHECK.

The question is: which operations on shared Wasm objects route through the non-shared GC path?

## Techniques That Work
1. Create shared Wasm structs with ref fields pointing to local (non-shared) objects
2. Write refs to shared struct fields during active GC (concurrent marking)
3. Trigger incremental marking with `--incremental-marking --expose-gc`
4. Trigger concurrent marking with `--concurrent-marking --expose-gc`
5. Force GC at specific points: `gc()` or `%CollectGarbage(0)` with `--allow-natives-syntax`
6. Create cross-space reference patterns: local → shared → local chains
7. Monitor for DCHECK crashes with debug build

## DO NOT Try These
- WeakMap/WeakSet with shared structs — TC-001/002 already confirmed
- JS API class_name() crash — TC-001 + shared-nonstring-boundary confirmed
- Shared string unsharing — TC-003/004/005 confirmed

## Evolution Plan
- v1: Source trace marking-barrier.cc:110 and sweeper.cc:573 to understand how to reach them. What operations trigger the regular (non-shared) marking barrier? Read the marking barrier code to understand the dispatch between shared and non-shared paths. Find the conditions under which a shared object ends up on the non-shared path.
- v2: Build test with cross-space references. Create shared Wasm struct with (ref any) field. Store a local JS object ref into the shared struct's field (via Wasm struct.set). Then trigger incremental/concurrent GC. The GC needs to process the shared struct's field → hits marking barrier → DCHECK?
- v3: Test sweeper path. Create many shared Wasm objects, let some become unreachable, trigger full GC with sweeping. Monitor if sweeper.cc:573 DCHECK fires when processing shared objects.

## Task
Start by reading:
1. `src/heap/marking-barrier.cc:100-140` — the two DCHECK paths. What functions are these in? When are they called?
2. `src/heap/marking-barrier-inl.h:70-80` — the conditional DCHECKs that distinguish shared from non-shared
3. `src/heap/heap-write-barrier.cc:100-130` — write barrier dispatch. How does V8 decide between shared and non-shared write barrier?
4. `src/heap/sweeper.cc:570-580` — sweeper DCHECK context
5. `src/wasm/turboshaft-graph-interface.cc` — search for `WriteBarrier` in struct.set. What WriteBarrierKind is used for shared struct field writes?

The type invariant to challenge: "Shared Wasm objects always route through shared GC paths (shared marking barrier, shared write barrier), never through non-shared paths." TC-002 proved this wrong for ephemeron marking. Find the next violation.

Use `--experimental-wasm-shared --expose-gc --incremental-marking` with `/Users/t/v8/out/fuzzbuild/d8`.
