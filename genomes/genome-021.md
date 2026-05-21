# Genome 021 — call-indirect-speculative

## Hypothesis
Speculative inlining of `call_indirect` in Turboshaft (turboshaft-graph-interface.cc:2563-2569) explicitly sets `kNeedsTypeOrNullCheck = false` when the call target matches the feedback-collected target. The assumption: "If the actual call target (at runtime) is equal to the inlined call target, we know already from the static check on the inlinee that the inlined code has the right signature."

This assumption can break if:
1. The dispatch table entry is MODIFIED between feedback collection and optimized execution (via `table.set`, `table.grow`, or import patching)
2. Two different functions at different indices happen to have the same code pointer but different canonical signatures (collision after JIT compilation)
3. The feedback is collected for one instance but the optimized code runs in a different instance sharing the same dispatch table (shared tables with `--experimental-wasm-shared`)

If the signature check is skipped and the actual function has a different signature, parameters are interpreted with wrong types — a direct type confusion. Parameters passed as `i32` could be read as `f64` or `ref`, or vice versa.

The guard at line 2641-2659 (`InlineTargetIsTypeCompatible`) validates the inlined target's signature BEFORE inlining, but this validation happens at compile time against the feedback target. If the dispatch table changes at runtime, the validation is stale.

Multi-factor: Speculative inlining + dispatch table mutation + signature check skip + parameter type confusion.

## DNA

### Source Facts

**Speculative inlining check skip** (turboshaft-graph-interface.cc:2563-2569):
```cpp
// In particular, we don't need a dynamic type or null check: If the
// actual call target (at runtime) is equal to the inlined call target,
// we know already from the static check on the inlinee (see below) that
// the inlined code has the right signature.
constexpr bool kNeedsTypeOrNullCheck = false;
auto [target, implicit_arg] = BuildIndirectCallTargetAndImplicitArg(
    decoder, index_wordptr, imm, kNeedsTypeOrNullCheck);
```

**Normal call_indirect signature check** (turboshaft-graph-interface.cc:8151-8227):
- Loads expected and actual signatures from dispatch table
- Compares for equality (final types) or subtype check (non-final)
- Traps on mismatch with TrapId::kTrapFuncSigMismatch
- Recent fix (409490cdd15) added `__ Unreachable()` after trap — previous code had control flow issue

**Dispatch table entry layout** (wasm-objects.h:824-872):
```cpp
static constexpr size_t kTargetBias = 0;      // WasmCodePointer (uint32_t)
static constexpr size_t kSigBias = 4;         // Canonical type index (uint32_t)
static constexpr size_t kImplicitArgBias = 8;  // Protected pointer
static constexpr size_t kEntrySize = 8 + kTaggedSize;
```

**Dispatch table growth bug** (dc31719c947, bug 483220222):
- Old table's protected_uses not cleared on growth → dangling reference
- This was a SANDBOX ESCAPE
- Shows dispatch table lifecycle has had real bugs

**InlineTargetIsTypeCompatible guard** (turboshaft-graph-interface.cc:2641-2659):
- Static check at compile time, not runtime
- Validates the FEEDBACK target signature
- Does NOT protect against runtime dispatch table changes

**Polymorphism limit** (wasm-constants.h:240):
- `kMaxPolymorphism = 4` — up to 4 inline targets per call site
- Beyond 4, falls back to slowpath with full signature check

### Key Attack Angles
1. **table.set mutation**: Set up feedback with function A (sig `(i32) -> i32`), then `table.set` to function B (sig `(f64) -> f64`). Optimized code inlines A, skips sig check, but table now has B.
2. **table.grow reallocation**: Grow the table causing reallocation. Old entries might be at different offsets.
3. **Shared dispatch table**: Two instances sharing a dispatch table. Instance 1 collects feedback, instance 2 modifies the table.
4. **Import re-instantiation**: Table imported from another module. Re-instantiate the source module with different function at same index.
5. **Tiering race**: Function compiles with Liftoff first (collects feedback), then Turboshaft compiles and inlines. Between feedback collection and Turboshaft execution, table.set changes the entry.

### Related Code Locations
- `src/wasm/turboshaft-graph-interface.cc` — Speculative inlining (2563-2569), BuildIndirectCallTargetAndImplicitArg (8151-8227), InlineTargetIsTypeCompatible (2641-2659)
- `src/wasm/wasm-objects.cc` — WasmDispatchTable::Set, WasmDispatchTable::Grow
- `src/wasm/wasm-objects.h` — Dispatch table layout
- `src/wasm/baseline/liftoff-compiler.cc` — CallIndirectImpl (10087-10196), feedback collection
- `src/wasm/module-instantiate.cc` — Dispatch table population

## Techniques That Work
1. Create module with two function signatures: `sig_a = (i32) -> i32` and `sig_b = (f64) -> f64`
2. Create functions `func_a: sig_a` and `func_b: sig_b`
3. Create a table of type `funcref`, initialized with `func_a` at index 0
4. Create a hot function that does `call_indirect sig_a [0]` in a loop (collects feedback for func_a)
5. After Liftoff compilation and feedback collection, use `table.set 0 func_b` to swap the entry
6. Run the hot function again — Turboshaft should speculatively inline func_a, skip sig check
7. But the actual target is now func_b with different signature
8. The runtime equality check (`target == inlined_target`) should catch this... OR should it?
9. Test: does the target pointer change when table.set replaces the entry? If func_a and func_b happen to compile to the same address...
10. Use `--no-liftoff --turboshaft-wasm` flags, also `--wasm-inlining` and `--wasm-speculative-inlining`

## DO NOT Try These
- Basic call_indirect without table mutation (signature checks are well-tested)
- call_ref type confusion (call_ref doesn't do runtime sig checks by design, trusts type system)
- Cross-module table sharing without mutation (already tested by cross-module-exact)

## Evolution Plan
- v1: **Source audit of speculative inlining safety**. Read the full speculative inlining path in turboshaft-graph-interface.cc. Determine: (a) what exactly is the runtime guard for speculative inlining (target equality?), (b) is the guard sufficient to prevent wrong-signature execution, (c) what happens when the guard fails (deopt? trap? slowpath?). Read the feedback collection in Liftoff to understand what's stored.
- v2: **Build table mutation test**. Create module with table.set that changes dispatch table entry between feedback collection and optimized execution. Verify if speculative inlining correctly deoptimizes when target changes.
- v3: **Shared table mutation test**. Use shared dispatch tables (if available) where one thread modifies the table while another thread's optimized code uses the feedback.

## Task
Start with v1. Read these files:
1. `src/wasm/turboshaft-graph-interface.cc` — Search for "speculative" and "InlineTargetIsTypeCompatible". Read the full speculative inlining code path around lines 2550-2700.
2. `src/wasm/turboshaft-graph-interface.cc` — BuildIndirectCallTargetAndImplicitArg around lines 8150-8230. Understand the runtime signature check.
3. `src/wasm/baseline/liftoff-compiler.cc` — CallIndirectImpl around lines 10087-10200. How is feedback collected? What's stored?
4. `src/wasm/wasm-objects.cc` — WasmDispatchTable::Set. What happens when a table entry is replaced?

Your type invariant to challenge: "Speculative inlining of call_indirect is safe because the runtime target equality check prevents wrong-signature execution." If the target check passes but the signature has changed (same function object, different compilation, or target collision), parameters are misinterpreted.

Post to #colony-workers with tags `call-indirect,speculative,inlining,signature`.
