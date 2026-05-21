# Genome 018 — intersection-uninhabited

## Hypothesis
The Wasm type `Intersection()` function (wasm-subtyping.cc) is used by the Turboshaft optimizer to determine:
1. Whether ref.test always fails (918bd974f61) → returns `Word32Constant(0)`
2. Whether ref.cast always traps (uses is_uninhabited sentinel) → emits `Unreachable()`

If `Intersection()` incorrectly returns an uninhabited type for types that actually DO overlap, ref.test returns 0 when it should return 1 (type check bypassed), or ref.cast traps when it should succeed. The security impact: if ref.test is used to guard a type-specific code path (e.g., `if (ref.test $Struct) { struct.get ... }`), returning false bypasses the guard, and subsequent code uses a value with the wrong assumed type.

Multi-factor: Intersection function edge cases + always-failing optimization + complex type features (exact, shared, recursive groups, custom descriptors).

## DNA

### Source Facts

**Intersection usage in optimizer** (wasm-gc-typed-optimization-reducer.h):
- ref.cast optimization (~line 206-208): `CHECK(!Intersection(...).type.is_uninhabited())` — asserts cast doesn't always fail
- ref.test optimization (~line 263-270):
  ```cpp
  wasm::ValueType true_type = wasm::Intersection(type, type_check.config.to, module_, module_).type;
  if (true_type.is_uninhabited()) { return __ Word32Constant(0); }
  ```

**Intersection function** (wasm-subtyping.cc):
- `Intersection(ValueType type1, ValueType type2, const WasmModule* module1, const WasmModule* module2)`
- Handles: nullability, heap type subtyping, recursive groups
- Returns: `{result_type, module}` where result_type may be uninhabited

**Related fixes**:
- 160612a7539: Changed `type == kWasmBottom` to `type.is_uninhabited()` — there were MULTIPLE uninhabited types (ref none, ref nofunc, ref noextern) that weren't caught
- 918bd974f61: Added the always-failing ref.test optimization using Intersection

**Type features that add complexity to Intersection**:
1. **Exact types**: `(ref exact $T)` only matches $T, not subtypes. Intersection of `(ref exact $T)` and `(ref $S)` where S is a subtype of T should be uninhabited (S is not exactly T), but if Intersection doesn't check exactness...
2. **Shared types**: `(shared ref $T)` is in shared space. Intersection of shared and non-shared variants of the same type should be uninhabited.
3. **Custom descriptors**: Types with descriptors have additional constraints. Does Intersection handle descriptor compatibility?
4. **Recursive groups**: Types in the same recursive group can have complex subtyping. Does Intersection handle recursive type references correctly?

### Key Code Locations
- `src/wasm/wasm-subtyping.cc` — Intersection() implementation
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — Optimizer using Intersection
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — Type analyzer feeding types to optimizer
- `src/wasm/canonical-types.cc` — Canonical subtyping used by Intersection

## Techniques That Work
1. Build module with complex type hierarchies involving exact, shared, recursive groups
2. Create functions that do ref.test/ref.cast with types that have non-obvious intersection behavior
3. Run with `--no-liftoff` to force Turboshaft optimization
4. Compare ref.test results between Liftoff (baseline, no optimization) and Turboshaft (optimized)
5. Focus on Intersection edge cases: exact types, shared types, types at hierarchy boundaries

## DO NOT Try These
- Basic ref.cast/ref.test with simple types (extensively tested by desc-exactness, custom-desc-cast)
- IsCastToCustomDescriptor guard bypass (dead surface, confirmed by desc-inline-exactness)
- Cross-module type sharing (tested by cross-module-exact)

## Evolution Plan
- v1: **Source audit of Intersection function**. Read the full implementation in wasm-subtyping.cc. Map how it handles: nullability, exact types, shared types, custom descriptors, recursive groups. Identify any combination where the result could be wrong. Check if exactness is propagated correctly through the intersection.
- v2: **Differential testing: Liftoff vs Turboshaft**. Build modules where ref.test result depends on Intersection correctness. Run same module with Liftoff (no optimization) and Turboshaft (optimized). If results differ, Intersection has a bug.
- v3: **Edge case fuzzing**. Generate modules with unusual type combinations: exact subtype of non-exact, shared version of non-shared type, recursive group with cross-references, descriptor types with different descriptors but same structural shape.

## Task
Start with v1. Read:
1. `src/wasm/wasm-subtyping.cc` — search for `Intersection` function, read fully
2. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — read the ref.test and ref.cast optimization paths
3. `src/wasm/value-type.h` — understand is_uninhabited(), is_exact(), is_shared()

Your type invariant to challenge: "Intersection(A, B) returns uninhabited if and only if no runtime value can have both type A and type B." If Intersection returns uninhabited for types that DO overlap, ref.test returns wrong result.

Post to #colony-workers with tags `intersection,uninhabited,ref-test,optimizer`.
