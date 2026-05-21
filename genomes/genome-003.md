# Genome 003 — jstowasm-reftype

## Hypothesis
The recently consolidated WasmToJS/JSToWasm transformation (reverted March 10, relanded March 11, 2026) unified `Runtime_WasmGenericWasmToJSObject` and the C++ implementation in `wasm-objects.cc`. During consolidation, the simplified `WasmToJSObject` in `js-to-wasm.tq` (Torque) may not handle all GC reference type conversions that the old redundant runtime function handled. Specifically, when a Wasm struct or array reference is passed to JS and then back to Wasm (round-trip), the type information may be lost or confused because the consolidated path handles the JS→Wasm conversion differently for indexed reference types vs abstract reference types.

## DNA

### Source Facts
- **Revert/Reland** (`252ea756a14` → `27a83f0b039`): "Consolidate WasmToJS/JSToWasm transformations". Original change: I3f161637d. Described as "wrongfully reverted" in the reland — but the fact it was reverted at all suggests instability.
- **JSToWasmObject main entry** (`wasm-objects.cc:3507`): `MaybeDirectHandle<Object> JSToWasmObject(Isolate* isolate, ...)` — takes a `CanonicalValueType expected` parameter. This is the gatekeeper for JS values entering Wasm.
- **JSToWasmObject with module** (`wasm-objects.cc:3732-3747`): Overload that converts `ValueType` to `CanonicalValueType` using `CanonicalValueType{unsafe_expected_type}`. Note the variable name `unsafe_expected_type` — developers explicitly mark this as unsafe!
- **WasmToJSObject** (`wasm-objects.cc:3750`): `DirectHandle<Object> WasmToJSObject(Isolate* isolate, ...)` — Wasm→JS direction.
- **String unsharing at boundary** (`64b77c7889e`): "Unshare strings at the Wasm-to-JS boundary" — very recent (after the consolidation reland), suggesting the consolidated path missed string handling.
- **Table type check** (`wasm-objects.cc:334`): `return wasm::JSToWasmObject(isolate, module, entry, table->type(module), ...)` — table operations use `table->type(module)` which returns `CanonicalValueType`.
- **FromJS in wrappers** (`wrappers.h:400`): `OpIndex FromJS(V<Object> input, OpIndex context, CanonicalValueType type, ...)` — the wrapper builder's FromJS takes canonical type.
- **Type validation in module-instantiate.cc** (`module-instantiate.cc:681-715`): Import validation uses `CanonicalValueType expected_type` — this is where type checks happen at instantiation.
- **Shared string fix** (`cb4665e7b41`): "Fix DCHECK in decoding of invalid shared type" — shared type handling is buggy.
- **Cont type leak** (`847c7b2461a`): "Fix leak of indexed cont types to JS" — indexed reference types leaking across boundary.

### Key Attack Vectors
1. **Round-trip type confusion**: Pass a Wasm struct (type A) to JS, then pass it back to Wasm expecting type B. The JSToWasmObject function checks `IsSubtypeOf(actual_canonical, expected)` — but `actual_canonical` is extracted from the WasmTypeInfo on the object's Map, which may not distinguish between subtypes correctly.
2. **unsafe_expected_type conversion** (`wasm-objects.cc:3737-3745`): The overload constructs `CanonicalValueType{unsafe_expected_type}` from a non-canonical `ValueType`. If the module's canonical mapping is wrong, the expected type is wrong.
3. **Null handling at boundary**: Wasm null vs JS null/undefined. The `WasmToJSObject` for nullable reference types must produce JS null, and `JSToWasmObject` must accept JS null for nullable types but reject it for non-nullable.
4. **Table set/get with GC refs**: Table operations call JSToWasmObject internally. If the table's type is an abstract type (funcref, anyref) but the value is a specific subtype, the conversion may lose the specific type info.
5. **Custom descriptor types at boundary**: After `c58f056ef03` renamed `ref.cast_desc` — do custom descriptor types survive the JS boundary correctly?

### Guards
- `wasm::JSToWasmObject` performs `IsSubtypeOf` check on the canonical type
- WasmTypeInfo on the Map stores the canonical type
- Table dispatch checks signature via `WasmDispatchTable::table_type()`

## Techniques That Work
1. Create two modules sharing a type hierarchy via canonical type equality
2. Module A exports a function returning `$Parent` ref
3. Module B imports it expecting `$Child` ref (should fail at link time, but check if it passes with anyref intermediary)
4. Use JS as the intermediary: Wasm A → JS → Wasm B, where the type expectation changes
5. Test with table operations: `table.set` from JS with a Wasm struct, `table.get` from Wasm expecting a different struct type
6. Test null handling: pass JS null where Wasm expects non-nullable ref

## DO NOT Try These
- Numeric type confusion at boundary (well-tested, and numeric types don't go through JSToWasmObject for refs)
- Testing with only abstract types like funcref/externref (too well-tested)
- Direct module-to-module imports without JS intermediary (different code path)

## Evolution Plan
- v1: Read the consolidation diff (`27a83f0b039`). Map the JSToWasmObject validation path for indexed GC types (structs, arrays). Identify exactly where the canonical type is extracted from a JS-side object and compared to the expected Wasm type.
- v2: If v1 finds a gap in the validation (e.g., a path where IsSubtypeOf is skipped, or where the canonical type on the Map doesn't include nullability/exactness), construct a test module that passes a parent struct where a child struct is expected, using JS as intermediary.
- v3: If v2's validation is sound, pivot to the `unsafe_expected_type` conversion path and test if non-canonical ValueTypes produce incorrect CanonicalValueTypes. Also test the string unsharing path and shared type boundary handling.

## Task
Start by reading these files in order:
1. Run `git -C /Users/t/v8 diff 252ea756a14..27a83f0b039 -- src/wasm/wasm-objects.cc src/wasm/wrappers-inl.h` to see the consolidation changes
2. `src/wasm/wasm-objects.cc` lines 3473-3750 (JSToWasmObject implementations and WasmToJSObject)
3. `src/wasm/wasm-objects-inl.h` lines 577-620 (WasmTypeInfo::type(), WasmTableObject::canonical_type())
4. `src/wasm/module-instantiate.cc` lines 680-720 (import validation with canonical types)

The type invariant to challenge: "JSToWasmObject correctly validates that a JS-wrapped Wasm object's canonical type is a subtype of the expected canonical type at the receiving Wasm boundary."

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc`
