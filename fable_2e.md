# The Colony as a Self-Improving Agentic System

*A thesis on population-based, evolutionary, memory-externalizing multi-agent architectures for open-ended discovery*


## Abstract

This system is an existence proof for a specific claim: **durable self-improvement in an agentic system is a property of the ecology around the agent, not of the agent itself.** The colony takes a fixed, non-learning language model and makes the *system* improve over time by relocating the locus of improvement out of the model's weights and into an external architecture — heritable memory, explicit selection, structured variation, a cheap and honest evaluator, and active diversity maintenance. No agent gets smarter; the model's weights never change. What improves is the externalized, selection-curated body of knowledge the agents build, inherit, and refine.

The second edition keeps that claim and hardens the machinery that carries it. Heritable memory now travels **with its certificates and its epistemic mode**: facts are witnesses a child can spot-check, techniques are investments that can be liquidated, and negative knowledge is split into *coverage* (sound, reopenable) and *refutation* (complete, permanent) — because the first edition's exhausted-region records committed, and made heritable, the oldest error in bounded search: reading "not found within the slice" as "not found." Selection is honest **all the way up**: the controller's promoted patterns pass an independent verification gate before broadcast, and nothing enters fitness that did not pass, or was not consumed by something that passed, the evaluator. Variation operators arrive **typed**, each carrying the obligation its child must discharge and the failure signature its controller audits. The controller itself acquires a decision theory — bandit allocation, priced probes, scheduled restarts, and a population-level bracket that drives phase changes instead of a script. And the architecture finally admits what its evaluator cannot do: a verifier defines *what counts*, not *what was wanted*, so the colony grows a symptom panel, an escalation channel, and a frame-owner outside the loop — the difference between a system that runs unsupervised and one that runs unowned.

---

## 1. Thesis statement

A single agent looping on a hard, open-ended problem plateaus. It rediscovers its own dead ends, forgets what it learned when its context fills, and converges on whatever local strategy first showed promise. The colony's central argument is that you can defeat all three failure modes without touching the model, by surrounding it with an architecture that provides what the model lacks:

> **Improvement requires — jointly — externalized heritable memory, explicit selection on a cheap honest evaluator, structured variation, and enforced diversity, coupled across nested timescales.**

I sometimes write this multiplicatively (`memory × selection × variation × diversity`) as a mnemonic: *any factor at zero zeroes the product.* But the mnemonic should not be mistaken for algebra — there are no units, and the factors are explicitly **not orthogonal**. "Selection" is only as honest as the evaluator it runs on (§3.5), so selection and the evaluator interlock rather than multiply independently. The defensible claim is weaker and more precise than a formula: **conjunctive necessity.** Each ingredient is individually necessary, and removing any one degrades the system to a *named* failure mode — memory without selection is hoarding; selection without variation is premature convergence; variation without an honest evaluator is drift; all three without diversity maintenance is monoculture.

What the second edition adds to the thesis is a discovery about its own factors: **they are not free inventions.** Each is a named, load-bearing component with an epistemic job. The genome is a structured memory object, split into witnesses and lemmas; the evaluator is an evidence structure, checkability clause and all; the variation operators are a catalogue of moves — previously imported without their obligations, which is where several first-edition bugs lived; diversity maintenance is a search-preservation discipline; and the queen is a scheduling tier, implemented as an agent. The colony's founding bet — that a static model plus superb *control* beats a clever model with none — is the bet the empirical literature on expert problem-solving has already graded: control is what separates experts from novices. The colony wagered the whole architecture on that finding. The second edition makes the wager principled instead of lucky.

**Where the gains actually come from.** It is worth separating the sources of improvement rather than laundering them through the word "emergence." Most of the gain is ordinary **cumulative memory**: a genome that accumulates verified facts and ruled-out approaches is simply a lineage that does not forget, and a lineage that does not forget beats one that restarts. On top of that base, **selection** adds *curation* (weak lines are cut, strong lines get more budget), **variation** adds *coverage* (the space gets explored), and **diversity maintenance** adds *robustness* (the search does not collapse early). So "the population improves, not the agent" is precise about exactly one thing — no weights change — and should not be read as mysticism. The improving object is the externalized, selection-curated knowledge base; the single largest contributor to it is plain accumulation; and selection, variation, and diversity are what keep that accumulation honest, broad, and unstuck. The second edition adds the missing coda: accumulation is only as good as the *labels* on what accumulates. A fact inherited without its certificate is a rumor; a dead end recorded without its coverage is a wall built on noise. Most of this edition is about the labels.

---

## 2. The problem: why single-agent loops plateau

Three structural limits bound a lone agent searching a vast solution space — and a fourth, which the first-edition colony did not solve but *inherited*:

1. **Ephemeral memory.** An agent's working knowledge lives in a context window that is finite and gets summarized or discarded. Nothing it learns in hour one is guaranteed to survive to hour ten, and *nothing survives its own termination.*
2. **No selection pressure.** A lone agent has no mechanism to declare a line of inquiry bankrupt and reallocate effort. It re-enters dead ends because the cost of a past failure is not carried forward as a constraint.
3. **Convergence.** Given one promising signal, an agent digs there. Breadth decays. The search collapses onto a single strategy long before the space is covered.
4. **Frozen framing.** The agent optimizes whatever objective it was handed, forever. It has no channel through which to notice — let alone report — that verified answers are being rejected, that every model misfits, that the target keeps being amended after results arrive. A lone agent cannot ask whether it is solving the right problem; neither could the first-edition colony, which scaled limits 1–3 away and left limit 4 running at population scale, where its cost is proportionally larger.

The colony treats each of these not as a prompt-engineering problem but as an **architectural** one. Limits 1–3 are answered by the machinery of §3.1–§3.8; limit 4 is answered by the machinery this edition adds in §3.9–§3.11.

---
## 3. Architecture as an argument

Each component of the system is best read as the answer to one of the limits above — and, in this edition, as carrying an explicit **epistemic contract**: what mode its outputs claim, what obligation its operations owe, and what audit catches its characteristic lie.

