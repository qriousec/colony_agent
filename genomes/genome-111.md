# Genome 111 — exception-gcref-tier

## Hypothesis
Wasm exception handling is now shipped (feature flag `--experimental-wasm-exnref` removed in `c794d17cf00`). When a Wasm exception carries a WasmGC struct ref as payload, the ref must survive:
1. The throw path (struct ref stored in exception object)
2. JS catch (if the exception crosses the JS boundary)
3. Rethrow back to Wasm (struct ref recovered from exception)
4. GC during any of these steps (struct ref must be updated)

The exception payload is stored in a `FixedArray`. If Turboshaft's compiled throw/catch path stores or recovers the struct ref differently than Liftoff's, there's a tier differential. Additionally, if the exception crosses the JS boundary, the `ToJS` path must handle the GC ref correctly — this is a boundary crossing.

## Task

### Step 1: Read Source
- `src/wasm/baseline/liftoff-compiler.cc` — Search for `kExprThrow` and `kExprTryCatch`. How does Liftoff handle exception payloads?
- `src/wasm/turboshaft-graph-interface.cc` — Search for `Throw` and `TryCatch`. How does Turboshaft handle exception payloads?
- `src/wasm/wasm-objects.h` — Search for `WasmExceptionPackage`. How is the exception payload stored?

### Step 2: Write Test — Exception with Struct Ref Payload
```javascript
// d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js

const builder = new WasmModuleBuilder();

// Struct type
const st = builder.addStruct([makeField(kWasmI32, true), makeField(kWasmI32, true)]);

// Exception tag with struct ref payload
const tag = builder.addTag(makeSig([wasmRefType(st)], []));

// Function: throw exception with struct ref
builder.addFunction('thrower', makeSig([kWasmI32, kWasmI32], []))
  .addBody([
    kExprLocalGet, 0,
    kExprLocalGet, 1,
    kGCPrefix, kExprStructNew, st,
    kExprThrow, tag,
  ]).exportFunc();

// Function: try-catch, return struct field from caught exception
builder.addFunction('catcher', makeSig([kWasmI32, kWasmI32], [kWasmI32]))
  .addBody([
    kExprTry, kWasmI32,
      kExprLocalGet, 0,
      kExprLocalGet, 1,
      kGCPrefix, kExprStructNew, st,
      kExprThrow, tag,
      ...wasmI32Const(-1),  // unreachable but needed for type
    kExprCatch, tag,
      // Exception payload: (ref st) on stack
      kGCPrefix, kExprStructGet, st, 0,
    kExprEnd,
  ]).exportFunc();

// Function: try-catch, return SECOND field
builder.addFunction('catcher_f1', makeSig([kWasmI32, kWasmI32], [kWasmI32]))
  .addBody([
    kExprTry, kWasmI32,
      kExprLocalGet, 0,
      kExprLocalGet, 1,
      kGCPrefix, kExprStructNew, st,
      kExprThrow, tag,
      ...wasmI32Const(-1),
    kExprCatch, tag,
      kGCPrefix, kExprStructGet, st, 1,
    kExprEnd,
  ]).exportFunc();

// Export the tag for JS interop
builder.addExportOfKind('tag', kExternalTag, tag);

const inst = builder.instantiate();

// Phase 1: Basic throw-catch within Wasm
const r1 = inst.exports.catcher(42, 99);
print('Phase 1 field0: ' + r1);
if (r1 !== 42) print('BUG: expected 42 got ' + r1);

const r1b = inst.exports.catcher_f1(42, 99);
print('Phase 1 field1: ' + r1b);
if (r1b !== 99) print('BUG: expected 99 got ' + r1b);

// Phase 2: Exception crossing JS boundary
try {
  inst.exports.thrower(123, 456);
  print('BUG: thrower should have thrown');
} catch (e) {
  print('JS caught exception: ' + e);
  print('  typeof: ' + typeof e);
  print('  instanceof WebAssembly.Exception: ' + (e instanceof WebAssembly.Exception));
  if (e instanceof WebAssembly.Exception) {
    // Try to read the payload via the tag
    try {
      const payload = e.getArg(inst.exports.tag, 0);
      print('  payload: ' + payload + ' (typeof: ' + typeof payload + ')');
    } catch (e2) {
      print('  getArg threw: ' + e2.message);
    }
  }
}

// Phase 3: Warmup for tier-up
for (let i = 0; i < 50000; i++) {
  const v = inst.exports.catcher(i, i + 1);
  if (v !== i) { print('BUG warmup: expected ' + i + ' got ' + v); break; }
}
print('Phase 3 (warmup) done');

// Phase 4: Post tier-up
const r4a = inst.exports.catcher(77777, 88888);
const r4b = inst.exports.catcher_f1(77777, 88888);
print('Phase 4 field0: ' + r4a + ' field1: ' + r4b);
if (r4a !== 77777) print('BUG post-tierup: field0 expected 77777 got ' + r4a);
if (r4b !== 88888) print('BUG post-tierup: field1 expected 88888 got ' + r4b);

// Phase 5: GC stress — exception + GC
gc();
const r5a = inst.exports.catcher(11111, 22222);
gc();
const r5b = inst.exports.catcher_f1(11111, 22222);
print('Phase 5 (post-GC): ' + r5a + ', ' + r5b);
if (r5a !== 11111 || r5b !== 22222) print('BUG post-GC!');

// Phase 6: Exception from Liftoff caught in Turboshaft (or vice versa)
// Use two functions: one that throws (stays in Liftoff if --no-wasm-inlining),
// one that catches (tier-up'd to Turboshaft)
print('All phases complete');
```

### Step 3: Run Differentially
```bash
d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --liftoff-only /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --no-liftoff /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --gc-interval=50 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --stress-compaction /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
```

### Step 4: If Clean, Escalate — Multiple GC Type Payloads
- Exception tag with MULTIPLE struct ref payloads of different types
- Exception tag with array ref payload
- Nested try-catch: inner throws, outer catches, modifies struct, rethrows
- Cross-module exception: Module A throws, Module B catches

## Evolution Plan
- v1: Basic exception + struct ref test across tiers (Steps 2-3)
- v2: JS boundary crossing — getArg/setArg with struct refs
- v3: Multiple payloads + nested rethrow
