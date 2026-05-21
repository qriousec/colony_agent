# V8 Wasm Type-Confusion Colony

A multi-agent fuzzing system that hunts WebAssembly type-confusion bugs in V8.
Instead of running a single fuzzer loop, it runs a **colony** of LLM-driven
agents: one *queen* directs a population of *workers*, each carrying a
*genome* (a hypothesis about a specific V8 seam where Wasm type soundness
might break). The colony is reactive — workers run continuously, the queen
wakes on a cycle to score them, evolve their genomes, and reallocate budget.

The current production colony is `wasm-tc`. It targets the seams where Wasm
type guarantees cross V8 subsystems: canonicalization, the Turboshaft GC
optimizer, JS↔Wasm boundaries, speculative inlining and deopt, and
cross-module type sharing.

## What it actually does

Each worker runs the loop **BUILD → TEST → OBSERVE → TROUBLESHOOT → IMPROVE**
against a real `d8`:

1. **BUILD** a Wasm module that pushes on one specific type invariant
   (e.g. 20-bit `HeapTypeField` overflow, nullability-ignoring
   `EqualValueType`, loop-header type narrowing in the GC optimizer).
2. **TEST** it as a tier differential — Liftoff vs TurboFan, or jitless vs
   tiered, or non-speculative vs speculative-inlined. Any output divergence
   is a bug candidate.
3. **OBSERVE** the Turboshaft graph: which `WasmTypeCast` / `WasmTypeCheck`
   / `AssertNotNull` / `TrapIf` nodes were eliminated, by which phase.
4. **TROUBLESHOOT** if clean — name the defense that caught it, read the
   source at that check, look for what it does *not* cover.
5. **IMPROVE** — mutate the genome (`narrow`, `intensify`, `lateral`,
   `recombine`, `source-pivot`, `factor-add`, `pattern-export`, `random`).

The differential output across compilation tiers is the **oracle**: the spec
says types are sound, so any tier-to-tier disagreement on a well-typed
module is a soundness violation somewhere in V8.

## Architecture

```
                     ┌──────────────────────┐
                     │   colony_cli.py      │  start / status / stop
                     └──────────┬───────────┘
                                │
                  ┌─────────────▼──────────────┐
                  │   QueenCycle  (colonylib)  │
                  │   • assess fitness         │
                  │   • read social feed       │
                  │   • SPAWN / KILL / GRAFT   │
                  │   • BUG_ARCHIVE            │
                  └────┬───────────────┬───────┘
                       │ writes genome │ tmux spawn
                       ▼               ▼
            ┌──────────────────┐   ┌──────────────────┐
            │ genomes/NNN.md   │   │ Worker  (Agent)  │  ×N
            │ hypothesis+DNA   │──▶│ runs in tmux pane│
            └──────────────────┘   └────┬─────────────┘
                                        │ d8 + tools/
                                        ▼
                              ┌────────────────────────┐
                              │ tools/wasm_module.py   │ build modules
                              │ tools/v8_test.py       │ run differentials
                              │ tools/wasm_graph.py    │ dump Turboshaft IR
                              │ tools/wasm_types.py    │ type-system probes
                              └────────────────────────┘
                                        │
                                        ▼
                              ┌────────────────────────┐
                              │ socialnet (posts/)     │ findings feed
                              │ → fitness → queen      │
                              └────────────────────────┘
```

### The pieces

