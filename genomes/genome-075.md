# Genome 075 — wasmfx-stack-lifecycle

## Hypothesis
WasmFX stack lifecycle management has two confirmed vulnerability patterns that may combine:

**Factor 1 — EPT stale reference:** Fix e97587a3751 centralized EPT clearing in RetireWasmStack (isolate.cc:4312-4317). Before the fix, EPT was NOT cleared on exception paths, causing UAF. The fix sets `stack_obj()->set_stack(this, nullptr)` + `set_stack_obj({})`. But `suspend_wasmfx_stack` (wasm-external-refs.cc:1044) copies `from->stack_obj()` to the continuation BEFORE suspension. If the suspended stack is later retired via exception (calls RetireWasmStack), the continuation still holds a WasmStackObject whose `stack` external pointer is now nullptr. Resuming that continuation would dereference nullptr or recycled memory.

**Factor 2 — Compressed/uncompressed confusion:** Fixes 5a64071bf9d (returns) and 2f8c6526740 (params) corrected MemoryRepresentation for ref types in WasmFX arg buffers (wrappers-inl.h:610-633). Before the fixes, refs were stored/loaded using `FromMachineType(type.machine_type())` which on pointer-compressed builds uses compressed representation, but off-heap buffers require uncompressed. Are there OTHER off-heap buffer locations (stack wrapper cache, cont.bind arg copying) that still use compressed MemoryRepresentation for refs?

**Combined:** The stack wrapper cache (68852a942b4) recycles WasmStackObjects. If a recycled stack inherits stale EPT state from its previous lifetime, AND its arg buffer has compressed/uncompressed mismatch from the different continuation type signatures, the recycled stack has both a dangling pointer AND type-confused reference arguments.

## DNA

### Source Facts — EPT Management
- **isolate.cc:4312-4317**: `RetireWasmStack` clears EPT: `stack_obj()->set_stack(this, nullptr)`, then `set_stack_obj({})`
- **wasm-external-refs.cc:1044**: `cont->set_stack_obj(from->stack_obj())` — copies stack_obj during suspend (BEFORE retirement)
- **wasm-external-refs.cc:1131**: `RetireWasmStack(from)` called in return_stack after SwitchStacks
- **wasm-external-refs.cc:1149**: `RetireWasmStack(stack)` called in retire_stack (standalone retirement)
- **runtime-wasm.cc:1334**: New stacks get `set_stack_obj(...)` during AllocateSuspender (JSPI path)
- **runtime-wasm.cc:2839**: New stacks get `set_stack_obj(...)` during WasmFX stack creation

### Source Facts — MemRep Confusion
- **wrappers-inl.h:610-613**: Fixed params: `type.is_ref() ? AnyUncompressedTagged() : FromMachineType(...)`
- **wrappers-inl.h:630-633**: Fixed returns: same pattern
- **wrappers-inl.h:591**: Uses `UncompressedTaggedPointer()` for instance data (always correct)
- **wrappers-inl.h:444**: Uses `AnyUncompressedTagged()` for another off-heap ref

### Source Facts — Stack Wrapper Cache
- Commit 68852a942b4: "Introduce stack wrapper cache" — recycles WasmStackObjects to avoid repeated allocation
- Stack wrapper cache manages WasmStackObject lifecycle — need to verify EPT clearing on recycle

### Key Guards to Map
1. Does RetireWasmStack clear EPT for ALL retirement paths (normal return, exception, GC finalization)?
2. Does the stack wrapper cache properly reinitialize EPT on stack reuse?
3. Are there other off-heap ref stores using FromMachineType instead of AnyUncompressedTagged?
4. Does cont.bind (wasm-external-refs.cc) properly handle arg buffer MemRep when binding ref-typed args?

## Techniques That Work
1. Create WasmFX continuations with ref-typed params/returns (externref, anyref, struct refs)
2. Suspend a continuation, then cause an exception on the suspended stack
3. Attempt to resume the suspended continuation after exception retirement
4. Run with `--experimental-wasm-wasmfx --experimental-wasm-shared` flags
5. Use GC stress (`--gc-interval=100`) to trigger cache recycling
6. Compare compressed vs uncompressed builds (may need ASan/MSan for detection)

## DO NOT Try These
- cont.bind GC roots — DEAD SURFACE confirmed by 2 workers
- WasmFX exception lifecycle basic paths — tested by wasmfx-except-lifecycle, CLEAN
- Stack pool management — tested by wasmfx-contbind-gcstress, DEFENDED

## Evolution Plan
- v1: Map the complete stack lifecycle — trace every path from stack creation → suspension → resumption → retirement. For each path, verify EPT is properly managed. Focus on the suspend→exception→retire→resume sequence. Create test with cont.new → cont.resume → suspend → throw exception in parent → attempt resume of abandoned continuation.
- v2: Audit ALL MemoryRepresentation usage in wrappers-inl.h and wasm-external-refs.cc for off-heap reference stores/loads. Check if cont.bind argument copying uses correct MemRep. Create test with ref-typed bound arguments on pointer-compressed build to check for truncation.
- v3: Test stack wrapper cache recycling — create many continuations to trigger cache saturation, then verify recycled stacks have clean EPT state. Combine with ref-typed args to check for stale references in recycled stack arg buffers.

## Task
Start by reading these files in order:
1. `src/execution/isolate.cc:4310-4330` — RetireWasmStack implementation (the fix)
2. `src/wasm/wasm-external-refs.cc:1033-1150` — suspend_wasmfx_stack, return_stack, return_wasmfx_stack, retire_stack
3. `src/wasm/wrappers-inl.h:570-640` — BuildWasmStackEntryWrapper with the MemRep fixes
4. `src/wasm/stacks.h` — StackMemory class, stack_obj accessor, set_stack_obj
5. `src/runtime/runtime-wasm.cc:2830-2850` — WasmFX stack creation

Name the type invariant to challenge: **Every WasmStackObject's external pointer must be either valid (pointing to live StackMemory) or null (cleared by RetireWasmStack). No continuation should hold a reference to a WasmStackObject with a stale/cleared external pointer.** The suspend→exception→retire path may leave a continuation holding a stale reference.

Build test modules using WasmModuleBuilder with:
- `--experimental-wasm-wasmfx` flags
- Continuations with ref-typed params/returns
- Exception-throwing operations after suspension
- Forced GC to trigger stack cache recycling

Start with v1 angle — trace the full stack lifecycle and test exception-path retirement.
