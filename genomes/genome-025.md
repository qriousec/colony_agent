# Genome 025 — jsfunc-sig-asymmetry

## Hypothesis
The JSToWasmObject boundary (wasm-objects.cc:3546-3596) has three distinct code paths for function references, each with DIFFERENT type checking strength:

1. **WasmExportedFunction** (line 3554): Uses `IsCanonicalSubtype(real_type_index, expected)` — full subtype + exactness check
2. **WasmJSFunction** (line 3566): Uses `MatchesSignature(canonical_index)` — INDEX-ONLY equality
3. **WasmCapiFunction** (line 3575): Uses `MatchesSignature(canonical_index)` — INDEX-ONLY equality

`MatchesSignature` (line 3395) is: `return sig->index() == other_canonical_sig_index;` — simple equality. It does NOT check:
- Subtyping relationships (a subtype of the expected signature would be REJECTED)
- Custom descriptor exactness (a type with same canonical ID but wrong descriptor would be ACCEPTED)
- The TODO at line 3390 explicitly notes this wasn't updated for custom descriptors

An attacker can control WHICH path is taken by wrapping a function differently:
- `new WebAssembly.Function(sig, jsFunc)` creates a WasmJSFunction → MatchesSignature path
- Exporting from a module creates a WasmExportedFunction → IsCanonicalSubtype path

If a function's canonical signature index matches the expected type's canonical index, but the actual semantics differ (due to custom descriptors, or because the canonical index was reused), MatchesSignature would incorrectly ACCEPT the function while IsCanonicalSubtype would correctly REJECT it.

Multi-factor: WebAssembly.Function wrapping + MatchesSignature index-only check + custom descriptor semantics bypass.

## DNA

### Source Facts

**JSToWasmObject function** (wasm-objects.cc:3546-3596):
```cpp
MaybeDirectHandle<Object> JSToWasmObject(Isolate* isolate,
                                         DirectHandle<Object> value,
                                         CanonicalValueType expected, ...) {
  // ...
  if (WasmExportedFunction::IsWasmExportedFunction(*value)) {
    // Uses IsCanonicalSubtype — FULL CHECK
    auto func = Cast<WasmExportedFunction>(value);
    CanonicalTypeIndex real_type_index = func->shared()...;
    if (!IsCanonicalSubtype(real_type_index, expected.ref_index(), ...)) {
      *error_message = "...type incompatibility...";
      return {};
    }
  } else if (WasmJSFunction::IsWasmJSFunction(*value)) {
    // Uses MatchesSignature — INDEX ONLY
    auto func = Cast<WasmJSFunction>(value);
    if (!func->MatchesSignature(expected.ref_index())) {
      *error_message = "...signature mismatch...";
      return {};
    }
  } else if (WasmCapiFunction::IsWasmCapiFunction(*value)) {
    // Uses MatchesSignature — INDEX ONLY
    // ...
  }
}
```

**MatchesSignature** (wasm-objects.cc:3390-3395):
```cpp
bool WasmJSFunction::MatchesSignature(CanonicalTypeIndex other_canonical_sig_index) {
  // TODO(14034): MatchesSignature was not updated for custom descriptors
  return sig->index() == other_canonical_sig_index;
}
```

**IsCanonicalSubtype** includes:
- Full subtype check with param contravariance and return covariance
- Exactness check for custom descriptor types
- Handles shared/non-shared variants

**WasmToJSObject** (wasm-objects.cc:3750-3772):
- NO type checks at all — just format conversion (WasmNull→null, WasmFuncRef→JSFunction)
- Output type is never validated

### Attack Vectors
1. **Custom descriptor bypass**: If two function types have the same canonical index but different descriptors, MatchesSignature accepts both. Create WebAssembly.Function with type A's index, pass it where type B (same canonical index, different descriptor) is expected.
2. **Subtype rejection**: If sig B is a subtype of expected sig A (param contravariance), WasmExportedFunction would accept it but WasmJSFunction would reject it. This is a conformance bug, not security — but the asymmetry might have security implications in code that depends on consistent behavior.
3. **Cross-module canonical index collision**: Two modules with different types that happen to canonicalize to the same index. MatchesSignature would accept a function from the wrong module.

## Techniques That Work
1. Create two modules:
   - Module A: defines `$sig_a = (i32) -> i32` with custom descriptor
   - Module B: defines `$sig_b = (i32) -> i32` without custom descriptor (or different descriptor)
2. Check if `$sig_a` and `$sig_b` get the same canonical index (they should if structurally identical without descriptor in canonical form)
3. Create `new WebAssembly.Function({parameters: ['i32'], results: ['i32']}, jsFunc)`
4. Try to pass this WasmJSFunction to a table/global expecting `(ref $sig_a)` — which path does it take?
5. If MatchesSignature accepts it, the function enters a slot expecting custom descriptor semantics
6. Subsequent calls through that slot might apply wrong descriptor operations
7. Run with `--experimental-wasm-js-interop` for custom descriptors
8. Test with `table.set` and `call_indirect` to trigger the full pipeline

## DO NOT Try These
- Basic signature matching without custom descriptors (MatchesSignature is correct for plain sigs)
- WasmExportedFunction path (already has full checks)
- WasmToJSObject direction (no type checks by design — outputs are always safe JS values)
- Cross-module sharing without WebAssembly.Function wrapping (covered by cross-module-exact)

## Evolution Plan
- v1: **Source audit of MatchesSignature**. Read: (a) full MatchesSignature implementation, (b) how canonical indices are assigned for function types with vs without custom descriptors, (c) whether the TODO(14034) has any partial fixes, (d) all callers of MatchesSignature to find other bypass points.
- v2: **Build WebAssembly.Function bypass test**. Create module with custom descriptor function type. Create WebAssembly.Function with matching canonical signature. Pass it to table/global expecting the descriptor type. Check if it's accepted and if subsequent operations apply wrong descriptor.
- v3: **Canonical index collision test**. Create two modules where structurally identical but semantically different function types get same canonical index. Use WebAssembly.Function to bridge between modules.

## Task
Start with v1. Read these files:
1. `src/wasm/wasm-objects.cc` — Search for "MatchesSignature". Read the full implementation. Find all callers (grep). Read JSToWasmObject around lines 3546-3596.
2. `src/wasm/canonical-types.cc` — How function types with custom descriptors are canonicalized. Does the descriptor affect the canonical index?
3. `src/wasm/wasm-js.cc` — Search for "WebAssembly.Function" constructor. How is the canonical type assigned to a WasmJSFunction?
4. Check: `grep -rn 'MatchesSignature' src/wasm/` — find ALL callers beyond JSToWasmObject.

Your type invariant to challenge: "MatchesSignature provides equivalent type safety to IsCanonicalSubtype for function references at the JS↔Wasm boundary." The TODO(14034) says it doesn't. Find what breaks.

Post to #colony-workers with tags `jsfunc,signature,boundary,asymmetry,custom-descriptor`.
