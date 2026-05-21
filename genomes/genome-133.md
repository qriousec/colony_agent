# Genome 133 — funcsig-subtype-table

## Hypothesis
**UPDATED after source research**: Canonicalization DOES preserve exactness and nullability (`is_equal_except_index()` compares all bits including IsExactField at bit 3 and IsNullableField at bit 2). For **final** function types, call_indirect uses a strict canonical ID equality check — two signatures differing only in exactness get different IDs.

**REDIRECTED HYPOTHESIS**: For **non-final** function types, call_indirect uses an RTT-based SUBTYPE check (liftoff-compiler.cc:10120-10186) rather than equality. This subtype check traverses the canonical RTT hierarchy. The question is: does the RTT subtype check correctly enforce contravariance for reference parameters? Specifically:
- Function subtyping is contravariant in parameters: a subtype function must accept SUPERTYPES of the supertype function's parameters.
- `(ref exact $T) -> i32` is NOT a subtype of `(ref $T) -> i32` because exact $T is MORE restrictive than $T (narrows, not widens).
- But `(ref $T) -> i32` IS a subtype of `(ref exact $T) -> i32` because $T accepts more than exact $T (parameter widens = contravariant correct).
- If the RTT subtype check implements this incorrectly (e.g., checks covariance instead of contravariance for params), a function expecting exact types could be called through a non-final supertype signature.

## DNA
- **canonical-types.h:383-392**: `EqualValueType` in `CanonicalEquality` struct — uses `is_equal_except_index()` which compares all bits except index. Exactness IS preserved in canonicalization.
- **value-type.h:233-247**: Bit field layout: TypeKindField(0-1), IsNullableField(2), IsExactField(3), IsSharedField(4), RefTypeKindField(5-7), PayloadField(index). `is_equal_except_index()` at line 1001-1003 compares all non-index bits.
- **liftoff-compiler.cc:10087-10195**: call_indirect validation. For FINAL types (line 10192-10194): strict canonical ID equality. For NON-FINAL types (lines 10120-10186): RTT-based subtype check via WasmCanonicalRtts.
- **wasm-subtyping.cc:85-112**: `ValidFunctionSubtypeDefinition` — enforces contravariance for parameters (line 96-101): `IsSubtypeOf(super_func->parameters()[i], sub_func->parameters()[i])`. Note the argument order: super param must be subtype of sub param.
- **canonical-types.cc**: `IsCanonicalSubtype` for function types — does this correctly implement contravariance at the canonical level?
- **EquivalentTypes (wasm-subtyping.cc:481-492)**: Explicitly checks `type1.is_exact() != type2.is_exact()` at line 486. Two types differing in exactness are NOT equivalent.
- **value-type.h**: Does CanonicalValueType encode exactness? If exactness is stripped during canonicalization, all `(ref $T)` and `(ref exact $T)` look the same.

## Techniques That Work
1. Define struct $Parent {i32} and $Child extends $Parent {i32, i64}
2. Define func type A: (ref exact $Parent) -> i32  — function reads field0
3. Define func type B: (ref $Parent) -> i32 — function reads field0
4. Store function of type A in table
5. call_indirect with type B from another function — does the signature check pass?
6. If yes, pass a $Child instance through the type B call — function A receives it, but A expects exact $Parent
7. Since A expects exact $Parent, it may use the exact layout (1 field) — accessing field1 would be OOB
8. Alternatively: test the reverse — exact in the call_indirect type vs non-exact in the table entry

## DO NOT Try These
- Shared types (--experimental-wasm-shared, BANNED)
- Simple i32/f64 signatures — exactness only applies to reference types
- call_ref (uses different dispatch, function reference types) — focus on call_indirect first
- Testing at Liftoff only — need Turboshaft for optimized dispatch

## Evolution Plan
- v1: **Confirm final type equality check works**. Build test: two FINAL function types differing only in parameter exactness. call_indirect should TRAP because they have different canonical IDs. Verify this is the case. If it traps — this path is defended. Then test with nullability difference — should also trap.
- v2: **Non-final function type subtype check**. Build test: create NON-FINAL function types with reference parameters. Define a supertype sig `(ref $Parent) -> i32` and a subtype sig `(ref exact $Parent) -> i32`. Store a function of the subtype sig in a table typed with the supertype sig. call_indirect with the supertype sig — does the RTT subtype check pass? Key question: does `IsCanonicalSubtype` for function types correctly implement contravariance? Read `canonical-types.cc` for the function-type-specific subtype check.
- v3: **Return type covariance attack**. Test return type differences with non-final function types. If function returns `(ref $Parent)` but caller expects `(ref $Child)` return via the dispatch table type, does the subtype check catch this? Return type should be covariant (subtype function returns subtype). If the check is missing for returns, the caller casts the return value as $Child when it's $Parent — field OOB.

## Task
Start by reading:
1. `src/wasm/canonical-types.cc` — full file. Focus on:
   - How function signatures are canonicalized (CanonicalizeTypeDef for function types)
   - Hash computation for function types — does it include exactness?
   - Equality comparison for function types — does it check exactness of parameters?
2. `src/wasm/value-type.h` — CanonicalValueType class. Does it store exactness?
3. `src/wasm/canonical-types.h` — CanonicalSig type and how it's used
4. `grep -rn 'CanonicalSigEqual\|sig.*==.*sig\|dispatch.*sig' /Users/t/v8/src/wasm/` — find signature comparison sites

Then build test 1:
- Struct $S {field0: i32}
- Func type A: (ref exact $S) -> i32
- Func type B: (ref $S) -> i32
- Function implA of type A: reads field0, returns it
- Table with implA at index 0
- call_indirect with type B, index 0 — does it succeed or trap?
- If it succeeds: the canonical IDs match, exactness is lost in canonicalization

Run with: `/Users/t/v8/out/fuzzbuild/d8 test.js`
No --experimental-wasm-shared flag.
