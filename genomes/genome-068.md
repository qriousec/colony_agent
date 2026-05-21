# Genome 068 — shared-waitqueue-type

## Hypothesis
The shared struct waitqueue feature (reverted TWICE: aaecfb4eb0b, b95c94a3ccd; relanded TWICE: 3c23aca5b1a, 480966b4b4c) implements wait/notify on shared Wasm structs. The first reland fixed FutexEmulation::NotifyWake() mutex handling; the second fixed `is_shared_space_isolate()` checks and ManagedPtrDestructors for shared isolates. These repeated fixes indicate the shared-space interaction is fragile.

Multi-factor: [shared struct waitqueue] × [waitqueue key type identity] × [GC of shared structs during wait] = potential type confusion or UAF when the waitqueue's struct reference and the actual struct disagree on type or liveness.

## DNA

### Source Facts
```
# Waitqueue implementation (480966b4b4c)
# 27 new files modified, 837 insertions total
# Key new code locations:

# Decoder: new opcodes for waitqueue operations
src/wasm/function-body-decoder-impl.h — 60 new lines for waitqueue decoding
src/wasm/wasm-opcodes.h — 3 new opcodes

# Compiler: Turboshaft + Liftoff support
src/wasm/turboshaft-graph-interface.cc — 55 new lines
src/wasm/baseline/liftoff-compiler.cc — 71 new lines

# Runtime: C++ implementation
src/runtime/runtime-wasm.cc — 33 new lines (Runtime_WasmWaitQueueWait, Runtime_WasmWaitQueueNotify)
src/execution/futex-emulation.cc — 259 lines changed (FutexManagedObjectWaitList)
src/execution/futex-emulation.h — 94 new lines

# Object model
src/wasm/wasm-objects.cc — 15 new lines (WasmStruct::GetOrCreateWaitQueue)
src/wasm/wasm-objects.h — 6 new lines

# Constant expression init
src/wasm/constant-expression-interface.cc — 76 lines changed
```

### Reland History (indicates fragility)
1. **9be6b8bfa99** — Original: shared struct waitqueue
2. **aaecfb4eb0b** — Revert 1: broke something (not specified)
3. **3c23aca5b1a** — Reland 1: Fixed NotifyWake mutex (was using global instead of per-list)
4. **b95c94a3ccd** — Revert 2: broke something else
5. **480966b4b4c** — Reland 2: Fixed is_shared_space_isolate() + shared ManagedPtrDestructors

### Key Questions
1. How does `WasmStruct::GetOrCreateWaitQueue` create and store the waitqueue? Is the waitqueue stored in the struct's memory or in a side table?
2. When a shared struct is used as a waitqueue key, what happens if the struct is GC'd while a waiter is blocked?
3. Can two structs of different types share the same waitqueue (via type confusion or aliasing)?
4. The FutexManagedObjectWaitList — how does it handle the lifecycle of the ManagedPtr? The reland fixed ManagedPtrDestructors for shared isolates, suggesting the destructor was running in the wrong isolate.

### Files to Audit
- `src/execution/futex-emulation.cc` — Core waitqueue implementation
- `src/execution/futex-emulation.h` — FutexManagedObjectWaitList API
- `src/wasm/wasm-objects.cc` — GetOrCreateWaitQueue
- `src/runtime/runtime-wasm.cc` — Runtime_WasmWaitQueueWait, Runtime_WasmWaitQueueNotify
- `src/wasm/function-body-decoder-impl.h` — Waitqueue opcode validation
- `src/objects/managed.cc` — ManagedPtrDestructors shared isolate fix

## Techniques That Work
- Multi-threaded testing with shared structs
- GC stress during wait/notify
- Type hierarchy: base waitqueue struct with child types, use base for wait, child for notify

## DO NOT Try These
- Single-threaded simple wait/notify (too basic, likely well-tested)
- Non-shared struct wait (should be rejected at validation)

## Evolution Plan
- v1: Source audit GetOrCreateWaitQueue + FutexManagedObjectWaitList lifecycle. Map all entry points. Check if waitqueue references prevent GC of the struct.
- v2: If v1 finds GC lifecycle gap, construct test with shared struct GC during wait. If v1 finds type handling gap, construct test with different struct types sharing a waitqueue.

## Task
Start by reading the waitqueue implementation:

```bash
# Read the new runtime functions
grep -n "WaitQueueWait\|WaitQueueNotify\|GetOrCreateWaitQueue" /Users/t/v8/src/runtime/runtime-wasm.cc /Users/t/v8/src/wasm/wasm-objects.cc /Users/t/v8/src/wasm/wasm-objects.h

# Read the FutexManagedObjectWaitList
grep -n "FutexManagedObject\|ManagedObjectWaitList" /Users/t/v8/src/execution/futex-emulation.h /Users/t/v8/src/execution/futex-emulation.cc

# Check the decoder validation
grep -n "waitqueue\|WaitQueue" /Users/t/v8/src/wasm/function-body-decoder-impl.h
```

Then trace the lifecycle: what happens to the waitqueue reference when the struct is GC'd?

Post with tags `shared-struct,waitqueue,futex,lifecycle,W31,coverage`.
