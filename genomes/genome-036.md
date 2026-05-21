# Genome 036 — interpreter-gc-type-cast

## Hypothesis
The Wasm interpreter (`--wasm-interpret-all`) has its own `DoRefCast` implementation at wasm-interpreter.cc:5826 that is entirely separate from Liftoff and Turboshaft compiled code. It uses a hardcoded switch on `GenericKind` that does NOT handle `kExtern`, `kExn`, or `kAny` (these hit UNREACHABLE at line 5855). It has ZERO support for custom descriptor operations (ref.get_desc, ref.cast_desc_eq, struct.new_desc — none found in the interpreter codebase). WASM-TC-003 was found via tier differential between Liftoff and Turboshaft. Adding the interpreter as a third comparison tier multiplies the differential surface:

1. **GenericKind gaps**: `DoRefCast` switch at line 5835 handles kEq, kI31, kStruct, kArray, kString, kNone, kNoExtern, kNoFunc — but NOT kExtern, kExn, or kAny. A ref.cast to `(ref extern)` or `(ref exn)` in the interpreter hits UNREACHABLE.
2. **SubtypeCheck divergence**: For indexed types, the interpreter calls `SubtypeCheck(ref, ref_type, rtt, target_type.ref_index(), null_succeeds)` which uses RTT-based map checking. Compiled code may use different check strategies (inline caching, range checks). Edge cases in subtype checking could produce different results.
3. **BranchOnCast value preservation**: `s2s_BranchOnCast` at line 5877 reads `ValueType::FromRawBitField(ref_bitfield)` from the bytecode stream. If the bytecode compiler encodes type bits differently from what the interpreter expects, the cast result may be wrong.
4. **No custom descriptor operations**: If a module uses ref.get_desc or ref.cast_desc_eq, what does the interpreter do? UNREACHABLE? Wrong result? Fallthrough?

Multi-factor: Wasm interpreter execution + GC type cast + generic kind gap or indexed subtype divergence + tier differential vs Liftoff/Turboshaft.

## DNA

### Source Facts

**Interpreter DoRefCast** (wasm-interpreter.cc:5826-5857):
```cpp
static bool DoRefCast(WasmRef ref, ValueType ref_type, HeapType target_type,
                      bool null_succeeds, WasmInterpreterRuntime* wasm_runtime) {
  if (target_type.has_index()) {
    DirectHandle<Map> rtt = wasm_runtime->RttCanon(target_type.ref_index().index);
    return wasm_runtime->SubtypeCheck(ref, ref_type, rtt, target_type.ref_index(), null_succeeds);
  } else {
    switch (target_type.generic_kind()) {
      case GenericKind::kEq: ...
      case GenericKind::kI31: ...
      case GenericKind::kStruct: ...
      case GenericKind::kArray: ...
      case GenericKind::kString: ...
      case GenericKind::kNone: case GenericKind::kNoExtern: case GenericKind::kNoFunc:
        DCHECK(null_succeeds);
        return wasm_runtime->IsNullTypecheck(ref, ref_type);
      case GenericKind::kAny:
        // "Any may never need a cast" — UNREACHABLE on default
      default:
        UNREACHABLE();
    }
  }
}
```

**Missing GenericKinds**: kExtern, kExn, kAny, kNoExn — all hit `default: UNREACHABLE()`

**BranchOnCast** (wasm-interpreter.cc:5877-5895):
- Reads target_type from bytecode as `HeapType::FromBits(static_cast<uint32_t>(Read<int32_t>(code)))`
- Reads ref_type from bytecode as `ValueType::FromRawBitField(ref_bitfield)`
- Push ref back to stack, then conditionally branch

### Key Source Files
```
src/wasm/interpreter/wasm-interpreter.cc — DoRefCast, s2s_BranchOnCast, s2s_RefCast, s2s_StructGet
src/wasm/interpreter/wasm-interpreter-runtime.cc — SubtypeCheck, RefIsEq, RefIsStruct, etc.
src/wasm/interpreter/wasm-interpreter-runtime.h — WasmInterpreterRuntime class
src/flags/flag-definitions.h:2234 — --wasm-interpret-all flag
```

