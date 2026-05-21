# Genome 125 — call-indirect-deopt-type

## Hypothesis
When call_indirect is speculatively inlined based on type feedback from one instance, then the same code is used for a different instance with a different type hierarchy, the deoptimization frame may reconstruct values with wrong types. Multiple recent fixes prove this area is extremely fragile:
- a497308a56b: Fix deopt loop with cross-instance call_indirect
- 7d2a6e8277a: Check instance before inlined call_indirect code
- d1df236102b: Fix TransitiveTypeFeedbackProcessor for multi-instance deopt feedback
- d62c19a7ab2: Fix deopt handling in case of too many inputs
- 72c9d556f93: Fix handling InstanceCache in return_call_indirect

If a function is optimized with feedback from instance A (where table[0] has signature (ref $A_struct) -> i32), and then called from instance B (where table[0] has signature (ref $B_struct) -> i32), and $A_struct and $B_struct have different field layouts but share a canonical signature, the deopt frame may interpret a $B_struct as a $A_struct.

## DNA
- **d1df236102b**: TransitiveTypeFeedbackProcessor fix for multi-instance deopt feedback. The processor was incorrectly propagating type feedback across instances.
- **7d2a6e8277a**: Instance check was MISSING before inlined call_indirect code. This means the optimized code could run with the wrong instance's types.
- **a497308a56b**: Deopt LOOP — the deopt triggered by cross-instance call led to re-optimization with the same wrong feedback, creating an infinite deopt loop. Fix broke the loop but the underlying type confusion may have siblings.
- **409490cdd15**: Removed back-edge from call_indirect error handler — recent cleanup suggests ongoing work.
- **Speculative inlining**: V8 uses feedback vectors to record which function is typically called at a call_indirect site. Turboshaft inlines the target function body. On deopt, the frame must be reconstructed with correct types.
- **Canonical signatures**: Two modules can have functions with the same canonical signature but different GC ref types as parameters. The canonical signature uses CanonicalValueType which includes the canonical index — so same canonical sig means same canonical types. But the module-local type IDs may differ.

## Techniques That Work
1. Create two modules (A and B) with identical function signatures but different struct layouts
2. Module A: struct with fields {i32, i64}. Module B: struct with fields {i32, ref $other}
3. Both export a function with the same canonical signature via table
4. Import both into a driver module using call_indirect on a shared table
5. Call with A's function many times (build feedback), then switch to B's function
6. The deopt frame may contain A's struct being treated as B's struct (or vice versa)
7. Especially dangerous: if A's i64 field is reinterpreted as B's ref field — controlled pointer

## DO NOT Try These
- Shared types (--experimental-wasm-shared, BANNED)
- Single-module call_indirect — cross-instance is the attack surface
- Functions with completely different signatures — canonical signatures must match for speculative inlining

## Evolution Plan
- v1: Source analysis of the speculative inlining pipeline. Read: (a) how feedback is collected for call_indirect (wasm-compiler.cc or turboshaft equivalent), (b) how the TransitiveTypeFeedbackProcessor works (search for it in src/wasm/), (c) the instance check fix at 7d2a6e8277a — what exactly was missing and is the fix complete?
- v2: Build a cross-instance test. Two modules share a table via JS. Module A's function takes/returns a struct type. Module B's function takes/returns a different struct type but with the same canonical signature. Call via call_indirect, trigger tier-up, then switch the table entry. Test if deopt correctly handles the type change.
- v3: If v2 is clean, escalate with return_call_indirect (tail call variant). The fix at 72c9d556f93 handled InstanceCache for return_call_indirect but return_call_ref had a separate fix — test if return_call_indirect's deopt frame is correctly constructed after the tail call optimization.

## Task
Start by reading:
1. `git -C /Users/t/v8 show 7d2a6e8277a` — the full diff of the instance check fix. Understand EXACTLY what check was missing and where it was added.
2. `git -C /Users/t/v8 show d1df236102b` — the TransitiveTypeFeedbackProcessor fix. What was wrong with multi-instance feedback?
3. `git -C /Users/t/v8 show a497308a56b` — the deopt loop fix. How was the loop broken?
4. Search for `TransitiveTypeFeedback` in src/wasm/ to find the feedback processor.
5. Read `src/compiler/turboshaft/wasm-in-js-inlining-reducer-inl.h` for how inlined Wasm bodies handle deopt.

Then build a test:
- Two modules with same function signature but different struct types
- Shared table via JS
- Hot loop with call_indirect triggering tier-up
- Switch table entry mid-execution
- Check if struct field accesses produce wrong values after deopt

Run with: `/Users/t/v8/out/fuzzbuild/d8 test.js`
No --experimental-wasm-shared flag.
