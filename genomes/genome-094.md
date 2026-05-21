# Genome 094 — br-on-cast-chain-narrow

## Hypothesis
br_on_cast branches based on type check results, narrowing the type in the success branch. The optimizer's REDUCE_INPUT_GRAPH(WasmTypeCast) at line 251 uses the TypeSnapshotTable to track narrowed types. In chains of br_on_cast (nested type checks with multiple levels of type hierarchy), the narrowing accumulates. If the intermediate narrowing at step N incorrectly eliminates a cast at step N+1 (because the optimizer treats the narrowed type as more specific than it actually is — e.g., confusing exact vs subtype, or losing nullable info through the chain), the final value could have a wrong type.

## DNA
### br_on_cast Type Narrowing
- **br_on_cast** checks if a value is a given type and branches accordingly
- In the success branch, the value's type is NARROWED to the cast target type
- In the fallthrough branch, the type remains the original (or is narrowed to the complement)
- The TypeSnapshotTable records these narrowings per-branch

### Chained Narrowing Pattern
```wasm
;; value: ref $A (wide type)
br_on_cast $l1 (ref $A) (ref $B)  ;; narrowed to ref $B in $l1
;; fallthrough: still ref $A
br_on_cast $l2 (ref $A) (ref $C)  ;; narrowed to ref $C in $l2
;; fallthrough: ref $A but NOT $B and NOT $C
br_on_cast $l3 (ref $A) (ref $D)  ;; narrowed to ref $D in $l3
```
Each br_on_cast narrows the fallthrough type. After checking $B and $C, the fallthrough type should be "$A but not $B and not $C". If the optimizer loses this negative type information, it may incorrectly eliminate a later cast.

### Key Code Locations
- **wasm-gc-typed-optimization-reducer.h:251**: REDUCE_INPUT_GRAPH(WasmTypeCast) — cast elimination
- **wasm-gc-typed-optimization-reducer.h:323**: REDUCE_INPUT_GRAPH(WasmTypeCheck) — check elimination
- **wasm-gc-typed-optimization-reducer.h:128**: TypeSnapshotTable — stores narrowed types
- **wasm-gc-typed-optimization-reducer.h:181-184**: AssertType skips shared types
- **turboshaft-graph-interface.cc**: BrOnCast emission — how narrowing is recorded

### Failure Modes
1. **Accumulated narrowing overflow**: After N br_on_cast checks, the accumulated narrowing information is wrong
2. **Exact vs subtype confusion**: br_on_cast $B narrows to exact $B, but optimizer treats it as $B-or-subtype
3. **Nullable loss**: br_on_cast (ref null $A) (ref $B) — nullability info lost through chain
4. **Fallthrough type over-narrowing**: Fallthrough branch type is narrowed too much, leading optimizer to eliminate a valid cast
5. **Custom descriptor interaction**: If any type in the hierarchy has a custom descriptor, narrowing may not account for descriptor-based subtyping correctly

### Attack Vectors
1. **Diamond hierarchy**: $A <: $B, $A <: $C (two subtypes of same base). br_on_cast to $B, fallthrough br_on_cast to $C. Optimizer may think fallthrough is definitely $C (wrong — could still be $A).
2. **Linear chain with nullable**: Chain of br_on_cast through $A → $B → $C → $D. Introduce null at some step. Does nullability survive correctly?
3. **br_on_cast_fail**: The "fail" variant narrows the TYPE of the value that doesn't match. Chaining br_on_cast and br_on_cast_fail creates complex narrowing patterns.

### Related Patterns
- **W3 (Type check elimination in optimizer)**: This genome targets the specific br_on_cast narrowing chain sub-surface, which is UNTESTED per target inventory
- **Genome 092 (loop-type-widening)**: Both target TypeSnapshotTable but different mechanisms (loops vs chains)

## Techniques That Work
1. Create type hierarchy with 4+ types: $Base, $A <: $Base, $B <: $Base, $C <: $A
2. Create function with chain of 3+ br_on_cast on same value
3. Pass value of each type through the chain — observe which branches are taken
4. After chain, use struct.get on the "narrowed" value — if type is wrong, wrong field accessed
5. Compare Liftoff vs Turboshaft — Liftoff does runtime checks, Turboshaft may eliminate them
6. Force Turboshaft with `--no-liftoff`
7. Use `--trace-turbo` to inspect which casts are eliminated
8. Test br_on_cast_fail alongside br_on_cast for complex narrowing

## DO NOT Try These
- Simple single br_on_cast — well-tested basic functionality
- Type check elimination for non-chained scenarios — covered by previous workers
- Shared type cast elimination — confirmed defended (dead surface)
- AnyConvertExtern shared bit — confirmed defended (dead surface)

## Evolution Plan
- v1: Source trace br_on_cast handling in the optimizer. Read wasm-gc-typed-optimization-reducer.h for how br_on_cast interacts with TypeSnapshotTable. How is the type narrowed in the success branch? How is it narrowed in the fallthrough branch? Does the fallthrough carry "negative" type information (not-$B)? Read turboshaft-graph-interface.cc for BrOnCast emission — what types are recorded?
- v2: Build test with 3-level br_on_cast chain. Create diamond hierarchy ($Base, $Left <: $Base, $Right <: $Base). Function receives ref $Base, chains br_on_cast for $Left, then $Right. In final fallthrough, try struct.get with $Base fields. In branch targets, access type-specific fields. Compare Liftoff vs Turboshaft for correct behavior.
- v3: Test br_on_cast_fail chains. Mix br_on_cast and br_on_cast_fail in the same chain. Test with nullable types. Create scenario where accumulated narrowing should prevent cast elimination but doesn't due to lost information.

## Task
Start by reading:
1. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — search for "BrOnCast", "TypeCheck", "TypeCast". How does the reducer handle br_on_cast? How is the TypeSnapshotTable updated for success vs fallthrough branches? Look at REDUCE_INPUT_GRAPH for both WasmTypeCast (line 251) and WasmTypeCheck (line 323).
2. `src/wasm/turboshaft-graph-interface.cc` — search for "BrOnCast". How is br_on_cast translated to Turboshaft operations? What narrowing information is attached?
3. `src/compiler/turboshaft/snapshot-table.h` — how does the snapshot table handle branching (if/br_on_cast creates two branches with different type info)?
4. `src/wasm/wasm-subtyping.h` — IsSubtypeOf and related functions. How does the type system handle exact vs subtype relationships in the context of br_on_cast narrowing?
5. `src/wasm/function-body-decoder-impl.h` — search for "BrOnCast". What types does the decoder push after br_on_cast?

The type invariant to challenge: "After a chain of br_on_cast operations, the TypeSnapshotTable correctly tracks the narrowed type at each point, and no subsequent cast is incorrectly eliminated based on accumulated narrowing." If narrowing is wrong, a cast is eliminated but the value has a wider type → type confusion.

Use `--no-liftoff --experimental-wasm-gc` with `/Users/t/v8/out/fuzzbuild/d8`. Add `--trace-turbo` for optimizer IR dumps.
