# Genome 054 — wrapper-reland-audit

## Hypothesis
The WasmToJS/JSToWasm consolidation reland (commit 27a83f0b039) simplified the Torque `WasmToJSObject` macro by removing type-specific handling. The OLD code had:
```torque
const refKind = (retType & kValueTypeRefKindBits) >>> kValueTypeRefKindShift;
if (refKind == RefTypeKind::kStruct || refKind == RefTypeKind::kArray) {
  return UnsafeCast<JSAny>(value);  // Fast path for struct/array
}
```
The NEW code just does:
```torque
if (Is<WasmFuncRef>(value)) { ... convert ... }
if (Is<String>(value) && InSharedSpace(...)) { ... unshare ... }
return UnsafeCast<JSAny>(value);  // Everything else returned as-is
```

The removal of type-specific handling means the Torque wrapper now returns ALL non-null, non-func, non-shared-string values directly. This is SIMPLER but could it be LESS SAFE?

Specific concern: before consolidation, the runtime function `WasmGenericWasmToJSObject` was the fallback for unrecognized types. It checked for WasmFuncRef and WasmNull. After consolidation, there's no fallback — everything is returned as-is.

**Could internal Wasm objects (WasmStruct, WasmArray, WasmFuncRef with no external) leak to JS incorrectly?**

## DNA

### Consolidation Diff (commit f64e44ed348)
Old WasmToJSObject in js-to-wasm.tq:
- Checked refKind for kStruct, kArray, kFunction
- Checked isGeneric for kAny, kEq, kI31, kExtern, kString, kNoExtern
- Fallback: `runtime::WasmGenericWasmToJSObject(context, value)`

New WasmToJSObject in js-to-wasm.tq:
- Checks for kWasmNull → JS null
- Checks for WasmFuncRef → convert to JSFunction (or runtime slow path)
- Checks for shared String → runtime unshare
- Everything else: `UnsafeCast<JSAny>(value)`

### Removed Runtime Function
Old: `Runtime_WasmGenericWasmToJSObject` checked WasmFuncRef and WasmNull, returned everything else as-is.
New: `Runtime_WasmWasmToJSObject` calls `wasm::WasmToJSObject` in C++ (handles FuncRef, WasmNull, shared strings).

### Key Question
The old code's struct/array fast path (`UnsafeCast<JSAny>`) and the new code's catch-all (`UnsafeCast<JSAny>`) both return the value as-is. So WasmStruct/WasmArray objects ARE expected to be returned to JS directly.

But what about:
1. WasmFuncRef with `external` field not yet created — old code hit the runtime fallback which called `GetOrCreateExternal`. New code checks `Is<JSFunction>(maybeJsFunc)` and falls back to runtime. Is this complete?
2. WasmNull that somehow gets past the first check — old code had the runtime fallback. New code returns it as-is via `UnsafeCast<JSAny>`.
3. Edge cases in multi-return where values are in a FixedArray — does the new code handle all types correctly?

## Evolution Plan
- v1: Source audit — compare old and new Torque code path by path. Identify any values that took a different path in old vs new code.
- v2: Build tests for edge cases: WasmFuncRef without external, WasmNull in unexpected position, multi-return with mixed types.
- v3: If any differential found, minimize and root-cause.

## Task
Start with v1 source audit. Read both old and new versions of WasmToJSObject in js-to-wasm.tq (use git show to get old version). Map every value type through both code paths.

Post with tags `wrapper-consolidation,reland,audit,torque,type-path`.
