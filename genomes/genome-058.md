# Genome 058 — shared-string-constant-leak

## Hypothesis
W18: Shared string constants created via `NewSharedStringFromUtf8` (commit 0a3073a0b0c) at module instantiation time. These strings are stored directly in shared space. If accessed through:

1. **Global.get JS API** — does `WebAssembly.Global.prototype.value` getter unshare the string?
2. **Compiled wrapper return** — same TC-003 bug: the compiled Turboshaft wrapper's `ToJS` only unshares for `kWasmSharedExternRef`, so shared anyref/eqref globals with string constants will leak.
3. **Table.get JS API** — if a shared string constant is stored in a table, does `WebAssembly.Table.prototype.get` unshare it?
4. **Exception payload** — if a shared string constant is thrown as exception payload, does the JS catch unshare it?

The shared string constants path is interesting because these strings are ALWAYS shared (created in shared space at instantiation), unlike TC-003 where shared strings had to be passed in from JS. This means the leak is deterministic — no need to create SharedStructType or pass shared strings through Wasm.

Multi-factor: [shared string constant in shared space] × [JS API access path] × [missing unshare guard] = shared string leaks to JS.

## DNA

### Source Facts
```cpp
// module-instantiate.cc:1818 — shared string constant creation
auto str = isolate->factory()->NewSharedStringFromUtf8(
    base::CStrVector(import_name.data()), AllocationType::kSharedOld);

// This is used for "wasm:js-string" module imports with shared globals
// The resulting string is in shared space
```

### JS API Paths
- `WebAssembly.Global.prototype.value` → calls `GetGlobalValue` in `wasm-js.cc`
- `WebAssembly.Table.prototype.get` → calls `GetFunctionTableEntry` or direct read
- `WebAssembly.Exception.getArg` → reads exception payload

### Key Question
Does the JS API path for global.get go through the same WasmToJSObject conversion that handles unsharing? Or does it bypass the Wasm wrapper entirely?

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-shared --allow-natives-syntax --shared-string-table --harmony-struct
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

function tierUp(fn) {
  for (let i = 0; i < 15000; i++) {
    try { fn(); } catch(e) {}
  }
}

// Create shared string for comparison
let ss = new (new SharedStructType(['s']));
ss.s = 'shared-string-constant-test';
let sharedStr = ss.s;
print('Setup: isShared(sharedStr)=' + %IsSharedString(sharedStr));

