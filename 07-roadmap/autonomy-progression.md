# Autonomy progression

A dependency-ordered roadmap for the 12 patterns that increase Memoria's *within-boundary* autonomy — the agent does more unattended work between human gates without moving the gates themselves.

## Scope

"More autonomous" has two readings:

1. **Same gate, fewer presses.** The human review state remains structurally blocking; the agent does more bookkeeping, drafting, surfacing, and verification per gate press. This document is *only* about reading 1.
2. **Move the gate.** Loosen the autonomy boundary so the agent decides what stays without operator approval. That is a redesign of [01-architecture/why-no-autonomous-synthesis.md](../01-architecture/why-no-autonomous-synthesis.md), not a borrow from the surveyed papers. Not covered here.

This document is the answer to "how do we let the agent do more between gates?" It is not the answer to "should the gate stay where it is?" That question is settled in [00-vision.md](../00-vision.md) and [01-architecture/why-no-autonomous-synthesis.md](../01-architecture/why-no-autonomous-synthesis.md).

## Principle

**Same gate, fewer presses.** Every pattern below is admissible because it routes outputs to `10-inbox/` or to a dashboard surface, never to canonical zones. The policy MCP's canonical-zone deny rule is the structural guarantee that this stays true regardless of how much unattended activity accumulates.

## The layers

Five layers, in strict dependency order. A pattern in layer N depends on at least one pattern in layer < N to function or to pay off meaningfully. Within a layer, patterns are independent of each other.

### Layer 0 — Reliability foundation

Preconditions, not autonomy gains in themselves. Without these, every layer above is unreliable; with them, retries become a manageable cost.

