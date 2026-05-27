# Future directions

Things to consider once the core system is running and stable. Each is documented with a "when to implement" trigger and explicit prerequisites — so the deferral has a clear off-ramp when conditions are met.

## Event-driven and scheduled operations

- Nightly enrichment refresh for source notes more than 30 days old.
- Weekly orphan-detection scan with a dashboard surface.
- Monthly merge-candidate report.
- On-add hook: when a new citekey appears in `00-meta/03-config/library.bib`, auto-trigger ingest.

## The overnight loop (proactive discovery pattern)

Once `research-directions.md` (see [04-workflows.md](../04-workflows.md)) is live and stable, the highest-leverage automation is the **overnight loop**: Hermes runs unattended on a schedule, fills `10-inbox/03-candidates/`, and posts a morning summary. This converts the system from *reactive* (you trigger discovery) to *proactive* (discovery happens while you sleep).

The pattern, drawn from Karpathy's Autoresearch by analogy:

```text
nightly (2am):
  1. read 00-meta/research-directions.md
  2. pick top N priorities (default: 3)
  3. for each priority: run `discover` (max 10 candidates)
  4. ingest confirmed candidates from the previous day's inbox
  5. enrich any source notes flagged as stale
  6. commit: "nightly: +N candidates, +M ingested, +K enriched"
  7. post morning summary to Telegram / email / dashboard
```

**Concrete defaults:**

