# Genome 028 — wasm-to-js-shared-apis

## Hypothesis
Shared Wasm GC objects (structs, arrays) pass through WasmToJSObject (wasm-objects.cc:3750-3772, js-to-wasm.tq:864-882) without any shared-space-specific handling. The string unshare fix (commit 64b77c7889e) only patches strings. When shared Wasm structs/arrays are returned to JS and used in JS APIs that enumerate properties, convert to string, or inspect object class — the APIs hit dispatch paths that don't handle shared-space Wasm GC objects. This is the same pattern as WASM-TC-001 (class_name() UNREACHABLE at js-objects.cc:588) and WASM-TC-002 (marking-visitor DCHECK for shared-space keys), applied to untested JS API siblings.

Multi-factor: Shared Wasm struct creation + externalization to JS + JS API invocation on shared-space object + missing dispatch for WasmStruct/WasmArray shared variants.

## DNA

### Confirmed Pattern (P4)
WASM-TC-001: `WeakSet.add(sharedWasmStruct)` → `ThrowTypeError` → `NoSideEffectsToString` → `class_name()` → `UNREACHABLE` at js-objects.cc:588. The `IsShared()` dispatch enumerates JSSharedStruct/JSSharedArray/JSAtomicsMutex/JSAtomicsCondition but misses WasmStruct/WasmArray shared variants.

WASM-TC-002: `WeakSet.add(sharedWasmStruct)` → GC → marking-visitor-inl.h:704 DCHECK(!HeapLayout::InWritableSharedSpace(key)) fails because WasmStruct IS a JSReceiver but IS in writable shared space.

### Source Facts

**WasmToJSObject** (wasm-objects.cc:3750-3772):
- IsWasmNull → null
- IsWasmFuncRef → get external
- IsString + InWritableSharedSpace → Unshare
- **Everything else → pass through** (no shared-space check for structs/arrays)

**Torque builtin** (js-to-wasm.tq:864-882):
- WasmNull → Null
- WasmFuncRef → check external
- String + InSharedSpace → runtime
- **Everything else → UnsafeCast<JSAny>(value)** (no shared-space check)

**class_name dispatch** (js-objects.cc:~588):
```
IsShared() switch covers: JSSharedStruct, JSSharedArray, JSAtomicsMutex, JSAtomicsCondition
MISSING: WasmStruct (shared), WasmArray (shared)
```

### Grep Commands
```bash
grep -rn "class_name\|NoSideEffectsToString\|UNREACHABLE" src/objects/js-objects.cc | head -20
grep -rn "IsShared.*switch\|case.*Shared" src/objects/js-objects.cc | head -20
grep -rn "InWritableSharedSpace\|InSharedSpace" src/objects/ | head -30
grep -rn "WasmStruct\|WasmArray" src/objects/js-objects.cc
```

### JS APIs to Test (Priority Order)
1. **Map/Set** (non-weak): `Map.set(sharedWasmStruct, v)` / `Set.add(sharedWasmStruct)` — different path from WeakMap/WeakSet
2. **JSON.stringify**: Calls `toJSON`, enumerates properties, converts to string
3. **Object.keys / Object.entries**: Property enumeration on shared objects
4. **String()**: Explicit toString conversion → may hit class_name()
5. **typeof**: Should return "object" but dispatch may be wrong
6. **structuredClone**: Cloning shared-space objects across isolation boundaries
7. **Reflect.ownKeys**: Property key enumeration
8. **for...in**: Property iteration
9. **Object.getOwnPropertyDescriptors**: Descriptor access
10. **console.log / util.inspect**: Debug formatting → class_name() path

### Related Commits
- 64b77c7889e: [wasm][shared] Unshare strings at the Wasm-to-JS boundary (NARROW FIX)
- 27a83f0b039: Reland [wasm] Consolidate WasmToJS/JSToWasm transformations (REMOVED type dispatching)
- cb4665e7b41: [wasm][shared] Fix DCHECK in decoding of invalid shared type

## Techniques That Work
1. Create shared Wasm module with `--experimental-wasm-shared`
2. Define shared struct type with fields (i32, f64)
3. Create instances via struct.new in shared function
4. Return shared struct to JS via externref or anyref export
5. Pass shared struct to each JS API in the priority list
6. Test in both debug (DCHECK) and release modes
7. Compare crash behavior vs expected error handling
8. For each crash, identify the dispatch path that hits UNREACHABLE or DCHECK

## DO NOT Try These
- WeakSet.add / WeakMap.set — already confirmed as WASM-TC-001/002
- String unsharing — already fixed (64b77c7889e)
- Shared custom descriptors — UNIMPLEMENTED (struct-types.h:87-90)

## Evolution Plan
- v1: **API sweep** — systematic test of top-10 JS APIs with shared Wasm struct. Run with debug d8 to catch DCHECKs. Focus on APIs that call class_name(), NoSideEffectsToString(), or IsShared() dispatch.
- v2: **Deep path analysis** — for any crash or DCHECK found in v1, trace the full call stack. Identify the missing dispatch case. Check if the same path can be triggered through other Wasm GC types (shared arrays, shared funcrefs).
- v3: **Release mode exploitation** — for DCHECK-only crashes (like WASM-TC-002), test release mode behavior. If the DCHECK is bypassed in release, what happens? Undefined behavior, memory corruption, or silent wrong result?

## Task
Start with v1. Create a test module:

1. Read `src/objects/js-objects.cc` around line 588 — find ALL IsShared() dispatch points and class_name() paths. List which object types are handled and which are missing.
2. Read `src/objects/js-objects-inl.h` — check for inline type checks that might miss shared Wasm types.
3. Build a shared Wasm module:
```javascript
let builder = new WasmModuleBuilder();
builder.startRecGroup();
let struct_type = builder.addStruct([makeField(kWasmI32, true), makeField(kWasmF64, true)], kNoSuperType, false, true /* shared */);
builder.endRecGroup();
// Export function that creates and returns shared struct
```
4. Test each JS API with the returned shared struct. Run with `--experimental-wasm-shared`.
5. Post results with tag `shared,js-api,pattern-export`.

Your type invariant to challenge: "All JS APIs that accept objects correctly handle shared-space Wasm GC objects." The confirmed bugs prove this invariant is violated for WeakSet/WeakMap — find more violations.
