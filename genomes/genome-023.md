# Genome 023 — extern-any-type-tracking

## Hypothesis
The Turboshaft optimizer removes redundant extern↔any conversion chains: `any.convert_extern(extern.convert_any(x)) == x` (wasm-gc-typed-optimization-reducer.h:530). While mathematically correct, this optimization may cause TYPE INFORMATION LOSS in the type analyzer. When a value crosses from the `any` hierarchy to `extern` and back:

1. `extern.convert_any(x)` — x is in the `any` hierarchy (could be struct, array, i31ref). The result is in `extern` hierarchy.
2. `any.convert_extern(y)` — y is in `extern` hierarchy. The result returns to `any` hierarchy.

If the optimizer removes both conversions, the type analyzer should preserve x's original type. But if the type analyzer tracks the INTERMEDIATE extern type (which has no struct/array subtypes), it may lose the specific struct/array type of x. When the conversion chain is removed and x is used directly, the type analyzer might:
- Know x is `(ref any)` but not that it's `(ref $SpecificStruct)`
- Allow a ref.cast to succeed that should fail, or eliminate a check that should remain
- Miscompute Intersection/Union with the lost type information

Additionally, the lowering at wasm-lowering-reducer.h:141 handles `AnyConvertExtern` by passing through non-primitive objects without adding type annotations. If the Turboshaft graph has a sequence where type information flows through an extern conversion and back, the intermediate loss of type specificity could affect downstream optimization decisions.

Multi-factor: Conversion chain optimization + type information loss + ref.test/ref.cast with lost type info.

## DNA

### Source Facts

**Conversion chain optimization** (wasm-gc-typed-optimization-reducer.h:518-534):
```cpp
V<Object> REDUCE(AnyConvertExtern)(V<Object> object, bool is_shared) {
  LABEL_BLOCK(no_change) {
    return Next::ReduceAnyConvertExtern(object, is_shared);
  }
  if (ShouldSkipOptimizationStep()) goto no_change;
  if (object.valid()) {
    const ExternConvertAnyOp* externalize =
        __ output_graph().Get(object).template TryCast<ExternConvertAnyOp>();
    if (externalize != nullptr) {
      // any.convert_extern(extern.convert_any(x)) == x
      return externalize->object();
    }
  }
  goto no_change;
}
```

**AnyConvertExtern lowering** (wasm-lowering-reducer.h:127-141):
```cpp
V<Object> REDUCE(AnyConvertExtern)(V<Object> object, bool is_shared) {
  // ...null, smi, heap_number checks...
  // For anything else, just pass through the value.
  GOTO(end_label, object);  // No type annotation added
}
```

**Type analyzer handling** (wasm-gc-typed-optimization-reducer.cc):
- ProcessExternConvertAny: should set result type to extern hierarchy
- ProcessAnyConvertExtern: should set result type back to any hierarchy
- When the chain is optimized away, what type does the result get?

**ExternConvertAny/AnyConvertExtern in type system**:
- `extern.convert_any`: any → extern (wrapping)
- `any.convert_extern`: extern → any (unwrapping)
- These cross hierarchy boundaries
- The `any` hierarchy has struct/array/i31 subtypes; `extern` does not

**Relevant type hierarchy facts**:
- In `any` hierarchy: struct types, array types, i31ref, string types
- In `extern` hierarchy: externref (opaque, no subtypes in Wasm)
- Converting to extern LOSES all type specificity
- Converting back to any should recover it... but does the optimizer know this?

### Key Attack Angles
1. Create a struct value, convert to extern, immediately convert back to any
2. The optimizer removes both conversions → value retains original type
3. BUT: does the type analyzer know the value is still the original struct type?
4. If not, subsequent ref.test/ref.cast decisions are made with less type info
5. Test: after round-trip conversion, does ref.test return the same result as without conversion?
6. Test: does the optimizer still eliminate casts after round-trip conversion?
7. Edge case: conversion in one function, reverse in another (cross-function type loss)
8. Edge case: conversion stored in a global/table, retrieved and reverse-converted

### Related Code Locations
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — REDUCE AnyConvertExtern (518-534), REDUCE ExternConvertAny (nearby)
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — Type analyzer handling of conversion operations
- `src/compiler/turboshaft/wasm-lowering-reducer.h` — AnyConvertExtern lowering (127-141)
- `src/wasm/wasm-subtyping.cc` — Hierarchy checks, IsSameTypeHierarchy

## Techniques That Work
1. Build module with struct type `$S` and functions:
   - `f1(x: ref $S) -> externref`: return `extern.convert_any(x)`
   - `f2(y: externref) -> ref any`: return `any.convert_extern(y)`
   - `f3(z: ref $S) -> i32`: return `ref.test $S (any.convert_extern(extern.convert_any(z)))`
2. Test `f3` — should return 1 (z is $S). If optimizer removes conversion chain but loses type info, ref.test might be computed incorrectly.
3. More complex: phi merge of converted and unconverted values
4. Run with `--no-liftoff` to force Turboshaft optimization
5. Compare with `--liftoff` for differential
6. Use `--trace-turbo` or `--trace-wasm-decoder` to see type information in the graph
7. Test with globals: `global.set` an externref that was converted from struct, `global.get` and convert back

## DO NOT Try These
- Simple extern.convert_any without round-trip (no type loss possible)
- Null values through conversion (covered by null-hierarchy, defended)
- Cross-hierarchy null confusion (covered by genome-019, defended)

## Evolution Plan
- v1: **Source audit of type tracking through conversion operations**. Read how ProcessExternConvertAny and ProcessAnyConvertExtern update the type analyzer's types_table. Check if the REDUCE optimization at line 530 preserves the original type annotation or loses it. Read the lowering to understand if type annotations survive the conversion removal.
- v2: **Build round-trip conversion type test**. Create module where struct values go through extern↔any round-trip. Test ref.test results with and without Turboshaft optimization. Check if type information is preserved.
- v3: **Cross-function conversion type loss**. Test conversion in one function and reverse in another. Check if the optimizer can still track types across function boundaries through conversions.

## Task
Start with v1. Read these files:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — Search for "ExternConvertAny" and "AnyConvertExtern" in the type analyzer. How does it track types through these operations?
2. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — REDUCE AnyConvertExtern (518-534) and REDUCE ExternConvertAny. Does the optimization preserve type annotations?
3. `src/compiler/turboshaft/wasm-lowering-reducer.h` — AnyConvertExtern lowering (127-141). Does the lowered code add type annotations?
4. `src/compiler/turboshaft/operations.h` — ExternConvertAnyOp, AnyConvertExternOp definitions

Your type invariant to challenge: "Removing the extern↔any conversion chain preserves the value's type information in the optimizer." If the type analyzer tracks the intermediate extern type (losing struct/array specificity) and the conversion removal doesn't restore the original type, downstream optimizations use stale/broad type information.

Post to #colony-workers with tags `extern,any,conversion,type-tracking`.
