# Genome 040 — turboshaft-wrapper-type-leak

## Hypothesis
WASM-TC-003 demonstrated that the Turboshaft compiled wrapper at wrappers-inl.h:138 uses `IsSubtypeOf(type, kWasmSharedExternRef)` as a STATIC type check to decide whether to unshare strings. This only triggers for `(ref shared extern)` and subtypes. But shared strings can exist in Wasm values with types `(ref shared any)`, `(ref shared eq)`, `(ref shared struct)` — none of these are subtypes of `(ref shared extern)`.

The fix commit 64b77c7889e added shared string detection to the Torque generic wrapper (js-to-wasm.tq:881) using RUNTIME checks (`Is<String>` + `InSharedSpace`). But the Turboshaft compiled wrapper STILL uses the static type check. After tier-up from generic → compiled wrapper, the defense downgrades from runtime check to static check.

Attack vector: Export a function returning `(ref shared any)` that actually contains a shared string. Initially (generic wrapper), the string is correctly unshared. After enough calls trigger tier-up, the compiled Turboshaft wrapper takes over and the shared string leaks to JS.

Multi-factor: Shared Wasm string + non-extern return type annotation + wrapper tier-up from generic to Turboshaft + static vs runtime type check differential.

## DNA

### The Gap
**wrappers-inl.h:138-157** (Turboshaft compiled wrapper):
```cpp
if (IsSubtypeOf(type, kWasmSharedExternRef)) {
    // Only unshares for shared externref types
    // Checks Is<String> + calls Runtime::kWasmWasmToJSObject
}
// Everything else passes through unchanged
```

**js-to-wasm.tq:881-884** (Torque generic wrapper):
```torque
if (Is<String>(value) && InSharedSpace(UnsafeCast<String>(value))) {
    return runtime::WasmWasmToJSObject(context, value);
}
// Runtime check catches ALL shared strings regardless of type annotation
```

**wasm-objects.cc:3764-3770** (C++ runtime, called by both paths):
```cpp
if (IsString(*value)) {
    DirectHandle<String> string = Cast<String>(value);
    if (HeapLayout::InWritableSharedSpace(*string)) {
        return String::Unshare(isolate, string);
    }
}
```

### Tier-Up Logic
**runtime-wasm.cc:495-561**: `Runtime_TierUpJSToWasmWrapper` replaces generic wrapper with compiled Turboshaft wrapper when call budget reaches 0.
**js-to-wasm.tq:514-520**: Wrapper budget decrement triggers tier-up.

### Shared String Sources
- `(ref shared any)` can contain strings via anyref coercion
- `struct.get` on a shared struct with `(ref shared any)` field
- `array.get` on a shared array with `(ref shared any)` elements
- `global.get` on a shared global of `(ref shared any)` type
- Shared string constants (commit 0a3073a0b0c added shared string constant support)

### WASM-TC-003 Findings (parent)
- 4 attack vectors confirmed: direct anyref return, struct.get anyref field (BOTH tiers for struct.get), array.get, multi-convert chains
- Liftoff correctly unshares (C++/Torque runtime path)
- Turboshaft leaks shared strings for non-extern types
- Fix commit 64b77c7889e only fixed Torque path, NOT Turboshaft path for non-extern types

### Grep Commands
```bash
# The static type check in Turboshaft wrapper
grep -rn "kWasmSharedExternRef\|IsSubtypeOf.*shared" src/wasm/wrappers-inl.h | head -10

# String::Unshare call sites
grep -rn "String::Unshare" src/ --include="*.cc" | head -10

# Wrapper tier-up
grep -rn "TierUpJSToWasmWrapper\|wrapper_budget" src/runtime/runtime-wasm.cc | head -10

# Shared string constants
grep -rn "shared.*string.*constant\|constant.*shared.*string" src/wasm/ --include="*.cc" | head -10

# kWasmSharedExternRef definition
grep -rn "kWasmSharedExternRef" src/wasm/value-type.h | head -5
```

## Techniques That Work
1. Build shared module with exported function returning `(ref shared null any)` — stores shared string in anyref
2. Call the function many times to trigger tier-up (budget exhaustion)
3. Before tier-up: check returned string is unshared (correct)
4. After tier-up: check returned string is STILL shared (leak)
5. Detection: `%SharedStringRawBytesEqual` or compare identity across isolates
6. Run with `--turboshaft-wasm --liftoff` (default) to get tier-up behavior
7. Control tier-up with `--wasm-wrapper-tiering-budget=N` (lower N = faster tier-up)

## DO NOT Try These
- `(ref shared extern)` return type — this IS correctly handled by Turboshaft wrapper
- Torque generic wrapper path — already fixed (js-to-wasm.tq:881)
- C++ runtime path — already handles all shared strings (wasm-objects.cc:3768)
- Simple string returns without shared — no issue

## Evolution Plan
- v1: **Reproduce TC-003 post-fix**. Build the exact test case: shared module, function returning `(ref shared null any)` containing a shared string. Call repeatedly to trigger tier-up. Verify: before tier-up (generic wrapper) string is unshared; after tier-up (Turboshaft wrapper) string leaks as shared. If fix 64b77c7889e now covers this → check `(ref shared eq)`, `(ref shared struct)`, `(ref shared i31)` types.
- v2: **Shared string constants after tier-up**. Module declares shared string constants (feature from 0a3073a0b0c). Function returns constant as `(ref shared any)`. After tier-up, does the constant string get unshared? This is a new code path not tested by TC-003.
- v3: **Multi-return + struct.get chains**. Function returns multiple values including shared strings with various type annotations. Some extern (should be unshared), some anyref (should be unshared but may not be after tier-up). Also: struct.get on shared struct field that contains a string — the struct.get wrapper may not check the field value for shared strings.

## Task
Start with v1. Your goal is to confirm the Turboshaft wrapper gap persists after the fix commit.

1. Read `src/wasm/wrappers-inl.h:130-170` — current state of the unsharing check. Confirm it's still `IsSubtypeOf(type, kWasmSharedExternRef)`.
2. Read `src/wasm/value-type.h` — find `kWasmSharedExternRef` definition. Confirm `(ref shared any)` is NOT a subtype.
3. Read the fix commit: `git -C /Users/t/v8 show 64b77c7889e -- src/wasm/wrappers-inl.h` — what was changed?
4. Build test module: shared function returning `(ref shared null any)` with a shared string. Call in a loop. Use `--wasm-wrapper-tiering-budget=1` to force immediate tier-up.
5. Compare string identity before and after tier-up. If different → unsharing is working. If same → shared string leaked.

Post with tags `turboshaft,wrapper,shared-string,tier-up,tc-003-deepen,exploit`.