### Grep Commands
```bash
# Interpreter DoRefCast
grep -rn "DoRefCast\|SubtypeCheck\|RefIs" src/wasm/interpreter/wasm-interpreter.cc | head -20

# Interpreter BranchOnCast
grep -rn "BranchOnCast\|BrOnCast" src/wasm/interpreter/wasm-interpreter.cc | head -10

# GenericKind enum
grep -rn "enum.*GenericKind\|kExtern\|kExn\|kAny" src/wasm/value-type.h | head -20

# What does interpreter do with custom descriptors?
grep -rn "get_desc\|cast_desc\|struct_new_desc\|kRefGetDesc" src/wasm/interpreter/ | head -10

# Interpreter bytecode compilation for GC ops
grep -rn "kExprRefCast\|kExprBrOnCast\|kExprRefTest" src/wasm/interpreter/wasm-interpreter.cc | head -15
```

### Tier Differential Precedent
- WASM-TC-003: Found by comparing Liftoff (static externref check) vs Turboshaft (C++ WasmToJSObject with string unshare) — 4 bypass vectors
- Pattern P10: "Separate implementations disagree on type semantics"

## Techniques That Work
1. Build module with GC type casts: ref.cast to struct, array, eq, i31, extern, exn
2. Run with `--wasm-interpret-all` vs `--liftoff --no-wasm-tier-up` vs `--no-liftoff --turboshaft-wasm`
3. Compare: does the cast succeed/fail the same way in all three tiers?
4. Focus on: (a) casts to extern/exn types (interpreter UNREACHABLE), (b) null + non-null variants, (c) indexed types with deep hierarchies
5. Test ref.test (returns bool) as well as ref.cast (traps on failure) for same conditions
6. Use `--experimental-wasm-custom-descriptors` — if descriptor ops hit interpreter, expect UNREACHABLE

## DO NOT Try These
- Simple struct.get/set without casts — no tier differential
- ref.cast to basic types (i31, struct, array) — these are handled in all tiers
- WeakMap/WeakSet shared crashes — different bug class (TC-001)
- String unsharing — already found (TC-003)

## Evolution Plan
- v1: **GenericKind gap exploitation**. Build a module with `ref.cast (ref extern)` and `ref.cast (ref exn)`. Run with `--wasm-interpret-all` in debug build. Expect UNREACHABLE crash. If confirmed, test release build behavior (wrong result? corruption?).
- v2: **Indexed type SubtypeCheck differential**. Build deep type hierarchy (A → B → C → D). Create value of type D, cast to A in each tier. Verify all three tiers agree. Then test: cast to B when value is actually C — does SubtypeCheck in interpreter match compiled code?
- v3: **Custom descriptor ops in interpreter**. Enable `--experimental-wasm-custom-descriptors`. Build module with ref.get_desc, ref.cast_desc_eq. Run in interpreter. Expected: UNREACHABLE (missing implementation). If it silently produces wrong result instead, that's a type confusion.

## Task
Start with v1. Read these files IN THIS ORDER:

1. `src/wasm/interpreter/wasm-interpreter.cc` around line 5826-5860 — Understand DoRefCast exhaustively. List every GenericKind handled and NOT handled.
2. `src/wasm/value-type.h` — Find the GenericKind enum. List all values. Cross-reference with DoRefCast switch.
3. `src/wasm/interpreter/wasm-interpreter-runtime.cc` — Find SubtypeCheck implementation. How does it differ from compiled code's type checking?
4. Build test: `ref.cast (ref extern)` on an externref value. Run with `--wasm-interpret-all` vs `--liftoff`. If interpreter crashes (UNREACHABLE) but Liftoff succeeds, that's a confirmed tier differential bug.

Your type invariant to challenge: "The Wasm interpreter produces the same results as compiled code for all type operations." The interpreter is a complete reimplementation — find where it disagrees.

Post with tags `interpreter,tier-diff,type-cast,gc,generic-kind`.
