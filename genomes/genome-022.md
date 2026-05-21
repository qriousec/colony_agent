# Genome 022 — branch-desc-shared-gap

## Hypothesis
The recent 5-bug fix (commit 48c35511792, bugs 446113731/446113732/446122633/446124892/446124893) corrected `ProcessBranchOnTarget` in the type analyzer to account for `IsCastToCustomDescriptor`. This was a critical fix: before it, the optimizer could incorrectly determine that a branch always succeeds based on static subtyping, even when custom descriptor semantics required a runtime check.

However, there is an explicit TODO at `wasm-gc-typed-optimization-reducer.h:182`: **"TODO(mliedtke): Extend this for shared types."** This means shared type variants of custom descriptor types are NOT handled by the type assertion optimization. If a shared custom descriptor type flows through ProcessBranchOnTarget:

1. The IsCastToCustomDescriptor guard checks `config.exactness == kExactMatchOnly` — but shared types might use different exactness
2. The type assertion optimization (line 182) doesn't handle shared types → might skip the guard entirely
3. ProcessBranchOnTarget might narrow the type on the taken branch without checking for custom descriptors on shared types

The 5-bug fix proves this code path has been buggy. The TODO proves shared types are not yet handled. Combining shared types with custom descriptors could bypass the guard.

Multi-factor: Shared types + custom descriptors + ProcessBranchOnTarget type narrowing + cast elimination.

## DNA

### Source Facts

**TODO for shared types** (wasm-gc-typed-optimization-reducer.h:182):
```cpp
// TODO(mliedtke): Extend this for shared types.
```
This is in the WasmTypeAnnotation handling — type assertions that the optimizer emits to record type knowledge.

**IsCastToCustomDescriptor guard** (wasm-gc-typed-optimization-reducer.h:21-26):
```cpp
inline bool IsCastToCustomDescriptor(const wasm::WasmModule* module,
                                     WasmTypeCheckConfig config) {
  return config.to.has_index() &&
         module->type(config.to.ref_index()).has_descriptor() &&
         config.exactness == compiler::kExactMatchOnly;
}
```

**ProcessBranchOnTarget fix** (48c35511792):
- Added `IsCastToCustomDescriptor` check to prevent incorrect branch narrowing
- Before fix: static subtype check → narrow type → eliminate subsequent cast
- After fix: if custom descriptor involved, don't narrow (descriptor check needed at runtime)

**5 bugs fixed simultaneously**:
- 446113731: ProcessBranchOnTarget incorrect reachability with custom descriptors
- 446113732: JSToWasmObject missing exact type support
- 446122633: ref.get_desc returning exact type incorrectly
- 446124892, 446124893: Related exactness issues

**Shared type handling gaps**:
- `--experimental-wasm-shared` enables shared types
- Shared struct/array types have `is_shared()` flag
- Custom descriptors + shared types = untested combination
- No explicit check in IsCastToCustomDescriptor for shared variants

**GetExactness for descriptors** (wasm-compiler-definitions.cc:32-36):
```cpp
SubtypeCheckExactness GetExactness(const WasmModule* module, HeapType type) {
  if (!type.has_index()) return kMayBeSubtype;
  auto& t = module->type(type.ref_index());
  return t.is_final
      ? (t.has_descriptor() ? kExactMatchLastSupertype : kExactMatchOnly)
      : kMayBeSubtype;
}
```
- For shared descriptor types, is `is_final` and `has_descriptor()` correctly set?

### Key Attack Angles
1. Create a shared struct type WITH custom descriptor
2. Use br_on_cast or ref.test to create a branch that narrows the type
3. Check if ProcessBranchOnTarget correctly prevents narrowing for shared descriptor types
4. If not, subsequent code uses the narrowed type assumption → descriptor check bypassed
5. Test whether IsCastToCustomDescriptor fires for shared types (does `ref_index()` resolve correctly for shared types?)
6. Test `ref.get_desc` on shared types — does it correctly handle shared descriptor types?

### Related Code Locations
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — IsCastToCustomDescriptor (21-26), TODO at line 182, REDUCE WasmTypeCast (274-287), REDUCE WasmTypeCheck (341-356)
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — ProcessBranchOnTarget (447-451), ProcessTypeCast, ProcessTypeCheck
- `src/compiler/wasm-compiler-definitions.cc` — GetExactness (32-36)
- `src/wasm/canonical-types.cc` — How shared types are canonicalized
- `src/wasm/wasm-module.h` — TypeDefinition with is_shared, has_descriptor

## Techniques That Work
1. Build module with `--experimental-wasm-shared --experimental-wasm-js-interop`:
   - Define a shared struct type with custom descriptor
   - Define a subtype of the shared struct
2. Create function that:
   - Takes `(ref null shared any)` parameter
   - Does `br_on_cast` to the shared descriptor type → taken branch has narrowed type
   - On the taken branch, does another `ref.cast` to a more specific type
3. Test whether the second ref.cast is eliminated (it shouldn't be if descriptor check is needed)
4. Compare Liftoff vs Turboshaft behavior
5. Use `--no-liftoff --turboshaft-wasm`
6. Test `ref.get_desc` on shared types — does it return correct descriptor?

## DO NOT Try These
- Non-shared custom descriptor casts (thoroughly tested by desc-exactness, custom-desc-cast — 12+ iterations, DEAD)
- Basic shared type dispatch (tested by shared-wasm-dispatch, 2 crash bugs found)
- IsCastToCustomDescriptor with kExactMatchLastSupertype (dead surface — semantically equivalent for final types)

## Evolution Plan
- v1: **Source audit of shared type + descriptor interaction**. Read: (a) how shared types are defined in WasmModule (TypeDefinition flags), (b) whether IsCastToCustomDescriptor handles shared types, (c) what the TODO at line 182 means concretely — what code path is missing for shared types, (d) whether ref.get_desc works for shared types.
- v2: **Build shared descriptor type test**. Create module with shared struct + descriptor. Test br_on_cast → ref.cast elimination chain. Compare Liftoff vs Turboshaft.
- v3: **Cross-test shared + exact + descriptor combinations**. Test edge cases: shared final type with descriptor, shared non-final type with descriptor, shared type in recursive group with descriptor.

## Task
Start with v1. Read these files:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — Focus on line 182 TODO, IsCastToCustomDescriptor (21-26), and all places where shared types might interact with type optimization
2. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — ProcessBranchOnTarget (search for IsCastToCustomDescriptor usage), ProcessTypeCast, ProcessTypeCheck
3. `src/wasm/wasm-module.h` — TypeDefinition struct: is_shared, has_descriptor, is_final flags
4. `src/wasm/canonical-types.cc` — How shared descriptor types are canonicalized

Your type invariant to challenge: "The IsCastToCustomDescriptor guard protects all cast elimination paths involving custom descriptor types." The TODO at line 182 explicitly states shared types are NOT yet handled. If shared custom descriptor types reach cast elimination without the guard, descriptor checks are skipped.

Post to #colony-workers with tags `shared,descriptor,branch-target,guard-gap`.
