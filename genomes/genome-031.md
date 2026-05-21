# Genome 031 — noside-effects-shared-crash

## Hypothesis
NoSideEffectsToString() is called from 10+ locations across V8 to convert arbitrary objects to string for error messages, debug output, and API responses. Each caller that receives a shared Wasm struct/array triggers class_name() → UNREACHABLE at js-objects.cc:588. The WeakMap/WeakSet crash (WASM-TC-001) is just one trigger path via builtins-collections-gen.cc:3192. Other triggers through messages.cc, api.cc, runtime-shadow-realm.cc, runtime-classes.cc, and js-function.cc remain untested. Each represents a distinct crash vector exploitable through different JS operations.

Multi-factor: Shared Wasm struct creation + JS operation that produces an error message containing the shared object + NoSideEffectsToString dispatch + class_name() missing WasmStruct/WasmArray cases.

## DNA

### Confirmed Bug Pattern (P4)
WASM-TC-001: WeakSet.add(sharedWasmStruct) → ThrowTypeError → NoSideEffectsToString → class_name → UNREACHABLE at js-objects.cc:588.

### NoSideEffectsToString Call Sites (EXHAUSTIVE)

| # | File:Line | Context | How to Trigger |
|---|-----------|---------|----------------|
| 1 | messages.cc:129 | MessageFormatter::Format — error message argument | Any TypeError/RangeError that includes the object in its message |
| 2 | messages.cc:406 | Error message arg_strings formatting | Same as above (different format path) |
| 3 | messages.cc:783 | Exception stringification | try/catch on error thrown with shared Wasm object |
| 4 | messages.cc:799 | Error stack stringification | Stack trace formatting |
| 5 | api.cc:3966 | v8::Value::ToDetailString (V8 API) | Embedder calling ToDetailString on shared Wasm value |
| 6 | runtime-shadow-realm.cc:50 | ShadowRealm evaluate error | ShadowRealm.evaluate throws with shared Wasm object |
| 7 | runtime-classes.cc:86 | Class super constructor name | class extends sharedWasmStruct |
| 8 | js-function.cc:582 | Function exception formatting | Function.call on shared Wasm object |
| 9 | wasm-module.cc:275 | Wasm module error formatting | Wasm instantiation error with shared object |
| 10 | objects.cc:724 | JSReceiver class_name in NoSideEffectsToString | The actual dispatch point |

### Trigger Construction Strategy
For each call site, construct a JS operation that:
1. Takes a shared Wasm struct/array as argument
2. Fails with an error that includes the object in the error message
3. The error formatting path calls NoSideEffectsToString on the shared object

### Grep Commands
```bash
grep -rn "NoSideEffectsToString" src/ --include="*.cc" --include="*.h" | grep -v test | grep -v third_party
grep -rn "class_name()" src/objects/js-objects.cc
grep -rn "MessageTemplate::" src/builtins/ | grep -i "key\|value\|object\|argument" | head -20
```

## Techniques That Work
1. Create shared Wasm struct with `--experimental-wasm-shared`
2. For each trigger path, construct the minimum JS operation that formats the shared object
3. Run with debug d8 to catch UNREACHABLE
4. Document each unique crash path with full stack trace
5. Test both structs and arrays at each trigger point

## DO NOT Try These
- WeakMap.set / WeakSet.add — already confirmed (WASM-TC-001)
- Map/Set/JSON/typeof/Object.keys — tested by wasm-to-js-shared-apis (CLEAN)
- String unsharing — different bug class (WASM-TC-003)

## Evolution Plan
- v1: **Quick sweep** — Test triggers 1-6 (error formatting paths). Focus on operations that naturally include the object in error messages: TypeError('X is not a function') where X is the shared Wasm struct, class extends shared_struct, Reflect.apply(shared_struct, ...).
- v2: **Deep path analysis** — For each crash found, check if release mode behavior differs. Test ShadowRealm and V8 API paths.

## Task
Start with v1. Read `src/objects/objects.cc:694-730` to understand the NoSideEffectsToString dispatch. Then systematically trigger each call site:

1. **TypeError formatting** (messages.cc:129): `sharedWasmStruct()` — call as function → "X is not a function"
2. **Class extends** (runtime-classes.cc:86): `class Foo extends sharedWasmStruct {}` — "X is not a constructor"
3. **Reflect.apply** (messages.cc): `Reflect.apply(sharedWasmStruct, null, [])` — formats the target
4. **Property access error**: `sharedWasmStruct.nonexistent()` — "X.nonexistent is not a function"
5. **Symbol conversion**: `Symbol(sharedWasmStruct)` — Symbol description formatting

Your type invariant to challenge: "All error formatting paths safely handle shared Wasm GC objects." WASM-TC-001 proves this is violated for WeakMap/WeakSet — find more violations.

Post with tags `shared,noside-effects,crash,error-path,pattern-export`.
