# Genome 009 — funcsig-subtype

## Hypothesis
Function signature subtyping in `wasm-subtyping.cc` may allow a function with a contravariant parameter mismatch to pass signature validation. Specifically, when function signatures contain GC reference type parameters with different nullability or exactness, the contravariant rule (parameter types must be supertypes in the subtype signature) may not account for all attribute bits. After the recent removal of serialized signatures from tags (`f5820a6b4b7`), the canonical signature comparison for call_indirect and imported functions may have gaps. A function whose canonical signature has a nullable parameter could be used where a non-nullable parameter is expected, bypassing the null check that the caller assumes the callee performs.

## DNA

### Source Facts
- **Function signature subtyping** (`wasm-subtyping.cc`): Function subtyping is contravariant in parameters and covariant in returns: `sig_sub <: sig_super` iff `params_super[i] <: params_sub[i]` and `returns_sub[i] <: returns_super[i]`.
- **call_indirect signature validation**: call_indirect checks that the function at the table index has a signature that matches the expected signature. The check uses canonical signature equality or subtyping.
- **Canonical signature comparison**: Uses `CanonicalSig` which includes `CanonicalValueType` for each parameter and return. Each `CanonicalValueType` includes nullability, exactness, shared, ref_type_kind, and index.
- **Tag signature removal** (`f5820a6b4b7`): "Remove serialized signature from tag" — tags (for exception handling) used to have serialized signatures. This was cleaned up. If the cleanup broke signature comparison for tag-related operations, exception handler dispatch could have wrong types.
- **Cont type leak** (`847c7b2461a`): "Fix leak of indexed cont types to JS" — continuation types were leaking to JS with wrong type info.
- **Historical context**: CVE-2024-4761 involved type confusion at boundaries where signatures didn't match.

### Key Attack Vectors
1. **Contravariant parameter nullability**: Function A has sig `(ref null $T) -> i32`. Function B has sig `(ref $T) -> i32`. B's parameter is MORE restrictive (non-nullable). Is B <: A? Contravariance says: A's param `ref null $T` must be subtype of B's param `ref $T`. But `ref null $T` is NOT a subtype of `ref $T` (nullable is not subtype of non-nullable). So B is NOT <: A. But if the subtyping check ignores nullability, B could be used where A is expected. Then calling B with a null argument (valid for A's signature) would bypass B's non-null assumption.
2. **Contravariant return type confusion**: Function A returns `ref $Parent`. Function B returns `ref $Child`. B <: A (covariant returns). But if the caller of A uses the return value as `ref $Parent` and the function is actually B, the returned `ref $Child` is correctly a subtype — this is safe. So returns are unlikely to be exploitable.
3. **Exactness in signatures**: If a parameter type is `exact ref $T` vs `ref $T`, does subtyping handle this correctly? `exact ref $T` is NOT a subtype of `ref $T` in the exact type system.
4. **call_indirect with type equality vs subtyping**: call_indirect traditionally uses signature EQUALITY (not subtyping) for dispatch. If the engine uses subtyping instead of equality (or if canonical equality doesn't account for all bits), wrong functions could be dispatched.
5. **Imported function signature mismatch**: When a function is imported, the signature is validated against the expected signature. If the validation uses `IsSubtypeOf` but the actual usage assumes equality, a subtype signature could pass validation but cause type confusion.

### Guards Expected
- Canonical signature equality uses full `CanonicalValueType` comparison (all bits)
- call_indirect uses signature equality, not subtyping
- `IsSubtypeOfImpl` in wasm-subtyping.cc handles contravariance correctly

## Techniques That Work
1. Read `IsSubtypeOfImpl` for function types in wasm-subtyping.cc
2. Read how call_indirect validates signatures (search for `CheckSignature` or `sig_check`)
3. Read how canonical signatures are compared (equality vs subtyping)
4. Test: create two functions with signatures differing only in parameter nullability. Put function B (non-null param) in a table typed for function A (nullable param). Call via call_indirect with null argument.
5. Test with imported functions: import expects sig A, provide sig B where B <: A but with different nullability.

## DO NOT Try These
- Testing with identical signatures (no mismatch to exploit)
- Testing with only numeric types (they don't have nullability/exactness)
- Testing without GC types (reference type attributes are the target)

## Evolution Plan
- v1: Read function signature subtyping implementation in wasm-subtyping.cc. Read call_indirect signature validation in the decoder and compiler. Determine if equality or subtyping is used.
- v2: If v1 finds equality is used, test if canonical equality correctly compares all attribute bits for function signatures. If subtyping is used, test contravariant parameter nullability. Create test modules with mismatched signatures.
- v3: If v2 finds a gap, construct an exploit: call_indirect dispatches to a function with wrong parameter types. The callee assumes non-null but receives null, leading to null dereference without check.

## Task
Start by reading these files in order:
1. `src/wasm/wasm-subtyping.cc` — search for function type subtyping (look for "FunctionType" or "sig" in IsSubtypeOf)
2. `src/wasm/turboshaft-graph-interface.cc` — search for "call_indirect" and "sig" to find signature validation
3. `src/wasm/module-instantiate.cc` — search for signature validation during import resolution
4. `src/wasm/canonical-types.cc` — search for "EqualSig" to understand canonical signature equality
5. Run `git -C /Users/t/v8 show f5820a6b4b7` to understand the tag signature cleanup

The type invariant to challenge: "call_indirect signature validation correctly accounts for all type attribute bits (nullability, exactness, shared) in function signatures with GC reference type parameters."

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc --no-wasm-lazy-compilation`