| # | Pattern | Source | Status |
| --- | --- | --- | --- |
| 0.1 | Structured-output handoffs at profile boundaries | MetaGPT (Hong 2024) | Already specified in [reference/schema.md](../reference/schema.md); recently named as a borrow in [pattern-provenance.md](../rationale/pattern-provenance.md) |
| 0.2 | Reflection on retry | NousResearch hermes-agent-self-evolution | [future-directions.md §"Execution-trace reflection on retry"](future-directions.md#execution-trace-reflection-on-retry) |

**Why first.** Layer 1+ run unattended; an unattended pipeline that fails on the third retry blocks until the operator wakes up. Structured handoffs prevent silent format drift between profiles; reflective retry prevents identical re-dispatch on structurally broken inputs. Together they raise the unattended success rate to the point where nightly autonomy is worth attempting.

### Layer 1 — Unattended cycles (the gateway)

The single largest autonomy gain in the roadmap. Everything above this layer compounds with it.

| # | Pattern | Source | Status |
| --- | --- | --- | --- |
| 1.1 | Overnight discovery loop | Karpathy Autoresearch | [future-directions.md §"The overnight loop"](future-directions.md#the-overnight-loop-proactive-discovery-pattern) |
| 1.2 | Hypothesis sampling at higher temperature | Qi 2023 | Tuning choice within 1.1; cited inline |
| 1.3 | CiteME-style Verifier nightly harness | Press 2024 (CiteME) | [future-directions.md §"CiteME-style Verifier regression harness"](future-directions.md#citeme-style-verifier-regression-harness) |

**Sequencing within the layer.** 1.1 is the headline pattern. 1.2 is a tuning bullet within 1.1 (sample candidates at higher temperature, evaluate them at lower temperature). 1.3 is *parallel* — it does not depend on 1.1 and provides a different kind of unattended autonomy (unattended quality tracking). Both 1.1 and 1.3 share the structural property "agent runs on a cron, writes only to inbox or dashboard, operator reads in the morning."

**Why this layer is the gateway.** Layers 2–4 amplify or refine the inbox produced by 1.1. Without unattended discovery cycles producing volume, the patterns above this layer have nothing to operate on at a scale that justifies their cost.

### Layer 2 — Amplifying nightly volume

Patterns that improve *what flows through* the overnight loop without operator hand-feeding.

| # | Pattern | Source | Status |
| --- | --- | --- | --- |
| 2.1 | Gap-seeking planner | AI-Researcher, ResearchAgent | [future-directions.md §"Gap-seeking planner"](future-directions.md#gap-seeking-planner) |
| 2.2 | Inspiration retrieval before drafting | SciMON (Wang 2024) | Adopted as Adapt in [pattern-provenance.md](../rationale/pattern-provenance.md); Writer-profile change |

**Why after layer 1.** Both work without the overnight loop (you can run the gap-planner on demand; inspiration retrieval helps any Writer invocation). But 2.1 produces *discovery queries* whose natural consumer is 1.1's nightly run, and 2.2 pays off most when there's enough vault content for retrieval to find non-trivial matches — which the overnight loop accelerates accumulating.

### Layer 3 — Reduce per-batch operator decisions

The overnight loop fills the inbox; this layer reduces operator time per inbox item.

| # | Pattern | Source | Status |
| --- | --- | --- | --- |
| 3.1 | Semi-autonomous triage | LatteReview, ResearchAgent | [future-directions.md §"Semi-autonomous triage"](future-directions.md#semi-autonomous-triage) |
| 3.2 | Tournament ranking for triage | AI co-scientist (Gottweis 2025) | [future-directions.md §"Tournament ranking for triage"](future-directions.md#tournament-ranking-for-triage) |

**Why after layer 1.** Both presuppose a triage queue with enough volume that batching and ranking matter. Below ~10 candidates per research direction per cycle (the threshold named in 3.2's implementation gate), the existing per-card Cartographer ordering and per-card operator decision are sufficient and cheaper.

**Order within the layer.** 3.1 first — it removes operator decisions from high-confidence candidates entirely (batch-approve). 3.2 second — it reorders the remaining lower-confidence candidates so the operator reads from a tournament top-K rather than from a flat list. 3.2 without 3.1 reorders the full list; 3.1 without 3.2 leaves the residual list flat-ordered. Together they handle both ends.

### Layer 4 — Corpus-density gated

Patterns whose value is dominated by *vault size*, not by *nightly volume*. They become meaningful only once the corpus has accumulated to a critical density.

| # | Pattern | Source | Status |
| --- | --- | --- | --- |
| 4.1 | Reviewer agents on draft synthesis | ResearchAgent (Baek 2025), AI co-scientist Reflection | Partial coverage in [future-directions.md §"More agent roles"](future-directions.md#more-agent-roles-and-internal-reviewers) and §"LLM-judge gate for export" |
| 4.2 | Propagation debts queue | Autonovel cross-layer change propagation | [future-directions.md §"Propagation debts"](future-directions.md#propagation-debts) |
| 4.3 | Cross-project reading as personal AgentRxiv | Schmidgall & Moor 2025 (AgentRxiv) | [future-directions.md §"Cross-project reading as personal AgentRxiv"](future-directions.md#cross-project-reading-as-personal-agentrxiv) |

**Why last.** Each has a corpus-density precondition:

- 4.1 needs enough drafts that a pre-review pass catches more structural issues than it falsely flags. Below ~20 drafts per project, the operator's own review is fast and the LLM pre-review is noise.
- 4.2 needs ~500 claim notes before the cost of materializing a dependents queue exceeds the cost of an Obsidian backlink walk.
- 4.3 needs ≥ 2 projects with ≥ 8 weeks of activity each. Single-project vaults give cross-project reading nothing to surface.

These three are also relatively independent — none requires another within the layer. They can ship in any order as their preconditions land.

## Dependency graph

```text
Layer 0 ──────────────────────────────────────────────────────────────
                                                                       
  0.1 structured-output handoffs ──┐                                   
                                   ├──► reliable unattended runs       
  0.2 reflection on retry ─────────┘                                   
                                                                       
Layer 1 ──────────────────────────────────────────────────────────────
                                                                       
  1.1 overnight loop ──────────────────┐                               
        │                              │                               
        └─ 1.2 temperature sampling    │                               
                                       │                               
  1.3 CiteME harness ──── (parallel) ──┤                               
                                       ▼                               
Layer 2 ─────────────────────── nightly inbox volume ─────────────────
                                                                       
  2.1 gap-seeking planner ──► feeds discovery queries to 1.1           
  2.2 inspiration retrieval ──► improves Writer outputs                
                                       │                               
                                       ▼                               
Layer 3 ────── operator reviews larger / better-ordered batches ──────
                                                                       
  3.1 semi-autonomous triage ──► batch-approve high-confidence         
  3.2 tournament ranking ──► reorder residual candidates               
                                       │                               
                                       ▼                               
Layer 4 ─────────── once corpus density justifies cost ───────────────
                                                                       
  4.1 reviewer agents on synthesis                                     
  4.2 propagation debts queue                                          
  4.3 cross-project reading                                            
```

## What each layer feels like at steady state

- **Pre-progression (today).** Every candidate is operator-discovered, operator-triaged, operator-reviewed. Retries fail loudly. The vault grows by direct operator action.
- **After layer 0.** Same operator workload; fewer interruptions for retry failures and format mismatches. The agent's nightly runs (when added) won't break on the third retry.
- **After layer 1.** Mornings start with a curated inbox produced overnight. The operator's discovery effort drops sharply; the per-candidate triage effort stays the same. Verifier drift is caught by a metric instead of by surprise.
- **After layer 2.** The inbox is not only larger but better-aimed (gap-planner) and Writer drafts come with relevant prior thinking pre-loaded (inspiration retrieval). The vault starts feeling self-suggesting.
- **After layer 3.** Triage is bulk-approve plus a short tournament-ranked residual. Per-card operator decisions drop to a small fraction of the inbox volume.
- **After layer 4.** Reviewer agents catch structural issues before the operator's review. Propagation queue surfaces what to re-read when a claim shifts. Cross-project reading surfaces "I wrote about this two projects ago" connections the operator no longer has to remember.

The cumulative shift: from "review every candidate, approve every triage, manually link across projects, hand-feed context to the Writer, watch the Verifier" to "review the morning's curated batch, approve in bulk, follow surfaced gaps." Same gate. Far fewer presses.

## What this is not

- **Not the autonomy boundary.** The 12 patterns above keep [01-architecture/why-no-autonomous-synthesis.md](../01-architecture/why-no-autonomous-synthesis.md) unchanged. Autonomous keep/revert, auto-promotion to canonical, scalar-metric optimization, and conversation-as-substrate remain refused.
- **Not a substitute for the human review gate.** Every pattern routes outputs to `10-inbox/` or to a dashboard, never to canonical zones. The policy MCP enforces this regardless of how much unattended activity accumulates.
- **Not a strict implementation order across layers.** Layers are dependency-ordered; *within* a layer (and between layers, where the preconditions are independent) the operator can sequence by available time, budget, or felt friction.
- **Not a commitment.** This is a roadmap, not a contract. Any of the 12 may be deferred indefinitely if the operator finds they aren't experiencing the friction they relieve. See each pattern's "When to implement" and "Why not earlier" in [future-directions.md](future-directions.md) for the per-pattern triggers.

## Related

- The autonomy boundary that this progression respects: [01-architecture/why-no-autonomous-synthesis.md](../01-architecture/why-no-autonomous-synthesis.md).
- The pattern-by-pattern borrow / adapt / ignore mapping: [rationale/pattern-provenance.md](../rationale/pattern-provenance.md).
- The per-entry "when / why not earlier / prerequisites" detail: [future-directions.md](future-directions.md).
- The MVS implementation phases (a different axis — *building* the system, not *expanding agent autonomy*): [timeline.md](timeline.md).
