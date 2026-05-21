# Genome 082 — decoder-compiler-mismatch

## Hypothesis
When the Wasm decoder accepts a feature/parameter combination (e.g., memory order, opcode variant) that Turboshaft's compiler backend does not fully implement, DCHECK/UNREACHABLE paths are hit in debug builds but silently produce undefined behavior in release builds. The AcqRel tier differential (3 bugs confirmed) is one instance of a SYSTEMATIC pattern where decoder validation is more permissive than compiler coverage.

## DNA
### Confirmed Pattern (Parent: genome-080 wasm-atomic-acqrel-tier)
- **function-body-decoder-impl.h:1127-1141**: Decoder validates atomic memory order ≤ kAcqRel for struct/array atomic ops — ACCEPTS AcqRel
- **wasm-lowering-reducer.h:277**: `DCHECK_NE(memory_order, AtomicMemoryOrder::kAcqRel)` in StructSet — REJECTS
- **wasm-lowering-reducer.h:368**: Same DCHECK in ArraySet — REJECTS
- **wasm-lowering-reducer.h:259-261**: TODO but NO DCHECK in StructGet — SILENT UB
- **Impact**: Liftoff handles AcqRel correctly. Turboshaft crashes (debug) or UB (release).

### Known DCHECK/UNREACHABLE/UNIMPLEMENTED Sites in Turboshaft Wasm Reducers
1. **wasm-lowering-reducer.h:672**: UNREACHABLE in RepresentationFor (kVoid/kTop/kBottom)
2. **wasm-lowering-reducer.h:761**: UNREACHABLE in CastChecks for unhandled GenericKind
3. **wasm-lowering-reducer.h:847**: UNREACHABLE in TrapOnSharedWasmObjectsIfUnshared
4. **wasm-revec-reducer.h:1333,1352,1364,1377**: 4x UNIMPLEMENTED() in SIMD vectorization op mapping
5. **wasm-in-js-inlining-reducer-inl.h:355,533**: DCHECK_WITH_MSG(false) for unhandled UnOp/BinOp (with bailout)
6. **wasm-in-js-inlining-reducer-inl.h:648**: UNREACHABLE for multi-value returns in wrapper inlining

### Feature Areas to Scan
- **Custom descriptors**: ref.cast_desc_eq, configure_all — recent fix a58753bac7b
- **Shared types**: shared struct/array atomic ops beyond AcqRel
- **Wide arithmetic**: 37753784ee1 "add opcodes and placeholders" — PLACEHOLDERS = incomplete
- **WasmFX**: cont.new, cont.bind, resume, suspend — Turboshaft-only (no Liftoff)
- **SIMD revec**: Auto-vectorization of 128→256 bit ops

## Techniques That Work
1. Build Wasm module with feature-specific bytecodes in Wasm binary (use d8 --experimental-wasm-* flags)
2. Force Turboshaft compilation with `--wasm-tier-mask-for-testing=2` or warmup loops
3. Compare debug build (DCHECK fires) vs release behavior (silent UB)
4. Grep for DCHECK/UNIMPLEMENTED/UNREACHABLE in Turboshaft wasm reducers
5. Trace decoder acceptance: grep function-body-decoder-impl.h for opcode handlers
6. Check if Liftoff handles what Turboshaft doesn't: `--liftoff-only` vs `--no-liftoff`

## DO NOT Try These
- AcqRel struct/array atomic ops — already confirmed by parent genome-080 (3 bugs)
- Generic WasmFX cont.new/resume without specific feature mismatch angle
- Testing features that are NOT behind experimental flags (well-tested in production)

## Evolution Plan
- v1: Scan wide arithmetic opcodes (37753784ee1 "placeholders"). Check if decoder accepts wide arithmetic bytecodes but Turboshaft has placeholder/TODO implementations. Focus on type handling for 128-bit integer results.
- v2: Scan custom descriptor opcodes in Turboshaft. The configureAll fix (a58753bac7b) was for Liftoff — does Turboshaft have COMPLETE support for all custom descriptor operations? Trace ref.cast_desc_eq, ref.test_desc, br_on_cast_desc through Turboshaft pipeline.
- v3: Check SIMD revec UNIMPLEMENTED() reachability. Can user-written Wasm SIMD code trigger the 4 UNIMPLEMENTED() paths in wasm-revec-reducer.h? If so, these are exploitable crashes.

## Task
Start by reading:
1. `src/wasm/function-body-decoder-impl.h` — search for wide arithmetic opcode handlers (grep "wide_arith" or "WideArith")
2. `src/compiler/turboshaft/wasm-lowering-reducer.h` — search for wide arithmetic handling
3. `37753784ee1` commit diff — what exactly are the "placeholders"?
4. `src/wasm/baseline/liftoff-compiler.cc` — does Liftoff handle what Turboshaft doesn't for these features?

For each feature area, build the acceptance/rejection matrix:
| Feature | Decoder accepts? | Liftoff handles? | Turboshaft handles? | Gap? |

The type invariant to challenge: "Every Wasm bytecode sequence accepted by the decoder can be correctly compiled by Turboshaft." The AcqRel bugs prove this is false — find the next false assumption.

Start with v1 (wide arithmetic). Use `/Users/t/v8/out/fuzzbuild/d8` with `--experimental-wasm-wide-arith` if the flag exists.
