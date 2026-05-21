# Genome 112 — arraycopy-subtype-ref

## Hypothesis
`array.copy` between arrays with subtype-compatible element types must:
1. Validate source element type is subtype of dest element type
2. Use correct element size for the copy
3. Emit write barriers when copying ref elements

The Turboshaft implementation in `turboshaft-graph-interface.cc` generates an element-by-element copy loop for ref arrays (not a bulk memcpy). The copy loop at lines ~5310-5350 reads from source array using `ArrayGet` and writes to dest array using `ArraySet`. If the loop uses the SOURCE array type for `ArrayGet` but a different barrier decision for `ArraySet` (based on dest type), there could be a mismatch.

Additionally: if source array has element type `(ref childStruct)` and dest array has element type `(ref null parentStruct)`, the copy should work (child <: parent). But does the copy read with childStruct layout and write with parentStruct layout? Since ref types are all pointer-sized this shouldn't matter for refs, but the WRITE BARRIER decision depends on whether the element is a ref — if the dest array type's `is_ref()` check fails for some reason, the barrier is skipped.

## Task

### Step 1: Read Source
- `src/wasm/turboshaft-graph-interface.cc` — Search for `ArrayCopy`. Find the copy loop implementation. Check:
  - Which array type is used for `ArrayGet`? Source or dest?
  - Which array type is used for `ArraySet`? Source or dest?
  - Which type is used for the write barrier decision?
  - What happens when source and dest arrays have DIFFERENT element types (subtype relation)?
- `src/wasm/baseline/liftoff-compiler.cc` — Search for `ArrayCopy`. How does Liftoff implement it?

