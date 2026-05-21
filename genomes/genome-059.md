# Genome 059 — shared-bit-type-opts

## Hypothesis
W3 + P4 export: The TC-003 root cause is that the compiled wrapper uses TYPE-BASED checking (`IsSubtypeOf(type, kWasmSharedExternRef)`) that misses runtime shared values. The same pattern may exist in the Turboshaft `WasmGCTypedOptimizationReducer`, which eliminates type checks based on declared types.

**Specific concern:** The `generic_kind()` function at value-type.h:424 strips the shared bit via `kGenericKindMask`. If the type optimization reducer uses `generic_kind()` or similar shared-bit-stripping functions to determine type check outcomes, it could:

1. Eliminate a `ref.cast` to a shared type because it thinks the input already has the right type (ignoring shared vs non-shared distinction)
2. Eliminate a null check because the shared variant has different nullability semantics
3. Fold two types as "equivalent" because their generic_kind matches after stripping shared

**The key pattern:** Anywhere V8 compares types and strips or ignores the shared bit could create a type confusion between shared and non-shared variants of the same structural type.

Multi-factor: [shared bit stripping in type comparisons] × [type check elimination in optimizer] × [different memory space for shared vs non-shared] = type confusion.

## DNA

### Source Facts
```cpp
// value-type.h:424 — generic_kind() strips shared bit
constexpr GenericKind generic_kind() const {
  return static_cast<GenericKind>(
      (kind_bits_ >> kGenericKindShift) & kGenericKindMask);
  // kGenericKindMask strips the shared bit!
}

// wasm-gc-typed-optimization-reducer.h — TypeCheckAlwaysSucceeds
// If this uses stripped types, shared ref.cast might be eliminated
```

### Files to Audit
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — type check elimination
- `src/compiler/turboshaft/wasm-gc-type-reducer.h` — type narrowing
- `src/wasm/wasm-subtyping.cc` — IsSubtypeOf, ValidSubtypeDefinition
- `src/wasm/value-type.h` — type comparison functions, shared bit handling
- `src/compiler/turboshaft/type-inference-reducer.h` — type inference at merge points

### Key Question
Does `TypeCheckAlwaysSucceeds` or `IsSubtypeOf` in the optimizer correctly distinguish shared vs non-shared types? If a non-shared struct is cast to a shared struct type (or vice versa), does the optimizer:
- Correctly identify this as a failing cast?
- Or strip the shared bit and conclude the types match?

### Related
- TC-003: TYPE-BASED guard in compiled wrapper stripped shared bit via IsSubtypeOf check
- TC-006: generic_kind() strips shared bit, making shared stringref invisible
- CVE-2024-2887: Type check elimination from wrong inference

## Ready-to-Run Test

```javascript
// Flags: --allow-natives-syntax --experimental-wasm-shared --experimental-wasm-gc --turboshaft-wasm --harmony-struct
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

let b = new WasmModuleBuilder();

// Shared and non-shared struct types with SAME field layout
let sharedStruct = b.addStruct([makeField(kWasmI32, true), makeField(kWasmI64, true)], kNoSuperType, false, true);  // shared
let localStruct = b.addStruct([makeField(kWasmI32, true), makeField(kWasmI64, true)]);  // non-shared

let kSharedAnyRef = wasmRefNullType(kWasmAnyRef).shared();
let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();
let kLocalAnyRef = kWasmAnyRef;

// Function 1: Create shared struct, return as shared anyref
b.addFunction('make_shared', makeSig([], [kSharedAnyRef]))
  .addBody([
    kExprI32Const, 42,
    kExprI64Const, 100,
    kGCPrefix, kExprStructNew, sharedStruct
  ]).exportFunc();

// Function 2: Create local struct, return as local anyref
b.addFunction('make_local', makeSig([], [kLocalAnyRef]))
  .addBody([
    kExprI32Const, 42,
    kExprI64Const, 100,
    kGCPrefix, kExprStructNew, localStruct
  ]).exportFunc();

// Function 3: Cast to SHARED struct type — should only succeed for shared struct
b.addFunction('cast_to_shared', makeSig([kSharedAnyRef], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprRefCast, sharedStruct,
    kGCPrefix, kExprStructGet, sharedStruct, 0,
  ]).exportFunc();

// Function 4: Cast to LOCAL struct type — should only succeed for local struct
b.addFunction('cast_to_local', makeSig([kLocalAnyRef], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprRefCast, localStruct,
    kGCPrefix, kExprStructGet, localStruct, 0,
  ]).exportFunc();

// Function 5: ref.test shared struct type
b.addFunction('test_shared', makeSig([kSharedAnyRef], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprRefTest, sharedStruct,
  ]).exportFunc();

// Function 6: ref.test local struct type
b.addFunction('test_local', makeSig([kLocalAnyRef], [kWasmI32]))
  .addBody([
    kExprLocalGet, 0,
    kGCPrefix, kExprRefTest, localStruct,
  ]).exportFunc();

let inst;
try {
  inst = b.instantiate();
} catch(e) {
  print('INSTANTIATION ERROR: ' + e.message);
  throw e;
}

let shared = inst.exports.make_shared();
let local = inst.exports.make_local();

// BASELINE: ref.test before tier-up
print('=== BEFORE TIER-UP ===');
print('test_shared(shared) = ' + inst.exports.test_shared(shared));  // should be 1
print('test_local(local) = ' + inst.exports.test_local(local));      // should be 1

// Cross-test: shared struct against local type and vice versa
// Need to convert: shared struct → shared anyref, then → shared externref → local externref → local anyref
// Actually, if both are anyref-compatible, we might be able to cross-feed directly

// TIER UP with correct types
for (let i = 0; i < 20000; i++) {
  inst.exports.test_shared(shared);
  inst.exports.test_local(local);
  inst.exports.cast_to_shared(shared);
  inst.exports.cast_to_local(local);
}

print('=== AFTER TIER-UP ===');
print('test_shared(shared) = ' + inst.exports.test_shared(shared));  // should be 1
print('test_local(local) = ' + inst.exports.test_local(local));      // should be 1

// KEY TEST: Does the optimizer still correctly distinguish shared vs local after tier-up?
// Try cast_to_shared with local struct (should throw)
try {
  let r = inst.exports.cast_to_shared(shared);
  print('cast_to_shared(shared) = ' + r);  // should be 42
} catch(e) {
  print('cast_to_shared(shared) THREW: ' + e.message);  // should NOT throw
}

try {
  let r = inst.exports.cast_to_local(local);
  print('cast_to_local(local) = ' + r);  // should be 42
} catch(e) {
  print('cast_to_local(local) THREW: ' + e.message);  // should NOT throw
}

print('ALL TESTS COMPLETE');
```

## Evolution Plan
- v1: Run the test. Check if the optimizer correctly distinguishes shared vs non-shared types after tier-up. Also audit `wasm-gc-typed-optimization-reducer.h` for shared-bit handling.
- v2: If clean, try deeper: loop with ref.cast inside, alternating shared and non-shared inputs. Check if the type reducer eliminates the cast.
- v3: Try br_on_cast with shared types, check if branch elimination incorrectly prunes the shared/non-shared distinction.

## Task
Run the test. Also read `wasm-gc-typed-optimization-reducer.h` and search for where it uses `IsSubtypeOf`, `TypeCheckAlwaysSucceeds`, or type comparison functions. Check if any of these strip or ignore the shared bit.

Post with tags `shared-bit,type-opts,cast-elimination,W3,tc-003-pattern`.
