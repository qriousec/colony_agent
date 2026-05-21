# Genome 042 — call-indirect-type-deopt

## Hypothesis
Speculative call_indirect inlining in Turboshaft (turboshaft-graph-interface.cc:2543-2730) sets `kNeedsTypeOrNullCheck = false` at line 2569. This SKIPS the runtime signature check from the dispatch table. The static signature check at line 2642 (`InlineTargetIsTypeCompatible`) only compares the inlinee's compile-time signature, not the runtime table entry. The deopt check at line 2666 compares code pointers but NOT type signatures.

Attack vector: A call_indirect through a table that is mutated after feedback collection. The feedback records function F (correct type). The table is then mutated to contain function G (different type). The deopt check at line 2672 catches if `target != inlined_target` (code pointer differs). But what if:
1. The same function is re-wrapped with a different canonical signature (same code, different type)?
2. Cross-instance table import shares the dispatch table but with different type context?
3. The inlining threshold allows polymorphic dispatch where one case has the wrong type?

Multi-factor: call_indirect + speculative inlining + feedback vector + table mutation + type check skip (`kNeedsTypeOrNullCheck = false`).

## DNA

### Critical Code Paths

**Inlining decision** (turboshaft-graph-interface.cc:2543-2730):
- Line 2546: Flag `v8_flags.wasm_inlining_call_indirect` gates feature
- Line 2569: `constexpr bool kNeedsTypeOrNullCheck = false` ← TYPE CHECK SKIPPED
- Line 2574: Gets feedback cases from inlining decisions
- Lines 2642-2659: `InlineTargetIsTypeCompatible()` — STATIC check only
- Line 2666: `DeoptIfNot()` for code pointer equality (last case)
- Line 2672: `Word32Equal(target, inlined_target)` branch check

**Target loading WITHOUT type check** (turboshaft-graph-interface.cc:8099-8247):
- Lines 8136-8234: Type checking logic — SKIPPED when kNeedsTypeOrNullCheck=false
- Lines 8237-8239: Loads target from dispatch table
- Lines 8240-8244: Loads implicit_arg from dispatch table

**Feedback collection** (module-compiler.cc:1261-1302):
- Line 1287: `CHECK_LT(untrusted_code_pointer, kIndexSpaceSize)` — bounds only, no type check
- Lines 1293-1298: Validation checks (cross-instance, anonymous)

**Feedback IC** (wasm.tq:772-881):
- UpdateCallRefOrIndirectIC: stores target pointer in feedback vector, NO type check
- CallIndirectIC: returns target + implicit_arg from dispatch table

**Static type compatibility** (turboshaft-graph-interface.cc:9078-9092):
- `InlineTargetIsTypeCompatible()`: compile-time subtype check on signatures
- Line 9106: `SBXCHECK(InlineTargetIsTypeCompatible(...))` — sandbox check

### Recent Security Fix
- Commit 954a5091cc5: "Check for invalid code pointer in feedback" — added bounds check on code pointer from feedback. Suggests feedback vector is UNTRUSTED (comes from JS heap).

### Key Flags
```
--wasm-inlining-call-indirect (enables call_indirect speculative inlining)
--wasm-inlining (enables general Wasm inlining)
--wasm-inlining-budget=N (control inlining budget)
```

### Grep Commands
```bash
grep -rn "wasm_inlining_call_indirect" src/flags/flag-definitions.h | head -5
grep -rn "kNeedsTypeOrNullCheck" src/wasm/turboshaft-graph-interface.cc | head -5
grep -rn "InlineTargetIsTypeCompatible" src/wasm/turboshaft-graph-interface.cc | head -10
grep -rn "CallIndirectIC\|UpdateCallRef" src/builtins/wasm.tq | head -10
grep -rn "ExtractCallTargetsFeedback" src/wasm/module-compiler.cc | head -5
```

## Techniques That Work
1. Build module with a table and call_indirect
2. Fill table with function of type A (e.g., i32→i32)
3. Call call_indirect many times to collect feedback
4. Mutate table entry (via table.set from Wasm or JS) to function of type B (e.g., f64→f64)
5. Call call_indirect again — does the inlined type A code run on type B arguments?
6. Run with `--wasm-inlining-call-indirect` flag enabled
7. Compare: with inlining enabled vs disabled (`--no-wasm-inlining`)

## DO NOT Try These
- call_ref (different mechanism, different feedback path)
- Direct calls (no dispatch table, no type confusion possible)
- call_indirect without table mutation (no type mismatch)

## Evolution Plan
- v1: **Source audit**. Read turboshaft-graph-interface.cc:2543-2730 to understand the exact inlining flow. Confirm `kNeedsTypeOrNullCheck = false`. Map: what exactly happens when the table entry changes? Does deopt trigger? If deopt triggers, does the fallback path do a type check?
- v2: **Build test module**. Module with table, two functions with different signatures, call_indirect through table. Fill table with func_A, call many times, then table.set to func_B, call again. Run with `--wasm-inlining-call-indirect`. Check for type confusion or crash.
- v3: **Cross-instance table sharing**. Module A exports a table. Module B imports it with a different type expectation. Module A fills the table and calls call_indirect. Module B mutates the table. Does Module A's inlined code break?

## Task
Start with v1. Read the source code at the exact lines referenced above. Focus on understanding:
1. WHY is `kNeedsTypeOrNullCheck` set to false? Is there a comment explaining the safety argument?
2. When the deopt triggers (code pointer mismatch), what happens? Slow path? Non-inlined call_indirect? Does THAT path do a type check?
3. Is there a way to get a different function (different type) with the SAME code pointer?

Then build and run the test module from v2. Use `--wasm-inlining-call-indirect --turboshaft-wasm --allow-natives-syntax`.

Post with tags `call-indirect,speculative-inlining,feedback,type-check,deopt,w6`.
