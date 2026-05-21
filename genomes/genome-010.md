# Genome 010 — wasmfx-cont-leak

## Hypothesis
WasmFX continuation types have had 10+ bug-fix commits recently. The compressed/uncompressed pointer confusion (2f8c6526740, 5a64071bf9d) in stack entry params means continuation stack frames were storing tagged GC references with the wrong pointer encoding. If a similar confusion still exists in an unpatched path, a tagged pointer stored as uncompressed (8 bytes) but read as compressed (4 bytes) would produce a corrupted pointer — interpreting the upper 4 bytes of one pointer as part of an adjacent value. This is a type confusion at the memory representation level. Additionally, the memory cache reload issues (74ec0dd3233, 8e06a26e126) after suspend/resume mean that Wasm linear memory caches may point to stale memory after a continuation switch, allowing reads/writes to freed memory.

## DNA

### Source Facts (from git log)
- `847c7b2461a [wasmfx] Fix leak of indexed cont types to JS` — indexed cont types could be exposed to JS (missing type check in WasmObjectToJSReturnValue and IsJSCompatibleSignature)
- `2f8c6526740 [wasmfx] Fix compressed/uncompressed confusion in stack entry params` — stack entry params stored with wrong pointer size
- `5a64071bf9d [wasmfx] Add tests and fix a compressed/uncompressed confusion` — ANOTHER compressed/uncompressed fix (!)
- `99d0d485292 [wasmfx] Fix stale reference in wasm::StackMemory` — stale reference bug
- `74ec0dd3233 [wasmfx] Reload memory cache after suspend` — memory cache not reloaded after suspend
- `8e06a26e126 [wasmfx] Add missing memory cache reload` — ANOTHER missing reload
- `12735419590 [wasmfx] Fix incorrect JSPI/WasmFX interaction` — interaction bug with JSPI
- `e97587a3751 [wasmfx] Clear stack EPT entry on all return paths` — stack entry not cleared

### Key Files
- `src/wasm/stacks.h` / `src/wasm/stacks.cc` — stack memory management
- `src/wasm/wasm-js.cc` — WasmFX JS boundary
- `src/wasm/turboshaft-graph-interface.cc` — WasmFX graph building
- `src/wasm/baseline/liftoff-compiler.cc` — WasmFX in Liftoff

### Attack Vectors
1. **Compressed/uncompressed confusion**: Find paths where stack entry params are stored with TaggedSize but read with kSystemPointerSize (or vice versa). The two prior fixes suggest this is a recurring pattern.
2. **Memory cache after suspend/resume**: After a continuation suspends and resumes, the linear memory may have been grown or the base pointer may have changed. If the memory cache isn't reloaded, subsequent memory accesses use stale pointers.
3. **Stale stack references**: If a StackMemory object is freed but a reference to it persists in a continuation, accessing it causes use-after-free.
4. **Cont type at JS boundary**: Even after the fix at 847c7b2461a, there may be other paths where cont references leak to JS (e.g., through exception handling, or table operations).

## Techniques That Work
1. Read the compressed/uncompressed fix diffs (2f8c6526740, 5a64071bf9d) to understand the pattern
2. Search for similar patterns in unpatched code: `grep -rn 'kTaggedSize\|kSystemPointerSize\|COMPRESS_POINTERS' src/wasm/stacks*`
3. Create a module using cont.new, cont.bind, resume, suspend to exercise continuation stack frames
4. Test memory growth during continuation suspension
5. Run with `--experimental-wasm-stack-switching`

## DO NOT Try These
- Testing without wasm stack switching flag
- Simple continuation tests without GC reference params (compressed/uncompressed only matters for tagged pointers)
- Testing with only numeric types in continuation parameters

## Evolution Plan
- v1: Read the two compressed/uncompressed fix diffs. Map the stack entry parameter storage code. Search for similar unpatched patterns.
- v2: If v1 finds an unpatched path, construct a test with a continuation that passes GC struct references through suspend/resume. Check if the reference is corrupted. If patched, pivot to memory cache: test memory.grow during a suspended continuation.
- v3: If v2 finds corruption, minimize. If clean, test stale reference: create a continuation, let its stack be collected, then try to resume it.

## Task
Start by reading:
1. `git -C /Users/t/v8 show 2f8c6526740` — first compressed/uncompressed fix
2. `git -C /Users/t/v8 show 5a64071bf9d` — second compressed/uncompressed fix
3. `git -C /Users/t/v8 show 74ec0dd3233` — memory cache reload fix
4. `src/wasm/stacks.h` — stack memory management, focus on param_types and stack entry storage
5. Search: `grep -rn 'kTaggedSize\|TaggedSize\|kCompressedPointerSize' src/wasm/stacks*`

Type invariant to challenge: "Continuation stack frames store and retrieve tagged GC references with correct pointer encoding."

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc --experimental-wasm-stack-switching`