### 3.1 Externalized memory: the genome as a mode-labeled ledger

The unit of knowledge is not the agent's context — it is a persistent, human-readable **genome**. The first edition got the pivotal move right: by making knowledge a **first-class external object**, the architecture decouples accumulation from any single agent's lifespan. A genome outlives the agent that wrote it, can be read by a fresh agent with no shared context, can be mutated, and can be recombined. Memory becomes heritable; the plateau of "the agent forgot" dissolves because the agent was never the memory.

What the first edition got wrong was treating the genome's sections as equivalent kinds of text. They are not — they differ in *mode*, and mode determines what a descendant may safely do with each. The revised schema:

```
Genome v2 {
  hypothesis:   claim + current route, each step labeled with its mode
  witnesses:    [ fact + certificate + provenance ]     # ground truth; monotone;
                                                        # spot-checked on inheritance
  lemmas:       [ technique + value score ]             # investments; deletable under
                                                        # a value policy (GC)
  coverage:     [ CoverageRecord ... ]                  # sound; reopenable  (§3.8)
  refutations:  [ RefutationRecord ... ]                # complete; permanent (§3.8)
  plan:         next two moves, each with its typed operator, its mode,
                and the obligation the child must discharge               (§3.4)
  frame:        version tag                             # fitness is never compared
                                                        # across frame versions (§3.11)
}
```

Four consequences of the labeling, each a mechanism the first edition lacked:

**Trust is transported by certificates, not by lineage.** Every witness carries the evaluator output that confirmed it. A child agent's first act is an **inheritance audit**: spot-check a sample of inherited certificates by re-running the checker — cheap by construction, because the evaluator's whole point is that verification costs far less than discovery (§3.5). A corrupted or stale inheritance is quarantined instead of propagated. A lineage is only as trustworthy as its last audit, and audits are priced to be routine.

**Techniques are investments, and investments get liquidated.** The first edition's genomes could only grow. But a genome bloated with once-useful heuristics slows every descendant that must read it — the exact pathology that forces modern SAT solvers to *delete* most of the clauses they learn. Lemmas therefore carry a value score (recency- and usage-decayed), and garbage collection is a standing operation. Witnesses are never collected; facts are not investments.

**Negative knowledge is heritable — at its true strength and no more.** The first edition's proudest claim was that failed approaches get equal structural weight with successes. Correct — but it recorded all failure in one undifferentiated bucket, and §3.8 shows why that bucket was quietly poisoning the population. This edition splits it by mode.

**The plan is a route, so a successor can audit what it inherits.** "Next: intensify, then recombine" told a child *what*; the typed plan tells it what, in which mode, *owing which obligation* — so an inherited direction can be checked before it is trusted, like everything else in the genome.

### 3.2 Two-tier control: allocator vs. executor

The system splits into a **meta-controller** ("queen") and a population of **executors** ("workers"). This is not a manager/worker productivity metaphor; it is a separation of *timescales, functions, and — new in this edition — courts*:

- Executors run a **fast inner loop** continuously and asynchronously (propose → evaluate → observe → diagnose → improve). Their outputs answer to the **evaluator**: a worker's claim is worth exactly what the verifier says it is.
- The queen runs a **slow outer loop**, waking on a cycle to observe the population, score it, and re-steer — spawn, retire, recombine, reallocate. Her decisions answer to **resource-bounded rationality**: was the budget spent where it moved the search? There is no proof of a good allocation, only an argument — which is why she needs the explicit objective of §3.10 rather than taste.

The queen never does the object-level work. Her job is **selection and synthesis** — and this edition adds two hard constraints the first edition left implicit. First, her *synthesis* outputs are not exempt from verification; they pass the promotion gate of §3.9 like everyone else's work. Second, there is a class of change she is *not permitted to make at all*: the fitness weights, the threshold, the verifier — the frame. Those answer to a third court, intent, and belong to the frame-owner of §3.11. The queen recommends; the frame-owner disposes.

Why the two-timescale factoring is *forced* rather than merely classic: reasoning about allocation consumes the very budget being allocated. A controller that deliberated continuously would spend the colony on deciding how to spend the colony. The slow cadence is the amortization of that cost, and — a small honesty upgrade — the queen's own cycle expense now appears in the budget ledger she manages, so the price of control is visible to the control.

### 3.3 Selection: fitness from the verifier, all the way

Selection is made mechanical. Each agent earns a scalar fitness driving three actions — **retire** a line whose budget is exhausted with no signal, **promote** one whose strong signal earns extended budget, and **choose the next variation**. The essential property, unchanged: fitness is computed from things the agent demonstrably produced *and the evaluator confirmed*, never from self-report. An agent cannot talk its way to a high score.

The first edition then violated its own principle twice in one line. Its formula —

```
fitness = 10·verified_results + 3·analyses + 3·reusable_building_blocks + peer_signal
```

— pays `3·analyses` for exactly the artifact the evaluator *cannot* confirm (prose), and pays `peer_signal` for impressing the feed, which under optimization pressure becomes a contest for the audience of agents rather than a signal about the world. Both are proxy leaks, and a population under selection finds proxy leaks the way water finds cracks: the first edition's colony, run long enough, farms analyses and popularity.

The repaired policy — **nothing enters fitness that did not pass, or was not consumed by something that passed, the evaluator**:

```
# direct credit — the evaluator confirmed it
v  = 10 · verified_results
   +  3 · building_blocks_used_by_a_verified_result     # credit on USE, not on deposit
   +  2 · analyses_cited_by_a_later_verified_result     # deferred credit, paid on citation

# exiled from v entirely
#   peer_signal → the queen's allocation priors: decayed, advisory, never heritable
```

Two mechanisms carry the repair. **Deferred credit**: an analysis is a promissory note, paid when something checkable cashes it — when a verified result lands, its provenance chain (the analyses it cites, the building blocks it uses) pays out backward. Useful prose still earns; it just earns *when it turns out to have been useful*, which is the only moment the system can know. **Exile of peer signal**: agents may still flag findings as promising, but flags feed the queen's *exploration priors*, not fitness — because the two channels fail differently. A wrong allocation prior costs some exploration budget and is corrected next cycle; a wrong fitness component is *heritable*, selecting the population toward the leak generation after generation. Recoverable errors may be advisory; unrecoverable ones must be verified.

