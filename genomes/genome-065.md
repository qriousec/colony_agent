# Genome 065 — custom-desc-exact-cast

## Hypothesis
Custom descriptors (--experimental-wasm-custom-descriptors) introduce exact type qualifiers. A `ref.cast` to an exact type uses `kExactMatchOnly` (map pointer equality), while a cast to a non-exact type uses `kMayBeSubtype` (RTT chain walk). The optimizer (`WasmGCTypedOptimizationReducer`) can eliminate casts when `TypeCheckAlwaysSucceeds`.

If the optimizer's type inference for exact types loses the exactness qualifier during phi merges, inlining, or other transformations, it could:
1. Treat an exact type as non-exact → allow subtype where only exact match is valid
2. Eliminate a cast based on wrong exactness → skip the map equality check

`ref.get_desc` requires an exact receiver — it reads the descriptor field from the Map. If the cast to exact is eliminated and the actual object has a different (subtype) map, `ref.get_desc` reads from the wrong map offset → type confusion.

Multi-factor: [exact type qualifier] × [optimizer type inference] × [ref.get_desc descriptor access] = type confusion.

## DNA

### Source Facts
```cpp
// wasm-gc-typed-optimization-reducer.h — IsCastToCustomDescriptor guard
// Lines 21-26: Returns true if cast target has custom descriptor
// These casts should NOT be eliminated by TypeCheckAlwaysSucceeds

// wasm-compiler-definitions.cc:21-39 — GetExactness
// For descriptor types: returns kExactMatchLastSupertype
// For final types: returns kExactMatchOnly

// turboshaft-graph-interface.cc:5554 — RefCastDescEq uses kExactMatchOnly
// turboshaft-graph-interface.cc:5540 — RefCast uses GetExactness

// Previous workers confirmed: the exactness gap between inlining and
// standalone code is NOT exploitable because LoadImmediateSuperRTT returns
// the same value as direct map comparison for canonical types.
// BUT: nobody tested with ACTUAL custom descriptors (ref.get_desc).
```

### Files to Audit
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — IsCastToCustomDescriptor, cast elimination
- `src/wasm/function-body-decoder-impl.h` — ref.get_desc validation, exact type requirements
- `src/compiler/turboshaft/wasm-lowering-reducer.h` — How kExactMatchOnly is lowered to machine code
- `src/wasm/wasm-feature-flags.h` — custom_descriptors feature flag

### Key Questions
1. After a phi merge of exact and non-exact types, what does the optimizer think the result type is?
2. Can ref.get_desc be reached after a phi merge that lost exactness?
3. Does the IsCastToCustomDescriptor guard prevent ALL cast elimination paths, or just some?
4. What happens if a loop body narrows to exact type, but the phi at the loop header widens to non-exact?

### Related
- desc-exactness (genome ~017) found wasm-inlining-into-js.cc:300 vs GetExactness inconsistency, but confirmed NOT exploitable for final types
- custom-desc-cast (genome ~032) tested IsCastToCustomDescriptor across 8 iterations, all CLEAN
- The configureAll fix (a58753bac7b) showed Liftoff stack confusion — different bug pattern

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-custom-descriptors --experimental-wasm-gc --experimental-wasm-js-interop --allow-natives-syntax --turboshaft-wasm
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

// TEST 1: Basic custom descriptor creation and access
(function TestBasicDescriptor() {
  try {
    let b = new WasmModuleBuilder();

    // Define struct with descriptor
    let baseStruct = b.addStruct([makeField(kWasmI32, true)]);

    // Try to create a type with custom descriptor
    // Check if the module builder supports descriptor types
    print('TEST1: Checking custom descriptor support...');

    // Simple struct operations first
    b.addFunction('make', makeSig([], [wasmRefType(baseStruct)]))
      .addBody([
        kExprI32Const, 42,
        kGCPrefix, kExprStructNew, baseStruct,
      ]).exportFunc();

    b.addFunction('get', makeSig([wasmRefType(baseStruct)], [kWasmI32]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprStructGet, baseStruct, 0,
      ]).exportFunc();

    let inst = b.instantiate();
    let obj = inst.exports.make();
    print('TEST1: Basic struct = ' + inst.exports.get(obj));

  } catch(e) {
    print('TEST1 ERROR: ' + e.message);
  }
})();

