# Genome 118 — type-guard-audit-p4

## Hypothesis
TC-003/004/005 root cause: `IsSubtypeOf(type, kWasmSharedExternRef)` at wrappers-inl.h:138 was a TYPE-based guard making runtime decisions based on STATIC type information, when it should have been a VALUE-based guard (runtime `Is<String>() + InSharedSpace()` check). This pattern — trusting static types to determine runtime behavior — is a systematic vulnerability class. If other code paths in V8's Wasm implementation use `IsSubtypeOf`, `is_reference_to`, or similar type-based checks to skip or modify runtime operations, AND the static type can be incomplete or overly broad, those paths have the same TC-003-class vulnerability.

## DNA

### Source Facts (file:line)
- **TC-003 root cause**: wrappers-inl.h:138 — `IsSubtypeOf(type, kWasmSharedExternRef)` — TYPE-based guard
- **Correct approach**: js-to-wasm.tq:881 — `Is<String>(value) && InSharedSpace(...)` — VALUE-based guard
- **Correct approach**: wasm-objects.cc:3762 — `IsString(*value) && HeapLayout::InWritableSharedSpace(...)` — VALUE-based
- **Pattern**: Static type analysis → skip/modify runtime operation → wrong behavior when static type is incomplete

### Systematic Grep Targets
1. `IsSubtypeOf` in non-validation contexts (wrappers, lowering, runtime):
   - wrappers-inl.h — compiled wrapper generation
   - wasm-lowering-reducer.h — Turboshaft lowering
   - wasm-gc-lowering.cc — legacy TurboFan lowering
   - turboshaft-graph-interface.cc — graph building
2. `is_reference_to` used as guards for runtime behavior:
   - wasm-lowering-reducer.h — extern/func/i31 checks in cast/check lowering
3. Type-conditional code in wrappers:
   - ToJS(), FromJS() in wrappers-inl.h — type-driven conversion
   - js-to-wasm.tq — Torque wrappers
4. `type.is_nullable()` / `type.is_shared()` / `type.is_exact()` used to skip checks:
   - Any place where nullable/shared/exact status skips a runtime check

### Key Question For Each Site
For every `IsSubtypeOf` / `is_reference_to` / type flag check found:
1. Is the check in VALIDATION (safe) or in CODEGEN/RUNTIME (potentially vulnerable)?
2. Does the check skip a runtime operation that should depend on the VALUE, not the TYPE?
3. What happens if the static type is a supertype that doesn't carry the property? E.g., (ref any) doesn't imply string, but the value could be a string.
4. Is there a tier differential? Does Liftoff/Turboshaft/interpreter handle the same site differently?

## Techniques That Work
1. Systematic grep for `IsSubtypeOf` in src/wasm/wrappers-inl.h, src/compiler/turboshaft/wasm-lowering-reducer.h, src/wasm/wasm-objects.cc
2. For each match: classify as validation-context vs codegen-context
3. For codegen-context matches: determine if a VALUE-based check would be more correct
4. Build test for most promising gap: create a value of supertype that carries a property the type check doesn't expect
5. Focus on shared types: the shared bit is newest and most likely to have incomplete type-based guards

## DO NOT Try These
- Re-testing wrappers-inl.h:138 (already confirmed as TC-003)
- Testing validation-time IsSubtypeOf calls (these are correct by definition)
- Testing type checks in the optimizer (those are optimization guards, not runtime guards)

## Evolution Plan
- v1: AUDIT phase. Grep for all IsSubtypeOf and is_reference_to calls in wrappers-inl.h, wasm-lowering-reducer.h, wasm-gc-lowering.cc, turboshaft-graph-interface.cc. Classify each as validation-context (safe) vs codegen-context (potential target). Catalog results with file:line.
- v2: For each codegen-context match: analyze what runtime decision is being made. Identify the 3 most promising gaps where the TYPE-based check could miss a VALUE-based property. Build targeted test for the #1 gap.
- v3: If v2 finds a gap: build PoC and minimize. If v2 is clean: extend audit to js-to-wasm.tq, wasm-to-js.tq, and runtime-wasm.cc. Check for type-conditional paths in Torque/C++ that trust type annotations.

## Task
1. Run systematic grep:
   - `grep -rn 'IsSubtypeOf' src/wasm/wrappers-inl.h` — find all type-based guards in compiled wrappers
   - `grep -rn 'IsSubtypeOf\|is_reference_to' src/compiler/turboshaft/wasm-lowering-reducer.h` — find type-based guards in lowering
   - `grep -rn 'IsSubtypeOf' src/wasm/wasm-objects.cc` — find type-based guards in runtime
   - `grep -rn 'type\.is_shared\|type\.is_nullable\|type\.is_exact' src/wasm/wrappers-inl.h` — find flag-based guards
2. For each match, read surrounding context to classify validation vs codegen.
3. Focus on codegen matches where TYPE doesn't fully determine VALUE properties.
4. Build test for most promising gap.
5. Run with: `--experimental-wasm-shared --no-wasm-lazy-compilation`
6. Start with Evolution Plan v1 (systematic audit).
