# Genome 033 — wasm-gc-branch-elim

## Hypothesis
The BranchEliminationReducer (branch-elimination-reducer.h) uses a ConditionManager with a ScopedSnapshotTable to track known branch conditions. When WasmGCTypedOptimizationReducer processes br_on_cast or ref.test, it narrows type knowledge in the taken/not-taken branches. BranchElimination can then eliminate subsequent redundant branches based on this knowledge. If type knowledge from a dominating br_on_cast is applied to a value that has been phi-merged, call-returned, or aliased, the branch elimination uses stale/wrong type knowledge. The eliminated branch guarded a type-specific operation (struct.get on a specific field index) — removing the guard allows wrong-typed values to flow into the operation.

Multi-factor: br_on_cast type narrowing + BranchElimination condition tracking + value aliasing through phi/call + struct.get on narrowed type without guard.

## DNA

### Source Facts

**BranchEliminationReducer** (branch-elimination-reducer.h):
- Uses `ConditionManager` to track known branch outcomes
- `ScopedSnapshotTable` stores condition→outcome mappings
- Eliminates branches where the condition is already known true/false from a dominator
- Works across ALL Turboshaft reducers, including Wasm GC

**WasmGCTypedOptimizationReducer** (wasm-gc-typed-optimization-reducer.h):
- `ProcessBranchOnTarget` (around line 300-380): Handles br_on_cast, ref.test, etc.
- Narrows type in taken/not-taken branches via `RefineTypeKnowledge`
- Eliminated type checks feed into BranchElimination's condition table

**Interaction point**: After WasmGCTypedOptimization narrows a type through br_on_cast, BranchElimination may see a later branch testing the same condition and eliminate it. But if the VALUE has changed (different operation maps to the same condition key), the elimination is wrong.

### Key Questions
1. Does BranchElimination use VALUE identity or CONDITION identity? If condition-based, two different values with the same condition pattern could be confused.
2. How does the snapshot table handle loop backedges? Are snapshots properly scoped to dominance regions?
3. What happens across call boundaries? Does a function call invalidate type knowledge?
4. Are Wasm GC type checks represented as branches in the Turboshaft graph? If so, BranchElimination applies.

### Grep Commands
```bash
# Branch elimination reducer
grep -rn "class BranchEliminationReducer\|ConditionManager\|KnownCondition" src/compiler/turboshaft/branch-elimination-reducer.h | head -20

# How Wasm GC type checks become branches
grep -rn "ProcessBranchOnTarget\|REDUCE.*Branch\|kBranch" src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h | head -20

# Snapshot table scoping
grep -rn "ScopedSnapshotTable\|Seal\|StartNewSnapshot" src/compiler/turboshaft/branch-elimination-reducer.h | head -20

# How branch conditions are keyed
grep -rn "ConditionWithHint\|condition_key\|GetConditionInfo" src/compiler/turboshaft/branch-elimination-reducer.h | head -20
```

### Related Work
- W3 (Type check elimination): WasmGCTypedOptimizationReducer is defended. But the INTERACTION between the type optimizer and BranchElimination is untested.
- W7 (Loop header type inference): Defended for phi patterns. But branch elimination adds another dimension.
- The branch elimination reducer is generic (not Wasm-specific) — it may not understand Wasm type semantics.

## Techniques That Work
1. Build Wasm module with deep type hierarchy ($A → $B → $C)
2. Use br_on_cast to narrow type in one branch
3. After the branch, create a phi or call that may return a different-typed value
4. Use a second br_on_cast or ref.test on the possibly-different value
5. If BranchElimination eliminates the second check based on the first, struct.get on $C-specific field OOBs
6. Run with `--no-liftoff --turboshaft-wasm` to force Turboshaft
7. Compare with Liftoff (no branch elimination)
8. Try `--trace-turbo-reduction` or `--trace-turbo` to see branch elimination decisions

## DO NOT Try These
- Simple ref.cast elimination (covered by wasm-gc-typed-optimization-reducer workers)
- Loop phi patterns (defended — W7 is dead)
- Custom descriptor cast guards (defended — custom-desc-cast, 8 iterations)

## Evolution Plan
- v1: **Source audit of BranchEliminationReducer**. Read branch-elimination-reducer.h to understand: (a) How conditions are keyed — by OpIndex? By value identity? (b) How snapshots are scoped — do they respect dominance? (c) Does Wasm GC type checking create branch conditions that BranchElimination tracks? (d) What happens across call boundaries?
- v2: **Construct exploit module**. Based on source audit, build a module where: br_on_cast $C narrows a value → later, the value is phi-merged with a $A-typed value → BranchElimination should NOT know the merged value is $C, but check if it does → struct.get $C-specific field.
- v3: **Cross-reducer interaction**. Test if WasmGCTypedOptimizationReducer's type narrowing creates BranchElimination conditions. If they're independent (type narrowing doesn't feed into condition table), the hypothesis is dead. If they share state, test the interaction.

## Task
Start with v1. Read these files IN THIS ORDER:

1. `src/compiler/turboshaft/branch-elimination-reducer.h` — The full file. Understand ConditionManager, how conditions are stored, how branches are eliminated. Focus on the `REDUCE(Branch)` and `REDUCE(DeoptimizeIf)` methods.
2. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — Search for `Branch` or `DeoptimizeIf` in REDUCE methods. Does the Wasm GC optimizer create branches that feed into BranchElimination?
3. `src/compiler/turboshaft/wasm-turboshaft-compiler.cc` or `pipeline.cc` — What's the pipeline order? Does BranchElimination run AFTER WasmGCTypedOptimization? If before, they don't interact.
4. Check if Wasm `br_on_cast` / `ref.test` compile to Turboshaft Branch operations that BranchElimination can track.

Your type invariant to challenge: "BranchElimination only eliminates branches when the exact same condition on the exact same value is known from a dominator." If the value changes (phi, call return) but BranchElimination uses stale condition knowledge, the eliminated branch was guarding a type check.

Post with tags `branch-elimination,optimizer,wasm-gc,type-narrowing,coverage`.
