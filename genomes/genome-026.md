# Genome 026 — shared-desc-branch-gap

## Hypothesis
The TODO at `wasm-gc-typed-optimization-reducer.h:182` explicitly states: **"TODO(mliedtke): Extend this for shared types."** This is in the WasmTypeAnnotation handling — the optimizer's mechanism for recording type knowledge from branches and casts.

The IsCastToCustomDescriptor guard (line 21-26) protects cast elimination for custom descriptor types. The 5-bug fix (commit 48c35511792) added this guard to ProcessBranchOnTarget. But the guard only checks:
```cpp
config.to.has_index() &&
module->type(config.to.ref_index()).has_descriptor() &&
config.exactness == compiler::kExactMatchOnly;
```

For SHARED custom descriptor types:
1. The `ref_index()` resolution might differ for shared types (shared type indices map differently)
2. The `has_descriptor()` check might not work for shared descriptor variants
3. ProcessBranchOnTarget might narrow shared descriptor types without triggering the guard → subsequent cast eliminated → descriptor runtime check skipped

This combines two fragile features: shared Wasm types (which already have 2 confirmed crash bugs WASM-TC-001/002) and custom descriptors (which had 5 bugs fixed simultaneously). Their intersection is UNTESTED.

Multi-factor: Shared types + custom descriptors + ProcessBranchOnTarget type narrowing + cast elimination bypass.

## DNA

### Source Facts

**TODO for shared types** (wasm-gc-typed-optimization-reducer.h:182):
```cpp
// TODO(mliedtke): Extend this for shared types.
```
Located in WasmTypeAnnotation processing — the code path that records type knowledge from branch outcomes.

**IsCastToCustomDescriptor** (wasm-gc-typed-optimization-reducer.h:21-26):
```cpp
inline bool IsCastToCustomDescriptor(const wasm::WasmModule* module,
                                     WasmTypeCheckConfig config) {
  return config.to.has_index() &&
         module->type(config.to.ref_index()).has_descriptor() &&
         config.exactness == compiler::kExactMatchOnly;
}
```

**ProcessBranchOnTarget** (wasm-gc-typed-optimization-reducer.cc:447-451):
- Uses IsCastToCustomDescriptor as guard before narrowing type
- If guard returns false, type is narrowed → downstream casts eliminated
- For shared descriptor types, if guard fails to detect the descriptor → type incorrectly narrowed

**5-bug fix** (commit 48c35511792):
- Fixed 446113731, 446113732, 446122633, 446124892, 446124893
- Added IsCastToCustomDescriptor checks to multiple paths
- Before fix: static subtype check → narrow → eliminate cast → descriptor check skipped
- Pattern is PROVEN vulnerable — only needs the guard to miss a case

**GetExactness** (wasm-compiler-definitions.cc:32-36):
```cpp
SubtypeCheckExactness GetExactness(const WasmModule* module, HeapType type) {
  if (!type.has_index()) return kMayBeSubtype;
  auto& t = module->type(type.ref_index());
  return t.is_final ? (t.has_descriptor() ? kExactMatchLastSupertype : kExactMatchOnly)
                    : kMayBeSubtype;
}
```
Question: for shared types, does `type.ref_index()` return a module-local index that correctly resolves to the shared type definition? Or does the shared type mapping break this?

**Shared type canonicalization** (canonical-types.cc):
- Shared types get their own canonical type indices
- `is_shared` flag is part of TypeDefinition
- Shared struct/array types use different GC space (writable shared space)

**Colony history**:
- WASM-TC-001/002 prove shared Wasm types bypass guards designed for JS-side objects
- Pattern P4: shared Wasm objects bypass JS-world checks

### Key Attack Angles
1. Define shared struct type WITH custom descriptor (requires `--experimental-wasm-shared --experimental-wasm-js-interop`)
2. Create function with `br_on_cast` to the shared descriptor type
3. On the taken branch, do `ref.cast` to a more specific type
4. If ProcessBranchOnTarget narrows the type (IsCastToCustomDescriptor returns false for shared variant), the second cast is eliminated
5. The eliminated cast should have included a descriptor check → descriptor semantics bypassed
6. Test whether `ref.get_desc` on the result returns wrong descriptor

## Techniques That Work
1. Build module with:
   - `--experimental-wasm-shared --experimental-wasm-js-interop`
   - Shared struct type `$S` with custom descriptor
   - Shared subtype `$T extends $S` with different descriptor
2. Function taking `(ref null shared any)`:
   - `br_on_cast $S` → taken branch
   - On taken branch: `ref.cast $T` → should require runtime descriptor check
   - `struct.get` specific to `$T`
3. Pass a value that is `$S` (not `$T`) — if cast eliminated, reads wrong field
4. Run with `--no-liftoff --turboshaft-wasm`
5. Compare with `--liftoff` for differential
6. Use `--trace-turbo` to verify cast elimination in graph

## DO NOT Try These
- Non-shared custom descriptor types (thoroughly defended — desc-inline-exactness confirmed)
- Shared types without custom descriptors (crash bugs found, not type confusion)
- IsCastToCustomDescriptor with non-shared types (dead surface — fixed by 48c35511792)

## Evolution Plan
- v1: **Source audit of shared type + descriptor interaction**. Read: (a) How shared types are defined in the module (TypeDefinition flags), (b) Whether IsCastToCustomDescriptor resolves correctly for shared type indices, (c) What exactly the TODO at line 182 means — what code is MISSING, (d) Whether shared types can even HAVE custom descriptors in current implementation.
- v2: **Build shared descriptor module**. Attempt to create a shared struct with custom descriptor. If the decoder rejects it, this surface is blocked by validation. If accepted, test the optimizer path.
- v3: **Lateral: ProcessBranchOnTarget with shared non-descriptor types**. If shared+descriptor is rejected by decoder, test whether ProcessBranchOnTarget handles shared non-descriptor types correctly. The TODO at line 182 may mean something else is missing for shared types.

## Task
Start with v1. Read these files:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — Line 182 in context. Find the function containing this TODO. Read the full function to understand what's missing for shared types.
2. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — ProcessBranchOnTarget around line 447. How does IsCastToCustomDescriptor interact with shared types?
3. `src/wasm/function-body-decoder-impl.h` — Search for "shared" near "descriptor" or "custom_desc". Can shared types have custom descriptors? Is this validated at decode time?
4. `src/wasm/wasm-module.h` — TypeDefinition: check if `is_shared` and `has_descriptor()` can both be true.

Your type invariant to challenge: "IsCastToCustomDescriptor guard protects all cast elimination paths for ALL type variants." The TODO at line 182 says shared types are NOT handled. Find what breaks.

Post to #colony-workers with tags `shared,descriptor,branch-target,guard-gap,TODO`.
