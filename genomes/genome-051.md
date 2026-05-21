# Genome 051 — compiled-wrapper-allrefs

## Hypothesis
The compiled Turboshaft wrapper's `ToJS` method (wrappers-inl.h:138) gates shared string unsharing on `IsSubtypeOf(type, kWasmSharedExternRef)`. This ONLY triggers for declared type `(ref shared extern)` or subtypes. But shared strings can exist in ANY shared ref type: `(ref shared any)`, `(ref shared eq)`, `(ref shared i31)`, shared struct types, shared array types. ALL these paths skip the unshare guard in the compiled wrapper and return shared strings to JS as-is after tier-up.

Multi-factor: [shared string value] × [non-shared-extern declared type] × [compiled Turboshaft wrapper after tier-up] = shared string leaks to JS.

Additionally, commit 0a3073a0b0c added `NewSharedStringFromUtf8` for shared string constants via the magic import path. These strings are created directly in shared space at module instantiation time. If returned through a non-shared-extern function, the compiled wrapper won't unshare them.

## DNA

### Source Facts

**Compiled wrapper unshare guard** (wrappers-inl.h:138):
```cpp
if (IsSubtypeOf(type, kWasmSharedExternRef)) {
  // Only triggers for shared externref and subtypes
  // Checks instance type < FIRST_NONSTRING_TYPE
  // Calls Runtime::kWasmWasmToJSObject for strings
}
```

**After the guard (lines 159-171):**
```cpp
if (!type.is_nullable()) return ret;      // returns as-is
if (!type.use_wasm_null()) return ret;    // returns as-is
// Nullable: convert WasmNull → JS null, otherwise return as-is
```

**Generic Torque wrapper** (js-to-wasm.tq:881):
```torque
if (Is<String>(value) && InSharedSpace(UnsafeCast<String>(value))) {
  return runtime::WasmWasmToJSObject(context, value);
}
```
This is VALUE-based, catches ALL shared strings regardless of declared type.

**C++ runtime** (wasm-objects.cc:3762):
```cpp
if (IsString(*value)) {
  DirectHandle<String> string = Cast<String>(value);
  if (HeapLayout::InWritableSharedSpace(*string)) {
    return String::Unshare(isolate, string);
  }
}
```
Also VALUE-based, catches all shared strings.

**Shared string constants** (commit 0a3073a0b0c):
- `module-instantiate.cc:1818`: `NewSharedStringFromUtf8` called for shared globals
- Creates strings directly in shared space
- These strings are stored in shared globals at instantiation time

**Key types that can carry shared strings but bypass the guard:**
- `kWasmSharedAnyRef` — `(ref shared any)` — strings are subtypes of anyref
- `kWasmSharedEqRef` — `(ref shared eq)` — strings are subtypes of eqref
- Shared struct types with shared anyref fields — `struct.get` returns shared anyref
- Shared array types with shared anyref elements — `array.get` returns shared anyref
- `kWasmSharedI31Ref` — probably cannot carry strings, but check

**Wrapper path selection:**
- Before ~10K calls: generic Torque wrapper (CORRECT unsharing)
- After ~10K calls: compiled Turboshaft wrapper (INCOMPLETE unsharing)
- This creates a tier-up differential: pre-warmup CLEAN, post-warmup DIRTY

### Related CVEs/Bugs
- TC-003: Confirmed on externref path
- TC-004: Confirmed on anyref path (via externref→anyref conversion)
- Bug 448741522, 42204563: Tracked by V8 team

## Techniques That Work
- Create shared string via `new (new SharedStructType(['str']))` constructor
- Use `%IsSharedString(value)` to check if string is shared
- Tier up with 10K+ calls to trigger compiled wrapper
- Compare before/after tier-up behavior

## DO NOT Try These
- Testing externref path — ALREADY CONFIRMED DIRTY by TC-003
- Testing anyref via externref→anyref conversion — ALREADY CONFIRMED DIRTY by TC-004
- Testing generic wrapper (Torque) directly — it's CORRECT, only compiled wrapper is buggy

## Evolution Plan
- v1: Test ALL shared ref types in the compiled wrapper: shared anyref (direct, not via convert), shared eqref, shared struct.get, shared array.get. Run with tier-up. Check each for shared string leak.
- v2: Test shared string constants (magic import path) — create a shared global string constant, return it through various function types, check if unshared after tier-up.
- v3: If dirty results found, cross-reference with wrappers-inl.h source to confirm the EXACT type path that leaks. Build minimal reproducer for each leaking type.

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-shared --allow-natives-syntax --shared-string-table --harmony-struct
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

