# Genome 128 — smi-check-regression-hunt

## Hypothesis
The refactoring crrev.com/c/7054524 changed how Wasm type representations work, causing equality comparisons like `config.from.AsNonShared() == wasm::kWasmExternRef` to silently return false where they previously returned true. This broke Smi/i31ref checks — the `object_can_be_i31` flag became false when it should be true, causing i31ref values to be treated as heap objects. The fix at d86212a39be changed 4 sites to use `config.from.is_reference_to(GenericKind::kExtern)` instead.

**The hypothesis**: There are OTHER equality comparisons in Turboshaft/TurboFan Wasm lowering that suffered the same silent regression — where the type representation change caused a comparison to flip from true to false, silently disabling a code path. Any such site where `object_can_be_i31` is wrong, or where a type-based branch takes the wrong arm, could lead to type confusion, crashes, or exploitable memory corruption.

## DNA
- **Fix d86212a39be**: Changed 4 sites:
  - `wasm-lowering-reducer.h:707`: `config.from.AsNonShared() == wasm::kWasmExternRef` → `config.from.is_reference_to(wasm::GenericKind::kExtern)` (for cast lowering, object_can_be_i31 computation)
  - `wasm-lowering-reducer.h:793`: Same pattern, different cast operation
  - `wasm-gc-lowering.cc:206`: Same pattern in TurboFan lowering
  - `wasm-gc-lowering.cc:382`: Same pattern
- **Root cause**: crrev.com/c/7054524 refactored Wasm type representation. After the refactoring, `AsNonShared()` returns a different representation that no longer compares equal to the predefined constant `kWasmExternRef` even for extern ref types. The `is_reference_to()` method works correctly with the new representation.
- **P11 pattern**: `grep -rn 'object_can_be_i31\|kWasmExternRef.*==\|AsNonShared.*==' src/compiler/turboshaft/wasm-lowering-reducer.h src/compiler/wasm-gc-lowering.cc`
- **Impact**: When `object_can_be_i31` is false but should be true, the compiler generates code that doesn't check for Smi/i31ref values. If a Wasm function receives an i31ref (which is represented as a Smi), the generated code treats it as a heap pointer. Dereferencing a Smi as a heap object leads to a controlled read at `2*smi_value` address.
- **Scope expansion**: The fix only patched `kWasmExternRef` comparisons. But the same pattern could exist for `kWasmAnyRef`, `kWasmEqRef`, `kWasmStructRef`, `kWasmArrayRef`, `kWasmFuncRef`, or any other predefined type constant.

## Techniques That Work
1. Search ALL files in `src/compiler/` and `src/wasm/` for `AsNonShared() ==` and `== kWasm` patterns that compare type representations using equality
2. For each match, check if `is_reference_to()` would be the correct comparison method
3. Build tests that exercise the specific code paths — e.g., cast an externref that is actually an i31ref (Smi), or cast an anyref that could be i31ref
4. Force Turboshaft tier-up so the lowering reducer runs
5. Use `--turboshaft-wasm` (default on) to engage Turboshaft lowering

## DO NOT Try These
- Shared types (--experimental-wasm-shared, BANNED)
- Liftoff-only testing — the regression is in Turboshaft/TurboFan lowering
- Types that don't involve reference types (i32, i64, f32, f64) — the pattern only affects ref type comparisons

## Evolution Plan
- v1: **Comprehensive grep audit**. Search for ALL instances of `AsNonShared()` followed by `==` in the entire `src/compiler/` and `src/wasm/` directories. Also search for `== kWasmAnyRef`, `== kWasmEqRef`, `== kWasmFuncRef`, `== kWasmStructRef`, `== kWasmArrayRef`, `== kWasmExternRef` across the codebase. For each hit, determine: (a) was it fixed in d86212a39be? (b) if not, is it a potential sibling bug? Build a map of all comparison sites and their status.
- v2: **Test unfixed siblings**. For each unfixed equality comparison found in v1, build a targeted test. Key test pattern: create a Wasm function that receives a reference type matching the comparison target, pass an i31ref value through that path, and check whether Smi handling is correct. If `object_can_be_i31` is wrong, the generated code will crash or produce wrong results on i31ref input.
- v3: **TurboFan vs Turboshaft parity check**. The fix patched both wasm-lowering-reducer.h (Turboshaft) and wasm-gc-lowering.cc (TurboFan). Check if there are paths where only one was fixed but not the other. Also check: `wasm-compiler.cc`, `wasm-graph-assembler.h`, and any other file that lowers Wasm type checks.

## Task
Start by running these searches:
1. `grep -rn 'AsNonShared().*==' /Users/t/v8/src/compiler/ /Users/t/v8/src/wasm/` — find ALL equality comparisons using AsNonShared
2. `grep -rn '== kWasm.*Ref\|kWasm.*Ref ==' /Users/t/v8/src/compiler/ /Users/t/v8/src/wasm/` — find ALL comparisons against predefined Wasm ref type constants
3. `grep -rn 'object_can_be_i31\|object_can_be_null' /Users/t/v8/src/compiler/turboshaft/wasm-lowering-reducer.h` — find ALL i31/null check computations
4. Read `src/compiler/turboshaft/wasm-lowering-reducer.h` lines 690-830 — the fixed code area, to understand the pattern

Cross-reference the grep results: any `AsNonShared() == kWasm*` that is NOT `kWasmExternRef` is a potential unfixed sibling. Any `== kWasm*Ref` in lowering code that doesn't use `is_reference_to()` is suspect.

Run with: `/Users/t/v8/out/fuzzbuild/d8 test.js`
No --experimental-wasm-shared flag.
