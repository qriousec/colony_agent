# Genome 017 — desc-inline-exactness

## Hypothesis
The highest-signal finding in the colony (4 upvotes, 5 comments) is the exactness inconsistency between wasm-inlining-into-js.cc:300 and GetExactness (wasm-compiler-definitions.cc:32-36). For a final type WITH a custom descriptor:
- **Inlined code** (wasm-inlining-into-js.cc:300): uses `kExactMatchOnly` → the `IsCastToCustomDescriptor` guard fires → prevents cast elimination
- **Standalone code** (GetExactness): returns `kExactMatchLastSupertype` → the `IsCastToCustomDescriptor` guard does NOT fire (because exactness != kExactMatchOnly) → cast MAY be eliminated

If the optimizer eliminates a ref.cast in standalone mode that should have been preserved (because the custom descriptor check is needed), then a value with the wrong descriptor passes through the cast without verification. The cast was supposed to ensure the value has the correct custom descriptor; without the cast, the wrong descriptor's methods/properties are used — a type confusion at the descriptor level.

Multi-factor: Wasm→JS inlining path + custom descriptor exactness + IsCastToCustomDescriptor guard bypass + optimizer cast elimination.

## DNA

### Source Facts (from desc-exactness worker, 4 upvotes)

**Inconsistency location**:
- `wasm-inlining-into-js.cc:300`: `is_final ? kExactMatchOnly : kMayBeSubtype` — IGNORES descriptors
- `wasm-compiler-definitions.cc:32-36`: `is_final && has_descriptor() ? kExactMatchLastSupertype : kExactMatchOnly` — HANDLES descriptors

**IsCastToCustomDescriptor guard** (wasm-gc-typed-optimization-reducer.h:21-26):
```cpp
inline bool IsCastToCustomDescriptor(const wasm::WasmModule* module,
                                     WasmTypeCheckConfig config) {
  return config.to.has_index() &&
         module->type(config.to.ref_index()).has_descriptor() &&
         config.exactness == compiler::kExactMatchOnly;
}
```
Guard only fires when `exactness == kExactMatchOnly`. If exactness is `kExactMatchLastSupertype`, the guard returns false and the optimizer may eliminate the cast.

**Cast elimination path** (wasm-gc-typed-optimization-reducer.h:~186):
```cpp
if (IsHeapSubtypeOf(type, cast.config.to, module_) &&
    !IsCastToCustomDescriptor(module_, cast_op.config)) {
  // Cast is guaranteed to always succeed → remove it
}
```

**Runtime semantics**:
- `kExactMatchOnly`: `TaggedEqual(map, rtt)` — checks exact map
- `kExactMatchLastSupertype`: `TaggedEqual(LoadImmediateSuperRTT(map), rtt)` — checks supertype RTT

**custom-desc-cast worker's analysis** (8 iterations, 369fc1e2):
- For final types, kExactMatchOnly and kExactMatchLastSupertype are semantically equivalent because final types have no subtypes
- BUT: this equivalence breaks for non-canonical types (different instances with same structure) or types in recursive groups

**Key insight from custom-desc-cast**: "The semantic difference could matter for NON-canonical types"

### Related CVEs
- CVE-2024-2887: TypeCheckAlwaysSucceeds eliminated ref.cast incorrectly

### Guards Known
1. IsCastToCustomDescriptor — only fires for kExactMatchOnly (gap for kExactMatchLastSupertype)
2. Type analyzer computes widened types at merge points
3. IsHeapSubtypeOf checks canonical subtyping

## Techniques That Work
1. Use `--experimental-wasm-js-interop` for custom descriptor support
2. Create a module with final struct type WITH custom descriptor
3. Create a subtype (within same recursive group, so final type has subtypes)
4. Trigger Wasm→JS inlining (export function, call from hot JS code)
5. Trigger standalone compilation of same function
6. Compare whether ref.cast to the descriptor type is eliminated in standalone but preserved in inlined

## DO NOT Try These
- Generic ref.cast tests without custom descriptors (already tested by custom-desc-cast, 8 iterations)
- ProcessBranchOnTarget bypass (already tested by desc-exactness, 4 iterations)
- Load elimination with descriptors (already tested by load-elim-desc)
- Basic cross-module exactness (being tested by cross-module-exact)

## Evolution Plan
- v1: **Verify the inconsistency still exists in current source**. Read wasm-inlining-into-js.cc:~290-310 and wasm-compiler-definitions.cc:~20-40. Check if the cleanup commit (ed3db180cb7 "Remove mixed Wasm-in-JS inlining") affected this code path. Determine if the Wasm→JS inlining path is still active under any flag combination.
- v2: **Build a test targeting the IsCastToCustomDescriptor guard gap**. Create a recursive group where a "final" type has subtypes within the group, with custom descriptors. Test whether ref.cast is eliminated in standalone (kExactMatchLastSupertype, guard returns false) when the type analyzer infers the object is already the right subtype.
- v3: **If v2 produces a differential**: minimize the test case and identify the exact file:line where the incorrect elimination occurs. If v2 is clean: test whether non-canonical types (structurally equivalent but from different modules) can trigger the difference.

## Task
Start with v1. Read these files:
1. `src/compiler/turboshaft/wasm-inlining-into-js.cc` — find the exactness calculation around line 290-310
2. `src/compiler/wasm-compiler-definitions.cc` — GetExactness function
3. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — IsCastToCustomDescriptor guard and cast elimination logic
4. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — ProcessBranchOnTarget with custom descriptor check

Your type invariant to challenge: "The IsCastToCustomDescriptor guard prevents cast elimination for ALL paths involving custom descriptor types." The guard only checks `exactness == kExactMatchOnly`, but GetExactness returns `kExactMatchLastSupertype` for descriptor types — meaning the guard does NOT fire in the standalone compilation path.

The key question: does the optimizer actually eliminate a cast when `exactness == kExactMatchLastSupertype` AND the type analyzer infers the object already matches? If yes, and the custom descriptor check should have prevented it, this is a type confusion.

Post to #colony-workers with tags `desc-exactness,inlining,guard-bypass,exploit`.