| Component | Files | What it does |
|-----------|-------|--------------|
| **CLI / event loop** | `colony_cli.py` | Boots the colony, waits `cycle_interval` between queen wakes, polls a `worker_done_event`. |
| **Queen cycle** | `colonylib/queen.py` | Renders the queen prompt, parses its JSON decision (`actions: [SPAWN, KILL, GRAFT, BUG_ARCHIVE]`), executes it. |
| **Worker runner** | `colonylib/worker.py`, `colonylib/tmux.py` | Spawns each worker as an `Agent` in its own tmux pane, runs the worker template, tracks iterations. |
| **Context / state** | `colonylib/context.py`, `colonylib/state.py` | Mutex-guarded shared state, persisted to `colony/<id>/colony_state.json`. |
| **Genomes** | `colonylib/genome.py`, `colony/<id>/genomes/genome-NNN.md` | Markdown files: hypothesis, accumulated DNA, techniques that work, failed approaches, evolution plan. |
| **Fitness** | `colonylib/fitness.py` | `10·differentials + 3·source_analyses + 3·primitives + social_score`. Drives kills, hall-of-fame, mutation choice. |
| **Guardrails** | `colonylib/guardrails.py` | Enforces min cycles, min iterations before kill, alive fraction floor, anti-monoculture (caps fraction of identical mutation type), pivots after consecutive zero-differential cycles. |
| **Social network** | `socialnet.py`, `social_cli.py`, `colonylib/social_context.py` | Reddit-style feed (`colony-workers` channel). Workers post `differential`, `source_analysis`, `primitive` findings; queen reads them as the digest. |
| **Prompts** | `prompts/v8/queen_wasm_tc.md`, `prompts/v8/worker_base_wasm_tc.md` | The actual instructions sent to Claude. The queen prompt carries the **Tier-1/2/3 attack-surface inventory** built from past CVEs. |
| **Tools** | `tools/wasm_module.py`, `tools/v8_test.py`, `tools/wasm_graph.py`, `tools/wasm_types.py` | The worker's hands: build modules, run d8 differentials, dump Turboshaft IR, probe canonicalization. |

### Colony roles

Defined in `colonylib/config.py:COLONY_ROLES`. The Wasm hunter is:

```python
"wasm-tc": {
    "queen_prompt":     "queen_wasm_tc.md",
    "worker_template":  "worker_base_wasm_tc.md",
    "default_budget":   {"max_iterations": 10, "max_wall_time_sec": 10800},
    "description":      "Wasm Type Confusion — hunt type canonicalization
                         overflow, nullability hash collision, compiler type
                         elimination, JS-Wasm boundary confusion, and
                         cross-module type sharing bugs",
}
```

Sibling roles (`maglev`, `maglev-cases`, `general`) reuse the same colony
machinery against other V8 surfaces.

## Why this finds Wasm bugs in V8

Three architectural choices do the work:

### 1. The oracle is differential, not crash-based

Most Wasm fuzzers wait for a crash. The colony does too, but its primary
signal is much earlier: **any output disagreement between Liftoff and
TurboFan on a well-typed Wasm module is a bug**, because the spec requires
both tiers to compute the same value. This catches silent type-confusion
miscompilations that would never SIGSEGV. The presets in `tools/v8_test.py`
codify the relevant differentials:

```
--preset wasm-tc       Liftoff vs TurboFan       (primary oracle)
--preset wasm-interop  jitless vs Liftoff vs TF  (JS↔Wasm boundary)
--preset wasm-deopt    Liftoff vs speculative    (call_indirect inlining)
--preset wasm-gc       null-check elim variants  (GC optimizer)
```

### 2. Search is steered by V8 source, not random bytes

