# Genome 073 — shared-string-const-leak

## Hypothesis
Shared string constants (commit 0a3073a0b0c) are created via Factory::NewSharedStringFromUtf8 and allocated in shared old space (AllocationType::kSharedOld). When a shared Wasm module declares string constants and returns them through exported functions typed as `(ref shared any)`, `(ref shared eq)`, or `(ref shared struct)` containing string fields, the compiled Turboshaft wrapper at wrappers-inl.h:138 won't unshare them because `IsSubtypeOf(type, kWasmSharedExternRef)` returns false for non-extern shared ref types. This is the TC-003/004 pattern applied to a NEW entry point for shared strings — module-level string constants that are inherently shared, bypassing the need for runtime shared string creation.

## DNA

### Source Facts — Shared String Constants Path
- **wasm-objects.cc:136-156**: `ExtractUtf8StringFromModuleBytes` now takes a `bool shared` parameter. When shared=true, calls `factory->NewSharedStringFromUtf8()` instead of `NewStringFromUtf8()`.
- **factory.cc:832-843**: `NewSharedStringFromUtf8` allocates with `AllocationType::kSharedOld` via `SharedStringBuilder{}`. These strings are in shared heap space.
- **module-instantiate.cc:1801-1815**: Magic string constants for imported globals — another path where shared strings enter.

### Source Facts — TC-003/004 Bug (Confirmed, Parent Pattern)
- **wrappers-inl.h:138**: `if (IsSubtypeOf(type, kWasmSharedExternRef))` — TYPE-based guard, only triggers for shared externref subtypes
- **wasm-subtyping.cc:402**: `IsSubtypeOf_CommonImpl` returns false when `is_shared()` differs between sub and supertype — shared anyref is NOT a subtype of shared externref (different hierarchies: any vs extern)
- **js-to-wasm.tq:881**: Torque path uses VALUE-based check (Is<String> && InSharedSpace) — correct
- **wasm-objects.cc:3765**: C++ path uses VALUE-based check — correct
- Compiled Turboshaft wrapper path uses TYPE-based check — INCOMPLETE (bug)

### Key Differential
| Path | Check | Shared string const through anyref |
|------|-------|------------------------------------|
| Torque generic wrapper (<1000 calls) | VALUE-based | UNSHARED (correct) |
| C++ runtime | VALUE-based | UNSHARED (correct) |
| Compiled Turboshaft (>1000 calls) | TYPE-based | LEAKED (bug) |

### New Attack Vectors via String Constants
1. `string.const` instruction returns a shared string constant → stored in anyref local → returned through exported function typed `(ref shared any)` → wrapper doesn't unshare
2. Shared struct with anyref field → field initialized from string.const → struct.get returns shared string → wrapper doesn't unshare
3. Global of type `(ref shared any)` initialized with string.const → exported function reads global → wrapper doesn't unshare
4. Array of `(ref shared any)` filled with string constants → array.get returns shared string → wrapper doesn't unshare

## Techniques That Work
1. Use `--experimental-wasm-shared --shared-string-table` flags
2. Create shared module with string.const instructions
3. Export function returning `(ref shared any)` that returns string constant
4. Call function >1000 times to trigger wrapper tier-up
5. Check `Atomics.isLockFree(value) === undefined` or `%IsSharedString(value)` to detect leaked shared strings
6. Compare behavior at call 1 (Torque) vs call 1001 (compiled)

## DO NOT Try These
- Testing shared externref return type — this is the ONE path that's correctly handled
- Testing without tier-up — the Torque generic wrapper correctly unshares
- Testing with --no-wasm-generic-wrapper — this just skips directly to compiled wrapper, same result as post-tier-up

## Evolution Plan
- v1: Create minimal shared module with string.const + exported function returning (ref shared any). Verify shared string constants are allocated in shared space. Test pre-tier-up vs post-tier-up differential. Expect DIRTY.
- v2: If v1 confirms, expand to struct.get, array.get, and global.get paths for shared string constants. Test all 4 attack vectors. Also test if string.const in a NON-shared module creates non-shared strings and whether converting them to shared anyref still leaks.
- v3: Investigate if shared string constants interact with string interning — if factory.cc interns the shared string, the same string object might be referenced from both shared and non-shared modules, creating a cross-isolate string reference without going through the boundary.

## Task
Start by reading:
1. `src/wasm/wasm-objects.cc:136-156` — ExtractUtf8StringFromModuleBytes with shared flag
2. `src/heap/factory.cc:830-850` — NewSharedStringFromUtf8 implementation
3. `src/wasm/module-instantiate.cc:1795-1830` — String constant initialization path
4. `src/wasm/wrappers-inl.h:130-160` — The buggy type-based guard (confirmed TC-003/004)

Name the type invariant to challenge: **Shared string constants returned through non-externref typed exports must be unshared at the Wasm-to-JS boundary.** The compiled wrapper only checks for shared externref subtypes, missing shared anyref and other shared ref types.

Build a test module using WasmModuleBuilder with:
- `--experimental-wasm-shared --shared-string-table --harmony-struct`
- Shared module with string.const declarations
- Exported functions returning `(ref shared any)` that return string constants
- Warmup loop of 2000 iterations to trigger wrapper tier-up
- Assert `%IsSharedString()` is false after boundary crossing

Start with v1 angle — minimal string.const + anyref return type.
