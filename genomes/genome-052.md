# Genome 052 — tc006-kstring-verify

## Hypothesis
TC-006: JSToWasmObject GenericKind::kString (wasm-objects.cc:3698) does `if (IsString(*value)) return value;` without calling ConvertToSharedIfExpected. kExtern (line 3612) and kAny (line 3627) both call ConvertToSharedIfExpected. kString does not.

The kString GenericKind is reached for Wasm stringref type. If shared stringref exists (with --experimental-wasm-shared), a non-shared JS string passed to a Wasm function expecting shared stringref will enter shared space without being shared.

**However:** shared-boundary-export could NOT reproduce this. Need to verify:
1. Is shared stringref a valid type in V8?
2. Does the kString path actually get reached for shared stringref?
3. Can we construct a test that passes a non-shared string through the kString path?

## DNA
### Source Facts
```cpp
// wasm-objects.cc:3698 — kString case (NO sharing check)
case GenericKind::kString:
  if (IsString(*value)) return value;  // ← MISSING ConvertToSharedIfExpected
  *error_message = "wrong type (expected a string)";
  return {};

// wasm-objects.cc:3612 — kExtern case (HAS sharing check)
case GenericKind::kExtern: {
  if (!ConvertToSharedIfExpected(isolate, &value, expected, error_message)) { return {}; }
  ...
}

// wasm-objects.cc:3488 — ConvertToSharedIfExpected
if (v8_flags.experimental_wasm_shared && expected.is_shared() && (**value).IsHeapObject()) {
  if (!HeapLayout::InWritableSharedSpace(heap_obj)) {
    if (!Object::Share(isolate, *value, ...).ToHandle(value)) { ... }
  }
}
```

### Key Questions
1. What CanonicalValueType maps to GenericKind::kString? Check `GenericKindFromTypeIndex` or equivalent dispatch.
2. Is `kWasmStringRef.shared()` valid? Does it have is_shared()=true?
3. After a non-shared string enters shared Wasm space, what happens on GC? DCHECK failure?

## Evolution Plan
- v1: Source audit — trace how GenericKind::kString is selected. Find which CanonicalValueType triggers it. Check if shared stringref is constructible.
- v2: If reachable, build test: Wasm function taking shared stringref, pass non-shared JS string, check if GC crashes or DCHECK fails.

## Task
Start with v1 source audit. Read wasm-objects.cc to find how GenericKind is determined from CanonicalValueType. Then check if shared stringref can be constructed in WasmModuleBuilder.

Post with tags `tc-006,kstring,verify,shared,stringref`.
