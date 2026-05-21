# Genome 062 — cross-module-canonical

## Hypothesis
Cross-module type sharing in Wasm GC relies on canonical type IDs. Two modules defining structurally equivalent types should get the same canonical ID, enabling cross-module ref.cast and ref.test. Attack angles:

1. **Canonical ID collision**: If two structurally DIFFERENT types hash to the same canonical ID, ref.cast would succeed when it should fail, allowing type confusion between different struct layouts.

2. **20-bit HeapTypeField overflow**: `value-type.h` uses a 20-bit field for heap type indices. With >1M types across modules, IDs could truncate, causing collisions. Creating modules with many recursive type groups could push IDs high.

3. **Recursive type group canonicalization**: Recursive groups add complexity — the canonicalizer must handle cycles. If two recursive groups are structurally different but hash the same way (especially with different recursion depths), they could get the same canonical ID.

4. **Supertype chain length**: If module A defines type T with supertype depth 5 and module B imports it but reconstructs the chain differently, subtype checks may behave inconsistently.

Multi-factor: [canonical type ID assignment] × [cross-module type sharing] × [recursive group hashing/equality] = type confusion between structurally different types.

## DNA

### Source Facts
```cpp
// canonical-types.cc — TypeCanonicalizer
// Manages canonical IDs for Wasm types
// Uses recursive group-level canonicalization

// value-type.h — 20-bit HeapTypeField
// HeapTypeField stores canonical type index
// CVE-2024-6100: overflow/truncation of this field

// canonical-types.cc — CanonicalGroup hashing
// Recursive type groups are hashed for equality checking
// Hash collisions could lead to wrong canonical ID assignment
```

### Files to Audit
- `src/wasm/canonical-types.cc` — TypeCanonicalizer, canonical group hashing, equality
- `src/wasm/canonical-types.h` — data structures for canonical types
- `src/wasm/value-type.h` — HeapTypeField size, type encoding
- `src/wasm/wasm-subtyping.cc` — IsSubtypeOf with canonical IDs
- `src/wasm/module-instantiate.cc` — how imported types resolve to canonical IDs

### Key Questions
1. How does TypeCanonicalizer handle hash collisions? Does it do full equality check after hash match?
2. What is the maximum canonical type ID before HeapTypeField overflows?
3. For recursive type groups, does the canonicalizer detect structurally different groups with the same hash?
4. When module B imports a type from module A, does it re-canonicalize or reuse module A's ID?

### Related CVEs
- CVE-2024-6100: Canonical type ID overflow in 20-bit HeapTypeField
- CVE-2024-2887: Type confusion from wrong canonical ID comparison
- CVE-2024-8194: Related canonicalization issue

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-gc --allow-natives-syntax
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

// TEST 1: Cross-module ref.cast with structurally equivalent types
(function TestCrossModuleCast() {
  try {
    // Module A: defines struct with (i32, i64)
    let bA = new WasmModuleBuilder();
    let stA = bA.addStruct([makeField(kWasmI32, true), makeField(kWasmI64, true)]);

    bA.addFunction('make', makeSig([], [wasmRefType(stA)]))
      .addBody([
        kExprI32Const, 42,
        kExprI64Const, 100,
        kGCPrefix, kExprStructNew, stA
      ]).exportFunc();

    bA.addFunction('get_extern', makeSig([], [kWasmExternRef]))
      .addBody([
        kExprI32Const, 42,
        kExprI64Const, 100,
        kGCPrefix, kExprStructNew, stA,
        kGCPrefix, kExprExternConvertAny,
      ]).exportFunc();

    let instA = bA.instantiate();

    // Module B: defines SAME struct with (i32, i64)
    let bB = new WasmModuleBuilder();
    let stB = bB.addStruct([makeField(kWasmI32, true), makeField(kWasmI64, true)]);

    // Cast externref to local struct type — should work if canonical IDs match
    bB.addFunction('cast_test', makeSig([kWasmExternRef], [kWasmI32]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kGCPrefix, kExprRefTest, stB,
      ]).exportFunc();

    bB.addFunction('cast_get', makeSig([kWasmExternRef], [kWasmI32]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kGCPrefix, kExprRefCast, stB,
        kGCPrefix, kExprStructGet, stB, 0,
      ]).exportFunc();

    let instB = bB.instantiate();

    // Cross-module: pass struct from A to B
    let ext = instA.exports.get_extern();
    let testResult = instB.exports.cast_test(ext);
    print('TEST1: ref.test cross-module = ' + testResult);  // Should be 1 (same canonical type)

    if (testResult === 1) {
      let castResult = instB.exports.cast_get(ext);
      print('TEST1: ref.cast cross-module field = ' + castResult);  // Should be 42
    }

    // Module C: DIFFERENT struct with (i64, i32) — different field order
    let bC = new WasmModuleBuilder();
    let stC = bC.addStruct([makeField(kWasmI64, true), makeField(kWasmI32, true)]);

    bC.addFunction('cast_test_diff', makeSig([kWasmExternRef], [kWasmI32]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kGCPrefix, kExprRefTest, stC,
      ]).exportFunc();

    let instC = bC.instantiate();
    let diffResult = instC.exports.cast_test_diff(ext);
    print('TEST1: ref.test different struct = ' + diffResult);  // Should be 0 (different canonical type)

    if (diffResult === 1) {
      print('TEST1: *** TYPE CONFUSION *** Different struct types have same canonical ID!');
    }

  } catch(e) {
    print('TEST1 ERROR: ' + e.message);
  }
})();

