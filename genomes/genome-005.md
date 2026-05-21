# Genome 005 — desc-exactness

## Hypothesis
`IsCastToCustomDescriptor` (wasm-gc-typed-optimization-reducer.h:21-26) is structurally dead — it checks `config.exactness == kExactMatchOnly`, but `GetExactness` (wasm-compiler-definitions.cc:21-39) returns `kExactMatchLastSupertype` for ALL final/exact types with descriptors. This means the guard never fires for descriptor types, allowing the Turboshaft optimizer to eliminate `ref.cast` and `ref.test` operations targeting descriptor types based on nominal `IsHeapSubtypeOf` alone. If `ref.cast_desc_eq` ALSO goes through `GetExactness`, it too gets `kExactMatchLastSupertype` and the guard never fires — meaning descriptor equality checks can be eliminated by the optimizer when it infers the value is already the right nominal type, even though the descriptor identity hasn't been verified.

## DNA

### Source Facts (from parent genome-004 + queen cycle 2 analysis)

- **IsCastToCustomDescriptor** (`wasm-gc-typed-optimization-reducer.h:21-26`):
  ```cpp
  inline bool IsCastToCustomDescriptor(const wasm::WasmModule* module,
                                       WasmTypeCheckConfig config) {
    return config.to.has_index() &&
           module->type(config.to.ref_index()).has_descriptor() &&
           config.exactness == compiler::kExactMatchOnly;
  }
  ```
  Checks for `kExactMatchOnly` ONLY.

- **GetExactness** (`wasm-compiler-definitions.cc:21-39`):
  ```cpp
  if (type.is_final || target.is_exact()) {
    return type.has_descriptor()
               ? SubtypeCheckExactness::kExactMatchLastSupertype
               : SubtypeCheckExactness::kExactMatchOnly;
  }
  ```
  For final/exact types WITH descriptor: returns `kExactMatchLastSupertype` (NOT `kExactMatchOnly`).

- **Runtime lowering for kExactMatchLastSupertype** (`wasm-lowering-reducer.h:897-908`):
  ```cpp
  V<Object> maybe_match = LoadImmediateSuperRTT(map);
  __ TrapIfNot(__ TaggedEqual(maybe_match, rtt.value()), TrapId::kTrapIllegalCast);
  ```
  Checks immediate supertype RTT — this is checking the canonical type map, NOT the object's own map (which includes descriptor identity).

- **WasmTypeCast elimination** (`wasm-gc-typed-optimization-reducer.h:274-280`):
  ```cpp
  if (wasm::IsHeapSubtypeOf(...) && !IsCastToCustomDescriptor(module_, cast_op.config)) {
    // cast removed
  }
  ```
  Since `IsCastToCustomDescriptor` returns false for descriptor types, the cast IS removed.

- **WasmTypeCheck elimination** (`wasm-gc-typed-optimization-reducer.h:341-345`):
  Same pattern — guard returns false for descriptor types, check is replaced with constant `1`.

- **wasm-inlining-into-js.cc:300 INCONSISTENCY**: Uses `kExactMatchOnly` for final types (doesn't call `GetExactness`). This means inlining and standalone compilation may use different exactness for the same cast.

- **Parent findings**: custom-desc-cast found that `testInvalidCast` returns 99 in both Liftoff and TurboFan. Need to determine if this is expected semantics or a bug.

### Critical Question
**Does `ref.cast_desc_eq` go through `GetExactness` or does it set its own exactness?**

If `ref.cast_desc_eq` has its own exactness path (setting `kExactMatchOnly` directly), then:
- The guard correctly protects `ref.cast_desc_eq` ✓
- But `ref.cast $DescType` (nominal) is unguarded — this is only a bug if `kExactMatchLastSupertype` carries descriptor semantics

If `ref.cast_desc_eq` goes through `GetExactness`, then:
- The guard NEVER fires for ANY descriptor cast ✗
- **This is a bug** — descriptor equality checks can be eliminated

### Guards Encountered
- `IsCastToCustomDescriptor` — checks wrong exactness constant
- Both WasmTypeCast and WasmTypeCheck reducers have the guard
- ProcessBranchOnTarget line 447-451 also has the guard for false-branch unreachability
- Runtime lowering correctly dispatches on exactness

## Techniques That Work
1. **Trace the ref.cast_desc_eq compilation path**: Find where `ref.cast_desc_eq` bytecode is decoded in `function-body-decoder-impl.h`, how it creates the WasmTypeCastOp, and what exactness it sets.
2. **Search for kExprRefCastDescEq**: This is the opcode. Trace from decoder to graph IR to find the config.
3. **Differential test**: Create a module with both `ref.cast_desc_eq` and `ref.cast $DescType`. If the type analyzer infers the input type, does the optimizer eliminate either/both?
4. **Three-tier differential**: Compare Liftoff (baseline), Turboshaft without optimization, Turboshaft with optimization.
5. **Construct the exploit**: Two objects of same nominal type but different descriptors. After optimization eliminates the descriptor check, `ref.get_desc` should return wrong descriptor.

## DO NOT Try These
- Testing the guard with non-descriptor types (guard correctly returns false)
- Testing with non-final types (GetExactness returns kMayBeSubtype, no exactness issue)
- Basic ref.cast tests without descriptors (well-tested, correct)
- Testing wasm-inlining-into-js.cc inconsistency (TurboFan path, being removed)

## Evolution Plan
- v1: Trace `ref.cast_desc_eq` from bytecode decoder through graph building to find its exactness. Search for `kExprRefCastDescEq` in `function-body-decoder-impl.h` and `turboshaft-graph-interface.cc`. Determine if it uses `GetExactness` or sets its own config. This is the KEY question.
- v2: If v1 confirms the guard is dead for `ref.cast_desc_eq`, construct a differential test: allocate struct with descriptor D1, cast with `ref.cast_desc_eq` targeting D2, and check if optimization eliminates the check while Liftoff correctly traps. If v1 shows `ref.cast_desc_eq` has correct exactness, pivot to testing `ref.cast $DescType` elimination — is it semantically correct to eliminate it?
- v3: If v2 finds a differential, minimize the test case and identify exact file:line in the optimizer where the incorrect elimination happens. Create exploit that reads wrong descriptor value.
- v4: If v2 is clean, investigate the wasm-inlining-into-js.cc inconsistency (kExactMatchOnly vs kExactMatchLastSupertype) and test if inlined code behaves differently.

## Task
Start by reading these files in order:
1. Search for `kExprRefCastDescEq` or `ref.cast_desc_eq` or `CastDesc` in `src/wasm/function-body-decoder-impl.h` — find the decoder path
2. Search for `cast_desc` or `CastDesc` in `src/wasm/turboshaft-graph-interface.cc` — find graph IR creation
3. `src/compiler/wasm-compiler-definitions.cc:21-39` — confirm GetExactness logic
4. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h:21-26` and lines 251-345 — confirm guard logic
5. `src/compiler/turboshaft/wasm-lowering-reducer.h:885-920` — understand runtime lowering for each exactness

The type invariant to challenge: "The IsCastToCustomDescriptor guard correctly prevents elimination of all casts that check descriptor identity."

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc --experimental-wasm-custom-descriptors --no-wasm-lazy-compilation --trace-wasm-typer`