// Create shared string
let ss = new (new SharedStructType(['str']));
ss.str = 'compiled-wrapper-test';
let sharedStr = ss.str;
print('Setup: isShared=' + %IsSharedString(sharedStr));

function tierUp(fn, arg) { for (let i = 0; i < 15000; i++) fn(arg); }

// =============================================
// TEST 1: Shared anyref DIRECT (not via convert)
// The value is a shared string, declared type is (ref shared null any)
// =============================================
(function TestSharedAnyrefDirect() {
  try {
    let b = new WasmModuleBuilder();
    let kSharedAnyRef = wasmRefNullType(kWasmAnyRef).shared();
    let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();

    // Import shared string as shared externref, convert to shared anyref, return
    b.addFunction('through_shared_any',
        makeSig([kSharedExternRef], [kSharedAnyRef]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern  // extern → any
      ]).exportFunc();

    // Also test: return as shared externref (control — known DIRTY)
    b.addFunction('through_shared_extern',
        makeSig([kSharedExternRef], [kSharedExternRef]))
      .addBody([kExprLocalGet, 0]).exportFunc();

    let inst = b.instantiate();

    // Test shared anyref path
    let before_any = %IsSharedString(inst.exports.through_shared_any(sharedStr));
    tierUp(inst.exports.through_shared_any, sharedStr);
    let after_any = %IsSharedString(inst.exports.through_shared_any(sharedStr));
    print('TEST1 shared_anyref: before=' + before_any + ' after=' + after_any +
          (after_any ? ' *** DIRTY ***' : ' CLEAN'));

    // Control: shared externref
    let before_ext = %IsSharedString(inst.exports.through_shared_extern(sharedStr));
    tierUp(inst.exports.through_shared_extern, sharedStr);
    let after_ext = %IsSharedString(inst.exports.through_shared_extern(sharedStr));
    print('TEST1 shared_extern(ctrl): before=' + before_ext + ' after=' + after_ext +
          (after_ext ? ' *** DIRTY ***' : ' CLEAN'));
  } catch(e) { print('TEST1 ERROR: ' + e.message); }
})();

// =============================================
// TEST 2: Shared eqref path
// =============================================
(function TestSharedEqref() {
  try {
    let b = new WasmModuleBuilder();
    let kSharedEqRef = wasmRefNullType(kWasmEqRef).shared();
    let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();

    b.addFunction('through_shared_eq',
        makeSig([kSharedExternRef], [kSharedEqRef]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern  // extern → any (which is supertype of eq)
        // Note: need ref.cast to eq if any→eq isn't implicit
      ]).exportFunc();

    let inst = b.instantiate();
    let before = %IsSharedString(inst.exports.through_shared_eq(sharedStr));
    tierUp(inst.exports.through_shared_eq, sharedStr);
    let after = %IsSharedString(inst.exports.through_shared_eq(sharedStr));
    print('TEST2 shared_eqref: before=' + before + ' after=' + after +
          (after ? ' *** DIRTY ***' : ' CLEAN'));
  } catch(e) { print('TEST2 ERROR: ' + e.message); }
})();

// =============================================
// TEST 3: Via shared struct field (struct.get returns shared anyref)
// =============================================
(function TestSharedStructField() {
  try {
    let b = new WasmModuleBuilder();
    let kSharedAnyRef = wasmRefNullType(kWasmAnyRef).shared();
    let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();

    let st = b.addStruct({
      fields: [makeField(kSharedAnyRef, true)],
      shared: true
    });

    // Store shared string in struct, return via struct.get as shared anyref
    b.addFunction('store_and_get',
        makeSig([kSharedExternRef], [kSharedAnyRef]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kGCPrefix, kExprStructNew, st,
        kGCPrefix, kExprStructGet, st, 0
      ]).exportFunc();

    let inst = b.instantiate();
    let before = %IsSharedString(inst.exports.store_and_get(sharedStr));
    tierUp(inst.exports.store_and_get, sharedStr);
    let after = %IsSharedString(inst.exports.store_and_get(sharedStr));
    print('TEST3 struct_field: before=' + before + ' after=' + after +
          (after ? ' *** DIRTY ***' : ' CLEAN'));
  } catch(e) { print('TEST3 ERROR: ' + e.message); }
})();

