# Genome 097 — phi-union-cast-elim

## Hypothesis

ProcessPhi in `wasm-gc-typed-optimization-reducer.cc:389-427` uses `wasm::Union()` to merge types at loop backedges. On the FIRST evaluation of a loop header (line 393-396), only the forward edge type is used — the backedge type is ignored. On REVISIT (line 400-418), backedge types are Union'd with the forward edge type.

If a loop iterates with values of different subtypes ($B1 on some iterations, $B2 on others, both subtypes of $A), the Union at the phi widens to $A. Inside the loop, if there's a `ref.cast $B1` followed by a field access, the first evaluation may eliminate the cast (because the phi type was $B1 from the forward edge). On revisit, the phi type becomes `Union($B1, $B2) = $A`, and the cast should be KEPT. But if the revisit doesn't correctly re-process the cast elimination, the stale optimization from the first evaluation persists.

The key question: does type narrowing from `ref.cast` inside the loop body correctly INVALIDATE on revisit when the phi type widens? The `TypeSnapshotTable` tracks types at each program point. When the loop is revisited with a wider phi type, the snapshot at the `ref.cast` point must be updated to reflect the wider input type — which means the cast is now NEEDED (can fail for $B2 values).

A second angle: `RefineTypeKnowledge` at `wasm-gc-typed-optimization-reducer.cc:584-634` uses `Intersection` of the new type with the previously known type. If the previous type was narrowed by a cast ($B1) and the new type from revisit is $A, the intersection `Intersection($B1, $A) = $B1` — the narrowing PERSISTS from the first evaluation! This could cause the optimizer to keep treating the value as $B1 even when the phi actually carries $B2 on some iterations.

## DNA

### Source Facts

1. **ProcessPhi first evaluation** — `wasm-gc-typed-optimization-reducer.cc:389-396`
   ```cpp
   if (is_first_loop_header_evaluation_) {
     RefineTypeKnowledge(graph_.Index(phi), GetResolvedType(phi.input(0)), phi);
     return;
   }
   ```
   Only uses forward edge type. Backedge is unknown.

2. **ProcessPhi revisit** — `wasm-gc-typed-optimization-reducer.cc:400-418`
   ```cpp
   for (int i = 1; i < phi.input_count; ++i) {
     wasm::ValueType input_type = GetResolvedType(phi.input(i));
     if (input_type.is_uninhabited()) continue;
     union_type = wasm::Union(union_type, input_type, module_).type;
   }
   ```
   Union of all input types including backedge.

3. **RefineTypeKnowledge** — `wasm-gc-typed-optimization-reducer.cc:584-634`
   ```cpp
   wasm::ValueType intersection_type =
       previous_value == wasm::ValueType()
           ? new_type
           : wasm::Intersection(previous_value, new_type, module_).type;
   ```
   Uses INTERSECTION with previous value. If previous was narrowed, intersection keeps the narrow type.

4. **TypeCheckAlwaysSucceeds** — `wasm-gc-typed-optimization-reducer.cc:~200-260`
   - Checks if a cast can be eliminated based on inferred type
   - If the inferred type at a WasmTypeCast is already a subtype of the cast target, the cast is removed
   - This decision is made during first evaluation and may not be reconsidered on revisit

5. **Loop revisitation mechanism** — `wasm-gc-typed-optimization-reducer.cc:44-60`
   ```cpp
   bool needs_revisit = CreateMergeSnapshot(
       base::VectorOf({old_snapshot, snapshot}),
       base::VectorOf({true, true}));
   ```
   If types changed at loop header, the loop is marked for revisit. But does revisit re-run ALL operations in the loop body, or just update the phi types?

6. **Union operation** — `wasm-subtyping.cc:~580-650`
   - Union(T1, T2) = LUB (least upper bound) in the type hierarchy
   - Union($B1, $B2) where both are subtypes of $A: returns $A (if no closer common supertype)
   - Correctly handles nullability and exactness

