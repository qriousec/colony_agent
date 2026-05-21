# Genome 029 — loop-unroll-phi-cast

## Hypothesis
Loop unrolling (loop-unrolling-reducer.h) transforms loop phi nodes by duplicating the loop body. After unrolling, the phi node may have its inputs reordered or multiplied. The Wasm GC type analyzer's ProcessPhi (wasm-gc-typed-optimization-reducer.cc:393-399) assumes backedge is at input index 1 (DCHECK_EQ(input_index, 1) at line 381). If loop unrolling creates phi nodes where the backedge is NOT at index 1, GetTypeForPhiInput reads the wrong type — using predecessor snapshot type instead of currently-analyzed type or vice versa.

Combined with first-iteration cast elimination being a permanent graph mutation (graph modification at wasm-gc-typed-optimization-reducer.h:274-287), incorrect phi type computation leads to wrong cast elimination decisions that persist after the analysis completes.

Multi-factor: Loop unrolling phi reordering + DCHECK assumption about backedge index + first-iteration cast elimination permanence + struct field OOB from wrong cast.

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
}
```

**GetTypeForPhiInput backedge assumption** (wasm-gc-typed-optimization-reducer.cc:370-385):
```cpp
wasm::ValueType WasmGCTypeAnalyzer::GetTypeForPhiInput(
    const PhiOp& phi, int input_index) {
  // ...
  if (graph_.DominatesInSameBlock(phi.input(input_index), graph_.Index(phi))) {
    DCHECK_EQ(input_index, 1);  // ASSUMES backedge is always index 1
    return types_table_.Get(phi.input(input_index));  // reads current analysis type
  }
  return types_table_.GetPredecessorValue(phi.input(input_index), ...);
}
```

**Cast elimination — permanent** (wasm-gc-typed-optimization-reducer.h:274-287):
```cpp
if (wasm::IsHeapSubtypeOf(...)) {
  return __ MapToNewGraph(cast_op.object());  // CAST REMOVED FROM GRAPH — irreversible
}
```

**Loop unrolling** modifies phi nodes — need to verify:
1. Does it preserve the backedge-at-index-1 invariant?
2. Does it run BEFORE or AFTER the type analyzer?
3. Do unrolled loop bodies create new phi nodes with different input ordering?

**Pipeline ordering** is critical:
- If loop unrolling runs BEFORE the type analyzer, the analyzer sees unrolled phi nodes
- If the type analyzer runs FIRST, unrolling doesn't affect type decisions

**Recent loop bug** (921b3e42db2):
- Loop unrolling created phi patterns the analyzer didn't expect → hit UNREACHABLE
- Fix: fallback to kWasmStructRef (overly broad)
- This confirms loop unrolling CAN create phi patterns that break the analyzer

### Parent DNA (genome 024 — phi-loop-cast-elim)
Parent hypothesized first-iteration cast elimination is wrong when backedge widens phi type. The key question parent was supposed to answer in v1: is cast elimination inline (permanent) or separate (reversible)? Parent produced 0 iterations. This worker takes a lateral angle focusing on loop unrolling.

## Techniques That Work
1. Build struct hierarchy: `$Parent` (i32 field) → `$Child` (i32 + f64 fields)
2. Loop where forward edge is `$Child`, loop body function call returns `(ref $Parent)`
3. `ref.cast $Child` on phi value inside loop
4. `struct.get` the f64 field of `$Child` after cast → OOB if parent
5. Run with `--no-liftoff --turboshaft-wasm` to force optimizer
6. Run with `--turboshaft-loop-unrolling` to trigger unrolling
7. Compare with Liftoff (no optimization): `--liftoff --no-wasm-tier-up`
8. Test with different loop iteration counts to trigger unrolling thresholds

## DO NOT Try These
- Null checks in loops (defended — wasm-null-hierarchy)
- Static hierarchies without phi merging (no loop analysis involved)
- Intersection of types without loops (intersection-uninhabited covered)
- Simple forward-backedge without unrolling (parent 024's v1-v2 angles)

## Evolution Plan
- v1: **Pipeline ordering audit**. CRITICAL: Determine the Turboshaft pipeline order for Wasm. Find where loop unrolling runs relative to the type analyzer + cast eliminator. Read `src/compiler/turboshaft/wasm-turboshaft-compiler.cc` or `pipeline.cc` for the reducer pipeline. If type analysis runs BEFORE unrolling, this hypothesis is dead. If AFTER, proceed.
- v2: **Phi reorder test**. If pipeline order allows, build a module where loop unrolling creates phi nodes with backedge at a non-1 index. Enable `--turboshaft-loop-unrolling`. Check if DCHECK_EQ fires. Test with $Parent/$Child struct hierarchy.
- v3: **Cast elimination permanence exploit**. If pipeline order allows AND phi reorder is confirmed, test: does the wrong phi type cause wrong cast elimination? Build full exploit module: loop body has ref.cast $Child → struct.get $Child.extra_field. With loop unrolling, phi type should be wrong → cast eliminated for wrong type → struct.get reads OOB.

## Task
Start with v1 — this is the GATING question. Read these files IN THIS ORDER:

1. `src/compiler/turboshaft/wasm-turboshaft-compiler.cc` — Find the reducer pipeline for Wasm compilation. Look for the order of: WasmGCTypedOptimizationReducer, LoopUnrollingReducer, and related passes. If type analysis runs first, we need a different angle.
2. `src/compiler/turboshaft/loop-unrolling-reducer.h` — How loop unrolling modifies phi nodes. Does it create new PhiOp with different input ordering? Does it preserve backedge-at-index-1?
3. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — Line 370-385: the GetTypeForPhiInput function. The DCHECK_EQ(input_index, 1) assumption. What happens in release mode if this assumption is violated?
4. `git log -1 -p 921b3e42db2` — The loop unrolling + analyzer bug fix. What exact phi pattern broke?

Your type invariant to challenge: "Loop unrolling preserves the phi input ordering assumptions used by the Wasm GC type analyzer." If loop unrolling reorders phi inputs, GetTypeForPhiInput reads the wrong type, leading to wrong cast elimination.

Post results with tags `loop,unrolling,phi,cast-elimination,pipeline-order`.