// =============================================
// TEST 4: Via shared array element (array.get returns shared anyref)
// =============================================
(function TestSharedArrayElement() {
  try {
    let b = new WasmModuleBuilder();
    let kSharedAnyRef = wasmRefNullType(kWasmAnyRef).shared();
    let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();

    let arr = b.addArray(kSharedAnyRef, true, kNoSuperType, false, true);  // shared

    b.addFunction('store_and_get_arr',
        makeSig([kSharedExternRef], [kSharedAnyRef]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kGCPrefix, kExprArrayNewFixed, arr, 1,
        kExprI32Const, 0,
        kGCPrefix, kExprArrayGet, arr
      ]).exportFunc();

    let inst = b.instantiate();
    let before = %IsSharedString(inst.exports.store_and_get_arr(sharedStr));
    tierUp(inst.exports.store_and_get_arr, sharedStr);
    let after = %IsSharedString(inst.exports.store_and_get_arr(sharedStr));
    print('TEST4 array_element: before=' + before + ' after=' + after +
          (after ? ' *** DIRTY ***' : ' CLEAN'));
  } catch(e) { print('TEST4 ERROR: ' + e.message); }
})();

// =============================================
// TEST 5: Non-shared externref carrying shared string
// (shared string passed through NON-shared externref function)
// =============================================
(function TestNonSharedExternref() {
  try {
    let b = new WasmModuleBuilder();
    b.addFunction('passthrough', makeSig([kWasmExternRef], [kWasmExternRef]))
      .addBody([kExprLocalGet, 0]).exportFunc();
    let inst = b.instantiate();
    let before = %IsSharedString(inst.exports.passthrough(sharedStr));
    tierUp(inst.exports.passthrough, sharedStr);
    let after = %IsSharedString(inst.exports.passthrough(sharedStr));
    print('TEST5 non_shared_externref: before=' + before + ' after=' + after +
          (after ? ' *** DIRTY ***' : ' CLEAN'));
  } catch(e) { print('TEST5 ERROR: ' + e.message); }
})();

// =============================================
// TEST 6: Shared string constant via magic import
// =============================================
(function TestSharedStringConstant() {
  try {
    let b = new WasmModuleBuilder();
    let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();
    let kSharedAnyRef = wasmRefNullType(kWasmAnyRef).shared();

    // Import a shared string constant
    let g = b.addImportedGlobal('wasm:text-encoder', 'shared-const-test',
                                 kSharedAnyRef, false);
    // Actually: shared string constants use the "wasm:js-string" module
    // Let's try the direct approach instead

    // Store shared string in global, return it
    let g2 = b.addGlobal(kSharedAnyRef, true, false,
      [kExprRefNull, ...wasmSignedLeb(kSharedAnyRef.heap_type, 5)]);

    b.addFunction('set_global', makeSig([kSharedExternRef], []))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kExprGlobalSet, g2.index
      ]).exportFunc();

    b.addFunction('get_as_any', makeSig([], [kSharedAnyRef]))
      .addBody([kExprGlobalGet, g2.index]).exportFunc();

    b.addFunction('get_as_extern', makeSig([], [kSharedExternRef]))
      .addBody([
        kExprGlobalGet, g2.index,
        kGCPrefix, kExprExternConvertAny
      ]).exportFunc();

    let inst = b.instantiate();
    inst.exports.set_global(sharedStr);

    // Test shared anyref return
    let before_any = %IsSharedString(inst.exports.get_as_any());
    tierUp(inst.exports.get_as_any, undefined);
    let after_any = %IsSharedString(inst.exports.get_as_any());
    print('TEST6a global_as_any: before=' + before_any + ' after=' + after_any +
          (after_any ? ' *** DIRTY ***' : ' CLEAN'));

    // Test shared externref return (should be unshared after fix)
    let before_ext = %IsSharedString(inst.exports.get_as_extern());
    tierUp(inst.exports.get_as_extern, undefined);
    let after_ext = %IsSharedString(inst.exports.get_as_extern());
    print('TEST6b global_as_extern: before=' + before_ext + ' after=' + after_ext +
          (after_ext ? ' *** DIRTY ***' : ' CLEAN'));
  } catch(e) { print('TEST6 ERROR: ' + e.message); }
})();
```

Run with:
```bash
/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-shared --allow-natives-syntax --shared-string-table --harmony-struct /tmp/compiled_wrapper_allrefs.js
```

## Task
**RUN THE TEST FIRST.** Save to `/tmp/compiled_wrapper_allrefs.js` and execute immediately. Post ALL output with tags `compiled-wrapper,shared-string,allrefs,tc-003,tc-004,pattern-export`.

Then for v2, audit the compiled wrapper source at `src/wasm/wrappers-inl.h:86-172` (the full `ToJS` method) to trace EXACTLY which type branches lead to the unshare check and which skip it. Map the complete type→unshare decision tree.

For v3, if dirty results found, build MINIMAL reproducers for each leaking type path.
