# Genome 107 — loop-phi-cast-widen

## Hypothesis
ProcessPhi at loop backedges widens types via `Union` (`wasm-gc-typed-optimization-reducer.cc:415`). If a loop body contains `br_on_cast` where only the taken branch (narrow type) continues the loop, the backedge always carries the narrow type. `Union(wide_forward_edge, narrow_backedge)` should stay wide. But if `CreateMergeSnapshot` (`wasm-gc-typed-optimization-reducer.cc:534-582`) incorrectly caches the narrow type from a previous fixed-point iteration's `RefineTypeKnowledge`, a `ref.cast` inside the loop could be eliminated on re-analysis — allowing wrong-type struct field access when the forward edge carries a wider type.

The key edge case: a loop where the FIRST iteration receives `anyref` (from the forward edge) but ALL subsequent iterations receive `StructA` (from the backedge via br_on_cast). If the analyzer's fixed-point converges with the narrow type and eliminates the cast, the first iteration accesses `anyref` as `StructA` without a cast — reading garbage fields.

All flags used are DEFAULT (--wasm-inlining, WasmGC, Turboshaft optimization are all on by default).

## Task

### Step 1: Read Source
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` lines 388-427 — `ProcessPhi`: how loop phi types are computed via Union
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` lines 534-582 — `CreateMergeSnapshot`: how merge snapshots handle type widening
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` lines 18-83 — Loop backedge reprocessing: `needs_revisit` logic and fixed-point iteration
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` lines 251-321 — `WasmTypeCast` reduction: when casts are eliminated based on `IsHeapSubtypeOf`

### Step 2: Write Test — Loop with br_on_cast + StructGet
```javascript
// d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
// All default flags — no experimental flags needed

const builder = new WasmModuleBuilder();

// Two struct types: parent and child
const parent = builder.addStruct([makeField(kWasmI32, true)]);
const child = builder.addStruct([makeField(kWasmI32, true), makeField(kWasmF64, true)], parent);

// Helper: create a parent struct
builder.addFunction('make_parent', makeSig([kWasmI32], [wasmRefType(parent)]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprStructNew, parent,
  ]).exportFunc();

// Helper: create a child struct
builder.addFunction('make_child', makeSig([kWasmI32, kWasmF64], [wasmRefType(child)]))
  .addBody([
    kExprLocalGet, 0,
    kExprLocalGet, 1,
    kGCPrefix, kExprStructNew, child,
  ]).exportFunc();

// Main function: loop that receives anyref, casts to parent, reads field 0
// The loop continues only if cast succeeds (br_on_cast taken = continue)
// This means the backedge always carries parent type
// But the FIRST iteration may receive something that IS a parent but the
// optimizer shouldn't assume all iterations are parent
builder.addFunction('loop_cast_read',
    makeSig([kWasmAnyRef, kWasmI32], [kWasmI32]))
  .addLocals(kWasmI32, 1)       // local 2: accumulator
  .addLocals(wasmRefNullType(parent), 1) // local 3: cast result
  .addBody([
    // accumulator = 0
    ...wasmI32Const(0),
    kExprLocalSet, 2,

    // loop $L
    kExprLoop, kWasmVoid,
      // Try to cast local 0 (anyref) to parent
      kExprBlock, kWasmVoid,
        kExprLocalGet, 0,
        // br_on_cast_fail: if cast fails, break out of block (exit loop)
        kGCPrefix, kExprBrOnCastFail, 0b01, 0, kAnyRefCode, parent,
        // Cast succeeded — we have (ref parent) on stack
        kExprLocalSet, 3,
        // Read field 0
        kExprLocalGet, 3,
        kGCPrefix, kExprStructGet, parent, 0,
        // Add to accumulator
        kExprLocalGet, 2,
        kExprI32Add,
        kExprLocalSet, 2,
        // Decrement counter
        kExprLocalGet, 1,
        ...wasmI32Const(1),
        kExprI32Sub,
        kExprLocalSet, 1,
        // If counter > 0, continue loop
        kExprLocalGet, 1,
        ...wasmI32Const(0),
        kExprI32GtS,
        kExprBrIf, 1,  // br to loop
      kExprEnd,  // end block
    kExprEnd,  // end loop

    kExprLocalGet, 2,
  ]).exportFunc();

const inst = builder.instantiate();

// Test with child struct (subtype of parent) — should work
const c = inst.exports.make_child(10, 3.14);
const r1 = inst.exports.loop_cast_read(c, 5);
print('Child struct, 5 iters: ' + r1);
if (r1 !== 50) print('BUG: expected 50 got ' + r1);

// Test with parent struct — should also work
const p = inst.exports.make_parent(7);
const r2 = inst.exports.loop_cast_read(p, 3);
print('Parent struct, 3 iters: ' + r2);
if (r2 !== 21) print('BUG: expected 21 got ' + r2);

// Warmup to trigger tier-up — always use child
for (let i = 0; i < 100000; i++) {
  inst.exports.loop_cast_read(inst.exports.make_child(1, 0.0), 1);
}
print('Warmup done (all child)');

// NOW test with parent — optimizer may have specialized for child
const r3 = inst.exports.loop_cast_read(inst.exports.make_parent(42), 1);
print('After tier-up with parent: ' + r3);
if (r3 !== 42) print('BUG: expected 42 got ' + r3 + ' — type confusion!');

// Test with a non-struct (should exit loop immediately, return 0)
const r4 = inst.exports.loop_cast_read(null, 5);
print('Null input: ' + r4);
if (r4 !== 0) print('BUG: expected 0 for null input, got ' + r4);
```

### Step 3: Run Differentially
```bash
d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --liftoff-only /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --no-liftoff /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --no-wasm-inlining /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
```

### Step 4: If Clean, Escalate — Nested Loops + Multiple Types
- Add a third struct type (grandchild) with 3 fields
- Inner loop casts to child, outer loop casts to parent
- Pass grandchild during warmup, then parent after tier-up
- The optimizer's phi widening must handle nested loop type states correctly

### Step 5: If Still Clean — br_on_cast + StructGet of CHILD fields
- After br_on_cast to parent succeeds, do a SECOND cast to child inside the loop
- If the optimizer eliminates the second cast because it thinks the value is already child (from backedge specialization), accessing child.field1 (f64) on a parent struct reads past the struct boundary

## Evolution Plan
- v1: Run the basic loop cast test (Steps 2-3)
- v2: Nested loops + multiple types (Step 4)
- v3: Double cast inside loop — parent then child (Step 5)
