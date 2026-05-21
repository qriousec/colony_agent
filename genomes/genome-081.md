# Genome 081 — wasmfx-jspi-stack-reuse

## Hypothesis
JSPI (JavaScript Promise Integration) and WasmFX share the StackMemory infrastructure but use different protocols for frame types, stack object lifecycle, and effect handler search. Three confirmed bugs reveal the architectural tension:

1. **Bug 388533754**: WasmFX `suspend_wasmfx_stack` expected a `WasmStackObject` on the active stack, but JSPI stacks had none → crash on `from->stack_obj()`
2. **Bug 489139911**: Effect handler search loop (wasm-external-refs.cc:1052-1074) expected only `StackFrame::WASM_STACK_EXIT` frames but JSPI uses `StackFrame::WASM_JSPI` frames → incorrect suspend behavior when both features active
3. **Bug 488930834**: `StackMemory::stack_obj_` not visited by GC, becomes stale when stack retires and is picked up from pool for a new operation → UAF if stale reference revisited

**Combined attack:** When a JSPI suspension creates a stack, that stack enters the pool. WasmFX picks it up for a continuation. The recycled stack may have:
- Stale `stack_obj_` from JSPI lifetime (EPT pointing to deallocated WasmStackObject)
- Wrong frame type markers (WASM_JSPI instead of WASM_STACK_EXIT)
- Effect handler search hits JSPI frame and misinterprets it as WasmFX handler boundary

Additionally, the effect handler search loop at wasm-external-refs.cc:1085-1104 has an `UNIMPLEMENTED()` case for switch handler targets — this is a reachable code path if `cont.switch` is used (part of the WasmFX spec).

## DNA

### Source Facts — Shared Infrastructure
- **stacks.h / stacks.cc**: `StackMemory` class — shared by both JSPI and WasmFX
- **Stack pool**: Both features allocate from and return to the same pool
- **WasmStackObject**: External pointer wrapper around StackMemory, stored in `stack_obj_` field
- **stack_obj_ lifecycle**: Set on allocation (runtime-wasm.cc:1334 for JSPI, :2839 for WasmFX), cleared on retirement (isolate.cc:4312-4317)

### Source Facts — Protocol Differences
| Aspect | JSPI | WasmFX |
|--------|------|--------|
| Frame type | WASM_JSPI | WASM_STACK_EXIT |
| Stack object setup | AllocateSuspender (runtime-wasm.cc:1334) | WasmFX stack creation (runtime-wasm.cc:2839) |
| Suspension trigger | Promise-based (async import) | cont.suspend instruction |
| Effect handlers | None (single entry/exit) | Per-tag handler table (effect-handler.h) |
| Resume protocol | Promise resolution callback | cont.resume / cont.resume_throw |

### Source Facts — Effect Handler Search Loop
- **wasm-external-refs.cc:1052-1074**: Walks stack frames looking for `WASM_STACK_EXIT` to find effect handler
- **wasm-external-refs.cc:1085-1104**: Iterates handler table entries, matching tag indices
- **wasm-external-refs.cc:1101**: `UNIMPLEMENTED()` for `is_switch()` handler — switch handlers not yet implemented
- **effect-handler.h**: `EffectHandlerTagIndex` packed with `is_switch` bit + index — new file (copyright 2026)

### Source Facts — Stack Recycling
- **Stack pool** (stacks.cc): `RetireWasmStack` returns stack to pool, clears EPT
- **Reuse**: Next `AllocateSuspender` or WasmFX stack creation pulls from pool
- **Bug 488930834 fix**: Added clearing of `stack_obj_` on retirement — but does the fix cover ALL retirement paths?

### Key Questions
1. When JSPI stack is recycled for WasmFX, is the frame type marker properly updated?
2. Does the effect handler search handle encountering a stale JSPI frame type gracefully?
3. Can `cont.switch` reach the UNIMPLEMENTED() path? What module bytecode triggers it?
4. Is `stack_obj_` properly initialized when a pool-recycled stack is used for the OTHER feature (JSPI→WasmFX or WasmFX→JSPI)?
5. Does the stack pool track which feature last used a stack? Any feature-specific cleanup needed on reuse?