// =============================================
// TEST 1: Shared global with string, accessed via JS API
// =============================================
(function TestGlobalJSAPI() {
  try {
    let b = new WasmModuleBuilder();
    let kSharedAnyRef = wasmRefNullType(kWasmAnyRef).shared();
    let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();

    // Shared mutable global of type (ref null shared any)
    let g = b.addGlobal(kSharedAnyRef, true, true,  // mutable, shared
      [kExprRefNull, ...wasmSignedLeb(kSharedAnyRef.heap_type, 5)]);
    g.exportAs('g');

    // Function to set global from Wasm side
    b.addFunction('set', makeSig([kSharedExternRef], []))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kExprGlobalSet, g.index
      ]).exportFunc();

    // Function to get global (goes through Wasm wrapper)
    b.addFunction('get_any', makeSig([], [kSharedAnyRef]))
      .addBody([kExprGlobalGet, g.index]).exportFunc();

    b.addFunction('get_extern', makeSig([], [kSharedExternRef]))
      .addBody([
        kExprGlobalGet, g.index,
        kGCPrefix, kExprExternConvertAny
      ]).exportFunc();

    let inst = b.instantiate();
    inst.exports.set(sharedStr);

    // JS API: global.value
    let jsApiResult = inst.exports.g.value;
    print('TEST1a JS API global.value: isShared=' + %IsSharedString(jsApiResult) +
          ((%IsSharedString(jsApiResult)) ? ' *** DIRTY ***' : ' CLEAN'));

    // Wasm wrapper: get_any (before tier-up)
    let wasmAnyBefore = inst.exports.get_any();
    print('TEST1b Wasm get_any before tierup: isShared=' + %IsSharedString(wasmAnyBefore) +
          ((%IsSharedString(wasmAnyBefore)) ? ' *** DIRTY ***' : ' CLEAN'));

    // Wasm wrapper: get_any (after tier-up)
    tierUp(inst.exports.get_any);
    let wasmAnyAfter = inst.exports.get_any();
    print('TEST1c Wasm get_any after tierup: isShared=' + %IsSharedString(wasmAnyAfter) +
          ((%IsSharedString(wasmAnyAfter)) ? ' *** DIRTY ***' : ' CLEAN'));

    // Wasm wrapper: get_extern (before tier-up — fresh instance)
    let b2 = new WasmModuleBuilder();
    let g2 = b2.addGlobal(kSharedAnyRef, true, true,
      [kExprRefNull, ...wasmSignedLeb(kSharedAnyRef.heap_type, 5)]);
    b2.addFunction('set', makeSig([kSharedExternRef], []))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kExprGlobalSet, g2.index
      ]).exportFunc();
    b2.addFunction('get_extern', makeSig([], [kSharedExternRef]))
      .addBody([
        kExprGlobalGet, g2.index,
        kGCPrefix, kExprExternConvertAny
      ]).exportFunc();
    let inst2 = b2.instantiate();
    inst2.exports.set(sharedStr);

    let extBefore = inst2.exports.get_extern();
    print('TEST1d Wasm get_extern before tierup: isShared=' + %IsSharedString(extBefore) +
          ((%IsSharedString(extBefore)) ? ' *** DIRTY ***' : ' CLEAN'));

    tierUp(inst2.exports.get_extern);
    let extAfter = inst2.exports.get_extern();
    print('TEST1e Wasm get_extern after tierup: isShared=' + %IsSharedString(extAfter) +
          ((%IsSharedString(extAfter)) ? ' *** DIRTY ***' : ' CLEAN'));

  } catch(e) { print('TEST1 ERROR: ' + e.message); }
})();

// =============================================
// TEST 2: Shared string in table, accessed via JS API
// =============================================
(function TestTableJSAPI() {
  try {
    let b = new WasmModuleBuilder();
    let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();
    let kSharedAnyRef = wasmRefNullType(kWasmAnyRef).shared();

    // Table of shared externref
    let t = b.addTable(kSharedExternRef, 1, 1, undefined, true);  // shared table
    t.exportAs('t');

    b.addFunction('set', makeSig([kSharedExternRef], []))
      .addBody([
        kExprI32Const, 0,
        kExprLocalGet, 0,
        kExprTableSet, t.index
      ]).exportFunc();

    b.addFunction('get', makeSig([], [kSharedExternRef]))
      .addBody([
        kExprI32Const, 0,
        kExprTableGet, t.index
      ]).exportFunc();

    let inst = b.instantiate();
    inst.exports.set(sharedStr);

    // JS API: table.get
    let jsApiResult = inst.exports.t.get(0);
    print('TEST2a JS API table.get: isShared=' + %IsSharedString(jsApiResult) +
          ((%IsSharedString(jsApiResult)) ? ' *** DIRTY ***' : ' CLEAN'));

    // Wasm get before tier-up
    let wasmBefore = inst.exports.get();
    print('TEST2b Wasm table get before tierup: isShared=' + %IsSharedString(wasmBefore) +
          ((%IsSharedString(wasmBefore)) ? ' *** DIRTY ***' : ' CLEAN'));

    // Wasm get after tier-up
    tierUp(inst.exports.get);
    let wasmAfter = inst.exports.get();
    print('TEST2c Wasm table get after tierup: isShared=' + %IsSharedString(wasmAfter) +
          ((%IsSharedString(wasmAfter)) ? ' *** DIRTY ***' : ' CLEAN'));

  } catch(e) { print('TEST2 ERROR: ' + e.message); }
})();

