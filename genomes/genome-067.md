# Genome 067 — turboshaft-isel-wasm-casts

## Hypothesis
Commit 687ac3ebd63 fixed a wrong `Cast<ChangeOp>` that should have been `Cast<TryChangeOp>` in `instruction-selection-normalization-reducer.h:89`. In a switch-on-opcode statement, `kTryChange` was matched but then cast to `ChangeOp` — a compiler-internal type confusion. This pattern can exist elsewhere in Turboshaft reducers.

If a Wasm-related opcode (e.g., `kWasmTypeCast`, `kWasmTypeCheck`, `kWasmRefCast`) is matched in a switch but cast to the wrong operation type, the reducer reads wrong fields from the operation struct. This could cause:
1. Wrong input being used (operation reads field at wrong offset)
2. Wrong kind/mode being used (e.g., treating a check-and-branch as check-and-trap)
3. Wrong type being used for optimization decisions

Multi-factor: [wrong Cast<> in switch-on-opcode] × [Wasm GC operation confusion] × [wrong field access] = incorrect code generation.

## DNA

### Source Facts
```cpp
// instruction-selection-normalization-reducer.h:86-89 (FIXED)
case Opcode::kTryChange:
  return IsComplexConstant(op.Cast<TryChangeOp>().input());  // was Cast<ChangeOp>

// ChangeOp and TryChangeOp are different structs:
// ChangeOp: input, kind, assumption, from, to
// TryChangeOp: input, kind, from, to (no assumption field!)
// If Cast<ChangeOp>() is used on TryChangeOp, field offsets may be wrong
```

### Files to Audit (grep for Cast<> patterns)
- `src/compiler/turboshaft/instruction-selection-normalization-reducer.h` — fixed file, check for other instances
- `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.h` — Wasm GC type operations
- `src/compiler/turboshaft/wasm-lowering-reducer.h` — Wasm lowering
- `src/compiler/turboshaft/wasm-dead-code-elimination-reducer.h` — DCE for Wasm
- `src/compiler/turboshaft/wasm-revec-reducer.h` — SIMD revectorization
- `src/compiler/turboshaft/value-numbering-reducer.h` — GVN
- `src/compiler/turboshaft/machine-lowering-reducer-inl.h` — machine lowering
- All `*reducer*.h` files in turboshaft

### Target Pattern
```
case Opcode::kFoo:
  ... op.Cast<BarOp>() ...  // WRONG if Foo != Bar
```

Where Foo and Bar are similar but different opcodes (like Change/TryChange, TypeCast/TypeCheck, etc.).

### Wasm-specific Operation Pairs to Check
- `kWasmTypeCast` / `kWasmTypeCheck` — cast vs check semantics
- `kWasmTypeCastAbstract` / `kWasmTypeCheckAbstract`
- `kWasmRefFunc` / `kWasmRefFuncWithRtt`
- `kWasmAllocateArray` / `kWasmAllocateStruct`
- Any Wasm opcode in a switch that might be cast to wrong Op type

## Ready-to-Run Grep Commands

```bash
# Find all Cast<> in switch cases across turboshaft reducers
grep -n 'Cast<.*Op>' src/compiler/turboshaft/*reducer*.h | grep -v test | head -100

# Find switch(opcode) blocks in reducers
grep -n 'case Opcode::k' src/compiler/turboshaft/*reducer*.h | head -100

# Cross-reference: find cases where opcode name != Cast type name
# e.g., case kFoo: ... Cast<BarOp> where Foo != Bar
```

## Evolution Plan
- v1: Run the grep commands. Cross-reference every `case Opcode::kX` with its `Cast<YOp>()` to find mismatches. Report any found.

## Task
**THIS IS A SOURCE-ONLY AUDIT.** Do not run Wasm tests. Instead:

1. Search all Turboshaft reducer files for `Cast<` patterns
2. For each, verify the cast type matches the opcode in the enclosing switch case
3. Focus on Wasm-specific operations (kWasm*, kGC*) and pairs that could be confused
4. If a mismatch is found, analyze the impact: what fields would be read at wrong offsets?

Post with tags `turboshaft,cast-pattern,wrong-cast,instruction-selection,W25,source-audit`.
