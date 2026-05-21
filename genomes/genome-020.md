# Genome 020 — phi-loop-type-confusion

## Hypothesis
The Turboshaft type analyzer's `ProcessPhi` (wasm-gc-typed-optimization-reducer.cc:388-427) handles loop headers specially: on the FIRST evaluation, it only uses the forward edge type, ignoring backedges. This is correct as a fixed-point iteration starting point, but the optimizer may make IRREVERSIBLE decisions (cast elimination, type check removal) based on this incomplete first-iteration type. If:

1. Forward edge carries narrow type (e.g., `(ref $Parent)`)
2. Optimizer eliminates a ref.cast to `$Parent` because "value is already $Parent"
3. Backedge brings wider type (e.g., `(ref any)` from a phi merge or function call)
4. Second iteration widens the phi type but the cast is already gone

The result: values flowing through the backedge bypass the cast that was eliminated on the first iteration. A value of type `$Unrelated` passes through as if it were `$Parent` — type confusion.

Recent bug fix 921b3e42db2 ("Fix unreachable in TS type analyzer") confirms loop analysis bugs exist: graph transformations (loop unrolling) created patterns the analyzer didn't expect, hitting UNREACHABLE. The fix was conservative (fallback to kWasmStructRef), but there may be remaining edge cases where the first-iteration type is WRONG rather than just unexpected.

Multi-factor: Loop header phi type analysis + cast elimination on first iteration + backedge type widening + custom descriptor exact types.

## DNA

### Source Facts

**ProcessPhi first-iteration behavior** (wasm-gc-typed-optimization-reducer.cc:388-399):
```cpp
void WasmGCTypeAnalyzer::ProcessPhi(const PhiOp& phi) {
  if (is_first_loop_header_evaluation_) {
    // Only use forward edge on first iteration
    RefineTypeKnowledge(graph_.Index(phi), GetResolvedType(phi.input(0)), phi);
    return;
  }
  // Second iteration: compute union of all inputs
  wasm::ValueType union_type = GetTypeForPhiInput(phi, 0);
  for (int i = 1; i < phi.input_count; ++i) {
    // ...
    union_type = wasm::Union(union_type, input_type, module_).type;
  }
}
```

**GetTypeForPhiInput edge detection** (wasm-gc-typed-optimization-reducer.cc:369-386):
```cpp
wasm::ValueType WasmGCTypeAnalyzer::GetTypeForPhiInput(const PhiOp& phi, int input_index) {
  OpIndex input = ResolveAliases(phi.input(input_index));
  if (current_block_->begin().id() <= input.id() && input.id() < phi_id.id()) {
    DCHECK(graph_.Get(input).Is<PhiOp>());
    DCHECK(current_block_->IsLoop());
    DCHECK_EQ(input_index, 1);  // Assumes backedge is always index 1
    return types_table_.Get(input);
  }
  return types_table_.GetPredecessorValue(input, input_index);
}
```
**DCHECK_EQ(input_index, 1)** — if graph transformations reorder phi inputs, this reads wrong type.

**RefineTypeKnowledge intersection** (wasm-gc-typed-optimization-reducer.cc:584-610):
- Computes `Intersection(previous_value, new_type)` to narrow types
- If previous type is unknown (ValueType()), uses new_type directly
- Marks block unreachable if intersection is uninhabited

**Cast elimination** (wasm-gc-typed-optimization-reducer.h:274-287):
```cpp
if (wasm::IsHeapSubtypeOf(type.heap_type(), cast_op.config.to.heap_type(), module_) &&
    !IsCastToCustomDescriptor(module_, cast_op.config)) {
  if (to_nullable || type.is_non_nullable()) {
    return __ MapToNewGraph(cast_op.object());  // CAST REMOVED
  }
}
```

**Recent loop bug** (921b3e42db2):
- Graph builder emits limited patterns, but loop unrolling creates others
- Code hit UNREACHABLE for unexpected phi patterns
- Fix: fallback to kWasmStructRef (conservative but possibly too broad)

### Key Attack Angles
1. Create a loop where the forward edge has a narrow struct type
2. Inside the loop, do ref.cast to the narrow type (should be eliminated on first iteration)
3. On the backedge, merge with a value of a wider or different type
4. The cast elimination on first iteration is permanent — backedge values skip the cast
5. Use custom descriptor types to maximize the impact of bypassing the cast
6. Test with loop unrolling (`--turboshaft-loop-unrolling`) to trigger graph transformations

### Related Code Locations
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — ProcessPhi (388-427), RefineTypeKnowledge (584-610)
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — REDUCE WasmTypeCast (274-287), REDUCE WasmTypeCheck (341-356)
- `src/compiler/turboshaft/loop-unrolling-reducer.h` — Loop unrolling that creates new phi patterns
- `src/wasm/wasm-subtyping.cc` — Union, Intersection used by type analyzer

## Techniques That Work
1. Build module with struct type hierarchy: `$Parent` (supertype) and `$Child` (subtype with extra field)
2. Create a function with a loop:
   - Before loop: create `$Child` instance → forward edge has narrow type `(ref $Child)`
   - Inside loop: `ref.cast $Child` on the phi value → optimizer should eliminate (already $Child)
   - Loop body: call a function that returns `(ref $Parent)` → could be `$Parent` not `$Child`
   - Backedge feeds the `(ref $Parent)` back to phi
3. After loop: `struct.get` the extra field of `$Child` → if value is actually `$Parent`, reads OOB
4. Run with `--no-liftoff --turboshaft-wasm` to force Turboshaft
5. Compare with `--liftoff` (no type optimization) — if behavior differs, bug found
6. Try with `--turboshaft-loop-unrolling` to trigger graph transformations
7. Try with custom descriptor types to bypass IsCastToCustomDescriptor guard

## DO NOT Try These
- Simple loop with no type narrowing (too basic, well-tested)
- Null checks in loops (covered by genome-019, defended)
- Static type hierarchy without phi merging (no loop analysis involved)

## Evolution Plan
- v1: **Source audit of ProcessPhi and cast elimination interaction**. Read ProcessPhi fully. Trace how the first-iteration type flows to REDUCE(WasmTypeCast). Determine if cast elimination on the first iteration is truly permanent — does the second iteration re-evaluate eliminated casts? Read the fixed-point iteration logic to understand if the graph is modified during analysis or only after.
- v2: **Build loop type widening test**. Create a module where the loop phi forward edge has type `(ref $Child)` and the backedge has type `(ref $Parent)`. Place a `ref.cast $Child` inside the loop. Check if Turboshaft eliminates the cast and allows `$Parent` values through.
- v3: **Loop unrolling + type confusion**. Enable loop unrolling to trigger graph transformations that create unexpected phi patterns (as in 921b3e42db2). Test if the conservative kWasmStructRef fallback is too broad, allowing unrelated struct types through.

## Task
Start with v1. Read these files:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — ProcessPhi (lines 388-427), GetTypeForPhiInput (369-386), RefineTypeKnowledge (584-610), the fixed-point iteration control flow
2. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — REDUCE WasmTypeCast (274-287), how it uses the type analyzer's results
3. `src/compiler/turboshaft/loop-unrolling-reducer.h` — How loop unrolling interacts with phi nodes
4. Check git log for 921b3e42db2 to understand the exact bug and fix

Your type invariant to challenge: "Cast elimination decisions made during type analysis are sound even when loop backedges bring wider types." If the optimizer eliminates a cast on the first iteration (narrow forward-edge type) and the backedge widens the phi type, the eliminated cast is never restored.

Post to #colony-workers with tags `phi,loop,type-analysis,cast-elimination`.
