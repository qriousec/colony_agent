# Genome 063 — tc006-kfunc-sibling

## Hypothesis
TC-006 showed that `kString` case in `JSToWasmObject` (wasm-objects.cc:3698) is missing `ConvertToSharedIfExpected`. Pattern export: does the `kFunc` case (line 3599-3609) also lack shared type handling?

The `kFunc` case:
```cpp
case GenericKind::kFunc: {
    if (!(WasmExternalFunction::IsWasmExternalFunction(*value) ||
          WasmCapiFunction::IsWasmCapiFunction(*value))) {
        *error_message = "...";
        return {};
    }
    return direct_handle(
        Cast<JSFunction>(*value)->shared()->wasm_function_data()->func_ref(),
        isolate);
}
```

No `ConvertToSharedIfExpected()` or `CheckExpectedSharedness()` is called. If the expected type is `(ref null shared func)` and a non-shared function is passed, the unwrapped `WasmFuncRef` could be in non-shared space while the Wasm code expects a shared-space value.

However, there's a subtlety: `WasmFuncRef` objects live in trusted space, not regular heap space. The sharing semantics for trusted-space objects may be different. This needs source verification.

Multi-factor: [kFunc case missing shared check] × [shared funcref type declaration] × [WasmFuncRef space allocation] = potential cross-space reference.

## DNA

### Source Facts
```cpp
// wasm-objects.cc:3599-3609 — kFunc case
// Returns func_ref() without any shared check
// Compare: kExtern (3611) has ConvertToSharedIfExpected
// Compare: kStruct (3641) has CheckExpectedSharedness
// Compare: kString (3698) is TC-006 — MISSING check

// Pattern P6: GenericKind switch missing shared handling
// Confirmed for kString (TC-006)
// Untested for kFunc

// wasm-objects.cc:3488-3504 — ConvertToSharedIfExpected
// Only operates on HeapObjects not in shared space
// Calls Object::Share to move to shared space

// wasm-objects.cc:3471-3486 — CheckExpectedSharedness
// Rejects values with wrong sharing status
```

### Key Questions
1. Can `(ref null shared func)` be declared as a valid Wasm type? Does the decoder accept shared funcref?
2. Where are WasmFuncRef objects allocated? Trusted space? Shared space?
3. If a non-shared function's WasmFuncRef is used in a shared context, does it cause GC issues?
4. Are there other GenericKind cases missing shared checks?

### Files to Audit
- `src/wasm/wasm-objects.cc` — JSToWasmObject GenericKind switch (full review)
- `src/wasm/function-body-decoder-impl.h` — Does it accept shared funcref types?
- `src/objects/wasm-objects.h` — WasmFuncRef allocation and space

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-shared --experimental-wasm-gc --allow-natives-syntax
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

// TEST 1: Can we declare shared funcref type?
(function TestSharedFuncRef() {
  try {
    let b = new WasmModuleBuilder();

    // Try shared funcref
    let kSharedFuncRef = wasmRefNullType(kWasmFuncRef);
    try {
      kSharedFuncRef = kSharedFuncRef.shared ? kSharedFuncRef.shared() : kSharedFuncRef;
    } catch(e) {
      print('TEST1: Cannot create shared funcref type: ' + e.message);
      return;
    }

    // Global of shared funcref type
    let g = b.addGlobal(kSharedFuncRef, true, false,
      [kExprRefNull, kFuncRefCode]);
    g.exportAs('g');

    // Function to set global
    b.addFunction('set', makeSig([kSharedFuncRef], []))
      .addBody([
        kExprLocalGet, 0,
        kExprGlobalSet, g.index
      ]).exportFunc();

    // Function to get global
    b.addFunction('get', makeSig([], [kSharedFuncRef]))
      .addBody([kExprGlobalGet, g.index]).exportFunc();

    let inst = b.instantiate();
    print('TEST1: Module with shared funcref global instantiated');

    // Create a regular (non-shared) function
    let b2 = new WasmModuleBuilder();
    b2.addFunction('dummy', makeSig([], [kWasmI32]))
      .addBody([kExprI32Const, 42]).exportFunc();
    let inst2 = b2.instantiate();

    // Try to set shared funcref global with non-shared function
    try {
      inst.exports.set(inst2.exports.dummy);
      print('TEST1: set(non-shared func) succeeded — CHECK if correctly shared');
    } catch(e) {
      print('TEST1: set(non-shared func) threw: ' + e.message);
    }

  } catch(e) {
    print('TEST1 ERROR: ' + e.message);
  }
})();