7. **Snapshot restoration on revisit** — The TypeSnapshotTable should be restored to the loop header snapshot before re-processing the loop body. This means type narrowings INSIDE the loop body should be re-derived from the new (wider) phi type. But if the snapshot restoration is incomplete (e.g., only restores phi types, not all intermediate points), cast eliminations from the first pass persist.

### Key Files to Read
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — FULL FILE, focus on ProcessPhi, RefineTypeKnowledge, loop revisit mechanism, TypeCheckAlwaysSucceeds
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — TypeSnapshotTable, class definition
- `src/wasm/wasm-subtyping.cc` — Union function implementation
- `src/compiler/turboshaft/loop-unrolling-reducer.h` — does loop unrolling interact with type inference?

## Techniques That Work

1. **Define 3-level type hierarchy**: `$A = struct{i32}`, `$B1 = struct{i32, i32} extends $A`, `$B2 = struct{i32, f64} extends $A`
2. **Loop with alternating subtypes**: Even iterations get $B1, odd iterations get $B2 via br_on_cast or select
3. **Cast inside loop**: `ref.cast $B1` on the phi value, then access $B1-specific field (f1 as i32)
4. **Detect elimination**: If the cast to $B1 is eliminated, accessing field f1 on a $B2 object reads f64 bits as i32 — type confusion!
5. **Tier-up differential**: `--no-liftoff` vs `--liftoff-only` — Liftoff doesn't eliminate casts
6. **Warmup with single type first**: First 10k iterations use only $B1 (monomorphic). Then switch to alternating. The optimizer sees monomorphic phi on first eval, may eliminate cast. On revisit (if it happens), phi should widen.
7. **Multiple phis**: Have several phi values in the loop, some with stable types, some with alternating — more complex merge behavior

## DO NOT Try These

- Don't test basic ref.cast correctness without loop context (already confirmed defended)
- Don't test br_on_cast chain narrowing without loop (br-on-cast-chain-narrow scope, though it was non-compliant)
- Don't test with only one subtype (Union($B1, $B1) = $B1, no widening)

## Evolution Plan

- v1: **Source trace** — Read wasm-gc-typed-optimization-reducer.cc FULLY. Answer these questions:
  1. When a loop is revisited, does the ENTIRE loop body get re-processed, or just the phis?
  2. If the entire body is re-processed, are cast elimination decisions from the first pass RECONSIDERED?
  3. How does the TypeSnapshotTable handle loop revisits? Is it restored to the header state?
  4. Does RefineTypeKnowledge's Intersection semantics cause narrowing to persist across revisits?

- v2: **Widening probe** — Write a test module with alternating subtypes in a loop. The phi should carry $B1 on forward edge, then widen to $A on revisit. Inside the loop, cast to $B1 and access $B1-specific field. Run with `--no-liftoff` and `--liftoff-only`. If the cast is incorrectly eliminated, the $B1-specific field access on a $B2 object produces a wrong value (type confusion differential).

- v3: **Escalation** — If v2 finds a differential, minimize the test case. If v2 is clean, try more complex scenarios: nested loops, multiple phis, casts with nullability differences, or phi values that come from function parameters (not just struct.new).

## Task

**Start by reading `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` FULLY.** This is a .cc file (not .h) and contains the actual optimization logic. Take notes on:
1. ProcessPhi (lines ~389-427) — first vs revisit handling
2. The loop revisit mechanism (lines ~44-83) — how does CreateMergeSnapshot work?
3. WasmTypeCast reduction (lines ~200-260) — when is a cast eliminated?
4. RefineTypeKnowledge (lines ~584-634) — how does Intersection preserve narrowing?
5. How `is_first_loop_header_evaluation_` is set and used

**Also read `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h`** for the TypeSnapshotTable definition.

**Type invariant to challenge:** Cast elimination decisions made during the first loop evaluation are correctly reconsidered when phi types widen on loop revisit.

**Start with Evolution Plan v1.**
