# 04 — Workflows

This document describes the operational workflows in Memoria. It has three layers:

1. **The two pipelines** — upstream (source → durable knowledge) and downstream (knowledge → written output).
2. **The named workflows** — concrete operations grouped as upstream, downstream, or maintenance. See the [workflow inventory](#workflow-inventory) for the full list; numbered IDs are stable across additions. Per-workflow detail lives in [`04-workflows/`](04-workflows/).
3. **The Command Palette** — how a human triggers any workflow from inside Obsidian.

For each workflow, "owner" means *who has decision authority* — not just who executes. Hermes can execute many steps; the human owns the decisions that determine whether the work moves forward.

## The two pipelines

Memoria has two distinct pipelines. They share the vault but operate independently — most days you work in one or the other, not both.

### Upstream: source → durable knowledge

The upstream pipeline transforms raw sources into stable, well-linked claim notes ready to feed writing. It has nine stages:

```text
discover ──► ingest ──► triage ──► process ──► synthesize ──► develop ──► stabilize ──► promote ──► archive
```

| Stage | Goal | Primary owner | Output |
| --- | --- | --- | --- |
| **Discover** | Find candidates worth adding to the corpus. | Researcher | `10-inbox/03-candidates/` |
| **Ingest** | Create the right source note in the right folder with enrichment. | Researcher | `20-sources/01-literature/` (or items / entities) |
| **Triage** | Promote agent-proposed classification into canonical metadata. | Operator | Source note with `triage_status: full` |
| **Process** | Transform the literature note into thinking through Socratic conversation. No artifact yet; the operator's understanding sharpens. | Human (Socratic profile, write-denied) | A `process` card closes; the operator is ready to synthesize |
| **Synthesize** | Distill the now-processed source into 1–3 atomic claim notes. | Writer (human-authored) | `30-synthesis/01-permanent/` with `maturity: seedling` |
| **Develop** | Cross-reference the claim, add to MOC, link related claims. | Writer / human | Claim with `maturity: budding` |
| **Stabilize** | Multi-source support; well-linked; ready for canonization. | Human | Claim with `maturity: evergreen` |
| **Promote** | Move evergreen claim to the reference layer. | Human | `30-synthesis/02-wiki/` (reference-note) |
| **Archive** | Preserve superseded material without deleting it. | Human | `95-archive/` |

**Note on stage granularity.** `discover`, `ingest`, `triage`, `process`, `synthesize`, `promote`, and `archive` each correspond to a *named workflow* below (#3, #2, #4, #14, #5, #6, #12). `develop` and `stabilize` are *maturity transitions* within the life of a claim note — they don't move files between folders, they just change `maturity` from `seedling` to `budding` to `evergreen`. The folder transitions happen at `promote`.

**Why `process` is a stage, not just a pattern.** Triaged literature notes don't change folders during processing; they sit in `20-sources/01-literature/` until the operator writes a claim from them. Without `process` as a stage, the question "which literature notes have I actually thought about?" has no answer the board can surface. With it, `process` cards open automatically when triage completes and close when the operator writes a claim (or marks the source as not yielding one). If `process` cards pile up, processing discipline is slipping — that signal is the whole point of making it a stage. See [#14 Process](04-workflows/14-process.md) for the operator-facing details.

### Downstream: knowledge → output

The downstream pipeline transforms stable claim notes into written deliverables. Eight stages, with one optional intermediate:

```text
scope ──► frame ──► [canvas] ──► outline ──► draft ──► verify ──► revise ──► export ──► deliverable
```

| Stage | Goal | Primary owner | Output |
| --- | --- | --- | --- |
| **Scope** | Map the corpus for a project: what's ready, what's thin, what's missing. Decide whether to proceed or read more first. | Cartographer (`scope-project`); human decides | `40-workbench/01-projects/<project>/corpus-map.md` |
| **Frame** | Generate 2–3 competing project framings before drafting; choose one. | Writer (with `counter-outline`, scratch-only); Socratic (with `lens-reading`) for lens-based explorations | `40-workbench/01-projects/<project>/framing/*.md` |
| **Canvas** *(optional)* | Arrange relevant claim notes spatially; identify gaps. | Human | `40-workbench/04-canvas/{section}.canvas` |
| **Outline** | Linearize the framing (and Canvas if used) into a heading scaffold. | Human (Hermes-assisted) | Outline block in draft note |
| **Draft** | Write the prose with inline citekeys. | Human (Writer support) | `40-workbench/02-drafts/{chapter}.md` |
| **Verify** | Trace every substantive claim back to a claim note; flag unsupported claims. | Verifier; operator decides per claim | `[!verification]` callout at top of draft + gap cards in upstream queue |
| **Revise** | Address verification findings; close the gap-loop or accept softened claims. | Human | Updated draft; closed `[!verification]` callout |
| **Export** | Run Pandoc to produce a Word / PDF / HTML output. | Human (Coder for the pipeline) | Output file |
| **Deliverable** | Final, frozen artifact. | Human | `50-deliverables/` |

The downstream pipeline is spread across [#8 Writing and drafting](04-workflows/08-writing-drafting.md) (the umbrella) plus [#15 Scope](04-workflows/15-scope.md), [#16 Frame](04-workflows/16-frame.md), [#17 Verify](04-workflows/17-verify.md), and [#18 Revise](04-workflows/18-revise.md) for the new stages.

**Why scope and frame are stages.** Without them, the operator goes from "I want to write X" straight to outlining, with no formal step that asks "do I have what I need?" or "what argument am I making?" Both judgments happen anyway — but as private deliberation, not as board-visible work. Promoting them to stages means a project card has to pass through scope (with a corpus map) and frame (with at least one committed framing) before drafting begins. The discipline isn't enforceable architecturally, but the board signal — "this project is sitting in scope for two weeks" — is the same diagnostic the upstream pipeline uses for processing debt.

**Why verify and revise are stages.** `cite-check` was always part of the export step but invisible — operators ran it (or didn't), and the result was an inline report or a forgotten subroutine. Promoting verify to its own stage gives the gap-loop a board state: failed traces spawn upstream gap cards (closing the loop into discovery), and the operator either revises or accepts soft claims through an explicit `revise → re-verify` cycle. Without these stages, "I exported a draft with broken cites" is a real failure mode.

**Canvas remains optional.** Canvas is a spatial argument-map step useful for chapter-sized work (8–15 claim notes). It bridges frame (which framing wins) and outline (linearize). For shorter pieces, framing leads directly into outline without Canvas. See the [Canvas → Draft sub-workflow in #8](04-workflows/08-writing-drafting.md#canvas--draft-sub-workflow).

### Maintenance (cross-cutting)

Maintenance runs continuously, not as part of either pipeline. It includes lint sweeps, merge / split / prune passes, archival reviews, and session logging. These workflows produce reports and structural adjustments; they don't transform notes from one stage to another.

### Lint is cross-cutting, not a stage

The pipeline diagrams above show *transforming* stages — each takes a note in one form and produces it in a new form. Lint does not fit this pattern. The linter runs *against* every stage — schema checks during ingest, draft-block validation during triage, broken-link checks during synthesize, link-density and orphan checks before promote — but it produces **reports**, not new notes. It surfaces issues; it does not advance any single note's lifecycle.

See [#10 Maintenance and linting](04-workflows/10-maintenance-linting.md) for the operational details and weekly ritual; the linter row in the [role × stage matrix](#role--stage-matrix) for per-stage behavior; and [profiles/linter.md](profiles/linter.md) for the thresholds and auto-fix policy.

### Why two pipelines, not one

- **Different cadences.** Upstream runs daily (new sources arrive constantly); downstream runs per writing project (months apart).
- **Different artifacts.** Upstream produces knowledge (claim notes, reference notes); downstream produces documents (drafts, exports, deliverables).
- **Different judgment.** Upstream judgment is "is this true and worth filing?"; downstream judgment is "does this argument hold together?". Mixing them produces drafts that re-litigate filing decisions instead of building arguments.
- **Most sources never reach downstream.** Many source notes provide supporting evidence (cited at the draft stage) without becoming the *originator* of a claim note. Knowing the upstream pipeline is independent of the downstream one makes this fine.

## The Kanban is the workflow's event bus

Workflows are not scripted procedures; they are **event sequences on the Kanban**. Each workflow describes which cards open, which profiles claim them, and which state transitions advance them. The board isn't storage that workflows happen to write to — it's the substrate workflows are *made of*. Three properties fall out of this:

- **Profiles don't call each other.** When Researcher finishes an ingest, it doesn't invoke Verifier; it sets the card's exit state. Verifier picks up cards in that state through the dispatcher (see [03-board.md dispatch interval](03-board.md#dispatch-interval)). The handoff is the state change, not a message.
- **The operator is one event source among several.** Operator action via the [command palette](reference/command-palette.md) creates cards. So do cron triggers (scheduled tasks), file-system watchers (PDF drops), and git hooks (draft commits). Each one is a card creation; the dispatcher routes them identically.
- **Workflows can be paused, resumed, or retried because the state is on the board.** A worker that fails mid-task leaves the card in `retry-needed`; the next dispatch picks it back up. A worker that succeeds moves the card to its exit state; the next workflow's trigger fires on that state change.

This is why the per-workflow `Card lifecycle` lines in each workflow file name explicit state transitions rather than function calls — workflows ARE state-machine paths, and reading them as paths makes the architecture's failure modes legible (a stuck card is a stuck workflow).

## Role × stage matrix

The matrix is split by pipeline. Upstream stages are operator-pace and per-source; downstream stages are project-pace and per-deliverable. Two separate matrices keep each readable.

### Upstream

| Role | Discover | Ingest | Triage | Process | Synthesize | Promote | Maintenance |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Researcher | **Primary** | **Primary** | Provides metadata | Read-only | Seed inputs only | Carry evidence forward | Context only |
| Cartographer | Read-only | Read-only | Read-only | Comparative-brief on new sources | Read-only | Read-only | Read-only |
| Socratic | No claim | No claim | No claim | **Primary** (operator invokes; write-denied) | No claim | No claim | No claim |
| Writer | Reads context | Reads context | Reads context | No claim (operator switches profiles) | **Primary** | Revise after review | Draft support |
| Verifier | No claim | No claim | No claim | No claim | Filing-time similarity-check | Read-only | Retraction sweep |
| Linter | Structure check | Schema check | Dry-run validation | Surfaces stale process queue | Catch broken links | Gate before canon | **Primary** |

### Downstream

| Role | Scope | Frame | Outline | Draft | Verify | Revise | Export |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Researcher | Source context | Read-only | Read-only | Read-only | Picks up gap cards | Read-only | Read-only |
| Cartographer | **Primary** (via `scope-project`) | Read-only | Read-only | Read-only | Read-only | Read-only | Read-only |
| Socratic | No claim | Optional (operator switches profiles) | No claim | No claim | No claim | No claim | No claim |
| Writer | Reads scope output | **Primary** (via `counter-outline`, scratch-only) | **Primary** | **Primary** | Read-only (Verifier executes `cite-check`) | Revises (operator-led) | Runs Pandoc |
| Verifier | Read-only | Read-only | Read-only | Read-only | **Primary** (cite-check, claim-trace) | Tracks open findings | Confirms verify-clean |
| Linter | Validates corpus-map shape | Validates framing folder | Catch broken links | Catch broken links | Catch lingering findings | Catch unrevised flags | Validates export readiness |

Rule (both pipelines): **finding and filing** are Researcher work; **mapping the corpus** is Cartographer work; **questioning without producing** is Socratic work; **durable writing and prose** are Writer work; **verifying claims trace** is Verifier work; **structural safety and audit-trail housekeeping** are Linter work. The operator (human) owns the judgments — what counts as ready, which framing wins, whether a gap matters — that none of these profiles is allowed to make.

**No Orchestrator row.** Routing is encoded in lane-overrides and Kanban dispatch — see [02-profiles.md](02-profiles.md#routing-without-an-orchestrator).

## Workflow inventory

Workflows are numbered stably — additions go at the end of the ID space, not interspersed. Pipeline order is conveyed by the `Group` column, not by row order.

| # | Workflow | Group | Foundation | Goal | Main owner |
| --- | --- | --- | --- | --- | --- |
| 1 | [Zotero capture](04-workflows/01-zotero-capture.md) | Upstream | LLM-wiki | Make Zotero the source of truth for references and PDFs. | Human |
| 2 | [Ingest](04-workflows/02-ingest.md) | Upstream | LLM-wiki | Create the right note in the right folder with enrichment. | Hermes (researcher); human resolves ambiguity |
| 3 | [Discover](04-workflows/03-discover.md) | Upstream | LLM-wiki | Find related papers, tools, people, venues worth adding. | Hermes surfaces; human confirms |
| 4 | [Triage](04-workflows/04-triage.md) | Upstream | Zettelkasten | Promote agent-proposed fields into canonical metadata. | Operator |
| 5 | [Synthesize](04-workflows/05-synthesize.md) | Upstream | Zettelkasten | Distill literature into durable claims. | Human (writer); Hermes drafts |
| 6 | [Promote to wiki](04-workflows/06-promote-wiki.md) | Upstream | LLM-wiki | Turn stable claim notes into reference pages. | Human; Hermes may flag |
| 7 | [Query and answer](04-workflows/07-query-answer.md) | Downstream | LLM-wiki | Get a cited synthesis from the vault. | Hermes; human verifies |
| 8 | [Writing and drafting](04-workflows/08-writing-drafting.md) | Downstream | Both | Build manuscripts from synthesized knowledge. | Human; Hermes assists |
| 9 | [Code artifacts](04-workflows/09-code-artifacts.md) | Downstream | Neither (transactional) | Treat code as a research output with provenance. | Human + coder; Hermes scaffolds |
| 10 | [Maintenance and linting](04-workflows/10-maintenance-linting.md) | Maintenance | LLM-wiki | Keep structure, links, queues healthy. | Hermes (linter); human decides on fixes |
| 11 | [Merge, split, prune](04-workflows/11-merge-split-prune.md) | Maintenance | Zettelkasten | Keep notes atomic and remove duplication. | Hermes identifies; human decides |
| 12 | [Archive](04-workflows/12-archive.md) | Maintenance | — | Preserve deprecated material without deleting it. | Human only |
| 13 | [Session logging](04-workflows/13-session-logging.md) | Maintenance | — | Record agent activity. | Hermes; Git preserves history |
| 14 | [Process](04-workflows/14-process.md) | Upstream | Zettelkasten | Sharpen thinking on a triaged source via Socratic conversation before writing a claim note. | Human (Socratic profile, write-denied) |
| 15 | [Scope](04-workflows/15-scope.md) | Downstream | LLM-wiki | Map the corpus for a project; decide if it's ready to write. | Cartographer (`scope-project`); human decides |
| 16 | [Frame](04-workflows/16-frame.md) | Downstream | Zettelkasten | Generate 2–3 competing project framings before drafting; commit to one. | Writer (with `counter-outline`, scratch-only); Socratic (with `lens-reading`); human chooses |
| 17 | [Verify](04-workflows/17-verify.md) | Downstream | LLM-wiki | Trace draft claims back to claim notes; spawn gap cards for failed traces. | Verifier; operator decides |
| 18 | [Revise](04-workflows/18-revise.md) | Downstream | — | Close the gap-loop: address verification findings before export. | Human |

## How the human triggers work: the Command Palette

The board state machine, profile contracts, and lane policies are all *passive* — they describe what *can* happen. Workflows still need a *trigger*. For the daily operator, that trigger is the Obsidian Command Palette.

The canonical catalog of `Memoria:` commands (capture, processing, interactive retrieval, project, maintenance, lens-reading) and their implementation lives in [reference/command-palette.md](reference/command-palette.md). This section describes the control flow that fires whenever any command is invoked.

### Control flow

Every command follows the same six-step shape:

1. **User invokes the command** from the palette.
2. **Plugin reads frontmatter** from the active note. No business logic; just field extraction.
3. **Plugin calls the local HTTP bridge** with a small JSON payload. No persistent state.
4. **Bridge translates** the call into one or more MCP tool invocations.
5. **MCP servers** validate, record, and dispatch. Policy MCP checks permissions; tasks MCP updates the board.
6. **Audit log is appended.** The decision and result are durable.

See [01-architecture/control-plane.md](01-architecture/control-plane.md) for the layer architecture, and [reference/command-palette.md](reference/command-palette.md) for per-command implementation (QuickAdd / Templater / Hermes API / agent-client).

If a step fails, the failure is visible at exactly one layer — the plugin shows a status notice, the bridge returns an HTTP error, or the MCP returns a `deny` / `dry_run` payload. There is no place for a silent failure to hide.

### Hotkey discipline

Most commands are palette-only — `Cmd-P → M → <2–3 letters>` is the primary input mode. A small set of high-frequency commands earn dedicated hotkeys or [Commander](operations/obsidian-plugins/cmdr.md) buttons; see the [recommended Commander set in command-palette.md](reference/command-palette.md#setting-up-the-bindings) for the canonical top five. Hotkey real estate is scarce; everything outside that set stays palette-only.

### What this section is not

- **Not a plugin spec.** The TypeScript implementation, HTTP endpoint schemas, and FastAPI bridge code live outside the design — see [01-architecture/control-plane.md](01-architecture/control-plane.md) for the layer architecture and the Hermes MCP reference for the wire formats.
- **Not the only trigger surface.** Cron jobs, the overnight loop (see [07-roadmap/future-directions.md](07-roadmap/future-directions.md#the-overnight-loop-proactive-discovery-pattern)), and Hermes-side `delegate_task` all also fire workflow steps. The Command Palette is the *human-facing* surface; the others are scheduled or agent-driven.

## Research directions (steering input)

`00-meta/research-directions.md` is a single human-edited Markdown file that tells Hermes what to weight. It is not a rule (`AGENTS.md`) or a vocabulary (`schema-decisions.md`) — it's a *steering* file. The agent reads it at session start and uses it to prioritize discover queries and flag relevance during triage.

### What goes in it

Four sections, updated weekly (or whenever priorities shift):

```markdown
## Current research priorities
- Deepen the receptivity-detection literature
- Map the JITAI timing-effects debate

## Open questions
- Does cognitive load moderate JITAI timing effects?
- Is there evidence linking sensemaking practices to JITAI design?

## Synthesis gaps
- Topics with literature but no claim notes:
  - therapeutic alliance in digital coaching
  - implementation science for mHealth

## Papers to prioritize in discovery
- Anything citing mamykina2010sense
- Recent CHI / CSCW work on JITAI receptivity
```

### How Hermes uses it

- **At session start** every profile reads `research-directions.md` and surfaces it as context for the session.
- **In `discover`** the researcher weights candidate generation toward priorities and listed authors.
- **In `triage`** the operator weights `_draft_classification` proposals against listed projects and topics. (The Researcher already proposed the draft fields; weighting them against priorities is an operator judgment.)
- **In `draft`** the writer cites it as the active research agenda — synthesis notes that don't address any listed priority are flagged for human review.
- **In `scope-project`** the cartographer treats the priorities list as a relevance signal when computing the corpus map.

### Maintenance

- Update weekly during the dashboard ritual (step 0).
- A stale directions file is worse than none — it pulls the agent toward yesterday's priorities. Delete sections that have shipped.
- Keep it short — under ~200 words per section. The agent reads it as context, not as a long brief.

### Why a file, not just frontmatter or tags

The current schema encodes *what notes are about* (frontmatter). The directions file encodes *what the human is working on right now*. These are different. Topics persist; priorities turn over. Mixing them into a single `tags` or `topics` field loses both.

## Cross-cutting ownership rules

These rules apply across all workflows:

- **Human owns judgment.** Selection, triage, synthesis, promotion, merge / split, and archive decisions are human-owned.
- **Hermes owns bookkeeping.** Type detection, routing, enrichment, candidate generation, cross-link suggestions, linting, and session logging are agent-owned.
- **Coding agents own code execution.** Hermes (coder) scaffolds and coordinates; the implementation may be delegated to Kilocode, Aider, Claude Code, or the human depending on the task.
- **Nothing canonical is overwritten automatically.** The agent may propose, draft, or flag, but human review is the promotion gate.

## Default operating model

A practical baseline cadence:

- **Daily.** Ingest new captures from Zotero. Skim `10-inbox` for synthesis drafts and discovery candidates.
- **Twice a week.** Triage partial source notes. Run discovery on a few recent papers.
- **Weekly.** Run the [weekly ritual](04-workflows/10-maintenance-linting.md#weekly-ritual). Clear unreviewed synthesis. Promote evergreen claim notes.
- **Per-project.** Draft from `30-synthesis/01-permanent/`, arrange in Canvas, write in `40-workbench/02-drafts/`, export via Pandoc.

The board surfaces what's stalled; the dashboards surface what's overdue.

## Anti-patterns

- **Skipping triage.** Letting auto-proposed classification become canonical without review. Symptom: drift between what the vault claims and what the literature actually says.
- **Synthesizing without source citation.** Claim notes that don't trace to a source note. Symptom: the vault remembers claims it cannot defend.
- **Promoting before stability.** Moving a note to `30-synthesis/02-wiki/` before it's evergreen. Symptom: reference pages that keep changing.
- **Treating archive as delete.** Archived notes are preserved for traceability; they should not be removed.
- **Running discover without a corpus boundary.** Generates noise that never gets triaged. Symptom: `10-inbox/03-candidates/` grows without bound.
- **Crossing pipelines mid-task.** Mixing upstream (triage, synthesize) work with downstream (draft, export) work in the same session. Each pipeline has its own rhythm; mixing them produces drafts that re-litigate filing decisions.

## Next

- For per-workflow detail: [`04-workflows/`](04-workflows/) (one file per workflow).
- For note types, folder roles, templates, and linking: [05-notes-folders.md](05-notes-folders.md).
- For dashboards and other operator surfaces that visualize workflow state: [06-surfaces.md](06-surfaces.md).
- For board states that workflows transit through: [03-board.md](03-board.md).
