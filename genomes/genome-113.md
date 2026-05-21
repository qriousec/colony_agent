# Genome 113 — configureall-stack-confusion

## Hypothesis
Bug fix a58753bac7b revealed that Liftoff's `PrototypeSetupSequenceDetector` could get confused about stack state when `configureAll` was called with an incorrect function signature. The fix added: (1) canonical signature validation in `ExpectCallWellKnownImport`, (2) subtraction of `num_locals` from `stack_height()`, (3) subtraction of `num_exceptions_` in DCHECK. If the exception counter `num_exceptions_` is tracked incorrectly (e.g., for nested try-catch around configureAll), or if `MergeIntoNewState` mishandles ref-typed locals during the fast-path jump, the compiled code operates on a corrupted stack — treating a ref as an i32 or vice versa.

## DNA

### Source Facts (file:line)
- **Fix commit**: a58753bac7b — "Fix configureAll fast path: For incorrect signatures of imported function, Liftoff could get confused about its stack state"
- **Regression test**: test/mjsunit/regress/wasm/regress-489109716.js — wrong-signature import
- **PrototypeSetupSequenceDetector**: liftoff-compiler.cc:~480-600 — bytecode pattern matcher that detects the configureAll idiom
- **Signature check added**: liftoff-compiler.cc:503-508 — `ExpectCallWellKnownImport(WKI::kConfigureAllPrototypes, kPredefinedSigIndex_configureAll)`
- **Stack depth fix**: liftoff-compiler.cc:8083-8085 — `stack_height() - num_locals` replaces `stack_height()`
- **DCHECK fix**: liftoff-compiler.cc:8085 — `decoder->stack_size() == stack_depth + 1 - num_exceptions_`
- **MergeIntoNewState**: liftoff-compiler.cc:8087 — `MergeIntoNewState(num_locals, 0, stack_depth)` — creates continuation state
- **WKI runtime check**: liftoff-compiler.cc:8030 — loads WellKnownImports at runtime to verify import identity
- **Fast path emission**: liftoff-compiler.cc:7997-8090 — entire configureAll fast path
- **Flag guard**: liftoff-compiler.cc:7999-8000 — `decoder->enabled_.has_custom_descriptors() && v8_flags.experimental_wasm_js_interop`

### Related CVEs/Bugs
- Bug 489109716 (fixed by a58753bac7b) — wrong signature causes stack confusion
- Custom descriptor feature is experimental (`--experimental-wasm-js-interop`)

### Guard Inventory
1. `ExpectCallWellKnownImport` validates function_index < num_imported_functions AND canonical sig matches
2. Runtime WKI check at line 8030 verifies import identity dynamically
3. `has_custom_descriptors() && experimental_wasm_js_interop` flag check

### Gap Analysis
1. **num_exceptions_ tracking**: Where is `num_exceptions_` incremented/decremented? If a try-catch block wraps the configureAll call pattern, does the detector account for the exception handler's stack effect?
2. **MergeIntoNewState with ref locals**: The merge creates a new register state. If locals include WasmGC ref types, the merge must preserve their liveness for GC. Does it?
3. **Signature check bypass via type aliasing**: The check uses `canonical_sig_id`. If two functions have different signatures that canonicalize to the same ID (e.g., via recursive type groups), the check could pass incorrectly.
4. **Race between compilation and import change**: The WKI check is at runtime, but the fast path code was compiled assuming a specific stack layout. If the import changes between compile time and runtime, the fallback path must restore stack correctly.

## Techniques That Work
1. Construct a module with `configureAll` pattern where the imported function has a subtly wrong signature (extra param, wrong ref type)
2. Wrap the configureAll call in try-catch to manipulate `num_exceptions_`
3. Use ref-typed locals (struct refs, array refs) to maximize damage from stack corruption
4. Test tier-up: Liftoff compiles fast path, then tier-up to Turboshaft — does the fast path state survive?
5. Provide an import that IS `configureAll` at instantiation but is replaced via import mutation

## DO NOT Try These
- Simple wrong-signature test — already covered by regress-489109716.js
- Testing without `--experimental-wasm-js-interop` flag — feature is gated

## Evolution Plan
- v1: Source audit of `num_exceptions_` tracking in Liftoff compiler. Trace where it's incremented (Catch/CatchAll handlers) and verify it's correct when try-catch wraps the configureAll sequence. Build test with nested try-catch + configureAll + ref locals.
- v2: Test MergeIntoNewState with complex ref-typed locals (shared structs, nested arrays, i31 refs). Verify GC map correctness after the fast-path merge. Force GC between the merge point and the continuation.
- v3: Investigate canonical signature aliasing — can two different signatures (one correct for configureAll, one with extra/wrong params) share the same canonical ID? Build test with recursive type groups to trigger aliasing.

## Task
1. Start by reading liftoff-compiler.cc:7990-8100 (the entire configureAll fast path).
2. Then read liftoff-compiler.cc:480-600 (PrototypeSetupSequenceDetector) to understand the pattern matching.
3. Search for `num_exceptions_` in liftoff-compiler.cc to find all increment/decrement sites.
4. Search for `MergeIntoNewState` to understand how it handles ref types in the merge.
5. Build test module: configureAll inside try-catch, with ref-typed locals, wrong-ish signature.
6. Run with `--experimental-wasm-js-interop --experimental-wasm-custom-descriptors --no-wasm-lazy-compilation`.
7. Diff Liftoff vs Turboshaft output. Focus on stack state after the configureAll call returns.
8. Start with Evolution Plan v1 (num_exceptions_ tracking).
