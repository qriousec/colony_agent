# Genome 046 — interp-custom-desc

## Hypothesis
The Wasm interpreter has ZERO support for custom descriptor operations. The interpreter-gc-type-cast worker (genome 036) confirmed that standard ref.cast/ref.test GenericKind gaps are validation-defended. But that worker did NOT test custom descriptor operations:
- `ref.get_desc` — get the descriptor of a described struct
- `ref.cast_desc_eq` — cast with descriptor equality
- `struct.new_desc` — create described struct

If the interpreter encounters these opcodes when running with `--wasm-interpret-all --experimental-wasm-custom-descriptors`, it may:
1. Hit UNREACHABLE (crash in debug builds)
2. Silently produce wrong results (type confusion in release builds)
3. Fall through to default handling (incorrect semantics)

The 7+ bug fixes in custom descriptors were all for the COMPILER paths (Liftoff, Turboshaft). The interpreter is a completely separate implementation that may not have been updated.

Multi-factor: Wasm interpreter + custom descriptor operations + experimental flag + tier differential vs compiled code.

## DNA

### Key Source Files
```
src/wasm/interpreter/wasm-interpreter.cc — Main interpreter
src/wasm/interpreter/wasm-interpreter-runtime.cc — Runtime helpers
src/wasm/wasm-opcodes.h:782-788 — Custom descriptor opcodes exist
src/wasm/function-body-decoder-impl.h:6234-6313 — Custom descriptor validation
```

### Grep Commands
```bash
# Do custom descriptor opcodes exist in the interpreter?
grep -rn "get_desc\|cast_desc\|struct_new_desc\|kRefGetDesc\|kRefCastDescEq\|kStructNewDesc" src/wasm/interpreter/ | head -10

# If no results, the interpreter DOES NOT handle these opcodes

# Custom descriptor opcodes in the decoder
grep -rn "kExprRefGetDesc\|kExprRefCastDescEq\|kExprStructNewDesc" src/wasm/ | head -10

# Interpreter opcode dispatch
grep -rn "switch.*opcode\|kExpr.*:" src/wasm/interpreter/wasm-interpreter.cc | tail -30

# What flag enables custom descriptors?
grep -rn "experimental_wasm_custom_descriptors\|custom.desc" src/flags/ | head -5
```

### Previous Colony Work
- interpreter-gc-type-cast (genome 036): 3 iters, CLEAN. Confirmed standard ref.cast gaps are validation-defended. Did NOT test custom descriptors.
- custom-desc-cast: 8 iters on IsCastToCustomDescriptor optimizer guard → CLEAN
- wasm-custom-desc-runtime (genome 035): non-compliant (0 output)

## Techniques That Work
1. Build module with custom descriptor types:
   ```javascript
   // Type $Struct is described by type $Desc
   let struct_type = builder.addStruct({fields: [makeField(kWasmI32, true)]});
   let desc_type = builder.addStruct({fields: [makeField(kWasmI32, true)], describes: struct_type});
   ```
2. Use ref.get_desc, ref.cast_desc_eq, struct.new_desc in function bodies
3. Run with `--wasm-interpret-all --experimental-wasm-custom-descriptors`
4. Compare with `--liftoff --no-wasm-tier-up --experimental-wasm-custom-descriptors`
5. Debug build: check for UNREACHABLE/DCHECK failures
6. Release build: check for wrong results (type confusion)

## DO NOT Try These
- Standard ref.cast/ref.test — confirmed defended in interpreter
- Custom descriptor OPTIMIZER guards — confirmed defended (8 iters)
- Shared + custom descriptors — UNIMPLEMENTED guard

## Evolution Plan
- v1: **Quick probe**. Search the interpreter source for ANY mention of custom descriptor opcodes. If absent, build a simple module with ref.get_desc and run with `--wasm-interpret-all`. Expect crash or wrong result.
- v2: **Differential test**. Build module using ref.get_desc + ref.cast_desc_eq. Run in interpreter vs Liftoff. Compare: does the interpreter produce the same result?
- v3: **Exploitation**. If interpreter mishandles these ops, determine: (a) does it crash (DoS)? (b) does it produce a wrong type (type confusion)? (c) can the wrong type lead to field access on a struct with different layout?

## Task
Start with v1. Run:
```bash
grep -rn "get_desc\|cast_desc\|struct_new_desc" /Users/t/v8/src/wasm/interpreter/ | head -20
```
If no results, the interpreter doesn't support these ops. Then build a test module:
```javascript
d8.file.execute("test/mjsunit/wasm/wasm-module-builder.js");
// ... build module with custom descriptors ...
// Run with --wasm-interpret-all --experimental-wasm-custom-descriptors
```

Post with tags `interpreter,custom-descriptor,tier-diff,probe`.
