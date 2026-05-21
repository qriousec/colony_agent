# Genome 069 — load-elim-shared-struct

## Hypothesis
The WasmLoadElimination reducer (`wasm-load-elimination-reducer.h`) caches struct field loads by `{base, offset, type_index, size, mutability}`. It does NOT track whether a struct is shared or non-shared. For shared mutable struct fields, another isolate/thread can modify the field between a cached load and its reuse.

Multi-factor: [load elimination of shared struct.get] × [concurrent field modification by another thread] × [stale cached value has different runtime type] = type confusion when the cached value (e.g., a shared string) is used where the modified value (e.g., an i31ref) is expected.

Combined with TC-003 pattern: if load elimination caches a shared struct field that contains a non-shared-string value, but another thread stores a shared string there, the cached non-string value is returned. Conversely, if the cache holds a shared string but the field was updated to a different type, the string might bypass unshare checks when it shouldn't.

## DNA

### Source Facts
```cpp
// WasmMemoryAddress — the key for cached loads (wasm-load-elimination-reducer.h:52-64)
struct WasmMemoryAddress {
  OpIndex base;           // The struct object
  int32_t offset;         // Field offset
  ModuleTypeIndex type_index;  // Struct type index
  uint8_t size;           // Field size
  bool mutability;        // Mutable or immutable
  // NOTE: NO shared/non-shared distinction!
};

// ProcessStructGet — caches the load (line 612)
case Opcode::kStructGet:
  ProcessStructGet(op_idx, op.Cast<StructGetOp>());
  break;

// ProcessStructSet — invalidates cached values (line 615)
case Opcode::kStructSet:
  ProcessStructSet(op_idx, op.Cast<StructSetOp>());
  break;

// ProcessCall — invalidates state (line 651)
case Opcode::kCall:
  ProcessCall(op_idx, op.Cast<CallOp>());
  break;

// WasmFXArgBuffer — IGNORED, no invalidation (line 682)
case Opcode::kWasmFXArgBuffer:
  break;

// kAtomicRMW — break, no invalidation for general cached loads (line 672)
case Opcode::kAtomicRMW:
  break;

// KEY QUESTION: Does StructAtomicRMW invalidate cached non-atomic loads?
case Opcode::kStructAtomicRMW:
  ProcessAtomicRMW(op_idx, op.Cast<StructAtomicRMWOp>());
  break;
```

### WasmFXArgBuffer commit (bae619a077d)
Added kWasmFXArgBuffer to the "safe to ignore" list because "its side effects are only for ordering purposes." But this means load elimination caches persist across WasmFX effect handler setup.

### Shared struct semantics
Shared structs can be accessed from multiple isolates. The Wasm spec requires atomic access for shared mutable fields. But the V8 optimizer may not enforce this:
- Non-atomic struct.get on a shared struct → the load elimination may cache this
- Another thread does atomic struct.set on the same field
- The cached value is stale

### Load elimination and calls
Calls invalidate load elimination state (ProcessCall). But what about:
- Calls to WasmFX builtins (suspend/resume)?
- Calls to waitqueue primitives (wait/notify)?
- Calls to GC builtins?

If any of these don't properly go through the kCall path, cached state persists.

## Techniques That Work
- Construct a shared struct with a mutable anyref field
- Thread A: read field (struct.get) in a loop with tier-up, check that load elimination reuses the cached value
- Thread B: concurrently modify the field to a different value/type
- Check if Thread A ever sees a stale cached value after tier-up

## DO NOT Try These
- Load elimination with non-shared structs (well-tested, single-threaded cache is correct)
- Immutable field caching (correct by construction — immutable fields never change)
- Raw memory loads (not handled by WasmLoadElimination)

## Evolution Plan
- v1: Source audit ProcessStructGet, ProcessStructSet, ProcessAtomicRMW. Check if shared struct accesses are handled differently. Check if atomic struct.get invalidates cached non-atomic loads. Read ProcessCall to understand what exactly gets invalidated.
- v2: If v1 finds no shared-struct-specific handling, construct test with shared struct + concurrent modification + load elimination enabled (--turboshaft-wasm-load-elimination). Verify with --no-liftoff to force Turboshaft compilation.

## Task
Start with source audit:

```bash
# Read ProcessStructGet implementation
grep -n "ProcessStructGet\|ProcessStructSet\|ProcessAtomicRMW" /Users/t/v8/src/compiler/turboshaft/wasm-load-elimination-reducer.h

# Check if shared structs have special handling
grep -n "shared\|is_shared\|InSharedSpace" /Users/t/v8/src/compiler/turboshaft/wasm-load-elimination-reducer.h

# Check ProcessCall — what gets invalidated
grep -n "ProcessCall\|InvalidateAll" /Users/t/v8/src/compiler/turboshaft/wasm-load-elimination-reducer.h
```

Key question: Does the load elimination distinguish between atomic and non-atomic struct accesses? If a shared struct field should only be accessed atomically, but the optimizer caches a non-atomic access, the stale cache is the bug.

Post with tags `load-elimination,shared-struct,cache,atomic,W15,exploit`.
