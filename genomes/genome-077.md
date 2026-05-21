# Genome 077 — wasm-inline-body-trap

## Hypothesis
The new Turboshaft Wasm-in-JS body inlining (wasm-in-js-inlining-reducer-inl.h) inlines simple Wasm function bodies directly into JS compilation. It bails out for LazyDeoptOnThrow calls and try-blocks, but proceeds for functions containing trapping operations (memory OOB, div by zero, table bounds, ref.cast trap). The developer explicitly stated in commit adc761a24dc: "I am not confident we handle it correctly" regarding exception handling in inlined bodies.

When a Wasm function is inlined into a JS function and the Wasm body traps:
1. The trap must create a WebAssembly.RuntimeError
2. The error must be catchable by JS try-catch around the call site
3. The deopt frame state must correctly restore the JS stack

If any of these fail, the trap either:
- Bypasses JS exception handlers (the comment at line 1404-1405 warns about this)
- Corrupts the JS stack frame on deoptimization
- Leaves the JS compilation in an inconsistent state where subsequent calls behave differently

The dry-run decoder (line 1445-1457) checks for unsupported instructions but may classify trapping instructions as "supported" since they're basic Wasm operations. The actual trap handling may not be implemented for the inlined case.

## DNA

### Source Facts — Inlining Bailout Logic
- **wasm-in-js-inlining-reducer-inl.h:1388-1410**: Bailout conditions:
  - Line 1399: `has_lazy_deopt_on_throw` → bail out (TODO for support)
  - Line 1403-1409: `has_current_catch_block` → bail out ("Wasm traps ignoring catch handlers in the inlined JS frame")
  - Line 1463: Unsupported instruction in dry-run → bail out
- **wasm-in-js-inlining-reducer-inl.h:1434-1436**: `DCHECK_LE(func.sig->return_count(), 1)` — no multi-return

### Source Facts — Trap Handling in Inlined Code
- When Wasm is inlined into JS, traps (OOB, div/0, null deref) generate machine-level signals
- V8's signal handler converts these to JS exceptions
- For standalone Wasm: trap → signal → V8 handler → throw WebAssembly.RuntimeError
- For inlined Wasm in JS: trap → signal → V8 handler → ??? (deopt? direct throw? frame corruption?)

### Source Facts — Developer Uncertainty
- **adc761a24dc commit message**: "The body inlining doesn't support any throwing operation in the body yet, so it is not broken right now. But as soon as we will add a throwing instruction to the body inlining implementation, I am not confident we handle it correctly"
- This means: body inlining CURRENTLY only supports non-throwing Wasm instructions. If the dry-run decoder allows a function with trapping operations through, the trap is UNHANDLED in the inlined path.

### Key Questions
1. Does the dry-run decoder (`WasmInJsInliningInterface`) classify trapping operations (memory.load, table.get, ref.cast, i32.div_s) as "supported" or "unsupported"?
2. If "supported", what code is emitted for the trap? Is it a Wasm trap node or a JS throw?
3. If the trap fires, does the deopt frame state correctly allow the JS caller to catch the error?
4. Is `--turboshaft-wasm-in-js-inlining` enabled by default or behind a flag?

### Entry Point for Testing
- The inlining is triggered when a JS function calls a Wasm export repeatedly (causing hot loop)
- TurboFan/Turboshaft decides to inline the Wasm call based on feedback
- The `JSWasmCallParameters` are attached to the call node by the TurboFan frontend

## Techniques That Work
1. Create a simple Wasm function that can trap (memory.load at boundary, div by zero)
2. Call it from a JS function in a hot loop to trigger inlining
3. After inlining, trigger the trap condition
4. Wrap the JS call in try-catch to verify the exception is catchable
5. Compare behavior: inlined (optimized) vs non-inlined (interpreted/baseline)
6. Use `%PrepareFunctionForOptimization`, `%OptimizeFunctionOnNextCall` to force optimization
7. Use `--trace-turbo` to verify the function was actually inlined

## DO NOT Try These
- Testing Wasm-in-JS inlining type gaps for IsCastToCustomDescriptor — DEAD SURFACE
- Testing exactness in inlined path — confirmed semantically equivalent by 3 workers
- Testing without triggering actual inlining — need to verify inlining happened

## Evolution Plan
- v1: Determine if Wasm-in-JS body inlining is enabled by default. If behind a flag, find the flag. Then create a minimal Wasm function with memory.load and call it from JS. Force optimization and verify the function is inlined. Trigger OOB trap. Check if the exception is properly caught by JS try-catch.
- v2: Test multiple trapping operations: (a) memory.load OOB, (b) i32.div_s by zero, (c) table.get OOB, (d) ref.cast null. For each, verify that the trap produces a catchable WebAssembly.RuntimeError. Compare the error message and stack trace between inlined and non-inlined execution.
- v3: Test the deopt path: Force optimization with inlined Wasm, trigger a deopt via a type change in the JS caller, then trigger a trap in the deoptimized code. Check if the frame state is correctly restored after deopt and if subsequent traps are properly handled.

## Task
Start by reading:
1. `src/compiler/turboshaft/wasm-in-js-inlining-reducer-inl.h:1380-1470` — TryInlineWasmCall, focusing on bailout conditions and dry-run decoder
2. `src/compiler/turboshaft/wasm-in-js-inlining-reducer-inl.h:78-130` — REDUCE(Call) entry point, how inlining decision is made
3. `src/compiler/turboshaft/wasm-in-js-inlining-reducer-inl.h:1200-1350` — TryInlineJSWasmCallWrapperAndBody, how wrapper + body are inlined
4. Search for `--turboshaft-wasm-in-js-inlining` or `turboshaft_wasm_in_js_inlining` flag to see if it's on by default

Name the type invariant to challenge: **Wasm traps in inlined Wasm body code must produce catchable WebAssembly.RuntimeError exceptions with correct stack traces, identical to non-inlined execution.** If trap handling is not implemented for inlined bodies, traps may crash, bypass catch handlers, or produce wrong error types.

Start with v1 — check flag status, verify inlining works, test basic trap handling.