// TEST 2: Cast to exact type in loop with alternating inputs
(function TestExactCastInLoop() {
  try {
    let b = new WasmModuleBuilder();

    // Two struct types in same rec group
    let stA = b.addStruct([makeField(kWasmI32, true)]);
    let stB = b.addStruct([makeField(kWasmI32, true), makeField(kWasmI64, true)], stA);

    // Loop that casts to stB (more specific)
    b.addFunction('loop_cast', makeSig([kWasmAnyRef, kWasmI32], [kWasmI32]))
      .addLocals(kWasmI32, 1)
      .addBody([
        kExprLoop, kWasmVoid,
          kExprLocalGet, 0,
          kGCPrefix, kExprRefCast, stB,
          kGCPrefix, kExprStructGet, stB, 0,
          kExprLocalGet, 2,
          kExprI32Add,
          kExprLocalSet, 2,
          kExprLocalGet, 1,
          kExprI32Const, 1,
          kExprI32Sub,
          kExprLocalTee, 1,
          kExprI32Const, 0,
          kExprI32GtS,
          kExprBrIf, 0,
        kExprEnd,
        kExprLocalGet, 2,
      ]).exportFunc();

    b.addFunction('make_b', makeSig([kWasmI32], [wasmRefType(stB)]))
      .addBody([
        kExprLocalGet, 0,
        kExprI64Const, 99,
        kGCPrefix, kExprStructNew, stB,
      ]).exportFunc();

    b.addFunction('make_a', makeSig([kWasmI32], [wasmRefType(stA)]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprStructNew, stA,
      ]).exportFunc();

    let inst = b.instantiate();
    let objB = inst.exports.make_b(42);

    // Tier up with stB (should succeed)
    for (let i = 0; i < 20000; i++) {
      inst.exports.loop_cast(objB, 1);
    }
    print('TEST2: Tiered up result = ' + inst.exports.loop_cast(objB, 5));

    // Now pass stA (should throw, stA is NOT subtype of stB)
    let objA = inst.exports.make_a(99);
    try {
      let r = inst.exports.loop_cast(objA, 1);
      print('TEST2: *** TYPE CONFUSION *** got ' + r + ' instead of throw');
    } catch(e) {
      print('TEST2: Correctly threw: ' + e.message);
    }

  } catch(e) {
    print('TEST2 ERROR: ' + e.message);
  }
})();

print('ALL TESTS COMPLETE');
```

## DO NOT Try These
- desc-exactness inlining gap (confirmed dead, semantically equivalent)
- IsCastToCustomDescriptor guard alone (8 iterations by custom-desc-cast, CLEAN)
- configureAll fast path (different bug pattern, fix in place)

## Evolution Plan
- v1: Check if custom descriptors are usable in current build. Source audit WasmGCTypedOptimizationReducer for how it handles exact types in phi merges. Check ref.get_desc lowering.
- v2: If custom descriptors work, construct test with ref.get_desc after a phi merge that might lose exactness. If not available, audit the code paths for when the feature is enabled.

## Task
First check if `--experimental-wasm-custom-descriptors` is a valid flag in the build. Search for existing custom descriptor tests:
```bash
find /Users/t/v8/test/mjsunit/wasm/ -name "*custom*desc*" -o -name "*descriptor*"
```

Then source audit `wasm-gc-typed-optimization-reducer.h` for how exact types interact with phi merges and cast elimination.

Post with tags `custom-desc,exact-type,ref-get-desc,cast-elimination,W20`.
