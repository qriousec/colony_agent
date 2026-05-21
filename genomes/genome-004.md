# Genome 004 — custom-desc-cast

## Hypothesis
The `IsCastToCustomDescriptor` guard in `wasm-gc-typed-optimization-reducer.h:21-26` only blocks type check elimination when `config.exactness == kExactMatchOnly`. But custom descriptor types can participate in casts where `exactness != kExactMatchOnly` (inexact/subtype casts). In those cases, the guard is NOT triggered, and the optimizer proceeds with standard `IsHeapSubtypeOf` check. If the type hierarchy for custom descriptor types has subtyping relationships that are sound for the descriptor's structural equality but unsound for the standard heap subtyping (which doesn't check descriptor identity), a `ref.cast` could be eliminated when it should trap, allowing a struct with descriptor A to be treated as having descriptor B.

## DNA

### Source Facts
- **IsCastToCustomDescriptor** (`wasm-gc-typed-optimization-reducer.h:21-26`):
  ```cpp
  inline bool IsCastToCustomDescriptor(const wasm::WasmModule* module,
                                       WasmTypeCheckConfig config) {
    return config.to.has_index() &&
           module->type(config.to.ref_index()).has_descriptor() &&
           config.exactness == compiler::kExactMatchOnly;
  }
  ```
  This returns true ONLY when exactness is `kExactMatchOnly`. For non-exact casts targeting a descriptor type, this returns false.

- **Cast elimination** (`wasm-gc-typed-optimization-reducer.h:274-280`): The cast is eliminated when:
  ```cpp
  if (wasm::IsHeapSubtypeOf(type.heap_type(), cast_op.config.to.heap_type(),
                            module_) &&
      !IsCastToCustomDescriptor(module_, cast_op.config)) {
  ```
  So: if `IsHeapSubtypeOf` says yes AND `IsCastToCustomDescriptor` says no (i.e., not an exact custom desc cast), the cast is removed.

- **Recent custom descriptor fixes**:
  - `48c35511792 [wasm-custom-desc] Fix subtyping` — subtyping was WRONG for custom descriptors
  - `0abc1b27119 [wasm-custom-desc] Fix ref.get_desc for shared types` — shared types broken
  - `e1308be1c52 [wasm-custom-desc] Handle nofunc in funcref array` — edge case missed
  - `a145aca28df [wasm-custom-desc] Fix typing of unreachable ref.get_desc` — unreachable path wrong
  - `921b3e42db2 [wasm-custom-desc] Fix "unreachable" in TS type analyzer` — type analyzer wrong
  - `c58f056ef03 Rename ref.cast_desc to ref.cast_desc_eq` — the "eq" suffix suggests exactness semantics

- **Subtyping fix** (`48c35511792`): This is critical — subtyping for custom descriptors was fixed recently. If the fix was incomplete, `IsHeapSubtypeOf` could return true for descriptor types that aren't actually subtypes in the descriptor sense.

- **Custom descriptor subtyping rules**: Custom descriptors add structural equality semantics on top of the nominal type hierarchy. Type A with descriptor D1 is NOT a subtype of type A with descriptor D2, even if D1 and D2 have the same structural type, because the descriptors carry identity. Standard `IsHeapSubtypeOf` checks the nominal hierarchy only.

- **wasm-load-elimination** (`3bb690ad095`): "Support ref.get_desc in WasmLoadElimination" — load elimination also interacts with custom descriptors.

- **Consistent hierarchy check** (`c047ab74b85`): "Verify consistent hierarchy in turboshaft wasm type casts/checks" — very recent cleanup suggests the hierarchy was inconsistent.

### Key Attack Vectors
1. **Non-exact cast to descriptor type**: Create a `ref.cast $DescType` (non-exact). If the type analyzer infers the input is already `$DescType` (because it's a subtype by nominal hierarchy), the cast is eliminated. But the value may have a different descriptor.
2. **Subtyping confusion with descriptors**: If `$Child <: $Parent` in the nominal hierarchy, and both have descriptors, then a `ref.cast $Parent` on a `$Child` value is eliminated by the optimizer. But the descriptors might differ, and `ref.get_desc` after the cast would return the wrong descriptor.
3. **Loop phi + descriptor types**: The phi type union doesn't account for descriptor identity. Merging a `$DescType` with a `$DifferentDescType` (same nominal type, different descriptor) would produce the nominal type, and a subsequent exact cast could be wrong.
4. **Load elimination with descriptors**: If `ref.get_desc` is subject to load elimination, a cached descriptor value from one control flow path could be used on another path where the descriptor is different.

### Guards
- `IsCastToCustomDescriptor` only guards exact casts
- Custom descriptor subtyping was recently fixed (`48c35511792`)
- Type annotations for unreachable ref.get_desc were fixed (`a145aca28df`)

## Techniques That Work
1. Create a type hierarchy with custom descriptors: `$Base` with desc D1, `$Child <: $Base` with desc D2
2. Use `ref.cast $Base` (non-exact, so `IsCastToCustomDescriptor` returns false) on a value the optimizer thinks is `$Base` but is actually `$Child`
3. Use `ref.get_desc` to extract the descriptor — if the cast was sound, should get D1; if confused, gets D2
4. Use `ref.cast_desc_eq` (exact) to verify — if the optimizer eliminated a regular cast that shouldn't have been eliminated, the descriptor check reveals the confusion
5. Test cross-function inlining where descriptor type info is lost

## DO NOT Try These
- Testing `ref.cast_desc_eq` elimination (this IS guarded by `IsCastToCustomDescriptor`)
- Testing without `--experimental-wasm-custom-descriptors` flag
- Testing with types that don't have descriptors (normal cast elimination is correct)

## Evolution Plan
- v1: Read the custom descriptor subtyping fix (`48c35511792`). Understand the descriptor type model — how descriptors interact with the nominal hierarchy. Read `IsCastToCustomDescriptor` usage sites to check if all cast/check operations are guarded.
- v2: If v1 finds an unguarded path (e.g., `WasmTypeCheck` reduction doesn't check custom descriptors), construct a module where the type check returns true (constant 1) when it should be dynamic. Test with `ref.test` instead of `ref.cast` — the `REDUCE_INPUT_GRAPH(WasmTypeCheck)` at line 323 also uses `IsHeapSubtypeOf` and might not have the descriptor guard.
- v3: If v2's type check path is guarded, pivot to load elimination: test if `ref.get_desc` values are incorrectly cached across different descriptor instances by `wasm-load-elimination-reducer.h`.

## Task
Start by reading these files in order:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` lines 21-26 (IsCastToCustomDescriptor) and lines 323-380 (WasmTypeCheck reduction — check if descriptor guard is present)
2. Run `git -C /Users/t/v8 show 48c35511792` to see the custom descriptor subtyping fix
3. `src/wasm/wasm-subtyping.cc` — search for "descriptor" to understand how subtyping handles descriptors
4. `src/compiler/turboshaft/wasm-load-elimination-reducer.h` — search for "desc" to see how load elimination handles descriptors
5. Check if `--experimental-wasm-custom-descriptors` is needed or if the feature is enabled by default

The type invariant to challenge: "The type optimizer never eliminates a cast that would be affected by custom descriptor identity."

The specific attack: Find a cast reduction path that uses `IsHeapSubtypeOf` without checking `IsCastToCustomDescriptor`, for a type that has a custom descriptor.

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc --experimental-wasm-custom-descriptors --no-wasm-lazy-compilation`
