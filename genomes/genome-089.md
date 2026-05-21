# Genome 089 — shared-table-dispatch

## Hypothesis
`FunctionSigMatchesTable` at wasm-objects.cc:526 has `DCHECK(!table_type.is_shared())` with an explicit comment "This code will need updating." The shared table infrastructure EXISTS — modules can declare shared tables (`module->tables[i].shared = true`), module instantiation creates shared table objects (module-instantiate.cc:1300), and dispatch tables are allocated for them (module-instantiate.cc:1308). But the dispatch table UPDATE code (`UpdateDispatchTable`, `SetForWrapper`) assumes ALL tables are non-shared: `kShared` is hardcoded to `false` at line 584, and `FunctionSigMatchesTable` doesn't account for shared types in subtype checks. If shared table operations reach this code, debug builds crash at the DCHECK and release builds perform incorrect subtype checks (shared bit ignored in comparison), potentially allowing type-mismatched functions to be installed in shared tables.

## DNA
### Critical Code Locations
- **wasm-objects.cc:523-543**: `FunctionSigMatchesTable` — THE TARGET
  ```cpp
  DCHECK(!table_type.is_shared());  // This code will need updating.
  ```
  The function checks if a function signature is compatible with the table type. For shared tables, this check is WRONG because it ignores the shared bit.

- **wasm-objects.cc:580**: `SBXCHECK(FunctionSigMatchesTable(sig_id, dispatch_table->table_type()))` — called from `UpdateDispatchTable`. SBXCHECK = sandbox check. If this check is wrong, sandbox escape is possible.

- **wasm-objects.cc:584**: `constexpr bool kShared = false;` — HARDCODED. When installing a function into a dispatch table, the WasmImportData is ALWAYS created as non-shared, even for shared tables.

- **wasm-objects.cc:651**: Second `SBXCHECK(FunctionSigMatchesTable(...))` call.

- **wasm-objects.cc:715**: Third `SBXCHECK(FunctionSigMatchesTable(...))` call.

### Shared Table Infrastructure
- **module-instantiate.cc:1300**: Table creation uses `table.shared` to select trusted data
- **module-instantiate.cc:1306**: Shared tables stored in `shared_tables`
- **module-instantiate.cc:1308**: Shared dispatch tables created
- **module-instantiate.cc:1504,1986**: Shared table access in other instantiation paths

### Attack Vectors
1. **Direct table.set with shared funcref**: Create shared table, store function via table.set
2. **Import resolution**: Import a shared table and populate via module instantiation
3. **table.init with shared element segment**: Initialize shared table from element segment
4. **Cross-module table sharing**: Share a table between modules, one shared and one not

### Related Patterns
- **P8**: Explicit "needs updating" DCHECK — same pattern as AcqRel where code explicitly acknowledges incomplete support
- **P6**: GC assumptions about shared space — shared table dispatch entries may violate these

## Techniques That Work
1. Create Wasm module with shared table: `(table (shared) 1 funcref)` or shared typed funcref
2. Declare shared function types for table entries
3. Use table.set to store functions into shared table
4. Use call_indirect through shared table
5. Force compilation to both Liftoff and Turboshaft
6. Monitor for DCHECK failures in debug build
7. Test with `--experimental-wasm-shared`

## DO NOT Try These
- Non-shared table operations — well-tested
- Shared struct/array at JS APIs — already confirmed (TC-001 + shared-nonstring-boundary)
- String unsharing — already confirmed (TC-003/004/005)

## Evolution Plan
- v1: Determine if shared tables can be created from JS/Wasm. Read WasmModuleBuilder API for shared table declaration. Check if `(table $t (shared) 1 funcref)` or similar syntax is supported. Try to create a module with a shared table using WasmModuleBuilder. If creation succeeds, try table.set to trigger FunctionSigMatchesTable.
- v2: If shared table creation works, test UpdateDispatchTable path. Create shared table, install functions via table.set and imports. Monitor for DCHECK crash at line 526. In release build, check if wrong functions can be installed (subtype check ignoring shared bit).
- v3: Test call_indirect through shared table. If a type-mismatched function is installed (due to missing shared check), calling it via call_indirect could cause type confusion at the call site — wrong number/types of arguments.

## Task
Start by reading:
1. `src/wasm/wasm-objects.cc:523-543` — FunctionSigMatchesTable with DCHECK
2. `src/wasm/wasm-objects.cc:545-620` — UpdateDispatchTable (where it's called)
3. `src/wasm/module-instantiate.cc:1280-1320` — table creation with shared flag
4. `src/wasm/wasm-module-builder.cc` — can WasmModuleBuilder create shared tables? Search for `shared` + `table`
5. `test/common/wasm/wasm-module-builder.h` — table builder API for shared flag

Key test: Create `(table (shared) 10 (ref null (shared func)))`. Then `(table.set $t (i32.const 0) (ref.func $f))`. If $f's type is checked by FunctionSigMatchesTable with shared table type → DCHECK crash.

Use `--experimental-wasm-shared` with `/Users/t/v8/out/fuzzbuild/d8`.
