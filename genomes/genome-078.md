# Genome 078 — wasmfx-memory-stale

## Hypothesis
Fix 74ec0dd3233 added `instance_cache_.ReloadCachedMemory()` after `WasmFXSuspend` in Turboshaft (turboshaft-graph-interface.cc:4091) because memory can grow while a stack is suspended. The memory cache (base pointer + bounds) becomes stale after suspension, enabling OOB access if the parent grew memory during the suspension.

ReloadCachedMemory exists at 5 callsites in turboshaft-graph-interface.cc: after regular calls (1539, 8310), after effect handler dispatch (3967), after EndEffectHandlers (4026), and after WasmFXSuspend (4091). **Missing from:**

1. **ResumeThrow** (line ~4010-4020): When a continuation is resumed by throwing an exception into it, the handler code runs with potentially stale memory cache. The throw path uses different builtins than normal resume.
2. **Liftoff compiler**: The Turboshaft fix is at the graph-building level. Liftoff generates code directly — does it have corresponding memory cache reload after WasmFX operations? A Liftoff/Turboshaft tier differential would mean the bug exists in non-optimized code.
3. **JSPI suspend/resume**: JSPI (JavaScript Promise Integration) uses different stack switching builtins but the same underlying StackMemory. If JSPI suspends and another thread grows shared memory, the resumed JSPI stack could have stale bounds.

## DNA

### Source Facts — ReloadCachedMemory Callsites
- **turboshaft-graph-interface.cc:1539**: After regular Wasm calls (CallDirect etc.)
- **turboshaft-graph-interface.cc:3967**: After effect handler suspend dispatch (OnSuspend handler)
- **turboshaft-graph-interface.cc:4026**: After `EndEffectHandlers` (resume return block)
- **turboshaft-graph-interface.cc:4091**: After `WasmFXSuspend` builtin call — **THE FIX**
- **turboshaft-graph-interface.cc:8310**: After general calls (end of call emit)

### Source Facts — WasmFX Resume Paths
- **ResumeCall** (turboshaft-graph-interface.cc ~3930-3970): Handles `cont.resume`. Effect handler dispatch suspends and returns control.
- **ResumeThrow** (turboshaft-graph-interface.cc ~4010): Resumes a continuation by throwing into it. Uses `WasmFXResumeThrow` builtin.
- **EndEffectHandlers** (turboshaft-graph-interface.cc ~8305-8315): Called after all effect handlers processed. Has ReloadCachedMemory.

### Source Facts — Memory Cache Mechanism
- **InstanceCache** class (turboshaft-graph-interface.cc ~6270-6300): Caches `memory0_start` and `memory0_size` from WasmTrustedInstanceData.
- `ReloadCachedMemory()` re-reads base pointer and size from the instance data.
- Without reload, compiled code uses cached values → stale if memory grew during suspension.

### Source Facts — The Original Bug (489349562)
- Test in regress-489349562.js: suspend → parent grows memory by 100 pages → resume → memory store
- Uses `--stress-wasm-memory-moving` to force buffer relocation on grow
- Without fix: store uses stale base pointer → writes to freed/moved memory → UAF/OOB

### Attack Vectors
1. ResumeThrow: suspend → parent grows memory → parent throws into continuation → catch handler uses stale memory
2. Liftoff tier: run WasmFX code without optimization → check if Liftoff reloads memory cache
3. Nested suspension: A suspends to B, B suspends to C, C grows memory, B resumes A → A has doubly-stale cache
4. JSPI + WasmFX interaction: JSPI suspend triggers in the middle of WasmFX handler
5. Shared memory grow from another thread while continuation is suspended

## Techniques That Work
1. Create WasmFX module with memory + suspend/resume operations
2. Grow memory between suspend and resume (like regress-489349562.js pattern)
3. Use `--stress-wasm-memory-moving --no-wasm-trap-handler --wasm-enforce-bounds-checks`
4. Test with `--liftoff-only` vs `--no-liftoff` for tier differential
5. Test ResumeThrow by having the parent throw an exception that's caught by the continuation's handler
6. Use %WasmTierUpFunction or warmup loops to ensure specific compilation tier

## DO NOT Try These
- cont.bind GC roots — DEAD SURFACE
- WasmFX exception lifecycle basic — tested CLEAN by wasmfx-except-lifecycle
- Stack pool management — tested CLEAN

## Evolution Plan
- v1: Source audit — trace ResumeThrow path in turboshaft-graph-interface.cc. Verify if ReloadCachedMemory is called after the throw reaches the handler. Also check Liftoff compiler (liftoff-compiler.cc) for WasmFX memory cache handling. Create test: suspend → grow 100 pages → ResumeThrow → memory access in catch handler.
- v2: Test Liftoff vs Turboshaft differential. Run the same WasmFX suspend→grow→resume test with --liftoff-only and --no-liftoff. If Liftoff doesn't reload cache but Turboshaft does (or vice versa), there's a tier differential.
- v3: Test nested suspension (A→B→C chain). Grow memory at C, resume B, then resume A. Check if A's memory cache is fresh. Also test shared memory grow from a Worker thread while continuation is suspended.

## Task
Start by reading these files in order:
1. `src/wasm/turboshaft-graph-interface.cc` — search for ALL `ReloadCachedMemory` calls, map what WasmFX operation each corresponds to
2. `src/wasm/turboshaft-graph-interface.cc` — search for `ResumeThrow`, trace the full path and verify ReloadCachedMemory presence
3. `src/wasm/baseline/liftoff-compiler.cc` — search for WasmFX suspend/resume operations, check for memory cache reload
4. `test/mjsunit/regress/wasm/regress-489349562.js` — understand the test pattern

Name the type invariant to challenge: **After any WasmFX context switch (suspend, resume, resume-throw), the memory cache (base pointer + bounds) must be reloaded because memory may have grown while the stack was inactive.** The fix covers one specific path (after WasmFXSuspend in Turboshaft). ResumeThrow, Liftoff, and JSPI paths may be missing the reload.

Start with v1 — audit ResumeThrow path and Liftoff WasmFX handlers.
