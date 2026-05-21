# Genome 008 — call-indirect-deopt

## Hypothesis
Speculative `call_indirect` inlining (turboshaft-graph-interface.cc:2546-2585, behind `--wasm-inlining-call-indirect`) guards the inlined path by comparing the runtime call target against the expected inlined target. At line 2569, the code comments: "we don't need a dynamic type or null check: If the actual call target (at runtime) is equal to the inlined call target, we know already from the static check on the inlinee that the inlined code has the right signature." This skips type/signature validation when the target matches. If the dispatch table can be mutated between the feedback collection and the speculative execution (e.g., `table.set` from JS), the same code pointer slot could be reused with a different function that has a different signature. The target comparison succeeds (same code pointer), but the inlined code operates on parameters with the wrong types.

## DNA

### Source Facts
- **Speculative inlining entry** (`turboshaft-graph-interface.cc:2546`): Guarded by `v8_flags.wasm_inlining_call_indirect`.
- **Target comparison** (`turboshaft-graph-interface.cc:2569-2571`): `kNeedsTypeOrNullCheck = false`. The BuildIndirectCallTargetAndImplicitArg is called without type/null check.
- **Feedback cases** (`turboshaft-graph-interface.cc:2574`): `base::Vector<InliningTree*> feedback_cases = GetFeedbackCases(decoder)` — uses collected feedback to decide which targets to inline.
- **Slowpath** (`turboshaft-graph-interface.cc:2577-2578`): "The slow path is the non-inlined generic `call_indirect`, or a deopt node if that is enabled."
- **Recent commits**:
  - `409490cdd15 [wasm][turboshaft] Remove back-edge from call_indirect error handler` — error handler changes
  - `adc761a24dc [wasm] Improve test and bailout for LazyDeoptOnThrow` — deopt improvements
  - `954a5091cc5 [wasm] Check for invalid code pointer in feedback` — feedback validation (security fix!)
- **Feedback validation** (`954a5091cc5`): This commit checks for invalid code pointers in feedback. This suggests invalid code pointers in feedback were a real issue.
- **Table mutation**: Wasm tables can be mutated via `table.set` from JS or from Wasm. If feedback was collected for a function at table index N, and then table.set changes index N to a different function, the inlined code may still be for the old function.

### Key Attack Vectors
1. **Table mutation after feedback collection**: Collect feedback for table[0] = funcA (sig (i32)->i32). Then table.set(0, funcB) where funcB has sig (f64)->f64. On next call_indirect through table[0], the speculative path compares targets. If the comparison is against the code pointer (not the full function reference), and funcB has a different code pointer, the comparison fails and falls through to slowpath (safe). But if funcB reuses funcA's code slot (unlikely but possible after GC/relocation), the comparison succeeds and wrong types are used.
2. **Signature subtyping confusion**: call_indirect validates signatures at the table level. If the table allows any funcref, and the feedback says "always function X", the inlined code assumes X's signature. But a subtype of X with different parameter types could be placed in the table.
3. **Deopt frame type confusion**: When the speculative path deoptimizes (target mismatch), the deopt frame must correctly represent the Wasm state. If the deopt frame encodes values with wrong types (e.g., an i32 encoded as f64), the resumed execution operates on corrupted values.
4. **Polymorphic dispatch**: With `kMaxPolymorphism` cases, the dispatch compares against multiple known targets. If a new target matches one of the known targets' code pointers but has a different signature, the wrong inlined code runs.

### Guards Expected
- Target comparison (code pointer equality) should prevent signature confusion
- `954a5091cc5` validates code pointers in feedback
- call_indirect always validates signature at the table level (before inlining kicks in)
- Deopt frames should be type-safe

## Techniques That Work
1. Read the full speculative call_indirect path in turboshaft-graph-interface.cc:2546-2730
2. Read the deopt frame construction for Wasm (search for LazyDeopt, DeoptFrame, WasmDeopt)
3. Read commit 954a5091cc5 to understand what "invalid code pointer in feedback" means
4. Construct a test: create a table, collect feedback with funcA, mutate table to funcB (different sig), call again
5. Test with `--wasm-inlining-call-indirect --wasm-inlining` flags

## DO NOT Try These
- Testing without inlining flags (speculative inlining won't activate)
- Testing with identical signatures (no type confusion possible)
- Testing with direct calls (different code path, not speculative)

## Evolution Plan
- v1: Read the speculative call_indirect path in full. Understand: (a) what is compared to guard the speculative path, (b) how the deopt/slowpath is constructed, (c) what type information is carried into the inlined code, (d) what commit 954a5091cc5 fixed.
- v2: If v1 finds the guard compares code pointers only (not signatures), test if table mutation + code pointer reuse can bypass the guard. If the guard also validates signatures, test if signature subtyping creates a gap (sig with nullable param vs non-nullable param).
- v3: If v2 finds a bypass, construct an exploit where the inlined code operates on values with wrong types. If v2 is clean, investigate the deopt frame construction — does it correctly restore types after deoptimization from speculative Wasm code?

## Task
Start by reading these files in order:
1. `src/wasm/turboshaft-graph-interface.cc` lines 2546-2730 (full speculative call_indirect path)
2. Run `git -C /Users/t/v8 show 954a5091cc5` to understand the feedback validation fix
3. Search for `LazyDeoptOnThrow` or `WasmDeoptData` in `src/wasm/` and `src/compiler/turboshaft/` to understand deopt frame construction
4. `src/wasm/turboshaft-graph-interface.cc` lines 2730-2852 (second half of call_indirect, handling after inlining)
5. Search for `BuildIndirectCallTargetAndImplicitArg` to understand target extraction

The type invariant to challenge: "Speculative call_indirect inlining is safe because the target comparison guarantees the inlined code has the correct signature."

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc --wasm-inlining --wasm-inlining-call-indirect --no-wasm-lazy-compilation`
