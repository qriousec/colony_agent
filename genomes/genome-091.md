# Genome 091 — shared-global-storage

## Hypothesis
Wasm globals storage was recently refactored (0cfa6078eae "[wasm] Refactor storage of globals") and the interpreter's globals handling was separately fixed (07e892290b6 "[Wasm interpreter] Fix handling of globals storage"). For shared type globals (globals with shared ref types), the storage must be in shared space so multiple isolates can access it. If the refactored storage code allocates shared globals in local space, uses wrong write barriers for shared ref global writes, or computes incorrect offsets in the shared vs local globals buffer, it creates cross-space reference bugs. A tier differential between Liftoff, Turboshaft, and the interpreter for shared global access may reveal inconsistencies introduced by the refactor.

## DNA
### Recent Globals Commits
- **0cfa6078eae**: "[wasm] Refactor storage of globals" — major refactor of how globals are stored
- **07e892290b6**: "[Wasm interpreter] Fix handling of globals storage" — interpreter-specific fix AFTER the refactor
- **8bb42f01a0b**: "[wasm interpreter] Use GetGlobalStorage in wasm-interpreter-runtime-inl.h" — another interpreter globals fix

The fact that the interpreter needed FIXES after the globals refactor suggests the refactor was incomplete or introduced edge cases.

### Globals Storage Architecture
- Wasm globals are stored in a buffer attached to the instance data
- For shared modules, globals may need to be in shared_trusted_instance_data
- Global.get and global.set access the buffer at computed offsets
- Ref-typed globals need write barriers (GC-managed references)

### Shared Global Access Paths
- **Liftoff** (baseline): Direct offset computation + load/store
- **Turboshaft** (optimized): May optimize global access patterns
- **Interpreter** (drumbrake): Uses GetGlobalStorage function (recently fixed)
- Each tier may compute offsets or storage locations differently

### Key Questions
1. Are shared type globals stored in shared_trusted_instance_data or regular instance data?
2. Does the offset computation account for the shared/non-shared split?
3. When writing a ref to a shared global, is the correct (shared) write barrier used?
4. Do all tiers agree on where shared globals are stored?

### Related Patterns
- **P7**: Write barrier computation for shared objects
- **P6**: GC DCHECK(!InWritableSharedSpace) paths
- Globals interact with both patterns: writes to shared globals need shared write barriers, and GC must correctly track shared global references.

## Techniques That Work
1. Create shared Wasm module with globals of various types:
   - `(global (shared) (mut i32) (i32.const 0))` — shared mutable i32
   - `(global (shared) (mut (ref null (shared any))) (ref.null (shared any)))` — shared mutable ref
2. Export global access functions (get/set)
3. Test with all tiers: `--liftoff-only`, `--no-liftoff`, `--wasm-jitless`
4. Compare values across tiers — if a global.set in Liftoff stores to the wrong location, global.get in Turboshaft reads the wrong value
5. Test ref-typed shared globals with GC: write refs, trigger GC, read back
6. Test with concurrent access (if possible with d8 workers)

## DO NOT Try These
- Non-shared globals — well-tested, standard path
- Shared struct field access — covered by shared-write-barrier-gc worker
- Shared string at boundary — TC-003/004/005 confirmed

## Evolution Plan
- v1: Source trace shared globals storage. Read module-instantiate.cc for how shared globals are allocated. Read turboshaft-graph-interface.cc for how global.get/global.set access shared globals. Read liftoff-compiler.cc for how Liftoff accesses shared globals. Compare offset computation and storage location.
- v2: Build test module with shared ref globals. Write shared struct refs into shared globals. Read back in different tier. Compare values. Trigger GC between write and read.
- v3: Test concurrent access to shared globals. Use d8's worker support (if available) to access shared globals from multiple workers simultaneously. Check for data races, incorrect values, or GC issues.

## Task
Start by reading:
1. `src/wasm/module-instantiate.cc` — search for "global" + "shared". How are shared globals allocated? Are they in shared_trusted_instance_data?
2. `src/wasm/turboshaft-graph-interface.cc` — search for "GlobalGet" or "GlobalSet". How does the graph builder handle shared globals?
3. `src/wasm/baseline/liftoff-compiler.cc` — search for "GlobalGet" or "GlobalSet". How does Liftoff access globals?
4. The diff of 0cfa6078eae — what exactly changed in the globals refactor?
5. The diff of 07e892290b6 — what was the interpreter fix? What was broken?

The type invariant to challenge: "Shared globals are stored in shared space and accessed consistently across all compilation tiers after the globals storage refactor." The interpreter fix suggests this wasn't true for at least one tier.

Use `--experimental-wasm-shared` with `/Users/t/v8/out/fuzzbuild/d8`. For tier comparison: `--liftoff-only` vs `--no-liftoff` vs `--wasm-jitless` (interpreter).
