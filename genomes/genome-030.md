# Genome 030 — shared-anyref-string-leak

## Hypothesis
The Turboshaft-compiled Wasm-to-JS wrapper (wrappers-inl.h ToJS function, line 138) only enters the shared string unsharing path when `IsSubtypeOf(type, kWasmSharedExternRef)`. But shared strings can be internalized via `extern.internalize` to type `(ref shared any)`, which is NOT a subtype of `(ref shared extern)` — they are in separate type hierarchies (any vs extern). The Torque builtin (js-to-wasm.tq:878-881) and C++ runtime (wasm-objects.cc:3765-3769) use RUNTIME type checks (`Is<String>`, `InSharedSpace/InWritableSharedSpace`) that catch shared strings regardless of their declared Wasm type. This creates a tier differential:

- **Turboshaft wrapper** (optimized): shared string via `(ref shared any)` return type → no unsharing (static type doesn't match shared externref) → shared string escapes to JS
- **Liftoff/generic wrapper**: same shared string → runtime check catches it → properly unshared

The escaped shared string in JS could cause data races between isolates, incorrect string identity checks, or violations of single-isolate string invariants.

Multi-factor: extern.internalize of shared string + return as shared anyref + static-vs-runtime type check mismatch at wrapper boundary + tier differential.

## DNA

### Source Facts

**Turboshaft wrapper ToJS** (wrappers-inl.h:132-160):
```cpp
// Line 138: STATIC type check
if (IsSubtypeOf(type, kWasmSharedExternRef)) {
  // Only enters this path for shared externref subtypes
  // Checks for strings and unshares them
  OpIndex instance_type = LoadInstanceType(LoadMap(ret));
  V<Word32> is_string = __ Uint32LessThan(
      instance_type, __ Word32Constant(FIRST_NONSTRING_TYPE));
  IF (is_string) {
    result = __ WasmCallRuntime(...Runtime::kWasmWasmToJSObject, {ret}...);
  }
}
// Line ~157: For non-shared-extern types, falls through to WasmNull handling
// NO string check for shared anyref, shared eqref, etc.
```

**Torque builtin** (js-to-wasm.tq:878-881):
```torque
// RUNTIME type check — catches ALL shared strings
if (Is<String>(value) && InSharedSpace(UnsafeCast<String>(value))) {
  return runtime::WasmWasmToJSObject(context, value);
}
```

**C++ WasmToJSObject** (wasm-objects.cc:3765-3769):
```cpp
// RUNTIME type check — catches ALL shared strings
if (IsString(*value)) {
  DirectHandle<String> string = Cast<String>(value);
  if (HeapLayout::InWritableSharedSpace(*string)) {
    return String::Unshare(isolate, string);
  }
}
```

**Key insight**: The Turboshaft wrapper uses the DECLARED type of the return value to decide whether to check for shared strings. The Torque/C++ paths use the ACTUAL type of the runtime value. When the declared type is `(ref shared any)` but the actual value is a shared string, the Turboshaft wrapper skips the unsharing check.

### Wasm Type Hierarchy
```
any (shared any)
├── eq (shared eq)
│   ├── i31
│   ├── struct (shared struct)
│   └── array (shared array)
├── func
└── ...

extern (shared extern)
├── string
└── ...
```

`(ref shared any)` is NOT a subtype of `(ref shared extern)`. A shared string internalized to `(ref shared any)` via `extern.internalize` retains its string identity but changes its Wasm type.

### Commit Trail
- 64b77c7889e: Unshare strings at Wasm-to-JS boundary (fix)
- 27a83f0b039: Consolidate WasmToJS/JSToWasm transformations (refactoring)
- 0a3073a0b0c: [wasm][shared] Implement shared string constants

### Grep Commands
```bash
grep -rn "kWasmSharedExternRef\|kWasmSharedAnyRef\|kWasmSharedEqRef" src/wasm/ src/compiler/
grep -rn "IsSubtypeOf.*Shared" src/wasm/wrappers-inl.h
grep -rn "extern.internalize\|extern_internalize\|ExternInternalize" src/wasm/
grep -rn "InSharedSpace\|InWritableSharedSpace" src/wasm/ src/builtins/js-to-wasm.tq
```

## Techniques That Work
1. Build shared Wasm module with shared string constant
2. Use `extern.internalize` to convert shared externref string to shared anyref
3. Export function returning `(ref shared any)` that returns the internalized shared string
4. Call the exported function from JS
5. Compare: `--no-liftoff --turboshaft-wasm` (Turboshaft wrapper) vs `--liftoff --no-wasm-tier-up` (Liftoff/generic wrapper)
6. Check if the returned string identity differs between tiers (shared vs unshared)
7. Test: `Object.is(result1, result2)` where result1 from optimized path, result2 from baseline
8. Test: try to use the shared string from optimized path in another isolate (Worker)

## DO NOT Try These
- Shared strings returned as shared externref — already fixed (64b77c7889e)
- Non-string shared objects (structs, arrays) — covered by wasm-to-js-shared-apis worker
- Shared custom descriptors — UNIMPLEMENTED

## Evolution Plan
- v1: **Source audit + differential test**. First verify that `extern.internalize` works for shared strings (check decoder acceptance). Build minimal module: shared string constant → extern.internalize → return as `(ref shared any)`. Run with Turboshaft vs Liftoff. Check if string identity/sharing differs.
- v2: **Other shared type paths**. If v1 confirms differential, test other shared ref types that might carry strings: `(ref shared eq)`, `(ref null shared any)`. Also test with `string.const` instruction directly if available.
- v3: **Exploitation**. If tier differential confirmed, test: can the shared string from the optimized path be used across isolate boundaries (SharedArrayBuffer postMessage scenario)? Can it cause a data race on string content?

## Task
Start with v1. Read these files IN THIS ORDER:

1. `src/wasm/wrappers-inl.h` around line 130-160 — the ToJS function. Understand exactly which types trigger the shared string check. Map all code paths for shared types.
2. `src/builtins/js-to-wasm.tq` around line 864-882 — the Torque WasmToJSObject macro. Compare with wrappers-inl.h: which checks differ?
3. `src/wasm/wasm-objects.cc` around line 3750-3772 — the C++ WasmToJSObject. Same comparison.
4. Check if `extern.internalize` accepts shared strings: `grep -rn "extern.internalize\|ExternInternalize" src/wasm/` — look at the decoder and the implementation.
5. Build test module and run differential.

Your type invariant to challenge: "The Turboshaft-compiled wrapper unshares shared strings consistently with the Torque/C++ runtime path, regardless of the declared return type." If `IsSubtypeOf(type, kWasmSharedExternRef)` is the gate, then shared strings returned via non-externref shared types bypass unsharing.

Post results with tags `shared,string,anyref,differential,tier-mismatch`.
