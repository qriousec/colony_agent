# Genome 117 — acqrel-shared-struct-release

## Hypothesis
Reland c5ba3206101 added AcqRel Turboshaft support for MEMORY instructions only. For non-memory Wasm GC operations (StructGet/Set, ArrayGet/Set), AcqRel handling is inconsistent: StructSet/ArraySet have DCHECK_NE(memory_order, kAcqRel) at wasm-lowering-reducer.h:277/368, but StructGet/ArrayGet have NO DCHECK — they silently accept AcqRel and produce `.Atomic()` load/store kind. In release builds (DCHECKs stripped), this means: (1) Get operations silently use whatever ordering .Atomic() defaults to, (2) Set operations have undefined behavior. On shared WasmGC structs with concurrent access from multiple isolates, if Liftoff correctly implements AcqRel ordering but Turboshaft doesn't, there's an observable tier differential in multi-threaded scenarios — values become visible in wrong order.

## DNA

### Source Facts (file:line)
- **StructSet DCHECK**: wasm-lowering-reducer.h:277 — `DCHECK_NE(memory_order, AtomicMemoryOrder::kAcqRel)` (added by c5ba3206101)
- **ArraySet DCHECK**: wasm-lowering-reducer.h:368 — same DCHECK
- **StructGet NO DCHECK**: wasm-lowering-reducer.h:~230-270 — `if (memory_order.has_value()) { load_kind = load_kind.Atomic(); }` — no AcqRel check
- **ArrayGet NO DCHECK**: wasm-lowering-reducer.h:~345-360 — same, no AcqRel check
- **TODO comment**: wasm-lowering-reducer.h:259-260 — "TODO(mliedtke): Support acquire release semantics if specified"
- **Reland**: c5ba3206101 — "[acqrel] Add Turboshaft support for relaxed atomics" (reverted once)
- **Original revert**: add9a87b8c0 — reverted due to missing TSAN release store
- **Liftoff AcqRel**: src/wasm/baseline/liftoff-compiler.cc — implements AcqRel differently (architecture-specific)
- **acqrel-array-silent finding**: Post 9f63cc3c confirmed ArraySet DCHECK crash in debug, ArrayGet silent pass

### Gap Analysis
1. **Release build behavior**: In release builds, DCHECK_NE is a no-op. What code actually executes for AcqRel struct/array operations in Turboshaft? Does `.Atomic()` produce SeqCst or AcqRel or something else?
2. **Liftoff vs Turboshaft ordering**: Does Liftoff correctly implement AcqRel for struct/array atomics? If yes and Turboshaft doesn't, there's a tier differential.
3. **Shared struct concurrent access**: When two workers access a shared struct's i32 field with AcqRel ordering, does one tier see updates in a different order than the other?
4. **Store ordering on Set**: In Turboshaft, the `__ Store()` call at line 302 now passes `memory_order` parameter (added by reland). But the DCHECK says AcqRel is NOT supported. What does the backend do with an AcqRel Store for non-memory operations?

### Confirmed Signal
acqrel-array-silent (post 9f63cc3c) confirmed:
- ArraySet with AcqRel crashes in debug build (DCHECK failure)
- ArrayGet with AcqRel works silently (no DCHECK)
- Liftoff handles both without crashing

## Techniques That Work
1. Build module with shared struct having i32 field, use atomic.struct.get/set with AcqRel ordering
2. Run in RELEASE build (or use d8 release binary) to bypass DCHECK
3. Multi-threaded test: Worker writes with AcqRel, main thread reads with AcqRel — check ordering
4. Compare tier outputs: run same test with --liftoff-only vs --no-liftoff
5. Use --experimental-wasm-shared for shared struct support
6. Use memory fence patterns to make ordering differences observable

## DO NOT Try These
- Debug build testing (DCHECK will crash before reaching the interesting behavior)
- Simple single-threaded tests (ordering differences only observable with concurrency)
- Non-atomic struct access (need explicit AcqRel memory_order annotation)

## Evolution Plan
- v1: Source audit of LoadOp::Kind::Atomic() and StoreOp::Kind::Atomic() — what memory ordering do they produce by default? Read src/compiler/turboshaft/operations.h for LoadOp/StoreOp definitions. Check if Atomic() implies SeqCst or if the memory_order parameter controls it. Read Liftoff's atomic struct access implementation for comparison.
- v2: Build multi-threaded test with shared struct. Use Worker to concurrently read/write a shared struct's i32 field with AcqRel ordering. Check if values appear in consistent order on both tiers. Run with release d8 binary.
- v3: If tier differential found, minimize. If not found, test with stress flags: --stress-compaction, --gc-interval=100. Check if GC during concurrent AcqRel access produces different behavior across tiers.

## Task
1. Read wasm-lowering-reducer.h:230-310 (StructGet/StructSet lowering with memory_order handling).
2. Read wasm-lowering-reducer.h:340-380 (ArrayGet/ArraySet lowering with memory_order handling).
3. Read src/compiler/turboshaft/operations.h — search for "Atomic\|memory_order\|MemoryOrder" to understand how .Atomic() interacts with memory ordering.
4. Read src/wasm/baseline/liftoff-compiler.cc — search for "atomic.*struct\|struct.*atomic\|AcqRel" to find Liftoff's implementation.
5. Check if d8 at /Users/t/v8/out/fuzzbuild/d8 is a release or debug build (run d8 --help and look for DCHECK-related flags, or just try the AcqRel ArraySet crash test).
6. Build test following v2 plan: shared struct + Worker + AcqRel get/set.
7. Run with: `--experimental-wasm-shared --no-wasm-lazy-compilation`
8. Start with Evolution Plan v1 (source audit of Atomic() load/store kind).
