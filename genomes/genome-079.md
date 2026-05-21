# Genome 079 — wasmfx-cont-boundary

## Hypothesis
Fix 847c7b2461a added indexed cont type checks to `WasmObjectToJSReturnValue` (wasm-js.cc:2696) and `IsJSCompatibleSignature` (wasm-opcodes.cc:47) because indexed continuation types could leak to JS via exports, globals, tables, and exceptions. The check pattern is:
```cpp
} else if (unsafe_type.has_index() && unsafe_type.ref_type_kind() == RefTypeKind::kCont) {
    thrower->TypeError("invalid type");
    return false;
}
```

This follows the SAME additive-check pattern that caused TC-003/004/005 — each type category needs its own explicit check, and missing categories leak through. Potential gaps:

1. **Shared indexed cont types**: The fix checks `ref_type_kind() == kCont`. Does this correctly handle shared variants? If shared cont types have a different kind encoding, they bypass the check.
2. **Cont references in exception tags**: If an exception tag has a cont-typed payload and the exception propagates from Wasm to JS, does the payload get the cont type check? Exception tag payloads go through a different serialization path than function returns.
3. **Cont references in globals/tables accessed from JS**: The fix adds checks in `WasmObjectToJSReturnValue` and `IsJSCompatibleSignature`. Are ALL global/table access paths covered? What about `WebAssembly.Global.prototype.value` getter? What about `WebAssembly.Table.prototype.get`?
4. **Indirect leaks**: Cont stored in anyref field of a struct, struct exported to JS, JS reads the field — does the field read check for cont type?

## DNA

### Source Facts — The Fix (847c7b2461a)
- **wasm-js.cc:2693-2699**: `WasmObjectToJSReturnValue` — checks generic cont types (existing) AND indexed cont types (new fix)
- **wasm-opcodes.cc:44-49**: `IsJSCompatibleSignature` — blocks functions with indexed cont params/returns
- **Test**: `stack-switching-no-cont-leak.js` — tests globals, tables, functions, exceptions for both abstract and indexed cont types

### Source Facts — Boundary Check Architecture
- `IsJSCompatibleSignature`: Called during module instantiation to validate exported function signatures. If it returns false, the function cannot be exported.
- `WasmObjectToJSReturnValue`: Called when a Wasm value crosses to JS. This is the runtime safety net.
- `WasmToJSObject` (wasm-objects.cc): Converts Wasm values to JS. Called AFTER `WasmObjectToJSReturnValue` allows the value through.

### Source Facts — Where Cont Types Can Appear
1. **Function params/returns**: Blocked by IsJSCompatibleSignature (if correct)
2. **Globals**: Exported globals with cont type — checked by IsJSCompatibleSignature for accessor functions
3. **Tables**: Tables with cont element type — table.get returns cont to JS
4. **Exception tags**: Tag with cont-typed payload — exception propagation to JS carries the payload
5. **Structs/arrays**: Struct with cont-typed field — if struct is anyref-compatible and exported, field access returns cont

### Key Questions
1. Does `ref_type_kind() == kCont` match shared cont types? (shared bit is separate from kind)
2. Does `WebAssembly.Table.prototype.get()` go through `WasmObjectToJSReturnValue`?
3. Does exception propagation from Wasm to JS check payload types against `IsJSCompatibleSignature`?
4. Can a cont reference be stored in an `anyref` or `eqref` field (subtyping allows it?) — NO, cont is not a subtype of any/eq. But what about `(ref null cont)` stored in `(ref null cont)` field of an exported struct? Does the struct field access path check?

### Related Patterns
- TC-003/004/005: Shared strings leak through type-based boundary check (wrappers-inl.h:138)
- This bug: Cont references leak through type-category-specific boundary checks
- Root cause: Boundary checks use additive type enumeration instead of deny-by-default

## Techniques That Work
1. Create WasmFX module with various cont type exposure paths
2. Test each exposure path: global export, table export, function return, exception propagation
3. Use `--experimental-wasm-wasmfx` flag
4. Test both abstract `contref` and indexed `(ref $cont_type)` variants
5. Test shared cont types if shared WasmFX is supported
6. Check if `WebAssembly.Table.get()`, `WebAssembly.Global.value`, and exception payloads all go through the same boundary checks

## DO NOT Try These
- Cont.bind GC roots — DEAD SURFACE
- Basic cont type checking in the type system — well-tested
- String boundary leaks (TC-003/004/005) — different type, already confirmed

## Evolution Plan
- v1: Source audit — trace ALL paths where Wasm values cross to JS. For each path, verify that cont type is checked. Map: (a) function return → WasmObjectToJSReturnValue, (b) global.value getter → ?, (c) table.get → ?, (d) exception propagation → ?, (e) struct/array field access from JS → ?. Create test for each path with indexed cont type.
- v2: Test shared cont types if supported. Create shared module with shared cont types and test all boundary paths. Also test cont types in multi-value returns (only first value may be checked).
- v3: Test indirect leak paths. Store cont in a Wasm-internal anyref global (if allowed by type system). Export a function that reads the global and returns it as anyref. Does the anyref-typed return still get checked for cont content at runtime?

## Task
Start by reading these files in order:
1. `src/wasm/wasm-js.cc:2680-2710` — `WasmObjectToJSReturnValue` with the cont type fix
2. `src/wasm/wasm-opcodes.cc:30-55` — `IsJSCompatibleSignature` with the cont type fix
3. `src/wasm/wasm-objects.cc` — search for `WasmToJSObject`, trace how it's called from table.get, global.value, exception payloads
4. `test/mjsunit/wasm/stack-switching-no-cont-leak.js` — the test for the fix, understand what's covered
5. `src/wasm/wasm-js.cc` — search for `WebAssemblyTableGet`, `WebAssemblyGlobalGetValue` to trace their type checking

Name the type invariant to challenge: **Continuation references must NEVER reach JavaScript through ANY path — function returns, global access, table access, exception propagation, or struct/array field access.** The fix covers function returns and signature validation but may miss other paths.

Start with v1 — map all Wasm-to-JS value crossing paths and check for cont type filtering.
