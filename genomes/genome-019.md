# Genome 019 — wasm-null-hierarchy

## Hypothesis
Wasm has distinct null types for each type hierarchy: `ref.null any` (anyref hierarchy), `ref.null func` (funcref hierarchy), `ref.null extern` (externref hierarchy). These have different bottom types: `nullref` (ref none), `nullfuncref` (ref nofunc), `nullexternref` (ref noextern). In V8, all null values map to the same runtime representation (`kWasmNull`), but the type system must track which hierarchy a null belongs to.

If the type analyzer or optimizer confuses nulls across hierarchies — for example, treating a value known to be `ref.null func` as being in the any hierarchy — it could:
1. Eliminate a ref.cast that should fail (casting a funcref null to anyref should trap for non-nullable)
2. Allow a value to cross hierarchy boundaries, bypassing IsSameTypeHierarchy validation
3. Incorrectly determine type intersection results (func null vs any null)

The recent removal of the `module` parameter from `IsSameTypeHierarchy` (c047ab74b85) reduced the function to `NullSentinelImpl(type1) == NullSentinelImpl(type2)`. If `NullSentinelImpl` has edge cases for indexed types that resolve to different hierarchies, the hierarchy check could be wrong.

## DNA

### Source Facts

**Null type hierarchy** (wasm-subtyping.cc):
- `NullSentinelImpl(HeapType)` maps types to their hierarchy root
- Used by `IsSameTypeHierarchy` to check if two types are in the same hierarchy
- `ToNullSentinel(TypeInModule)` returns the null sentinel for a type-in-module

**IsSameTypeHierarchy change** (c047ab74b85):
- Removed `module` parameter: `IsSameTypeHierarchy(HeapType type1, HeapType type2)`
- Implementation: `NullSentinelImpl(type1) == NullSentinelImpl(type2)`
- Used in function-body-decoder-impl.h for validating ref.cast/ref.test/br_on_cast

**WasmNull representation**:
- Single `kWasmNull` object at runtime (not hierarchy-specific)
- Type system distinguishes null types: kWasmNullRef (any), kWasmNullFuncRef (func), kWasmNullExternRef (extern)

**Type analyzer** (wasm-gc-typed-optimization-reducer.cc):
- Tracks value types through the program
- At merge points (phis), widens types using Union
- The Union of `ref.null func` and `ref.null any` should be... what? They're in DIFFERENT hierarchies!

**Validation** (function-body-decoder-impl.h):
- ref.cast validates `IsSameTypeHierarchy(source, target)` at decode time
- But what about dynamically-typed values at merge points?

### Key Attack Angles
1. Phi merging of null values from different hierarchies
2. br_on_null feeding into ref.test/ref.cast for a different hierarchy
3. NullSentinelImpl edge cases for indexed types
4. Type narrowing after null checks: `if (ref.is_null) ... else ...` — does the else branch correctly narrow the type?

### Related Code Locations
- `src/wasm/wasm-subtyping.cc` — NullSentinelImpl, IsSameTypeHierarchy, Intersection
- `src/wasm/value-type.h` — kWasmNullRef, kWasmNullFuncRef, kWasmNullExternRef
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — Type analyzer, ProcessPhi
- `src/wasm/function-body-decoder-impl.h` — Validation of ref.cast/ref.test hierarchy

## Techniques That Work
1. Build modules that create null values from different hierarchies
2. Use `select` or phi (if/else) to merge values from different hierarchies
3. Test ref.test/ref.cast on the merged value
4. Compare behavior between Liftoff (no type optimization) and Turboshaft (optimized)
5. Test null propagation through function calls (funcref null → anyref parameter)
6. Test br_on_null + br_on_cast combinations across hierarchies

## DO NOT Try These
- Simple ref.cast null tests (basic null handling is well-tested)
- Cross-module type sharing (already tested)
- Custom descriptor exactness (dead surface)

## Evolution Plan
- v1: **Source audit of NullSentinelImpl and type hierarchy tracking**. Read wasm-subtyping.cc NullSentinelImpl for all HeapType kinds. Check if every indexed type correctly maps to its hierarchy root. Read the type analyzer's ProcessPhi to understand how it handles merge of nullable types from different hierarchies. Read IsSameTypeHierarchy callers in function-body-decoder-impl.h.
- v2: **Build cross-hierarchy null tests**. Create a module that: (a) gets ref.null func and ref.null any, (b) merges them via select or phi, (c) does ref.test on the merged value for each hierarchy. Compare Liftoff vs Turboshaft results.
- v3: **br_on_null + type narrowing tests**. Create a module where br_on_null feeds into code that assumes the non-null branch has a specific type. If the type narrowing is wrong (e.g., narrows to wrong hierarchy), subsequent operations use the value with wrong type assumptions.

## Task
Start with v1. Read these files:
1. `src/wasm/wasm-subtyping.cc` — NullSentinelImpl, IsSameTypeHierarchy, ToNullSentinel
2. `src/wasm/value-type.h` — null type definitions, is_nullable(), heap_type()
3. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — ProcessPhi, type merging
4. `src/wasm/function-body-decoder-impl.h` — IsSameTypeHierarchy usage in validation

Your type invariant to challenge: "Values from different type hierarchies (any, func, extern) can never be confused by the type system or optimizer." If phi merge, null propagation, or IsSameTypeHierarchy has edge cases, hierarchy boundaries could be crossed.

Post to #colony-workers with tags `null,hierarchy,type-system,optimizer`.