// =============================================
// TEST 3: Shared string via exception payload
// =============================================
(function TestExceptionPayload() {
  try {
    let b = new WasmModuleBuilder();
    let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();

    let tag = b.addTag(makeSig([kSharedExternRef], []));
    b.addExportOfKind('tag', kExternalTag, tag);

    b.addFunction('throw_shared', makeSig([kSharedExternRef], []))
      .addBody([
        kExprLocalGet, 0,
        kExprThrow, tag
      ]).exportFunc();

    b.addFunction('catch_shared', makeSig([kSharedExternRef], [kSharedExternRef]))
      .addBody([
        kExprTry, kWasmVoid,
          kExprLocalGet, 0,
          kExprThrow, tag,
        kExprCatch, tag,
          kExprReturn,
        kExprEnd,
        kExprUnreachable,
      ]).exportFunc();

    let inst = b.instantiate();

    // Catch from JS side
    try {
      inst.exports.throw_shared(sharedStr);
    } catch(e) {
      if (e instanceof WebAssembly.Exception) {
        let payload = e.getArg(inst.exports.tag, 0);
        print('TEST3a JS catch exception payload: isShared=' + %IsSharedString(payload) +
              ((%IsSharedString(payload)) ? ' *** DIRTY ***' : ' CLEAN'));
      } else {
        print('TEST3a: Not a WebAssembly.Exception: ' + e);
      }
    }

    // Catch from Wasm side, return through wrapper
    let wasmBefore = inst.exports.catch_shared(sharedStr);
    print('TEST3b Wasm catch before tierup: isShared=' + %IsSharedString(wasmBefore) +
          ((%IsSharedString(wasmBefore)) ? ' *** DIRTY ***' : ' CLEAN'));

    tierUp(() => inst.exports.catch_shared(sharedStr));
    let wasmAfter = inst.exports.catch_shared(sharedStr);
    print('TEST3c Wasm catch after tierup: isShared=' + %IsSharedString(wasmAfter) +
          ((%IsSharedString(wasmAfter)) ? ' *** DIRTY ***' : ' CLEAN'));

  } catch(e) { print('TEST3 ERROR: ' + e.message); }
})();

// =============================================
// TEST 4: Non-shared anyref carrying shared string constant
// =============================================
(function TestNonSharedAnyrefConstant() {
  try {
    let b = new WasmModuleBuilder();
    let kSharedAnyRef = wasmRefNullType(kWasmAnyRef).shared();
    let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();

    // Store shared string, return as NON-shared anyref
    let g = b.addGlobal(kSharedAnyRef, true, true,
      [kExprRefNull, ...wasmSignedLeb(kSharedAnyRef.heap_type, 5)]);

    b.addFunction('set', makeSig([kSharedExternRef], []))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kExprGlobalSet, g.index
      ]).exportFunc();

    // Return as non-shared externref (extern.convert_any on shared value)
    b.addFunction('get_nonshared', makeSig([], [kWasmExternRef]))
      .addBody([
        kExprGlobalGet, g.index,
        kGCPrefix, kExprExternConvertAny
      ]).exportFunc();

    let inst = b.instantiate();
    inst.exports.set(sharedStr);

    let before = inst.exports.get_nonshared();
    print('TEST4a non-shared externref before tierup: isShared=' + %IsSharedString(before) +
          ((%IsSharedString(before)) ? ' *** DIRTY ***' : ' CLEAN'));

    tierUp(inst.exports.get_nonshared);
    let after = inst.exports.get_nonshared();
    print('TEST4b non-shared externref after tierup: isShared=' + %IsSharedString(after) +
          ((%IsSharedString(after)) ? ' *** DIRTY ***' : ' CLEAN'));

  } catch(e) { print('TEST4 ERROR: ' + e.message); }
})();

print('ALL TESTS COMPLETE');
```

## Evolution Plan
- v1: Run test. Check ALL paths for shared string leaks.
- v2: If JS API paths leak, audit wasm-js.cc `GetGlobalValue` and table accessor implementations.
- v3: If exception path leaks, audit exception payload extraction in the JS API.

## Task
**RUN THE TEST FIRST.** Save to `/tmp/shared_string_constants.js` and execute. Post ALL output. Key question: does the JS API (global.value, table.get, exception.getArg) unshare strings, or does it leak them like the compiled wrapper does?

Post with tags `shared-string-constant,js-api,global,table,exception,tc-003,W18`.
