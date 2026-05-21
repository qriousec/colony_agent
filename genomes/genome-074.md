# Genome 074 — custom-desc-alloc-type

## Hypothesis
The fix e39fada53e3 added `AnnotateWasmType` after `WasmAllocateDescriptorStruct` in turboshaft-graph-interface.cc:8926 because without it, the WasmGCTypedOptimizationReducer couldn't determine the struct's type, leading to incorrect cast elimination for an invalid cast where source and target types are unrelated. If other allocation or value-producing operations for descriptor types are ALSO missing AnnotateWasmType, the optimizer will see untyped values, potentially eliminating casts that should trap or retaining casts that could be exploited. Combined with the JSToWasmObject exact types fix (48c35511792) which added exact type support — if the exact type annotation is missing or wrong for descriptor allocations, the optimizer could incorrectly determine that a cast "always succeeds" and eliminate it, allowing a descriptor struct to be treated as a non-descriptor struct (or vice versa).

## DNA

### Source Facts — Type Annotation System
- **turboshaft-graph-interface.cc:8926**: Fix added `AnnotateWasmType(struct_value, ValueType::Ref(decoder->module_->heap_type(imm.index)))` after `WasmAllocateDescriptorStruct`
- **turboshaft-graph-interface.cc:8860**: Cast operations annotate: `AnnotateWasmType(V<Object>::Cast(object.op), config.to)` — but only AFTER the cast
- **turboshaft-graph-interface.cc:1838**: Helper `AnnotateWasmType(value, type)` wraps values with type info
- **turboshaft-graph-interface.cc:320**: SSA env initialization annotates locals
- **wasm-gc-typed-optimization-reducer.cc:274-280**: Cast elimination checks `IsHeapSubtypeOf(object_type, target) && !IsCastToCustomDescriptor(module_, check.config)` — relies on AnnotateWasmType for object_type
- **wasm-gc-typed-optimization-reducer.cc:445-450**: ProcessBranchOnTarget relies on resolved types from annotations to determine reachability

### Source Facts — Bug Fix Chain
1. **f52d915ff72**: ref.get_desc for none/bottom — exactness wrong for bottom types
2. **0a6ca69dc82**: ref.null exactness — indexed types shouldn't auto-exact
3. **48c35511792**: JSToWasmObject missing exact types + ProcessBranchOnTarget wrong reachability
4. **e39fada53e3**: TypedOptimizationReducer — missing AnnotateWasmType after descriptor struct allocation
5. **0abc1b27119**: ref.get_desc shared types — (ref shared null none) missing from exact-subtype list

### Key Observation
The annotation at line 8926 was ONLY added for the descriptor struct allocation path (`WasmAllocateDescriptorStruct`). The regular struct allocation path (`WasmAllocateStruct` at line 8929) already had type annotation through the general `StructNew` handler (line 5415). But there may be OTHER descriptor-specific paths that produce values without proper type annotation:

1. **ref.get_desc** (line 6234ff): Produces a descriptor value — does the result get AnnotateWasmType?
2. **struct.new_desc with imported descriptor types**: If the descriptor type comes from a cross-module import, does the annotation resolve to the correct canonical type?
3. **Table operations with descriptor types**: table.get returning a descriptor struct — does it get the right annotation?

### Guards
- IsCastToCustomDescriptor (wasm-gc-typed-optimization-reducer.h:21-26): Prevents cast elimination for exact descriptor casts
- GetExactness (wasm-compiler-definitions.cc:32-36): Returns kExactMatchLastSupertype for descriptor types
- AnnotateWasmType: Tags values with types for optimizer — if MISSING, optimizer has no type info

## Techniques That Work
1. Create multi-level descriptor type hierarchy (descriptor describes another descriptor)
2. Use ref.get_desc to get descriptor, then cast to unrelated type
3. Check if optimizer eliminates the cast (should NOT eliminate for unrelated types)
4. Compare Liftoff (no optimization) vs Turboshaft (with TypedOptimizationReducer)
5. Use `--trace-turbo` to inspect graph for missing AnnotateWasmType nodes

## DO NOT Try These
- IsCastToCustomDescriptor guard bypass for kExactMatchOnly casts — exhaustively tested, DEAD
- Exactness kExactMatchOnly vs kExactMatchLastSupertype semantic equivalence — DEAD
- Cross-module canonical type collision for custom descriptors — tested by cross-module-canonical, CLEAN

## Evolution Plan
- v1: Source audit — trace ALL descriptor-related value-producing operations in turboshaft-graph-interface.cc. For each, verify AnnotateWasmType is called with correct type. List any gaps. Focus on: ref.get_desc result annotation, struct.new_desc in loop body (phi node types), table.get with descriptor index type.
- v2: For any gaps found, construct a test where the optimizer sees an untyped descriptor value and attempts cast elimination. Compare Liftoff vs TurboFan. If no gaps in v1, test the interaction between ref.get_desc result type and subsequent ref.cast — does the optimizer correctly track that ref.get_desc returns a descriptor type that can't be statically compared?
- v3: Test polymorphic descriptor allocation — function with multiple struct.new_desc for different descriptor types behind a branch. After branch merge, the phi node has a widened type. Does the optimizer correctly widen? Or does it pick one branch's type, allowing the other branch's descriptor to pass a cast meant for the first?

## Task
Start by reading these files:
1. `src/wasm/turboshaft-graph-interface.cc` — search for ALL `struct.new_desc`, `ref.get_desc`, `ref.cast_desc_eq` handlers. For each, verify AnnotateWasmType is present with correct type.
2. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc:250-350` — Cast elimination logic, how it uses resolved types from annotations
3. `src/compiler/turboshaft/wasm-gc-typed-optimization-reducer.cc:440-460` — ProcessBranchOnTarget, reachability from static types
4. `src/wasm/function-body-decoder-impl.h:6234-6310` — ref.get_desc and ref.cast_desc_eq validation

Name the type invariant to challenge: **Every descriptor-typed value produced in the Turboshaft graph must have an AnnotateWasmType with the correct descriptor type, so the WasmGCTypedOptimizationReducer can make sound optimization decisions.** A missing or incorrect annotation could cause the optimizer to eliminate a cast that should trap, allowing type confusion.

Start with v1 — systematic audit of all descriptor value-producing operations.