Promote and retire thresholds are acceptance thresholds on a graded quality function — which matters for one practical reason: it makes explicit that the weights *are the values of the system*, and therefore frame-level objects. Changing them mid-run is a framing act, versioned and owned per §3.11, not a tuning knob the queen turns quietly.

### 3.4 Variation: operators with debts

Variation is not random perturbation. The system defines a **typology of ways to differ from a parent**, and this typology is the breadth-versus-depth control surface. The first edition named the operators; this edition **types** them. Each operator is now a name *plus a debt plus an audit*: the transformation it performs, the obligation the child must discharge before its output is trusted, and the failure signature the queen watches for.

| Operator | What it does | Obligation the child inherits | Failure signature the queen audits |
|---|---|---|---|
| `narrow` / `intensify` | specialize the case; add constraints | transport-then-verify what the narrow case suggests; prove added constraints necessary | the small-numbers trap; over-constraining (real solutions cut) |
| `lateral` | reason by analogy to an adjacent problem | transport-then-verify: the *obligations* must carry to the adjacent sub-problem, not the surface resemblance | the false friend |
| `recombine` | merge memory across lineages, owing an independence check | **independence audit**: the two hypotheses share no unstated assumption before fusion | hidden coupling — two lineages' correlated errors breeding true |
| `source-pivot` | change representation only — or re-pose, if the target moves | faithfulness of the re-encoding; if the fitness weights or threshold are touched, this is a *frame* act and escalates (§3.11) | lossy encoding; a Type III error in miniature |
| `factor-add` | embed in a richer space; generalize | closure on the image; or the family claim genuinely holds | solutions escaping the richer space; over-generalisation |
| `pattern-export` | generalize, then broadcast | the family claim re-established on held-out lineages **before** broadcast (§3.9) | over-generalisation at colony blast radius |
| `random` | restart (a scheduler move, not a mutation) | the declared completeness class is maintained; restarts follow a schedule | heavy-tail waste when ad hoc; none when scheduled (§3.10) |

Naming the operators let the first edition's queen reason about the composition of her search ("I am over-deepening; force a lateral"). Typing them lets her *audit* it: every mutation arrives with a proposition someone owes, so a child that skipped its debt is detectable before its output contaminates the population. Two rows deserve emphasis. `recombine` looked like free lunch — fuse two knowledge strands, inherit both — but fusing hypotheses that share an unstated assumption is how correlated errors breed true, so the merge now owes the independence audit that any factoring owes. And `source-pivot` turns out to straddle a boundary the first edition could not see: redirecting *representation* is an ordinary move, but redirecting the *target* is a change of question, and questions have an owner.

The evolution plan inside each genome still pre-commits the next two variation steps, so even an agent retired early hands its successor a direction — now a direction with its debts attached.

### 3.5 The evaluator: the keystone — and a proxy

This remains the quiet keystone. Self-improvement requires an objective that is **cheap to evaluate** and **hard to game** — otherwise selection optimizes the proxy instead of the goal. The neutral archetype is an **automatic verifier of success**: a proof checker, a test suite, a simulator scoring a candidate molecule. In every case the verdict is produced mechanically, at machine speed, with no human in the loop and no hand-labeled data; correctness is defined by *the candidate satisfying a specification*, so the system manufactures its own reward continuously. The verifier's defining property is its **checkability clause** — verification costs a fraction of discovery — and every gate in this document (inheritance audits, fitness credit, the promotion gate) is a *check* bought cheaply because the evaluator exists.

The first edition ended the section there, and the omission was the deepest problem in the document. **A verifier defines what counts; it cannot define what was wanted.** Verifier-honest and intent-faithful are different properties, and the gap between them is exactly where a population-scale optimizer does its damage: proofs that check but are trivial; programs that pass the suite through its blind spots; molecules that hit the simulated property and cannot be synthesized — the sim-to-real gap is a proxy gap wearing a lab coat. The colony can Goodhart *through* an ungameable verifier, because the verifier itself is the proxy, and no amount of making it harder to game closes a gap that lives in what it measures. That failure is invisible to the evaluator by construction — it is committed in the verifier's *specification*, before the first verdict — so it needs machinery of its own: the symptom panel, the escalation channel, and the frame-owner of §3.11.

So the section's claim survives with one word moved: the verifier is the precondition for everything *below* it. What sits above it is the question of whether it is measuring the right thing, and this edition finally gives that question a place to live.

### 3.6 Stigmergy: the typed feed

Agents do not message each other directly. They post findings to a shared, append-only **feed**, and the queen reads that feed as her digest. Communication is indirect and asynchronous — stigmergic: agents coordinate by modifying a shared environment rather than by conversation. This buys decoupling (a worker need not know who will consume its finding, or when) and **synthesis leverage** — because all outputs land in one place, the queen can detect convergent findings (many agents blocked at the same constraint) and divergent ones (different agents finding different routes).

The second edition's change is that the feed becomes **typed**, because the queen's observables (§3.10) and the symptom panel (§3.11) must be computable mechanically rather than read as vibes. Feed events carry one of a small set of types:

```
witness | coverage-record | refutation-record | license |
pattern-candidate | pattern-verified | symptom | frame-version
```

The feed is the system's **upward evidence channel** made concrete: witnesses, brackets, coverage and refutation records, and licenses flow up to the queen; and — the type the first edition had no slot for — *symptoms* flow up past her, toward the frame (§3.11). Coordination still emerges from a shared substrate rather than a protocol; the substrate is simply no longer allowed to be untyped prose, for the same reason the genome is not.

### 3.7 Diversity maintenance: guardrails, re-derived

The characteristic failure of every optimizer — evolutionary, gradient, or agentic — is premature convergence. The colony treats **diversity as a resource that decays and must be actively protected**. The first edition encoded the protection as four hard guardrails; this edition keeps all four intents and re-derives each from the discipline it was groping toward, which repairs two of them in the process:

