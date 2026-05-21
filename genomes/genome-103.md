# Genome 103 — custom-desc-interop

## Hypothesis
`--experimental-wasm-js-interop` enables JS-Wasm interop for custom descriptors. The interop prototype setup at `runtime-wasm.cc:1398-1597` casts externref array elements to JSReceiver. With JS interop enabled, additional paths exist where Wasm structs with custom descriptors are created from JS. The feature is brand new and test coverage (`custom-descriptors-interop.js`) may not cover adversarial inputs.

Specifically: when JS interop is enabled, `WasmJSInteropBuildDescriptorObject` (runtime-wasm.cc) creates JS objects that mirror Wasm struct fields as property descriptors. If the descriptor object's prototype chain is tampered with (e.g., via Proxy or modified getters), the field access path may read wrong offsets or types. The interop path also creates `WasmJSInteropAccessor` objects — if these accessors are extracted and applied to non-Wasm objects, the accessor's internal [[WasmStruct]] slot reference may be used on a non-struct, causing type confusion.

## Task

### Step 1: Read Source
- `src/runtime/runtime-wasm.cc` — Search for `JSInterop` or `InteropBuild` or `DescriptorObject`. Find how JS interop objects are created and what validation exists.
- `test/mjsunit/wasm/custom-descriptors-interop.js` — Read the ENTIRE file. Understand every test case. Identify what's NOT tested.
- `src/wasm/module-instantiate.cc` — Search for `custom_descriptor` or `js_interop`. Find the instantiation path.

### Step 2: Write Basic Interop Test
```javascript
// d8 --experimental-wasm-custom-descriptors --experimental-wasm-js-interop \
//    /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js

// Goal: Create custom descriptor structs via JS interop, then stress the
// accessor extraction and reapplication path.

const builder = new WasmModuleBuilder();

// Define a struct with custom descriptors
const desc_type = builder.addStruct([
  makeField(kWasmI32, true),   // field 0: "value"
  makeField(kWasmF64, true),   // field 1: "data"
]);

// Export a function that creates the struct
builder.addFunction('make', makeSig([kWasmI32, kWasmF64], [kWasmExternRef]))
  .addBody([
    kExprLocalGet, 0,
    kExprLocalGet, 1,
    kGCPrefix, kExprStructNew, desc_type,
    kGCPrefix, kExprExternConvertAny,
  ]).exportFunc();

// Export a function that reads field 0
builder.addFunction('get_value', makeSig([wasmRefType(desc_type)], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprStructGet, desc_type, 0,
  ]).exportFunc();

const inst = builder.instantiate();

// Create a struct via Wasm
const s = inst.exports.make(42, 3.14);
print('Created struct:', typeof s, s);

// Try to get property descriptors from the struct
const descs = Object.getOwnPropertyDescriptors(s);
print('Descriptors:', JSON.stringify(Object.keys(descs)));

// If descriptors have get/set accessors, extract them
for (const [key, desc] of Object.entries(descs)) {
  print('  ' + key + ':', JSON.stringify(desc));
  if (desc.get) {
    // Try calling the getter on a non-Wasm object
    try {
      const val = desc.get.call({});
      print('  getter on plain obj returned:', val);
    } catch (e) {
      print('  getter on plain obj threw:', e.message);
    }
    // Try calling the getter on null
    try {
      const val = desc.get.call(null);
      print('  getter on null returned:', val);
    } catch (e) {
      print('  getter on null threw:', e.message);
    }
    // Try calling the getter on another Wasm struct of different type
    const builder2 = new WasmModuleBuilder();
    const st2 = builder2.addStruct([makeField(kWasmI64, true)]);
    builder2.addFunction('make2', makeSig([], [kWasmExternRef]))
      .addBody([
        ...wasmI64Const(0xDEADBEEF),
        kGCPrefix, kExprStructNew, st2,
        kGCPrefix, kExprExternConvertAny,
      ]).exportFunc();
    const inst2 = builder2.instantiate();
    const wrong_struct = inst2.exports.make2();
    try {
      const val = desc.get.call(wrong_struct);
      print('  getter on wrong struct type returned:', val, '(TYPE CONFUSION?)');
    } catch (e) {
      print('  getter on wrong struct threw:', e.message);
    }
  }
}

// Test: Proxy wrapping a Wasm struct
try {
  const proxy = new Proxy(s, {
    get(target, prop) {
      print('  Proxy get:', prop);
      return Reflect.get(target, prop);
    }
  });
  print('Proxy value access:', proxy.value);
} catch (e) {
  print('Proxy threw:', e.message);
}
```

### Step 3: Run
```bash
d8 --experimental-wasm-custom-descriptors --experimental-wasm-js-interop \
   /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --experimental-wasm-custom-descriptors \
   /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
```

### Step 4: If Clean, Escalate — configureAll with Adversarial Prototypes
Study `/Users/t/v8/test/mjsunit/wasm/custom-descriptors-interop.js` and build a test that:
1. Creates a configureAll module with custom descriptors
2. Passes a Proxy as prototype in the configureAll array
3. Passes a WasmStruct (from different module) as prototype
4. Uses `Object.defineProperty` to redefine accessors on the JS interop object

### Step 5: Tier Differential
```bash
d8 --experimental-wasm-custom-descriptors --experimental-wasm-js-interop --liftoff-only test.js
d8 --experimental-wasm-custom-descriptors --experimental-wasm-js-interop --no-liftoff test.js
```

## Evolution Plan
- v1: Read custom-descriptors-interop.js, run basic test (Steps 1-3)
- v2: Accessor extraction + cross-type application
- v3: configureAll with adversarial prototypes (Proxy, wrong-type structs)
