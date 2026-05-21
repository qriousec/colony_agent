# Genome 035 — wasm-custom-desc-runtime

## Hypothesis
Custom descriptors have had 7+ bug fixes in 6 months, with commit 48c35511792 alone fixing 5 bugs across the optimizer, runtime, decoder, and subtyping. The fixes were narrow and targeted specific edge cases. The interactions between exactness, descriptors, shared types, and subtyping create a combinatorial explosion of cases that cannot all have been tested. Remaining gaps likely exist in:

1. **ref.get_desc with nested descriptors**: A descriptor type that itself has a descriptor. The fix at 0abc1b27119 added `(ref shared null none)` to the exact subtype list, but other shared+exact+descriptor combinations may be missing.
2. **JSToWasmObject with exact descriptor types at cross-module boundaries**: Fix 48c35511792 added exact type support to JSToWasmObject, but cross-module import/export with descriptor types that differ in exactness may bypass the check.
3. **configureAll with descriptor subtypes**: Fix a58753bac7b corrected the Liftoff fast path for configureAll with incorrect signatures. But configureAll with a function whose parameter types use descriptors with subtype relationships may confuse the signature validation.
4. **ref.cast_desc_eq (renamed from ref.cast_desc)**: The rename suggests semantic changes. If the cast equality semantics changed but callers weren't all updated, the cast may be too permissive or too restrictive.

Multi-factor: Custom descriptor type definition + exactness annotation + subtype relationship + runtime type operation (ref.get_desc, ref.cast_desc_eq, JSToWasmObject) + tier differential.

## DNA

### Bug Fix Cluster (7+ fixes)

| Commit | What was broken | Files touched |
|--------|----------------|---------------|
| 48c35511792 | 5 bugs: ProcessBranchOnTarget wrong reachability, JSToWasmObject missing exact types, ref.get_desc wrong exactness for subtypes, descriptor/described subtyping mismatch, ref.func wrong exactness | 12 files |
| 0abc1b27119 | ref.get_desc: shared null none missing from exact subtype list | value-type.cc/h, decoder |
| 0a6ca69dc82 | ref.null: indexed types automatically exact (wrong) | decoder |
| a58753bac7b | configureAll: Liftoff stack state confusion for wrong signatures | liftoff-compiler.cc |
| e39fada53e3 | TypedOptimizationReducer: descriptor handling in optimizer | turboshaft |
| 4c589fb487a | ref.get_desc for (ref none) | wasm-objects.cc |
| c58f056ef03 | Rename ref.cast_desc to ref.cast_desc_eq | semantic change |
| bee60b882ce | Require struct.new_desc | decoder validation |

### Key Source Files
```
src/wasm/wasm-subtyping.cc — IsSubtypeOf with exactness + descriptors
src/wasm/value-type.h — ValueType::is_exact(), has_descriptor()
src/wasm/function-body-decoder-impl.h — ref.get_desc, ref.cast_desc_eq decoding
src/wasm/wasm-objects.cc — JSToWasmObject with exact types
src/wasm/canonical-types.h — EqualStructType includes descriptor/describes
src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h — ProcessBranchOnTarget with descriptors
```

### Grep Commands
```bash
# Custom descriptor subtyping
grep -rn "is_exact\|set_exact\|exactness\|kExact" src/wasm/wasm-subtyping.cc | head -20

# ref.get_desc implementation
grep -rn "ref.get_desc\|GetDesc\|kRefGetDesc\|kGCPrefix.*0xfb" src/wasm/function-body-decoder-impl.h | head -10

# JSToWasmObject exact type support
grep -rn "exact\|is_exact" src/wasm/wasm-objects.cc | head -15

# ProcessBranchOnTarget descriptor handling
grep -rn "IsCastToCustomDescriptor\|descriptor\|describes\|exact" src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h | head -15

# configureAll
grep -rn "configureAll\|ConfigureAll\|kConfigureAll" src/wasm/ | head -10
```

### Previous Colony Work
- **custom-desc-cast**: 8 iterations on IsCastToCustomDescriptor guard in optimizer → CLEAN
- **desc-inline-exactness**: 3 iterations on inlining vs standalone exactness → CLEAN
- **shared-desc-branch-gap**: 1 iteration → BLOCKED (shared+descriptor UNIMPLEMENTED)
- **configureAll-liftoff-stack**: 2 iterations → CLEAN (fix complete, no similar fast paths)

All focused on OPTIMIZER or COMPILER. NONE tested RUNTIME paths.

## Techniques That Work
1. Build modules with descriptor type hierarchies:
   - Type A (descriptor) describes Type B
   - Type C subtypes Type A (descriptor subtype) describes Type D subtypes Type B
2. Use ref.get_desc, ref.cast_desc_eq, struct.new_desc at runtime
3. Cross-module: export descriptor-typed value from module 1, import in module 2 with different exactness
4. Test table operations: table.set/get with descriptor-typed elements
5. Run with `--no-liftoff --turboshaft-wasm` vs `--liftoff --no-wasm-tier-up` for differentials
6. Focus on: shared + exact + descriptor combinations (smallest tested surface)

## DO NOT Try These
- IsCastToCustomDescriptor optimizer guard — covered (custom-desc-cast, 8 iters, CLEAN)
- Inlining vs standalone exactness — covered (desc-inline-exactness, 3 iters, CLEAN)
- Shared + descriptor types — UNIMPLEMENTED guard prevents creation
- configureAll Liftoff stack state — covered (configureAll-liftoff-stack, CLEAN)

## Evolution Plan
- v1: **Source audit of bug fixes**. Read the 3 key fix commits (48c35511792, 0abc1b27119, 0a6ca69dc82) in detail. For each fix, understand: (a) what was the exact precondition that triggered the bug? (b) what was the exact check that was added/changed? (c) what similar preconditions might trigger the same class of bug? List untested edge cases.
- v2: **Exploit untested edge cases**. Based on v1 source audit, build modules that exercise the edge cases identified. Focus on: (a) ref.get_desc with a value that is an exact subtype of the instruction's type immediate but has a different descriptor, (b) JSToWasmObject with a descriptor-typed value where exactness differs between the import and export, (c) ref.cast_desc_eq where the descriptor matches but the described type doesn't.
- v3: **Cross-module descriptor matching**. Two modules define structurally similar descriptor types. Module 1 exports a descriptor-typed value. Module 2 imports it with a type that has the same canonical structure but different descriptor relationships. Does the canonical type system correctly distinguish them?

## Task
Start with v1. Read these commits IN THIS ORDER:

1. `git -C /Users/t/v8 show 48c35511792 -- src/wasm/wasm-subtyping.cc src/wasm/wasm-objects.cc` — The 5-bug fix. Focus on subtyping changes and JSToWasmObject exact type support. What exact conditions trigger each fix?
2. `git -C /Users/t/v8 show 0abc1b27119 -- src/wasm/value-type.cc src/wasm/value-type.h` — What exact types were added to the subtype list? What OTHER types might be missing?
3. `git -C /Users/t/v8 show 0a6ca69dc82 -- src/wasm/function-body-decoder-impl.h` — ref.null exactness fix. What other instructions might have the same issue?
4. Read current `src/wasm/wasm-subtyping.cc` around the exactness/descriptor code to understand the full type relationship.

Your type invariant to challenge: "Custom descriptor type operations (ref.get_desc, ref.cast_desc_eq, JSToWasmObject, configureAll) correctly handle all combinations of exactness, descriptors, shared types, and subtyping." The 7+ bug fixes prove this invariant has been repeatedly violated — find the next violation.

Post with tags `custom-descriptor,exactness,subtyping,runtime,source-pivot`.
