# Genome 096 — load-elim-gc-alias

## Hypothesis

WasmLoadElimination (wasm-load-elimination-reducer.h) caches struct field loads and avoids re-loading when a subsequent store at the same offset involves a type deemed "unrelated" by `HeapTypesUnrelated` (wasm-subtyping.h:98-102). `HeapTypesUnrelated(T1, T2)` returns true when neither `IsHeapSubtypeOf(T1, T2)` nor `IsHeapSubtypeOf(T2, T1)`.

For sibling struct types ($B1, $B2 both subtypes of $A), TypesUnrelated($B1, $B2) = true. If a struct.get from a `(ref $B1)` is cached, and then a struct.set to a `(ref $B2)` at the same field offset occurs, the cache is NOT invalidated. This is correct IF the two references genuinely point to different objects.

BUT: if through a sequence of upcasts and downcasts, the SAME runtime object is accessed through both `(ref $B1)` and `(ref $B2)` type annotations (e.g., by exploiting a loop phi that widens to $A and then narrows back to the wrong sibling), the stale cached load would produce a value of the wrong type.

The attack requires TWO factors: (1) load elimination's TypesUnrelated skip, and (2) a type system gap that allows the same object to be accessed through sibling types. Factor 2 could come from incorrect phi widening, incorrect cast elimination, or a cross-boundary type confusion.

Even without factor 2, there's a simpler scenario: `non_aliasing_objects_` tracking (line 158). If `AllocateStruct` results are marked non-aliasing, subsequent stores through ANY type annotation won't invalidate cached loads for that object — even if the object is CAST to a different type and modified. Does the non-aliasing flag survive casts?

## DNA

### Source Facts

1. **TypesUnrelated check** — `wasm-load-elimination-reducer.h:163`
   ```cpp
   if (TypesUnrelated(type_index, key.data().mem.type_index)) {
     ++it; continue;  // Skip invalidation
   }
   ```

2. **HeapTypesUnrelated** — `wasm-subtyping.h:98-102`
   ```cpp
   bool HeapTypesUnrelated(HeapType h1, HeapType h2, const WasmModule* m) {
     return !IsHeapSubtypeOf(h1, h2, m) && !IsHeapSubtypeOf(h2, h1, m);
   }
   ```

3. **Non-aliasing objects** — `wasm-load-elimination-reducer.h:155-161`
   - AllocateStruct/AllocateArray results are marked non-aliasing
   - Non-aliasing objects skip ALL invalidation (line 158)
   - Does this survive ref.cast? If cast creates a new SSA value, is it still marked non-aliasing?

4. **Invalidation on function calls** — `wasm-load-elimination-reducer.h:185-209`
   - `InvalidateMaybeAliasing()` invalidates all non-non-aliasing entries
   - Called on function calls that could modify any struct
   - But immutable fields (mutability == false, line 197) are KEPT even across calls

5. **Field offset calculation** — `wasm-load-elimination-reducer.h:212-214`
   - `WasmStruct::kHeaderSize + type->field_offset(field_index)`
   - Two sibling structs inheriting from same parent SHARE field offsets for inherited fields
   - Fields added by different siblings at the same index have the SAME offset

6. **StructGetOp caching** — `wasm-load-elimination-reducer.h:216-230`
   - `FindImpl(ResolveBase(get.object()), offset, type_index, ...)`
   - Cached by (base, offset, type_index) triple
   - If same base accessed with different type_index, separate cache entries

7. **ResolveBase** — traces through AssertNotNull, TypeGuard operations to find underlying object
   - If ref.cast creates a TypeGuard, ResolveBase may or may not trace through it
   - This determines whether loads through different casts of the same object share cache entries

### Key Files to Read
- `src/compiler/turboshaft/wasm-load-elimination-reducer.h` — FULL FILE, focus on Invalidate, FindImpl, Insert, non_aliasing_objects_, ResolveBase
- `src/wasm/wasm-subtyping.h` — HeapTypesUnrelated
- `src/wasm/wasm-subtyping.cc` — IsHeapSubtypeOf implementation details
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` or `.cc` — how TypeGuard and AssertNotNull are inserted after casts

## Techniques That Work

1. **Define sibling struct hierarchy**: `$A = struct{i32 f0}`, `$B1 = struct{i32 f0, i32 f1} extends $A`, `$B2 = struct{i32 f0, f64 f1} extends $A`
2. **Load then store through siblings**: Load $B1.f1, then store $B2.f1 at same offset, then read $B1.f1 again — does load elimination reuse the first load?
3. **Non-aliasing probe**: Allocate struct with struct.new, load a field, then pass the struct to a function that casts and stores, then load the field again — is the non-aliasing flag cleared by the function call?
4. **Tier-up differential**: Run test with `--no-liftoff` (Turboshaft optimizes) vs `--liftoff-only` (no load elimination)
5. **Warmup**: Use enough iterations (`for (let i = 0; i < 100000; i++)`) to trigger Turboshaft compilation and optimization

## DO NOT Try These

- Don't test with shared types (shared boundary is dead surface)
- Don't test field offset computation correctness (confirmed defended by struct-offset-gc)
- Don't test cross-module type sharing (confirmed defended by cross-module-canonical)

## Evolution Plan

- v1: **Source trace** — Read wasm-load-elimination-reducer.h FULLY. Trace ResolveBase to understand how casts affect cache identity. Check if non_aliasing_objects_ flag survives ref.cast/TypeGuard. Check how Invalidate uses TypesUnrelated and what happens with sibling types sharing field offsets.

- v2: **Non-aliasing escape probe** — Write a test where struct.new allocates a struct, loads a field, passes the struct to an opaque function (call_indirect) that modifies the field, then loads the field again. If load elimination incorrectly reuses the pre-call load (because non-aliasing wasn't cleared by the call), differential between optimized and unoptimized.

- v3: **Sibling type aliasing probe** — Write a test where the same memory location is accessed through sibling struct types $B1 and $B2. Use ref.cast from parent $A to create references of both sibling types to the same object (this requires the object to be a union-compatible type). Load $B1.f1, store $B2.f1, load $B1.f1 again. If TypesUnrelated skips invalidation, the second load returns the stale first value.

## Task

**Start by reading `src/compiler/turboshaft/wasm-load-elimination-reducer.h` completely.** This is the PRIMARY target file. Take notes on:
1. How `non_aliasing_objects_` is populated and queried
2. How `ResolveBase` traces through cast/guard operations
3. How `TypesUnrelated` interacts with `Invalidate`
4. What `InvalidateMaybeAliasing` invalidates vs keeps (especially immutable fields)
5. How cache keys are constructed (base + offset + type_index)

**Then read `src/wasm/wasm-subtyping.h:98-102`** for HeapTypesUnrelated.

**Type invariant to challenge:** Load elimination correctly invalidates cached struct field loads when a store through a different type could affect the same memory location.

**Start with Evolution Plan v1** — complete the source trace before writing any test.
