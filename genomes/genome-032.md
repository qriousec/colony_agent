# Genome 032 — wasmfx-compress-refs

## Hypothesis
Commits 5a64071bf9d and 2f8c6526740 fixed compressed/uncompressed pointer confusion in the WasmFX stack entry wrapper (wrappers-inl.h BuildWasmStackEntryWrapper). References were stored using `MemoryRepresentation::FromMachineType(machine_type())` which returns compressed pointers for tagged values, but the consumer expected uncompressed pointers. The fix changed to `AnyUncompressedTagged()` for reference types. The 2-week gap between the return-value fix (Jan 30) and the parameter fix (Feb 13) demonstrates the pattern was incompletely reviewed. Other WasmFX code paths manipulating reference types in stack/buffer contexts likely have the same confusion: cont.bind argument forwarding, suspend/resume arg buffers, stack frame GC visitors, and WasmFXArgBuffer.

Multi-factor: WasmFX stack switching + reference type in argument/return buffer + compressed vs uncompressed pointer representation + GC or deref on wrong pointer width.

## DNA

### Confirmed Bug Pattern (P7)

**Fix 1** (5a64071bf9d, Jan 30 2026):
```cpp
// BEFORE (broken):
__ StoreOffHeap(result_buffer, returns[index],
    MemoryRepresentation::FromMachineType(sig_->GetReturn(index).machine_type()),
    offset);

// AFTER (fixed):
CanonicalValueType type = sig_->GetReturn(index);
MemoryRepresentation rep = type.is_ref()
    ? MemoryRepresentation::AnyUncompressedTagged()
    : MemoryRepresentation::FromMachineType(type.machine_type());
__ StoreOffHeap(result_buffer, returns[index], rep, offset);
```

**Fix 2** (2f8c6526740, Feb 13 2026):
Same pattern for parameters (not returns).

### Key Source Files
```
src/wasm/wrappers-inl.h — BuildWasmStackEntryWrapper (fixed), but also:
  - BuildWasmSuspendWrapper
  - BuildWasmResumeWrapper
  - ToJS / FromJS (boundary conversion)
  - IterateWasmFXArgBuffer (iteration helper)

src/wasm/baseline/liftoff-compiler.cc — Liftoff handling of wasmfx ops
src/compiler/turboshaft/wasm-lowering-reducer.h — Turboshaft wasmfx lowering
src/wasm/wasm-code-manager.cc — Stack management
```

### Grep Commands
```bash
# Find all FromMachineType usages for potential compressed/uncompressed confusion
grep -rn "FromMachineType\|AnyUncompressedTagged\|MemoryRepresentation" src/wasm/wrappers-inl.h

# Find all WasmFX-related wrapper builders
grep -rn "BuildWasm.*Wrapper\|SuspendWrapper\|ResumeWrapper\|StackEntryWrapper" src/wasm/wrappers-inl.h

# Find arg buffer manipulation
grep -rn "WasmFXArgBuffer\|arg_buffer\|result_buffer" src/wasm/wrappers-inl.h

# Find all wasmfx stack manipulation
grep -rn "cont\.bind\|cont_bind\|ContBind\|suspend\|resume" src/wasm/ --include="*.cc" --include="*.h" | head -30

# Recent wasmfx fixes
git -C /Users/t/v8 log --oneline --grep="wasmfx\|cont\\.bind\|stack.*switch\|suspend\|resume" -20 -- src/wasm/
```

### Related Commits
- 5a64071bf9d: Fix compressed/uncompressed confusion in stack entry returns
- 2f8c6526740: Fix same confusion in stack entry params (2 weeks later!)
- e97587a3751: Clear stack EPT entry on all return paths
- 74ec0dd3233: Reload memory cache after suspend
- a93849fa707: Implement cont.bind
- bc11de986c1: Skip empty stacks for code GC

## Techniques That Work
1. Build Wasm module using `--experimental-wasm-stack-switching` (or equivalent flag)
2. Create continuation types with reference-typed parameters and returns
3. Use cont.new, cont.bind, resume, suspend with reference arguments
4. Test with GC stress to trigger pointer decompression issues
5. Compare Liftoff vs Turboshaft behavior for reference args through stack switching
6. Focus on reference-typed values (structref, arrayref, funcref, externref) in wasmfx arg buffers

## DO NOT Try These
- Stack entry wrapper returns — FIXED (5a64071bf9d)
- Stack entry wrapper params — FIXED (2f8c6526740)
- Non-reference types (i32, f64) — not affected by compression
- Shared string unsharing — different bug class (WASM-TC-003)

## Evolution Plan
- v1: **Source audit of BuildWasmSuspendWrapper and BuildWasmResumeWrapper**. Check all `StoreOffHeap` and `LoadOffHeap` calls for reference types. Are they using `FromMachineType` (broken) or `AnyUncompressedTagged` (correct)? Also check `IterateWasmFXArgBuffer` helper — does it handle the compressed/uncompressed distinction?
- v2: **cont.bind argument forwarding**. cont.bind takes a continuation and additional arguments. When bound arguments are stored, are reference types stored compressed or uncompressed? Build a module: `cont.bind` with structref argument → `resume` → access the bound arg → compare with original.
- v3: **GC stress + stack switching**. Reference types in stack buffers that survive GC. If the GC visitor sees compressed pointers where it expects uncompressed (or vice versa), it may corrupt the pointer or miss a root. Test: resume continuation after forced GC, verify reference integrity.

## Task
Start with v1. Read these files IN THIS ORDER:

1. `src/wasm/wrappers-inl.h` — Search for ALL `StoreOffHeap` and `LoadOffHeap` calls. For each one involving reference types, check if it uses `FromMachineType` (potentially broken) or `AnyUncompressedTagged` (correct). Map all wrapper builders: BuildWasmStackEntryWrapper (fixed), BuildWasmSuspendWrapper, BuildWasmResumeWrapper, etc.
2. `src/wasm/wrappers-inl.h` — Find `IterateWasmFXArgBuffer` — how does it calculate offsets and sizes for reference types? Does it account for uncompressed size?
3. Check `src/wasm/baseline/liftoff-compiler.cc` for wasmfx ops — does Liftoff handle reference compression differently from Turboshaft?
4. `git -C /Users/t/v8 log --oneline -10 -- src/wasm/wrappers-inl.h` — recent changes to this file.

Your type invariant to challenge: "All reference types in WasmFX stack/buffer contexts use correct (uncompressed) pointer representation." The 2-week gap between fixes proves incomplete review. Find the next unfixed instance.

Post with tags `wasmfx,compression,pointer,stack-switching,lateral`.
