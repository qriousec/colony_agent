# Genome 072 — custom-desc-setup-stack

## Hypothesis
When Liftoff's PrototypeSetupSequenceDetector (liftoff-compiler.cc:452) pattern-matches a function body for the configureAll fast path, the matcher records 9 constants and emits a fast-path sequence with stack merging. The recent fix (a58753bac7b) corrected signature validation and stack height calculation (subtracting num_locals), but the DCHECK at line 8090 `decoder->stack_size() == stack_depth + 1 - num_exceptions_` reveals that exception handlers alter the equation. If a function has try/catch blocks that modify num_exceptions_ before the array.new_data + call sequence, the stack merge at MergeIntoNewState may compute wrong stack_depth, causing the fast path to merge with a state where stack slot types don't match the slow path — type confusion between stack values.

## DNA

### Source Facts
- **PrototypeSetupSequenceDetector** (liftoff-compiler.cc:452-512): Pattern-matches byte sequence: struct.new calls, array.new_data, global.get, call $configureAll. Relies on strict ordering.
- **Fix a58753bac7b** added two guards:
  1. Signature validation via `TypeCanonicalizer::kPredefinedSigIndex_configureAll` at line 507-508
  2. Stack height fix: `stack_depth = cache_state()->stack_height() - num_locals` at line 8087-8088 (was missing `- num_locals`)
  3. Added `has_custom_descriptors() && js_interop` guards at line 8001-8002
- **DCHECK at line 8090**: `decoder->stack_size() == stack_depth + 1 - num_exceptions_` — accounts for exception handlers
- **MergeIntoNewState** (line 8091-8092): Creates merge state for fast path → slow path join
- **DropValues(2)** at line 8083-8084: Drops 2 constants from stack. If locals pushed these, wrong values dropped.
- **FastPathEndState** (line 10805): Stores label + state for delayed merge at the call instruction (line 4506-4516)
- **Stack steal** at line 4513: `cache_state()->Steal(prototype_setup_end_->state)` — replaces current stack state entirely
- **num_exceptions_** field: Tracks try/catch depth, affects decoder->stack_size() but NOT Liftoff's cache_state()->stack_height()

### Related Fixes
- a58753bac7b: Stack confusion fix for incorrect configureAll signatures
- Bug 489109716: Original crash from confused stack state

### Key Guards
- Signature check at ExpectCallWellKnownImport (line 576-584)
- Feature flag guards (line 8000-8002)
- DCHECK on stack size (line 8090) — only in debug builds!

### Gaps Identified
1. DCHECK at line 8090 only fires in debug builds — release builds may silently have wrong stack state
2. The `num_exceptions_` subtraction suggests exception handling interacts with this code but isn't validated by the matcher
3. DropValues(2) assumes exactly 2 constants on top of stack, but if matcher partially matches a different sequence, different values could be on top
4. The matcher is a Decoder subclass that reads raw bytes — it doesn't validate control flow or exception handling context

## Techniques That Work
1. Build a Wasm module with custom descriptors that has the exact byte pattern for configureAll fast path
2. Add try/catch blocks BEFORE the matched sequence to alter num_exceptions_
3. Use locals of different types to shift stack layout
4. Test in release build where DCHECK doesn't fire
5. Compare Liftoff output with TurboFan (which doesn't have this optimization) for differentials

## DO NOT Try These
- Testing IsCastToCustomDescriptor guard — exhaustively covered by 4 workers, confirmed DEAD
- Testing exactness kExactMatchOnly vs kExactMatchLastSupertype — confirmed semantically equivalent

## Evolution Plan
- v1: Map the PrototypeSetupSequenceDetector matcher precisely — what byte sequences does it accept? Then construct a module where try/catch wraps the matched region, modifying num_exceptions_ before MergeIntoNewState runs. Test Liftoff vs TurboFan differential.
- v2: Test with functions that have locals of ref types (not just i32) so that the DropValues(2) drops a ref-typed value instead of an i32 constant. If the fast path drops refs that the slow path expects as i32 constants (or vice versa), the merged state has type-confused stack slots.
- v3: Test edge case where the matcher partially matches (e.g., two array.new_data calls in one function) — does the first partial match corrupt state for the second real match?

## Task
Start by reading these files in order:
1. `src/wasm/baseline/liftoff-compiler.cc:452-512` — Full PrototypeSetupSequenceDetector class
2. `src/wasm/baseline/liftoff-compiler.cc:7993-8099` — ArrayNewSegment handler with fast path
3. `src/wasm/baseline/liftoff-compiler.cc:4500-4520` — Call handler where fast path merges
4. `src/wasm/baseline/liftoff-assembler.h` — MergeIntoNewState, DropValues, cache_state API

Name the type invariant to challenge: **Liftoff's stack state at the fast-path/slow-path merge point must have identical types in all stack slots.** If the fast path computes stack_depth incorrectly (due to exception handlers or locals), the merged state has type-confused slots.

Build a test module using WasmModuleBuilder with:
- `--experimental-wasm-custom-descriptors --experimental-wasm-js-interop`
- Custom descriptor types (struct with descriptor)
- try/catch before the configureAll pattern
- Multiple locals of different ref types
- Compare Liftoff vs TurboFan execution

Start with v1 angle.
