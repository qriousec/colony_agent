# Genome 083 — shared-nonstring-boundary

## Hypothesis
TC-003/004/005 confirmed that the compiled Turboshaft wrapper (wrappers-inl.h) uses a TYPE-based check `IsSubtypeOf(type, kWasmSharedExternRef)` for string unsharing, while Torque/C++ use VALUE-based checks (`Is<String>` && `InSharedSpace`). The fix at 64b77c7889e added string-specific unsharing. But shared NON-STRING objects (WasmStruct, WasmArray, i31) may violate boundary invariants when passing through the compiled wrapper — the TYPE-based check architecture means objects in shared heap space could leak into JS contexts that don't expect shared-space objects.

## DNA
### Confirmed Bug Pattern (TC-003/004/005)
- **wrappers-inl.h:138-150**: Compiled wrapper ToJS checks `IsSubtypeOf(type, kWasmSharedExternRef)`. For shared externref type, detects strings via instance_type < FIRST_NONSTRING_TYPE and routes to Runtime::kWasmWasmToJSObject for unsharing.
- **js-to-wasm.tq:879-882**: Torque macro checks `Is<String>(value) && InSharedSpace(...)` — VALUE-based, catches ALL shared strings regardless of declared type.
- **wasm-objects.cc:3764-3769**: C++ runtime checks `IsString(*value)` && `HeapLayout::InWritableSharedSpace(*string)` — VALUE-based.
- **Gap**: Compiled wrapper only acts on type=shared externref. Values flowing through non-shared externref, shared anyref, or non-shared anyref types bypass the check.

### Current State After Fix (64b77c7889e)
The fix added string unsharing at all three layers:
1. Torque: VALUE-based check for shared strings → routes to runtime
2. C++: VALUE-based check → calls String::Unshare()
3. Compiled wrapper: TYPE-based check for shared externref → routes to runtime

### Shared Non-String Objects at Boundary
When shared WasmStruct or WasmArray objects cross the Wasm→JS boundary:
- **wasm-objects.cc:3766**: "All other objects → return as-is" — shared structs/arrays pass through
- **js-to-wasm.tq:881**: "return value" — shared structs/arrays pass through
- **wrappers-inl.h**: Non-funcref, non-null, non-string → pass through as-is

### Question: Is "pass through as-is" safe for shared structs/arrays?
- Shared WasmStruct IS in writable shared space (confirmed by TC-002 marking-visitor-inl.h:704 DCHECK)
- TC-001 showed shared WasmStruct in WeakMap/WeakSet crashes via class_name()
- Does passing shared structs to OTHER JS APIs also crash?
- What about FinalizationRegistry, Map/Set, Proxy/Reflect, JSON.stringify?

### Boundary Conversion Paths
| Object Type | wrappers-inl.h (compiled) | js-to-wasm.tq (Torque) | wasm-objects.cc (C++) |
|---|---|---|---|
| WasmNull | → JS null | → JS null | → JS null |
| WasmFuncRef | → extract external | → extract external | → extract external |
| Shared String | → unshare (if type=shared extern) | → unshare (value check) | → unshare (value check) |
| Shared Struct | → pass through as-is | → pass through as-is | → pass through as-is |
| Shared Array | → pass through as-is | → pass through as-is | → pass through as-is |

## Techniques That Work
1. Create shared Wasm module with struct/array types
2. Export functions returning shared structs/arrays to JS
3. Force compiled wrapper tier-up with warmup (10000+ calls)
4. Pass returned shared objects to various JS APIs:
   - WeakMap/WeakSet (known crash — TC-001/002)
   - Map/Set
   - FinalizationRegistry
   - JSON.stringify/JSON.parse
   - Object.keys/Object.getOwnPropertyNames
   - structuredClone
   - Proxy/Reflect
5. Compare behavior with Torque wrapper (first few calls before tier-up) vs compiled wrapper (after tier-up)

## DO NOT Try These
- Shared string unsharing — already confirmed (TC-003/004/005) and fixed
- WeakMap/WeakSet with shared structs — already confirmed (TC-001/002)
- Testing non-shared objects — they work correctly

## Evolution Plan
- v1: Map shared struct/array boundary behavior across JS APIs. Identify which APIs crash/behave incorrectly when receiving shared-space WasmStruct/WasmArray objects. Focus on GC-related APIs (FinalizationRegistry, WeakRef) and serialization APIs (structuredClone, JSON.stringify).
- v2: Test the COMPILED WRAPPER specifically — does the tier-up from Torque→compiled wrapper change behavior for shared structs? The compiled wrapper's ToJS function doesn't do any special handling for shared structs (no unsharing, no wrapping). Does this create a tier differential?
- v3: Test FromJS direction — passing shared JS objects INTO Wasm through the compiled wrapper. Does `wrappers-inl.h` FromJS handle shared objects correctly? Check ConvertToSharedIfExpected in wasm-objects.cc:3485-3500.

## Task
Start by reading:
1. `src/wasm/wrappers-inl.h:63-160` — current ToJS function after fix. Map ALL type paths for shared objects.
2. `src/wasm/wrappers-inl.h:400-520` — FromJS function. How does it handle shared objects coming from JS?
3. `src/wasm/wasm-objects.cc:3747-3769` — WasmToJSObject C++ after fix.
4. `src/wasm/wasm-objects.cc:3504-3600` — JSToWasmObject for indexed shared types.

Then build test modules:
- Shared module with `(type $shared_struct (shared (struct (field i32))))`
- Export function returning shared struct as externref
- Export function returning shared struct as anyref
- Force compiled wrapper tier-up
- Pass to JS APIs: Map, Set, FinalizationRegistry, WeakRef, JSON.stringify, structuredClone

The type invariant to challenge: "Shared Wasm objects can safely be exposed to all JS APIs after crossing the Wasm→JS boundary." TC-001/002 proved this false for WeakMap/WeakSet — find the next unsafe API.

Use `--experimental-wasm-shared --harmony-struct` flags with `/Users/t/v8/out/fuzzbuild/d8`.