- **Batch size, not duration.** Our bottleneck is human triage capacity, not compute. Cap at ~10 candidates per priority per night, not a wall-clock budget. (Karpathy's 5-minute time budget makes ML experiments comparable; here, it would just mean fewer candidates with no analog payoff.)
- **The loop generates; the morning triages.** The overnight pass only writes to `10-inbox/03-candidates/`. It does not promote to `_draft_classification`, file synthesis, or close cards. All keep/revert decisions happen during the morning triage session.
- **Fail loud, not silent.** A failed run sends an alert (Telegram, email, or a dashboard flag). Silent cron failure is the dominant operational risk — three days of missing morning summaries should be impossible to overlook.
- **Cost budget.** ~10 ingests + 3 discoveries per night ≈ $1–3/day in API calls ≈ ~$500/year. Set this as an explicit budget; alert if a nightly run exceeds 2× the median.

**Prerequisites:**

- **[Deployment Option C](deployment-options.md) (Syncthing + VPS)** — sleep-prone WSL2 won't survive the cron schedule.
- `research-directions.md` actively maintained.
- `00-meta/screening-protocol.md` or equivalent inclusion criteria — without explicit criteria, the inbox floods with low-quality candidates and the morning triage time explodes.
- Two weeks of tuning before the loop is trustworthy. Expect the first nightly batches to require aggressive pruning.
- **Model routing configured** so embed / classify / quick-summary calls go to a cheap model — see [01-architecture/capability-stack.md](../01-architecture/capability-stack.md#model-routing-synthesis-on-claude-cheap-tasks-elsewhere). Without this the cost budget below blows out 3–5×.

**Pros.** Proactive corpus growth; surfaces work that would otherwise never get done; scales to ~3,500 candidates/year filtered down to ~200 ingested — the difference between a hobby corpus and a research corpus.

**Cons.** Always-on infrastructure cost; first-month tuning friction; bad criteria flood the inbox; silent cron failures.

**When to implement.** After: (1) MVS is stable, (2) `research-directions.md` has been maintained for ≥ 4 weeks, (3) [Option C](deployment-options.md) is deployed, (4) inclusion criteria are written down. Not before.

## Semi-autonomous triage

- Hermes proposes triage promotions for notes where draft fields have high confidence.
- The human approves in batch via a triage-approval view.
- Lower-confidence triages remain manual.

## Autonomous synthesis (cautious)

- Hermes drafts synthesis notes from research questions on a schedule.
- All drafts land in `10-inbox/` as `synthesis-note`, awaiting review.
- Never auto-promoted.

## Gap-seeking planner

- A "gap" agent identifies under-cited claims, contradictory claims, or topics with sparse coverage.
- Proposes discovery queries to fill gaps.
- Surfaces results to the human as a research-planning aid.

## Self-evaluation and benchmarks

- Periodic retrieval-quality checks against held-out questions.
- Note-completeness audits.
- Link-density and orphan-rate trends over time.

## Cross-vault read-only retrieval

Memoria assumes one vault per researcher. If you eventually run multiple vaults — say a personal vault and a project-specific or lab-shared vault — the right pattern is **MCP-mediated, read-only cross-vault retrieval**, not direct vault-to-vault links or sync.

The shape:

- Each vault stays isolated. No profile writes to a vault other than its own.
- Profiles can *read* an external vault through an MCP proxy that:
  - normalizes the external vault's notes into a uniform "research object" schema (`object_id`, `source_type`, `title`, `authors`, `created`, `canonical_url`);
  - exposes only a search/fetch surface — no write tools at all;
  - returns results that are clearly tagged as foreign so the worker can decide whether to create a local stub note in its own vault.
- The local stub captures the reference but never replaces the original. The vault that owns the original stays canonical for it.

**When to implement.** When at least one of these is true:

- You're collaborating with another researcher who maintains their own Memoria vault.
- You have a project vault separate from your reading vault, and ingest needs to query both.
- A lab or team adopts Memoria and you need a shared corpus alongside personal corpora.

**Why not adopt earlier.** Single-vault is structurally simpler in every way — one policy MCP, one audit log, one set of dashboards. Adding the cross-vault proxy doubles the surface area of the policy layer. Don't pay that cost until there's a real second vault.

**Prerequisites.** A stable "research object" schema (overlapping but not identical to the source-note frontmatter), an MCP proxy server, and explicit allowlisting in each vault for which other vaults may read it.

**What this is not.** Not federation — there is no sync, no conflict resolution, no global identity. Each vault is the source of truth for its own notes. Cross-vault is read-only retrieval, not shared state.

## Fleet observability

Once the corpus is large enough that the human eye stops noticing slow regressions, stand up the [fleet-observability dashboard](../dashboards/fleet-observability.md). It tracks per-lane and per-skill metrics (cost per task, success rate, retry rate, latency) on a daily / weekly / monthly / quarterly cadence.

**When to implement.** After Phase 6, and only when at least one of these is true:

- More than ~50 tasks per week — eyeballing the board misses regressions.
- Multiple lanes running scheduled work — drift in one lane is hard to spot from another.
- API spend on the order of $50+/month — cost optimization starts paying for the dashboard's overhead.

**Prerequisites.** A scheduled Hermes task that aggregates the audit log and board history into `lane-metric` and `skill-metric` notes under `00-meta/08-metrics/`. Until that aggregator exists, the dashboard is a placeholder.

**Why not earlier.** Before this scale, the [audit-log dashboard](../dashboards/audit-log.md) and [board-state dashboard](../dashboards/board-state.md) carry the operational load. Adding the fleet view too early is overhead without payoff.

## Propagation debts

When a high-traffic note changes — a `claim-note` promoted to `evergreen`, a `reference-note` updated, a `source-note` retracted — the change should ripple to anything that depends on it. The implicit answer today is "follow the backlinks in Obsidian." That works at low corpus density. At high density it doesn't, because the human can't tell which dependents are *load-bearing* and which are casual mentions.

The pattern, borrowed from Autonovel's cross-layer change propagation: maintain a small "propagation debt" queue in the linter's report. When a triggering change happens, enumerate the dependents that need re-evaluation and record them as actionable items, not just backlinks. Example: promoting a claim to `evergreen` queues "review each draft that cites this claim — does the prose still match?"

**When to implement.** When the corpus passes ~500 claim notes *and* the human notices reading a draft and realizing the cited claim has shifted. Below that density, the backlink walk in Obsidian is fine.

**Why not earlier.** Premature debt-tracking adds maintenance overhead without a corresponding read of the queue. The system needs to feel the *absence* of this view before adding it.

**Prerequisites.** A "trigger event" taxonomy (what changes warrant queuing dependents), a linter check that materializes the queue into a dashboard, and a habit of working the queue down rather than letting it accumulate.

**What this is not.** Not automatic propagation. The agent never rewrites a draft to match a shifted claim. The queue is a *reading prompt*, not an edit pipeline.

## LLM-judge gate for export

The export step (`draft → deliverable`) currently has one quality check: `cite-check` verifies every citekey resolves to a real source. That catches mechanical breakage. It does not catch prose problems — argument gaps, tonal drift between sections, a paragraph that contradicts an earlier one.

A bounded LLM-judge gate would close that gap: at export time only, a `prose-check` command invokes a model to score the manuscript on a small fixed rubric (argument coherence, voice consistency, citation grounding, one-claim-per-paragraph). The result is a report attached to the export card; the human reads it and decides whether to revise or proceed. The gate is **terminal-only** — it never runs against synthesis, never auto-edits, never blocks the export by itself. The human reads the report and decides.

**When to implement.** When a deliverable has been re-exported more than twice because of issues a model could have caught on the first read.

**Why not earlier.** A blanket LLM judge on every draft is the autonomous-keep/revert pattern Memoria refuses (see [01-architecture/why-no-autonomous-synthesis.md](../01-architecture/why-no-autonomous-synthesis.md)). What makes the export-only variant safe is that it's strictly informational, runs only on the terminal artifact, and never gates promotion through the canonical zones.

**Prerequisites.** A small, stable rubric (drift-prone if open-ended), a writer-lane skill that returns structured output, and the discipline of reading the report before accepting the export.

**What this is not.** Not editorial control. The model proposes; the human disposes. If the model's report becomes the authority, the gate has failed its design intent.

## Execution-trace reflection on retry

Today's retry pattern (see [03-board.md](../03-board.md#retry-pattern)) re-dispatches a failed card with the same task packet — same prompt, same context, same expectations. Two retries on a flaky API succeed where one didn't. But two retries on a structurally broken prompt fail identically — the lane burns through `retry_count` and the card escalates to `blocked-on-human` with three identical failure traces stacked in the comments.

The pattern, borrowed from [NousResearch/hermes-agent-self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution): on `retry-needed`, a **reflection skill** (loadable by the lane's primary worker) reads the *failure trace* — what tool was called, what arguments, what error came back, what the agent said next — and synthesizes a *modified* task packet for the next attempt. Not a model evaluating the work; a model reading the trace and adjusting the inputs. The next attempt may swap a tool, narrow a search, drop an irrelevant constraint, or escalate immediately if the trace shows the task is infeasible as written.

**When to implement.** When retry-induced API spend becomes visible in the [fleet-observability dashboard](../dashboards/fleet-observability.md) — typically when sustained retry rate > 0.10 across a lane, or when a specific skill's `retry_rate` crosses 0.20. Below that volume the retry-as-identical-redispatch pattern is fine; the cost of building the reflection layer outweighs the savings.

**Why not earlier.** Reflective retry adds a model call to every retry, which doubles the cost of cards that were going to succeed on second attempt anyway. The pattern only pays off when the retries that *would have* failed identically get fixed by the reflection. That ratio improves with corpus size and task complexity.

**Prerequisites.** Structured failure traces (the Kanban dispatcher must capture tool calls and errors, not just the agent's final message), a reflection skill bounded to packet rewriting (never to direct execution), and a `reflection_count` field on cards distinct from `retry_count` (so the operator can see when reflection itself is exhausted).

**What this is not.** Not autonomous evolution. The reflection layer never rewrites prompts, skills, or system contracts — it rewrites the *task packet* for one card. If the same kind of failure recurs across cards, that's a design issue the human resolves by editing the skill or the prompt; the reflection layer is per-card-scoped specifically to keep it from drifting into the autonomous-keep/revert pattern Memoria [refuses](../01-architecture/why-no-autonomous-synthesis.md).

## Literate `code-note` (weave + tangle)

Today's `code-note` (see [05-notes-folders.md](../05-notes-folders.md#note-types)) is markdown prose plus a separate code artifact in the same folder. The link between them is convention: humans keep the prose in sync with the source, the linter doesn't enforce it. Drift between what the code *does* and what the note *says it does* is a real failure mode at corpus scale.

The pattern, borrowed from [tlehman/litprog-skill](https://github.com/tlehman/litprog-skill) (which implements Knuth's 1984 literate-programming model): a `code-note` becomes a single `.lit.md` source from which two artifacts are derived — **weave** (the human-readable narrative-with-code that lives in the vault) and **tangle** (the executable source that lives alongside, runnable as-is). Edit either; the other regenerates. The prose-source drift problem becomes structural: drift is detected the moment the artifacts disagree.

**When to implement.** When prose-vs-source drift starts producing real incidents — a researcher reading a `code-note` to understand a result, finding the code does something different, and having to reconstruct which is correct. Below that pain threshold, the convention-based pattern is fine.

**Why not earlier.** Litprog adds a build step to the code-note lifecycle. For one-off analysis scripts this is overhead with no payoff. The pattern only pays off when `code-note`s are read repeatedly across months, not just at write time. The current `code-note` usage pattern is still settling; locking in litprog before the usage stabilizes risks tooling that doesn't match how the notes are actually used.

**Prerequisites.** A litprog implementation that handles Python and Jupyter notebooks at minimum (Memoria's two dominant code surfaces), a build hook integrated into the coder profile's compile step, and a linter check that flags `.lit.md` files whose weave-hash and tangle-hash disagree (a sibling of M2 vault-hash drift).

**What this is not.** Not a replacement for `code-note` as a type. The litprog file *is* a `code-note`, with `format: literate` distinguishing it from `format: notebook` and `format: script`. Not an enforcement mechanism either — the human still owns whether to use literate format for a given project; small scripts can stay convention-based.

## Memoria Inspector Obsidian plugin

The five existing surfaces (Obsidian dashboards, command palette, CLI, Telegram, API server — see [01-architecture.md](../01-architecture.md#surfaces-each-one-owns-one-mode)) cover daily workflow well but leave a small gap: a *browse-the-Hermes-state* admin surface. CLI is precise but not discoverable; dashboards aggregate rather than drill down per session. The temptation is to adopt a community web UI (e.g., the `hermes-workspace` hackathon project); the architectural problem is that any peer system competes for the same modes the existing surfaces already cover, ships a chat tab that bypasses the policy MCP, and locks Memoria into someone else's maintenance schedule.

The pattern: build a small Obsidian plugin (~500–1500 lines of TypeScript) exposing read-only Hermes admin views as a sidebar pane — Sessions, Audit, Skills, Memory, all tabs. Connects to the API server on `127.0.0.1:8642`. No chat (that surface belongs to ACP); no editing (skills and memory are write-through the compilation pipeline, not the inspector). The plugin extends Obsidian-as-interface rather than introducing a peer.

**When to implement.** When CLI forensics start dominating the operator's time, *and* the desire to browse rather than query becomes recurrent. A rule of thumb: more than ~3 `hermes kanban show` invocations per session means dashboards aren't drilling deep enough and a browse surface is overdue.

**Why not earlier.** Dashboards already cover 90% of inspection needs (audit log, board state, retry hotspots, drift signals). Building an Obsidian plugin before that gap is *felt* is premature optimization — the build cost is real (two to four weekends of TypeScript) and the saved time is hypothetical until the operator has been using the system long enough to notice the missing affordance.

**Prerequisites.** The API server reachable on loopback with a stable contract for session/audit/skill/memory queries, an Obsidian plugin scaffold that participates in the vault's mobile-app sync path (so the inspector is reachable on phone via Obsidian mobile rather than requiring a separate PWA), and a policy guarantee that read-only paths in the plugin can never escalate to writes through misuse.

**What this is not.** Not a chat interface — agent conversations stay in ACP panes and Telegram. Not an editing tool — modifying skills, memory, or lane-overrides still goes through the compilation pipeline so the audit log captures every change. Not a substitute for dashboards — the inspector serves drill-down forensics; dashboards serve filtered, decision-oriented views.

## Hermes → Todoist gap-card integration

The downstream gap loop ([04-workflows/08-writing-drafting.md](../04-workflows/08-writing-drafting.md)) closes through the upstream pipeline today: Verifier's `cite-check` flags an unsupported claim, the gap becomes a card in the upstream queue, the operator reads it during the weekly ritual. That works for gaps the operator can act on at the desk. It does not work for gaps that need scheduling, deadlines, or external commitment — "ask co-author about IRB scope before next meeting" or "submit amendment to ethics board by 2026-06-15."

The pattern: a thin Hermes skill that mirrors substantive gap cards to Todoist via the Todoist MCP server already in the operator's stack. The skill creates a Todoist task tagged `#memoria` and `#research-gap` with a backlink to the verification report, priority P3, into the operator's existing project structure. One-way only — Todoist is the scheduling surface; Memoria's gap card remains the canonical artifact in the vault. Closing the Todoist task does not close the gap card; the gap closes when the operator pursues the reading, files a new source note, and the next Verifier pass traces the claim cleanly.

**When to implement.** After the gap loop has run for a corpus of at least one project and the operator has noticed gaps slipping because they needed deadline-driven attention rather than weekly-ritual review. Below that, the weekly ritual covers what gaps need covering.

**Why not earlier.** Adding a second task surface (alongside the Hermes Kanban) creates the same maintenance-time-flavored risk as a Kanban UI: if the operator starts grooming Todoist as a research backlog, the discipline of working from the corpus map and weekly ritual erodes. The integration only pays off when the operator's Todoist is already well-tended as a *commitments-and-deadlines* system, which is a precondition, not a goal.

**Prerequisites.** The Todoist MCP server configured in the operator profile (typically `mcp__claude_ai_Todoist_MCP__*`), a Todoist project structure stable enough that Memoria-generated tasks land somewhere sensible, and a labelling convention that distinguishes Memoria-originated tasks from operator-created ones so the operator can filter them out when reviewing their own commitments.

**What this is not.** Not bidirectional. Closing a Todoist task does not modify the vault. Not a replacement for the gap card. Not a general "Memoria notifies via Todoist" channel — only substantive citation gaps qualify; routine link suggestions and triage reminders stay inside Memoria's own surfaces.

## Static HTML admin reports

Most admin tasks are retrospective: weekly audit review, monthly drift analysis, quarterly trust-score retrospectives. Real-time inspection (the [Memoria Inspector plugin](#memoria-inspector-obsidian-plugin) above) is the wrong shape for these — what's needed is a snapshot rendered on a schedule that can be archived, diffed, and read on any device.

The pattern: a Hermes cron skill (`hermes run admin-report`) that renders static HTML pages from the audit log, board history, and trust-score time series. Outputs land in `00-meta/05-reports/<period>/` — version-controllable, archivable, openable in Obsidian directly or served via a simple HTTP server when remote viewing is needed. No runtime infrastructure, no auth surface, no failure mode beyond "stale report" (which the underlying system survives).

**When to implement.** This is the smallest of the three admin-surface entries and the most defensible to build first. A reasonable trigger: the first time the operator wants to read the audit log on a phone over a train ride. Until then, dashboards + CLI cover the use case.

**Why not earlier.** Static reports duplicate what dashboards already render in Obsidian. The duplication is justified only when the report needs to survive on a device that isn't running Obsidian (a phone via the browser, a colleague's laptop via email, an archived PDF for an annual review). For everything else, the live dashboard is fresher.

**Prerequisites.** A small templating layer (Jinja2 or similar) for the report shape, a cron entry on the linter profile (whose dry-run posture matches the read-only nature of report generation), and a `00-meta/05-reports/` folder convention that doesn't conflict with the existing `00-meta/05-dashboards/`.

**What this is not.** Not interactive. Not a billing dashboard (cost reconciliation happens in the Hermes config). Not a substitute for the Inspector plugin (live drill-down) or the audit-log dashboard (filterable live view). Pure retrospective snapshots.

## Open-design integration

[Open-design](https://github.com/nexu-io/open-design) is the self-hosted, agent-native rendering tool whose architecture composes cleanly with Memoria's. It uses the SKILL.md convention, supports Hermes as one of its 16 agent CLIs, and operates on portable Markdown design systems. The integration is structured as the **external rendering agent** pattern — a peer of the existing [external coding agent](../rationale/coder-external-agent.md#the-pattern-generalizes-external-rendering-agents) — so it absorbs into Memoria's architecture without new control-plane concepts.

**What's already in the design** (ready to use as soon as open-design is installed locally):

- **Render-agent pattern** ([rationale/coder-external-agent.md](../rationale/coder-external-agent.md#the-pattern-generalizes-external-rendering-agents)). Memoria scaffolds the rendering task in a `deliverable` note; open-design reads the note + vault content + `design-system.md`; the human reviews the rendered artifact in `50-deliverables/`. Same review gate, same audit log, same lane policy as code work — only the artifact type differs.
- **Design-system source** ([reference/design-system.md](../reference/design-system.md)). The canonical visual-style file lives at `00-meta/06-schema/design-system.md` in open-design's portable DESIGN.md format (9 sections: color, typography, spacing, layout, components, motion, voice, brand, anti-patterns). One vault, one design system, multiple consumers (open-design renders, Pandoc exports, CSS snippets).

**Operationally valuable next steps** (worth doing once open-design is installed and the render-agent pattern has been exercised once or twice):

- **Slide-deck generation from drafts.** A new command — `Memoria: render slide deck from chapter` — that takes a section of a draft and produces a designed PPTX/HTML deck via open-design's 9 deck-mode skills. The deck lands at `50-deliverables/<project>/decks/`. High-leverage because conference/lab-meeting decks are a weekly time-sink that this kind of rendering directly attacks.
- **Selective open-design skill loading.** Open-design ships 132 SKILL.md bundles. A small subset (deck-from-markdown, infographic-template, landing-page, poster-template) are operationally relevant. Add them to the Writer lane's allowed skill list in [02-profiles.md](../02-profiles.md) — same SKILL.md convention, same lane-permissions enforcement.

**Deferred — documented but not first-cut work**:

- **`render` card type.** Adding `render` as a first-class card type on the Coder or a new Render lane. Lifecycle: `pending` → `ready` → `active` (open-design generates) → `awaiting-review` (operator inspects) → `approved` → `done`. The render-agent pattern works without this (the operator can invoke open-design directly via the existing coder pattern); promoting to its own card type makes sense when render volume justifies queue dispatch.
- **Hermes ↔ open-design MCP composition.** Open-design exposes MCP tools (`search_files`, `get_file`, `get_artifact`). Memoria's Writer could call these tools as remote skills during drafting (e.g., embed a previously-rendered figure). Same composition discipline as the existing policy-MCP + generic-rest-bridge layering. Defer until there's a recurring "I want to reuse an existing artifact mid-draft" pain point.
- **Canvas → designed artifact pipeline.** A Memoria `canvas` note (Obsidian Canvas, spatial argument mapping) can be exported through open-design as a polished figure — methodology diagram for a paper, conceptual map for a grant proposal, mechanism schematic for a thesis. Useful but niche; revisit when the operator has more than ~3 canvases per project and is recreating figures manually.

**Pros.** Memoria gains a polished-artifact path without authoring its own renderer or learning a new control plane. The boundary stays clean — content authoring in Memoria, visual rendering in open-design, the human gates promotion. A single `design-system.md` becomes the visual source of truth for everything Memoria-produced.

**Cons.** Adds a daemon (Express + SQLite + Next.js) and another agent stack to maintain. Cost overlay if image-generation models are used. Boundary discipline against Pandoc must be enforced (Pandoc for body-text exports, open-design for visual artifacts — don't double-implement). AI-generated visuals must be gated for research outputs; default disabled in Memoria's profile.

**When to implement.** When the operator hits a specific high-effort rendering task — a conference poster, a defense-talk deck, a project landing page — and recognizes they'd be hand-templating it without rendering help. Pre-emptively standing up open-design before that pain point is premature.

**Why not earlier.** Open-design is a daemon + a curated skill catalog + a design-system corpus. The integration cost (setup, design-system tuning, daily-use discipline) is real. Memoria's principle is "add complexity only when you feel the absence" — until the operator has a deliverable that *would have been* nicer with rendering support, the cost outweighs the benefit. Once they do, the pattern is in place and ready.

**Prerequisites.** Open-design installed locally (daemon running on a known port). The vault has a populated `00-meta/06-schema/design-system.md` (either from the [template](../reference/design-system.md) or imported from one of open-design's 150 built-in systems). A `50-deliverables/<project>/` folder exists for the target project. Workspace settings give open-design read access to the vault and write access to `50-deliverables/`.

**What this is not.** Not a replacement for Pandoc — they cover different artifact types. Not a one-click "make my thesis pretty" tool — open-design renders specific artifacts (decks, posters, landing pages) within an agreed design system, not arbitrary content. Not a content authoring tool — Memoria still owns content; open-design just renders it. Not a visual-AI generator by default — image generation is opt-in per project, with anti-patterns documented in the [design-system template](../reference/design-system.md).

## More agent roles and internal reviewers

- A dedicated *claim checker* that verifies every claim note traces to at least one source.
- A *consistency checker* that flags contradictions between claim notes.
- An *evidence-strength assessor* that ranks claim notes by how well-supported they are.

These additions all preserve the central rule: agents propose, humans decide.

## CiteME-style Verifier regression harness

Today's [Verifier profile](../profiles/verifier.md) traces draft claims to claim notes by prompt, with no numeric quality signal. A prompt regression — a model change, a context-length cut, a subtle template edit — degrades attribution accuracy silently until the operator notices wrong citations downstream. The Verifier is the gate that protects canonical synthesis from false attribution; running it without a regression harness is the kind of "agent does bookkeeping, but the bookkeeping is unmeasured" risk the design otherwise refuses elsewhere.

The pattern, adapted from Press et al. 2024 (*CiteME: Can Language Models Accurately Cite Scientific Claims?*): construct ~50 (excerpt → target-claim-note) pairs from approved drafts in the operator's vault, where each excerpt is a passage from a `40-workbench/02-drafts/` note and the target is the specific `30-synthesis/02-claims/` claim note the excerpt is meant to cite. Score the Verifier nightly on the fixture; record accuracy in the [skill-lifecycle dashboard](../dashboards/skill-lifecycle.md). The harness is the Verifier's *acceptance criterion* — a Verifier prompt change ships only when fixture accuracy is at or above the running 90th-percentile baseline.

The CiteME paper itself shows frontier LMs scoring 4–18% on the public benchmark and a tooled CiteAgent reaching 35%. Memoria's task is structurally easier (candidate space is bounded by one vault), so the baseline should land much higher — but the *shape* of the test (excerpt → cited artifact) and the *failure mode it protects against* (confident wrong attribution) transfer directly.

**Pros.** Converts the Verifier's correctness from a felt property into a measured one. Catches regressions across model changes, prompt edits, and context-length tuning. The fixture is reusable across model swaps and across the Verifier-companion roles ([more agent roles](#more-agent-roles-and-internal-reviewers) above) when those land.

**Cons.** Fixture construction is one-time effort (a few hours of operator labelling) plus periodic refresh as the vault's claim-note shape evolves. Nightly runs add a small recurring API cost. If the fixture is too small or too easy, the harness gives false confidence; if it is too large or too unrepresentative, the operator stops trusting the signal.

**When to implement.** When the Verifier profile has shipped at least one prompt version and is running against real drafts. Implementing earlier — before the Verifier's behavior is settled enough to label its outputs — produces a fixture pinned to whichever prompt happened to be running when the fixture was built.

**Why not earlier.** The fixture's quality depends on the operator having approved drafts and approved claim notes to draw from. Pre-corpus, there's nothing to construct excerpt-target pairs against. The harness is also the kind of measurement infrastructure whose value compounds once there's enough volume to detect a drop; built too early, it is a flat line.

**Prerequisites.** Verifier profile is live and shipping work. At least one project has produced ≥ 20 approved drafts that cite ≥ 20 approved claim notes. A skill-lifecycle metric for `verifier_attribution_accuracy` with a running-baseline definition (e.g., 30-day 90th-percentile floor). A nightly cron entry on the Linter lane (dry-run posture matches the read-only nature of the harness).

**What this is not.** Not a content-quality gate — Verifier still owns the binary "claim traces / doesn't trace" decision per draft; the harness only certifies that the gate itself hasn't drifted. Not a substitute for the human review gate — promotion still requires `review_state: approved`. Not a public-benchmark contribution — the fixture is operator-private because it draws from the operator's vault.

## Agent-consensus pre-filter

Today the Researcher profile runs a card to completion, posts the output to `awaiting-review`, and the operator decides whether to approve. Most cards approve quickly — the agent did exactly what was asked, the frontmatter is correct, the source is well-classified. A few cards require real deliberation. The operator pays the same review-attention cost for both, and the easy cards dominate the volume.

The pattern, adapted from Long 2026 (*AI-Supervisor: Autonomous AI Research Supervision via a Persistent Research World Model*): a *consensus pre-filter* between worker output and `awaiting-review`. Instead of routing a Researcher output directly to operator review, dispatch the same card to a second profile (a Cartographer in read-only mode, or a second Researcher pass with a different prompt). When the two outputs agree on key fields (`_draft_classification`, suggested wikilinks, frontmatter values), the card routes to `awaiting-review` with a `consensus: agreed` flag and the operator can batch-approve. When they disagree, the card routes to a distinct `awaiting-review` lane with a `consensus: disagreement` flag and the disagreement is the first thing the operator sees.

This is a *milder* version of Memoria's blocking-human-review pattern. The human gate remains structurally required — consensus does not bypass review, it filters review. Long 2026's framing treats agent consensus as sufficient for commit; Memoria's adaptation treats it as a pre-filter only.

**Pros.** Reduces operator review load for high-confidence cards (consensus = likely correct = quick approval). Surfaces disagreements as a distinct queue (lower-confidence, deserves more operator attention). Compatible with the blocking-review commitment — the human gate is still structurally required. Validates Long 2026's consensus mechanism inside Memoria's autonomy boundary rather than replacing the boundary.

**Cons.** Doubles API cost per card the pre-filter touches. Two agents with correlated errors may agree on wrong outputs (the AI co-scientist tournament problem at smaller scale) — agreement rate is not the same as accuracy. Adds another lane, another card flag, and another dashboard surface — design complexity grows. The savings depend on what fraction of cards actually reach consensus, which is unknown today.

**When to implement.** As a *prototype* on one profile (start with Researcher, where the task has the clearest agreement criterion) for ~50 cards. Measure (a) consensus rate, (b) operator agreement with consensus outputs, (c) operator disagreement with high-confidence consensus (the worrying case — agents agreed but got it wrong). Decide based on data whether to extend to other profiles or abandon.

**Why not earlier.** Pre-MVS, there isn't enough card volume to measure consensus rate meaningfully. Pre-overnight-loop, the operator review load is small enough that pre-filtering doesn't pay back its cost. The prototype only makes sense once the operator is feeling review-fatigue on routine cards.

**Prerequisites.** Researcher profile shipping and producing routine output. Defined "key fields" for agreement (the frontmatter slots that count as agreement vs. divergence). A `consensus:` field on cards (`agreed` / `disagreement` / `not-checked`). A second-profile-pass skill that runs the same task with a different prompt or model, returning comparable output. A budget for ~2× inference cost on cards the pre-filter touches.

**What this is not.** Not a replacement for human review — `awaiting-review` is still structurally required for promotion. Not auto-approval — consensus does not bypass the operator; it routes them to a faster queue. Not multi-agent voting — two agents is the prototype scale; expanding to N agents is a separate decision with worse cost characteristics. Not the AI-Supervisor pattern verbatim — Long 2026 commits to the RWM on consensus alone; Memoria pre-filters but does not commit.

## Scenario-typed retrieval

Memoria's current link semantics are untyped: a `[[wikilink]]` from claim A to claim B does not say *why* A links to B. The operator and the agent both have to infer the relation from context. At low corpus density this is fine — the links are few and the operator remembers the reasons. At higher density, a useful query like "show me contradictory claims about X" cannot be answered directly from the graph; the operator has to walk every backlink and re-derive the relation.

The pattern, adapted from PARNESS (Wang & Luan 2026): a `relation_type:` field on wikilinks (or on claim notes referencing other claim notes), with a small fixed vocabulary. PARNESS uses four values — `similar` / `contradictory` / `cross-domain` / `counter-intuitive` — chosen for ML/science workflows.

For Memoria, the prudent adoption is a *minimal taxonomy first* — two values, `similar` and `contradictory`, both of which are unambiguous and operator-meaningful in knowledge work. Expand to a third or fourth value only if the operator finds themselves wanting it. The risk in adopting PARNESS's full four-value taxonomy from day one is taxonomy mismatch — `cross-domain` is concrete in ML benchmarks (a method from vision applied to NLP), less concrete in a knowledge-work vault.

**Pros.** Enables retrieval queries the wikilink graph cannot answer today (the contradictory-claims case is the headline example). Surfaces relationships the operator has implicitly noticed but not explicitly typed, making them visible to the Cartographer and Writer profiles. Forward-compatible with future agentic retrieval — Cartographer's gap reports could surface "contradictory work on this draft's thesis" by query rather than by walk.

**Cons.** Frontmatter schema growth has cost — templates need updating, the Linter needs a check for `relation_type:` values outside the vocabulary, dashboards need to surface the new field. The taxonomy may be wrong-shaped for knowledge work and require iteration. Most damagingly: if the operator does not consistently set `relation_type:` on new links, the field becomes noisy (some links typed, most untyped) and queries return incomplete answers, eroding trust in the field.

**When to implement.** When two conditions hold together: (1) the vault contains ≥ 200 claim notes with ≥ 500 inter-claim wikilinks, *and* (2) the operator notices themselves wanting to query "find work that contradicts X" or "find work similar in shape to X" and resorting to manual backlink walks. Below that, the cost of adopting and maintaining the field exceeds the value.

**Why not earlier.** A typed-link field is one of the easiest schema additions to *add* and one of the hardest to *retire*. Adding it pre-density commits the operator to setting it on every new link forever; the cost is borne immediately, the value is borne only at density. Defer until the value side has arrived.

**Prerequisites.** A claim-note template update to include `links:` as a structured list with `target:` and `relation_type:` per entry (or a wikilink-extension Obsidian plugin if the convention is to type links inline). A Linter check that flags `relation_type:` values outside the vocabulary. A dashboard surface (likely in the [open-questions dashboard](../dashboards/open-questions.md) or a new one) that lists `contradictory` links as candidates for resolution. A migration plan for existing untyped links — accept that they stay untyped, do not retrofit.

**What this is not.** Not a replacement for wikilinks — typed and untyped wikilinks coexist; typed is opt-in. Not a substitute for MOCs — MOCs provide topic-level grouping; relation-types provide pair-level semantics. Not the full PARNESS taxonomy — Memoria starts with two values and earns more by usage. Not agent-assigned — the operator types the link when they make it; the agent does not infer `relation_type:` retroactively (that would re-introduce the agent-judgment-on-canonical surface that the autonomy boundary refuses).

## Coder lane experiment loop

Today the Coder profile executes one task per dispatch: a script change, a test addition, a fixture build. Iteration — "edit, run tests, keep if better, revert otherwise" — runs at human pace, with the operator pressing the loop manually. When a task has a stable scalar success criterion (a passing test suite, a runtime benchmark, a coverage threshold), the operator's per-iteration judgment is doing no work the metric isn't already doing. The loop is a candidate for unattended execution.

The pattern, adapted from Chen et al. 2026 (*Toward Autonomous Long-Horizon Engineering for ML Research*) and Karpathy Autoresearch: a lane-bounded skill — `coder-experiment-loop` — reads a `code-experiment` card (a variant of the existing `code` card with a `success_metric:` field naming the test, benchmark, or measurement), then runs propose → test → keep-if-improved → revert-otherwise for up to N iterations or until a budget cap. Outputs land in `40-workbench/03-code/<project>/experiments/<run-id>/`. When the budget exhausts, the skill posts a single summary card to `awaiting-review` containing the best variant, the diff against the starting point, and the metric trajectory. The operator reviews the summary and decides whether the best variant promotes into the project's working code.

The autonomy is the inner loop. The gate is the operator's review of the summary. The autonomy boundary in [01-architecture/why-no-autonomous-synthesis.md](../01-architecture/why-no-autonomous-synthesis.md) explicitly admits this pattern in the Coder lane — see [§"Scope: these boundaries apply to synthesis"](../01-architecture/why-no-autonomous-synthesis.md#scope-these-boundaries-apply-to-synthesis) for the precondition table.

**Pros.** Removes operator attention from a step where the scalar metric is already doing the keep/revert judgment. Maps cleanly onto a well-understood pattern (Chen 2026's AiScientist shows 0.903 → 0.982 AUC in 23 hours of unattended improvement on MLE-Bench Lite). The summary-card structure preserves auditability — the operator sees the trajectory, not just the endpoint. Compounds with the [overnight loop](#the-overnight-loop-proactive-discovery-pattern): nightly experiment runs alongside nightly discovery runs.

**Cons.** Picking the wrong success metric is the dominant failure mode — the loop optimizes the metric, not the underlying goal, so a poorly-specified `success_metric:` produces a winner that game-played the test. Iteration count can balloon API spend if the budget isn't tight (the cost is N model calls × N test runs per card). The summary-card review is genuinely harder than reviewing a single change — the operator now has to evaluate a trajectory, which is a different cognitive task.

**When to implement.** When the operator notices themselves running the same "edit code → run test → revert if worse" cycle in the Coder lane more than ~10–20 times per project, *and* the cycle has a stable scalar success criterion that existed before the cycle started (a fixture, a test, a benchmark — not a judgment-call disguised as a metric). Below those triggers, ordinary Coder execution with operator-in-the-loop is cheaper and the autonomy gain isn't paying for the infrastructure.

**Why not earlier.** The pattern requires the Coder lane to be in routine use and to have accumulated enough task variety that "code with a scalar success criterion" is a recognizable subset of its work. Before that, the loop is a hammer for a problem the operator hasn't measured yet — and a hammer whose failure mode (game-playing a poorly-specified metric) is invisible until the operator reads the summary and notices.

**Prerequisites.** A `code-experiment` card type (variant of `code` with `success_metric:`, `budget_iterations:`, `budget_cost_usd:`). A `coder-experiment-loop` skill bound to the Coder lane, with the policy MCP permitting writes only to `40-workbench/03-code/<project>/experiments/<run-id>/`. A summary-card template (best variant, diff, metric trajectory, iteration log). A cron entry or on-demand trigger; the loop is not strictly overnight-bound, but overnight is the natural cadence.

**What this is not.** Not synthesis autonomy — [01-architecture/why-no-autonomous-synthesis.md](../01-architecture/why-no-autonomous-synthesis.md) remains unchanged for every lane other than Coder. Not auto-promotion — the best variant goes to `awaiting-review`, not into the project's working code; the operator's review state is still the gate. Not metric-design autonomy — the operator specifies `success_metric:` on the card, the agent does not propose it mid-run. Not exempt from the canonical-zone deny rule — the policy MCP continues to block writes outside the experiment-run directory regardless of how many iterations the loop has executed.

## Tournament ranking for triage

When the inbox holds ten or more candidate sources for the same research direction, the operator's triage step is "rank these and discard the bottom half." Today the Cartographer profile orders candidates by relevance score; the operator reads from the top until they stop finding value, then closes the rest. This works at moderate inbox volume but flattens late-tier candidates into a long tail of near-identical scores that the operator either over-reads or under-reads.

The pattern, adapted from Gottweis et al. 2025 (AI co-scientist) and Baek et al. 2025 (ResearchAgent): when the candidate set crosses ~10, the Cartographer (or a small dedicated triage skill) runs a **pairwise tournament** — each candidate is compared against several others on a small fixed rubric (relevance to the active research direction, methodological novelty, citation context overlap with existing claim notes), and the win-loss record produces a triage-aware ordering distinct from the raw relevance score. The operator reads the tournament top-K, not the raw top-K.

**Pros.** Pairwise comparisons handle ties and ambiguity better than scalar ranking — a candidate that wins narrowly against eight peers reads differently from one that ties with five. Compatible with the existing triage flow: the tournament's output is just a reordered candidate list, not a new card type or new state. The rubric is editable per project.

**Cons.** Tournament rounds multiply API calls (n^2 in the naïve case, n log n with bracket structure). The rubric is another piece of LLM-judgment surface that drifts and needs review — adding a tournament step is adding another place where the agent's bias can leak into operator decisions. Most importantly, it does not change what the operator does — they still pick from a top-K — so the value is only in *how good the top-K is*, which is hard to measure without a separate ground-truth.

**When to implement.** When two conditions hold together: (1) the daily inbox exceeds ~10 candidates per research direction often enough to be a recurring friction, *and* (2) the operator notices themselves discarding from the middle of the candidate list rather than from the bottom — a signal that scalar ranking is mis-ordering. Below either threshold, the existing Cartographer ordering is sufficient.

**Why not earlier.** The tournament pattern only pays off when there are enough candidates that order matters and enough variation that scalar relevance is too coarse. Pre-overnight-loop ([overnight loop](#the-overnight-loop-proactive-discovery-pattern) above), the inbox does not consistently produce that volume; the tournament would be a heavy mechanism for a light problem.

**Prerequisites.** A stable [research-directions.md](../04-workflows.md) so the tournament rubric has something to score relevance *against*. The overnight loop running long enough to produce the candidate volume that justifies tournament cost. A budget for ~n^2 inference calls per triage session (small per session, recurring across nights).

**What this is not.** Not a quality gate — the tournament reorders, it does not decide. Not autonomous keep/revert — the operator still chooses which candidates to ingest; the tournament only changes the order they are read in. Not the same pattern as the full AI co-scientist evolve loop, which scales test-time compute on the *output* (hypothesis quality); here the tournament only orders *inputs* to the human's existing triage decision.

## Cross-project reading as personal AgentRxiv

Memoria assumes one vault per operator. Within that vault, the operator typically runs multiple concurrent projects (a thesis, a side-investigation, a literature scope for a future grant). Today the Researcher in project A does not read the synthesis produced by the Writer in project B unless the operator explicitly links across projects — claim notes are searched globally, but project-scoped drafts are read project-locally. Cross-project synthesis happens by the operator's hand, not by the agent's discovery.

The pattern, adapted from Schmidgall & Moor 2025 (AgentRxiv): treat the vault's `40-workbench/01-projects/*/drafts/` collection as the operator's personal "preprint pool." When a profile in project A runs a discovery or synthesis step, it also queries draft synthesis notes in projects B and C with a `cross_project: true` flag, and surfaces relevant prior thinking as inspiration context — clearly tagged as foreign so the worker creates a stub or wikilink rather than mistaking the foreign draft for in-project material. The original draft is never modified; the cross-project read is one-way and audit-logged.

**Pros.** Makes the vault's grown-over-time synthesis useful across projects without manual cross-linking. Captures the AgentRxiv empirical finding (agents reading prior agent work outperform isolated agents) inside a single-user setting. Surfaces "I wrote about this two projects ago" connections that the operator currently has to remember by hand.

**Cons.** Cross-project reads broaden the context any single agent step takes in — more tokens, slower, more places for noise to enter. Project-scoped privacy assumptions (a Project A draft might cite an embargoed source that should not surface in Project C) need to be encoded as policy, not as habit. Without good tagging discipline, the same idea can spawn parallel drafts in two projects, which the cross-project loop then re-cross-reads.

**When to implement.** When the operator has at least two concurrent projects active for ≥ 8 weeks each *and* notices "I already wrote about this" moments that the agent did not surface. Below that, the absence is not felt — the cost of building the cross-project read pattern outweighs the savings.

**Why not earlier.** Single-project use does not exercise the pattern. Building cross-project reading before there's a second populated project is building for an empty corpus; the read paths return nothing, and the policy/privacy edges are untested.

**Prerequisites.** The "research object" schema discussed in [cross-vault read-only retrieval](#cross-vault-read-only-retrieval) above — the same normalization that lets a foreign vault be queried also lets a foreign project be queried. An `embargo:` or `confidentiality:` field in project draft frontmatter so cross-project reads can be filtered. A clear UI cue in agent output that flags foreign-project context distinctly from in-project context (matching the foreign-vault pattern).

**What this is not.** Not a global re-merge — projects remain distinct top-level folders. Not bi-directional write — the cross-project loop is read-only, just like the foreign-vault pattern. Not a substitute for explicit cross-project wikilinks — the operator still owns canonical cross-project synthesis; the loop only surfaces candidates for that linking.
