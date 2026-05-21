# Genome 050 — shared-string-js-to-wasm

## Hypothesis
TC-003/TC-004 found shared strings LEAKING from Wasm to JS (missing unshare). The REVERSE direction may have a similar gap: when JS passes a NON-shared string to a Wasm function expecting a SHARED type, does V8 correctly SHARE the string?

The JS-to-Wasm conversion is handled by `JSToWasmObject` in wasm-objects.cc and the Torque macro `JSToWasmObject` in js-to-wasm.tq. For shared types, the JS string must be placed in shared space. If the sharing is incomplete (e.g., only done for shared externref but not shared anyref), a non-shared string enters shared Wasm space, which violates the invariant that shared space objects don't reference non-shared objects.

This could lead to:
1. GC violations (shared space references non-shared object)
2. Data races (non-shared string accessed from multiple threads)
3. DCHECK failures in debug builds

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-shared --allow-natives-syntax --shared-string-table --harmony-struct
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

// TEST: Pass a regular JS string to a shared Wasm function
(function TestJSToWasmSharing() {
  let b = new WasmModuleBuilder();
  let kSharedRefExtern = wasmRefType(kWasmExternRef).shared();
  let kSharedNullRefAny = wasmRefNullType(kWasmAnyRef).shared();

  // Shared struct with shared anyref field
  let st = b.addStruct({
    fields: [makeField(kSharedNullRefAny, true)],
    shared: true
  });

  // Function takes shared externref, stores as shared anyref in struct
  b.addFunction('store_in_shared', makeSig([kSharedRefExtern], [wasmRefType(st)]))
    .addBody([
      kExprLocalGet, 0,
      kGCPrefix, kExprAnyConvertExtern,
      kGCPrefix, kExprStructNew, st
    ]).exportFunc();

  // Getter
  b.addFunction('get_field', makeSig([wasmRefType(st)], [kSharedRefExtern]))
    .addBody([
      kExprLocalGet, 0,
      kGCPrefix, kExprStructGet, st, 0,
      kGCPrefix, kExprExternConvertAny
    ]).exportFunc();

  let inst;
  try {
    inst = b.instantiate();
  } catch(e) {
    print('Instantiation error: ' + e.message);
    return;
  }

  // Pass a regular (non-shared) JS string to shared Wasm function
  let regularStr = "regular-string";
  print('Input: isShared=' + %IsSharedString(regularStr));

  let struct_val;
  try {
    struct_val = inst.exports.store_in_shared(regularStr);
    print('Stored in shared struct successfully');
  } catch(e) {
    print('Store error: ' + e.message);
    return;
  }

  // Read back
  let result = inst.exports.get_field(struct_val);
  print('Read back: ' + result + ' isShared=' + %IsSharedString(result));

  // Now tier-up the getter
  for (let i = 0; i < 10000; i++) inst.exports.get_field(struct_val);
  let after = inst.exports.get_field(struct_val);
  print('After tier-up: ' + after + ' isShared=' + %IsSharedString(after));

  // Check: was the input string properly shared when entering Wasm?
  // If not shared, the struct in shared space holds a non-shared string → invariant violation
  try {
    print('InWritableSharedSpace result: ' + %IsInWritableSharedSpace(result));
  } catch(e) {}
})();

// TEST 2: Shared global with non-shared string
(function TestSharedGlobal() {
  try {
    let b = new WasmModuleBuilder();
    let kSRE = wasmRefType(kWasmExternRef).shared();
    let g = b.addImportedGlobal("m", "g", kSRE, false);
    b.addFunction('get', makeSig([], [kSRE]))
      .addBody([kExprGlobalGet, g]).exportFunc();
    // Pass a non-shared string as shared global
    let inst = b.instantiate({m: {g: "non-shared-string"}});
    let val = inst.exports.get();
    print('TEST2 shared global: ' + val + ' isShared=' + %IsSharedString(val));
    // Tier up
    for (let i = 0; i < 10000; i++) inst.exports.get();
    let after = inst.exports.get();
    print('TEST2 after tier-up: ' + after + ' isShared=' + %IsSharedString(after));
  } catch(e) { print('TEST2 ERROR: ' + e.message); }
})();
```

## Evolution Plan
- v1: Run test. Check if non-shared strings get properly shared when entering Wasm shared space.
- v2: If sharing works, try to find paths where it's skipped (e.g., table.set with shared table, non-shared value).
- v3: Test the interaction with GC — if a non-shared string enters shared space without being shared, GC may crash.

## Task
Run the test first. Save and execute with d8. Post ALL output.

Post with tags `js-to-wasm,sharing,shared-string,reverse-direction,boundary`.
