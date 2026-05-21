# Genome 085 — liftoff-wellknown-stack

## Hypothesis
The configureAll fix (a58753bac7b) revealed that Liftoff's well-known import fast paths can get confused about stack state when the imported function has an incorrect signature. The fix added compile-time and runtime signature validation for configureAll. Other well-known imports (string builtins, DataView operations, memory/table builtins) that perform pattern-matching rewrites may have similar stack confusion vulnerabilities — the fast path assumes a specific signature and manipulates the stack accordingly, but if the signature validation is missing or incomplete, Liftoff's stack model diverges from reality.

## DNA
### Confirmed Bug Pattern (configureAll fix — a58753bac7b)
- **liftoff-compiler.cc**: configureAll fast path did pattern-matching rewrite assuming specific function signature
- **Bug**: For incorrect signatures, Liftoff got confused about stack state
- **Fix**: Added `ExpectCallWellKnownImport` signature validation at compile time + runtime bailout
- **Impact**: Stack confusion → type confusion (stack slots treated as wrong types)

### Well-Known Import Mechanism
- **wasm-module.h**: `WellKnownImport` enum defines recognized imports with fast paths
- **liftoff-compiler.cc**: `CallWellKnownImport` dispatches to optimized implementations
- Each well-known import may manipulate the Liftoff stack differently from a normal call
- If signature validation is missing, the fast path's stack manipulation doesn't match the actual values

### Attack Vector
1. Declare a Wasm import matching a well-known import NAME but with WRONG SIGNATURE
2. Liftoff's well-known import detection may match on name only, not signature
3. Fast path manipulates stack assuming correct signature
4. Actual values on stack don't match → type confusion

### Key Source Files
- `src/wasm/baseline/liftoff-compiler.cc` — well-known import dispatch and stack manipulation
- `src/wasm/baseline/liftoff-assembler.cc` — stack model and validation
- `src/wasm/well-known-imports.h` — WellKnownImport enum
- `src/wasm/wasm-module.h` — import resolution

### Recent Related Changes
- **a58753bac7b**: Fix configureAll fast path — added signature validation
- **7df101eb619**: [wasm-custom-desc] Compliance with upstream tests — more custom descriptor changes
- **ed3db180cb7**: Remove mixed Wasm-in-JS inlining — changes to how well-known imports interact with inlining

## Techniques That Work
1. Enumerate ALL WellKnownImport values from well-known-imports.h
2. For each, check if signature validation exists in liftoff-compiler.cc CallWellKnownImport
3. Create test modules that import well-known functions with WRONG signatures
4. Monitor for stack corruption: compare Liftoff behavior vs interpreter (--wasm-jitless)
5. Focus on imports that manipulate multiple stack slots (multi-return, multi-param)
6. Test with `--liftoff-only` to isolate Liftoff behavior

## DO NOT Try These
- configureAll with incorrect signature — already fixed (a58753bac7b)
- Normal function calls (not well-known imports) — these go through standard calling convention
- Testing with correct signatures — the bug only manifests with INCORRECT signatures

## Evolution Plan
- v1: Enumerate well-known imports and audit signature validation. Read `src/wasm/well-known-imports.h` for the WellKnownImport enum. Read `src/wasm/baseline/liftoff-compiler.cc` for CallWellKnownImport dispatch. For each well-known import: does it validate the signature before doing its fast-path stack manipulation? List which ones have `ExpectCallWellKnownImport` and which don't.
- v2: Build test modules for UNVALIDATED well-known imports. Create imports matching the well-known import name but with wrong parameter count/types. Force Liftoff compilation. Check for crashes, assertion failures, or incorrect behavior.
- v3: Test tier differential. If Liftoff has a stack confusion bug, Turboshaft likely compiles the same code correctly (it doesn't use well-known import fast paths in the same way). Compare Liftoff-only vs Turboshaft-only output for the same module.

## Task
Start by reading:
1. `src/wasm/well-known-imports.h` — enumerate ALL WellKnownImport values
2. `src/wasm/baseline/liftoff-compiler.cc` — search for `WellKnownImport` handling:
   - Find the CallWellKnownImport function
   - For each case: does it validate the import's signature?
   - Does it have `ExpectCallWellKnownImport` or equivalent guard?
   - How does it manipulate the stack?
3. `src/wasm/wasm-module.cc` — how are well-known imports matched/resolved? Is matching name-only or name+signature?
4. The regress test from the fix: `test/mjsunit/regress/wasm/regress-489109716.js` — understand the attack pattern

The type invariant to challenge: "Every well-known import fast path in Liftoff correctly validates the import's signature before performing stack manipulation." The configureAll fix proves this was false for at least one import — find others.

Use `--liftoff-only` with `/Users/t/v8/out/fuzzbuild/d8`.
