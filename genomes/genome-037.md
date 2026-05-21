# Genome 037 — table-descriptor-exactness

## Hypothesis
Wasm tables have element types. When `table.set` stores a value, V8 must validate the value's type is a subtype of the table's element type. When the element type involves custom descriptors with exactness annotations, this validation must correctly handle the exactness/descriptor/subtype interaction. The 7+ custom descriptor bug fixes (commit 48c35511792 fixing 5 bugs at once) focused on direct type operations (ref.get_desc, ref.cast_desc_eq, JSToWasmObject) but NOT on table operations. Tables are a separate runtime path:

1. **table.set type validation**: `WasmTableObject::Set` in wasm-objects.cc validates the stored value's type. Does this validation handle exact descriptor types? The JSToWasmObject fix (48c35511792) added exact type support but JSToWasmObject is called at the JS↔Wasm boundary, not for table.set within Wasm.
2. **table.get type annotation**: When reading from a table, the returned value gets the table's element type. If the element type has a descriptor, does the returned ref correctly carry the descriptor type annotation for subsequent operations (ref.get_desc)?
3. **Imported table type matching**: `ProcessImportedTable` in module-instantiate.cc:1982 checks the imported table's type against the expected type. With descriptor-typed elements, does this check handle exactness? Line 2040: "we cannot allow the importing module to write supertyped values" — does this hold for descriptor types?
4. **table.copy between tables with different descriptor element types**: If table A has element type `(ref exact $DescA)` and table B has `(ref $DescB)` where $DescB subtypes $DescA, does table.copy correctly validate?

Multi-factor: Table with descriptor-typed elements + exactness annotation + runtime type operation (table.set/get/copy) + cross-module import.

## DNA

### Bug Fix Cluster (Custom Descriptors)
| Commit | Fix | Table path tested? |
|--------|-----|--------------------|
| 48c35511792 | 5 bugs: JSToWasmObject, ref.get_desc, subtyping | NO — JSToWasmObject is boundary, not table |
| 0abc1b27119 | ref.get_desc: shared null none missing | NO |
| 0a6ca69dc82 | ref.null: indexed types auto-exact (wrong) | UNKNOWN — ref.null into table? |
| a58753bac7b | configureAll: Liftoff fast path | NO |

### Key Source Files
```
src/wasm/wasm-objects.cc — WasmTableObject::Set, WasmTableObject::Get, JSToWasmObject
src/wasm/module-instantiate.cc:1982 — ProcessImportedTable, type matching
src/wasm/function-body-decoder-impl.h — table.set/get/copy decoding and type validation
src/wasm/wasm-subtyping.cc — IsSubtypeOf with exactness + descriptors
src/wasm/canonical-types.h — Canonical type matching for cross-module tables
```

### Grep Commands
```bash
# Table set implementation
grep -rn "WasmTableObject::Set\|table\.set\|TableSet" src/wasm/wasm-objects.cc | head -15

# Table type validation
grep -rn "table.*type\|element_type\|IsSubtypeOf.*table" src/wasm/wasm-objects.cc | head -15

# ProcessImportedTable
grep -rn "ProcessImportedTable" src/wasm/module-instantiate.cc | head -10

# Table operations in decoder
grep -rn "kExprTableSet\|kExprTableGet\|kExprTableCopy" src/wasm/function-body-decoder-impl.h | head -15

# Exactness in table context
grep -rn "exact.*table\|table.*exact\|is_exact.*element" src/wasm/ --include="*.cc" --include="*.h" | head -10
```

### Untested Surface Analysis
- **table.set with exact descriptor type**: table element type is `(ref exact $Desc)`. Stored value type is `(ref $DescSub)` where $DescSub subtypes $Desc. Exactness should REJECT this (exact = no subtypes). Does table.set enforce this?
- **table.get + ref.get_desc**: Get value from table with descriptor element type. Use ref.get_desc on returned value. Does the returned value carry correct descriptor information?
- **Imported table with descriptor elements**: Module A exports table with `(ref $Desc)` elements. Module B imports expecting `(ref exact $Desc)` elements. Should import fail? Does it?

## Techniques That Work
1. Build module with custom descriptor types: `$Desc` describes `$Struct`, `$DescSub` subtypes `$Desc`
2. Create table with element type `(ref null $Desc)` or `(ref null exact $Desc)`
3. table.set with values of $Desc and $DescSub types
4. table.get + ref.get_desc on retrieved values
5. Cross-module: export table, import with different exactness in element type
6. Run with `--no-liftoff --turboshaft-wasm` vs `--liftoff --no-wasm-tier-up`
7. Use `--experimental-wasm-custom-descriptors`

## DO NOT Try These
- IsCastToCustomDescriptor optimizer guard — covered (8 iters, CLEAN)
- Inlining vs standalone exactness — covered (3 iters, CLEAN)
- Shared + descriptor types — UNIMPLEMENTED guard prevents creation
- configureAll Liftoff stack state — covered (CLEAN)
- Simple table operations without descriptors — well-tested

## Evolution Plan
- v1: **Source audit of WasmTableObject::Set**. Read wasm-objects.cc to find the Set implementation. Trace the type validation: does it use IsSubtypeOf? Does IsSubtypeOf handle exact descriptor types correctly when called from the table path? Also check table.get — does the returned value's type annotation include descriptor info?
- v2: **Exploit exact + descriptor in table.set**. Build module: table with `(ref null exact $Desc)` element type. Try table.set with `(ref $DescSub)` value. Expected: trap (exact rejects subtypes). If it succeeds, the value with wrong type is now in the table. Then table.get + ref.get_desc to trigger type confusion.
- v3: **Cross-module table import with descriptor types**. Module A exports table with `(ref null $Desc)` elements containing $DescSub values. Module B imports table expecting `(ref null exact $Desc)` elements. Get value from table — is it treated as exact $Desc even though it's actually $DescSub?

## Task
Start with v1. Read these files IN THIS ORDER:

1. `src/wasm/wasm-objects.cc` — Search for `WasmTableObject::Set` and `WasmTableObject::Get`. Trace the type validation logic. What check does Set perform on the stored value's type? Does it call IsSubtypeOf? How does it handle the table's element type?
2. `src/wasm/function-body-decoder-impl.h` — Search for `kExprTableSet`. What compile-time type checking does the decoder perform? Does it validate exactness of the value against the table element type?
3. `src/wasm/wasm-subtyping.cc` — Read the IsSubtypeOf logic for exact types. If type A is `(ref exact $Desc)` and type B is `(ref $DescSub)`, does IsSubtypeOf(B, A) correctly return false?
4. `src/wasm/module-instantiate.cc:1982-2060` — ProcessImportedTable. What type compatibility checks are performed for imported tables with descriptor element types?

Your type invariant to challenge: "Table operations correctly enforce exactness and descriptor type relationships on their element types." The 7+ descriptor bug fixes never touched table paths — find the gap.

Post with tags `table,custom-descriptor,exactness,runtime,recombine`.
