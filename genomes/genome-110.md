# Genome 110 — write-barrier-array-gc

## Hypothesis
Commit `53cc35a6e9a` (Feb 2026, "Improve write-barrier treatment", 19 files, 232 insertions) changed when write barriers are skipped in BOTH Liftoff (`liftoff-compiler.cc`, 64 lines changed) and Turboshaft (`turboshaft-graph-interface.cc`, 94 lines changed).

Key changes:
- `ArrayNew` with non-shared ref elements: `kNoWriteBarrier` (array is freshly allocated in young space)
- `ArrayNewDefault`: always `kNoWriteBarrier` (default values are RO-space)
- `ArrayNewFixed`: `kNoWriteBarrier` for non-shared, `kFullWriteBarrier` for shared
- `StructSet`/`ArraySet` (explicit writes): always `kFullWriteBarrier` for ref fields
- Helper functions `FieldImmediateToWriteBarrier` and `ArrayIndexImmediateToWriteBarrier` always return `kFullWriteBarrier` for ref types

The risk: if Liftoff and Turboshaft disagree on barrier emission for any of these operations, a tier differential could cause:
- Liftoff: emits barrier → GC correctly tracks refs
- Turboshaft: skips barrier → GC misses ref → ref collected → dangling pointer

Also: for `ArrayNewFixed` with many elements, the initialization stores happen in a generated loop. If GC is triggered between allocation and the last element store, and the array has been tenured...

## Task

### Step 1: Read Source
- `src/wasm/turboshaft-graph-interface.cc` — Find `FieldImmediateToWriteBarrier` (around line 9285) and `ArrayIndexImmediateToWriteBarrier` (around line 9291). Verify they always return `kFullWriteBarrier` for ref types.
- `src/wasm/baseline/liftoff-compiler.cc` — Search for `write_barrier` or `WriteBarrier`. Find the corresponding Liftoff logic. Compare with Turboshaft.
- `src/compiler/turboshaft/wasm-lowering-reducer.h` — Find `ReduceStructSet` (around line 269) and `ReduceArraySet` (around line 358). Verify the write barrier parameter is used correctly.

### Step 2: Write Test — ArrayNewFixed with Ref Elements + GC Stress
```javascript
// d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js

const builder = new WasmModuleBuilder();
const st = builder.addStruct([makeField(kWasmI32, true)]);

// Array type: (array (ref null struct))
const arrType = builder.addArray(wasmRefNullType(st), true);

// Create struct helper
builder.addFunction('make_struct', makeSig([kWasmI32], [wasmRefType(st)]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprStructNew, st,
  ]).exportFunc();

// Create array with ref elements using array.new
builder.addFunction('make_array', makeSig([kWasmI32], [kWasmExternRef]))
  .addLocals(wasmRefType(st), 1)  // local 1: a struct
  .addBody([
    // Create a struct to use as initial value
    kExprLocalGet, 0,
    kGCPrefix, kExprStructNew, st,
    kExprLocalSet, 1,
    // Create array of 100 elements, all pointing to the same struct
    kExprLocalGet, 1,
    ...wasmI32Const(100),
    kGCPrefix, kExprArrayNew, arrType,
    kGCPrefix, kExprExternConvertAny,
  ]).exportFunc();

// Read element from array and return its i32 field
builder.addFunction('read_element', makeSig([kWasmExternRef, kWasmI32], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprAnyConvertExtern,
    kGCPrefix, kExprRefCast, arrType,
    kExprLocalGet, 1,  // index
    kGCPrefix, kExprArrayGet, arrType,
    kExprRefAsNonNull,
    kGCPrefix, kExprStructGet, st, 0,
  ]).exportFunc();

// Set element in array
builder.addFunction('set_element', makeSig([kWasmExternRef, kWasmI32, kWasmI32], []))
  .addLocals(wasmRefType(st), 1)  // local 3: new struct
  .addBody([
    // Create new struct with the given value
    kExprLocalGet, 2,
    kGCPrefix, kExprStructNew, st,
    kExprLocalSet, 3,
    // Set element
    kExprLocalGet, 0,
    kGCPrefix, kExprAnyConvertExtern,
    kGCPrefix, kExprRefCast, arrType,
    kExprLocalGet, 1,  // index
    kExprLocalGet, 3,  // new struct
    kGCPrefix, kExprArraySet, arrType,
  ]).exportFunc();

const inst = builder.instantiate();

// Phase 1: Create array, verify all elements readable
const arr = inst.exports.make_array(42);
for (let i = 0; i < 100; i++) {
  const v = inst.exports.read_element(arr, i);
  if (v !== 42) { print('BUG phase1: element ' + i + ' = ' + v); break; }
}
print('Phase 1 (initial array) OK');

// Phase 2: Modify elements, trigger GC, verify
for (let i = 0; i < 100; i++) {
  inst.exports.set_element(arr, i, i * 3);
}
gc();  // Force GC — should update all refs
for (let i = 0; i < 100; i++) {
  const v = inst.exports.read_element(arr, i);
  if (v !== i * 3) { print('BUG phase2: element ' + i + ' = ' + v + ' expected ' + (i*3)); break; }
}
print('Phase 2 (after set + GC) OK');

// Phase 3: Warmup for tier-up
for (let i = 0; i < 50000; i++) {
  const a = inst.exports.make_array(i);
  inst.exports.read_element(a, 0);
  inst.exports.set_element(a, 0, i + 1);
  inst.exports.read_element(a, 0);
}
print('Phase 3 (warmup) done');

// Phase 4: Post tier-up — same test as phase 2
const arr2 = inst.exports.make_array(99);
for (let i = 0; i < 100; i++) {
  inst.exports.set_element(arr2, i, i * 7);
}
gc();
for (let i = 0; i < 100; i++) {
  const v = inst.exports.read_element(arr2, i);
  if (v !== i * 7) { print('BUG phase4: element ' + i + ' = ' + v + ' expected ' + (i*7)); break; }
}
print('Phase 4 (post tier-up) OK');
```

### Step 3: Run with GC Stress
```bash
d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --gc-interval=50 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --stress-compaction /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --liftoff-only /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --no-liftoff /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --liftoff-only --gc-interval=50 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --no-liftoff --gc-interval=50 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
```

### Step 4: If Clean, Read Liftoff Write Barrier Code
```bash
grep -n 'write_barrier\|WriteBarrier\|kNoWriteBarrier\|kFullWriteBarrier' src/wasm/baseline/liftoff-compiler.cc | head -30
```
Compare Liftoff's barrier decisions with Turboshaft's (from `FieldImmediateToWriteBarrier` and `ArrayIndexImmediateToWriteBarrier`).

## Evolution Plan
- v1: Basic array ref element test + GC stress (Steps 2-3)
- v2: Read and compare Liftoff vs Turboshaft barrier logic (Step 4)
- v3: If divergence found, construct targeted test exposing the tier differential
