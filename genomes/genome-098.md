# Genome 098 — funcsig-exact-import

## Hypothesis

WasmJSFunction::MatchesSignature (wasm-objects.cc:3386-3396) checks function signature compatibility using canonical signature INDEX equality only (TODO 14034 explicitly notes "Subtyping and custom descriptors are not considered"). In contrast, WasmExportedFunction uses IsCanonicalSubtype (wasm-objects.cc:3554) which checks full subtype + exactness relations.

When a JavaScript-created function (via `new WebAssembly.Function(...)`) is passed to a Wasm module that expects an exact-typed funcref, MatchesSignature passes if the canonical indices match — but it ignores:
1. The exactness flag on the expected type
2. Whether the function's signature is a subtype (not exact match) of the expected signature
3. Custom descriptor compatibility

If the module's optimizer assumes exactness (because the type declaration says exact), it may eliminate type checks. But MatchesSignature didn't enforce exactness at import time. This creates a gap: the optimizer's assumptions don't match the actual validation that was performed.

Specific scenario: A function type `$FT = (func (param (ref exact $Struct)) (result i32))` expects an exact struct reference. A WebAssembly.Function created with a signature `(func (param (ref $Struct)) (result i32))` (without exact) passes MatchesSignature because the canonical indices match. When this function is called through a table or funcref, the optimizer may skip the exactness check on the parameter — allowing a subtype to be passed where only the exact type should be accepted.

## DNA

### Source Facts

1. **MatchesSignature (WasmJSFunction)** — `src/wasm/wasm-objects.cc:3386-3396`
   ```cpp
   // TODO(14034): Change this to a type canonicalization or subtype check.
   return IsCanonicalSubtype(type_index, canonical_id);
   ```
   Wait — the TODO says it should be changed. Let me verify: does the current code actually use IsCanonicalSubtype or just index equality? The jsfunc-sig-asymmetry worker reported it uses `MatchesSignature(canonical_index)` which is index-only. Need to verify current code.

2. **WasmExportedFunction path** — `src/wasm/wasm-objects.cc:3554`
   ```cpp
   if (!wasm::IsCanonicalSubtype(func_index, canonical_id))
   ```
   Full subtype check including exactness.

3. **JSToWasmObject boundary** — `src/wasm/wasm-objects.cc:3546-3596`
   - WasmExportedFunction (line 3554): IsCanonicalSubtype — EXACTNESS CHECKED
   - WasmJSFunction (line 3566): MatchesSignature — INDEX ONLY
   - WasmCapiFunction (line 3575): MatchesSignature — INDEX ONLY
   - This asymmetry is the core attack surface.

4. **Exact types in function signatures** — `src/wasm/value-type.h`
   - ValueType has an `is_exact()` bit
   - Exact means "this SPECIFIC type, not any subtype"
   - Used in WasmGC for struct references in function parameters

5. **GetExactness** — `src/wasm/wasm-compiler-definitions.cc:21-39`
   - Returns `kExactMatchOnly` for final exact types
   - Returns `kExactMatchLastSupertype` for final types with descriptors
   - Returns `kMayBeSubtype` otherwise
   - The optimizer uses exactness to decide whether to eliminate casts

6. **Canonical signature comparison** — `src/wasm/canonical-types.cc`
   - Canonical signature IDs are per-signature, not per-type
   - Two signatures with different exactness on parameters would get DIFFERENT canonical IDs
   - So MatchesSignature with index equality would REJECT a signature with different exactness
   - BUT: if the module declares `(func (param (ref exact $S)))` and the WebAssembly.Function is created with `(func (param (ref $S)))`, do they get the same or different canonical IDs?

7. **WebAssembly.Function constructor** — `src/wasm/wasm-js.cc`
   - How does JS create a function with a specific Wasm signature?
   - Can JS create functions with exact type annotations?
   - If JS cannot express exact types, the created function always has non-exact params
   - The canonical ID would differ, and MatchesSignature would reject it

### Key Questions to Answer
- Can WebAssembly.Function express exact types in signatures? If not, this hypothesis may be moot.
- Does MatchesSignature actually use index equality or IsCanonicalSubtype? The source may have changed.
- How are canonical IDs computed for signatures with exact vs non-exact params? Same or different?

### Key Files to Read
- `src/wasm/wasm-objects.cc:3380-3600` — MatchesSignature, JSToWasmObject, the full type checking dispatch
- `src/wasm/wasm-js.cc` — WebAssembly.Function constructor, how JS types map to Wasm types
- `src/wasm/canonical-types.cc` — How signatures are canonicalized, effect of exactness on canonical ID
- `src/wasm/canonical-types.h` — CanonicalEquality::EqualValueType, now checks is_equal_except_index (fix a970eed8995)

## Techniques That Work

1. **Define exact-typed function signature**: `(func (param (ref exact $Struct)) (result i32))`
2. **Create WebAssembly.Function from JS**: `new WebAssembly.Function({parameters: ['externref'], results: ['i32']}, jsFunc)`
3. **Import the JS function into Wasm module**: Put it in a table or pass as funcref import
4. **Call through table or call_ref**: The import validation path is MatchesSignature
5. **Compare with WasmExportedFunction**: Export a Wasm function with the same signature, re-import it — uses IsCanonicalSubtype path
6. **Tier-up differential**: Test if optimizer eliminates exactness checks after import
7. **Use `--experimental-wasm-custom-descriptors`** if exact types require it

## DO NOT Try These

- Don't test WasmExportedFunction import path (already uses IsCanonicalSubtype — correct)
- Don't test basic function signature matching without exact types (well-tested)
- Don't test desc-exactness inlining gap (confirmed DEAD surface)

## Evolution Plan

- v1: **Source trace** — Read wasm-objects.cc:3380-3600 to verify current MatchesSignature implementation. Read wasm-js.cc to determine if WebAssembly.Function can express exact types. Read canonical-types.cc to understand if exact vs non-exact signatures get different canonical IDs. Answer the 3 key questions above.

- v2: **Import asymmetry probe** — If the source trace confirms a gap: construct a module with an exact-typed funcref table, create a WebAssembly.Function with a non-exact signature, set it into the table, and call it with a subtype argument. If the import validation doesn't enforce exactness, the optimizer may skip the runtime exactness check, allowing a subtype where only the exact type is valid.

- v3: **Cross-module variant** — If v2 is clean, try cross-module import: Module A exports a function with exact param type. Module B imports it. The import validation uses a different path than direct function creation. Test if this path has the same asymmetry.

## Task

**Start by reading `src/wasm/wasm-objects.cc` around lines 3380-3600.** This is the PRIMARY target. Answer:
1. What does MatchesSignature actually do now? Is it still index-only, or has TODO 14034 been addressed?
2. What's the exact difference between the WasmJSFunction and WasmExportedFunction validation paths?
3. Are there other entry points for function validation that might have different behavior?

**Then read `src/wasm/wasm-js.cc`** — search for `WebAssembly.Function` or `WasmJSFunction` to understand how JS creates Wasm-typed functions and whether exact types can be expressed.

**Type invariant to challenge:** Function signature validation at the JS-Wasm boundary enforces exactness when the importing module declares an exact-typed funcref.

**Start with Evolution Plan v1.**
