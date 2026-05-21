# Genome 034 — shared-wasm-gc-stress

## Hypothesis
WASM-TC-002 proved the GC marking visitor (marking-visitor-inl.h:704) DCHECK-fails when encountering shared-space Wasm objects as ephemeron keys. 20+ other GC code paths have similar `DCHECK(!HeapLayout::InWritableSharedSpace(...))` assumptions. When shared Wasm structs/arrays are stored in non-shared heap containers (regular JS Maps, closures, arrays, object properties), GC operations beyond ephemeron marking encounter these shared-space objects and violate their assumptions:

1. **Sweeper** (sweeper.cc:573): DCHECK(!InWritableSharedSpace(host)) — sweeping a non-shared page that references shared Wasm objects
2. **Marking barrier** (marking-barrier.cc:110/131): DCHECK(!InWritableSharedSpace(host)) — write barrier for storing shared Wasm objects in non-shared containers
3. **Write barrier** (heap-write-barrier.cc:117/251): DCHECK(!InWritableSharedSpace(host)) — generational write barrier when non-shared host stores shared value
4. **Descriptor marking** (marking-visitor-inl.h:883): DCHECK(!InWritableSharedSpace(descriptors_as_heap_object)) — marking descriptors of objects in shared space

In release builds, violated DCHECK assumptions may cause: missing GC roots (shared objects collected while still referenced), incorrect remembered sets (shared objects not tracked across spaces), or corrupted marking state.

Multi-factor: Shared Wasm struct creation + storage in non-shared container + GC trigger (minor/major/concurrent) + shared-space assumption violation in GC visitor.

## DNA

### Confirmed Bug Pattern (P8)
WASM-TC-002: `WeakSet.add(sharedWasmStruct)` → GC → marking-visitor-inl.h:704 `DCHECK(!HeapLayout::InWritableSharedSpace(key))` fails because WasmStruct IS a JSReceiver and IS in writable shared space.

### GC DCHECK Sites (EXHAUSTIVE from grep)

| # | File | Line | DCHECK | Context |
|---|------|------|--------|---------|
| 1 | marking-inl.h | 318 | DCHECK(chunk->InWritableSharedSpace()) | Marking shared space chunk |
| 2 | sweeper.cc | 573 | DCHECK(!InWritableSharedSpace(host)) | Sweeping page with host object |
| 3 | marking-barrier.cc | 71 | CHECK(InWritableSharedSpace(host)) | Shared barrier activation |
| 4 | marking-barrier.cc | 110 | DCHECK(!InWritableSharedSpace(host)) | Non-shared marking barrier |
| 5 | marking-barrier.cc | 131 | DCHECK(!InWritableSharedSpace(host)) | Non-shared marking barrier (write) |
| 6 | marking-visitor-inl.h | 704 | DCHECK(!InWritableSharedSpace(key)) | Ephemeron key marking ← **WASM-TC-002** |
| 7 | marking-visitor-inl.h | 883 | DCHECK(!InWritableSharedSpace(descriptors)) | Descriptor marking |
| 8 | marking-visitor-inl.h | 962 | DCHECK_EQ(use_shared_table, InWritableSharedSpace(host)) | Shared table marking |
| 9 | heap-write-barrier.cc | 117 | DCHECK(!InWritableSharedSpace(host)) | Write barrier fast path |
| 10 | heap-write-barrier.cc | 251 | DCHECK(!InWritableSharedSpace(host)) | Write barrier |
| 11 | heap-write-barrier.cc | 127 | if (!InWritableSharedSpace(host)) — logic branch | Shared-to-non-shared write |
| 12 | marking-barrier-inl.h | 71 | DCHECK_IMPLIES(InWritableSharedSpace(host),...) | Barrier consistency |
| 13 | marking-barrier-inl.h | 73 | DCHECK_IMPLIES(InWritableSharedSpace(value),...) | Value consistency |

