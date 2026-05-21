# Genome 087 — shared-funcref-external

## Hypothesis
When a shared function reference (funcref from a shared Wasm module) crosses the Wasm→JS boundary, the runtime calls WasmInternalFunction::GetOrCreateExternal to create/retrieve the JS-facing WasmExternalFunction object. For shared funcref, the internal function resides in shared heap space, but the external function wrapper may be created in local isolate space. This cross-space relationship (shared internal → local external, or vice versa) may violate GC invariants: write barriers, marking, and object lifecycle assumptions don't expect this configuration. The compiled wrapper (wrappers-inl.h:91-132) has fast/slow paths for funcref extraction that may not handle the shared case.

## DNA
### Funcref Boundary Conversion Paths
- **wrappers-inl.h:91-132**: Compiled wrapper ToJS for funcref:
  - Nullable: checks WasmNull, then extracts `external` from internal function
  - Non-nullable: always extracts external
  - Fast path: `internal.external` field already cached
  - Slow path: `Builtin::kWasmInternalFunctionCreateExternal`
- **js-to-wasm.tq:871-880**: Torque WasmToJSObject for funcref:
  - Fast: `wasmFunc.internal.external` if exists
  - Slow: `runtime::WasmWasmToJSObject(context, value)`
- **wasm-objects.cc:3752-3761**: C++ WasmToJSObject for funcref:
  - Fast: `internal->try_get_external()`
  - Slow: `WasmInternalFunction::GetOrCreateExternal(isolate, internal)`

### Key Question: Shared Space Handling
When a funcref comes from a shared Wasm module:
- The WasmInternalFunction is in shared heap space
- GetOrCreateExternal must create a WasmExternalFunction
- Where is the WasmExternalFunction allocated? Shared space or local isolate space?
- If local: the shared internal function has a pointer to a local external function
  - This cross-space reference must have correct write barriers
  - Different isolates calling the same shared function get different externals
- If shared: all isolates share the same external function object
  - Must handle concurrent access, GC across isolates

### Related Code
- **wasm-objects.cc**: WasmInternalFunction::GetOrCreateExternal implementation
- **wasm-objects-inl.h**: WasmInternalFunction::external() accessor
- **wasm-objects.h**: WasmExternalFunction definition

### TC Pattern Connection
- TC-001: Shared WasmStruct passed to JS → crash at class_name() because IsShared dispatch missing WasmStruct
- Similar pattern: Shared funcref passed to JS → external function in wrong heap space → GC confusion

## Techniques That Work
1. Create shared Wasm module with exported function
2. Get funcref to shared function from JS
3. Pass shared funcref through Wasm→JS boundary
4. Force compiled wrapper tier-up (warmup)
5. Check if external function is in shared or local space
6. Trigger GC after external function creation
7. Test from multiple "simulated" isolates (if possible with d8 workers)
8. Compare Liftoff path vs Turboshaft compiled wrapper path

## DO NOT Try These
- Non-shared funcref boundary conversion — well-tested, standard path
- Shared string boundary conversion — TC-003/004/005 already confirmed
- Shared struct/array at JS APIs — shared-nonstring-boundary already handling

## Evolution Plan
- v1: Source trace WasmInternalFunction::GetOrCreateExternal. Read the implementation in wasm-objects.cc. Does it check if the function is shared? Does it allocate the external in the correct space? What write barrier is used when storing the external reference in the internal function?
- v2: Build test: shared Wasm module with exported function. Get funcref in JS. Force compiled wrapper tier-up. Check behavior: does the external function survive GC? Is it in the right heap space? Compare with Torque path (before tier-up).
- v3: Test edge cases: shared funcref stored in table, retrieved from table, passed to another module. Shared funcref as callback. Shared funcref in WeakRef (does this crash like shared struct in WeakMap?).

## Task
Start by reading:
1. `src/wasm/wasm-objects.cc` — search for `GetOrCreateExternal`. How does it create the external function? Does it handle shared internal functions?
2. `src/wasm/wasm-objects.h` — WasmInternalFunction::external() field. How is the external reference stored? Tagged field? External pointer?
3. `src/wasm/wrappers-inl.h:91-132` — compiled wrapper funcref extraction. Is there any shared-specific handling?
4. `src/builtins/js-to-wasm.tq:871-880` — Torque funcref extraction.

Key question: When `WasmInternalFunction::GetOrCreateExternal` is called for a shared internal function, in which heap space is the external allocated?

Use `--experimental-wasm-shared` with `/Users/t/v8/out/fuzzbuild/d8`.
