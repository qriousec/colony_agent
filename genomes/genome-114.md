# Genome 114 — waitqueue-shared-gc

## Hypothesis
Shared struct waitqueue support (Reland^2 commit 480966b4b4c) was reverted TWICE before landing. The first revert fixed a mutex bug (NotifyWake taking the wrong FutexWaitList mutex). The second revert fixed shared isolate memory accounting (ReleaseSharedPtrs checking `is_shared_space_isolate()`). When a shared WasmGC struct is used as a synchronization target via Atomics.wait on its i32/i64 fields, the FutexWaitListNode holds a reference to the struct's backing memory. If the struct is the sole reference keeping a field address valid, GC could relocate the struct while the wait node still references the old address. The triple-land pattern strongly suggests edge cases remain in: (1) wait node lifetime vs struct GC, (2) cross-isolate destructor ordering for ManagedPtrDestructors, (3) interrupt handling during wait on shared struct field.

## DNA

### Source Facts (file:line)
- **Reland^2 commit**: 480966b4b4c — "[wasm][shared] Implement waitqueue wait/notify for structs"
- **First revert**: aaecfb4eb0b — wrong mutex in NotifyWake
- **Second revert**: b95c94a3ccd — shared isolate memory accounting
- **Reland fix 1**: 3c23aca5b1a — FutexEmulation::NotifyWake takes mutex of FutexManagedObjectWaitList
- **Reland fix 2**: 480966b4b4c — check `is_shared_space_isolate()` in `Isolate::ReleaseSharedPtrs()`; use shared isolate for ExternalMemoryAccounter::Decrease
- **Bug references**: 475455008, 42204563
- **FutexEmulation**: src/execution/futex-emulation.cc — core wait/notify implementation
- **FutexWaitListNode**: src/execution/futex-emulation.h — wait list node structure
- **Shared struct layout**: src/wasm/wasm-objects.h (WasmStruct) — field access via offsets
- **Shared space GC**: src/heap/marking-visitor-inl.h — shared space marking
- **ExternalMemoryAccounter**: src/heap/heap.h — tracks external memory

### Related Bugs
- TC-001 (shared struct crash in WeakMap/WeakSet via class_name UNREACHABLE)
- TC-002 (shared struct DCHECK crash in GC marking visitor via WeakSet)
- Both show that shared WasmGC structs interact poorly with JS APIs

### Guard Inventory
1. FutexWaitList mutex protects wait list during add/remove
2. ManagedPtrDestructor tracked per-isolate
3. `is_shared_space_isolate()` check added in fix
4. Shared struct in shared space → not moved by minor GC

### Gap Analysis
1. **Wait on struct field during major GC**: Shared objects in shared space are NOT relocated by minor GC. But during major GC / shared space GC, are they relocated? If so, does the FutexWaitListNode update its address?
2. **Destructor ordering**: If main isolate dies while shared isolate has pending destructors for wait nodes, what happens?
3. **Interrupt during wait**: If an interrupt fires while waiting on a shared struct field, and the interrupt handler triggers GC, does the wait node survive?
4. **Struct with only wait-node reference**: If the JS/Wasm code drops all references to the shared struct but a FutexWaitListNode still references it, is the struct kept alive? Or does GC collect it, leaving a dangling wait node?
5. **Notify from wrong isolate**: If isolate A waits on shared struct field, and isolate B notifies, does the notification correctly reach isolate A's wait node?
6. **Multiple waiters on same struct field**: Contention between multiple isolates waiting on the same shared struct field — does the wait list handle concurrent add/remove correctly?

## Techniques That Work
1. Create shared struct with i32 field, wait on it from main thread, notify from worker
2. Force GC during wait (via interrupt or concurrent GC trigger)
3. Drop all references to shared struct except the wait node
4. Test with SharedArrayBuffer and shared struct interleaved
5. Multiple workers waiting on same struct field, rapid notify/wait cycling
6. Test destructor ordering: create shared struct, start wait, terminate worker isolate

## DO NOT Try These
- Testing without `--experimental-wasm-shared` flag
- Non-shared struct waitqueue (not supported)
- Simple Atomics.wait on SAB (this is the standard, well-tested path)

## Evolution Plan
- v1: Source audit of FutexEmulation for shared struct support. Read futex-emulation.cc to find where shared struct wait/notify diverges from SAB wait/notify. Identify the exact data structures holding references to struct fields. Build test: shared struct wait + GC stress.
- v2: Test cross-isolate scenarios. Create Worker that waits on shared struct field, main thread notifies after GC. Test isolate shutdown during pending wait. Test rapid wait/notify cycling with GC.
- v3: Test destructor ordering edge case. If the fix at `Isolate::ReleaseSharedPtrs()` still has races, construct a scenario where the shared isolate and main isolate both try to release the same ManagedPtrDestructor. Test with `--stress-compaction` and `--gc-interval` flags.

## Task
1. Start by reading src/execution/futex-emulation.h to understand FutexWaitListNode and FutexManagedObjectWaitList structures.
2. Then read src/execution/futex-emulation.cc, searching for "WasmStruct", "shared", "managed" to find the shared struct wait/notify code paths.
3. Read the full diff of commit 480966b4b4c to understand what was added (git show 480966b4b4c).
4. Read the two revert diffs (aaecfb4eb0b, b95c94a3ccd) to understand what broke.
5. Search for `ReleaseSharedPtrs` in the codebase to find the fix location.
6. Build test module: shared struct with i32 field, Atomics.wait from Worker, Atomics.notify from main, GC during wait.
7. Run with `--experimental-wasm-shared --stress-compaction --gc-interval=100`.
8. Diff behavior: Worker with/without GC stress during wait.
9. Start with Evolution Plan v1 (source audit of FutexEmulation shared struct paths).
