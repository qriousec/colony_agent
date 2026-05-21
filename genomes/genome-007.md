# Genome 007 — subtype-union-null

## Hypothesis
The `wasm::Union` and `wasm::Intersection` functions in `wasm-subtyping.cc` may produce incorrect nullability or exactness attributes when combining types from different hierarchy positions with different nullability/exactness settings. The type optimizer's `RefineTypeKnowledge` (wasm-gc-typed-optimization-reducer.cc:584-610) uses `Intersection` to narrow types. If `Intersection(exact non-null $Child, nullable non-exact $Parent)` produces a type with wrong nullability (e.g., nullable when it should be non-nullable), the optimizer may fail to insert a null check, or if it produces wrong exactness, subsequent exact-match casts may be incorrectly eliminated. The historical bug (commit a970eed8995) showed that nullability was ignored in `EqualValueType` — the same pattern of ignoring attribute bits may exist in the type algebra functions.

## DNA

### Source Facts
- **Historical bug** (`commit a970eed8995`, found by canonical-hash-collision worker): `EqualValueType` only checked type indices, not non-index bits including nullability. Fix: added `is_equal_except_index()` check.
- **wasm::Intersection** (used by `RefineTypeKnowledge` at wasm-gc-typed-optimization-reducer.cc:589-592):
  ```cpp
  wasm::ValueType intersection_type =
      previous_value == wasm::ValueType()
          ? new_type
          : wasm::Intersection(previous_value, new_type, module_).type;
  ```
  This is called every time type knowledge is refined — at casts, null checks, phi merges, branch conditions.

- **wasm::Union** (used by `ProcessPhi` at wasm-gc-typed-optimization-reducer.cc:415):
  ```cpp
  union_type = wasm::Union(union_type, input_type, module_).type;
  ```
  Called when merging types at phi nodes.

- **Type attributes**: Each ValueType has: is_nullable (bit 2), is_exact (bit 3), is_shared (bit 4), ref_type_kind (bits 5-7), plus the index. All of these must be correctly handled in Union/Intersection.

- **Exactness semantics**: `is_exact()` means "this is exactly this type, not a subtype." The exact flag interacts with subtyping: `exact $Child` is NOT a subtype of `exact $Parent` (even though `$Child <: $Parent`).

- **RefineTypeKnowledge returns bottom for uninhabited intersection** (line 602-607): If intersection is uninhabited, the block is marked unreachable. If the intersection computation is wrong (says uninhabited when it's not), valid code becomes unreachable. If it says inhabited when it's not, unsound type knowledge propagates.

### Key Attack Vectors
1. **Exact + non-exact intersection**: `Intersection(exact non-null $Child, nullable non-exact $Parent)` — what's the result? Should be `exact non-null $Child` (since exact $Child is more specific). But if the function doesn't handle exactness correctly, it might return `non-exact non-null $Child` (loses exactness) or `exact nullable $Child` (wrong nullability).
2. **Non-null + nullable union at phi**: If one phi input is `non-null $Type` and another is `nullable $Type`, the union should be `nullable $Type`. If it incorrectly returns `non-null $Type`, a subsequent null access won't be checked.
3. **Cross-hierarchy intersection**: `Intersection(non-null $StructA, nullable $ArrayB)` where $StructA and $ArrayB are unrelated. Should be `bottom`. If it incorrectly returns a non-bottom type, unsound type knowledge propagates.
4. **Shared + non-shared type algebra**: If Union/Intersection don't correctly handle the shared bit, a shared type could be confused with a non-shared type.

### Guards Expected
- `wasm::Intersection` and `wasm::Union` in `wasm-subtyping.cc` should handle all attribute bits
- The optimizer checks `is_uninhabited()` on intersection results
- There may be DCHECKs for attribute consistency

## Techniques That Work
1. Read `wasm::Union` and `wasm::Intersection` in `wasm-subtyping.cc` in full detail
2. Test all combinations of: (nullable, non-nullable) × (exact, non-exact) × (same-type, subtype, unrelated)
3. Use `--trace-wasm-typer` to see what types the analyzer infers at each point
4. Differential: Compare inferred types with actual runtime values
5. Focus on edge cases: bottom types, top types (anyref), abstract types meeting indexed types

## DO NOT Try These
- Testing with only numeric types (Union/Intersection are for reference types)
- Testing EqualValueType (already fixed in a970eed8995)
- Simple cases where both types have same attributes (no attribute interaction)

## Evolution Plan
- v1: Read `wasm::Union` and `wasm::Intersection` implementations in `wasm-subtyping.cc`. Map how each attribute bit (nullable, exact, shared, ref_type_kind) is handled. Identify any asymmetries or missing checks.
- v2: If v1 finds an attribute handling gap, construct a test module that exercises the specific combination. Use a loop or branch to create a phi that merges types with different attributes. Check if the optimizer's inferred type matches expectations.
- v3: If v2 finds a type inference mismatch, construct an exploit: create a code path where the wrong inferred type causes a cast to be eliminated or a null check to be skipped. Test with `struct.get` after a phi where one input is nullable but the union incorrectly says non-nullable → null dereference without check.

## Task
Start by reading these files in order:
1. `src/wasm/wasm-subtyping.cc` — search for `Union` and `Intersection` function definitions. Read the full implementation.
2. `src/wasm/wasm-subtyping.h` — check the function signatures and any attribute-related helpers
3. Run `git -C /Users/t/v8 show a970eed8995` to see the historical nullability fix in EqualValueType (understand the pattern)
4. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc:584-610` — RefineTypeKnowledge uses Intersection
5. Search for test files: `find /Users/t/v8/test -name "*subtyp*" -o -name "*union*" -o -name "*intersection*"` — understand test coverage

The type invariant to challenge: "wasm::Union and wasm::Intersection correctly preserve nullability and exactness attributes across all type combinations."

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc --no-wasm-lazy-compilation --trace-wasm-typer`