### Attack Vectors
1. JSPI suspend → return stack to pool → WasmFX cont.new picks up same stack → effect handler search encounters JSPI remnants
2. WasmFX suspend → retire stack → JSPI allocates same stack → resume JSPI → WasmFX-specific state confuses JSPI resume
3. Trigger cont.switch instruction → hit UNIMPLEMENTED() in effect handler search → potential undefined behavior in release
4. Interleave JSPI and WasmFX operations rapidly to stress the stack pool with mixed-protocol stacks
5. GC during stack reuse window — pool holds stack, GC visits, stack_obj_ state is inconsistent

## Techniques That Work
1. Create module with both JSPI (async imports) and WasmFX (cont.new/resume/suspend)
2. Use `--experimental-wasm-jspi --experimental-wasm-wasmfx` flags
3. Trigger JSPI suspension via async import, then create WasmFX continuation
4. Force GC between JSPI return and WasmFX allocation to stress pool management
5. Use `--stress-wasm-stack-switching` if available for rapid stack creation/destruction
6. Test cont.switch bytecode to reach UNIMPLEMENTED() handler search path
7. Use ASan/MSan builds for detecting UAF from stale stack_obj_ references

## DO NOT Try These
- cont.bind GC roots — DEAD SURFACE
- WasmFX exception lifecycle basic — tested CLEAN
- Stack pool management in isolation — tested DEFENDED
- Liftoff WasmFX — ALL WasmFX ops unsupported in Liftoff (DEAD SURFACE DS-14)

## Evolution Plan
- v1: Source audit — trace the full stack recycling path. Read stacks.cc to understand pool management. Read wasm-external-refs.cc:1033-1120 to map all suspend/resume paths and their stack_obj_ handling. Verify that bug 488930834 fix covers ALL retirement paths. Create test: JSPI suspend → WasmFX cont.new → does cont.new get a clean stack?
- v2: Test effect handler search with mixed frame types. Create a scenario where JSPI and WasmFX are both active. Suspend via WasmFX inside a function called through JSPI. Does the effect handler search correctly skip JSPI frames? Also attempt to reach UNIMPLEMENTED() via cont.switch bytecode.
- v3: Stress test — rapidly create and destroy JSPI suspensions and WasmFX continuations with GC pressure. Use --gc-interval=50 to force GC during pool operations. Check for stale EPT references, wrong frame types, or memory corruption.

## Task
Start by reading these files in order:
1. `src/wasm/stacks.h` — StackMemory class definition, stack_obj_ field, pool management
2. `src/wasm/stacks.cc` — Pool allocation, retirement, reuse logic
3. `src/wasm/wasm-external-refs.cc:1033-1120` — suspend_wasmfx_stack, effect handler search loop, UNIMPLEMENTED case
4. `src/wasm/effect-handler.h` — EffectHandlerTagIndex with is_switch bit
5. `src/runtime/runtime-wasm.cc:1320-1350` — JSPI AllocateSuspender (stack_obj_ setup)
6. `src/runtime/runtime-wasm.cc:2830-2850` — WasmFX stack creation (stack_obj_ setup)
7. `src/execution/isolate.cc:4310-4320` — RetireWasmStack (stack_obj_ clearing)

Name the type invariant to challenge: **Stack objects recycled through the pool must be fully re-initialized for the new feature's protocol. A stack previously used by JSPI must not carry JSPI-specific state (frame types, missing WasmStackObject, no effect handler table) when reused by WasmFX, and vice versa.** The pool treats all stacks as interchangeable, but JSPI and WasmFX have different invariants.

Start with v1 — map the complete stack recycling lifecycle and verify cross-feature reuse safety.
