# Genome 070 — write-barrier-skip-shared

## Hypothesis
Commit 53cc35a6e9a "[wasm] Improve write-barrier treatment" adds `WriteBarrierKind` to StructSet and ArraySet operations, allowing write barriers to be SKIPPED when:
1. The host object is freshly allocated in young space
2. The written value is in read-only space

Multi-factor: [write barrier skip for young allocation] × [shared struct allocated in shared space, not young space] × [value stored is a reference to young-gen object] = GC fails to track the young-gen reference, collects it, UAF.

If the write barrier skip logic checks "is this object freshly allocated?" but doesn't account for shared structs being allocated directly in shared space (bypassing young generation), it could incorrectly skip barriers. Conversely, if a shared struct IS initially allocated in young space before being promoted to shared space, the write barrier skip could be applied during the brief window before promotion.

## DNA

### Source Facts
```
# Commit 53cc35a6e9a modifies 19 files, 232 insertions, 137 deletions
# Key changes:

# Turboshaft assembler: StructSet/ArraySet now take WriteBarrierKind
src/compiler/turboshaft/assembler.h — StructSet signature changed
src/compiler/turboshaft/operations.h — StructSetOp/ArraySetOp now store WriteBarrierKind

# Turboshaft graph interface: where WriteBarrierKind is decided
src/wasm/turboshaft-graph-interface.cc — 94 lines changed (determines when to skip)

# Liftoff compiler: same optimization in baseline compiler
src/wasm/baseline/liftoff-compiler.cc — 64 lines changed

# Lowering: how WriteBarrierKind affects code generation
src/compiler/turboshaft/wasm-lowering-reducer.h — 18 lines changed

# GC type optimization: uses WriteBarrierKind
src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h — 3 lines

# Constant expression init: write barriers during module init
src/wasm/constant-expression-interface.cc — 61 lines changed

# Factory: allocation changes
src/heap/factory.cc — 9 lines changed
```

### Key Questions
1. How does `turboshaft-graph-interface.cc` determine the WriteBarrierKind for a struct.set?
2. Is there a check for `is_shared()` when deciding to skip the write barrier?
3. What happens when a shared struct is initialized via constant expressions? The constant-expression-interface.cc changes suggest initialization-time write barriers changed too.
4. Does Liftoff use the same write barrier skip logic as Turboshaft?
5. In `wasm-lowering-reducer.h`, how is WriteBarrierKind translated to actual machine code?

### Write Barrier Basics
V8's generational GC needs write barriers to track cross-generation references:
- Young → Old: OK (young is scanned completely)
- Old → Young: NEEDS write barrier (old objects aren't scanned every minor GC)
- Shared → Young: NEEDS write barrier (shared space isn't scanned every minor GC)
- Shared → Shared: depends on GC implementation

If a write barrier is skipped for Shared → Young, the minor GC won't know about the reference and may collect the young object.

### Shared Struct Allocation
Shared structs are allocated in shared space. They should NOT be treated as "freshly allocated in young space." But the question is: does the write barrier skip logic correctly distinguish shared-space allocations from young-space allocations?

## Techniques That Work
- Allocate a shared struct, store a reference to a young-gen object in a mutable field
- Trigger minor GC
- Check if the young-gen object is still alive (access it through the shared struct field)
- Compare behavior with and without write barrier optimization

## DO NOT Try These
- Immutable field testing (write barriers are for mutable fields/initialization only)
- Non-shared struct write barriers (well-tested, young-space optimization is established)
- Read-only space values (correctly skipped, no GC tracking needed)

## Evolution Plan
- v1: Source audit turboshaft-graph-interface.cc for WriteBarrierKind determination. Check if is_shared() or shared-space allocation is accounted for. Audit Liftoff compiler path too.
- v2: If v1 finds no shared-space check, construct test with shared struct holding reference to young-gen object + minor GC stress. Compare behavior with --single-generation (disables young space) vs default.

## Task
Start with source audit of the WriteBarrierKind determination:

```bash
# Find where WriteBarrierKind is decided for struct.set
grep -n "WriteBarrierKind\|write_barrier\|kNoWriteBarrier\|kFullWriteBarrier" /Users/t/v8/src/wasm/turboshaft-graph-interface.cc | head -30

# Check if shared structs are handled
grep -n "is_shared\|shared.*write.*barrier\|shared.*barrier" /Users/t/v8/src/wasm/turboshaft-graph-interface.cc

# Check Liftoff path
grep -n "WriteBarrier\|write_barrier\|kNoWriteBarrier" /Users/t/v8/src/wasm/baseline/liftoff-compiler.cc | head -30

# Check constant expression initialization
grep -n "WriteBarrier\|write_barrier" /Users/t/v8/src/wasm/constant-expression-interface.cc
```

The critical check: is there EVER a path where a StructSet to a shared struct gets `kNoWriteBarrier`?

Post with tags `write-barrier,shared-struct,GC,young-gen,W33,coverage`.
