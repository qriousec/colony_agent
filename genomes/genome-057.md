# Genome 057 — func-sig-subtype-dispatch

## Hypothesis
W11: Wasm GC function signature subtyping. Wasm GC allows function type subtyping (contravariant parameters, covariant returns). When `call_ref` dispatches through a typed function reference, the callee's actual signature may differ from the call site's expected signature. The wrapper/trampoline must handle this correctly.

**Specific concern:** If a function type `$ft_sub` is a subtype of `$ft_base` (sub has more specific return, less specific params), and a `call_ref` expects `$ft_base` but receives a `(ref $ft_sub)` function, does the call site:
1. Use the caller's expected signature for stack layout?
2. Or the callee's actual signature?

If the call site uses the caller's signature but the callee writes return values according to its own signature, a mismatch could occur.

**Second concern:** `table.set` with a function of subtype signature into a table typed with supertype signature, then `call_indirect` which dispatches by table signature. If `call_indirect` uses the table's declared element type for the call but the actual function has a different (but subtype-compatible) signature, return value handling may differ.

Multi-factor: [function signature subtyping] × [call_ref/call_indirect dispatch] × [return value layout mismatch] = type confusion.

## DNA

### Source Files to Audit
- `src/wasm/wasm-subtyping.cc` — function type subtyping rules
- `src/compiler/turboshaft/wasm-lowering-reducer.h` — call_ref lowering
- `src/wasm/function-body-decoder-impl.h` — call_ref/call_indirect validation
- `src/wasm/module-instantiate.cc` — table element type checking
- `src/wasm/wasm-code-manager.h` — wrapper dispatch

### Key Mechanism
```wasm
(type $base (func (param anyref) (result anyref)))
(type $sub (sub $base (func (param anyref) (result (ref struct $S)))))
;; $sub is subtype of $base: more specific return type

(table 1 (ref null $base))  ;; table typed with $base
(elem ... (ref.func $f_sub))  ;; store $sub-typed function

;; call_indirect with $base signature — callee is $sub
;; Does the return value get treated as anyref or (ref struct $S)?
;; If the compiler inlined based on $base but callee returns $S layout...
```

### Related Patterns
- P3: Type check elimination from wrong inference — if the compiler knows the function is $sub but the call expects $base, it may narrow the return type incorrectly
- CVE-2024-2887: Wrong type inference led to cast elimination

## Ready-to-Run Test

