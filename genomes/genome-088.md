# Genome 088 — shared-anyconvert-propagation

## Hypothesis
The any.convert_extern (internalize) and extern.convert_any (externalize) operations have explicit is_shared handling in the optimizer (wasm-gc-typed-optimization-reducer.h:518). When a shared externref is internalized to anyref, the shared bit must propagate through the conversion so downstream operations correctly treat the value as shared. If the shared bit is lost during conversion (optimizer computes wrong is_shared, or runtime path doesn't preserve it), the result could be: (1) a shared object treated as non-shared, bypassing shared-specific handling (unshare checks, shared write barriers), or (2) a non-shared object treated as shared, causing allocation in wrong heap space.

## DNA
### Conversion Operations Source
- **wasm-gc-typed-optimization-reducer.h:518-520**:
  ```cpp
  V<Object> REDUCE(AnyConvertExtern)(V<Object> object, bool is_shared) {
    return Next::ReduceAnyConvertExtern(object, is_shared);
  }
  ```
  The optimizer passes through `is_shared` unchanged — but does the LOWERING handle it correctly?

- **turboshaft-graph-interface.cc**: Where are AnyConvertExtern and ExternConvertAny emitted? How is `is_shared` determined?

- **wasm-lowering-reducer.h**: How is AnyConvertExtern lowered? Does it use `is_shared` to select the correct conversion path?

### Conversion Semantics
- **any.convert_extern**: Converts externref → anyref (internalize). The extern value becomes visible to Wasm GC type system.
- **extern.convert_any**: Converts anyref → externref (externalize). The value becomes opaque to Wasm GC.

For shared types:
- `(any.convert_extern (ref shared extern))` should produce `(ref shared any)`
- `(extern.convert_any (ref shared any))` should produce `(ref shared extern)`
- The shared bit must be preserved through the conversion

### Potential Attack Vectors
1. **Shared bit loss during conversion**: If internalization drops the shared bit, a shared object appears non-shared after conversion. Downstream code skips shared-specific handling.
2. **Shared bit flip during optimization**: If the optimizer incorrectly infers is_shared=false for a conversion that should be shared, the lowered code may use the wrong runtime path.
3. **Cross-space reference via conversion chain**: extern.convert_any(shared_obj) → pass to JS → any.convert_extern(received) → loses shared bit.

### Related Findings
- TC-003/004/005: TYPE-based check (IsSubtypeOf) vs VALUE-based check gap. The shared bit is part of the type but the runtime uses value-based checks.
- gc-optimizer-shared-elim (084): AssertType skip for shared types means optimizer errors for shared type conversions won't be caught by assertions.

## Techniques That Work
1. Create shared Wasm module with functions that:
   - Accept (ref shared extern) parameter, internalize via any.convert_extern
   - Accept (ref shared any) parameter, externalize via extern.convert_any
   - Chain conversions: extern→any→extern→any
2. Pass shared objects from JS through conversion chains
3. After conversion, use ref.test/ref.cast to check if shared bit is preserved
4. Compare Liftoff (baseline, no optimization) vs Turboshaft (optimized)
5. Force Turboshaft with warmup or `--wasm-tier-mask-for-testing=2`
6. Use struct.get on converted values to check if shared field access works correctly

## DO NOT Try These
- Non-shared any.convert_extern — well-tested standard path
- Shared string unsharing at boundary — TC-003/004/005 already confirmed
- Optimizer cast elimination for shared types — gc-optimizer-shared-elim confirmed defended

## Evolution Plan
- v1: Source trace AnyConvertExtern and ExternConvertAny lowering for shared types. Read turboshaft-graph-interface.cc for emission, wasm-lowering-reducer.h for lowering. How is is_shared used? Does the lowered code differ for shared vs non-shared? What runtime functions are called?
- v2: Build test with conversion chains. Create a shared extern value, internalize it, check if the result is correctly typed as shared. Then externalize a shared any value and check if the result is correctly shared. Test with ref.test after each conversion step.
- v3: Test tier differential. Compare Liftoff (uses different conversion implementation) vs Turboshaft for shared conversions. If there's a difference, identify which tier is correct and which is wrong.

## Task
Start by reading:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h:518-520` — AnyConvertExtern optimization
2. `src/compiler/turboshaft/wasm-lowering-reducer.h` — search for `AnyConvertExtern` and `ExternConvertAny`. How are they lowered?
3. `src/wasm/turboshaft-graph-interface.cc` — search for `AnyConvertExtern` and `ExternConvertAny`. How is `is_shared` determined from the Wasm type?
4. `src/wasm/baseline/liftoff-compiler.cc` — how does Liftoff handle these conversions for shared types?

The type invariant to challenge: "The shared bit is correctly propagated through any.convert_extern and extern.convert_any conversions in all compilation tiers." If it's lost, shared objects bypass shared-specific handling.

Use `--experimental-wasm-shared` with `/Users/t/v8/out/fuzzbuild/d8`.
