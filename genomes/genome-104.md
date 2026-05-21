# Genome 104 — inlining-castnarrow-tier

## Hypothesis
Wasm-in-Wasm inlining (`--wasm-inlining`, enabled by default) can inline a function containing `ref.cast`. When the inlined function narrows a type via `ref.cast`, the caller's subsequent field accesses use the narrowed type. If the inlining doesn't correctly propagate the cast check (e.g., the inlined cast is eliminated because the inlining reducer thinks the type is already narrow), the field access operates on the wrong type.

The specific concern: `wasm-gc-typed-optimization-reducer.cc` performs type narrowing after `ref.cast` by calling `RefineTypeKnowledge`. If a function `f(anyref x) { return ref.cast<StructA>(x).field0; }` is inlined into a caller that already knows `x` is `StructB <: StructA`, the optimizer may think the cast is redundant (StructB is a subtype of StructA, so cast always succeeds). But if the optimizer then uses StructB's field layout for the access when it should use StructA's, and StructB has additional fields or different offsets, the field access reads wrong memory.

This is a fresh angle on W3 (type check elimination in optimizer).

## Task

### Step 1: Read Source
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — Search for `RefineTypeKnowledge` and `ref.cast` handling. How does it narrow types? Does it use the CAST target type or the KNOWN type?
- `src/compiler/turboshaft/wasm-in-js-inlining-reducer-inl.h` — Search for `InlineWasm` or `Inline`. How are inlined function types propagated to the caller's type state?
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — Search for `StructGet` and `StructSet`. After inlining + cast, what type does it use for field access?

### Step 2: Write Test — Inlined Cast with Subtype Field Access
```javascript
// d8 --wasm-inlining /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js

const builder = new WasmModuleBuilder();

// Parent struct: 1 i32 field
const parent = builder.addStruct([makeField(kWasmI32, true)]);

// Child struct: inherits i32 field, adds f64 field
const child = builder.addStruct([makeField(kWasmI32, true), makeField(kWasmF64, true)], parent);

// Function that casts to parent and reads field 0
// This function should be inlinable
const castAndRead = builder.addFunction('cast_read',
    makeSig([kWasmAnyRef], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprRefCast, parent,
    kGCPrefix, kExprStructGet, parent, 0,
  ]);

// Main: creates a child, passes to cast_read (which gets inlined)
// The optimizer knows the arg is a child (subtype of parent)
// After inlining, does it use child's layout or parent's for field 0?
builder.addFunction('main', makeSig([kWasmI32], [kWasmI32]))
  .addBody([
    // Create child struct with field0=arg, field1=3.14
    kExprLocalGet, 0,
    ...wasmF64Const(3.14),
    kGCPrefix, kExprStructNew, child,
    // Call cast_read (will be inlined)
    kExprCallFunction, castAndRead.index,
  ]).exportFunc();

// Second main: creates a PARENT (not child), passes to cast_read
// This tests that the cast actually works when the type is exact parent
builder.addFunction('main_parent', makeSig([kWasmI32], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprStructNew, parent,
    kExprCallFunction, castAndRead.index,
  ]).exportFunc();

const inst = builder.instantiate();

// Warmup — always pass child, optimizer specializes
for (let i = 0; i < 100000; i++) {
  const r = inst.exports.main(i);
  if (r !== i) { print('BUG main: expected ' + i + ' got ' + r); break; }
}
print('main warmup OK');

// Warmup main_parent
for (let i = 0; i < 100000; i++) {
  const r = inst.exports.main_parent(i);
  if (r !== i) { print('BUG main_parent: expected ' + i + ' got ' + r); break; }
}
print('main_parent warmup OK');

// Cross-check: Liftoff result vs optimized result
const check1 = inst.exports.main(99999);
const check2 = inst.exports.main_parent(99999);
print('Final check: main=' + check1 + ' main_parent=' + check2);
if (check1 !== 99999 || check2 !== 99999) print('BUG: wrong result after optimization');
```

### Step 3: Run Differentially
```bash
d8 --wasm-inlining /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --no-wasm-inlining /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --liftoff-only /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --no-liftoff /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
```

### Step 4: If Clean, Escalate — Deeper Subtype Chain
- Add a grandchild struct (3 levels deep)
- Inline a function that casts to the middle type
- Pass the grandchild — optimizer knows it's grandchild, cast targets middle
- The field access after inlined cast must use the correct layout

### Step 5: If Still Clean — Abstract Type Cast + Inlining
- Inline a function that does `ref.cast<anyref>` or `ref.cast<eqref>` (abstract types)
- After inlining, the type is narrowed to the abstract type
- Does StructGet after the cast use the concrete type knowledge from the caller?

## Evolution Plan
- v1: Source read + basic inlining test (Steps 1-3)
- v2: Deep subtype chain test (Step 4)
- v3: Abstract type cast + inlining (Step 5)
