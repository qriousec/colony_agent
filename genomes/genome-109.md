# Genome 109 — i31-extern-smi-cast

## Hypothesis
Commit `d86212a39be` ("Fix missing Smi checks since crrev.com/c/7054524") fixed 4 locations where `config.from.AsNonShared() == wasm::kWasmExternRef` was too strict — it only matched the exact `externref` type, missing `(ref null extern)` and `(ref extern)`. The fix changed to `config.from.is_reference_to(wasm::GenericKind::kExtern)`.

The missing Smi check means: when casting from an extern type, i31 values (which are Smis on the heap level) were not detected as Smis. The cast would treat the Smi as a HeapObject and read from the Smi value as if it were a pointer — reading invalid memory.

The fix touched 4 locations across `wasm-lowering-reducer.h` and `wasm-gc-lowering.cc`. Search for remaining locations with similar strict equality patterns that might have the same bug.

## Task

### Step 1: Read Source — Verify Fix and Search for Siblings
- `src/compiler/turboshaft/wasm-lowering-reducer.h` lines 700-800 — Read `ReduceWasmTypeCheck` and `ReduceWasmTypeCast`. Find the fixed Smi check. Look for other type comparisons using equality.
- `src/compiler/wasm-gc-lowering.cc` lines 200-400 — Same functions in the old TurboFan pipeline.
- Search the entire `src/compiler/turboshaft/` directory for patterns: `== wasm::kWasm` or `AsNonShared() ==` or `AsNullable() ==` — any remaining strict equality comparisons for Wasm types.

```bash
grep -rn 'AsNonShared() ==' src/compiler/turboshaft/ src/compiler/wasm-gc-lowering.cc src/wasm/
grep -rn '== wasm::kWasm' src/compiler/turboshaft/wasm-lowering-reducer.h
grep -rn 'is_reference_to' src/compiler/turboshaft/wasm-lowering-reducer.h
```

### Step 2: Write Test — i31 Through Extern Ref Paths
```javascript
// d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
// Test: i31 values flowing through extern paths with casts

const builder = new WasmModuleBuilder();

// Function: take externref, cast to (ref i31), return as i32
builder.addFunction('extern_to_i31', makeSig([kWasmExternRef], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprAnyConvertExtern,  // extern -> any
    kGCPrefix, kExprRefCast, kI31RefCode,  // any -> (ref i31)
    kGCPrefix, kExprI31GetS,  // (ref i31) -> i32
  ]).exportFunc();

// Function: take anyref, test if i31, return 1/0
builder.addFunction('test_i31', makeSig([kWasmAnyRef], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprRefTest, kI31RefCode,
  ]).exportFunc();

// Function: create i31, convert to extern, return
builder.addFunction('i31_to_extern', makeSig([kWasmI32], [kWasmExternRef]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprRefI31,  // i32 -> (ref i31)
    kGCPrefix, kExprExternConvertAny,  // any -> extern
  ]).exportFunc();

// Function: roundtrip i31 through extern
builder.addFunction('i31_roundtrip', makeSig([kWasmI32], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprRefI31,           // i32 -> (ref i31)
    kGCPrefix, kExprExternConvertAny, // -> externref
    kGCPrefix, kExprAnyConvertExtern, // -> anyref
    kGCPrefix, kExprRefCast, kI31RefCode,  // -> (ref i31)
    kGCPrefix, kExprI31GetS,          // -> i32
  ]).exportFunc();

const inst = builder.instantiate();

// Phase 1: basic roundtrip
for (const val of [0, 1, -1, 42, -42, 1073741823, -1073741824]) {
  const r = inst.exports.i31_roundtrip(val);
  if (r !== val) { print('BUG roundtrip: ' + val + ' -> ' + r); }
}
print('Phase 1 (basic roundtrip) OK');

// Phase 2: i31 created in Wasm, sent to JS as externref, sent back
for (const val of [0, 1, -1, 42, 999999, -999999]) {
  const ext = inst.exports.i31_to_extern(val);
  const back = inst.exports.extern_to_i31(ext);
  if (back !== val) { print('BUG extern roundtrip: ' + val + ' -> ' + back); }
}
print('Phase 2 (extern roundtrip) OK');

// Phase 3: warmup to trigger tier-up
for (let i = 0; i < 100000; i++) {
  inst.exports.i31_roundtrip(i & 0x3FFFFFFF);
  inst.exports.i31_to_extern(i & 0x3FFFFFFF);
}

// Phase 4: post tier-up — same tests
for (const val of [0, 1, -1, 42, -42, 1073741823, -1073741824]) {
  const r = inst.exports.i31_roundtrip(val);
  if (r !== val) { print('BUG post-tierup roundtrip: ' + val + ' -> ' + r); }
}

for (const val of [0, 1, -1, 42, 999999, -999999]) {
  const ext = inst.exports.i31_to_extern(val);
  const back = inst.exports.extern_to_i31(ext);
  if (back !== val) { print('BUG post-tierup extern: ' + val + ' -> ' + back); }
}
print('Phase 4 (post tier-up) OK');

// Phase 5: test_i31 with various types
const ext_i31 = inst.exports.i31_to_extern(42);
print('test_i31(i31): ' + inst.exports.test_i31(ext_i31));  // expect 1... but wait, test_i31 takes anyref, not externref

// Phase 6: Cross-module i31 passing
const builder2 = new WasmModuleBuilder();
builder2.addFunction('make_i31', makeSig([kWasmI32], [kWasmExternRef]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprRefI31,
    kGCPrefix, kExprExternConvertAny,
  ]).exportFunc();
const inst2 = builder2.instantiate();

// i31 from module 2, cast in module 1
const cross = inst2.exports.make_i31(12345);
const cross_result = inst.exports.extern_to_i31(cross);
print('Cross-module i31: ' + cross_result);
if (cross_result !== 12345) print('BUG: cross-module i31 = ' + cross_result);
```

### Step 3: Run Differentially
```bash
d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --liftoff-only /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --no-liftoff /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --gc-interval=100 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
```

### Step 4: If Clean, Search for Unfixed Siblings
Search ALL of `src/compiler/` and `src/wasm/` for remaining strict type equality comparisons:
```bash
grep -rn 'AsNonShared() ==' src/compiler/ src/wasm/ | grep -v test | grep -v '.pyc'
grep -rn 'AsNullable() ==' src/compiler/ src/wasm/ | grep -v test | grep -v '.pyc'
grep -rn '== wasm::kWasmExternRef\|== wasm::kWasmAnyRef\|== wasm::kWasmEqRef' src/compiler/ src/wasm/
```
For each match, check if it should use `is_reference_to()` instead.

## Evolution Plan
- v1: Source search for remaining strict equality patterns + basic i31 test (Steps 1-3)
- v2: Report any remaining patterns found (Step 4)
- v3: If remaining patterns found, construct targeted test to trigger them