// TEST 2: Many types to test canonical ID space
(function TestManyTypes() {
  try {
    let b = new WasmModuleBuilder();
    let types = [];

    // Create 500 unique struct types
    for (let i = 0; i < 500; i++) {
      let fields = [];
      // Use field count as differentiator
      for (let j = 0; j <= (i % 10); j++) {
        fields.push(makeField(kWasmI32, true));
      }
      // Add a unique field type pattern
      if (i % 3 === 0) fields.push(makeField(kWasmI64, true));
      if (i % 5 === 0) fields.push(makeField(kWasmF64, true));
      types.push(b.addStruct(fields));
    }

    // Function to create and test instances
    b.addFunction('test', makeSig([], [kWasmI32]))
      .addBody([
        kExprI32Const, 1, // return success
      ]).exportFunc();

    let inst = b.instantiate();
    print('TEST2: Module with 500 struct types instantiated OK');

  } catch(e) {
    print('TEST2 ERROR: ' + e.message);
  }
})();

// TEST 3: Recursive type groups
(function TestRecursiveGroups() {
  try {
    let bA = new WasmModuleBuilder();

    // Recursive group: struct A has field (ref null A)
    let stRA = bA.addStruct([
      makeField(kWasmI32, true),
      makeField(wasmRefNullType(0), true)  // self-reference by index 0
    ]);

    bA.addFunction('make_recursive', makeSig([], [kWasmExternRef]))
      .addBody([
        kExprI32Const, 42,
        kExprRefNull, stRA,
        kGCPrefix, kExprStructNew, stRA,
        kGCPrefix, kExprExternConvertAny,
      ]).exportFunc();

    let instA = bA.instantiate();

    // Module B: same recursive structure
    let bB = new WasmModuleBuilder();
    let stRB = bB.addStruct([
      makeField(kWasmI32, true),
      makeField(wasmRefNullType(0), true)  // self-reference
    ]);

    bB.addFunction('cast_recursive', makeSig([kWasmExternRef], [kWasmI32]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kGCPrefix, kExprRefTest, stRB,
      ]).exportFunc();

    let instB = bB.instantiate();

    let ext = instA.exports.make_recursive();
    let result = instB.exports.cast_recursive(ext);
    print('TEST3: Recursive cross-module ref.test = ' + result);  // Should be 1

  } catch(e) {
    print('TEST3 ERROR: ' + e.message);
  }
})();

// TEST 4: Subtype hierarchy cross-module
(function TestSubtypeHierarchy() {
  try {
    // Module A: base struct + derived
    let bA = new WasmModuleBuilder();
    let baseA = bA.addStruct([makeField(kWasmI32, true)]);
    let derivedA = bA.addStruct([makeField(kWasmI32, true), makeField(kWasmI64, true)], baseA);

    bA.addFunction('make_derived', makeSig([], [kWasmExternRef]))
      .addBody([
        kExprI32Const, 42,
        kExprI64Const, 99,
        kGCPrefix, kExprStructNew, derivedA,
        kGCPrefix, kExprExternConvertAny,
      ]).exportFunc();

    let instA = bA.instantiate();

    // Module B: same hierarchy
    let bB = new WasmModuleBuilder();
    let baseB = bB.addStruct([makeField(kWasmI32, true)]);
    let derivedB = bB.addStruct([makeField(kWasmI32, true), makeField(kWasmI64, true)], baseB);

    bB.addFunction('test_base', makeSig([kWasmExternRef], [kWasmI32]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kGCPrefix, kExprRefTest, baseB,
      ]).exportFunc();

    bB.addFunction('test_derived', makeSig([kWasmExternRef], [kWasmI32]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprAnyConvertExtern,
        kGCPrefix, kExprRefTest, derivedB,
      ]).exportFunc();

    let instB = bB.instantiate();

    let ext = instA.exports.make_derived();
    print('TEST4: ref.test base cross-module = ' + instB.exports.test_base(ext));
    print('TEST4: ref.test derived cross-module = ' + instB.exports.test_derived(ext));

  } catch(e) {
    print('TEST4 ERROR: ' + e.message);
  }
})();

print('ALL TESTS COMPLETE');
```

## DO NOT Try These
- Simple same-module ref.cast — well-tested
- Type check elimination in optimizer — dead surface (W3)
- JS-Wasm boundary string sharing — covered by TC-003/TC-006

## Evolution Plan
- v1: Run cross-module tests. Audit canonical-types.cc for hashing and equality. Check HeapTypeField size.
- v2: If cross-module works correctly, try stress: 1000+ types across 10+ modules. Try recursive groups with similar structure but different recursion depth.
- v3: If still clean, try supertype chain mismatches: module A has depth-3 hierarchy, module B has depth-5 hierarchy with same leaf type. Does ref.cast check the full chain?

## Task
Run the tests. Also read `src/wasm/canonical-types.cc` to understand:
1. How canonical IDs are assigned (hash → lookup → compare → assign or reuse)
2. The equality check for recursive groups
3. Whether HeapTypeField size limits apply

Post with tags `cross-module,canonical-types,type-sharing,W8,recursive-groups`.
