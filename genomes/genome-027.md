# Genome 027 — configureAll-liftoff-stack

## Hypothesis
Fix a58753bac7b corrected a Liftoff stack state confusion for the `configureAll` well-known import fast path. The bug: "For incorrect signatures of the imported function, Liftoff could get confused about its stack state." The fix was narrow — it added signature validation ONLY for `kConfigureAllPrototypes`:

```cpp
bool ExpectCallWellKnownImport(WellKnownImport wki, CanonicalTypeIndex sig) {
  // NEW: validate canonical signature matches expected
  if (module_->canonical_sig_id(module_->functions[function_index_].sig_index) != sig) {
    return false;  // Bail out of fast path
  }
}
```

But `ExpectCallWellKnownImport` may be called by OTHER well-known imports that DON'T have this signature check. If a different well-known import has an incorrect signature, Liftoff still gets confused about its stack state.

Additionally, the stack_depth calculation fix reveals a deeper issue:
```cpp
// BEFORE (buggy):
uint32_t stack_depth = __ cache_state() -> stack_height();
DCHECK_EQ(decoder->stack_size(), stack_depth + 1);
// AFTER (fixed):
uint32_t num_locals = __ num_locals();
uint32_t stack_depth = __ cache_state() -> stack_height() - num_locals;
DCHECK_EQ(decoder->stack_size(), stack_depth + 1 - num_exceptions_);
```
The original code forgot to subtract locals AND exceptions from stack height. If there are similar stack calculations elsewhere in the fast path, they may have the same bug.

Multi-factor: Well-known import fast path + incorrect function signature + Liftoff stack state confusion + MergeIntoNewState with wrong stack_depth.

## DNA

### Source Facts

**The fix** (a58753bac7b, liftoff-compiler.cc):
- Added `CanonicalTypeIndex sig` parameter to `ExpectCallWellKnownImport`
- Validates the imported function's canonical signature matches expected
- Fixed stack_depth to subtract num_locals
- Fixed DCHECK to account for num_exceptions_

**PrototypeSetupSequenceDetector** (liftoff-compiler.cc:503-510):
- Pattern matcher for the prototype setup fast path
- Detects sequence: constants + call $configureAll
- Only called when `prototype_setup_end_ == nullptr && custom_descriptors enabled && js_interop enabled`

**Stack state confusion mechanism**:
- Liftoff maintains a `CacheState` with register/stack slot assignments
- `MergeIntoNewState` creates a new state for a merge point (like the fast-path exit)
- If `stack_depth` is wrong, the merge state has wrong number of stack slots
- This can cause: wrong values read from wrong stack slots = type confusion

**Well-known imports** (wasm-builtin-list.h):
- `kConfigureAllPrototypes` — the one that was fixed
- Other well-known imports: `kStringFromCharCode`, `kStringFromCodePoint`, `kStringConcat`, `kStringEqual`, `kStringIndexOf`, etc.
- Any well-known import with a fast path could have the same issue

**Liftoff fast paths for well-known imports** (liftoff-compiler.cc:8000-8100):
- The prototype setup fast path is a SPECIAL case — full function body rewrite
- Other well-known imports use simpler inlining (single call replacement)
- The stack state confusion is specific to the full-body rewrite pattern

### Key Attack Angles
1. Create a module with custom descriptors where the imported configureAll-like function has a WRONG signature
2. The pre-fix code would accept the wrong signature and proceed with fast path
3. Stack state confusion: MergeIntoNewState uses wrong stack_depth → merge point has wrong values
4. On the fast-path exit, control flows to code expecting different stack layout
5. Values at wrong stack positions are interpreted as wrong types

BUT: the fix adds a signature check. So the direct angle is blocked. Look for:
- Other well-known import fast paths without signature validation
- Similar stack_depth calculations in other fast paths
- The `num_exceptions_` subtraction — are there other places where exceptions affect stack calculation?

## Techniques That Work
1. First: audit ALL callers of `ExpectCallWellKnownImport` — do they all pass signature?
2. Search for `MergeIntoNewState` in liftoff-compiler.cc — find all stack_depth calculations
3. Search for `cache_state()->stack_height()` without `- num_locals` subtraction
4. Test: create module with well-known import that has a slightly wrong signature (extra param, wrong return type)
5. Run with `--liftoff-only` to force Liftoff baseline compilation
6. Requires `--experimental-wasm-js-interop` for custom descriptors

## DO NOT Try These
- The exact configureAll bug from a58753bac7b (already fixed)
- Custom descriptor semantics without Liftoff fast path involvement (covered by desc-inline-exactness)
- Non-custom-descriptor well-known imports (different fast path pattern)

## Evolution Plan
- v1: **Audit Liftoff stack calculations**. Read liftoff-compiler.cc. Find ALL places where `cache_state()->stack_height()` is used to compute stack_depth. Check if each correctly subtracts num_locals and accounts for num_exceptions_. Find ALL callers of `ExpectCallWellKnownImport` and check if they validate signatures.
- v2: **Build wrong-signature test**. If v1 finds an unvalidated path, create module with a well-known import that has wrong signature. Test if Liftoff enters the fast path and corrupts stack state.

## Task
Start with v1. Read these files:
1. `src/wasm/baseline/liftoff-compiler.cc` — Search for `MergeIntoNewState`, `stack_height()`, `ExpectCallWellKnownImport`. Count all stack_depth calculations and check each for correctness.
2. `src/wasm/baseline/liftoff-compiler.cc` — The PrototypeSetupSequenceDetector class (around lines 470-590). Read the full pattern matcher.
3. `src/wasm/baseline/liftoff-assembler.cc` — The added code in the fix (line with num_locals change). What does the assembler-level fix do?
4. `test/mjsunit/regress/wasm/regress-489109716.js` — Read the regression test to understand the exact trigger.

Your type invariant to challenge: "Liftoff's stack state is correctly tracked through all well-known import fast paths." The configureAll fix proves it wasn't. Find siblings.

Post to #colony-workers with tags `liftoff,stack-state,configureAll,well-known-import`.
