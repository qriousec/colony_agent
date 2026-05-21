# Genome 134 — br-on-cast-complement-type

## Hypothesis
`br_on_cast` and `br_on_cast_fail` provide type narrowing in both the taken and fall-through branches. In the taken branch of `br_on_cast`, the value is narrowed to the cast target type. In the fall-through (or taken branch of `br_on_cast_fail`), the type should be narrowed to the COMPLEMENT — the original type minus the cast target type.

Computing this complement is complex. For example, if the original type is `(ref null $Parent)` and the cast target is `(ref $Child)`:
- Taken (cast succeeds): value is `(ref $Child)` (non-null, is Child)
- Fall-through (cast fails): value is `(ref null $Parent)` minus `(ref $Child)` = `null` OR `$Parent instances that are NOT $Child`

If the complement computation in the Turboshaft type optimizer is wrong — e.g., it computes `(ref $Parent)` instead of the correct complement, or it drops nullability, or it incorrectly concludes the complement is uninhabited (bottom type) — the optimizer makes wrong assumptions:
- If complement = bottom → the fall-through branch is considered dead → code is eliminated → if reached at runtime, undefined behavior
- If complement loses nullability → null passes through a non-null path → null deref
- If complement is too wide → less optimization (safe but missed opportunity)

## DNA
- **wasm-gc-typed-optimization-reducer.cc:435-491**: `ProcessBranchOnTarget` handles WasmTypeCheck conditions in branches.
  - True branch (cast succeeds): refines object type to `check.config.to` (target type) — CORRECT
  - False branch (cast fails): marks unreachable if check always succeeds, otherwise **NO narrowing** — just keeps original type. **NO complement computation exists.**
- **turboshaft-graph-interface.cc:8846-8861**: `BrOnCastFailImpl`:
  - Line 8860: `value_on_fallthrough->op = AnnotateWasmType(object.op, config.to);` — annotates fallthrough with TARGET type.
  - This is CORRECT: br_on_cast_fail fallthrough = cast SUCCEEDED, so value IS target type.
  - Branch taken (cast failed) = value retains source type, no narrowing to complement.
- **NO complement type operation exists**: V8 has `Union()` and `Intersection()` in wasm-subtyping.cc but NO `Difference()` or `Complement()`. The false branch of br_on_cast cannot be narrowed to "source minus target."
- **Analyzer gap for abstract casts**: `ProcessBranchOnTarget` (line 440-463) only handles `WasmTypeCheck` and `IsNull`. BrOnCastAbstract and other abstract cast variants may bypass the type analyzer entirely — no type narrowing in either branch direction.
- **function-body-decoder-impl.h:6634-6693**: Decoder br_on_cast semantics — fallthrough type is source type with modified nullability (lines 6689-6691). When `flags.src_is_null && !flags.res_is_null`, fallthrough remains nullable (null stays in fallthrough). Otherwise becomes non-nullable.
- **Null handling edge case**: In br_on_cast with `null_succeeds=true`, null branches AWAY. The fallthrough should be non-null. But the TypeGuard at turboshaft-graph-interface.cc:5851-5856 may not always be emitted.
- **CVE-2024-2887 adjacent**: Wrong type inference in optimizer. The CONSERVATIVE approach (no complement narrowing) is safe for the false branch. But the TRUE branch narrowing at line 444 could be wrong if `check.config.to` is more specific than what the cast actually guarantees.
- **Key attack surface**: The INTERACTION between br_on_cast narrowing and loop phi widening. If br_on_cast narrows in the true branch, and the narrowed value feeds back to a loop phi, the revisit union may widen it. But the cast was already eliminated based on the narrow type...

## Techniques That Work
1. Create a type hierarchy: $A, $B extends $A, $C extends $A (siblings under $A)
2. Use br_on_cast to test for $B — taken branch gets (ref $B), fall-through should get non-$B $A's
3. In the fall-through, perform operations that assume the value is not $B — e.g., cast to $C
4. If the fall-through type is incorrectly narrowed (e.g., optimizer thinks fall-through type IS $C), the cast to $C might be eliminated
5. Then pass a $B instance that fails the first br_on_cast — it enters the fall-through with wrong type
6. Force Turboshaft tier-up for the optimizer to apply type narrowing
7. Test with null variants to check null handling in complement

## DO NOT Try These
- Shared types (--experimental-wasm-shared, BANNED)
- br_on_cast without type hierarchies — simple types don't exercise complement computation
- Liftoff testing — need Turboshaft optimizer for type narrowing
- Single-path tests (no branching) — need both taken and fall-through to test complement

## Evolution Plan
- v1: **Abstract cast analyzer bypass**. Source research shows `ProcessBranchOnTarget` only handles `WasmTypeCheck` and `IsNull` opcodes. Test: use `BrOnCastAbstract` (abstract casts like br_on_cast to i31, struct, array, eq, any) which may bypass the type analyzer. If the analyzer doesn't process the abstract cast branch, the true branch retains the original (overly broad) type instead of being narrowed. In a loop, this could cause the phi to use wrong types. Build test: br_on_cast to structref (abstract), in true branch access struct fields — does the optimizer properly narrow?
- v2: **br_on_cast + loop phi interaction**. CRITICAL angle: combine br_on_cast true-branch narrowing with loop phi widening. Scenario: loop header phi starts with `(ref $Parent)`. In the loop body, br_on_cast to $Child narrows to `(ref $Child)` in true branch. This narrowed value feeds back to phi. On revisit: union(`(ref $Parent)` from forward, `(ref $Child)` from back) = `(ref $Parent)`. Was the cast in the loop body eliminated during first pass (when phi was narrower)? If the Turboshaft fixpoint doesn't properly re-evaluate eliminated casts, the first-pass elimination persists with the widened type.
- v3: **Null TypeGuard emission gap**. Source research shows br_on_cast with `null_succeeds=true` should emit a TypeGuard to ensure fallthrough is non-null. Check turboshaft-graph-interface.cc:5851-5856 — is the TypeGuard always emitted? If missed, fallthrough value is typed as non-null but actually could be null (the null branched away). In optimized code, subsequent struct.get skips null check → null deref on a value the optimizer guarantees non-null.

## Task
Start by reading:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — search for BranchOp processing, TypeAfterBranch, or how branches update type snapshots. Also search for ProcessTypeCast, ProcessTypeCheck.
2. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — read the full ProcessTypeCast and ProcessTypeCheck functions. How do they update the type snapshot?
3. `src/wasm/wasm-subtyping.cc` — read Intersection() function (around line 626-655). Also search for complement/difference/negation operations on types.
4. `grep -rn 'br_on_cast\|BrOnCast\|TypeAfterBranch\|branch.*narrow' /Users/t/v8/src/compiler/turboshaft/wasm-gc*` — find all branch-narrowing code

Then build test 1:
- Struct $A {i32}, $B extends $A {i32, i64}, $C extends $A {i32, f64}
- Function: takes (ref $A). br_on_cast (ref $B) → branch1. Fall-through: ref.cast (ref $C), struct.get f64 field.
- Call 10000 times with $C instances (Turboshaft tier-up) — both casts succeed normally.
- Then call with $B instance: should fail br_on_cast to $B (wait — $B IS $B, so it succeeds).
- Correction: Call with $A instance (base, not $B, not $C). br_on_cast to $B fails → fall-through → ref.cast to $C should TRAP.
- If ref.cast to $C was eliminated (optimizer inferred fall-through is only $C), then $A passes as $C → OOB field access.

Run with: `/Users/t/v8/out/fuzzbuild/d8 test.js`
No --experimental-wasm-shared flag.