### Key Insight
WASM-TC-002 was found via WeakSet (ephemeron path). But shared Wasm objects can enter ANY of these GC paths through different JS operations:
- Storing a shared Wasm struct as a property of a regular JS object → write barrier
- Storing in a regular (non-weak) Map → marking during major GC
- Storing in a closure that gets collected → sweeper
- Storing as descriptor of a type → descriptor marking

### Grep Commands
```bash
grep -rn "DCHECK.*!.*InWritableSharedSpace\|DCHECK.*!.*InSharedSpace" src/heap/
grep -rn "InWritableSharedSpace\|InSharedSpace" src/heap/ --include="*.cc" --include="*.h" | wc -l
grep -rn "WasmStruct\|WasmArray\|WASM_STRUCT_TYPE\|WASM_ARRAY_TYPE" src/heap/
```

## Techniques That Work
1. Create shared Wasm module with shared struct/array types
2. Store shared Wasm objects in various non-shared containers:
   - `let obj = {}; obj.x = sharedStruct;` (regular property)
   - `let arr = [sharedStruct];` (regular array)
   - `let m = new Map(); m.set('key', sharedStruct);` (regular Map value)
   - `let closure = () => sharedStruct;` (captured in closure)
   - `globalThis.shared = sharedStruct;` (global property)
3. Force different GC types:
   - `%CollectGarbage('minor')` or `--gc-interval=100` (scavenge)
   - `%CollectGarbage('major')` or `gc()` (full GC)
   - `--expose-gc --gc-global` + allocation pressure (concurrent marking)
   - `--stress-marking` + `--stress-scavenge` (stress modes)
4. Run with debug build to catch DCHECKs
5. Run with release build + `--verify-heap` to check for GC corruption
6. Compare: object survives GC correctly? Fields accessible after GC?

## DO NOT Try These
- WeakSet/WeakMap with shared objects — already confirmed (WASM-TC-002)
- String unsharing — different bug class (WASM-TC-003)
- class_name crashes — different bug class (WASM-TC-001)

## Evolution Plan
- v1: **Write barrier trigger**. Store shared Wasm struct in regular JS object property. Force minor GC (scavenge). Check if write barrier DCHECK at heap-write-barrier.cc:117 or 251 fires. This is the simplest trigger: any property write to a non-shared object with a shared value should invoke the write barrier.
- v2: **Marking barrier trigger**. Create many shared Wasm objects stored in non-shared containers. Force major GC with concurrent marking. Check marking-barrier.cc:110/131 DCHECKs. Test with `--stress-marking`.
- v3: **Release mode GC corruption**. If DCHECKs fire in debug, test release mode: does the shared object survive GC? Are fields still correct after GC? Does the GC miss a root (use-after-free)? Test: create shared struct with recognizable field values → force GC → read fields → verify values match.

## Task
Start with v1. Read these files IN THIS ORDER:

1. `src/heap/heap-write-barrier.cc` around line 117 and 251 — understand the write barrier fast path. What triggers `DCHECK(!InWritableSharedSpace(host))`? What code executes AFTER the DCHECK in release mode if the assumption is wrong?
2. `src/heap/marking-barrier.cc` around line 110 and 131 — the marking barrier. What happens when a non-shared host stores a shared value? Does the shared value get added to the correct remembered set?
3. `src/heap/marking-visitor-inl.h` around line 704 (WASM-TC-002 site) and line 883 (descriptor site) — understand the exact conditions that trigger the DCHECK.
4. Build test: `let obj = {}; obj.sharedProp = sharedWasmStruct; gc();` — simple property write + GC.

Your type invariant to challenge: "GC correctly handles shared-space Wasm objects stored in non-shared containers." WASM-TC-002 proved this is violated for ephemeron keys. Find violations in write barriers, marking barriers, and sweepers.

Post with tags `shared,gc,write-barrier,marking,stress,pattern-export`.
