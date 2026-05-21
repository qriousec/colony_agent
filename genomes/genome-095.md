# Genome 095 — wasmfx-contbind-type

## Hypothesis

WasmFX cont.bind (implemented in commit a93849fa707) stores bound reference arguments in a StackMemory argument buffer using right-to-left layout. Two recent fixes corrected compressed/uncompressed pointer confusion:
- **5a64071bf9d**: Return buffer stored references as compressed but expected uncompressed
- **2f8c6526740**: Same bug in stack entry parameters

The bound argument buffer in StackMemory may have the SAME compressed pointer confusion. When cont.bind binds a reference parameter (e.g., a WasmStruct or WasmArray), and the continuation is later resumed, the reference is read from the buffer. If it was stored compressed but read uncompressed (or vice versa), the pointer is garbage — on little-endian systems this could point to a valid but WRONG object, causing type confusion. On big-endian systems this causes crashes (which is how 2f8c6526740 was discovered).

Additionally, cont.bind changes the continuation's expected parameter types (stored in StackMemory::param_types_) but does NOT re-validate already-bound arguments against the new type vector. If a continuation expects `(ref $A, ref $B)` and we bind a `(ref $A)` first, then bind again with a different continuation type, the already-bound argument may violate the new type expectation.

## DNA

### Source Facts

1. **cont.bind implementation** — `src/wasm/turboshaft-graph-interface.cc:~5600-5700`
   - Builds argument buffer, stores bound args right-to-left
   - Updates param_types_ vector on StackMemory
   - Calls Runtime::kWasmContBind

2. **StackMemory argument storage** — `src/wasm/stacks.h:~100-150` and `src/wasm/stacks.cc:~50-100`
   - `param_types_` stores vector of expected parameter types
   - `num_bound_args_` tracks how many args are already bound
   - GC must visit bound references (added in a93849fa707)

3. **Compressed pointer confusion FIX 1** — `src/wasm/wrappers-inl.h` (5a64071bf9d)
   - Return buffer: references stored as compressed pointers
   - Expected as uncompressed by caller
   - Fix: store with `MemoryRepresentation::AnyTagged()` instead of compressed

4. **Compressed pointer confusion FIX 2** — `src/wasm/wrappers-inl.h` (2f8c6526740)
   - Same issue but in parameter passing for stack entry
   - Applied similar fix to parameters

5. **GC visiting bound references** — `src/wasm/stacks.cc` (in VisitStack or similar)
   - Must correctly identify which slots contain references vs non-references
   - Uses param_types_ to determine which bound args are references
   - If param_types_ is stale after multiple cont.binds, GC may miss references

6. **cont.bind type checking** — `src/wasm/function-body-decoder-impl.h`
   - Decoder validates cont.bind at decode time: checks that bound arg types match first N params
   - But RUNTIME state (after multiple cont.binds) may diverge from decode-time assumptions

7. **Related bugs found by wasmfx-jspi-stack-reuse**: UNIMPLEMENTED() crash at `wasm-external-refs.cc:1102` for kOnSwitch handler. Shows WasmFX has reachable UNIMPLEMENTED paths.

8. **Stale reference fix** (99d0d485292): StackMemory kept stale stack_obj_ after retirement. Stack pool could pick up retired stack with stale reference. Shows GC interaction is fragile in this subsystem.

### Key Files to Read
- `src/wasm/stacks.h` — StackMemory class, param_types_, bound args
- `src/wasm/stacks.cc` — StackMemory implementation, GC visiting
- `src/wasm/turboshaft-graph-interface.cc` — search for `ContBind` or `cont.bind`
- `src/wasm/wrappers-inl.h` — look for stack entry/exit parameter handling
- `src/wasm/wasm-external-refs.cc` — resume/suspend mechanics, argument buffer handling
- `src/runtime/runtime-wasm.cc` — Runtime::kWasmContBind implementation
- `test/mjsunit/wasm/cont-bind.js` — existing tests (find uncovered paths)

## Techniques That Work

1. **Create multi-param continuation types**: `(cont (func (ref $StructA) (ref $StructB) i32 -> i32))`
2. **Bind reference arguments with cont.bind**: Bind a WasmStruct as first param
3. **Trigger GC between cont.bind and resume**: Force GC to visit bound references
4. **Check if bound reference is still valid after GC**: Read struct fields, compare to expected values
5. **Use tier-up differential**: Run with `--liftoff-only` vs `--no-liftoff` — different code paths for argument buffer
6. **Multiple successive cont.binds**: Bind one param at a time, triggering GC between each
7. **Mix ref and non-ref parameters**: `(ref $Struct, i32, ref $Array, i64)` — test buffer layout with gaps

## DO NOT Try These

- Don't test kOnSwitch handler (already confirmed UNIMPLEMENTED crash by wasmfx-jspi-stack-reuse)
- Don't test JSPI stack allocation (already confirmed fixed by 12735419590)
- Don't test basic stack switching without cont.bind (already explored)

## Evolution Plan

- v1: **Source trace** — Read stacks.h/stacks.cc to find exactly how bound arguments are stored, how param_types_ is updated, and how GC visits bound references. Read wrappers-inl.h to find the compressed pointer fixes and identify if the cont.bind argument buffer uses the SAME storage mechanism (pre-fix pattern). Identify the exact MemoryRepresentation used for bound reference storage.

- v2: **Compressed pointer probe** — Construct a module with cont.bind that binds WasmStruct and WasmArray references. Verify the bound references survive GC by reading struct fields after resume. Use `--wasm-tier-mask-for-testing` to force Turboshaft compilation. Compare results with `--liftoff-only`. If there's a differential, the compressed pointer confusion is present.

- v3: **Multi-bind type confusion** — Create a scenario with multiple cont.binds. First cont.bind binds a `(ref $A)`. Second cont.bind (to a different continuation type) binds a `(ref $B)` where $B has different struct layout than $A. Test if the already-bound `(ref $A)` is re-validated or just left in the buffer. If not re-validated, resume may pass it where `(ref $B)` is expected.

## Task

**Start by reading these source files in order:**
1. `src/wasm/stacks.h` — Find StackMemory class, param_types_ vector, bound argument storage, GC visiting interface
2. `src/wasm/stacks.cc` — Find how GC visits bound references, how param_types_ is updated on cont.bind
3. `src/wasm/turboshaft-graph-interface.cc` — Search for `ContBind` to find the graph building for cont.bind. Note how arguments are stored in the buffer.
4. `src/wasm/wrappers-inl.h` — Search for the compressed pointer fixes (look for `MemoryRepresentation` near stack entry code). Compare to the cont.bind argument storage code.
5. `src/runtime/runtime-wasm.cc` — Search for `WasmContBind` runtime function

**Type invariant to challenge:** Bound reference arguments in the cont.bind argument buffer are correctly stored, correctly visited by GC, and correctly read on resume — with no compressed/uncompressed pointer confusion.

**Compilation path:** Turboshaft compiled path (force with `--no-liftoff --wasm-tier-mask-for-testing=2`)

**Start with Evolution Plan v1** — trace the source to find the exact storage mechanism for bound arguments before writing any test code.
