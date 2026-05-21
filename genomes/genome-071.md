# Genome 071 — tojs-consolidation-diff

## Hypothesis
The WasmToJS/JSToWasm consolidation (27a83f0b039, reverted then relanded) dramatically simplified the Torque `WasmToJSObject` macro. The TC-003 fix (64b77c7889e) was applied AFTER the consolidation and added value-based shared string checks to both the Torque and C++ paths. But the compiled Turboshaft wrapper (wrappers-inl.h) still uses a TYPE-BASED check.

Multi-factor: [three different wrapper paths with different check logic] × [TC-003 fix applied unevenly] × [edge case value types (i31ref, multi-return, funcref wrapping)] = differential behavior between Torque/C++/compiled paths that reveals a boundary gap.

Specific angles:
1. **i31ref representation**: Does the Torque path handle i31ref (Smi) differently than the compiled path?
2. **Multi-return with mixed types**: When a Wasm function returns multiple values (e.g., externref + i32 + funcref), does the Torque path process each return value correctly?
3. **Funcref wrapping edge case**: The consolidation changed funcref handling from `Cast<WasmFuncRef>() otherwise unreachable` to `Is<WasmFuncRef>()` + `UnsafeCast`. Are there objects that pass `Is<WasmFuncRef>` but aren't actually WasmFuncRef?

## DNA

### Source Facts
```
# Consolidation commit (27a83f0b039) — simplified WasmToJSObject:

# OLD Torque WasmToJSObject (pre-consolidation):
# - Takes (context, value, retType: uint32)
# - Branches on refKind: struct/array pass through, function wraps, generic types pass through
# - Unknown types: runtime fallback
# - Total: ~40 lines of type-specific logic

# NEW Torque WasmToJSObject (post-consolidation):
# - Takes (context, value) — NO TYPE PARAMETER
# - WasmNull → null
# - WasmFuncRef → wrap or runtime call
# - Everything else → UnsafeCast<JSAny>(value)
# - TC-003 fix added: Is<String>(value) && InSharedSpace → runtime call
# - Total: ~15 lines

# NEW C++ WasmToJSObject (wasm-objects.cc:3750-3770):
# - WasmNull → null
# - WasmFuncRef → wrap (fast path: try_get_external, slow: GetOrCreateExternal)
# - TC-003 fix: IsString && InWritableSharedSpace → Unshare
# - Everything else → pass through

# COMPILED Turboshaft wrapper (wrappers-inl.h ToJS):
# - Funcref: elaborate wrapping with external function caching
# - Nullable types: WasmNull → JS null
# - Shared externref: type-based unshare check (TC-003 partial fix)
# - Everything else: pass through
# NOTE: Still uses TYPE-BASED check, not VALUE-BASED
```

### Three-Way Differential
| Value Type | Torque Path | C++ Runtime | Compiled Wrapper |
|-----------|-------------|-------------|-----------------|
| WasmNull | → JS null | → JS null | → JS null |
| WasmFuncRef | Is + UnsafeCast → wrap | Cast → wrap | Tagged equality + wrap |
| Shared string (externref) | VALUE check → runtime | VALUE check → Unshare | TYPE check → unshare IFF shared externref |
| Shared string (anyref) | VALUE check → runtime | VALUE check → Unshare | NO CHECK → leaks |
| i31ref | Pass through (Smi) | Pass through | Pass through |
| Shared struct | Pass through | Pass through | Pass through |
| Non-shared externref with shared string value | VALUE check → runtime | VALUE check → Unshare | NO CHECK → leaks |

### Key Files
- `src/builtins/js-to-wasm.tq:864-885` — WasmToJSObject (post-consolidation + TC-003 fix)
- `src/wasm/wasm-objects.cc:3750-3770` — C++ WasmToJSObject
- `src/wasm/wrappers-inl.h:88-175` — Compiled wrapper ToJS
- `src/builtins/js-to-wasm.tq:940,1053` — Call sites of WasmToJSObject

## Techniques That Work
- Call same Wasm function via generic wrapper (pre-tier-up) and compiled wrapper (post-tier-up)
- Compare return values for each value type
- Focus on the differential between Torque and compiled paths

## DO NOT Try These
- Testing shared string leak for externref (already confirmed as TC-003)
- Testing basic null/funcref handling (well-tested)
- Testing non-ref types (i32, f64 — no conversion needed)

## Evolution Plan
- v1: Source audit the three wrapper paths for each value type. Construct test matrix covering edge cases: i31ref Smi, shared struct through non-shared externref, multi-return with mixed types. Run before and after tier-up.
- v2: If v1 finds differential, minimize and identify file:line. If v1 is clean, test interaction between consolidation and TC-003 fix: what happens when a funcref check returns false for a shared string (Is<WasmFuncRef> returns false, falls through to shared string check — correct. But what if the order is wrong?).

## Task
First verify the current state of all three paths:

```bash
# Torque path (post-consolidation + TC-003 fix)
sed -n '860,890p' /Users/t/v8/src/builtins/js-to-wasm.tq

# C++ path
grep -A 25 "DirectHandle<Object> WasmToJSObject" /Users/t/v8/src/wasm/wasm-objects.cc

# Compiled path
sed -n '85,180p' /Users/t/v8/src/wasm/wrappers-inl.h
```

Then construct a test with:
1. A shared struct with a shared string field (anyref type)
2. A function that returns this field
3. Call before tier-up (Torque path) and after tier-up (compiled path)
4. Compare: does the compiled path leak the shared string while the Torque path unshares it?

This is the TC-003 differential but specifically testing the consolidation's interaction with the fix.

Post with tags `wasm-to-js,consolidation,boundary,tc-003,differential,W26,export`.