```javascript
// Flags: --allow-natives-syntax --experimental-wasm-gc --turboshaft-wasm
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

let b = new WasmModuleBuilder();

// Base function type: (anyref) -> anyref
let ftBase = b.addType(makeSig([kWasmAnyRef], [kWasmAnyRef]));

// Struct types with different layouts
let structSmall = b.addStruct([makeField(kWasmI32, true)]);
let structBig = b.addStruct([makeField(kWasmI32, true), makeField(kWasmI64, true), makeField(kWasmF64, true)]);

// Sub function type: (anyref) -> (ref structSmall)  [more specific return]
let ftSub = b.addType(makeSig([kWasmAnyRef], [wasmRefType(structSmall)]), ftBase);

// Function with $sub signature — returns structSmall
b.addFunction('func_sub', ftSub)
  .addBody([
    kExprI32Const, 42,
    kGCPrefix, kExprStructNew, structSmall
  ]).exportFunc();

// Function with $base signature — returns its input
b.addFunction('func_base', ftBase)
  .addBody([
    kExprLocalGet, 0
  ]).exportFunc();

// Table typed with (ref null $base)
let table = b.addTable(wasmRefNullType(ftBase), 2);
b.addActiveElementSegment(table.index, wasmI32Const(0),
  [[kExprRefFunc, 0], [kExprRefFunc, 1]], wasmRefType(ftBase));

// call_ref with $base type
b.addFunction('call_via_ref', makeSig([wasmRefType(ftBase), kWasmAnyRef], [kWasmAnyRef]))
  .addBody([
    kExprLocalGet, 1,
    kExprLocalGet, 0,
    kExprCallRef, ftBase,
  ]).exportFunc();

// call_indirect with $base signature
b.addFunction('call_via_table', makeSig([kWasmI32, kWasmAnyRef], [kWasmAnyRef]))
  .addBody([
    kExprLocalGet, 1,
    kExprLocalGet, 0,
    kExprCallIndirect, ftBase, table.index,
  ]).exportFunc();

// Multi-return variant
let ftMultiBase = b.addType(makeSig([kWasmAnyRef], [kWasmAnyRef, kWasmI32]));
let ftMultiSub = b.addType(makeSig([kWasmAnyRef], [wasmRefType(structSmall), kWasmI32]), ftMultiBase);

b.addFunction('func_multi_sub', ftMultiSub)
  .addBody([
    kExprI32Const, 99,
    kGCPrefix, kExprStructNew, structSmall,
    kExprI32Const, 7,
  ]).exportFunc();

let table2 = b.addTable(wasmRefNullType(ftMultiBase), 1);
b.addActiveElementSegment(table2.index, wasmI32Const(0),
  [[kExprRefFunc, 4]], wasmRefType(ftMultiBase));

b.addFunction('call_multi_indirect', makeSig([kWasmAnyRef], [kWasmAnyRef, kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kExprI32Const, 0,
    kExprCallIndirect, ftMultiBase, table2.index,
  ]).exportFunc();

let inst;
try {
  inst = b.instantiate();
} catch(e) {
  print('INSTANTIATION ERROR: ' + e.message);
  // May need to adjust type definitions
  throw e;
}

// Get function refs
let funcSub = inst.exports.func_sub;
let funcBase = inst.exports.func_base;

// Make a structBig to pass as input
let b2 = new WasmModuleBuilder();
let sBig = b2.addStruct([makeField(kWasmI32, true), makeField(kWasmI64, true), makeField(kWasmF64, true)]);
b2.addFunction('make_big', makeSig([], [kWasmAnyRef]))
  .addBody([
    kExprI32Const, 0xDEAD,
    kExprI64Const, 0x42,
    ...wasmF64Const(3.14),
    kGCPrefix, kExprStructNew, sBig
  ]).exportFunc();
let inst2 = b2.instantiate();
let bigObj = inst2.exports.make_big();

// TEST 1: call_ref with sub function through base signature
print('TEST 1: call_ref $base with $sub function');
for (let i = 0; i < 20000; i++) {
  inst.exports.call_via_ref(inst.exports.func_sub, bigObj);
}
let r1 = inst.exports.call_via_ref(inst.exports.func_sub, bigObj);
print('  result type: ' + typeof r1 + ', value: ' + r1);

// TEST 2: call_indirect with sub function in base-typed table
print('TEST 2: call_indirect $base, table entry is $sub function');
for (let i = 0; i < 20000; i++) {
  inst.exports.call_via_table(0, bigObj);
}
let r2 = inst.exports.call_via_table(0, bigObj);
print('  result type: ' + typeof r2 + ', value: ' + r2);

// TEST 3: Compare call_ref results for base vs sub
print('TEST 3: Compare base vs sub function results');
let rBase = inst.exports.call_via_ref(inst.exports.func_base, bigObj);
let rSub = inst.exports.call_via_ref(inst.exports.func_sub, bigObj);
print('  base result: ' + typeof rBase);
print('  sub result: ' + typeof rSub);

// TEST 4: Multi-return through subtyped indirect call
print('TEST 4: Multi-return call_indirect with subtyped function');
try {
  for (let i = 0; i < 20000; i++) {
    inst.exports.call_multi_indirect(bigObj);
  }
  let [ref, num] = inst.exports.call_multi_indirect(bigObj);
  print('  ref: ' + ref + ', num: ' + num);
} catch(e) {
  print('  ERROR: ' + e.message);
}

print('ALL TESTS COMPLETE');
```

## Evolution Plan
- v1: Run the test. Check for type confusion in return values.
- v2: Audit wasm-lowering-reducer.h for how call_ref/call_indirect handle signature subtyping in the compiled code.
- v3: Test with deeply nested subtype chains (>3 levels deep) and with ref.cast on call_ref results.

## Task
Run the test. Check if results are consistent across tiers. Also audit the call_ref lowering in Turboshaft to see how it handles signature variance.

Post with tags `func-sig,subtyping,call_ref,call_indirect,dispatch,W11`.
