# Genome 011 — wasm-to-js-indexed

## Hypothesis
The cont type leak fix (847c7b2461a) revealed that `WasmObjectToJSReturnValue` in `wasm-js.cc` checked abstract cont types but missed indexed cont types. This is pattern P6: "abstract type check present but indexed variant missed." The fix added a check for `has_index() && ref_type_kind() == kCont`. But there may be OTHER indexed reference types that should also be blocked or handled differently at the WasmToJS boundary. Additionally, `IsJSCompatibleSignature` in `wasm-opcodes.cc` was also modified — there may be similar gaps in OTHER callsites of this function, or in the `WasmToJSObject` function itself.

## DNA

### Source Facts
- **Fix location** (`wasm-js.cc:2693-2696`):
  ```cpp
  } else if (unsafe_type.has_index() &&
             unsafe_type.ref_type_kind() == i::wasm::RefTypeKind::kCont) {
    thrower->TypeError("invalid type %s", unsafe_type.name().c_str());
    return false;
  }
  ```
  This was added AFTER the existing abstract type checks. The `else if` means it only triggers if the abstract type check didn't catch it.

- **IsJSCompatibleSignature** (`wasm-opcodes.cc:44`): Also modified to reject indexed cont types. This function gates whether a Wasm function can be exported to JS.

- **Pattern**: The original code checked `switch (unsafe_type.heap_type())` for abstract types (kCont, etc.) but missed that `has_index()` types can ALSO be continuation types.

- **Other indexed types to check**:
  - `kStruct` — indexed struct types (should be exportable as JS objects)
  - `kArray` — indexed array types (should be exportable as JS objects)
  - `kFunction` — indexed function types (should be exportable as JS functions)
  - `kCont` — indexed continuation types (should NOT be exportable — FIXED)
  - What about custom descriptor types? Shared types?

- **WasmToJSObject** (`wasm-objects.cc:3750`): The actual conversion function. Does it handle all indexed types correctly?

### Key Attack Vectors
1. **Other indexed types at WasmToJS boundary**: Are there indexed types other than cont that should be blocked? What about types with custom descriptors? Shared indexed types?
2. **IsJSCompatibleSignature gaps**: This function determines if a function signature can cross the JS boundary. If it allows a signature with an incompatible indexed type, the function can be exported.
3. **WasmToJSObject for exotic indexed types**: When an indexed struct/array is converted to JS, the resulting JS object wraps the Wasm object. If the wrapping doesn't preserve type safety (e.g., allows field access with wrong types), type confusion is possible.
4. **Import/export validation for indexed types**: When a Wasm module imports/exports functions with indexed reference types in their signatures, does the validation correctly handle all indexed type variants?
5. **Global/table export with indexed types**: Besides functions, globals and tables can also expose Wasm values to JS. The cont type fix checked these paths, but are all similar paths covered?

## Techniques That Work
1. Read the full fix diff: `git -C /Users/t/v8 show 847c7b2461a`
2. Search for ALL callsites of `WasmObjectToJSReturnValue` and `IsJSCompatibleSignature`
3. Search for ALL places that check `ref_type_kind()` — are all RefTypeKind values handled?
4. Test exporting/importing functions with various indexed reference types
5. Test global.get from JS for globals containing indexed types
6. Test table.get from JS for tables containing indexed types

## DO NOT Try These
- Testing the JSToWasm direction (already defended by jstowasm-reftype)
- Testing with abstract types only (the bug was specifically about INDEXED variants)
- Testing with numeric types

## Evolution Plan
- v1: Read the full fix diff. Map ALL callsites of `WasmObjectToJSReturnValue`, `IsJSCompatibleSignature`, and `WasmToJSObject`. Check if each callsite handles all `RefTypeKind` values for indexed types.
- v2: If v1 finds a gap (e.g., a callsite that handles kStruct and kArray but not kCont or vice versa), construct a test module that exports a function/global/table with the unhandled indexed type. If all callsites are comprehensive, test edge cases: shared indexed types, exact indexed types, custom descriptor types at the boundary.
- v3: If v2 finds a bypass, create an exploit that leaks a type-confused value to JS. If clean, test the interaction between exported WasmGC objects and JS proxy/wrapper semantics.

## Task
Start by reading:
1. `git -C /Users/t/v8 show 847c7b2461a` — full diff of the cont type leak fix
2. `src/wasm/wasm-js.cc` — search for `WasmObjectToJSReturnValue` and ALL callers
3. `src/wasm/wasm-opcodes.cc` — search for `IsJSCompatibleSignature` and ALL callers
4. `src/wasm/wasm-objects.cc` — search for `WasmToJSObject` — how does it handle each indexed type?
5. `grep -rn 'ref_type_kind\|RefTypeKind' src/wasm/wasm-js.cc src/wasm/wasm-opcodes.cc`

Type invariant to challenge: "All indexed reference types that cannot safely cross the JS boundary are blocked at ALL WasmToJS conversion points."

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc --experimental-wasm-stack-switching`
