# Genome 132 — loop-phi-type-widening

## Hypothesis
At loop headers, Turboshaft's ProcessPhi (wasm-gc-typed-optimization-reducer.cc:388-427) computes type unions from forward and back edges. On FIRST evaluation (line 393-399), only the forward edge type is used — potentially an overly specific type like `(ref exact $Child)`. On revisit, the union of all phi inputs is computed, which may WIDEN to `(ref $Parent)` if the back-edge carries a less specific type.

**The vulnerability**: If optimizations (cast elimination, field access lowering) were applied during the first pass based on the overly specific forward-edge type, they may NOT be re-evaluated after the revisit widens the type. Specifically:
1. First pass: phi type = `(ref exact $Child)` (from forward edge)
2. Optimizer eliminates a `ref.cast (ref $Child)` in the loop body (TypeCheckAlwaysSucceeds because exact $Child is subtype of $Child)
3. Revisit: phi type widened to `(ref $Parent)` (union with back-edge)
4. The eliminated cast is NOT restored — now parent instances can flow through the loop body unchecked
5. Code that accesses $Child-specific fields (extra fields beyond $Parent) reads OOB

## DNA
- **wasm-gc-typed-optimization-reducer.cc:388-427**: ProcessPhi() — first pass uses only forward edge type (lines 393-399). Revisit computes union (lines 400-427).
- **wasm-gc-typed-optimization-reducer.cc:393-399**: `if (first_loop_header_evaluation)` block — sets phi type to forward-edge-only type, ignores back-edge. This is standard fixpoint iteration.
- **CONFIRMED**: Fixpoint iteration DOES re-process the loop body after revisit. When `needs_revisit == true` (line 44), the loop body is re-evaluated with widened types. So first-pass optimizations ARE re-evaluated. This weakens the naive hypothesis.
- **CRITICAL TODO at lines 408-410**: `"// TODO(mliedtke): Ideally, we'd skip unreachable predecessors here completely, as we might loosen the known type due to an unreachable predecessor."` — unreachable predecessors can contribute to union, WIDENING the phi type unnecessarily.
- **Unreachable predecessor issue**: When one predecessor is unreachable but carries a type (e.g., StructB from a dead path), union(StructA, StructB) = common ancestor (possibly anyref). Even though the path is dead, it widens the phi type. The code at line 411 skips uninhabited types but NOT all unreachable types.
- **wasm-gc-typed-optimization-reducer.cc:564-571**: Reachability check `if (!reachable[i]) continue;` — but this is in a DIFFERENT merge function. ProcessPhi at line 411 uses `input_type.is_uninhabited()` which is NOT the same as unreachable.
- **wasm-gc-typed-optimization-reducer.h:270-280**: ReduceWasmTypeCast — this runs DURING the type analysis pass. If the loop body is re-processed, casts are re-evaluated with the widened phi type. But: are casts that were ELIMINATED on the first pass restored on the second pass? The type ANALYSIS is re-run, but the GRAPH MODIFICATIONS may already be committed.
- **CVE-2024-2887**: Related — type confusion via TypeCheckAlwaysSucceeds.
- **Union computation**: wasm-subtyping.cc Union() (lines 594-629) — computes least upper bound.

## Techniques That Work
1. Create a type hierarchy: $Parent {i32}, $Child extends $Parent {i32, i64}
2. Build a loop where the forward edge brings a $Child instance and the back-edge brings a $Parent instance (or the back-edge type widens through a br_on_cast_fail)
3. Inside the loop body, perform ref.cast to $Child and access the extra i64 field
4. Force Turboshaft tier-up (call 10000+ times with $Child instances)
5. Then call once with a $Parent instance entering the loop
6. If the cast was eliminated based on first-pass type, the $Parent passes through and the i64 field access is OOB
7. Use `struct.get` on the i64 field to read 8 bytes past the $Parent's layout

## DO NOT Try These
- Shared types (--experimental-wasm-shared, BANNED)
- Simple linear casts without loops — need loop merge points for phi widening
- Module-level type errors — the validator catches obvious mismatches
- Liftoff-only testing — need Turboshaft optimization passes

## Evolution Plan
- v1: **ProcessPhi fixpoint analysis**. Read wasm-gc-typed-optimization-reducer.cc:388-427 and the surrounding loop analysis code. Map: (a) how does the first-pass forward-edge-only type get used? (b) are optimizations made during the first pass reverted or re-checked on revisit? (c) is there a convergence check that forces re-processing of the loop body when the phi type widens? Build test: loop with forward edge $Child, back edge $Parent, ref.cast $Child in body, struct.get of extra field.
- v2: **Exactness loss at phi merge**. Even if v1 is clean for simple widening, test exactness specifically. Forward edge: `(ref exact $T)`. Back edge: `(ref $T)`. Union should be `(ref $T)` (loses exactness). But if an optimization inside the loop depends on exactness (e.g., devirtualization, field layout assumption), the loss of exactness after revisit could leave the optimization in place. Test: exact-type-based field access after phi merge.
- v3: **Nullability gain at phi merge**. Forward edge: `(ref $T)` (non-null). Back edge: `(ref null $T)` (nullable). Union: `(ref null $T)`. If a null check was eliminated based on the first-pass non-null type, null can flow through after revisit. Test: struct.get on a potentially-null value after phi merge — should trap on null but might not if null check was eliminated.

## Task
Start by reading:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc` — ProcessPhi (lines 388-427). Also read the loop analysis setup: how is `first_loop_header_evaluation` determined? What happens after revisit — is the loop body re-processed?
2. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — ReduceWasmTypeCast (lines 270-280). How does it use the phi type? Is the type snapshot a snapshot-in-time or does it track updates?
3. `src/compiler/turboshaft/loop-finder.h` or similar — how does Turboshaft handle loop fixpoint iteration?
4. `src/wasm/wasm-subtyping.cc` — Union() function for reference types

Then build test 1:
- Type $Parent: struct {field0: i32}
- Type $Child: struct extends $Parent {field0: i32, field1: i64}
- Function: loop parameter is ref $Parent. First iteration: create $Child, enter loop. Back-edge: receive ref from function parameter (could be $Parent). Body: ref.cast to $Child, struct.get field1.
- Call 10000 times with $Child (Turboshaft tier-up), then call with $Parent.
- Expected: trap on ref.cast. If no trap → cast was eliminated → type confusion.

Run with: `/Users/t/v8/out/fuzzbuild/d8 test.js`
No --experimental-wasm-shared flag.
