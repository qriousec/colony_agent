# Genome 043 — wasmfx-cont-compression

## Hypothesis
WasmFX stack switching has had 3 pointer compression/decompression bugs fixed in wrappers-inl.h within the last 2 months:
1. 5a64071bf9d: Return buffer — refs stored compressed, read uncompressed
2. 2f8c6526740: Params buffer — same bug for parameters
3. aa899a23983: func_ref loading — stored uncompressed, loaded compressed

These are all the SAME bug class: incorrect MemoryRepresentation for reference types in off-heap buffers. A potential UNFIXED instance exists at turboshaft-graph-interface.cc:3853-3855:

```cpp
V<WasmContinuationObject> stack_cont = __ Load(
    stack, LoadOp::Kind::RawAligned(),
    MemoryRepresentation::UintPtr(),  // ← SHOULD THIS BE AnyUncompressedTagged()?
    StackMemory::current_continuation_offset());
```

`current_cont_` is `Tagged<WasmContinuationObject>` (stacks.h:267), GC-visited as a FullObjectSlot (stacks.cc:115). On pointer-compressed builds, loading a Tagged value as UintPtr could yield a truncated/wrong address.

Additionally, any NEW WasmFX code that handles reference types in off-heap buffers may repeat the bug if the developer forgets to use `AnyUncompressedTagged()` instead of the default representation.

Multi-factor: WasmFX continuation + pointer compression + off-heap buffer + incorrect MemoryRepresentation + GC interaction.

## DNA

### Fixed Bugs (Same Pattern)

| Commit | Location | Bug | Fix |
|--------|----------|-----|-----|
| 5a64071bf9d | wrappers-inl.h:627-635 | Return buffer: refs stored compressed | Use AnyUncompressedTagged() for StoreOffHeap |
| 2f8c6526740 | wrappers-inl.h:607-615 | Params buffer: refs stored compressed | Use AnyUncompressedTagged() for LoadOffHeap |
| aa899a23983 | wrappers-inl.h:590-592 | func_ref: stored uncompressed, loaded compressed | Use UncompressedTaggedPointer() |

### Helper Function (Correct Pattern)
```cpp
// turboshaft-graph-interface.cc:3860-3862
MemoryRepresentation MemoryRepresentationOffHeap(ValueType type) {
  return type.is_ref() ? MemoryRepresentation::AnyUncompressedTagged()
                       : MemoryRepresentationFor(type);
}
```

### Potential Issue
**turboshaft-graph-interface.cc:3853-3855**:
```cpp
V<WasmContinuationObject> stack_cont = __ Load(
    stack, LoadOp::Kind::RawAligned(),
    MemoryRepresentation::UintPtr(),  // ← POTENTIAL BUG
    StackMemory::current_continuation_offset());
```

**stacks.h:267**: `Tagged<WasmContinuationObject> current_cont_;`
**stacks.cc:115**: GC-visited as `FullObjectSlot(reinterpret_cast<Address>(&current_cont_))`

### Grep Commands
```bash
# All MemoryRepresentation uses in WasmFX code
grep -rn "MemoryRepresentation" src/wasm/wrappers-inl.h | head -30
grep -rn "MemoryRepresentation" src/wasm/turboshaft-graph-interface.cc | grep -i "cont\|stack\|resume\|suspend" | head -20

# All off-heap loads of Tagged values in WasmFX
grep -rn "LoadOffHeap\|StoreOffHeap\|__ Load.*stack\|__ Store.*stack" src/wasm/wrappers-inl.h | head -20
grep -rn "current_continuation\|current_cont" src/wasm/ | head -20

# WasmFX experimental flag
grep -rn "experimental_wasm_wasmfx\|experimental.*stack.*switch" src/flags/ | head -5

# All ref type handling in wrappers
grep -rn "is_ref\|AnyUncompressedTagged\|UncompressedTaggedPointer" src/wasm/wrappers-inl.h | head -20
```

## Techniques That Work
1. Search for ALL instances where `MemoryRepresentation::UintPtr()` or `MemoryRepresentation::TaggedPointer()` is used for values that could be Tagged references in WasmFX code
2. Check every `LoadOffHeap`/`StoreOffHeap` in wrappers-inl.h for ref types not using AnyUncompressedTagged
3. Build module using cont.new + cont.bind + resume to exercise the continuation_offset loading path
4. Run with `--experimental-wasm-wasmfx` or `--experimental-wasm-stack-switching` (check which flag enables it)
5. Enable pointer compression assertions if available
6. Compare debug vs release build behavior

## DO NOT Try These
- The 3 already-fixed bugs (5a64071bf9d, 2f8c6526740, aa899a23983)
- cont.bind argument type validation — confirmed correct by research
- Basic stack switching without reference types — no compression issue

## Evolution Plan
- v1: **Source audit for MemoryRepresentation mismatches**. Read ALL uses of MemoryRepresentation in wrappers-inl.h and turboshaft-graph-interface.cc WasmFX code paths. List every Load/Store that handles a ref type. Flag any that DON'T use AnyUncompressedTagged(). Specifically investigate the current_continuation_offset load at line 3853.
- v2: **Build test module**. Create a module with cont.new, cont.bind with reference arguments, resume/suspend cycle. Store references in continuation, trigger GC, retrieve them. If pointer compression caused a wrong address, the retrieved reference is corrupt.
- v3: **Pattern grep across codebase**. Find ALL instances of `MemoryRepresentation::UintPtr()` or plain `MemoryRepresentation::TaggedPointer()` being used for Tagged values in Wasm code. Each is a potential compression bug.

## Task
Start with v1. Read these files:
1. `src/wasm/turboshaft-graph-interface.cc:3850-3900` — The WasmFX continuation code. Check if line 3853 is a bug.
2. `src/wasm/stacks.h` around line 267 — Confirm current_cont_ is Tagged
3. `src/wasm/wrappers-inl.h:580-650` — All the WasmFX stack entry/exit code. Verify every ref-type load/store uses correct representation.
4. Search for `MemoryRepresentation::UintPtr` in all Wasm code — any loading a Tagged value?

Post with tags `wasmfx,pointer-compression,continuation,memory-representation,pattern-audit`.
