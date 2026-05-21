# Genome 131 — exnref-tag-type-mismatch

## Hypothesis
The exnref feature was recently shipped (commit c794d17cf00 removed the `--experimental-wasm-exnref` flag). When a Wasm exception carries GC reference values in its payload, the tag's canonical signature determines how values are encoded and decoded. If a module catches an exception using an imported tag whose canonical signature matches but whose local type representation differs — for example, same canonical struct index but from different recursive type groups with different field layouts due to cross-module type aliasing — the caught exception payload values could be decoded with wrong types, enabling struct field confusion.

**Specific attack vector**: Module A defines tag $t with signature `[ref $StructA]`. Module B defines tag $t with signature `[ref $StructB]`. If $StructA and $StructB are in different recursive type groups that happen to canonicalize to different types, but the tag import/export matching uses a looser check, then Module B catching Module A's exception could interpret the payload as $StructB when it's actually $StructA. If $StructA has fewer fields than $StructB, reading $StructB's extra fields on a $StructA instance is an OOB read.

## DNA
- **Commit c794d17cf00**: Removed `--experimental-wasm-exnref` flag, making exnref a shipped feature. Newly shipped features often have edge cases that weren't fully tested during the experimental phase.
- **Exception handling flow**: `throw $tag value1 value2...` → encodes payload values into exception object → propagates up call stack → `try_table (catch $tag $label)` → decodes payload at catch site → values pushed on stack at $label
- **Tag canonical matching**: Tags are matched by their canonical signature. Two tags from different modules match if their canonical parameter types match. But canonical type matching might not verify all aspects of the type (e.g., field layouts of struct types in the signature).
- **wasm-objects.h/cc**: `WasmExceptionPackage` stores payload values. `GetExceptionValues` extracts them using the tag's signature to determine types and sizes.
- **Cross-module tag import**: Module B imports tag from Module A. The import matching checks canonical signature equality. If the check is structural (same canonical types), it should be safe. But if there's a gap — e.g., the check uses `EquivalentIndices` instead of full type comparison — nullability or exactness differences could slip through.
- **GC ref in payload**: When a GC ref (struct, array) is part of an exception payload, it's stored as a tagged pointer in the exception object. On catch, the value is extracted and cast to the expected type. If the expected type differs from the actual type, this is a type confusion.
- **Interaction with optimizing compiler**: Turboshaft may optimize catch blocks based on the declared tag signature. If the optimizer assumes the caught value has type $StructB (as declared in Module B's tag), but the actual value is $StructA (from Module A's throw), optimized field accesses could be wrong.

## Techniques That Work
1. Create two modules with different struct types that DO NOT canonicalize to the same type
2. Module A: define tag with struct ref payload, export tag and a throwing function
3. Module B: import tag, but with a different struct type in the tag signature (if import matching allows it)
4. Module B catches the exception and accesses struct fields based on its local type understanding
5. Use cross-module instantiation (importObject wiring)
6. Also test: same tag with nullable vs non-nullable ref in payload
7. Also test: exception re-throw across module boundaries with type narrowing in the catcher
8. Force Turboshaft tier-up for optimized catch block behavior

## DO NOT Try These
- Shared types (--experimental-wasm-shared, BANNED)
- Old exception handling proposal (deprecated try/catch syntax)
- Testing without GC ref payloads — simple i32/i64 payloads won't cause type confusion
- Testing only at Liftoff tier — type confusion from optimization is the main target

## Evolution Plan
- v1: **Source analysis of exception payload encoding/decoding**. Read:
  - `src/wasm/wasm-objects.cc` — `GetExceptionValues`, exception payload storage and retrieval
  - `src/wasm/module-instantiate.cc` — tag import matching logic. What check is used? Canonical signature comparison? Full subtype check? Just index equality?
  - `src/wasm/wasm-module.cc` — tag definition and how tag signatures are stored
  - Search for `kExceptionTag`, `WasmExceptionTag`, `WasmExceptionPackage` to find all related code
  Map: (a) how payloads are encoded (per-field by type?), (b) how payloads are decoded (uses catcher's type or thrower's type?), (c) what checks ensure consistency between throw site and catch site type expectations.
- v2: **Cross-module tag type mismatch test**. Build test:
  - Module A: rec group with struct $A { i32 }, tag $t with param (ref $A), function that throws $t with struct instance
  - Module B: rec group with struct $B { i32, i64 }, tag $t with param (ref $B), function that catches $t and reads i64 field
  - Wire tag import. If the import matching rejects because signatures differ (different struct canonical IDs), try variations: make the structs canonically identical but with different module-local interpretations, or use supertype/subtype relationship.
- v3: **Optimizer interaction**. Even if v2 doesn't find a direct type mismatch, test the optimizer's assumptions about caught values. Build a function with try_table that catches a GC-ref-payload exception inside a loop. Force Turboshaft tier-up. Check if the optimizer narrows the caught ref type based on the tag signature and eliminates subsequent casts. Then trigger an edge case: throw from a different module with a compatible-but-different type, or throw null in a non-nullable payload position. Does the optimizer's assumption hold?

## Task
Start by reading:
1. `src/wasm/module-instantiate.cc` — search for tag import resolution. How are imported tags matched? What signature comparison is used?
2. `src/wasm/wasm-objects.cc` — search for `GetExceptionValues` and `WasmExceptionPackage`. How are payload values extracted? Which type info is used — the tag's declared types or the catch site's expected types?
3. `git -C /Users/t/v8 show c794d17cf00` — the exnref shipping commit. What was changed beyond removing the flag? Any last-minute fixes included?
4. `grep -rn 'exnref\|kExnRef\|WasmExceptionPackage' /Users/t/v8/src/wasm/` — map all exnref-related code

Then build test 1:
- Module A: struct $S { i32 }, tag $t [ref $S], function throws $t with $S instance { field0: 0x41414141 }
- Module B: imports $t from A, function uses try_table to catch $t, extracts the ref payload, casts to its local struct type, reads fields
- Verify: does the catch succeed? Are the payload values correctly typed? What happens with null payloads?

Run with: `/Users/t/v8/out/fuzzbuild/d8 test.js`
No --experimental-wasm-shared flag.
