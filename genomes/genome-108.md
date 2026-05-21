# Genome 108 — br-on-cast-fallthrough

## Hypothesis
Commit `dad3b1c03cf` ("Fix missing type propagation on br_on_cast_fail fallthrough") fixed a DCHECK crash where br_on_cast_fail's fallthrough type was not annotated. **But the commit message explicitly says the type NARROWING on the fallthrough path is NOT implemented:**

> "There is also a missed optimization as for br_on_cast the analyzer correctly handles the branch on a condition that is WasmTypeCheckOp but for br_on_cast_fail the condition is an equality ComparisonOp of a WasmTypeCheckOp and a Word32Constant(0) which it does not use to propagate the type check's narrowing. This is not addressed in this CL."

This means: for `br_on_cast`, the optimizer narrows the type on the taken branch. For `br_on_cast_fail`, the optimizer does NOT narrow the type on the fallthrough (successful cast) path because the condition is `ComparisonOp(WasmTypeCheckOp, Word32Constant(0))` instead of a bare `WasmTypeCheckOp`. The type narrowing in the type reducer (`wasm-gc-typed-optimization-reducer.cc:444-485`) only handles bare `WasmTypeCheckOp` conditions, not the negated comparison pattern.

This is currently a missed optimization — but if other passes (load elimination, inlining, dead code elimination) ASSUME the type is narrowed after a successful br_on_cast_fail fallthrough, the missing narrowing creates a gap where the optimizer uses the wrong (wider) type, potentially failing to eliminate a cast that should succeed or — worse — eliminating a different cast based on stale type info from a different path.

All flags used are DEFAULT — no experimental flags needed.

## Task

### Step 1: Read Source
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` lines 430-500 — Find how `BranchOp` conditions handle `WasmTypeCheckOp` vs `ComparisonOp(WasmTypeCheckOp, Word32Constant(0))`. Confirm the asymmetry described in dad3b1c03cf.
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — Find `WasmTypeCast` reduction — does it check known types from the type narrowing? If narrowing is missing for br_on_cast_fail, does WasmTypeCast still eliminate casts?
- `src/wasm/turboshaft-graph-interface.cc` around line 7772 — Read the fixed code from dad3b1c03cf. The `AnnotateWasmType` call annotates the static type but doesn't feed into the type reducer's narrowing.
- Commit `dad3b1c03cf` diff — already read, confirms the gap.

### Step 2: Write Test — br_on_cast Fallthrough Type Confusion
```javascript
// d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js

const builder = new WasmModuleBuilder();

// Three struct types: parent, childA, childB
const parent = builder.addStruct([makeField(kWasmI32, true)]);  // field 0: tag
const childA = builder.addStruct([
  makeField(kWasmI32, true),    // field 0: tag (inherited)
  makeField(kWasmI32, true),    // field 1: childA-specific
], parent);
const childB = builder.addStruct([
  makeField(kWasmI32, true),    // field 0: tag (inherited)
  makeField(kWasmF64, true),    // field 1: childB-specific (DIFFERENT TYPE than childA!)
], parent);

// Function: classify — takes anyref, returns field values based on type
// Uses br_on_cast to check childA first, then falls through to check childB
builder.addFunction('classify', makeSig([kWasmAnyRef], [kWasmI32]))
  .addBody([
    // Block $exit (result i32)
    kExprBlock, kWasmI32,
      // Block $not_a (anyref passthrough)
      kExprBlock, kWasmVoid,
        kExprLocalGet, 0,
        // br_on_cast to $not_a if NOT childA (fallthrough = IS childA)
        // Wait — br_on_cast branches if cast SUCCEEDS
        // br_on_cast_fail branches if cast FAILS
        // Use br_on_cast: if input IS childA, branch to inner block
        // Actually let's use the simpler pattern:

        // Try cast to childA
        kGCPrefix, kExprBrOnCastFail, 0b01, 0, kAnyRefCode, childA,
        // Cast to childA SUCCEEDED — read childA.field1
        kGCPrefix, kExprStructGet, childA, 1,
        kExprBr, 1,  // br $exit with childA.field1 (i32)
      kExprEnd,
      // Fallthrough: NOT childA. Try childB.
      // The value on stack should still be the original anyref
      // But after br_on_cast_fail, what's left on stack?

      // Actually, br_on_cast_fail leaves the ORIGINAL value on fallthrough
      // (with potentially narrowed type — this is the key question)

      // Try cast to childB
      kExprBlock, kWasmVoid,
        kExprLocalGet, 0,
        kGCPrefix, kExprBrOnCastFail, 0b01, 0, kAnyRefCode, childB,
        // Cast to childB SUCCEEDED — read field1 as f64, convert to i32
        kGCPrefix, kExprStructGet, childB, 1,
        kExprI32TruncSatF64S,
        kExprBr, 2,  // br $exit
      kExprEnd,

      // Neither childA nor childB — return -1
      ...wasmI32Const(-1),
    kExprEnd,
  ]).exportFunc();

