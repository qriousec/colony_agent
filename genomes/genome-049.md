# Genome 049 — wasm-null-js-null-confusion

## Hypothesis
WebAssembly has its own null sentinel (`kWasmNull`) distinct from JavaScript's `null`. At the Wasm↔JS boundary, `kWasmNull` must be converted to JS `null` and vice versa. The conversion happens in wrapper code (wrappers-inl.h, js-to-wasm.tq). If the conversion is incomplete:

1. **kWasmNull leaking to JS**: JS code receives an internal V8 object instead of `null`. This can lead to type confusion or crashes when JS operates on it.
2. **JS null entering Wasm as wrong value**: If `null` is passed to a Wasm function expecting a nullable ref type but the conversion doesn't produce `kWasmNull`, the Wasm code may fail a null check incorrectly.
3. **Externref null vs anyref null**: `extern.internalize(null)` should produce `kWasmNull`. `extern.externalize(kWasmNull)` should produce JS `null`. Are these consistent?

The WasmToJSObject macro at js-to-wasm.tq:867 handles this:
```torque
if (value == kWasmNull) return Null;
```

But is this check present in ALL paths? After tier-up? For multi-return? For globals?

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-shared --allow-natives-syntax
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

// TEST 1: Return null from non-nullable type function
(function TestNullReturn() {
  let b = new WasmModuleBuilder();
  // Function returns nullable externref
  b.addFunction('get_null', makeSig([], [kWasmExternRef]))
    .addBody([kExprRefNull, kExternRefCode]).exportFunc();
  // Function returns nullable anyref
  b.addFunction('get_null_any', makeSig([], [wasmRefNullType(kWasmAnyRef)]))
    .addBody([kExprRefNull, kAnyRefCode]).exportFunc();
  let inst = b.instantiate();

  // Before tier-up
  let r1 = inst.exports.get_null();
  print('TEST1a externref null before: ' + r1 + ' === null: ' + (r1 === null));
  let r2 = inst.exports.get_null_any();
  print('TEST1b anyref null before: ' + r2 + ' === null: ' + (r2 === null));

  // Tier up
  for (let i = 0; i < 10000; i++) {
    inst.exports.get_null();
    inst.exports.get_null_any();
  }

  // After tier-up
  let r3 = inst.exports.get_null();
  print('TEST1c externref null after: ' + r3 + ' === null: ' + (r3 === null));
  let r4 = inst.exports.get_null_any();
  print('TEST1d anyref null after: ' + r4 + ' === null: ' + (r4 === null));

  if (r3 !== null || r4 !== null) print('*** DIRTY: kWasmNull leaked to JS ***');
})();

// TEST 2: Pass null to Wasm, get it back
(function TestNullRoundtrip() {
  let b = new WasmModuleBuilder();
  b.addFunction('roundtrip', makeSig([kWasmExternRef], [kWasmExternRef]))
    .addBody([kExprLocalGet, 0]).exportFunc();
  let inst = b.instantiate();

  let before = inst.exports.roundtrip(null);
  print('TEST2a null roundtrip before: ' + before + ' === null: ' + (before === null));

  for (let i = 0; i < 10000; i++) inst.exports.roundtrip(null);

  let after = inst.exports.roundtrip(null);
  print('TEST2b null roundtrip after: ' + after + ' === null: ' + (after === null));
  if (after !== null) print('*** DIRTY: null conversion broken after tier-up ***');
})();

// TEST 3: extern.internalize(null) vs ref.null
(function TestInternalizeNull() {
  try {
    let b = new WasmModuleBuilder();
    // Return extern.internalize(ref.null extern) as anyref
    b.addFunction('intern_null', makeSig([], [wasmRefNullType(kWasmAnyRef)]))
      .addBody([
        kExprRefNull, kExternRefCode,
        kGCPrefix, kExprAnyConvertExtern
      ]).exportFunc();
    // Return ref.null any directly
    b.addFunction('ref_null_any', makeSig([], [wasmRefNullType(kWasmAnyRef)]))
      .addBody([kExprRefNull, kAnyRefCode]).exportFunc();
    let inst = b.instantiate();

    let r1 = inst.exports.intern_null();
    let r2 = inst.exports.ref_null_any();
    print('TEST3a intern_null: ' + r1 + ' === null: ' + (r1 === null));
    print('TEST3b ref_null_any: ' + r2 + ' === null: ' + (r2 === null));
    print('TEST3c same: ' + (r1 === r2));

    for (let i = 0; i < 10000; i++) {
      inst.exports.intern_null();
      inst.exports.ref_null_any();
    }

    let r3 = inst.exports.intern_null();
    let r4 = inst.exports.ref_null_any();
    print('TEST3d intern_null after: ' + r3 + ' === null: ' + (r3 === null));
    print('TEST3e ref_null_any after: ' + r4 + ' === null: ' + (r4 === null));
  } catch(e) { print('TEST3 ERROR: ' + e.message); }
})();
```

## Evolution Plan
- v1: Run the null confusion tests. Post results.
- v2: If CLEAN, test shared nulls: shared struct types with null fields, passing nulls through shared globals.
- v3: If any DIRTY, test undefined vs null confusion (JS undefined vs Wasm null).

## Task
Run the test first. Save to /tmp and execute with d8. Post results immediately.

Post with tags `null-confusion,wasm-js,boundary,w10,tier-up`.