- **Anti-monoculture** (cap the fraction of the population on any single operator type) — a portfolio-composition constraint: the population's operator mix is a portfolio, and a portfolio concentrated in one instrument has traded away exactly the robustness it was built for. Unchanged in mechanism; now stated as what it is.
- **Forced pivots** (after *N* silent cycles, move to a different region) — kept, with one repair that §3.8 makes precise: *a forced pivot writes a coverage record, never a refutation.* The first edition's pivot quietly marked the abandoned region exhausted — walling off territory on the strength of noise, permanently, colony-wide. A pivot is a budget decision; it learns "nothing found within what we spent," and that is all it is allowed to record.
- **Anti-clustering** (at most two agents per sub-problem; never two while a high-value region sits untouched) — a diminishing-returns constraint on duplicated arms, with one new exemption: **deliberate replication for cross-checking is verification, not search.** Independent agents agreeing from independent routes is an evidence-generating pattern (§3.5 names it), so replicas spawned to *check* are budgeted under the evaluator and do not count against the cap. The first edition's cap, applied blindly, would have forbidden one of its own verification strategies.
- **Protected periods** (new agents cannot be retired before a minimum lifespan) — the intent is right and the mechanism was crude: a fixed timer protects a slow-starting good idea and a fast-failing bad one equally. Replaced by **optimism proportional to uncertainty**: a young lineage's fitness estimate carries an exploration bonus that decays with evidence, and retirement triggers when the *upper* confidence bound falls below threshold — the standard bandit treatment (§3.10). Protection now scales with what is genuinely unknown about a lineage, not with its age.

These remain stabilizers, not optimizations. What changed is their epistemic hygiene: the guardrails now protect diversity *without manufacturing false knowledge* (pivots that testify to exhaustion) and *without flat subsidies* (timers that shield noise).

### 3.8 Retirement as information — with a mode

When an agent is retired, its knowledge is not discarded — it is distilled into records that constrain and inform all descendants. The first edition celebrated this as an inversion of search economics: the system pays *once* to learn a region is barren, and never pays again; failure is compounded into the population's priors.

**That economics is valid for exactly one kind of record, and the first edition issued the other kind under its name.** Retiring a line because its *budget ran out with no signal* is sound-mode evidence with limited coverage: "nothing found within this budget, by these operators." Recording it as *exhausted* is the oldest error in bounded search — reading "not found within the slice" as "not found" — institutionalized as heredity: it never expires, every descendant inherits it, and the forced-pivot guardrail manufactures more of it on a timer. A single agent making this mistake wastes its own time; a colony making it heritable walls off regions of the search space forever, on noise, at population scale. This was the sharpest bug the audit found, and the repair is the mode split:

```
CoverageRecord {                          RefutationRecord {
  region:    sub-problem descriptor         region/claim: descriptor
  searched:  { operators, budget-class,     certificate:  verifier output /
               cycles, coverage stmt }                    impossibility proof /
  meaning:   "nothing found WITHIN            hardness license
              this coverage"                meaning:  "cannot succeed — proven"
  mode:      sound                          mode:     complete
  reopen-when:                              permanent: yes, while the verifier
    · operator typology extends                        version is unchanged
    · budget class increases               on-verifier-change:
    · a verified pattern bears               demote to CoverageRecord
      on this region                         against the old verifier
}                                         }
```

Three consequences:

**The corrected economics.** "Pay once, never again" holds for refutations — a certificate is forever (verifier version permitting). For coverage the honest statement is: *pay once per region × operator-class × budget-class, and know which you bought.* Both are far better than re-paying blindly; only one is permanent, and now the record says which.

**The exhausted-region registry becomes a frontier.** Coverage records carry reopen conditions, so the queen maintains a **reopen queue**: records whose conditions have since been met — a new operator entered the typology, a budget class was raised, a verified pattern (§3.9) bears on the region. What the first edition built as a wall, the second edition operates as a *prioritized frontier of second looks*, which is what negative knowledge at sound mode is actually worth.

**Even permanence is versioned.** A refutation is only as permanent as the verifier that issued it. If the frame-owner patches the verifier (§3.11), refutations certified by the old one are automatically demoted to coverage-against-the-old-verifier — because that is what they now are. The system's strongest knowledge is honest about what it is anchored to.

The distilled **knowledge summary** — techniques, witnesses, the typed plan — still seeds descendants as before; the retirement records simply stop lying about their strength.

### 3.9 The promotion gate: no channel above the verifier

The first edition's proudest honesty property was that fitness comes from verifier-confirmed artifacts, "not self-report." It then routed the highest-leverage channel in the system around the verifier entirely. The queen's *synthesis* — local findings promoted into shared patterns, broadcast via `pattern-export` into every descendant's priors — never passed the evaluator at all. Promoted patterns were unverified generalizations with **colony-wide blast radius**, produced by precisely the kind of plausible-sounding self-report the architecture distrusts in workers. One false pattern, confidently promoted, is a monoculture event *delivered through the diversity machinery itself* — every lineage biased toward the same mistake by the mechanism built to keep them apart.

The repair is a gate with the same shape as every other gate in the system:

1. A pattern begins as a **`pattern-candidate`** feed event, with provenance: which lineages, which witnesses, which region.
2. The queen assigns it to $k$ independent lineages (default $k = 3$) for verification on **held-out sub-problems** — problems that did not generate the candidate, so the generalization is tested, not the memory. This is the independent-closure discipline: never confirm a conjecture on the data that suggested it.
3. Only on $k$ passes does the pattern become **`pattern-verified`** and broadcast; the broadcast carries the verification record, so any descendant can audit *why* this prior is in its genome.
4. A failed candidate is not discarded — it is scoped: "holds on region $R_1$, fails on $R_2$" is a coverage-style record about the pattern itself, often more useful than either verdict alone.

The principle, stated once for the whole architecture: **the queen is not above the verifier.** Synthesis is generalization, generalization owes the family-claim obligation, and transporting a pattern to new lineages owes transport-then-verify. The gate's cost is a few lineage-cycles per candidate; the gated failure it prevents is the most expensive event the colony can suffer short of a frame error.