// Helpers to create each type
builder.addFunction('make_parent', makeSig([kWasmI32], [kWasmExternRef]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprStructNew, parent,
    kGCPrefix, kExprExternConvertAny,
  ]).exportFunc();

builder.addFunction('make_childA', makeSig([kWasmI32, kWasmI32], [kWasmExternRef]))
  .addBody([
    kExprLocalGet, 0,
    kExprLocalGet, 1,
    kGCPrefix, kExprStructNew, childA,
    kGCPrefix, kExprExternConvertAny,
  ]).exportFunc();

builder.addFunction('make_childB', makeSig([kWasmI32, kWasmF64], [kWasmExternRef]))
  .addBody([
    kExprLocalGet, 0,
    kExprLocalGet, 1,
    kGCPrefix, kExprStructNew, childB,
    kGCPrefix, kExprExternConvertAny,
  ]).exportFunc();

const inst = builder.instantiate();

// Phase 1: basic correctness
const a = inst.exports.make_childA(1, 100);
const b = inst.exports.make_childB(2, 200.5);
const p = inst.exports.make_parent(3);

print('childA: ' + inst.exports.classify(a));  // expect 100
print('childB: ' + inst.exports.classify(b));  // expect 200
print('parent: ' + inst.exports.classify(p));  // expect -1

// Phase 2: warmup with ONLY childA — optimizer specializes for childA path
for (let i = 0; i < 100000; i++) {
  const obj = inst.exports.make_childA(1, i);
  const r = inst.exports.classify(obj);
  if (r !== i) { print('BUG warmup: expected ' + i + ' got ' + r); break; }
}
print('Warmup done (all childA)');

// Phase 3: now pass childB — the fallthrough path fires
// If optimizer incorrectly narrowed fallthrough type, this may read wrong fields
const b2 = inst.exports.make_childB(2, 42.0);
const r_b = inst.exports.classify(b2);
print('After tier-up, childB: ' + r_b);
if (r_b !== 42) print('BUG: expected 42 got ' + r_b + ' — fallthrough type confusion!');

// Phase 4: pass parent — neither branch taken
const p2 = inst.exports.make_parent(3);
const r_p = inst.exports.classify(p2);
print('After tier-up, parent: ' + r_p);
if (r_p !== -1) print('BUG: expected -1 got ' + r_p);

// Phase 5: pass null
const r_null = inst.exports.classify(null);
print('Null: ' + r_null);
if (r_null !== -1) print('BUG: expected -1 for null, got ' + r_null);
```

### Step 3: Run Differentially
```bash
d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --liftoff-only /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --no-liftoff /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --no-wasm-inlining /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
```
Compare outputs across all flag combos. Any difference = bug.

### Step 4: If Clean, Escalate — Three-Way Cast Chain
- Add childC (third sibling)
- Chain: br_on_cast<childA> → br_on_cast<childB> → br_on_cast<childC> → fallthrough
- Each fallthrough narrows further. Test with childC after warmup with childA.

### Step 5: If Still Clean — br_on_cast in Loop + Fallthrough
- Combine with loop: loop body does br_on_cast, taken continues loop, fallthrough exits
- Pass different types on different iterations
- The loop phi must track the fallthrough type correctly across iterations

## Evolution Plan
- v1: Source read + basic br_on_cast fallthrough test (Steps 1-3)
- v2: Three-way cast chain (Step 4)
- v3: Loop + br_on_cast combination (Step 5)
