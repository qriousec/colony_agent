# Genome 101 — configureall-proto-type

## Hypothesis
The configureAll prototype setup in `runtime-wasm.cc:1398-1597` reads externref arrays as prototype sources and casts elements to JSObject. The fast path fix (a58753bac7b) only fixed SIGNATURE validation — it didn't add TYPE validation for the array CONTENTS. If a Wasm module provides a WasmStruct, WasmArray, or i31 value in the externref prototype array, the runtime may cast it to JSReceiver incorrectly.

The key code path: `NextPrototypeInternal()` at `runtime-wasm.cc:1945-1947` reads from a FixedArray (externref elements) and returns them as prototype objects. These are then used in `SetupPrototypes` which expects JSReceiver objects. If a non-JSReceiver passes through, it creates a type confusion in the prototype chain.

## Task

### Step 1: Read Source (BRIEF)
Read these specific locations:
- `src/runtime/runtime-wasm.cc` — Search for `configureAll` or `ConfigureAllPrototypes` or `SetupPrototypes`. Find where the prototype array elements are read and how they're validated.
- `src/wasm/baseline/liftoff-compiler.cc` — Search for `PrototypeSetupSequenceDetector` around line 452-615. Understand what bytecode pattern triggers the fast path.
- `test/mjsunit/wasm/custom-descriptors.js` — Read existing tests to understand the API.

### Step 2: Study Existing Tests
```bash
# Find custom descriptor test files
find /Users/t/v8/test/mjsunit/wasm -name "*custom*" -o -name "*configureAll*" -o -name "*descriptor*" | head -10
# Read the most relevant one
cat /Users/t/v8/test/mjsunit/wasm/custom-descriptors.js | head -100
```

### Step 3: Write Test
```javascript
// d8 --experimental-wasm-custom-descriptors --experimental-wasm-js-interop \
//    /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
//
// Goal: Put a WasmStruct into the prototype externref array
// and see if configureAll crashes or produces type confusion.

// First, understand the configureAll API:
// 1. Module exports a configureAll function
// 2. configureAll takes arrays of prototypes and functions
// 3. It sets up prototype chains for Wasm struct instances

// Strategy: Create a module with custom descriptors.
// Provide a WasmStruct where a JS object prototype is expected.

// NOTE: This test requires understanding the custom descriptor API.
// Start by reading the existing tests at:
// /Users/t/v8/test/mjsunit/wasm/custom-descriptors.js

print('Starting configureAll type confusion test');

// Step 1: Create a helper module that produces WasmStruct objects
const helperBuilder = new WasmModuleBuilder();
const st = helperBuilder.addStruct([makeField(kWasmI32, true)]);
helperBuilder.addFunction('make_struct', makeSig([kWasmI32], [kWasmExternRef]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprStructNew, st,
    kGCPrefix, kExprExternConvertAny,  // Convert to externref
  ]).exportFunc();
const helper = helperBuilder.instantiate();

// Step 2: Get a WasmStruct as an externref
const wasmStruct = helper.exports.make_struct(42);
print('Got WasmStruct as externref:', typeof wasmStruct);

// Step 3: Try to use it as a prototype
// This tests the JS API path, not configureAll directly
try {
  const obj = Object.create(wasmStruct);
  print('Object.create succeeded (unexpected):', obj);
} catch (e) {
  print('Object.create threw:', e.message);
}

// Step 4: Try to use it in WeakSet/WeakMap (TC-001 pattern)
try {
  const ws = new WeakSet();
  ws.add(wasmStruct);
  print('WeakSet.add succeeded');
} catch (e) {
  print('WeakSet.add threw:', e.message);
}

// Step 5: Try to set it as __proto__
try {
  const obj = {};
  Object.setPrototypeOf(obj, wasmStruct);
  print('setPrototypeOf succeeded (unexpected)');
} catch (e) {
  print('setPrototypeOf threw:', e.message);
}
```

### Step 4: Run
```bash
d8 --experimental-wasm-custom-descriptors --experimental-wasm-js-interop test.js
d8 --experimental-wasm-custom-descriptors test.js
d8 test.js  # without custom descriptors
```

### Step 5: If The Above Is Clean, Try configureAll Directly
Study `/Users/t/v8/test/mjsunit/wasm/custom-descriptors.js` for the full configureAll pattern, then modify it to inject WasmStruct/WasmArray objects where prototypes are expected.

## Evolution Plan
- v1: Run the basic tests above (Steps 2-4), read existing custom descriptor tests
- v2: Construct a proper configureAll module with WasmStruct in the prototype array
- v3: Test with different non-JSReceiver types (WasmArray, i31, null variants)
