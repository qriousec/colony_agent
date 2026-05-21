# Genome 060 — wasmfx-contbind-gcstress

## Hypothesis
WasmFX `cont.bind` stores bound ref arguments in `StackMemory::arg_buffer_`. The GC visits these via `IterateWasmFXArgBuffer` in `stacks.cc:120-127`, using `param_types_[index].is_ref()` to decide which slots to trace as `FullObjectSlot`. Three attack angles:

1. **Stack pool reuse without clearing**: When a stack is retired and returned to `StackPool`, does `StackMemory::Reset()` call `clear_bound_args()`? If not, a reused stack from the pool retains stale `param_types_` and `arg_buffer_`, and the GC visitor traces freed/reused memory.

2. **Partially-bound GC race**: After `cont.bind` stores some args but before `resume`, a GC cycle runs. The `num_bound_args_` was incremented by `Runtime_WasmAllocateBoundContinuation` (runtime-wasm.cc:2855) BEFORE the new continuation object is allocated (which may trigger GC). But the bound args were stored by the compiler-generated code BEFORE the runtime call. So the sequence is: store args → runtime: bind_arguments() → GC may run during allocation → GC sees incremented count and visits the correct slots. This looks safe. BUT: what if the arg_buffer pointer itself is stale?

3. **param_types_ lifetime**: `param_types_` is a `base::Vector<const CanonicalValueType>` whose memory is "owned by the type canonicalizer" (stacks.h:271). The canonicalizer's types are never freed per current design, but if a module with shared WasmFX types is instantiated and GC'd, does the canonical type entry survive?

Multi-factor: [cont.bind bound ref args in off-heap buffer] × [stack pool reuse lifecycle] × [GC visiting correctness of arg_buffer] = GC corruption.

## DNA

### Source Facts
```cpp
// stacks.h:227-234 — bind_arguments increments count with DCHECK-only overflow check
void bind_arguments(int count) {
    num_bound_args_ += count;
    DCHECK_LE(num_bound_args_, param_types_.size());
}

// stacks.h:231-235 — clear_bound_args resets everything
void clear_bound_args() {
    param_types_ = {};
    arg_buffer_ = kNullAddress;
    num_bound_args_ = 0;
}

// stacks.cc:120-127 — GC visits bound ref args
IterateWasmFXArgBuffer(param_types_, [this, v](size_t index, int offset) {
    if (static_cast<int>(index) < num_bound_args_ &&
        param_types_[index].is_ref()) {
        v->VisitRootPointer(Root::kStackRoots, "wasm cont ref bound argument",
                            FullObjectSlot(reinterpret_cast<Address>(
                                this->arg_buffer_ + offset)));
    }
});

// runtime-wasm.cc:2845-2860 — Runtime_WasmAllocateBoundContinuation
// Order: bind_arguments() first, THEN allocate new cont object
// The allocation may trigger GC, which visits the just-bound args

// stacks.h:295-311 — StackPool: freelist of reusable stacks
// When a stack is returned to pool, is clear_bound_args called?

// IterateWasmFXArgBuffer iterates right-to-left:
// param[N-1] at lowest offset, param[0] at highest offset
```

### Files to Audit
- `src/wasm/stacks.cc` — StackMemory::Reset(), StackMemory::IterateRootsIfActive()
- `src/wasm/stacks.h` — StackPool::Add(), clear_bound_args()
- `src/runtime/runtime-wasm.cc` — Runtime_WasmAllocateBoundContinuation, return paths
- `src/wasm/turboshaft-graph-interface.cc` — ContBind (3865-3895), Resume (3924-3947)
- `src/wasm/wrappers-inl.h` — BuildWasmStackEntryWrapper

### Key Questions
1. Does StackMemory::Reset() call clear_bound_args()? If not, stale GC roots survive pool recycling.
2. What happens to arg_buffer when a stack is suspended, then GC'd, then resumed?
3. After cont.bind, the old continuation is replaced by a new one. Is the old StackMemory properly cleaned up, or does it linger with bound refs?

### Related Commits
- 5a64071bf9d: Fixed compressed refs in WasmFX return buffer
- 2f8c6526740: Fixed compressed refs in WasmFX stack entry params
- e97587a3751: Fixed EPT entry not cleared on exception paths
- a93849fa707: Implemented cont.bind

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-wasmfx --experimental-wasm-gc --experimental-wasm-shared --allow-natives-syntax --expose-gc
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

