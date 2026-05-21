# Genome 064 — shared-func-cont-decoder

## Hypothesis
`module-decoder-impl.h:699` parses the shared prefix for function/continuation types then hits a TODO (42204563: "Support shared functions/continuations"). The DCHECK fix (cb4665e7b41) changed `DCHECK(type.is_shared)` to `DCHECK(type.is_shared || failed())`, revealing that `consume_describing_type` can fail and leave `type.is_shared` in an inconsistent state.

After the DCHECK, lines 700-705:
```cpp
if (type.kind == TypeDefinition::kFunction ||
    type.kind == TypeDefinition::kCont) {
  // TODO(42204563): Support shared functions/continuations.
  errorf(pc(), "shared function types are not supported yet");
}
```

The question is: does `errorf()` immediately abort decoding, or does it set an error flag and continue? If it continues, the type definition with `is_shared=true` but `kind=kFunction` may be partially registered before the error is detected. This could leave the module's type section in an inconsistent state where some types reference a shared function type that was "rejected" but still partially processed.

Multi-factor: [shared prefix parsed] × [errorf doesn't abort] × [type partially registered] = inconsistent type table.

## DNA

### Source Facts
```cpp
// module-decoder-impl.h:697-705
consume_bytes(1, " shared", tracer_);
TypeDefinition type = consume_describing_type(current_type_index, true);
DCHECK(type.is_shared || failed());
if (type.kind == TypeDefinition::kFunction ||
    type.kind == TypeDefinition::kCont) {
  errorf(pc(), "shared function types are not supported yet");
}

// Decoder::errorf typically sets error_ flag but continues parsing
// if the next check is `if (failed()) return;`
```

### Files to Audit
- `src/wasm/module-decoder-impl.h` — SharedType parsing, consume_describing_type, errorf behavior
- `src/decoder.h` — Decoder base class, errorf semantics, does it throw or set flag?
- `src/wasm/canonical-types.cc` — Does the type canonicalizer register types before validation completes?
- `src/wasm/wasm-module.cc` or `wasm-module.h` — Module type table population

### Key Questions
1. Does `consume_describing_type(current_type_index, true)` ADD the type to the module's type section before returning?
2. Does `errorf()` stop all further processing, or just set a flag?
3. If a shared function type IS partially registered, can subsequent types reference it?
4. Can we craft a module where a shared function type is parsed, "rejected" by errorf, but still present in the type table?

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-shared --experimental-wasm-gc
// NOTE: This test uses raw module bytes because the WasmModuleBuilder
// may not support shared function types directly

// Test 1: Try to create module with shared function type via raw bytes
(function TestSharedFuncDecoder() {
  // Minimal Wasm module with a shared function type
  // Type section: one shared function type () -> ()
  let bytes = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d,  // magic
    0x01, 0x00, 0x00, 0x00,  // version
    // Type section (id=1)
    0x01,                     // section id
    0x06,                     // section size
    0x01,                     // 1 type
    // Shared function type: shared prefix (0x65) + func (0x60) + no params + no results
    0x65,                     // shared prefix
    0x60,                     // function type
    0x00,                     // 0 params
    0x00,                     // 0 results
    // padding byte for alignment
  ]);

  try {
    let module = new WebAssembly.Module(bytes);
    print('TEST1: Module compiled successfully — UNEXPECTED (shared func should be rejected)');
  } catch(e) {
    print('TEST1: Correctly rejected: ' + e.message);
  }
})();

// Test 2: Module with shared function type followed by regular types
(function TestSharedFuncFollowedByRegular() {
  let bytes = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d,
    0x01, 0x00, 0x00, 0x00,
    // Type section
    0x01,
    0x0B,                     // section size (11 bytes)
    0x02,                     // 2 types
    // Type 0: shared function type (should be rejected)
    0x65,                     // shared prefix
    0x60,                     // function type
    0x00,                     // 0 params
    0x00,                     // 0 results
    // Type 1: regular struct type (should this still parse?)
    0x5f,                     // struct type
    0x01,                     // 1 field
    0x7f,                     // i32
    0x01,                     // mutable
  ]);

  try {
    let module = new WebAssembly.Module(bytes);
    print('TEST2: Module compiled — CHECK if type 1 is usable despite type 0 error');
  } catch(e) {
    print('TEST2: Rejected: ' + e.message);
  }
})();

// Test 3: Shared struct type (valid) referencing shared function type (invalid)
(function TestSharedStructRefSharedFunc() {
  let bytes = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d,
    0x01, 0x00, 0x00, 0x00,
    // Type section
    0x01,
    0x0F,                     // section size
    0x02,                     // 2 types in a rec group
    // Type 0: shared function type
    0x65, 0x60, 0x00, 0x00,
    // Type 1: shared struct with field of type (ref 0)
    0x65, 0x5f, 0x01,
    0x6b, 0x00,               // ref type index 0
    0x01,                     // mutable
  ]);

  try {
    let module = new WebAssembly.Module(bytes);
    print('TEST3: Module compiled — shared struct refs shared func');
  } catch(e) {
    print('TEST3: Rejected: ' + e.message);
  }
})();

print('ALL TESTS COMPLETE');
```

## DO NOT Try These
- Shared struct/array types (already well-tested)
- TC-006 kString path (confirmed)
- kFunc shared sibling (architecturally blocked by decoder)

## Evolution Plan
- v1: Source audit of errorf() behavior and consume_describing_type registration. Run raw byte tests.
- v2: If partial registration found, construct module where later types reference the "ghost" shared function type. Test if ref.cast or call_indirect can reach the ghost type.

## Task
**SOURCE AUDIT FIRST.** Read:
1. `src/wasm/module-decoder-impl.h` — trace `consume_describing_type` and what it does with the type
2. `src/decoder.h` — does `errorf()` abort or set flag?
3. Check if the type is added to `module_->types` before or after the shared function check

Then run the raw byte tests. The key question: is `errorf()` sufficient to prevent a partially-registered shared function type from being used?

Post with tags `shared-func-cont,decoder,module-decoder,W23,errorf,partial-parse`.
