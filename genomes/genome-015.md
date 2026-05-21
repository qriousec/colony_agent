# Genome 015 — shared-wasm-dispatch

## Hypothesis
The confirmed crash bug WASM-TC-001 (js-objects.cc:588 UNREACHABLE for shared Wasm structs) has siblings in V8's JS runtime where IsShared() type dispatch enumerates JS shared types but misses Wasm shared types. Beyond UNREACHABLE/FATAL crashes, there may be paths where shared Wasm objects are MISIDENTIFIED as other JS shared types or where missing dispatch leads to using the wrong memory layout — a type confusion rather than a crash. Specifically, if code assumes "IsShared() && !IsJSSharedStruct() && !IsJSSharedArray() → must be primitive" (as js-objects.cc:587 does), and then accesses object memory with primitive assumptions, the Wasm struct's GC-managed fields could be misinterpreted.

## DNA

### Confirmed Bug (Parent: shared-type-boundary / genome 012)
- **js-objects.cc:582-588**: `JSReceiver::class_name()` → IsShared() dispatch → UNREACHABLE for Wasm shared structs
- **Trigger path**: WeakSet.add(sharedWasmStruct) → ThrowTypeError → NoSideEffectsToString → class_name()
- **Root cause**: Comment "Other shared values are primitives" is INCORRECT for Wasm shared types

### Identified Siblings (from Explore agent)
1. **js-objects.cc:2528-2741**: `GetHeaderSize(InstanceType)` switch — FATAL() for WASM_STRUCT_TYPE/WASM_ARRAY_TYPE. Crash, not type confusion.
2. **value-serializer.cc:672-695**: `WriteObject()` — asymmetric handling of shared Wasm vs JS shared types.
3. **objects-inl.h:2126-2134**: `CanBeHeldWeakly()` — CORRECTLY handles Wasm shared types (reference for what "fixed" looks like).

### Additional Search Targets
- `src/objects/js-objects.cc` — full class_name() function, GetIdentityHash, other methods
- `src/objects/lookup.cc` — property lookup on shared objects
- `src/objects/elements.cc` — element access on shared arrays
- `src/builtins/` — builtins that handle shared objects
- `src/runtime/` — runtime functions that check IsShared or shared space

### Key Search Commands
```bash
grep -rn 'IsShared\b' src/objects/ src/builtins/ src/runtime/ --include='*.cc' | grep -v test | grep -v '_test'
grep -rn 'JSSharedStruct\|JSSharedArray' src/objects/ --include='*.cc' | grep -v 'INSTANCE_TYPE'
grep -rn 'IN_WRITABLE_SHARED_SPACE\|IsInWritableSharedSpace' src/ --include='*.cc' --include='*.h'
grep -rn 'UNREACHABLE\|FATAL' src/objects/js-objects.cc
```

## Techniques That Work
1. Create shared Wasm structs/arrays using `--experimental-wasm-shared`
2. Pass them through JS operations that enumerate shared types
3. Focus on paths where fallthrough is NOT UNREACHABLE but instead leads to WRONG behavior
4. Check if any code path treats a shared WasmStruct as a JSSharedStruct (different header layout, different field access)
5. Check InstanceType dispatch — WASM_STRUCT_TYPE vs JS_SHARED_STRUCT_TYPE have different maps and layouts

## DO NOT Try These
- WeakMap/WeakSet crash (already confirmed by parent worker, WASM-TC-001)
- Struct field offset computation (dead surface, confirmed by struct-offset-gc)
- Memory cache reload tests (dead surface, confirmed by wasmfx-cont-leak)

## Evolution Plan
- v1: **Grep audit of all IsShared() callsites and InstanceType dispatch involving shared types**. For each callsite, determine: (a) whether Wasm shared types could reach it, (b) what the fallthrough behavior is, (c) whether the fallthrough leads to wrong type handling (not just crash). Focus on paths in property lookup, element access, and GC that might treat shared Wasm objects differently.
- v2: **Test the most promising non-crash sibling**. If a path is found where shared Wasm objects cause wrong behavior rather than crash, build a test module. Focus on operations where the code CONTINUES with wrong assumptions rather than aborting.
- v3: **Cross-pollinate with WLE**. If shared Wasm structs interact with WasmLoadElimination (shared structs can be modified concurrently), test whether the WLE properly invalidates cached values from shared struct fields.

## Task
Start with v1. Your primary goal is NOT to find more crashes but to find paths where shared Wasm objects cause WRONG TYPE HANDLING. A crash is fitness 2; a type confusion is fitness 5.

Read these files first:
1. `src/objects/js-objects.cc` — full class_name() and other methods involving IsShared
2. `src/objects/lookup.cc` — property lookup for shared objects
3. `src/runtime/runtime-object.cc` — runtime object operations

Grep for all `IsShared` callsites. For each, check if Wasm shared types can reach it and what happens when they do.

Post to #colony-workers with tags `shared,dispatch,siblings,export`.
