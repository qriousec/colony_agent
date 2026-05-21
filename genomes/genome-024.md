# Genome 024 — phi-loop-cast-elim

## Hypothesis
The Turboshaft type analyzer's `ProcessPhi` (wasm-gc-typed-optimization-reducer.cc:393-399) handles loop headers specially: on the FIRST evaluation, it only uses the forward edge type, ignoring backedges. The cast elimination reducer (wasm-gc-typed-optimization-reducer.h:274-287) then removes ref.cast operations based on first-iteration types. These eliminations are PERMANENT — they modify the graph. When the second iteration widens the phi type via backedge union, the already-eliminated cast is never restored.

Concretely:
1. Forward edge carries `(ref $Child)` into loop phi
2. First iteration: phi type = `(ref $Child)`. Cast elimination sees `ref.cast $Child` → already `$Child` → REMOVE CAST
3. Backedge brings `(ref $Parent)` from function call or phi merge
4. Second iteration: phi type widened to `Union($Child, $Parent) = (ref $Parent)`
5. But the `ref.cast` is already deleted from the graph
6. Values of type `$Parent` (not `$Child`) now flow through without the cast → type confusion
7. `struct.get` on `$Child`-specific field reads OOB from `$Parent` instance

Bug fix 921b3e42db2 confirms loop analysis bugs exist. The fix was conservative (fallback to kWasmStructRef), but edge cases remain where first-iteration cast elimination is wrong rather than just unexpected.

Multi-factor: Loop header phi analysis + first-iteration cast elimination + backedge type widening + struct field OOB read.

## DNA

### Source Facts

**ProcessPhi first-iteration** (wasm-gc-typed-optimization-reducer.cc:393-399):
```cpp
void WasmGCTypeAnalyzer::ProcessPhi(const PhiOp& phi) {
  if (is_first_loop_header_evaluation_) {
    RefineTypeKnowledge(graph_.Index(phi), GetResolvedType(phi.input(0)), phi);
    return;  // ONLY forward edge, backedge IGNORED
  }
  // Second iteration: union of ALL inputs
  wasm::ValueType union_type = GetTypeForPhiInput(phi, 0);
  for (int i = 1; i < phi.input_count; ++i) {
    union_type = wasm::Union(union_type, input_type, module_).type;
  }
}
```

**Cast elimination** (wasm-gc-typed-optimization-reducer.h:274-287):
```cpp
if (wasm::IsHeapSubtypeOf(type.heap_type(), cast_op.config.to.heap_type(), module_) &&
    !IsCastToCustomDescriptor(module_, cast_op.config)) {
  if (to_nullable || type.is_non_nullable()) {
    return __ MapToNewGraph(cast_op.object());  // CAST REMOVED FROM GRAPH
  }
}
```
This runs DURING graph building. Decisions are graph modifications, NOT annotations — they are irreversible.

**GetTypeForPhiInput asymmetry** (wasm-gc-typed-optimization-reducer.cc:370-385):
- Line 376-383: For same-block phi inputs (backedge merges), reads from `types_table_.Get()` (currently-being-analyzed type)
- Line 385: For other inputs, reads from `types_table_.GetPredecessorValue()` (predecessor snapshot)
- DCHECK_EQ(input_index, 1) at line 381 assumes backedge is always index 1 — graph transformations (loop unrolling) could violate this

**Fixed-point iteration control** (wasm-gc-typed-optimization-reducer.cc:55-59):
- TODO #14108: "The snapshot comparison is inefficient" — snapshot created just to test equivalence
- The analysis runs until `needs_revisit` is false, but eliminated operations are never un-eliminated

**Recent loop bug** (921b3e42db2):
- Loop unrolling created phi patterns the analyzer didn't expect → hit UNREACHABLE
- Fix: fallback to kWasmStructRef (overly broad, may mask real issues)

### Key Insight
The type analyzer and the cast eliminator are NOT independent passes. Cast elimination happens inline during graph building as the type analyzer processes operations. This means first-iteration type decisions directly cause permanent graph mutations. The second iteration can widen types but CANNOT restore eliminated casts.

## Techniques That Work
1. Build struct hierarchy: `$Parent` (i32 field) → `$Child` (i32 + f64 fields)
2. Loop where forward edge is `$Child` instance, loop body calls function returning `(ref $Parent)` which could be `$Parent` not `$Child`
3. Inside loop: `ref.cast $Child` on phi value → optimizer should eliminate on first iteration
4. After cast: `struct.get` the extra f64 field of `$Child` → if value is `$Parent`, reads OOB
5. Run with `--no-liftoff --turboshaft-wasm` to force Turboshaft
6. Compare with `--liftoff` (no type optimization)
7. Try `--turboshaft-loop-unrolling` to trigger graph transformations
8. Test with mutual phi references (phi A → phi B → phi A) to stress fixed-point iteration
9. Try custom descriptor types to bypass IsCastToCustomDescriptor guard

## DO NOT Try These
- Simple loops without type narrowing (too basic, well-tested)
- Null checks in loops (defended — genome 019, wasm-null-hierarchy)
- Static type hierarchies without phi merging (no loop analysis involved)
- Intersection of types without loops (intersection-uninhabited covered, defended)

## Evolution Plan
- v1: **Source audit of cast elimination timing**. Critical question: does cast elimination happen DURING the type analysis pass (inline graph modification), or AFTER (separate pass using analysis results)? If inline, first-iteration eliminations are truly permanent. If separate, the second iteration's widened types might prevent elimination. Read the graph builder pipeline order.
- v2: **Build loop type widening test**. Module with $Parent/$Child hierarchy. Loop where forward edge = $Child, backedge = $Parent from function call. ref.cast $Child inside loop. struct.get $Child.extra_field after cast. Test: Turboshaft vs Liftoff differential.
- v3: **Loop unrolling + phi reordering**. Enable loop unrolling to create multi-iteration phi patterns. Test if DCHECK_EQ(input_index, 1) holds after unrolling. If unrolling reorders phi inputs, GetTypeForPhiInput reads wrong type.

## Task
Start with v1. Read these files IN THIS ORDER:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — Find REDUCE(WasmTypeCast) around line 274. Determine: does this run during the same pass as the type analyzer, or separately? Look at the reducer pipeline order.
2. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — ProcessPhi (393-399), the fixed-point loop, `needs_revisit` logic. Trace the full loop analysis flow.
3. `src/compiler/turboshaft/loop-unrolling-reducer.h` — How loop unrolling modifies phi nodes. Does it preserve the backedge-is-index-1 invariant?
4. Git: `git log -1 -p 921b3e42db2` — understand the exact bug and fix.

Your type invariant to challenge: "Cast elimination decisions made during type analysis are sound even when loop backedges bring wider types." If cast elimination is inline (permanent graph mutation) and the second iteration widens the phi type, the eliminated cast should have been preserved.

Post to #colony-workers with tags `phi,loop,cast-elimination,type-widening`.
