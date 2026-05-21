# Genome 129 — inlining-trycatch-deopt

## Hypothesis
The fix at 250e6dbb826 added frame state propagation (`lazy_deopt_on_throw`) to `CallBuiltin` calls during JS-to-Wasm wrapper inlining inside try-catch. Before the fix, when a Wasm function was inlined into JS inside a try block, exceptions thrown during argument conversion (e.g., `FromJS` calling `ToNumber`) weren't caught by the enclosing try-catch because the `CallBuiltin` nodes lacked frame states needed for deoptimization.

**The hypothesis**: The fix only added `lazy_deopt_on_throw` to specific `CallBuiltin` calls in the wrapper inlining path. There may be OTHER builtins called during wrapper inlining — type conversion builtins (ToBigInt, ToObject), allocation builtins, or validation builtins — that still don't propagate frame state correctly. Exceptions from these builtins would bypass catch handlers, potentially corrupting execution state or leaking uninitialized values.

## DNA
- **Fix 250e6dbb826**: Added `lazy_deopt_on_throw` parameter to `CallBuiltin` in `src/wasm/wrappers-inl.h` and `src/wasm/wrappers.h`
  - The fix ensures that when a builtin called during Wasm argument conversion throws, the runtime can deoptimize back to the JS frame and propagate the exception to the catch handler
- **Reland f9d7d0fad66**: The original Wasm-in-JS inlining feature (W5 target). Has been relanded multiple times, suggesting instability
- **wrappers-inl.h**: Contains `BuildJSToWasmWrapper` and related functions that build the inlined wrapper graph. Uses `CallBuiltin` for type conversions (`kWasmFloat32ToNumber`, `kWasmFloat64ToNumber`, `kWasmTaggedNonSmiToInt32`, `kWasmTaggedToFloat32`, `kWasmTaggedToFloat64`, `kTrapHandlerThrowWasmError`, etc.)
- **wrappers.h**: Contains declarations and the `lazy_deopt_on_throw` parameter definition
- **P13 pattern**: `grep -rn 'CallBuiltin.*frame_state\|lazy_deopt_on_throw' src/wasm/wrappers-inl.h src/wasm/wrappers.h`
- **Frame state gap pattern**: If a `CallBuiltin` inside the inlined wrapper doesn't have a frame state, any exception it throws during deoptimization will not find a catch handler. The exception propagates up the stack incorrectly, potentially: (a) skipping finally blocks, (b) leaving stack frames in inconsistent state, (c) not releasing locks or resources held by the try block
- **Turboshaft inlining pipeline**: The wrapper is inlined at the Turboshaft level during the `WasmInJSInlining` phase. Frame states must be propagated through every potentially-throwing operation for the deoptimizer to reconstruct the JS frame

## Techniques That Work
1. Create a Wasm function with various parameter types (i32, i64, f32, f64, externref, funcref)
2. Call it from JS inside a try-catch block
3. Pass arguments that will cause type conversion to throw (e.g., Symbol for i32, BigInt for f64, object with throwing valueOf/toString)
4. Check that the exception is properly caught by the JS catch handler
5. Force inlining by calling the function many times (Turboshaft tier-up)
6. Test with `--turboshaft-wasm-in-js-inlining` flag if not enabled by default
7. After catching, verify no state corruption: check that locals in the try block are still valid, that the Wasm instance is still functional

## DO NOT Try These
- Shared types (--experimental-wasm-shared, BANNED)
- Non-inlined calls (Liftoff only) — the bug only manifests in Turboshaft inlining
- Exception handling without try-catch (bare throw) — need the catch handler to detect the bypass
- Testing only with primitives — need objects with throwing conversion methods

## Evolution Plan
- v1: **Audit all CallBuiltin sites in wrappers-inl.h**. Read the full file. For every `CallBuiltin` call: (a) does it have `lazy_deopt_on_throw`? (b) can the builtin throw? (c) is it inside a potentially-inlined wrapper path? Build a matrix of all CallBuiltin calls × throw potential × frame state presence. Any throwing CallBuiltin without frame state inside the inlined path is a candidate bug.
- v2: **Test argument conversion builtins**. For each Wasm parameter type, identify which builtins are called during JS→Wasm conversion. Test each with a throwing input:
  - i32: pass `{ valueOf() { throw "oops" } }` — does the try-catch catch it?
  - i64 (BigInt): pass `{ valueOf() { throw "oops" } }` — different conversion path
  - f32/f64: pass Symbol (TypeError) or throwing valueOf
  - externref: should not convert, but what about null vs undefined edge cases?
  - funcref: pass non-callable — does validation throw catchably?
  Verify each is caught AFTER inlining (run 10000 times first, then trigger the throw).
- v3: **Return value conversion (ToJS path)**. The fix focused on FromJS (argument conversion). But ToJS (return value conversion) also calls builtins. If a Wasm function returns a value that requires conversion (e.g., externref → JS object, or i64 → BigInt), and that conversion throws during inlining, is the frame state correct? Test: Wasm function returns i64, JS catches conversion exceptions. Also test: multiple return values where one conversion succeeds and the next throws — is partial state cleaned up?

## Task
Start by reading:
1. `src/wasm/wrappers-inl.h` — full file. Map EVERY `CallBuiltin` call. For each one, note:
   - What builtin is called
   - Is `lazy_deopt_on_throw` passed?
   - Can the builtin throw (throwing builtins vs non-throwing)
   - Is this in the JS→Wasm wrapper inlining path or elsewhere?
2. `src/wasm/wrappers.h` — the `lazy_deopt_on_throw` parameter and where it's used
3. `git -C /Users/t/v8 show 250e6dbb826` — the exact fix to understand what was added

Then build test 1: Wasm function taking i32 parameter, called from JS inside try-catch with throwing valueOf. Run 10000 times with valid input (force Turboshaft inlining), then call once with throwing input. Verify the exception is caught.

Run with: `/Users/t/v8/out/fuzzbuild/d8 test.js`
No --experimental-wasm-shared flag.
