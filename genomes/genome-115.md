# Genome 115 — deopt-throw-gcref

## Hypothesis
LazyDeoptOnThrow is V8's mechanism for deferring exception handler code generation. Commit adc761a24dc added a bailout: Wasm body inlining SKIPS functions with LazyDeoptOnThrow::kYes, leaving a TODO. The deopt frame builder (LiftoffFrameDescriptionForDeopt at wasm-deopt-data.h:97-108) must preserve Wasm GC refs across deoptimization boundaries. If LiftoffVarState doesn't correctly distinguish tagged GC refs from untagged integers in deopt frames, or if the GC visitor doesn't scan deopt frame slots for refs, a struct/array ref could be treated as an integer after deopt — either missed by GC (UAF) or misinterpreted (type confusion).

## DNA

### Source Facts (file:line)
- **LazyDeoptOnThrow enum**: src/compiler/globals.h:27 — `enum class LazyDeoptOnThrow : uint8_t { kNo, kYes }`
- **TSCallDescriptor field**: src/compiler/turboshaft/operations.h:4303 — `LazyDeoptOnThrow lazy_deopt_on_throw`
- **Bailout on inlining**: src/compiler/turboshaft/wasm-in-js-inlining-reducer-inl.h:1407-1414 — bails out when LazyDeoptOnThrow::kYes
- **Deopt data structure**: src/wasm/wasm-deopt-data.h:97-108 — LiftoffFrameDescriptionForDeopt
- **LiftoffVarState**: src/wasm/baseline/liftoff-varstate.h:13-100 — tracks register/stack state for locals
- **Recent fix**: commit adc761a24dc — "Improve test and bailout for LazyDeoptOnThrow"
- **Wasm wrappers deopt**: src/wasm/wrappers.cc:43 — uses LazyDeoptOnThrow::kNo
- **Turboshaft interface**: src/wasm/turboshaft-graph-interface.cc:8286,8321 — sets LazyDeoptOnThrow::kNo for GC ops

### Gap Analysis
1. **LiftoffVarState tagging**: Does LiftoffVarState correctly mark GC ref slots vs integer slots? If deopt frame restores a ref slot as kIntConst or untagged kRegister, GC won't trace it.
2. **Frame state builder**: When building FrameState for deopt, does the Wasm path correctly include GC ref map? Compare with JS deopt frame builder.
3. **Cross-tier deopt**: Turboshaft → Liftoff deopt: Turboshaft may allocate refs differently (registers vs stack). Does the deopt translation preserve ref-ness?
4. **GC during deopt materialization**: If GC fires while materializing the deopt frame, are the refs in the partially-built frame visible to the GC?

### Related CVEs/Commits
- adc761a24dc: "Improve test and bailout for LazyDeoptOnThrow" (March 2026)
- TODO at wasm-in-js-inlining-reducer-inl.h: "Support lazy deopts in Wasm"

## Techniques That Work
1. Create Wasm function that throws, with try-catch holding GC struct refs in locals
2. Trigger tier-up (Liftoff → Turboshaft) mid-execution, force deopt back to Liftoff
3. Use --wasm-deopt to enable Wasm deoptimization
4. Force GC during the throw/catch/deopt sequence
5. Verify ref integrity after deopt by accessing struct fields

## DO NOT Try These
- Simple try-catch without deopt trigger — already well-tested
- WasmFX stack switching (separate subsystem, 9 workers found nothing)
- configureAll patterns (dead surface)

## Evolution Plan
- v1: Source audit of wasm-deopt-data.h and liftoff-varstate.h. Understand how GC refs are encoded in deopt frames. Search for LiftoffVarState usage in deopt frame construction. Map all code paths that build deopt frames for Wasm.
- v2: Build test: Wasm function with struct locals, throws exception, catches it, accesses struct after catch. Run with --wasm-deopt + tier-up flags. Diff behavior: does the struct ref survive deopt correctly?
- v3: Force GC during deopt materialization. Use --gc-interval=1 to maximize GC pressure during deopt. Check if partially-built deopt frames have correct GC maps.

## Task
1. Read src/wasm/wasm-deopt-data.h (full file) — understand LiftoffFrameDescriptionForDeopt structure.
2. Read src/wasm/baseline/liftoff-varstate.h (full file) — understand how ref types are tracked.
3. Search for "deopt" in src/wasm/baseline/liftoff-compiler.cc — find where deopt frames are constructed.
4. Search for "LiftoffFrameDescriptionForDeopt" across the codebase to find all construction sites.
5. Search for "tagged\|kRef\|is_ref" in wasm-deopt-data.h and liftoff-varstate.h — verify ref tracking.
6. Build test following v1 plan.
7. Run with: `--wasm-deopt --no-wasm-lazy-compilation --liftoff --wasm-tiering-budget=1`
8. Start with Evolution Plan v1 (source audit of deopt frame ref tracking).
