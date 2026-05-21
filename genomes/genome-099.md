# Genome 099 — globals-reftype-tier

## Hypothesis
The globals storage refactor (commit 0cfa6078eae, 20 files changed) split globals into FixedArray (ref-typed) and ByteArray (numeric). The interpreter fix (07e892290b6) confirmed this broke globals because "they can be moved by the GC." If Liftoff and Turboshaft disagree on buffer selection or offset for imported mutable ref globals, one tier could read from the wrong buffer.

## Task

### Step 1: Read These Files
- `src/wasm/wasm-module.h` — Find `WasmGlobal` struct, look for `index_in_buffer` field and `offset` field
- `src/wasm/baseline/liftoff-compiler.cc` — Search for `GetBufferAndOffsetForNumericGlobal` and `TaggedGlobalsBuffer` and `ImportedMutableGlobalsBuffers`
- `src/compiler/turboshaft/wasm-lowering-reducer.h` — Search for `GlobalGet` and `GlobalSet` to see how Turboshaft handles globals

Answer: Do Liftoff and Turboshaft use the same logic to decide FixedArray vs ByteArray? Do they compute the same offset?

### Step 2: Write Test (Cross-Module Imported Mutable Ref Global)
```javascript
// Load with: d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js thisfile.js
// The cross-module imported mutable global is the most complex path.

const struct_type_idx = 0; // will be defined per module

// Module A: exports a mutable ref global
const builderA = new WasmModuleBuilder();
const stA = builderA.addStruct([makeField(kWasmI32, true), makeField(kWasmI32, true)]);
const gA = builderA.addGlobal(wasmRefNullType(stA), true, false, [kExprRefNull, kNullRefCode]);
builderA.addFunction('set', makeSig([kWasmI32, kWasmI32], []))
  .addBody([
    kExprLocalGet, 0, kExprLocalGet, 1,
    kGCPrefix, kExprStructNew, stA,
    kExprGlobalSet, gA.index
  ]).exportFunc();
builderA.addFunction('get0', makeSig([], [kWasmI32]))
  .addBody([
    kExprGlobalGet, gA.index, kExprRefAsNonNull,
    kGCPrefix, kExprStructGet, stA, 0
  ]).exportFunc();
builderA.addFunction('get1', makeSig([], [kWasmI32]))
  .addBody([
    kExprGlobalGet, gA.index, kExprRefAsNonNull,
    kGCPrefix, kExprStructGet, stA, 1
  ]).exportFunc();
builderA.exportGlobal('g', gA.index, true);
const instA = builderA.instantiate();

// Module B: imports the mutable ref global
const builderB = new WasmModuleBuilder();
const stB = builderB.addStruct([makeField(kWasmI32, true), makeField(kWasmI32, true)]);
const gB = builderB.addImportedGlobal('m', 'g', wasmRefNullType(stB), true);
builderB.addFunction('read0', makeSig([], [kWasmI32]))
  .addBody([
    kExprGlobalGet, gB, kExprRefAsNonNull,
    kGCPrefix, kExprStructGet, stB, 0
  ]).exportFunc();
builderB.addFunction('read1', makeSig([], [kWasmI32]))
  .addBody([
    kExprGlobalGet, gB, kExprRefAsNonNull,
    kGCPrefix, kExprStructGet, stB, 1
  ]).exportFunc();
const instB = builderB.instantiate({m: {g: instA.exports.g}});

// Test: set from A, read from B
instA.exports.set(42, 99);
const r0 = instB.exports.read0();
const r1 = instB.exports.read1();
print('Cross-module ref global: ' + r0 + ', ' + r1);
if (r0 !== 42 || r1 !== 99) print('BUG: Wrong values!');

// Warmup to trigger tier-up in both modules
for (let i = 0; i < 100000; i++) {
  instA.exports.set(i, i * 2);
  instB.exports.read0();
  instB.exports.read1();
}
instA.exports.set(12345, 67890);
const t0 = instB.exports.read0();
const t1 = instB.exports.read1();
print('After tier-up: ' + t0 + ', ' + t1);
if (t0 !== 12345 || t1 !== 67890) print('BUG: Wrong values after tier-up!');
```

### Step 3: Run Differentially
```bash
d8 test.js                                    # default (Liftoff + tier-up)
d8 --liftoff-only test.js                     # Liftoff only
d8 --no-liftoff test.js                       # Turboshaft only
d8 --wasm-tier-mask-for-testing=1 test.js     # Function 0 Turboshaft, rest Liftoff
```
Compare outputs. Any difference = bug.

### Step 4: Add GC Stress
```bash
d8 --gc-interval=100 test.js                  # Force frequent GC
d8 --stress-compaction test.js                # Force compaction GC
```
If GC moves the FixedArray buffer and a tier doesn't reload, stale pointer.

## Evolution Plan
- v1: Source trace of Liftoff vs Turboshaft global access paths (Step 1)
- v2: Run differential tests (Steps 2-4)
- v3: If clean, test with mixed numeric + ref globals in same module (buffer index confusion)
