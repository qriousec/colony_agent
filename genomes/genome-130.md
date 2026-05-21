# Genome 130 — tagged-untagged-gc-stale

## Hypothesis
The fix at 47a3535cddf corrected a stale reference bug in `array.fill`/`array.new` where `ChangeInt32ToInt64()` on a tagged value created an untagged representation. GVN (Global Value Numbering) then de-duplicated the tagged and untagged versions, spilling the untagged version to a stack slot. Since the stack slot was marked as untagged (Int64), the garbage collector ignored it during collection. If GC moved the referenced object, the stack slot held a stale pointer to the old location — a use-after-move bug.

**The hypothesis**: The pattern — tagged-to-untagged conversion applied to a GC reference value before a potential GC point — may exist in OTHER Turboshaft Wasm operations. Any site in `turboshaft-graph-interface.cc` that:
1. Takes a tagged reference value (OpIndex representing a Wasm GC ref)
2. Applies a bitwise/numeric conversion (ChangeInt32ToInt64, BitcastTaggedToWord, etc.) creating an untagged representation
3. Has a subsequent operation that can trigger GC (allocation, call, array.new, struct.new)

...creates a window where GVN might de-duplicate and the GC misses the reference. The fix changed `StoreInInt64StackSlot` to `StoreInStackSlot` (which preserves tagging), but only at the specific `array.fill`/`array.new` site.

## DNA
- **Fix 47a3535cddf**: In `src/wasm/turboshaft-graph-interface.cc:8001`
  - Changed `StoreInInt64StackSlot` → `StoreInStackSlot`
  - `StoreInInt64StackSlot` creates an untagged Int64 stack slot — invisible to GC
  - `StoreInStackSlot` preserves the tagged representation — GC traces it
- **GVN deduplication**: When `ChangeInt32ToInt64(tagged_value)` creates a new untagged OpIndex, GVN recognizes that the underlying value is the same as the tagged OpIndex. If the untagged version is spilled to the stack, GVN may eliminate the tagged version's stack slot, leaving only the untagged one. GC won't scan the untagged slot.
- **P12 pattern**: `grep -rn 'ChangeInt32ToInt64\|StoreInInt64StackSlot\|StoreInStackSlot' src/wasm/turboshaft-graph-interface.cc`
- **turboshaft-graph-interface.cc**: This is the main file that translates Wasm bytecode to Turboshaft IR. It handles ALL Wasm operations. Any Wasm operation that passes GC refs to C++ runtime calls via stack slots is at risk.
- **Runtime calls**: Many Wasm operations call into the V8 runtime (e.g., `CallBuiltinThroughJumptable`, `CallRuntime`). These calls can trigger GC. If a GC ref argument is stored in an untagged stack slot for the call, GC won't update it.
- **array.new_fixed, array.copy, struct.new**: These operations allocate new GC objects and take GC refs as arguments. They're prime candidates for the same pattern.

## Techniques That Work
1. Search turboshaft-graph-interface.cc for ALL uses of `StoreInInt64StackSlot` — each one is a potential stale ref site
2. Search for `ChangeInt32ToInt64` applied to values that could be tagged GC refs
3. Search for `BitcastTaggedToWord` or other tag-stripping operations before GC points
4. Build tests that: (a) create many GC objects to fill the young generation, (b) perform the target operation with a GC ref argument, (c) trigger GC during the operation (via concurrent allocation or large allocation), (d) use the returned reference and verify it's still valid
5. Use `--stress-compaction` or `--gc-interval=100` to increase GC frequency
6. Force Turboshaft tier-up for the lowering to apply

## DO NOT Try These
- Shared types (--experimental-wasm-shared, BANNED)
- Liftoff testing — the stale ref only occurs in Turboshaft-compiled code where GVN runs
- Tests without GC pressure — the bug only manifests when GC actually moves objects
- Simple tests with no intervening allocations between store and use

## Evolution Plan
- v1: **Grep audit of turboshaft-graph-interface.cc**. Find ALL instances of:
  - `StoreInInt64StackSlot` — any remaining uses with tagged values
  - `ChangeInt32ToInt64` — where the input could be a tagged ref
  - `BitcastTaggedToWord`, `BitcastWordPtrToSmi`, or similar untagging operations
  - `CallBuiltinThroughJumptable` and `CallRuntime` — any call that takes GC ref args from stack slots
  For each match, determine: is the input a GC ref? Is there a GC point between the conversion and the use? Was it fixed in 47a3535cddf?
- v2: **Test remaining StoreInInt64StackSlot sites**. For each unfixed site found in v1, build a test that exercises that specific code path with GC pressure. Key operations to test: `array.new_fixed` (multiple ref arguments stored to stack), `array.copy` (source array ref + dest array ref), `struct.new` with many ref fields, `table.set` with ref argument, `global.set` with ref argument.

## Task
Start by reading:
1. `src/wasm/turboshaft-graph-interface.cc` — search for `StoreInInt64StackSlot`. List ALL occurrences. For each, determine what value is being stored and whether it could be a GC ref.
2. `git -C /Users/t/v8 show 47a3535cddf` — the exact fix, to understand the precise pattern
3. Search for `ChangeInt32ToInt64` in the same file — map where tagged values undergo untagging
4. Search for `BitcastTaggedToWord` and similar operations

Then build test 1: Create a Wasm module with `array.new` taking a ref-typed fill value. Allocate many objects to fill the young generation. Call `array.new` with a ref-typed fill value (struct instance). Use `--gc-interval=50` to trigger frequent GC. After array.new returns, iterate the array and verify all elements still point to valid objects. Force Turboshaft tier-up.

Run with: `/Users/t/v8/out/fuzzbuild/d8 --gc-interval=50 test.js`
No --experimental-wasm-shared flag.
