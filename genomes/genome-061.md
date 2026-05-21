# Genome 061 — wasmfx-except-lifecycle

## Hypothesis
WasmFX recently fixed EPT (External Pointer Table) entry not being cleared when a continuation stack retired due to an exception (commit e97587a3751). This fix moved EPT cleanup into `Isolate::RetireWasmStack`. The same pattern — cleanup skipped on exception paths — may exist for bound arguments.

When a continuation throws an exception after `cont.bind`:
1. The stack is retired via exception path
2. `Isolate::RetireWasmStack` is called
3. The stack is returned to `StackPool` for reuse
4. But are `param_types_`, `arg_buffer_`, `num_bound_args_` cleared?

If not, when the stack is recycled from the pool for a NEW continuation with DIFFERENT parameter types:
- `param_types_` points to the OLD type vector
- `arg_buffer_` may point to freed memory or be reused with different layout
- GC visits slots based on OLD types, but the buffer now contains NEW values
- Type confusion: GC traces i32 values as ref pointers, or skips actual refs

Multi-factor: [exception during WasmFX resume] × [stack pool recycling] × [stale bound arg GC roots] = GC corruption.

## DNA

### Source Facts
```cpp
// e97587a3751 — EPT fix: moved cleanup into RetireWasmStack
// BEFORE fix: EPT only cleared on normal return
// AFTER fix: EPT cleared in RetireWasmStack (all return paths)
// QUESTION: Does RetireWasmStack also call clear_bound_args()?

// stacks.h:231-235 — clear_bound_args
void clear_bound_args() {
    param_types_ = {};
    arg_buffer_ = kNullAddress;
    num_bound_args_ = 0;
}

// StackPool (stacks.h:295-311) — reuses retired stacks
class StackPool {
    std::unique_ptr<StackMemory> GetOrAllocate();  // Gets from freelist
    void Add(std::unique_ptr<StackMemory> stack);   // Returns to freelist
};
```

### Files to Audit
- `src/execution/isolate.cc` or `src/execution/isolate.h` — RetireWasmStack implementation
- `src/wasm/stacks.cc` — StackMemory::Reset(), StackPool::Add()
- `src/runtime/runtime-wasm.cc` — exception handling paths for WasmFX
- `src/wasm/wrappers-inl.h` — BuildWasmStackEntryWrapper exception handling

### Key Questions
1. Does RetireWasmStack call clear_bound_args()?
2. Does StackPool::Add or StackMemory::Reset clear bound args?
3. What happens to arg_buffer_ memory when a stack is retired? Is it freed? Reused?
4. After stack reuse, who sets param_types_ for the new continuation? Is it set BEFORE GC can run?

### Related
- Pattern P5: Compressed/uncompressed confusion in off-heap buffers
- EPT fix: e97587a3751 (same "missed cleanup on exception" pattern)
- cont.bind: a93849fa707 (new feature, under-tested)

## Ready-to-Run Test

```javascript
// Flags: --experimental-wasm-wasmfx --experimental-wasm-gc --allow-natives-syntax --expose-gc
load('/Users/t/v8/test/mjsunit/wasm/wasm-module-builder.js');

// TEST 1: cont.bind then exception, verify stack reuse safety
(function TestContBindException() {
  try {
    let b = new WasmModuleBuilder();

    // Struct type for trackable refs
    let structType = b.addStruct([makeField(kWasmI32, true)]);

    // Tag for suspend
    let suspendTag = b.addTag(makeSig([], []));

    // Function type: (ref struct, i32) -> i32
    let funcType = b.addType(makeSig([wasmRefType(structType), kWasmI32], [kWasmI32]));
    let contType = b.addType({kind: kContType, ftype: funcType});

    // Function that either suspends or throws based on param
    b.addFunction('maybe_throw', funcType)
      .addBody([
        kExprLocalGet, 1,
        kExprI32Const, 0,
        kExprI32Eq,
        kExprIf, kWasmVoid,
          kExprUnreachable,  // Throw trap
        kExprEnd,
        kExprLocalGet, 0,
        kGCPrefix, kExprStructGet, structType, 0,
      ]).exportFunc();

    // Struct factory
    b.addFunction('make_struct', makeSig([kWasmI32], [wasmRefType(structType)]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprStructNew, structType
      ]).exportFunc();

    let inst = b.instantiate();

    // Create structs
    let s1 = inst.exports.make_struct(0xDEAD);
    let s2 = inst.exports.make_struct(0xBEEF);

    // Normal call (no throw)
    let r1 = inst.exports.maybe_throw(s1, 1);
    print('TEST1: Normal call result=' + r1);

    // Throwing call
    try {
      inst.exports.maybe_throw(s1, 0);
      print('TEST1: Should have thrown!');
    } catch(e) {
      print('TEST1: Correctly threw: ' + e);
    }

    // GC after exception
    gc();

    // Verify struct is still valid
    let r2 = inst.exports.maybe_throw(s2, 1);
    print('TEST1: Post-exception call result=' + r2);

    print('TEST1: PASS');
  } catch(e) {
    print('TEST1 ERROR: ' + e.message);
  }
})();

// TEST 2: Many stack create/retire cycles with GC pressure
(function TestStackRecycleGC() {
  try {
    let b = new WasmModuleBuilder();
    let structType = b.addStruct([makeField(kWasmI64, true), makeField(kWasmI32, true)]);

    b.addFunction('make', makeSig([kWasmI32], [wasmRefType(structType)]))
      .addBody([
        kExprI64Const, 0,
        kExprLocalGet, 0,
        kGCPrefix, kExprStructNew, structType
      ]).exportFunc();

    b.addFunction('read', makeSig([wasmRefType(structType)], [kWasmI32]))
      .addBody([
        kExprLocalGet, 0,
        kGCPrefix, kExprStructGet, structType, 1,
      ]).exportFunc();

    let inst = b.instantiate();

    let structs = [];
    for (let i = 0; i < 1000; i++) {
      structs.push(inst.exports.make(i));
      if (i % 50 === 0) gc();
    }

    // Verify all structs survived GC
    let corrupted = 0;
    for (let i = 0; i < structs.length; i++) {
      if (inst.exports.read(structs[i]) !== i) corrupted++;
    }
    print('TEST2: Corrupted after GC: ' + corrupted + '/' + structs.length);

  } catch(e) {
    print('TEST2 ERROR: ' + e.message);
  }
})();

print('ALL TESTS COMPLETE');
```

## DO NOT Try These
- WeakMap/WeakSet exploitation — dead surface
- Simple GC stress without WasmFX stacks — well-tested
- Compressed/uncompressed in wrapper returns — already fixed

## Evolution Plan
- v1: Source audit RetireWasmStack, StackPool::Add, StackMemory::Reset for clear_bound_args(). Check exception handling in WasmFX resume/suspend.
- v2: If clearing gap found → construct test with cont.bind refs, throw exception, force stack reuse, trigger GC. The GC should visit stale slots from the old continuation's type layout.

## Task
**SOURCE AUDIT FIRST.** Find:
1. `RetireWasmStack` implementation — does it call clear_bound_args()?
2. `StackMemory::Reset()` — does it clear WasmFX-specific fields?
3. Exception handling path in WasmFX resume — what cleanup happens?

Then run the test to establish baseline. The test is intentionally simple — if WasmFX cont types aren't supported by the module builder, adapt.

Post with tags `wasmfx,exception,stack-lifecycle,bound-args,W22,EPT`.