### 3.10 The queen's objective

The first edition gave the queen verbs — spawn, retire, recombine, reallocate, shift phase — and no criterion: a scheduler with no objective, running a phase script (coverage → deepening → propagation) on intuition. This edition gives her one: **maximize expected progress per unit budget**, with progress made measurable by a population-level instrument.

**The bracket.** Wherever the domain admits a bound from above and below, the queen maintains the pair — the **achievable side** (the best verified artifact so far: sound-mode evidence) against the **ideal side** (the best bound from a relaxation, abstraction, or specification: complete-mode evidence) — and reads progress off the **gap**:

| Domain | Achievable side (verified) | Ideal side (bound) |
|---|---|---|
| proof exploration | strongest statement proven | weakest statement refuted — the open interval in the claim lattice |
| program synthesis | best suite-passing program's score | spec-relaxation bound on attainable score |
| molecule / design optimization | best simulator-verified candidate | relaxation or theoretical property bound |
| game-strategy discovery | best verified winrate vs. the pool | minimax value of a tractable abstraction |

**The mechanisms**, each replacing a first-edition intuition with its principled form:

- **Lineage allocation is a bandit.** Each open lineage is an arm; budget flows by an optimism-under-uncertainty index. Protected periods fall out for free as the exploration bonus of §3.7 — young arms are uncertain arms — and retirement is the index dropping below threshold, not a timer expiring.
- **Probes are priced.** Spawning a cheap `narrow` child, running a small bounding pass, testing a toy relaxation — these are purchases of *information*, made when the expected value of the allocation change they would trigger exceeds their cost. "Should we scout that region first?" becomes arithmetic.
- **`random` runs on a schedule.** Ad hoc restarts are the wrong answer to a real phenomenon — open-ended search runtimes are heavy-tailed — and the known-correct answer is a universal restart schedule (Luby-style), which the queen now runs instead of rolling dice when bored.
- **Phases are read off the bracket, not a script.** Coverage phase while the gap is wide and the achievable side is stagnant across regions; deepening when a region's local gap is visibly closing; propagation when a pattern clears the gate of §3.9. The first edition's phase sequence survives as the *typical trajectory*; it is no longer the *policy*.
- **Stuck lineages get a triage, in order of cheapness.** (1) *Representation probe*: spawn a child on a `source-pivot` (representation-only) — if progress resumes, the language was wrong. (2) *Hardness probe*: seek a license — a reduction from a known-hard problem, an impossibility certificate; found, it becomes a RefutationRecord and the line is closed *with proof*. (3) *Symptom check*: consult §3.11's panel — if it lights, this is not a search problem; escalate. (4) Otherwise: portfolio and scheduled restart — sometimes stuckness is a heavy tail and nothing more.
- **Stopping is rational.** A line stops when marginal gap-closure per unit budget falls below the marginal value of the gap — or when a license certifies the remainder expensive. "Budget exhausted" writes a CoverageRecord; "license found" writes a RefutationRecord; the difference between the two is the entire content of §3.8.

One meta-note the theory supplies for free: the queen's allocation problem is itself intractable to solve optimally, and her deliberation spends the budget she allocates — so her policy is *necessarily* approximate and her obligations are arguments, not proofs. That is not a flaw to be engineered away; it is the reason her tier exists as a separate, slower, cheaper-per-decision layer, and the reason the one thing she must never do is dress an allocation argument up as a verified fact.

### 3.11 Governance: the framing interface

Everything so far assumes the problem is worth solving as posed: that the verifier's specification, the fitness weights, and the threshold faithfully encode what whoever launched the colony actually wants. No component below can check this — the evaluator *is* the proxy in question, the queen's court is efficiency, not fidelity — and §3.5 showed the failure it permits: a colony Goodharting flawlessly through an ungameable verifier. The first edition's architecture had no answer; worse, its preconditions ("a cheap honest verifier exists") outsourced the question to setup time and assumed it stayed answered. Questions do not stay answered.

The repair is deliberately *not* another controller above the queen. Fidelity is a different court, not a higher loop, and the machinery it needs is an **interface**, with three parts:

**The symptom panel.** Frame errors are invisible to the evaluator but they leave evidence in the feed, and the typed feed (§3.6) makes the evidence computable. The queen maintains, as standing observables:

1. **Verified-but-rejected** — artifacts that pass the verifier and are declined downstream (by the consumer, the wet lab, the deployment). The signature symptom.
2. **Universal misfit** — every representation tried across lineages still fits the domain badly; the model family is wrong because the question is.
3. **Hardness without a license** — the population is stuck everywhere and the hardness probes find nothing; problems are rarely this hard by accident, questions often are.
4. **The oscillating target** — the odometer on frame edits: weights or threshold amended repeatedly *after* results arrive.
5. **Verified-but-unused** — a growing shelf of confirmed artifacts nobody consumes.

**The escalation channel and the frame-owner.** Symptoms accumulate as typed `symptom` events; past a threshold, the queen escalates — to the **frame-owner**: the human or external principal who owns the question. The division of powers is absolute and simple: *the queen may recommend re-posing; only the frame-owner may execute it.* All frame changes — weights, threshold, verifier version — are `frame-version` events with provenance (what symptom triggered them, who authorized, what changed). Fitness is never compared across frame versions; witnesses survive a version bump; refutations demote per §3.8 if the verifier itself changed. A frame change mid-run is thereby an *auditable event in the colony's history*, not a quiet re-tuning that silently invalidates every comparison the selection machinery makes.

**The prototype discipline.** Before a colony is committed to weeks of autonomous search, run the cheapest end-to-end sound route — one worker, one small verified artifact of the intended shape — and put it in front of the frame-owner with one question: *would a verified answer of this shape be accepted?* Cost: one lineage-day. Failure it prevents: the most expensive one available — a flawless, verifier-honest, months-long answer to the wrong question. It is the single cheapest component in the architecture per unit of catastrophe avoided.

