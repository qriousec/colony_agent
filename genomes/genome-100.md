# Genome 100 — deopt-structref-resume

## Hypothesis
Wasm deoptimization (--wasm-deopt, enabled by default) deopts from Turboshaft back to Liftoff. When the deopt frame contains WasmGC struct references, the frame description must correctly tag them so GC can visit them during deopt. If a struct ref in the deopt frame is misidentified as a raw integer, GC won't update it when moving the struct — resumed execution follows a dangling pointer.

## Task

### Step 1: Read These Files (BRIEF — just find the key mechanisms)
- `src/wasm/wasm-deopt-data.h` — Find `WasmDeoptEntry` and `LiftoffFrameDescriptionForDeopt` — how are stack slots described?
- `src/wasm/turboshaft-graph-interface.cc` — Search for `CreateFrameState` — how are GC refs encoded in the frame state?
- `src/deoptimizer/deoptimizer.cc` — Search for `Wasm` — how does the deoptimizer handle Wasm frames?

### Step 2: Write and Run This Test
```javascript
// d8 --wasm-deopt /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
// Goal: trigger deopt while struct refs are on the stack

const builder = new WasmModuleBuilder();
const st = builder.addStruct([
  makeField(kWasmI32, true),   // field 0
  makeField(kWasmI32, true),   // field 1
]);

// A polymorphic call target via table
const sig = builder.addType(makeSig([kWasmI32], [kWasmI32]));
const table = builder.addTable(kWasmFuncRef, 2, 2);

// Function 0: simple add
const fn_add = builder.addFunction('fn_add', sig)
  .addBody([kExprLocalGet, 0, ...wasmI32Const(1), kExprI32Add]);

// Function 1: multiply
const fn_mul = builder.addFunction('fn_mul', sig)
  .addBody([kExprLocalGet, 0, ...wasmI32Const(2), kExprI32Mul]);

builder.addActiveElementSegment(table.index, wasmI32Const(0),
  [[kExprRefFunc, fn_add.index], [kExprRefFunc, fn_mul.index]],
  kWasmFuncRef);

// Main function: creates struct, calls through table (can deopt), reads struct
builder.addFunction('main', makeSig([kWasmI32, kWasmI32], [kWasmI32]))
  .addLocals(wasmRefNullType(st), 1)  // local 2 = struct ref
  .addBody([
    // Create struct with field0=arg0, field1=arg1
    kExprLocalGet, 0,
    kExprLocalGet, 1,
    kGCPrefix, kExprStructNew, st,
    kExprLocalSet, 2,
    // Call through table — optimizer will speculate on target
    // This call may trigger deopt if table entry changes
    kExprLocalGet, 0,
    kExprLocalGet, 1, // table index (0 or 1)
    kExprCallIndirect, sig, table.index,
    kExprDrop,
    // After potential deopt: read struct fields
    kExprLocalGet, 2,
    kExprRefAsNonNull,
    kGCPrefix, kExprStructGet, st, 0,
  ])
  .exportFunc();

const inst = builder.instantiate();

// Phase 1: warmup with table[0] (fn_add) — optimizer specializes
for (let i = 0; i < 50000; i++) {
  const r = inst.exports.main(i, 0);
  if (r !== i) { print('BUG phase1: expected ' + i + ' got ' + r); break; }
}
print('Phase 1 (warmup) OK');

// Phase 2: switch to table[1] (fn_mul) — trigger deopt
// After deopt, the struct local should still be valid
for (let i = 0; i < 1000; i++) {
  const r = inst.exports.main(i, 1);
  if (r !== i) { print('BUG phase2: expected ' + i + ' got ' + r); break; }
}
print('Phase 2 (deopt) OK');

// Phase 3: GC stress — force collection while struct is in deopt frame
gc();  // trigger GC
for (let i = 0; i < 10000; i++) {
  const r = inst.exports.main(i, 0);
  if (r !== i) { print('BUG phase3: expected ' + i + ' got ' + r); break; }
}
print('Phase 3 (post-GC) OK');
```

### Step 3: Run Differentially
```bash
d8 --wasm-deopt test.js                           # default
d8 --wasm-deopt --gc-interval=50 test.js          # GC stress
d8 --wasm-deopt --stress-compaction test.js        # compaction stress
d8 --no-wasm-deopt test.js                        # no deopt (baseline)
d8 --liftoff-only test.js                         # no optimization (baseline)
```

### Step 4: If Clean, Escalate
- Add more struct refs to the stack (multiple locals of different struct types)
- Add nested function calls with struct refs at multiple stack depths
- Test with nullable refs (some null, some non-null) in the deopt frame
- Test with array refs and i31 refs

## Evolution Plan
- v1: Run the test above, report results for all flag combinations
- v2: If clean, add more complex scenarios (multiple struct types, nested calls)
- v3: If still clean, source trace the deopt frame construction for GC ref handling
