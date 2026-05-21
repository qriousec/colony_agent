# Genome 066 — loop-unroll-type-narrow

## Hypothesis
Loop unrolling (recently fixed by d829f3e2f68 for non-commutative increment ops) duplicates loop bodies. The `WasmGCTypedOptimizationReducer` performs type-based cast elimination AFTER unrolling. If the optimizer's type snapshot from the original loop body carries over to the unrolled copy, it could eliminate casts in the unrolled iteration that are still needed.

Scenario:
1. Loop body contains `ref.cast T` where T is a specific struct type
2. After the cast, the optimizer knows the value has type T
3. Loop unrolling creates a second copy of the body
4. In the second copy, the cast to T is preceded by the first copy's "known type T" → optimizer may think cast always succeeds
5. But the second iteration might receive a DIFFERENT value from the loop's update logic

The parent genome (056, loop-header-type-narrowing) found the loop header phi handling is sound for the basic case. This genome targets the UNROLLED case: does unrolling preserve correct type knowledge boundaries?

Multi-factor: [loop unrolling] × [type knowledge carry-over between unrolled copies] × [ref.cast elimination] = type confusion in unrolled iteration.

## DNA

### Source Facts
```cpp
// loop-unrolling-reducer.h — Loop unrolling in Turboshaft
// Duplicates loop body N times
// Question: does it reset type snapshots between copies?

// wasm-gc-typed-optimization-reducer.h — TypeCheckAlwaysSucceeds
// Uses TypeSnapshotTable for type knowledge at each program point
// Question: after unrolling, does each copy get fresh type knowledge?

// d829f3e2f68 — Fixed loop unrolling for non-commutative ops
// The increment operation was incorrectly transformed after unrolling
// This shows the unroller has had recent bugs

// Parent genome 056 findings:
// - Loop header phi uses fixed-point widening (correct)
// - First evaluation uses forward edge only (conservative)
// - ProcessPhi correctly unions all inputs
// - BUT: unrolling was not tested
```

### Files to Audit
- `src/compiler/turboshaft/loop-unrolling-reducer.h` — How loop bodies are duplicated
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — Type snapshot handling during unrolling
- `src/compiler/turboshaft/loop-peeling-reducer.h` — Loop peeling (related to unrolling)
- `src/compiler/turboshaft/phase.h` — Pass ordering: does type optimization run before or after unrolling?

### Key Questions
1. What is the pass ordering? Does WasmGCTypedOptimization run before or after loop unrolling?
2. If type optimization runs AFTER unrolling, does it see the unrolled copies as straight-line code? If so, type knowledge from copy 1 flows into copy 2.
3. If type optimization runs BEFORE unrolling, the casts aren't eliminated yet when unrolling happens. After unrolling, does type optimization run again?
4. Does the loop unroller insert phi nodes at boundaries between copies?

### Related
- Parent genome 056 (loop-header-type-narrowing): 1 iter, found basic mechanism sound
- CVE-2024-2887: Type check elimination from wrong inference (same general pattern)

## Ready-to-Run Test