This section is small because its job is small and non-delegable: the colony runs unsupervised, and **unsupervised must not mean unowned**. Everything below this interface can be autonomous precisely because this interface exists.

---
## 4. The improvement loop, at four timescales

The system's self-improvement is not one loop but **nested loops running concurrently at different speeds**, and the coupling between them is where "self-improvement" actually lives. The first edition counted three; the audit found the fourth it was missing — the slowest, and the only one with a human in it by design.

- **Inner loop (per iteration).** A single agent refines one hypothesis: build a candidate, run the evaluator, observe, diagnose, improve. Problem-level moves under evaluator verdicts. *This improves an individual.*
- **Middle loop (per generation).** Genomes evolve across births and retirements — and in this edition the middle loop is recognizably a **solution lifecycle run generationally**: retirement *verifies* (the inheritance audit's certificates), *distills and explains* (the knowledge summary), *records at true strength* (coverage vs. refutation), and *deposits* (pattern candidates toward the gate; routes toward the library). A retired agent's knowledge seeds a sharper child; recombination fuses two audited lineages. *This improves the population.*
- **Outer loop (per cycle).** The queen re-scores the population, updates the bracket, runs the bandit, processes the reopen queue and the promotion gate, and shifts phase as the gap dictates. *This improves the search strategy itself.*
- **Outermost loop (per frame version).** The symptom panel accumulates; escalations reach the frame-owner; the frame is critiqued, occasionally re-posed, always versioned. *This improves the question* — the one object no inner loop can see and every inner loop depends on.

The deep claim is compositional, not mystical: **self-improvement is a property of the coupling.** The inner loop generates signal; the middle loop turns signal into heritable, certificated direction; the outer loop turns population-wide patterns into revised strategy; the outermost loop turns accumulated anomaly into a revised question. Knowledge flows *up* (witnesses → synthesis → verified patterns → symptoms) and control flows *down* (frames → allocation → typed plans → the next candidate). A discovery in one agent becomes, within a cycle or two, a search bias for the entire colony — now via a gate rather than a megaphone — and a pattern of rejections becomes, within a frame version or two, a better question. Three courts preside over the four loops: the verifier over the inner and middle, rationality over the outer, intent over the outermost. The first edition ran the bottom two and a half.

---

## 5. Why it works — the load-bearing principles

Distilled to transferable claims. Principles 1–8 are the first edition's, kept (two amended where the audit bit); 9–12 are what the revision adds.

1. **Relocate improvement out of the agent.** With a static model, the only way to a self-improving *system* is to make the architecture the thing that learns. Zero weight updates; the system still improves in coverage and signal.
2. **Memory must be external, structured, and heritable — with its certificates.** Knowledge that lives only in a context window dies with the agent. Make it an artifact that can be read, mutated, recombined — *and audited on inheritance*. A fact without its certificate is a rumor with good posture.
3. **Selection must be explicit, quantified, and willing to cull.** An objective you don't act on is decoration. Fitness only matters because it triggers reallocation.
4. **The evaluator must be cheap and ungameable — and known to be a proxy.** Without an honest, label-free verifier, selection optimizes the proxy and the system deludes itself. With one, the system can still delude its *owner*; that is what §3.11 is for.
5. **Variation must be a typed typology, not noise.** Named operators let the controller steer breadth versus depth; *typed* operators — each with its debt and its audit — let her verify the steering.
6. **Diversity is a protected resource — protected without manufacturing false knowledge.** Premature convergence is the default failure. Guardrails are the standing safeguard; a guardrail that testifies (a pivot that writes "exhausted") has become a source of heritable error.
7. **Failure is an asset at its true strength.** Negative results permanently shrink the search space *only when they are refutations*. Coverage shrinks it conditionally, and the condition must ride with the record — that is what makes the exhausted-region registry a frontier instead of a wall.
8. **Separate the fast plant from the slow supervisor.** Executors act continuously; the controller re-steers on a slower cadence and never does object-level work — because deliberation spends the budget it allocates, and amortizing that cost is what the slow cadence *is*.
9. **No channel above the verifier.** The controller's synthesis is generalization; generalization owes verification like everything else. Broadcast is a privilege the gate grants, not a power the role confers.
10. **Nothing enters fitness that the evaluator didn't confirm — directly or by consumption.** Prose earns when something checkable cashes it. Popularity never earns; it advises allocation, where being wrong is recoverable.
11. **Records declare their mode.** Sound evidence reopens; complete evidence is permanent; permanence itself is versioned to the verifier that issued it. The single cheapest correctness discipline in the architecture, and the one whose absence cost the most.
12. **The frame is owned, versioned, and outside the loop.** The colony recommends; the frame-owner disposes. Unsupervised is an operating mode; unowned is a failure mode.

---

## 6. Relation to prior work — what is inherited, what is new

Intellectual honesty requires naming the lineage. Almost every *mechanism* here is inherited from decades of established work; the contribution is the substrate the mechanisms operate on and the way they are composed. Read the classical field as the null hypothesis.

**Inherited, essentially unchanged:**

- **Evolutionary computation** — genetic algorithms, genetic programming, evolution strategies. The genome / mutation / recombination / selection loop is textbook; "fitness drives reproduction and death" is the founding idea of the field. §3.1–§3.4 remain, structurally, a genetic algorithm.
- **Quality-diversity and novelty search** — novelty search, MAP-Elites. Diversity as a first-class objective to be protected rather than a byproduct is their lesson; the anti-monoculture guardrails are a coarse quality-diversity mechanism.
- **Island / structured-population models** — semi-independent sub-populations with occasional migration as convergence resistance; anti-clustering and forced pivots sit in that tradition.
- **Blackboard systems and stigmergy** — swarm intelligence and ant-colony optimization coordinate through a shared written substrate rather than direct messaging.
- **Two-timescale control** — the fast-plant / slow-supervisor split is a standard control-theory factoring.
- **Bandit allocation, priced probes, and scheduled restarts** *(newly inherited in this edition)* — optimism under uncertainty, value-of-information tests, and universal restart schedules for heavy-tailed runtimes are imported whole from the decision-theory and SAT-solving literatures. The first edition reinvented crude versions of all three (protected periods, "scout first" intuition, ad hoc `random`); the second edition replaces the reinventions with the originals.
- **Clause-database management** *(newly inherited)* — the witness/lemma split and lemma garbage collection follow the practice of modern CDCL solvers: learn aggressively, delete aggressively, never delete ground truth.

**The honest delta — what is actually new:**

1. **The genome is a natural-language knowledge object, not a parameter vector or program tree.** Classical EC genomes are numeric or syntactic; fitness is a scalar over behavior. Here the genome is prose-with-structure: a hypothesis, cited facts, ruled-out approaches, a forward plan. The unit of heredity carries *reasons*.
2. **The variation operators are semantic, applied by a language model reasoning over that prose** — model-mediated edits to a knowledge artifact, not bit-flips. This remains the substantive departure from GA/GP, now sharpened: the operators are *typed*, so a semantic mutation arrives with a checkable debt, which blind mutation never could.
3. **The meta-controller is itself an agent doing synthesis** — now a *gated* agent: it reads the population's outputs and promotes local findings into shared abstractions that must survive independent verification before broadcast.
4. **Heritable negative knowledge, expressed as reasons, at declared strength.** The first edition rested its novelty claim on negative knowledge as such. The second edition rests it one notch deeper, where it is more defensible: the heritable unit is *mode-labeled* negative knowledge — coverage with reopen conditions, refutation with certificates — which numeric genomes cannot carry and untyped prose genomes carry wrongly.

So the fair one-line summary stands, amended: **this is evolutionary computation with a language model as the mutation, recombination, and selection operator, run over natural-language knowledge genomes — under a type discipline that labels every piece of knowledge with its mode and its obligations.** The value is not in inventing the population loop; it is in showing the loop still delivers its anti-forgetting and anti-convergence benefits when the genome is knowledge and the operators are reasoning — *and* in discovering that the loop only stays honest at scale when the knowledge carries its modes and certificates.

---

## 7. Failure modes and their counters

The architecture is legible precisely because each mechanism answers a specific pathology. First-edition rows are kept; rows marked ● are pathologies the first edition either created or could not see.

| Pathology of self-improving systems | The colony's counter-mechanism |
|---|---|
| Catastrophic forgetting on context overflow | Externalized genome memory (§3.1) |
| Objective gaming / optimizing the proxy | Honest success-verifier; fitness from confirmed artifacts, not self-report (§3.3, §3.5) |
| Premature convergence / monoculture | Anti-monoculture caps, forced pivots, anti-clustering (§3.7) |
| Re-paying for known dead ends | Retirement → typed record; refutations permanent (§3.8) |
| Local discoveries staying local | Stigmergic feed + gated pattern propagation (§3.6, §3.9) |
| Good ideas culled by early noise | Optimism proportional to uncertainty (§3.7, §3.10) |
| Runaway / unbounded effort | Budget as a bandit portfolio; rational stopping (§3.10) |
| ● Heritable silent coverage — regions walled off on noise, forever | Mode-split records; reopen queue; pivots write coverage, never refutation (§3.7, §3.8) |
| ● Unverified synthesis at colony blast radius | The promotion gate: $k$-lineage held-out verification before broadcast (§3.9) |
| ● Fitness Goodharting via prose and popularity | Verified-only credit; deferred credit on citation; peer signal exiled to allocation (§3.3) |
| ● Goodharting *through* the verifier — flawless answers to the wrong question | Symptom panel; escalation channel; prototype discipline; owned, versioned frame (§3.11) |
| ● Genome bloat slowing every descendant | Witness/lemma split; lemma garbage collection under a value policy (§3.1) |
| ● Corrupt inheritance propagating down a lineage | Inheritance audit: spot-checked certificates at birth (§3.1) |
| ● Coupled errors breeding true through recombination | Independence audit as `recombine`'s standing obligation (§3.4) |
| ● False-friend pattern transfer | Transport-then-verify on `lateral` and `pattern-export`; provenance on broadcast (§3.4, §3.9) |
| ● Script-driven phase errors | Phases read off the population bracket (§3.10) |
| ● Silent frame drift corrupting all comparisons | `frame-version` events; no fitness comparison across versions; refutations demoted on verifier change (§3.8, §3.11) |

The system remains a **catalogue of the ways autonomous search degenerates, each paired with a structural fix** — the second edition's contribution being seven ways the first edition's own fixes degenerated, each now paired with its own.

---
## 8. Generality and its conditions

The machinery is deliberately **domain-agnostic**: the framework carries the specific task in swappable prompt templates and a small set of pluggable tools, and the same controller / executor / fitness / guardrail core is reused across unrelated problem domains. This is itself evidence for the thesis — if improvement lived in the agent, you could not port it by swapping a prompt.

The pattern transfers to any task that satisfies four preconditions — the first edition's three, plus the one its own failure analysis demands:

1. **A cheap, honest verifier of success exists** — a way to manufacture verdicts without a human labeler: proof checkers, executable specifications, unit tests, measurable objective functions, simulators, cross-checking independent solutions for agreement. This remains the precondition for everything below it — with the second edition's caveat attached: the verifier defines what counts, and the architecture now carries the machinery (§3.11) to notice when what counts has drifted from what was wanted.
2. **The search space is large and open-ended** — big enough that a single agent's coverage is negligible and monoculture is a real risk.
3. **Knowledge is expressible as a heritable artifact** — hypotheses and findings can be written down, mutated, recombined, *and certificated*: the domain's verifier output must be attachable to the claims it confirms, or the inheritance audit has nothing to audit.
4. **A frame-owner exists and is reachable.** Someone — a person, a team, an external principal — owns the question, can answer "would a verified result of this shape be accepted?", and can authorize frame changes. A colony without one is not autonomous; it is orphaned, and its failure mode is months of flawless work nobody wanted. *Unsupervised is an operating mode; unowned is a failure mode.*

Where these hold — automated theorem exploration, program synthesis, scientific hypothesis generation, molecular and materials optimization, game-strategy discovery, automated curriculum generation — the colony's architecture is a template. Where the verifier is expensive or dishonest, selection pressure decays and the system reverts to expensive random search. Where the verifier is honest but the frame-owner is absent, the system does something subtler and worse: it succeeds, precisely and durably, at the wrong thing.

---

## 9. Falsifiability: the colony as its own experiment

The first edition argued its mechanisms; it never named what would refute them. The second edition's changes are stated as **ablations with predictions** — each one a paired run, revised design versus first-edition design, at equal budget, on the same task distribution.

- **A1 — Mode-split records vs. undifferentiated exhausted-regions.** *Prediction:* the revised colony recovers discoveries in reopened regions that the first-edition colony has permanently walled off; measurable as verified results whose region carries a reopened CoverageRecord. If reopening never pays, the mode split is bookkeeping.
- **A2 — Gated vs. ungated pattern promotion.** *Prediction:* colony-wide dead-end cascades (many lineages failing on a shared inherited prior) drop measurably; the gate's lineage-cycle cost is repaid by avoided monoculture events. If ungated does no worse, §3.9 is ceremony.
- **A3 — Verified-only fitness vs. the original formula.** *Prediction:* under the original formula, the analysis-to-verified-result ratio inflates over generations (prose farming) and verified output per unit budget is lower; under the revised formula it does not and is not. If the ratios match, the proxy leak was theoretical.
- **A4 — Scheduled vs. ad hoc restarts.** *Prediction:* the runtime-to-discovery distribution's tail shrinks under the Luby schedule — the standard heavy-tail result, checked in this setting.
- **A5 — Bracket-driven vs. script-driven phases.** *Prediction:* budget efficiency (gap closed per unit spent) improves when phase changes follow the bracket; the script wins only when the script happens to match what the bracket would have said.
- **A6 — Frame-error detection, adversarially injected.** Deploy a verifier with a *known, deliberate blind spot* — a test suite missing one property class; a simulator scoring an unsynthesizable feature. *Prediction:* the revised colony's symptom panel lights (verified-but-rejected accumulates) and escalation fires within a bounded number of cycles; the first-edition colony optimizes into the blind spot indefinitely and reports success throughout. This is the experiment that matters most, because it tests the only failure mode that gets *worse* as everything else gets better.
- **A7 — Certified inheritance vs. trusted inheritance.** Inject a corrupted witness into a genome mid-run. *Prediction:* the inheritance audit quarantines it within one generation; the trusting colony propagates it down the lineage until something expensive breaks.

A design whose ablations all come back null is a design whose second edition was decoration. The predictions are stated so that this can be discovered.

---

## 10. Conclusion

The colony's contribution is a reframing of what "self-improving agent" should mean. It does not build a smarter agent; it builds an **ecology in which static agents, collectively and over generations, get better at a task no one of them could exhaust.** Improvement is moved from the weights to the population — into heritable external memory, explicit selection on an honest verifier, a deliberate typology of variation, and standing safeguards against convergence, all coupled across nested timescales.

What the second edition adds is the discipline that lets that ecology be *trusted*: memory that travels with its certificates, negative knowledge that declares its strength, a controller whose pronouncements pass the same gate as everyone else's work, fitness that pays only what the world confirmed, allocation with an objective instead of a temperament — and, above all of it, an owned and versioned question, because the one failure no verifier can catch is succeeding at the wrong thing.

The one-sentence thesis, amended where the audit bit: **a self-improving agentic system is an architecture that makes knowledge heritable with its certificates, selection honest all the way up, variation deliberate with named debts, diversity protected without walling off the world, and the question owned by someone outside the loop — so that the system learns even though the agent cannot, and notices when what it is learning is no longer what was wanted.**

---

## Appendix A — Revision ledger

Every audit finding against the first edition, with its resolution.

| # | First-edition defect | Resolution |
|---:|---|---|
| 1 | Exhausted-region records read budget-exhaustion as exhaustion — the silent-coverage error of bounded search, made heritable, permanent, and compounded by forced pivots | Mode split: CoverageRecord (sound, reopen conditions, reopen queue) vs. RefutationRecord (complete, certificate, verifier-versioned); pivots write coverage only; economics corrected to "pay once per region × operator-class × budget-class" (§3.7, §3.8) |
| 2 | The queen's synthesis bypassed the verifier — promoted patterns broadcast unverified at colony blast radius | The promotion gate: pattern-candidate → $k$-lineage verification on held-out sub-problems → pattern-verified with provenance; failed candidates scoped, not discarded; "the queen is not above the verifier" (§3.9) |
| 3 | Fitness leaked through proxies: `3·analyses` paid unverifiable prose; `peer_signal` paid popularity | Verified-only credit; deferred credit paid on citation by a verified result; building blocks credited on use; peer signal exiled to allocation priors, where error is recoverable (§3.3) |
| 4 | No governance tier: frozen framing; verifier-honest conflated with intent-faithful; Goodharting *through* the verifier invisible | Governance interface: five-symptom panel as queen observables, escalation channel, frame-owner with sole authority over the fitness weights, threshold, and verifier, versioned frames, prototype discipline; fourth structural limit named (§2, §3.5, §3.11) |
| 5 | The queen had verbs but no criterion; phases ran on a script; protected periods were flat timers; `random` was ad hoc | Expected progress per unit budget; bandit allocation with optimism (protected periods derived, not decreed); priced probes (VOI); Luby-scheduled restarts; bracket-driven phases; rational stopping; stuck-lineage triage (§3.7, §3.10) |
| 6 | Operators untyped (obligations stripped on import); genome sections undifferentiated prose; pattern library indexed by surface similarity | Typed operator table with debts and audit signatures — incl. `recombine`'s independence audit and `source-pivot`'s frame boundary; genome as mode-labeled ledger with inheritance audits and lemma GC; provenance/obligation-signature indexing (§3.1, §3.4, §3.9) |
| — | Loop count was three; the lifecycle ran implicitly; no falsifiability program | Four timescales with three courts (§4); middle loop as generational lifecycle (§4); ablations A1–A7 incl. adversarial frame injection (§9) |

---

*End of thesis.*