// TEST 2: Check function-body-decoder-impl.h for shared funcref validation
// If the decoder rejects shared funcref, this surface is architecturally blocked
(function TestSharedFuncRefDecoder() {
  try {
    let b = new WasmModuleBuilder();

    // Simpler: shared anyref global, store a function in it
    let kSharedAnyRef = wasmRefNullType(kWasmAnyRef).shared();
    let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();

    let g = b.addGlobal(kSharedAnyRef, true, true,
      [kExprRefNull, ...wasmSignedLeb(kSharedAnyRef.heap_type, 5)]);
    g.exportAs('g');

    b.addFunction('store_any', makeSig([kSharedExternRef], []))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kExprGlobalSet, g.index
      ]).exportFunc();

    b.addFunction('load_any', makeSig([], [kSharedAnyRef]))
      .addBody([kExprGlobalGet, g.index]).exportFunc();

    let inst = b.instantiate();

    // Get a function reference
    let b2 = new WasmModuleBuilder();
    b2.addFunction('target', makeSig([], [kWasmI32]))
      .addBody([kExprI32Const, 42]).exportFunc();
    let inst2 = b2.instantiate();

    // Store function via shared externref → shared anyref path
    try {
      inst.exports.store_any(inst2.exports.target);
      print('TEST2: Stored function in shared anyref global');

      let loaded = inst.exports.load_any();
      print('TEST2: Loaded value type: ' + typeof loaded);

    } catch(e) {
      print('TEST2: Store/load threw: ' + e.message);
    }

  } catch(e) {
    print('TEST2 ERROR: ' + e.message);
  }
})();

// TEST 3: Quick audit — is kEq missing checks for shared strings?
(function TestSharedEqRef() {
  try {
    let b = new WasmModuleBuilder();
    let kSharedEqRef = wasmRefNullType(kWasmEqRef).shared();
    let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();

    // Function taking shared eqref
    b.addFunction('take_eq', makeSig([kSharedExternRef], [kWasmI32]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kGCPrefix, kExprRefTest, ...wasmSignedLeb(kSharedEqRef.heap_type, 5),
      ]).exportFunc();

    let inst = b.instantiate();

    // Pass a string — should it be accepted as eqref?
    // In the spec, strings are NOT subtypes of eq
    print('TEST3: Module instantiated');

  } catch(e) {
    print('TEST3 ERROR: ' + e.message);
  }
})();

print('ALL TESTS COMPLETE');
```

## DO NOT Try These
- kString path — TC-006 already confirmed
- kAny/kExtern paths — already have ConvertToSharedIfExpected
- kStruct/kArray paths — already have CheckExpectedSharedness
- Compiled wrapper shared string leak — TC-003/TC-004

## Evolution Plan
- v1: Check if shared funcref is even a valid type in the decoder. If decoder rejects it, surface is architecturally blocked. If accepted, test kFunc path for sharing gap.

## Task
**SOURCE AUDIT FIRST.** Read:
1. `src/wasm/function-body-decoder-impl.h` — search for `kFuncRefCode` or `kFunc` in shared type validation. Does it accept `shared funcref`?
2. `src/wasm/wasm-objects.cc:3599-3609` — the kFunc case, check if ConvertToSharedIfExpected is needed
3. `src/objects/wasm-objects.h` — WasmFuncRef allocation space (trusted vs shared)

If the decoder rejects shared funcref (like it rejects shared exnref and contref), post as CLEAN and explain why the surface is architecturally blocked.

Post with tags `tc-006,kfunc,sibling,shared-funcref,GenericKind,pattern-export`.
