# Genome 039 — class-name-shared-siblings

## Hypothesis
WASM-TC-001 crashed V8 via WeakSet.add(sharedWasmStruct) → ThrowTypeError → NoSideEffectsToString → class_name() → UNREACHABLE at js-objects.cc:588. The fix added `IsWasmObjectInstanceType` + `IN_WRITABLE_SHARED_SPACE` checks to `GotoIfCannotBeHeldWeakly` in builtins-collections-gen.cc:495-502. But class_name() has 8+ OTHER callers that don't go through this guard:

1. **Object::NoSideEffectsToString()** (objects.cc:724) — called for ANY error message formatting. Any JS operation that throws a TypeError with a shared Wasm object as context reaches this.
2. **JSON.stringify circular ref** (json-stringifier.cc:869) — `AppendConstructorName()` for circular reference error messages uses `class_name()` directly.
3. **Inspector/Debugger** (value-mirror.cc:310,363,401) — `descriptionForError()`, `ObjectMirror::description()` call `GetConstructorName()` which can reach `class_name()`.
4. **Heap profiler** (heap-snapshot-generator.cc:2548,918) — `GetConstructorName()` for snapshot naming.
5. **API DeepFreeze** (api.cc:7035,7055) — `DeepFreezeRecursionVisitor::Visit()` calls `class_name()` directly.
6. **Stack traces** (call-site-info.cc:542) — `GetFunctionName()` for error stack formatting.
7. **Super constructor error** (runtime-classes.cc:86) — `ThrowNotSuperConstructor()` calls `NoSideEffectsToString()`.
8. **WeakRef/FinalizationRegistry** — PROTECTED (use GotoIfCannotBeHeldWeakly). NOT a target.

The root cause is in `class_name()` itself (js-objects.cc:582-588): the `IsShared(*this)` check catches shared Wasm objects (they ARE in shared space), but only enumerates JSSharedStruct, JSSharedArray, JSAtomicsMutex, JSAtomicsCondition — NOT WasmStruct, WasmArray, WasmFuncRef. Shared Wasm GC objects hit UNREACHABLE().

Multi-factor: Shared Wasm GC object + JS API that calls class_name() or NoSideEffectsToString + error path triggering string conversion.

## DNA

### Root Cause (UNFIXED)
**js-objects.cc:582-588**:
```cpp
if (IsShared(*this)) {
    if (IsJSSharedStruct(*this)) return roots.SharedStruct_string();
    if (IsJSSharedArray(*this)) return roots.SharedArray_string();
    if (IsJSAtomicsMutex(*this)) return roots.AtomicsMutex_string();
    if (IsJSAtomicsCondition(*this)) return roots.AtomicsCondition_string();
    // Other shared values are primitives.
    UNREACHABLE();  // ← CRASHES here for shared WasmStruct/WasmArray
}
```

### Callers of class_name() / NoSideEffectsToString()
```
objects.cc:724 — NoSideEffectsToString → class_name() (via Cast<JSReceiver>)
json-stringifier.cc:869 — AppendConstructorName → class_name() directly
value-mirror.cc:310,363,401 — Inspector APIs → GetConstructorName
heap-snapshot-generator.cc:2548,918 — Heap profiler → GetConstructorName
api.cc:7035,7055 — DeepFreeze → class_name() directly
call-site-info.cc:542 — Stack traces → NoSideEffectsToString
runtime-classes.cc:86 — Super constructor → NoSideEffectsToString
```

### Fixed Callers (DO NOT test)
- WeakSet/WeakMap/WeakRef/FinalizationRegistry — guarded by GotoIfCannotBeHeldWeakly (builtins-collections-gen.cc:495-502)

### Grep Commands
```bash
# All callers of class_name()
grep -rn "class_name()" src/objects/ src/api/ src/json/ src/inspector/ src/profiler/ | head -20

# All callers of NoSideEffectsToString
grep -rn "NoSideEffectsToString" src/ --include="*.cc" | head -20

# GetConstructorName callers
grep -rn "GetConstructorName" src/ --include="*.cc" | head -20
```

## Techniques That Work
1. Create shared Wasm struct: `--experimental-wasm-shared` + shared struct type in module
2. Get the shared struct into JS via Wasm export function
3. Trigger each caller path:
   - `JSON.stringify` with circular ref containing shared Wasm struct
   - `v8.deepFreeze` (API) on shared Wasm struct
   - Throw an error where shared Wasm struct appears in error context
   - Take heap snapshot while shared Wasm struct exists
   - Use shared Wasm struct in a class that triggers super constructor check
4. Run in debug build (DCHECK) AND release build (potential UB)
5. Each crash = same root cause (class_name()) but different trigger vector

## DO NOT Try These
- WeakSet.add / WeakMap.set — already found (TC-001), now fixed
- WeakRef / FinalizationRegistry — now guarded
- Direct class_name() call from C++ — not reachable from user code

## Evolution Plan
- v1: **Easiest crash vectors**. Test: (a) `JSON.stringify({a: sharedWasmStruct}, (k,v) => v)` with circular ref, (b) Pass shared Wasm struct to any API that triggers ThrowTypeError → NoSideEffectsToString (e.g., `Object.defineProperty(sharedWasmStruct, ...)`, `Proxy.revocable(sharedWasmStruct, ...)`). Confirm crash in debug build.
- v2: **Deep freeze + Inspector**. Test: (a) `v8.deepFreeze(sharedWasmStruct)` — direct class_name() call at api.cc:7035, (b) Build scenario where inspector value-mirror hits shared Wasm object during error inspection. (c) Trigger heap profiler during GC with live shared Wasm objects.
- v3: **Release build impact analysis**. For each confirmed debug crash, run in release build. In release, UNREACHABLE() is `__builtin_unreachable()` which is UB — the function may return garbage or fall through to `return roots.Object_string()`. Determine: does release silently continue (benign) or corrupt control flow (exploitable)?

## Task
Start with v1. This is a PATTERN EXPORT from WASM-TC-001 — the root cause (class_name() UNREACHABLE) was never fixed, only the WeakCollection entry point was guarded. Your job is to find ALL other entry points.

1. Read `src/objects/js-objects.cc:536-592` — understand class_name() dispatch. Confirm shared WasmStruct hits UNREACHABLE.
2. Read `src/objects/objects.cc` around line 724 — understand NoSideEffectsToString. What types does it handle? When does it call class_name()?
3. Build the shared Wasm struct test module (same as TC-001).
4. Try each trigger path from the DNA section. For each, confirm crash in debug build.
5. Post ALL confirmed crash vectors with exact trigger path.

Post with tags `class-name,shared,crash,pattern-export,tc-001-siblings`.
