# Genome 012 — shared-type-boundary

## Hypothesis
Shared Wasm types (structs, arrays, strings with the `shared` attribute) have had multiple recent fixes at boundaries: strings not unshared at WasmToJS boundary (64b77c7889e), DCHECK failure in decoding invalid shared types (cb4665e7b41), and waitqueue for shared structs reverted TWICE (480966b4b4c). The shared bit (bit 4 in CanonicalValueType) may not be consistently checked across all boundary paths. If a shared struct reference is accepted where a non-shared struct is expected (or vice versa), the reference could be accessed without proper synchronization — or more directly, if the subtype check ignores the shared bit, a shared type could be treated as a different non-shared type.

## DNA

### Source Facts
- **Shared bit**: `IsSharedField` at bit 4 in value-type.h. Shared types are "safe to use in a shared function."
- **String unsharing** (`64b77c7889e`): "Unshare strings at the Wasm-to-JS boundary" — shared strings needed to be unshared when crossing to JS. This was missed in the WasmToJS consolidation.
- **Shared type DCHECK** (`cb4665e7b41`): "Fix DCHECK in decoding of invalid shared type" — the decoder had a DCHECK that failed for invalid shared types, meaning the validation was wrong.
- **Waitqueue reverts** (`480966b4b4c Reland^2`, `b95c94a3ccd Revert`, `3c23aca5b1a Reland`, `aaecfb4eb0b Revert`): Shared struct waitqueue was reverted TWICE, suggesting fundamental issues with shared struct handling.
- **Write barrier for shared structs** (`ca4637f8b73`): "Fix write barriers for shared struct/array" — GC write barriers were wrong for shared objects.
- **is_equal_except_index** (`value-type.h:1001-1003`): Compares all bits except index, INCLUDING the shared bit. So two types differing only in shared bit are NOT equal.
- **IsSubtypeOf**: Does subtyping check the shared bit? A shared struct type and a non-shared struct type with the same structure should NOT be subtypes of each other.

### Key Attack Vectors
1. **Shared/non-shared subtype confusion**: Create a shared struct type and a non-shared struct type with identical fields. Check if `IsSubtypeOf(shared_type, non_shared_type)` incorrectly returns true.
2. **Cross-module shared type matching**: Module A exports a shared struct. Module B imports expecting a non-shared struct. Does canonicalization distinguish them?
3. **Table operations with shared types**: table.set with a shared struct into a non-shared table, or vice versa. Does JSToWasmObject check the shared bit?
4. **String shared/unshared confusion**: Pass a shared string to a function expecting a non-shared string. After the unsharing fix, this should work, but check if the unshared string retains the original data correctly.
5. **GC shared struct concurrent access**: Two threads access a shared struct, one reading, one writing. If type confusion occurs, the read may see an inconsistent value.

### Guards Expected
- `is_equal_except_index` checks shared bit
- Canonicalization should distinguish shared from non-shared
- IsSubtypeOf should check shared bit
- JSToWasmObject should validate shared bit

## Techniques That Work
1. Read `IsSubtypeOfImpl` in `wasm-subtyping.cc` — search for "shared" handling
2. Read the string unsharing fix: `git -C /Users/t/v8 show 64b77c7889e`
3. Read the shared type DCHECK fix: `git -C /Users/t/v8 show cb4665e7b41`
4. Test cross-module type sharing with shared vs non-shared types
5. Run with `--experimental-wasm-shared` flag

## DO NOT Try These
- Testing without shared types enabled
- Testing shared numeric types (they're always considered shared)
- Single-threaded tests for concurrent access issues (won't trigger races)

## Evolution Plan
- v1: Read IsSubtypeOf for shared type handling. Read the recent fix diffs. Map all places where shared bit is checked vs not checked.
- v2: If v1 finds a gap (shared bit not checked in some subtype/equality path), construct a test with shared + non-shared types of same structure. If all paths check shared bit, test the boundary: shared struct exported to JS and imported back — does the shared attribute survive round-trip?
- v3: If v2 finds confusion, create exploit. If clean, test concurrent access patterns where shared struct is accessed by multiple agents (SharedArrayBuffer + WebAssembly.Memory).

## Task
Start by reading:
1. `src/wasm/wasm-subtyping.cc` — search for "shared" in IsSubtypeOf and related functions
2. `git -C /Users/t/v8 show 64b77c7889e` — string unsharing fix
3. `git -C /Users/t/v8 show cb4665e7b41` — shared type DCHECK fix
4. `src/wasm/canonical-types.cc` — how does canonicalization handle shared bit?
5. `src/wasm/wasm-objects.cc` — search for "shared" in JSToWasmObject

Type invariant to challenge: "Shared and non-shared types with identical structure are always distinguished by the type system across all code paths."

Run with: `/Users/t/v8/out/fuzzbuild/d8 --experimental-wasm-gc --experimental-wasm-shared`
