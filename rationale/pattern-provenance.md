# Pattern provenance: borrow / adapt / ignore

Memoria draws on a broad survey of contemporary AI-research systems — LitSearch, ResearchArena, SciLitLLM, LitLLM, LatteReview, ResearchAgent, Idea2Story, AI-Scientist (v1 and v2), AgentLaboratory, Sibyl, AutoResearchClaw, AI-Researcher, Auto-Research, OmegaWiki, CORAL, AIDE ML, MLR-Copilot, Karpathy Autoresearch, AI co-scientist (Gottweis et al.), AiScientist long-horizon engineering (Chen et al.), AgentRxiv, SciMON, MOOSE, CiteME, MetaGPT, AutoGen, OpenHands, AI-Supervisor (Long 2026), PARNESS (Wang & Luan 2026), and the surveys of Gridach et al. (ICLR 2025), Chen 2026 (*From Copilots to Colleagues*), Huang et al. 2025 (*Deep Research Agents*), Ren et al. 2025 (*Scientific Intelligence*), and Xu & Peng 2025 (*Deep Research Systems*).

This document is the synthesized borrow / adapt / ignore table. The headline patterns are summarized in [00-vision.md](../00-vision.md#contemporary-ai-research-systems); the autonomy boundary that rejects several of these patterns wholesale is in [01-architecture/why-no-autonomous-synthesis.md](../01-architecture/why-no-autonomous-synthesis.md).

## Borrow

| Pattern | Source repos | Why borrow it | Where it fits |
| --- | --- | --- | --- |
| Stage-gated pipeline | ResearchArena, MLR-Copilot, AutoResearchClaw | Prevents everything from collapsing into one giant prompt. | Ingest → triage → synthesize → promote. |
| Explicit roles per agent | AI-Scientist v2, Sibyl, LatteReview, AgentLaboratory | Keeps planning, execution, review, and writing separate. | Hermes profiles vs. coding agent vs. human review. |
| Strong schema with validated handoffs | AI-Scientist, AutoResearchClaw | Makes automation reliable and debuggable. | Frontmatter namespaces, `_draft_classification`, `_enrichment`. |
| Persistent knowledge graph | OmegaWiki, Idea2Story (Idea2Paper) | Preserves relationships instead of re-searching every time. | Wikilinks, MOCs, entity / item notes. |
| Reviewable organization artifacts | LitLLM, LatteReview | Makes synthesis inspectable, not hidden in prompts. | Canvas, MOCs, reference notes. |
| Persistent Kanban + worker lanes | Hermes Agent (NousResearch) | Durable state machine across sessions and retries. | Board layer; seven profiles claim lanes. |
| Point-of-action discovery loop | Karpathy Autoresearch | Reactive → proactive: scheduled discovery while you sleep. | Overnight loop pattern in [07-roadmap.md](../07-roadmap.md). |
| File-as-Bus / "thin control over thick state" | Chen et al. 2026 (AiScientist long-horizon engineering) | Independent empirical validation: removing the durable-artifact bus drops PaperBench by 6.41 and MLE-Bench Lite by 31.82 points. Confirms the three-layer split. | The whole architecture; cited in [01-architecture.md](../01-architecture.md#why-three-layers-not-one). |
| Structured outputs at handoff boundaries | MetaGPT (Hong et al. 2024) | Independent validation that structured-output handoffs (vs. free-text chat) reduce cascading hallucinations across roles. | Frontmatter schema discipline ([reference/schema.md](../reference/schema.md)); profile handoff contracts. |
| Cross-run knowledge accumulation as structural commitment | PARNESS (Wang & Luan 2026) | Third independent corroboration of the durable-state thesis (alongside Chen 2026 and AgentRxiv). PARNESS names "no existing tool persists cross-run knowledge in a form retrievable into a finite LLM context" as one of five structural problems in the field and addresses it with a persistent knowledge layer. | Vault layer ([01-architecture.md §"Thin control over thick state"](../01-architecture.md#thin-control-over-thick-state)). |
| Paper ↔ code repository linking as first-class artifact | PARNESS (Wang & Luan 2026) | A paper's open-source repository is often the only complete specification of its experimental scheme; PARNESS makes the paper↔code correspondence a typed object in the knowledge graph. Memoria's `code-note` type does not currently link to upstream papers explicitly. | Source-note enrichment step ([profiles/researcher.md](../profiles/researcher.md)) — when ingesting a paper, automatically search for and link its code repository when available. |

## Adapt

| Pattern | Source repos | Why adapt it | How |
| --- | --- | --- | --- |
| ResearchArena-style discover / select / organize | ResearchArena (S2ORC-based) | Good conceptual model, but we need a deeper writing layer. | Map to discover → ingest → triage → synthesize → promote. |
| AI-Scientist modular roles | AI-Scientist v1 + v2 (SakanaAI) | Useful, but full autonomy is too broad. | Keep separate planner / writer / code-executor; humans canonize. |
| Knowledge-graph compilation | Idea2Story (Idea2Paper), OmegaWiki | Great for durable memory, but should not replace citation-based notes. | Graph artifacts are a layer on top of Zotero + Obsidian, not a replacement. |
| Benchmark-style evaluation loops | LitSearch (Princeton-NLP), ResearchArena | Good for quality control, not as the main workflow. | Periodic checks for retrieval quality, note completeness, link density. |
| Multi-reviewer systematic-review | LatteReview, LitLLM | Strong for formal reviews, overkill for ongoing reading. | Optional `review_mode: systematic-review` layer (see [07-roadmap.md](../07-roadmap.md) Decision 19). |
| Domain-adapted LLM | SciLitLLM (Uni-SMART) | Useful long-term upgrade path, heavy now. | Defer; benchmark synthesis quality first via LitSearch-style harnesses. |
| Memory module + Meta-review artifact | Gottweis et al. 2025 (AI co-scientist) | The Memory store + Meta-review research-overview pair is the same shape as Memoria's vault + MOC. Architectural validation of a vault-plus-roles design from the most production-mature system in the survey. | Vault layer ([01-architecture.md](../01-architecture.md#layer-3-vault-obsidian-folders)); MOC type ([05-notes-folders.md](../05-notes-folders.md)). The tournament/evolution loop on top is *not* adapted — see Ignore. |
| Inspiration retrieval before drafting | SciMON (Wang et al. 2024) | The "retrieve related prior ideas, then draft" mechanic improves grounding. We adopt the retrieval step, not the novelty optimizer that drives keep/revert. | Writer profile reads related claim notes as inspiration context before drafting; novelty is *not* a stopping criterion — operator stops. |
| Hypothesis-feedback taxonomy (present / past / future) | MOOSE (Yang et al. 2024) | Three-channel feedback structure is a useful rubric shape for review prompts. | Verifier profile prompt design ([profiles/verifier.md](../profiles/verifier.md)) — borrow the taxonomy, not the autonomous quality gate. |
| Citation-attribution benchmark | CiteME (Press et al. 2024) | Frontier LMs hit 4–18% on it; CiteAgent reaches 35%. Gives the Verifier a numeric regression target where there currently is none. | Verifier acceptance criterion — see [07-roadmap/future-directions.md](../07-roadmap/future-directions.md#citeme-style-verifier-regression-harness). |
| Agent-readable shared synthesis pool | AgentRxiv (Schmidgall & Moor 2025) | Empirical: agents reading prior agent-generated reports gain ~11% on MATH-500 over isolated agents. Validates the Karpathy LLM-Wiki claim quantitatively. | Cross-project reading inside a single user's vault — [07-roadmap/future-directions.md](../07-roadmap/future-directions.md#cross-project-reading-as-personal-agentrxiv). |
| Consensus pre-filter before human review | AI-Supervisor (Long 2026) | Long's "Research World Model" commits findings only when multiple independent agents corroborate. A *milder* version of Memoria's blocking-human-review pattern — does not replace the human gate but could pre-filter high-confidence cards, reducing operator review load without moving the autonomy boundary. | Lane-bounded prototype on the Researcher profile, with measurement before broader adoption — [07-roadmap/future-directions.md §"Agent-consensus pre-filter"](../07-roadmap/future-directions.md#agent-consensus-pre-filter). |
| Scenario-typed retrieval (similar / contradictory / cross-domain / counter-intuitive) | PARNESS (Wang & Luan 2026) | Richer than untyped wikilinks for surfacing prior work with a specific relation to a draft or claim. Operator queries like "show me contradictory claims about X" are not directly supported by current wikilink semantics. | A `relation_type:` frontmatter field on claim notes or wikilinks — minimal taxonomy first (`similar` / `contradictory`); expand if usage justifies — [07-roadmap/future-directions.md §"Scenario-typed retrieval"](../07-roadmap/future-directions.md#scenario-typed-retrieval). |

## Reference

Papers that inform Memoria's framing or positioning but contribute no borrowable pattern — typically surveys and taxonomies.

| Pattern / contribution | Source | Why referenced |
| --- | --- | --- |
| L1–L5 autonomy taxonomy | Chen 2026 (*From Copilots to Colleagues*) | Vocabulary for positioning Memoria precisely on the autonomy spectrum. Memoria targets L3-with-structurally-enforced-ceiling — see [00-vision.md §"Position on the autonomy spectrum"](../00-vision.md#position-on-the-autonomy-spectrum). |
| "Persistent knowledge accumulation is the #1 barrier to L5" finding | Chen 2026 | Independent validation of Memoria's vault-as-load-bearing-piece thesis from a 95-paper survey. Memoria's central commitment is what Chen identifies as the field's open problem. |
| Deep Research as sibling-but-distinct category | Huang et al. 2025; Xu & Peng 2025 | Defines a category (query-driven, ephemeral-report agents like OpenAI DR / Gemini DR / Perplexity DR) that Memoria explicitly is not. Cited in [00-vision.md §"What Memoria is not"](../00-vision.md#what-memoria-is-not) as negative positioning. |
| Autonomous-vs-collaborative axis | Gridach et al. ICLR 2025 | The binary that Memoria sits unambiguously on the collaborative side of. Survey findings that literature-review automation is the field's weakest sub-task and that system reliability remains open reinforce Memoria's "agent does bookkeeping; human owns judgment" thesis. |
| Scientific-agents survey | Ren et al. 2025 | Surveys LLM-based agents specialized for scientific domains (chemistry, biology, materials). Lower priority — relevant if Memoria's future-directions ever explore domain-science applications, not a current commitment. |
| Uncertainty-boosts-diversity finding | Qi et al. 2023 | A tuning observation for the overnight discovery loop: sample candidates at higher temperature, evaluate at lower. Not a borrowable pattern, just a temperature default. |

## Ignore

| Pattern | Source repos | Why ignore |
| --- | --- | --- |
| Full autonomous scientist mode | AI-Scientist v2, Sibyl, AI-Researcher, Auto-Research | Misaligned with human-review philosophy. |
| Tree search over experiment code | AIDE ML, AI-Scientist v2 | Knowledge work, not ML benchmarking. |
| Physical-lab / robotics extension | Auto-Research (openags) | Not relevant to a vault-centered system. |
| One-model-does-everything | (anti-pattern across early frameworks) | The system benefits from separation of retrieval, synthesis, and code execution. |
| Autonomous keep / revert without review | Karpathy Autoresearch | Synthesis correctness isn't a scalar metric; see the [autonomy boundary](../01-architecture/why-no-autonomous-synthesis.md). |
| Tournament-evolve autonomy loop | AI co-scientist (Gottweis et al. 2025) | Self-improving hypothesis quality via ranking tournaments + test-time-compute scaling, with no blocking human gate per iteration. The Memory/Meta-review pair is *Adapt*; this loop on top is the same scalar-optimization shape Memoria refuses. |
| Conversation-as-substrate | AutoGen (Wu et al. 2023) | Inverts the central commitment that conversation is ephemeral and the vault is the memory ([00-vision.md](../00-vision.md#design-goals) goal 5). Adopting a chat-as-orchestrator runtime would route durable state through an inappropriate layer. The conversation *pattern* (humans-in-the-loop chats) is fine within an ACP pane; the *substrate* is not. |
| Generalist sandboxed dev agent as drop-in worker | OpenHands (Wang et al. 2025) | Strong runtime in isolation, but its permission model is sandbox-vs-host, not per-zone-per-profile. Adopting it as the Coder runtime today would replace Memoria's lane-scoped policy MCP with a coarser boundary. Re-evaluate if a concrete Coder limitation surfaces; otherwise stay on Claude Code. |

## Net effect

The design shift versus a generic "agent-assisted knowledge base" is from agent-assisted to **bounded, stage-gated knowledge production**. The agent becomes better at bookkeeping, retrieval, and drafting; the human remains the gatekeeper for meaning, promotion, and final structure. This makes the architecture more reliable, easier to debug, and less likely to accumulate polished but untrusted content.
