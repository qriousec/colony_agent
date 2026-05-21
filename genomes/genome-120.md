# Genome 120 — shared-type-assert-skip

## Hypothesis
At wasm-gc-typed-optimization-reducer.h:181-183, AssertType() returns early for all shared types with a TODO comment. This means runtime type assertions are completely disabled for shared WasmGC types. The optimizer still performs type refinement and check elimination for shared types — but the safety net that catches incorrect type inference is absent. If the optimizer incorrectly narrows a shared type (e.g., infers (ref shared $StructA) when the actual value is (ref shared $StructB)), no assertion fires to catch the error. This creates a blind spot where shared type confusions propagate undetected through the optimization pipeline.

## DNA

### Source Facts (file:line)
- **AssertType skip**: wasm-gc-typed-optimization-reducer.h:181-183 — `if (type.is_shared()) { return; }` with TODO(mliedtke)
- **AssertType normal path**: wasm-gc-typed-optimization-reducer.h:176-220 — builds WasmTypeAssertionFailed trap for uninhabited types, calls AssertNotNull/WasmTypeAssert for others
- **Cast elimination**: wasm-gc-typed-optimization-reducer.h:274-280 — still runs for shared types (no shared check here)
- **Type check elimination**: wasm-gc-typed-optimization-reducer.h:341-345 — still runs for shared types
- **Type refinement**: wasm-gc-typed-optimization-reducer.h:284-320 — still refines types for shared casts
- **instance_data_ init**: wasm-gc-typed-optimization-reducer.h:170-173 — WasmInstanceDataParameter loaded for type assertions
- **IsHeapSubtypeOf**: used at lines 274, 341, 451 — subtype checks that could be incorrect for shared types

### Gap Analysis
1. **Assertions disabled, optimizations enabled**: The optimizer eliminates casts and refines types for shared GC values, but the assertion mechanism that validates these decisions is off. This is like running with seat belt unbuckled on a new road.
2. **Shared struct type confusion**: If two shared struct types $A and $B have different field layouts, and the optimizer incorrectly infers $A for a $B value, the assertion that would trap is disabled. Subsequent field access uses $A's layout on $B's memory → type confusion.
3. **Shared type subtyping**: Does IsHeapSubtypeOf correctly handle shared types? If shared($A) <: shared($B) computation has bugs, the optimizer could make wrong decisions with no assertion to catch them.
4. **Interaction with GC**: Shared types live in shared space. The GC handles them differently. If type refinement changes the expected layout and GC uses the refined type for tracing, incorrect refinement could cause GC to miss fields or scan wrong offsets.

### Attack Strategy
The goal is NOT to test the assertion mechanism itself (that's a debugging aid). The goal is to find a case where the optimizer makes a WRONG type decision for shared types that would be caught by AssertType in non-shared mode, but goes undetected in shared mode.

Steps:
1. Build two shared struct types with different field layouts
2. Create a function where control flow merges could cause the optimizer to pick the wrong type
3. The optimizer refines the type at the merge point
4. Access fields based on the refined (possibly wrong) type
5. If the refinement is wrong: in non-shared mode, AssertType would trap. In shared mode, it silently proceeds → type confusion.

## Techniques That Work
1. Create shared struct types with incompatible field layouts (e.g., i32 vs externref)
2. Use phi merges where different branches produce different shared struct types
3. Force tier-up to Turboshaft to activate the type optimization reducer
4. Compare behavior with `--wasm-assert-types` on non-shared vs shared variants of the same test
5. Use GC stress (`--gc-interval=100`) to exercise shared space GC with potentially wrong type info
6. Use `--experimental-wasm-shared` for shared type support

## DO NOT Try These
- Testing AssertType with non-shared types — this is the well-tested path
- Testing without Turboshaft optimization — assertions only matter in optimized code
- Simple single-type tests — need at least two types at a merge point to create ambiguity
- Testing the assertion failure mechanism itself — we want to find cases where assertions SHOULD fire but DON'T

## Evolution Plan
- v1: Source audit of type refinement for shared types. Read wasm-gc-typed-optimization-reducer.h/cc for all places where types are refined. Check if IsHeapSubtypeOf handles shared types correctly. Search for "shared" in wasm-subtyping.cc to verify subtyping rules.
- v2: Build test with two shared struct types, phi merge, field access after refinement. Run with --wasm-assert-types to see if assertions would fire for non-shared equivalent. Run shared version to see if the optimization makes wrong decisions.
- v3: If v2 finds a gap: exploit the type confusion. If not: search for cases where the optimizer makes DIFFERENT decisions for shared vs non-shared types — check if shared subtyping is more permissive or restrictive than expected.

## Task
1. Read wasm-gc-typed-optimization-reducer.h:176-220 (full AssertType function) to understand what assertions are skipped for shared types.
2. Read wasm-gc-typed-optimization-reducer.h:274-320 (cast elimination + type refinement) to check if shared types are handled differently.
3. Search for "shared" in src/wasm/wasm-subtyping.cc — find how shared types interact with subtype checking.
4. Search for "is_shared" in src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h — find all shared-type special cases.
5. Build test: two shared structs ($A with i32 field, $B with i64 field), merge at phi, access field after cast.
6. Run with: `--experimental-wasm-shared --no-wasm-lazy-compilation --wasm-tiering-budget=1 --wasm-assert-types`
7. Start with Evolution Plan v1 (source audit of shared type handling in optimizer).