The queen prompt ships a curated **attack-surface inventory** keyed to real
V8 file paths and past CVEs (`prompts/v8/queen_wasm_tc.md` §"Attack Surface
Inventory"):

| Tier | Example surface | Files | CVEs |
|------|----------------|-------|------|
| W1 | Canonical type-ID overflow (20-bit `HeapTypeField`) | `canonical-types.cc`, `value-type.h` | CVE-2024-2887, -6100, -8194 |
| W2 | Nullability ignored in `EqualValueType` hash | `canonical-types.cc` | CVE-2025-5959 |
| W3 | Type-check elimination at loop merge | `wasm-gc-typed-optimization-reducer.h` | related to 2024-2887 |
| W4 | JS↔Wasm boundary (`FromJS`) | `wasm-compiler.cc`, `wasm-objects.cc` | CVE-2024-4761 |
| W5 | Wasm-in-JS inlining without GC type opts | `wasm-in-js-inlining-reducer.h` | — |
| W6 | Speculative `call_indirect` inlining + deopt | TurboFan feedback / Wasm deopt | M137+ |
| W7 | Loop-header type narrowing | `wasm-gc-typed-optimization-reducer.h` | — |
| W8 | Cross-module type sharing | `canonical-types.cc` | — |
| W9–W15 | field offsets, null sentinels, signature subtyping, GVN, DCE, BranchElim, LoadElim | various reducers | — |

Workers don't probe randomly: they read V8 source in `{V8_ROOT}/src/wasm/`
and `src/compiler/turboshaft/`, locate the type invariant a piece of code
*assumes*, find a check that *doesn't* enforce it, and craft a module that
exploits the gap. The worker prompt makes this explicit:

> *"Wasm type confusion bugs live at seams between V8 subsystems. A type
> that validates correctly can confuse the canonicalizer. A canonicalized
> type can be truncated passing to the compiler. An optimization pass can
> eliminate a check that was actually needed."*

### 3. Bad ideas die, good ideas evolve

A single agent searching the same space repeats itself. The colony doesn't:

- **Fitness selects** — the queen kills workers whose budget runs out with
  zero differentials *and* zero source-analysis posts. The killed worker's
  hall-of-fame entry stays as a "dead surface" warning for descendants.
- **Mutations recombine** — when two workers find adjacent signals (e.g.
  one shows canonicalization quirk, another shows optimizer elimination),
  a `recombine` mutation spawns a child carrying both DNA strands.
- **Guardrails prevent monoculture** — `colonylib/guardrails.py` caps the
  fraction of workers using the same mutation type and force-pivots the
  colony after `consecutive_zero_diff_pivot` empty cycles. This keeps the
  search from collapsing onto one surface.
- **Pattern export** — once a real bug pattern is confirmed, a
  `pattern-export` mutation grep-walks the bug shape across sibling
  subsystems to find variants.

The current `colony/wasm-tc/colony_state.json` shows 134 genomes generated
across ~dozens of distinct hypotheses (e.g. `inlining-trycatch-deopt` —
"other builtins called during JS-to-Wasm wrapper inlining may not propagate
frame state correctly"; `exnref-tag-type-mismatch` — "cross-module tag
canonicalization could decode exception payloads with wrong types"). Each
report under `fuzzer/` is a sealed investigation — the system produces both
confirmed bugs (`BUG_ARCHIVE` actions) and reasoned negative results
(`*_FINAL_REPORT.md` documenting what was tried, what defended, and what
the next pivot should be).

## Quickstart

Requires Python 3.10+, a tmux build, a built `d8`, a V8 source checkout,
and an `agent` runtime configured for whichever model you use (see
`agent.py`).

```bash
# Start the Wasm type-confusion colony with 4 workers against a local d8
python3 colony_cli.py start \
  --colony-role wasm-tc \
  --colony-id   wasm-tc \
  --target      v8 \
  --d8-binary   ~/v8/out/fuzzbuild/d8 \
  --max-workers 4

# Live status (alive workers, fitness, last activity)
python3 colony_cli.py status --colony-id wasm-tc

# Read the social feed (what workers are posting)
python3 social_cli.py feed --channel colony-workers --sort new --limit 20

# Stop everything
python3 colony_cli.py stop --colony-id wasm-tc
```

The colony writes:

| Path | Contents |
|------|----------|
| `colony/wasm-tc/colony_state.json` | Live state: workers, fitness, hall of fame, confirmed bugs. |
| `colony/wasm-tc/genomes/genome-NNN.md` | Per-worker hypothesis + DNA. |
| `colony/wasm-tc/poc/`, `repro/` | Minimal Wasm/JS reproductions for confirmed bugs. |
| `decisions/queen-wasm-tc.json` | Last queen decision (JSON). |
| `social/`, `colony/wasm-tc/posts/` | Social-network posts. |
| `fuzzer/*.md` | Long-form per-investigation reports. |

## Layout

```
v8_wasm_agent/
├── colony_cli.py          # entry point
├── agent.py               # LLM Agent wrapper (tmux-driven)
├── socialnet.py           # social network backend
├── colonylib/             # queen, worker, fitness, guardrails, state
├── prompts/v8/            # queen + worker prompt templates
├── tools/                 # wasm_module / v8_test / wasm_graph / wasm_types
├── colony/<id>/           # per-colony state, genomes, repros, posts
├── fuzzer/                # per-investigation reports + generated cases
└── memory/, decisions/    # historical context, queen decision files
```

## Extending it

- **New target surface** — add a row to the attack-surface table in
  `prompts/v8/queen_wasm_tc.md` and add a recipe to `tools/wasm_module.py`
  (the named generators: `overflow`, `nullability`, `hierarchy`,
  `elimination`, `boundary`, `cross-module`).
- **New oracle** — add a preset to `tools/v8_test.py` for a new differential
  (e.g. WasmFX continuations, shared types, memory64).
- **New role** — add an entry to `COLONY_ROLES` in `colonylib/config.py`
  and write `prompts/v8/queen_<role>.md` + `worker_base_<role>.md`. The
  rest of the machinery is role-agnostic.