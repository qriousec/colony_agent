# Genome 038 — canonical-recgroup-desc

## Hypothesis
The canonical type system (canonical-types.cc) merges structurally equivalent type groups into a single canonical ID. For recursive type groups, type indices within the group are made relative using `MakeGroupRelative()`. Two recursive type groups that have the same relative structure get the same canonical ID — meaning values of one group's types can be used interchangeably with the other. This is correct for truly equivalent types, but custom descriptors introduce a relationship (descriptor/describes) that could create false equivalences:

1. **Descriptor pointing within vs outside rec group**: Type group {A, B} where A describes B (both within group) has descriptor=relative(B). Another group {C, D} where C describes D has descriptor=relative(D). If A and C are at the same position within their groups, their relative descriptor indices match, and the groups are canonically merged. But A's described type B has different fields from D — the descriptor relationship is semantically different.

2. **Descriptor chain length confusion**: Type group {Desc, Struct} where Desc describes Struct. Type group {Desc2, Struct2} where Desc2 describes Struct2 but Desc2 also HAS a descriptor (nested). The `EqualType` check at canonical-types.h:364 checks `EqualTypeIndex(type1.descriptor, type2.descriptor)` — if Desc has descriptor=kNoType and Desc2 has descriptor=some_index, they SHOULD differ. But what if the index comparison has an edge case with kNoType vs a valid index that happens to map the same way through MakeGroupRelative?

3. **Exactness in field types across groups**: Two groups with identical struct fields except one field has `(ref exact $T)` and the other has `(ref $T)`. The canonical equality at `EqualStructType` (line 402) compares field types via `CanonicalValueType` operator== which compares bit_field_ directly (line 990-991). Exactness IS in the bit field. But if $T is a group-relative index, the comparison uses `is_equal_except_index` + separate index comparison. Does the exactness survive the separate handling?

4. **Shared vs non-shared in descriptor targets**: CanonicalEquality.EqualType checks `is_shared` at line 357. But does it check shared-ness of the DESCRIPTOR TARGET type? If type A describes shared type B, and type C describes non-shared type D, but A and C match on all other fields, will EqualType correctly distinguish them?

Multi-factor: Recursive type group + descriptor/describes relationship + canonical type merging + MakeGroupRelative index transformation + field type exactness.

## DNA

### Source Facts

**FindCanonicalGroup** (canonical-types.cc:458-475):
```cpp
CanonicalTypeIndex TypeCanonicalizer::FindCanonicalGroup(
    const CanonicalGroup& group) const {
  auto it = canonical_groups_.find(group);
  return it == canonical_groups_.end() ? CanonicalTypeIndex::Invalid()
                                       : it->first;
}
```
Uses hash-based lookup with CanonicalGroup::operator== for equality.

**CanonicalGroup::operator==** (canonical-types.h:435-441):
```cpp
bool operator==(const CanonicalGroup& other) const {
  CanonicalEquality equality{{first, last}, {other.first, other_last}};
  return equality.EqualTypes(types, other.types);
}
```
Delegates to CanonicalEquality which handles group-relative indices.

**CanonicalEquality.EqualType** (canonical-types.h:353-370):
```cpp
if (!EqualTypeIndex(type1.supertype, type2.supertype)) return false;
if (type1.is_final != type2.is_final) return false;
if (type1.is_shared != type2.is_shared) return false;
...
case kStruct:
  return EqualTypeIndex(type1.descriptor, type2.descriptor) &&
         EqualTypeIndex(type1.describes, type2.describes) &&
         EqualStructType(*type1.struct_type, *type2.struct_type);
```
Checks: supertype, is_final, is_shared, descriptor, describes, and struct fields.

**MakeGroupRelative** (canonical-types.h:236-239):
```cpp
uint32_t MakeGroupRelative(CanonicalTypeIndex index) {
  uint32_t is_relative = recgroup.Contains(index) ? 1 : 0;
  uint32_t relative = index.index - is_relative * recgroup.first.index;
  return (relative << 1) | is_relative;
}
```
Encodes whether an index is within the group (relative) or outside (absolute).

**EqualTypeIndex** (canonical-types.h:338-349):
```cpp
bool EqualTypeIndex(CanonicalTypeIndex index1, CanonicalTypeIndex index2) {
  // Both must be relative or both absolute.
  // If relative, their offsets must match.
  // If absolute, their canonical indices must match.
}
```