```javascript
// Flags: --allow-natives-syntax --experimental-wasm-gc --turboshaft-wasm
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

let b = new WasmModuleBuilder();

// Two struct types with different field layouts
let stA = b.addStruct([makeField(kWasmI32, true)]);
let stB = b.addStruct([makeField(kWasmF64, true)]);

// Function with a tight loop that does ref.cast on each iteration
// The loop is designed to be unrollable (simple counter, known bounds)
b.addFunction('unroll_cast', makeSig([kWasmAnyRef, kWasmI32], [kWasmI32]))
  .addLocals(kWasmI32, 1)  // local 2: accumulator
  .addBody([
    kExprLoop, kWasmVoid,
      // Cast to stA — should this be eliminated after unrolling?
      kExprLocalGet, 0,
      kGCPrefix, kExprRefCast, stA,
      kGCPrefix, kExprStructGet, stA, 0,
      kExprLocalGet, 2,
      kExprI32Add,
      kExprLocalSet, 2,
      // Counter
      kExprLocalGet, 1,
      kExprI32Const, 1,
      kExprI32Sub,
      kExprLocalTee, 1,
      kExprBrIf, 0,
    kExprEnd,
    kExprLocalGet, 2,
  ]).exportFunc();

// Factory functions
b.addFunction('make_a', makeSig([kWasmI32], [wasmRefType(stA)]))
  .addBody([kExprLocalGet, 0, kGCPrefix, kExprStructNew, stA]).exportFunc();

b.addFunction('make_b', makeSig([], [wasmRefType(stB)]))
  .addBody([...wasmF64Const(3.14), kGCPrefix, kExprStructNew, stB]).exportFunc();

let inst = b.instantiate();

// Phase 1: Train with stA for unrolling (high iteration count)
let a = inst.exports.make_a(7);
for (let i = 0; i < 50000; i++) {
  inst.exports.unroll_cast(a, 4);  // Small loop body, likely to be unrolled
}
print('Phase 1: Trained, result=' + inst.exports.unroll_cast(a, 4));

// Phase 2: Pass stB — should throw
try {
  let bObj = inst.exports.make_b();
  let r = inst.exports.unroll_cast(bObj, 1);
  print('Phase 2: *** TYPE CONFUSION *** result=' + r + ' (should have thrown!)');
} catch(e) {
  print('Phase 2: Correctly threw: ' + e.message);
}

// Phase 3: Variant with loop that alternates objects
b = new WasmModuleBuilder();
let sX = b.addStruct([makeField(kWasmI32, true)]);
let sY = b.addStruct([makeField(kWasmI64, true)]);

// Loop takes array of anyref, casts each to sX
b.addFunction('array_cast_loop', makeSig([kWasmAnyRef, kWasmAnyRef, kWasmI32], [kWasmI32]))
  .addLocals(kWasmI32, 1)
  .addBody([
    kExprLoop, kWasmVoid,
      // Even iterations: cast param 0 to sX
      // Odd iterations: cast param 1 to sX
      kExprLocalGet, 2,
      kExprI32Const, 1,
      kExprI32And,
      kExprIf, kWasmI32,
        kExprLocalGet, 0,
        kGCPrefix, kExprRefCast, sX,
        kGCPrefix, kExprStructGet, sX, 0,
      kExprElse,
        kExprLocalGet, 1,
        kGCPrefix, kExprRefCast, sX,
        kGCPrefix, kExprStructGet, sX, 0,
      kExprEnd,
      kExprLocalGet, 3,
      kExprI32Add,
      kExprLocalSet, 3,
      kExprLocalGet, 2,
      kExprI32Const, 1,
      kExprI32Sub,
      kExprLocalTee, 2,
      kExprBrIf, 0,
    kExprEnd,
    kExprLocalGet, 3,
  ]).exportFunc();

b.addFunction('mk_x', makeSig([kWasmI32], [wasmRefType(sX)]))
  .addBody([kExprLocalGet, 0, kGCPrefix, kExprStructNew, sX]).exportFunc();

let inst2 = b.instantiate();
let x1 = inst2.exports.mk_x(10);
let x2 = inst2.exports.mk_x(20);

// Tier up with correct types
for (let i = 0; i < 50000; i++) {
  inst2.exports.array_cast_loop(x1, x2, 8);
}
print('Phase 3: Trained, result=' + inst2.exports.array_cast_loop(x1, x2, 8));

print('ALL TESTS COMPLETE');
```

## DO NOT Try These
- Basic loop header phi handling (parent found it sound)
- Simple ref.cast correctness without unrolling
- Cross-module type sharing (confirmed dead)

## Evolution Plan
- v1: Source audit pass ordering. Run test to check for type confusion after unrolling.
- v2: If clean, construct test with explicit unroll-friendly patterns (counted loop, small body). Audit loop-unrolling-reducer.h for how it handles operations with side effects (traps from ref.cast).

## Task
First determine PASS ORDERING:
```bash
grep -n "LoopUnrolling\|WasmGCTyped\|WasmGCType" /Users/t/v8/src/compiler/turboshaft/phase.h
```

This tells us if type optimization runs before/after unrolling. Then audit loop-unrolling-reducer.h for how it handles ref.cast operations.

Post with tags `loop-unroll,type-narrow,cast-elimination,turboshaft,W7,unrolling`.
