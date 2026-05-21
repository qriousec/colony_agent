# Genome 119 — gc-opt-exact-cast-elim

## Hypothesis
The IsCastToCustomDescriptor guard at wasm-gc-typed-optimization-reducer.h:21-26 only returns true when the target type has a descriptor AND exactness is kExactMatchOnly. For exact casts on non-descriptor types, this guard returns false. The cast elimination logic at line 274 then checks IsHeapSubtypeOf(inferred_type, cast_target) — but IsHeapSubtypeOf operates on HeapType which may not account for exactness semantics. If the inferred static type is a proper subtype of the cast target (e.g., $Child <: $Parent), and the cast is `ref.cast (ref exact $Parent)`, the optimizer could eliminate the cast because IsHeapSubtypeOf($Child, $Parent) returns true. But the cast should FAIL because $Child is not EXACTLY $Parent.

## DNA

### Source Facts (file:line)
- **IsCastToCustomDescriptor**: wasm-gc-typed-optimization-reducer.h:21-26 — returns true only when `has_descriptor() && kExactMatchOnly`
- **Cast elimination**: wasm-gc-typed-optimization-reducer.h:274-280 — `IsHeapSubtypeOf(type.heap_type(), config.to.heap_type(), module_) && !IsCastToCustomDescriptor(...)`
- **Type check elimination**: wasm-gc-typed-optimization-reducer.h:341-345 — same guard pattern for WasmTypeCheck
- **Block unreachability**: wasm-gc-typed-optimization-reducer.cc:446-456 — same guard for dead branch elimination
- **GetExactness**: wasm-compiler-definitions.cc:21-39 — returns kExactMatchOnly for `is_final || is_exact()` without descriptor
- **SubtypeCheckExactness enum**: wasm-compiler-definitions.h:37-51 — kMayBeSubtype, kExactMatchOnly, kExactMatchLastSupertype
- **AssertType shared skip**: wasm-gc-typed-optimization-reducer.h:181-183 — shared types skip assertion entirely
- **Type refinement preserved**: wasm-gc-typed-optimization-reducer.h:312-318 — exactness config preserved in refined cast

### Gap Analysis
1. **IsHeapSubtypeOf vs exactness**: IsHeapSubtypeOf checks heap type subtyping. If the inferred type is $Child and target is (exact $Parent), IsHeapSubtypeOf($Child, $Parent) is true (because $Child <: $Parent). But the exact cast should only succeed for values that are EXACTLY $Parent, not subtypes. The guard was designed for descriptor types but exact casts can exist without descriptors.
2. **Non-final exact types**: The spec allows `(ref exact $T)` for any indexed type. For final types, exact is redundant (no subtypes exist). For NON-FINAL types, exact is meaningful — it restricts to the exact type, excluding subtypes.
3. **Config.exactness vs HeapType**: The exactness is in the WasmTypeCheckConfig, not in the HeapType. IsHeapSubtypeOf doesn't see it. The IsCastToCustomDescriptor guard was supposed to catch this, but it only checks descriptor types.
4. **Can you create (ref exact $T) without $T having a descriptor?**: If yes, and if $T is non-final, the gap is exploitable.

### Key Vulnerability Chain
1. Define non-final struct $Parent with field i32
2. Define struct $Child extends $Parent with additional field i32
3. Create function that receives (ref $Parent) parameter — optimizer infers type as (ref $Parent)
4. Downcast parameter with `ref.cast (ref exact $Parent)` — optimizer sees IsHeapSubtypeOf($Parent, $Parent) = true, IsCastToCustomDescriptor = false → eliminates cast
5. Pass a $Child value — cast should FAIL (value is $Child, not exact $Parent), but optimizer eliminated it
6. Now have type confusion: $Child value treated as exact $Parent

## Techniques That Work
1. Create two types: non-final $Parent and $Child extends $Parent
2. Use `ref.cast (ref exact $Parent)` in optimized code after type narrowing
3. Trigger Turboshaft optimization to activate the type check elimination reducer
4. Pass $Child value through the "eliminated" cast
5. Use `--experimental-wasm-custom-descriptors` to enable exact type syntax
6. Compare behavior: Liftoff (no optimization) vs Turboshaft (with optimization)
7. Access fields after the cast — if $Child has extra fields, accessing $Parent-only layout on $Child value could cause type confusion

## DO NOT Try These
- Exact casts on types WITH descriptors — IsCastToCustomDescriptor correctly blocks these
- Exact casts on FINAL types — exact is redundant, no subtypes exist
- Simple subtype casts without exact — these are correct (subtype satisfies non-exact cast)
- Testing with --wasm-assert-types on shared types (assertions disabled for shared)

## Evolution Plan
- v1: Verify whether `(ref exact $T)` syntax works without $T having a descriptor. Build minimal module with non-final struct and exact cast. Check if the module validates. If not, check what spec rules apply to exact types without descriptors.
- v2: If v1 validates: build test with $Parent/$Child hierarchy, exact cast in hot loop, tier-up to Turboshaft. Check if cast is eliminated by reading Turboshaft IR (--trace-turbo). Pass $Child through eliminated cast, access fields.
- v3: If v2 shows elimination: try to exploit. Access $Parent-layout fields on $Child value. Check for OOB read (extra field offset) or type confusion (field type mismatch). Verify with ASAN/MSAN.

## Task
1. Check if `(ref exact $T)` is valid without custom descriptors. Read wasm-module-builder.js or test files for exact type usage without descriptors.
2. Build module: struct $Parent (non-final, field i32), struct $Child extends $Parent (field i64). Function takes (ref $Parent), does ref.cast (ref exact $Parent), returns field.
3. Run with Liftoff only — verify $Child is rejected by exact cast.
4. Run with Turboshaft — check if exact cast is eliminated for values known to be (ref $Parent).
5. If eliminated: pass $Child, observe behavior. Does it trap or succeed incorrectly?
6. Run with: `--experimental-wasm-custom-descriptors --no-wasm-lazy-compilation --wasm-tiering-budget=1`
7. Start with Evolution Plan v1 (validate exact type syntax without descriptors).