### Key Vulnerability Window
The vulnerability is in the INTERACTION between:
1. MakeGroupRelative encoding descriptor/describes indices
2. EqualTypeIndex comparing them
3. The semantic meaning of descriptors being DIFFERENT even when relative positions match

If two recursive groups have descriptors at the same relative position but the described types have DIFFERENT FIELDS, EqualStructType on the described types SHOULD catch the difference. But if the described type is ALSO at a position where its fields match (despite having different descriptor chains), the entire group could be falsely merged.

### Grep Commands
```bash
# MakeGroupRelative
grep -rn "MakeGroupRelative" src/wasm/canonical-types.h

# EqualTypeIndex implementation
grep -rn "EqualTypeIndex" src/wasm/canonical-types.h

# How descriptor/describes are set during canonicalization
grep -rn "descriptor\|describes" src/wasm/canonical-types.cc | head -20

# kNoType handling
grep -rn "kNoType" src/wasm/canonical-types.h | head -10

# AddRecursiveGroup — how groups are added
grep -rn "AddRecursiveGroup\|AddRecursiveSingletonGroup" src/wasm/canonical-types.cc | head -10
```

## Techniques That Work
1. Build two modules with recursive type groups:
   - Module 1: rec group {$Desc1 describes $Struct1, $Struct1 with fields [i32, i32]}
   - Module 2: rec group {$Desc2 describes $Struct2, $Struct2 with fields [i32, i32]}
   - Same structure, same fields — should get same canonical ID (CORRECT merge)
2. Then vary: give $Struct2 an EXTRA field, or different mutability, or different descriptor chain
3. Check: do the types still merge? Export from module 1, import in module 2
4. Use struct.get on a field index that exists in one but not the other
5. Test with `--experimental-wasm-custom-descriptors`
6. Compare debug (DCHECKs) vs release (potential type confusion)

## DO NOT Try These
- Simple struct subtyping without descriptors — well-tested
- Custom descriptor optimizer guards — covered (8 iters, CLEAN)
- Shared + descriptor types — UNIMPLEMENTED guard
- Non-recursive type groups — simpler, better tested

## Evolution Plan
- v1: **Source audit of canonicalization with descriptors**. Read canonical-types.cc:AddRecursiveGroup and the full EqualType/EqualTypeIndex/MakeGroupRelative chain. Map exactly how descriptors flow through canonicalization. Identify: what conditions would cause two semantically different type groups to get the same canonical ID?
- v2: **Build collision candidates**. Based on v1, construct two recursive type groups that are structurally identical except for descriptor chain semantics. Export a value from group 1, import/use it as group 2's type. If canonical IDs match incorrectly, struct.get on a field that differs between the "same" type produces wrong value.
- v3: **Exactness in field types across groups**. Two groups where struct fields differ only in exactness of a reference type. Verify EqualStructType catches the exactness difference. If not, a non-exact ref is treated as exact, bypassing subtype checks.

## Task
Start with v1. Read these files IN THIS ORDER:

1. `src/wasm/canonical-types.cc` — Find `AddRecursiveGroup` (the main entry point for canonicalization). Trace how a recursive group with descriptor types flows through: (a) hash computation, (b) FindCanonicalGroup equality check, (c) storage of the canonical group.
2. `src/wasm/canonical-types.h:338-370` — Read the FULL EqualTypeIndex and EqualType implementations. For descriptor/describes indices: what happens when one is kNoType and the other is a valid index? What happens when both are group-relative but at different positions?
3. `src/wasm/canonical-types.h:236-239` — MakeGroupRelative. What happens with kNoType (invalid index)? Does `recgroup.Contains(kNoType)` return false? If so, kNoType stays absolute — is it then compared correctly?
4. Build test: Two modules with identical recursive groups except Module 2 has a descriptor where Module 1 doesn't. Export from Module 1, import in Module 2 — does the import succeed (canonical match) when it shouldn't?

Your type invariant to challenge: "Two recursive type groups receive the same canonical ID if and only if they are truly structurally equivalent, including descriptor relationships." The rapid descriptor bug fixes prove the descriptor integration into canonicalization was done under pressure — find the corner case.

Post with tags `canonical,recursive-types,descriptor,type-collision,source-pivot`.