### Step 2: Write Test — array.copy with Subtyped Ref Elements
```javascript
// d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js

const builder = new WasmModuleBuilder();

// Parent and child struct types
const parent = builder.addStruct([makeField(kWasmI32, true)]);
const child = builder.addStruct([makeField(kWasmI32, true), makeField(kWasmF64, true)], parent);

// Array types with different element types
const parentArr = builder.addArray(wasmRefNullType(parent), true);
const childArr = builder.addArray(wasmRefNullType(child), true);

// Create array of child structs
builder.addFunction('make_child_array', makeSig([kWasmI32], [kWasmExternRef]))
  .addBody([
    // Create a child struct as initial value
    kExprLocalGet, 0,
    ...wasmF64Const(3.14),
    kGCPrefix, kExprStructNew, child,
    // Create array of 10 elements
    ...wasmI32Const(10),
    kGCPrefix, kExprArrayNew, childArr,
    kGCPrefix, kExprExternConvertAny,
  ]).exportFunc();

// Create empty parent array (filled with null)
builder.addFunction('make_parent_array', makeSig([], [kWasmExternRef]))
  .addBody([
    kExprRefNull, kNullRefCode,
    ...wasmI32Const(10),
    kGCPrefix, kExprArrayNew, parentArr,
    kGCPrefix, kExprExternConvertAny,
  ]).exportFunc();

// Copy from child array to parent array
// This should work because (ref null child) <: (ref null parent)
builder.addFunction('copy_child_to_parent',
    makeSig([kWasmExternRef, kWasmExternRef, kWasmI32, kWasmI32, kWasmI32], []))
  .addBody([
    // dst (parent array)
    kExprLocalGet, 0,
    kGCPrefix, kExprAnyConvertExtern,
    kGCPrefix, kExprRefCast, parentArr,
    // dst_index
    kExprLocalGet, 2,
    // src (child array)
    kExprLocalGet, 1,
    kGCPrefix, kExprAnyConvertExtern,
    kGCPrefix, kExprRefCast, childArr,
    // src_index
    kExprLocalGet, 3,
    // length
    kExprLocalGet, 4,
    kGCPrefix, kExprArrayCopy, parentArr, childArr,
  ]).exportFunc();

// Read element from parent array — get field 0
builder.addFunction('read_parent', makeSig([kWasmExternRef, kWasmI32], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprAnyConvertExtern,
    kGCPrefix, kExprRefCast, parentArr,
    kExprLocalGet, 1,
    kGCPrefix, kExprArrayGet, parentArr,
    kExprRefAsNonNull,
    kGCPrefix, kExprStructGet, parent, 0,
  ]).exportFunc();

// Read element from child array — get field 0
builder.addFunction('read_child', makeSig([kWasmExternRef, kWasmI32], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprAnyConvertExtern,
    kGCPrefix, kExprRefCast, childArr,
    kExprLocalGet, 1,
    kGCPrefix, kExprArrayGet, childArr,
    kExprRefAsNonNull,
    kGCPrefix, kExprStructGet, child, 0,
  ]).exportFunc();

// Set individual child array elements with unique values
builder.addFunction('set_child', makeSig([kWasmExternRef, kWasmI32, kWasmI32], []))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprAnyConvertExtern,
    kGCPrefix, kExprRefCast, childArr,
    kExprLocalGet, 1,  // index
    // Create new child struct
    kExprLocalGet, 2,  // i32 value
    ...wasmF64Const(2.718),
    kGCPrefix, kExprStructNew, child,
    kGCPrefix, kExprArraySet, childArr,
  ]).exportFunc();

const inst = builder.instantiate();

// Phase 1: Create arrays, set unique values, copy, verify
const childA = inst.exports.make_child_array(0);
const parentA = inst.exports.make_parent_array();

// Set unique values in child array
for (let i = 0; i < 10; i++) {
  inst.exports.set_child(childA, i, (i + 1) * 100);
}

// Verify child array
for (let i = 0; i < 10; i++) {
  const v = inst.exports.read_child(childA, i);
  if (v !== (i + 1) * 100) { print('BUG: child[' + i + '] = ' + v); }
}

// Copy child -> parent
inst.exports.copy_child_to_parent(parentA, childA, 0, 0, 10);

// Verify parent array (should have same values)
for (let i = 0; i < 10; i++) {
  const v = inst.exports.read_parent(parentA, i);
  if (v !== (i + 1) * 100) {
    print('BUG: parent[' + i + '] = ' + v + ' expected ' + ((i+1)*100));
  }
}
print('Phase 1 (basic copy) OK');

// Phase 2: GC after copy
gc();
for (let i = 0; i < 10; i++) {
  const v = inst.exports.read_parent(parentA, i);
  if (v !== (i + 1) * 100) {
    print('BUG post-GC: parent[' + i + '] = ' + v);
  }
}
print('Phase 2 (post-GC) OK');

// Phase 3: Warmup for tier-up
for (let k = 0; k < 20000; k++) {
  const c = inst.exports.make_child_array(k);
  const p = inst.exports.make_parent_array();
  inst.exports.copy_child_to_parent(p, c, 0, 0, 10);
  inst.exports.read_parent(p, 0);
}
print('Phase 3 (warmup) done');

// Phase 4: Post tier-up copy + verify
const childB = inst.exports.make_child_array(0);
const parentB = inst.exports.make_parent_array();
for (let i = 0; i < 10; i++) {
  inst.exports.set_child(childB, i, (i + 1) * 777);
}
inst.exports.copy_child_to_parent(parentB, childB, 0, 0, 10);
gc();
for (let i = 0; i < 10; i++) {
  const v = inst.exports.read_parent(parentB, i);
  if (v !== (i + 1) * 777) {
    print('BUG post-tierup: parent[' + i + '] = ' + v + ' expected ' + ((i+1)*777));
  }
}
print('Phase 4 (post tier-up + GC) OK');
```

### Step 3: Run Differentially
```bash
d8 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --liftoff-only /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --no-liftoff /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --gc-interval=50 /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
d8 --stress-compaction /Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js test.js
```

### Step 4: If Clean, Escalate
- Overlapping copies (src and dst ranges overlap)
- Partial copies (copy 3 of 10 elements, verify untouched elements still null)
- Copy from parent array to child array (should FAIL — parent NOT <: child)
- Copy between arrays from DIFFERENT modules (type canonicalization + copy)

## Evolution Plan
- v1: Basic subtyped array copy + GC stress (Steps 2-3)
- v2: Overlapping and partial copies
- v3: Cross-module array copy