// TEST 1: Basic cont.bind with ref args, then GC stress
(function TestContBindGCStress() {
  try {
    let b = new WasmModuleBuilder();

    // Struct type to create trackable ref objects
    let structType = b.addStruct([makeField(kWasmI32, true)]);

    // Tag for suspend/resume
    let tag = b.addTag(makeSig([kWasmI32], []));

    // Continuation type: takes (ref struct, i32) -> void
    let funcType = b.addType(makeSig([wasmRefType(structType), kWasmI32], []));
    let contType = b.addType({kind: kContType, ftype: funcType});

    // Function to be wrapped in continuation
    b.addFunction('target', funcType)
      .addBody([
        // Read struct field to verify ref is valid
        kExprLocalGet, 0,
        kGCPrefix, kExprStructGet, structType, 0,
        kExprDrop,
        // Suspend with the i32 param
        kExprLocalGet, 1,
        kExprThrow, tag,
      ]).exportFunc();

    // Create a struct
    b.addFunction('make_struct', makeSig([kWasmI32], [wasmRefType(structType)]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprStructNew, structType
      ]).exportFunc();

    let inst = b.instantiate();
    print('TEST1: Module instantiated');

    // Create struct objects
    let s1 = inst.exports.make_struct(42);
    let s2 = inst.exports.make_struct(99);
    print('TEST1: Structs created');

    // Trigger GC to move objects
    gc();
    print('TEST1: GC survived');

  } catch(e) {
    print('TEST1 ERROR: ' + e.message);
    print('TEST1 STACK: ' + e.stack);
  }
})();

// TEST 2: Stack pool reuse — create and retire many stacks
(function TestStackPoolReuse() {
  try {
    let b = new WasmModuleBuilder();
    let structType = b.addStruct([makeField(kWasmI32, true)]);
    let tag = b.addTag(makeSig([], []));

    // Simple suspendable function
    let funcType = b.addType(makeSig([wasmRefType(structType)], []));
    let contType = b.addType({kind: kContType, ftype: funcType});

    b.addFunction('suspender', funcType)
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprStructGet, structType, 0,
        kExprDrop,
      ]).exportFunc();

    b.addFunction('make_struct', makeSig([kWasmI32], [wasmRefType(structType)]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprStructNew, structType
      ]).exportFunc();

    let inst = b.instantiate();

    // Create many structs and let them be GC'd
    for (let i = 0; i < 100; i++) {
      let s = inst.exports.make_struct(i);
      // Force GC every 10 iterations
      if (i % 10 === 0) gc();
    }
    print('TEST2: 100 struct creations with GC stress survived');

  } catch(e) {
    print('TEST2 ERROR: ' + e.message);
  }
})();

print('ALL TESTS COMPLETE');
```

## DO NOT Try These
- Simple struct creation/access without WasmFX — that's well-tested
- WeakMap/WeakSet with shared objects — 4-layer defense confirmed dead
- String sharing checks — already covered by TC-003/TC-006

## Evolution Plan
- v1: Source audit StackMemory::Reset() and StackPool::Add() for clear_bound_args() call. Run basic cont.bind test with refs. Check if WasmFX cont types are supported in module builder.
- v2: If Reset() doesn't clear bound args → construct stack reuse test: create continuation, bind refs, resume/complete, create new continuation on recycled stack, trigger GC. If Reset() does clear → test partially-bound state during GC (bind args, force GC before resume).
- v3: Test with shared struct refs across stacks. Check if shared objects in arg_buffer cause cross-space GC issues.

## Task
**START WITH SOURCE AUDIT.** Read:
1. `src/wasm/stacks.cc` — find `Reset()` method, check if `clear_bound_args()` is called
2. `src/runtime/runtime-wasm.cc` — `Runtime_WasmAllocateBoundContinuation` and `Runtime_WasmResumeContinuation`
3. `src/wasm/turboshaft-graph-interface.cc` — ContBind, Resume implementation

Then construct tests. The key question is whether StackMemory::Reset() properly clears bound arg GC roots when a stack is recycled.

Post with tags `wasmfx,cont-bind,gc-roots,stack-pool,W19,W21`.
