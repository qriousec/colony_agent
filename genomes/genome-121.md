# Genome 121 — interpreter-shared-string-leak

## Hypothesis
The TC-003 fix (64b77c7889e) added shared string unsharing at two Wasm→JS boundary points: (1) js-to-wasm.tq WasmToJSObject macro at line 881 and (2) wasm-objects.cc WasmToJSObject at line 3762. But the Wasm interpreter has its OWN WasmToJSObject at wasm-interpreter-runtime.cc:2590-2604 that was NOT updated. This function handles WasmFuncRef→external conversion and WasmNull→null conversion, but does NOT check for shared strings or call String::Unshare(). If a shared Wasm module runs via the interpreter and returns a shared string through externref, the string reaches JS in shared heap space without being unshared — same class of bug as TC-003 but in a different tier.

## DNA

### Source Facts (file:line)
- **Interpreter WasmToJSObject**: wasm-interpreter-runtime.cc:2590-2604 — handles FuncRef and WasmNull ONLY, NO string unshare
- **Torque fix**: js-to-wasm.tq:881-882 — `Is<String>(value) && InSharedSpace(UnsafeCast<String>(value))` → calls runtime
- **C++ fix**: wasm-objects.cc:3762 — `IsString(*value) && HeapLayout::InWritableSharedSpace(...)` → calls String::Unshare
- **Interpreter callers**: wasm-interpreter-runtime.cc:1146,2239 — WasmToJSObject called for ref/refnull return values
- **Comment at line 219**: "Note: WasmToJSObject(ref) already called in ContinueExecution or CallExternalJSFunction" — confirms interpreter relies on its own WasmToJSObject
- **Fix commit**: 64b77c7889e — "[wasm][shared] Unshare strings at the Wasm-to-JS boundary"
- **String::Unshare**: objects/string-inl.h — template function added by 64b77c7889e

### Gap Analysis
1. **Interpreter tier gap**: The interpreter is a third execution tier alongside Liftoff and Turboshaft. Fix 64b77c7889e updated the Torque wrapper (generic) and the C++ runtime (slow path), but the interpreter has its own fast path that bypasses both.
2. **When is the interpreter used?**: The Wasm interpreter is used for debugging (--wasm-interpret-all), for initial execution before compilation, and when Liftoff/Turboshaft are disabled.
3. **Shared module in interpreter**: To trigger: create a shared Wasm module, run with --wasm-interpret-all or equivalent, return a string through externref.
4. **Impact**: If a shared string reaches JS without being unshared, subsequent JS operations on it may violate the shared heap invariant — concurrent access from multiple isolates without proper synchronization.

### Exploitation Path
1. Create shared Wasm module with function that takes an externref and returns it
2. Pass a shared string (from a shared array or imported from another module) as externref
3. Run through interpreter (--wasm-interpret-all)
4. The return value bypasses the Torque/C++ unshare paths because the interpreter uses its own WasmToJSObject
5. The shared string reaches JS in shared space
6. Use the string from multiple workers simultaneously — data race

## Techniques That Work
1. Create shared Wasm module with externref-returning function
2. Run with `--wasm-interpret-all` to force interpreter execution
3. Pass a string that lives in shared space (via shared struct field or shared global)
4. Check if the returned string is in shared space (observable via SharedArrayBuffer-like behavior)
5. Use Worker to access the string concurrently — if it's still shared, concurrent access works
6. Compare behavior: Liftoff (has unshare via Torque wrapper) vs Interpreter (no unshare)

## DO NOT Try These
- Testing with Liftoff or Turboshaft — these use the fixed Torque/C++ paths
- Testing with non-shared strings — they're not in shared space, no issue
- Testing with non-shared modules — only shared modules produce shared strings
- Testing WasmFuncRef or WasmNull — interpreter handles these correctly

## Evolution Plan
- v1: Verify interpreter path: confirm that WasmToJSObject at wasm-interpreter-runtime.cc:2590 is called when interpreter returns a ref value. Trace the call chain from function return to WasmToJSObject. Check if there's an outer layer that catches shared strings before/after the interpreter's WasmToJSObject.
- v2: Build test: shared module, externref passthrough function, force interpreter. Pass a shared string, check if it's unshared on the JS side. If still shared → confirmed TC-003 variant.
- v3: If v2 confirms the leak: build multi-worker test demonstrating concurrent access to the leaked shared string. Minimize and document the PoC.

## Task
1. Read wasm-interpreter-runtime.cc:2590-2604 (WasmToJSObject) to confirm no string unshare logic.
2. Search for callers of WasmToJSObject in the interpreter: grep for "WasmToJSObject" in src/wasm/interpreter/.
3. Read the call chains at lines 1146 and 2239 — understand when WasmToJSObject is invoked.
4. Check if there's an outer unshare layer: search for "Unshare\|unshare\|SharedString" in src/wasm/interpreter/.
5. Check if the interpreter can run shared modules: search for "shared\|is_shared" in src/wasm/interpreter/.
6. Build test module: shared module with `(func (export "passthrough") (param externref) (result externref) local.get 0)`.
7. Run with: `--experimental-wasm-shared --wasm-interpret-all`
8. Start with Evolution Plan v1 (verify interpreter call chain).
