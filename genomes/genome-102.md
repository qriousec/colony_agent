# Genome 102 — null-coercion-boundary

## Hypothesis
V8 uses THREE different null representations at the JS-Wasm boundary:
1. `WasmNull` — for ref types with `use_wasm_null()` (e.g., anyref, structref)
2. `JS null` — for non-use_wasm_null ref types
3. `JS undefined` — for externref defaults (wasm-js.cc:1480-1490)

When a function's declared return type is externref but the actual value flowing through is a null from an anyref path (WasmNull), the ToJS conversion must handle this correctly. If the conversion uses the DECLARED type to select the null representation but the ACTUAL value is WasmNull (from an anyref → extern_convert_any path), the conversion may produce the wrong null.

The compiled wrapper (wrappers-inl.h) checks `type.use_wasm_null()` to decide whether to convert WasmNull → JS null. If the type says "no wasm null" but the value IS WasmNull, the conversion skips the check and returns WasmNull directly to JS — where it's not a valid JS value.

## Task

### Step 1: Read This Code
- `src/wasm/wrappers-inl.h` — Read the `ToJS` function (around line 65-148). Look for how nullable references and WasmNull are handled. Specifically:
  - Line 136: `if (!type.is_nullable()) return ret;` — non-nullable: skip null check
  - Line 137: `if (!type.use_wasm_null()) return ret;` — non-wasm-null: skip conversion
  - Line 138: `if (type.is_none_type()) return null;` — none type: always null
  - Lines 142-147: nullable + use_wasm_null: convert WasmNull to JS null
- `src/builtins/js-to-wasm.tq` — Search for `WasmToJSObject` (around line 866-881). How does the Torque generic wrapper handle nulls?

### Step 2: Write and Run This Test
```javascript
// d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
// Test: Pass null through different ref type paths and check JS representation

const builder = new WasmModuleBuilder();

// Function returning null as externref
builder.addFunction('null_extern', makeSig([], [kWasmExternRef]))
  .addBody([kExprRefNull, kExternRefCode])
  .exportFunc();

// Function returning null as anyref
builder.addFunction('null_any', makeSig([], [kWasmAnyRef]))
  .addBody([kExprRefNull, kAnyRefCode])
  .exportFunc();

// Function returning null anyref converted to externref via extern.convert_any
builder.addFunction('null_any_to_extern', makeSig([], [kWasmExternRef]))
  .addBody([
    kExprRefNull, kAnyRefCode,
    kGCPrefix, kExprExternConvertAny,
  ])
  .exportFunc();

// Function returning null structref
builder.addFunction('null_struct', makeSig([], [kWasmStructRef]))
  .addBody([kExprRefNull, kStructRefCode])
  .exportFunc();

// Function returning null eqref
builder.addFunction('null_eq', makeSig([], [kWasmEqRef]))
  .addBody([kExprRefNull, kEqRefCode])
  .exportFunc();

const inst = builder.instantiate();

// Check each null representation
const results = {
  null_extern: inst.exports.null_extern(),
  null_any: inst.exports.null_any(),
  null_any_to_extern: inst.exports.null_any_to_extern(),
  null_struct: inst.exports.null_struct(),
  null_eq: inst.exports.null_eq(),
};

for (const [name, val] of Object.entries(results)) {
  print(name + ': value=' + val + ', typeof=' + typeof val +
    ', ===null:' + (val === null) + ', ===undefined:' + (val === undefined));
}

// Warmup to trigger tier-up
for (let i = 0; i < 100000; i++) {
  inst.exports.null_extern();
  inst.exports.null_any();
  inst.exports.null_any_to_extern();
  inst.exports.null_struct();
  inst.exports.null_eq();
}

print('--- After tier-up ---');
const results2 = {
  null_extern: inst.exports.null_extern(),
  null_any: inst.exports.null_any(),
  null_any_to_extern: inst.exports.null_any_to_extern(),
  null_struct: inst.exports.null_struct(),
  null_eq: inst.exports.null_eq(),
};

for (const [name, val] of Object.entries(results2)) {
  print(name + ': value=' + val + ', typeof=' + typeof val +
    ', ===null:' + (val === null) + ', ===undefined:' + (val === undefined));
}

// Check for tier differential
for (const name of Object.keys(results)) {
  if (results[name] !== results2[name]) {
    print('DIFFERENTIAL: ' + name + ' changed across tiers!');
    print('  Before: ' + results[name] + ' (' + typeof results[name] + ')');
    print('  After:  ' + results2[name] + ' (' + typeof results2[name] + ')');
  }
}
```

### Step 3: Run with Different Flags
```bash
d8 test.js                     # default
d8 --liftoff-only test.js      # Liftoff only
d8 --no-liftoff test.js        # Turboshaft only
```
Compare outputs. Any difference in null representation across tiers = bug.

### Step 4: If Differential Found
- Minimize the test to the specific type path that differs
- Identify which wrapper (Torque generic vs compiled Turboshaft) produces the wrong null
- Check if the wrong null representation causes crashes or type confusion in JS

### Step 5: If Clean, Escalate
- Test with `--experimental-wasm-shared` and shared ref types
- Test with imported functions (JS function returning null for different ref type expectations)
- Test with table.get returning null for different ref types

## Evolution Plan
- v1: Run the test above across all flag combinations (Steps 2-3)
- v2: If differential found, minimize. If clean, test shared ref type nulls.
