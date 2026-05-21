# Genome 080 — wasm-atomic-acqrel-tier

## Hypothesis
Turboshaft contains `DCHECK_NE(memory_order, kAcqRel)` for `array.atomic.set` and `struct.atomic.set` operations. The Wasm decoder ACCEPTS AcqRel memory order (raw_order=1, defined in wasm-constants.h). Liftoff handles AcqRel silently — on x64 it emits a regular store. This creates a tier differential:

1. **Liftoff (baseline)**: Compiles AcqRel atomic operations without error, emitting platform-specific stores
2. **Turboshaft (optimized)**: Hits `DCHECK_NE(memory_order, kAcqRel)` → crash in debug builds, undefined behavior in release builds

On release builds where DCHECK is stripped, Turboshaft code for AcqRel atomics executes whatever fallthrough logic exists — potentially wrong memory ordering, wrong store semantics, or type-confused operations. The random module generator in `fuzzing/random-module-generation.cc` actively generates both memory orders, confirming this is a reachable path.

**Attack angles:**
1. **DCHECK-removed undefined behavior**: What does Turboshaft actually emit for AcqRel `array.atomic.set`/`struct.atomic.set` in release mode? If it falls through to SeqCst logic, the behavior may be benign. But if it indexes into an array/table using the memory_order value, the wrong path is taken.
2. **Tier-up differential**: Function runs in Liftoff first (AcqRel works), then tier-up to Turboshaft changes semantics mid-execution. Observable state difference between tiers = correctness bug.
3. **Shared struct atomics**: `struct.atomic.set` with AcqRel on shared structs in shared memory — the wrong memory ordering could cause data races visible to other threads.

## DNA

### Source Facts — Memory Order Constants
- **wasm-constants.h**: `kAtomicSeqCst=0`, `kAtomicAcqRel=1`
- **Decoder**: Accepts both values (does NOT reject AcqRel at decode time)
- **Liftoff**: Handles both — on x64, AcqRel uses regular store (implicit acquire/release on x86 TSO)
- **Turboshaft**: `DCHECK_NE(memory_order, kAcqRel)` — assertion fails, then undefined behavior in release

### Source Facts — Affected Operations
- `array.atomic.set` (opcode 0xfe6a)
- `struct.atomic.set` (opcode in struct atomic family)
- `array.atomic.get` and `struct.atomic.get` may also have AcqRel handling gaps
- `array.atomic.rmw.*` and `struct.atomic.rmw.*` — check if AcqRel is rejected for RMW operations

### Source Facts — Turboshaft DCHECK Location
- **turboshaft-graph-interface.cc**: Search for `DCHECK_NE.*memory_order.*kAcqRel` or `DCHECK_NE.*kAcqRel`
- The DCHECK is in the code path that lowers atomic struct/array operations to machine operations
- After DCHECK removal in release, the switch/if statement falls through or takes wrong branch

### Source Facts — Fuzzer Confirmation
- **fuzzing/random-module-generation.cc**: Random module generator creates atomic operations with both SeqCst and AcqRel memory orders
- This confirms the code path IS reachable in production fuzzing

### Key Questions
1. What code does Turboshaft emit for AcqRel atomics in release mode? (Falls through to SeqCst? Takes wrong branch? Crashes differently?)
2. Does the decoder validate memory_order for ALL atomic operations, or only some?
3. Is there a difference between `struct.atomic.set` and `array.atomic.set` handling of AcqRel?
4. Can AcqRel be combined with shared GC types to create observable semantic differences between tiers?
5. Does the interpreter handle AcqRel atomics correctly?

## Techniques That Work
1. Create a Wasm module with `struct.atomic.set` and `array.atomic.set` using AcqRel memory order via raw bytecode
2. Run with `--liftoff-only` to verify Liftoff compiles and executes correctly
3. Run with `--no-liftoff` to verify Turboshaft behavior (DCHECK crash in debug, undefined in release)
4. Compare results: Liftoff execution output vs Turboshaft execution output for same module
5. Test with `--experimental-wasm-shared` for shared struct/array atomics
6. Use `%WasmTierUpFunction` to force tier-up mid-execution and observe state change
7. Build and test on both debug (expect DCHECK crash) and release (expect silent misbehavior)

## DO NOT Try These
- Non-atomic struct/array operations — well-tested standard path
- SeqCst memory order — correctly handled by all tiers
- Memory.atomic.* operations (i32/i64) — different code path, likely correctly validates memory order
- cont.bind GC roots — DEAD SURFACE

## Evolution Plan
- v1: Source audit — find all `DCHECK_NE.*kAcqRel` in turboshaft-graph-interface.cc. Map the exact code path for struct.atomic.set and array.atomic.set with AcqRel. Determine what happens in release mode when DCHECK is stripped. Also check decoder (function-body-decoder-impl.h) for memory_order validation. Create test: build module with raw bytecode containing array.atomic.set + AcqRel, run under both Liftoff and Turboshaft.
- v2: Test tier-up differential. Create a function that: (a) writes shared struct field with AcqRel atomic set, (b) reads it back. Run function in Liftoff loop, then force tier-up. If Turboshaft emits different code for AcqRel, the read-back value or memory ordering may change. Use a Worker thread to observe the difference.
- v3: Test all atomic operations for AcqRel handling: struct.atomic.get, array.atomic.get, struct.atomic.rmw.add, array.atomic.rmw.cmpxchg. Create a matrix of (operation × memory_order × tier) and check for inconsistencies.

## Task
Start by reading these files in order:
1. `src/wasm/wasm-constants.h` — search for `kAtomic` to find memory order enum values
2. `src/wasm/turboshaft-graph-interface.cc` — search for `kAcqRel` or `AcqRel` to find all DCHECK locations
3. `src/wasm/baseline/liftoff-compiler.cc` — search for `kAcqRel` or `AcqRel` to see how Liftoff handles it
4. `src/wasm/function-body-decoder-impl.h` — search for `memory_order` or `atomic_op` to check decoder validation
5. `src/wasm/fuzzing/random-module-generation.cc` — search for `kAcqRel` to confirm reachability

Name the type invariant to challenge: **All compilation tiers (Liftoff, Turboshaft, Interpreter) must produce identical observable behavior for the same valid Wasm bytecode. If the decoder accepts AcqRel memory order for atomic GC operations, all backends must either reject it consistently or handle it consistently.** The DCHECK in Turboshaft indicates it was never meant to reach that path, but the decoder allows it through.

Start with v1 — audit all AcqRel DCHECK locations and determine release-mode behavior.
