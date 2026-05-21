# Genome 076 — wasm-deser-memory

## Hypothesis
Three recent fixes in Wasm memory deserialization (value-serializer.cc) reveal weak validation in the ValueDeserializer path:

1. **966e72ab572**: Added maximum_pages validation — before the fix, deserialized WasmMemory could have out-of-range max pages causing Smi DCHECK
2. **66e649c24c6**: Completely rewrote cycle-breaking logic, removed `kResizableNotFollowedByWasmMemory` tag, changed allocation order to: empty WasmMemoryObject → AB → link. Before the fix, multiple ABs pointing to the same WasmMemoryObject had broken symbol links.
3. **7f4edfc4050**: Fixed GSAB byte_length=0 invariant — GC could observe GSAB with non-zero byte_length during deserialization window

The new code (66e649c24c6) creates a WasmMemoryObject WITHOUT a buffer first, registers its ID, then allocates the AB. During this window, the WasmMemoryObject is in an inconsistent state (no buffer, no backing store). If GC or concurrent code accesses it during this window, it observes an incomplete object. Additionally, the removal of the `kResizableNotFollowedByWasmMemory` tag means old serialized data with that tag byte ('w' vs 'r') may be misparsed, causing field confusion.

## DNA

### Source Facts — Deserialization Path
- **value-serializer.cc:2464**: `ReadWasmMemory()` — entry point for WasmMemory deserialization
- **value-serializer.cc:2464-2495**: Reads maximum_pages, memory64_byte, validates max (post-fix)
- **value-serializer.cc:2498+**: Creates empty WasmMemoryObject FIRST, then reads AB
- **WasmMemoryArrayBufferTag**: `kFixedLength='f'`, `kResizable='r'` (old: also `kResizableNotFollowedByWasmMemory='r'`, `kResizableFollowedByWasmMemory='w'`)

### Source Facts — WasmMemoryObject Invariants
- **wasm-objects.cc:891-896**: `WasmMemoryObject::New` has DCHECKs for max pages within spec range
- **wasm-objects.cc:1015-1030**: `SetNewBuffer` freezes shared buffers and sets byte_length=0 for GSABs
- **wasm-objects.cc:1040-1055**: `FixUpResizableArrayBuffer` sets max_byte_length — DCHECK byte_length==0 for shared

### Source Facts — What's NOT Validated
- The deserialization path validates `maximum_pages` (post-fix) and `memory64_byte` (>1 check)
- Does NOT validate: initial pages, buffer size consistency with max pages, backing store alignment
- The empty WasmMemoryObject window: between ID registration and AB linking, any GC-triggered code that iterates objects may encounter an incomplete WasmMemoryObject

### Recent Fixes
- 966e72ab572: max pages validation (bug 490970052)
- 66e649c24c6: AB cycle deserialization rewrite (bugs 489029655, 487444465)
- 7f4edfc4050: GSAB byte_length during deser (bug 482759504)
- 52927e23d8d: toFixedLengthBuffer byte_length for fixed-length shared (bug 483643012)
- 527e2021856: GSAB byte_length ordering during SetNewBuffer (bug 491575818)

### Attack Vectors
1. Craft serialized data with old 'w' tag (kResizableFollowedByWasmMemory) — how does new deserializer handle it?
2. Trigger GC during the empty WasmMemoryObject window — create an object that needs finalization, then deserialize
3. Create serialized WasmMemory with shared=true + resizable + very large max_pages at spec boundary
4. Deserialize WasmMemory and immediately call memory.grow before SetNewBuffer completes

## Techniques That Work
1. Use `d8.serializer.serialize/deserialize` for creating serialized blobs
2. Craft raw byte arrays (like regress-490970052.js) for invalid serialized data
3. Test with shared + resizable memory configurations
4. Trigger GC via `gc()` or large allocations between deserialization steps
5. Test cross-version serialized data compatibility

## DO NOT Try These
- Normal Wasm module creation (doesn't go through deserialization)
- Testing memory growth in isolation (well-tested standard path)

## Evolution Plan
- v1: Map the complete deserialization path for WasmMemory in value-serializer.cc. Identify all fields read and their validation (or lack thereof). Create tests with: (a) old-format serialized data with 'w' tag, (b) boundary values for max_pages (just at/over kSpecMaxMemory32Pages), (c) memory64 flag combined with 32-bit max pages.
- v2: Test the GC-during-deserialization window. Create a test that: (a) has a weak reference to an object that triggers finalization on GC, (b) deserializes a WasmMemory (which creates empty object first), (c) forces GC during the AB allocation step. Check if the empty WasmMemoryObject causes crashes during GC iteration.
- v3: Test interaction between deserialized WasmMemory and Wasm module import. Create a module that imports a memory, deserialize a crafted WasmMemoryObject, and pass it as the import. Check if the module trusts the deserialized max_pages/address_type without re-validation.

## Task
Start by reading:
1. `src/objects/value-serializer.cc` — search for `ReadWasmMemory`, read the full function
2. `src/objects/value-serializer.cc` — search for `ReadJSArrayBuffer`, read the Wasm-specific branches
3. `src/wasm/wasm-objects.cc:891-920` — WasmMemoryObject::New
4. `src/wasm/wasm-objects.cc:1010-1060` — SetNewBuffer and FixUpResizableArrayBuffer
5. `test/mjsunit/regress/wasm/regress-490970052.js` — example of crafted deserialized data

Name the type invariant to challenge: **Deserialized WasmMemoryObjects must satisfy the same invariants as module-created ones: valid max_pages, consistent buffer size, proper byte_length state for GSABs.** The deserialization path may create objects that violate invariants normally enforced by the module decoder.

Start with v1 — full deserialization path audit + boundary value testing.
