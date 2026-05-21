# Genome 093 — call-ref-feedback-type

## Hypothesis
call_ref uses type feedback for speculative inlining (turboshaft-graph-interface.cc:2941). When call_ref is called with multiple function types (polymorphic), the feedback tracks which functions were seen. If the optimizer speculatively inlines based on monomorphic feedback but the actual call target changes to a function with a DIFFERENT signature (different param/return types in the GC type hierarchy), the deoptimization frame must correctly restore type state. If deopt doesn't preserve type information, the resumed execution may treat return values with wrong types.

## DNA
### call_ref Feedback-Based Inlining
- **turboshaft-graph-interface.cc:2941**: call_ref speculative inlining entry point
- The optimizer collects feedback about which functions are actually called via call_ref
- If feedback shows monomorphic (always same function), optimizer can speculatively inline
- If the speculation is wrong (different function called), must deoptimize

### Deoptimization Type Safety
- When deopt occurs, the execution must resume in the interpreter or baseline
- The deopt frame must contain correct type information for all live values
- If a value was narrowed by the optimizer (e.g., known to be ref $B after inline check), the deopt frame must use the ORIGINAL wider type (ref $A)
- If the deopt frame uses the narrowed type but the actual value has the wider type → type confusion

### Speculative Inlining + GC Types
- call_ref with `(ref $functype)` where $functype is a function type in the GC type hierarchy
- If $functype has GC-typed parameters or return values, inlining creates type assumptions
- Example: `(type $f1 (func (result (ref $B))))` — inlined version assumes return is ref $B
- If actual function returns ref $A (supertype), the inlined code may skip type checks
- After deopt, the ref $A value is treated as ref $B → field access OOB

### Key Code Locations
- **turboshaft-graph-interface.cc:2941**: call_ref feedback handling
- **turboshaft-graph-interface.cc**: Search for "InlineWasmCall", "speculative", "feedback" — inlining decision and implementation
- **wasm-deopt-data.h/cc**: Deoptimization data for Wasm — what type info is preserved?
- **deoptimizer/deoptimizer-wasm.cc**: Wasm deoptimizer — how does it reconstruct state?
- **wasm/interpreter/**: If deopt goes to interpreter, how does interpreter receive type info?

### Attack Vector
1. Create two function types with compatible but different GC signatures:
   - `(type $f1 (func (param (ref $Base)) (result (ref $Derived))))`
   - `(type $f2 (func (param (ref $Base)) (result (ref $Base))))`
2. Create table of funcref, populate with function matching $f1
3. Warm up call_ref so optimizer sees monomorphic feedback for $f1
4. Optimizer speculatively inlines assuming return type is ref $Derived
5. Switch table entry to function matching $f2 (returns ref $Base)
6. call_ref calls wrong function → deopt
7. After deopt, return value (ref $Base) is treated as ref $Derived → type confusion

### Related Findings
- No prior workers have tested call_ref feedback inlining
- W26 is completely untouched (coverage 0.0)
- Deoptimization is the ONE area where V8 Wasm has historically had type confusion bugs

## Techniques That Work
1. Create GC type hierarchy with shared fields + derived-only fields
2. Create call_ref with funcref typed parameter
3. Warm up with monomorphic target (100+ calls same function)
4. Switch target to different function with different return type
5. Access derived-only field on return value after deopt → crash or wrong value if type confused
6. Force Turboshaft with `--no-liftoff` or warmup
7. Use `--trace-deopt` to monitor deoptimization events
8. Compare with `--liftoff-only` (no speculative inlining, should be correct)

## DO NOT Try These
- call_indirect type checks — different mechanism, well-tested (dead surface)
- Non-GC call_ref (i32/f64 params) — no type hierarchy interaction
- Shared funcref external creation — dead surface
- Table dispatch sig matching — dead surface

## Evolution Plan
- v1: Source trace call_ref feedback inlining in Turboshaft. Read turboshaft-graph-interface.cc around line 2941. How is feedback collected for call_ref? What triggers speculative inlining? Read the inlining implementation — how are return types handled? Read wasm-deopt-data.h/cc — what type information is stored in deopt frames? Is GC type information preserved?
- v2: Build test with polymorphic call_ref. Create two functions with different GC return types. Warm up call_ref with one, switch to other. Check if deopt correctly handles the type change. Access return value fields to detect type confusion. Compare Liftoff vs Turboshaft results.
- v3: Stress test with deep type hierarchy. Create 5+ level hierarchy, call_ref returns different levels. Test rapid target switching to trigger deopt frequently. Look for type confusion in deopt frames with `--trace-deopt`.

## Task
Start by reading:
1. `src/wasm/turboshaft-graph-interface.cc` — search for line 2941 and surrounding context. How does call_ref handle feedback? Search for "CallRef", "InlineWasmCall", "speculative", "feedback". Understand the full inlining pipeline for call_ref.
2. `src/wasm/wasm-deopt-data.h` — what data is stored for Wasm deoptimization? Are Wasm GC types preserved in deopt frames?
3. `src/deoptimizer/deoptimizer-wasm.cc` or similar — how does the deoptimizer reconstruct Wasm state? Does it handle GC types correctly?
4. `src/wasm/function-body-decoder-impl.h` — search for "call_ref". What validation does the decoder do for call_ref? What types does it push on the value stack?
5. `src/compiler/turboshaft/wasm-in-js-inlining-reducer.h` or `wasm-js-lowering-reducer.h` — any cross-boundary inlining for call_ref?

The type invariant to challenge: "When call_ref speculative inlining triggers deoptimization, the deopt frame correctly preserves GC type information so that resumed execution does not have type confusion." If deopt loses type info, the interpreter/baseline may treat values with wrong types.

Use `--no-liftoff --experimental-wasm-gc --trace-deopt` with `/Users/t/v8/out/fuzzbuild/d8`.
