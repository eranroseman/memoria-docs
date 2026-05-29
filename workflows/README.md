---
mode: explanation
audience: operator
topic: workflows
---

# Workflows

This document describes the operational workflows in Memoria across three sections:

1. **The two pipelines** — upstream (source → durable knowledge) and downstream (knowledge → written output), plus cross-cutting maintenance.
2. **The named workflows** — concrete operations grouped as upstream, downstream, or maintenance. See the [workflow inventory](#workflow-inventory) for the full list. Per-workflow detail lives in the [upstream/](upstream/), [downstream/](downstream/), and [maintenance/](maintenance/) subfolders.
3. **The Command Palette** — how a human triggers any workflow from inside Obsidian.

For each workflow, "owner" means *who has decision authority* — not just who executes. Hermes can execute many steps; the human owns the decisions that determine whether the work moves forward.

## What's in this document

**Pipelines and stages** — [The two pipelines](#the-two-pipelines) (upstream, downstream, maintenance), [The Kanban is the workflow's event bus](#the-kanban-is-the-workflows-event-bus) (why workflows are state-machine paths, not procedures).

**Who does what** — [Role × stage matrix](#role--stage-matrix) (per-pipeline matrices), [Workflow inventory](#workflow-inventory) (the 18 named workflows).

**Triggering work** — [How the human triggers work: the Command Palette](#how-the-human-triggers-work-the-command-palette), [Research directions (steering input)](#research-directions-steering-input).

**Discipline** — [Cross-cutting ownership rules](#cross-cutting-ownership-rules), [Default operating model](#default-operating-model), [Anti-patterns](#anti-patterns).

## The two pipelines

Memoria has two distinct pipelines. They share the vault but operate independently — most days the human works in one or the other, not both.

Both are **conditional paths, not fixed sequences.** A note or project advances only as far as its nature warrants: most sources never become claims, most claims never reach promotion, and stages like `discuss`, `revise`, and `archive` fire only when their precondition holds. The diagrams below show the *canonical* path. A `[bracketed]` stage (currently only `arrange`) is a **skippable intermediate** — one bypassed *mid-path* on an otherwise-complete pipeline, distinct from a stage a note simply never reaches.

### Upstream: source → durable knowledge

The upstream pipeline transforms raw sources into stable, well-linked claim notes ready to feed writing. It has nine stages:

```text
find ──► ingest ──► classify ──► discuss ──► distill ──► link ──► corroborate ──► promote ──► archive
```

| Stage | Goal | Primary owner | Output |
| --- | --- | --- | --- |
| **Find** | Find candidates worth adding to the corpus. | Librarian | `10-inbox/03-candidates/` |
| **Ingest** | Create the right paper note in the right folder with enrichment. | Librarian | `20-sources/01-papers/` (or items / entities) |
| **Classify** | Promote agent-proposed classification into canonical metadata. | Human | Paper note with `lifecycle: current` |
| **Discuss** | Read and think the paper note through via Socratic conversation. No artifact yet; the human's understanding sharpens. | Human (Socratic profile, write-denied) | A `discuss` card closes; the human is ready to distill |
| **Distill** | Distill the now-discussed source into 1–3 atomic claim notes. | Writer (human-authored) | `30-synthesis/01-claims/` with `maturity: seedling` |
| **Link** | Cross-reference the claim, add to MOC, link related claims. | Writer / human | Claim with `maturity: budding` |
| **Corroborate** | Multi-source support; well-linked; ready for canonization. | Human | Claim with `maturity: evergreen` |
| **Promote** | Move evergreen claim to the reference layer. | Human | `30-synthesis/02-reference/` (reference-note) |
| **Archive** | Preserve superseded material without deleting it. | Human | `95-archive/` |

**Note on stage granularity.** `find`, `ingest`, `classify`, `discuss`, `distill`, `promote`, and `archive` each correspond to a *named workflow* in the [inventory](#workflow-inventory) below (Find → Ingest → Classify → Discuss → Distill → Promote → Archive). `link` and `corroborate` are *maturity transitions* within the life of a claim note — they don't move files between folders, they just change `maturity` from `seedling` to `budding` to `evergreen`. The folder transitions happen at `promote`.

**Why `discuss` is a stage.** Making discussion board-visible is what answers "which paper notes have I actually thought about?" — a `discuss` card opens when classify completes and closes when the human writes a claim, so a pile-up is the signal that discussion is slipping. Full rationale in [Discuss](upstream/discuss.md#why-the-card-auto-closes-only-on-human-action).

### Downstream: knowledge → output

The downstream pipeline transforms stable claim notes into written deliverables. Seven stages, plus one skippable intermediate (`arrange`):

```text
assess ──► frame ──► [arrange] ──► outline ──► draft ──► verify ──► revise ──► export
```

| Stage | Goal | Primary owner | Output |
| --- | --- | --- | --- |
| **Assess** | Map the corpus for a project (what's ready, thin, missing) and decide whether to proceed or read more first. | Mapper (`scope-project`); human decides | `40-workbench/01-projects/<project>/map/corpus-map.md` |
| **Frame** | Generate 2–3 competing project framings before drafting; choose one. | Writer (with `counter-outline`, scratch-only); Socratic (with `lens-reading`) for lens-based explorations | `40-workbench/01-projects/<project>/framing/*.md` |
| **Arrange** *(optional)* | Arrange relevant claim notes spatially on a Canvas; identify gaps. | Human | `40-workbench/01-projects/<project>/canvas/{section}.canvas` |
| **Outline** | Linearize the framing (and Canvas if used) into a heading scaffold. | Human (Writer-assisted) | Outline block in draft note |
| **Draft** | Write the prose with inline citekeys. | Human (Writer support) | `40-workbench/01-projects/<project>/drafts/{chapter}.md` |
| **Verify** | Trace every substantive claim back to a claim note; flag unsupported claims. | Verifier; human decides per claim | `[!verification]` callout at top of draft + gap cards in upstream queue |
| **Revise** | Address verification findings; close the gap-loop or accept softened claims. | Human | Updated draft; closed `[!verification]` callout |
| **Export** | Run Pandoc to produce the final, frozen artifact (Word / PDF / HTML). | Coder (Pandoc); human decides to ship | `50-deliverables/` (the deliverable) |

The downstream pipeline is spread across [Write](downstream/write.md) (the umbrella) plus [Assess](downstream/assess.md), [Frame](downstream/frame.md), [Verify](downstream/verify.md), and [Revise](downstream/revise.md) for the new stages.

**Why assess and frame are stages.** Both judgments — "do I have what I need?" and "what argument am I making?" — happen anyway; making them stages turns them from private deliberation into board-visible work, so a card stalled in `assess` for two weeks is the same processing-debt diagnostic the upstream pipeline uses. Full rationale in [Assess](downstream/assess.md#why-not-skip-straight-to-drafting) and [Frame](downstream/frame.md#why-this-is-a-stage-and-not-a-pattern-inside-write).

**Why verify and revise are stages.** Promoting `cite-check` and the gap-loop to stages surfaces broken cites weeks before export instead of the day before — failed traces spawn upstream gap cards, and export is blocked until the `revise → re-verify` loop closes. Full rationale in [Verify](downstream/verify.md#why-verify-is-a-stage-instead-of-part-of-export) and [Revise](downstream/revise.md#why-revise-is-a-stage-instead-of-just-keep-editing).

**Arrange remains optional.** Arranging claims spatially on a Canvas is a useful step for chapter-sized work (8–15 claim notes). It bridges frame (which framing wins) and outline (linearize). For shorter pieces, framing leads directly into outline without it. See the [Canvas → Draft sub-workflow in Write](downstream/write.md#canvas--draft-sub-workflow).

### Maintenance (cross-cutting)

Maintenance runs continuously, not as part of either pipeline. It includes lint sweeps, merge / split / prune passes, and archival reviews. These produce reports and structural adjustments; they don't transform notes from one stage to another. (Session logging — the per-session audit trail every action writes — is a system *mechanism* rather than a workflow; it has no card and no state transitions. See [operations/session-logging.md](../operations/session-logging.md).)

### Lint is cross-cutting, not a stage

The pipeline diagrams above show *transforming* stages — each takes a note in one form and produces it in a new form. Lint does not fit this pattern. The Linter runs *against* every stage — schema checks during ingest, draft-block validation during classify, broken-link checks during distill, link-density and orphan checks before promote — but it produces **reports**, not new notes. It surfaces issues; it does not advance any single note's lifecycle.

See [Lint](maintenance/lint.md) for the operational details and weekly ritual; the Linter row in the [role × stage matrix](#role--stage-matrix) for per-stage behavior; and [profiles/linter.md](../profiles/linter.md) for the thresholds and auto-fix policy.

### Why two pipelines, not one

- **Different cadences.** Upstream runs daily (new sources arrive constantly); downstream runs per writing project (months apart).
- **Different artifacts.** Upstream produces knowledge (claim notes, reference notes); downstream produces documents (drafts, exports, deliverables).
- **Different judgment.** Upstream judgment is "is this true and worth filing?"; downstream judgment is "does this argument hold together?". Mixing them produces drafts that re-litigate filing decisions instead of building arguments.
- **Most sources never reach downstream.** Many paper notes provide supporting evidence (cited at the draft stage) without becoming the *originator* of a claim note. Knowing the upstream pipeline is independent of the downstream one makes this fine.

## The Kanban is the workflow's event bus

Workflows are not scripted procedures; they are **event sequences on the Kanban**. Each workflow describes which cards open, which profiles claim them, and which state transitions advance them. The board isn't storage that workflows happen to write to — it's the substrate workflows are *made of*. Three properties fall out of this:

- **Profiles don't call each other.** When Librarian finishes an ingest, it doesn't invoke Verifier; it sets the card's exit state. Verifier picks up cards in that state through the dispatcher (see [board/README.md dispatch interval](../board/README.md#dispatch-interval)). The handoff is the state change, not a message.
- **The human is one event source among several.** Human action via the [command palette](../surfaces/command-palette.md) creates cards. So do cron triggers (scheduled tasks), file-system watchers (PDF drops), and git hooks (draft commits). Each one is a card creation; the dispatcher routes them identically.
- **Workflows can be paused, resumed, or retried because the state is on the board.** A worker that fails mid-task leaves the card to be re-dispatched (returned to `ready`); the next dispatch picks it back up. A worker that succeeds completes the card to its exit state; the next workflow's trigger fires on that state change.

This is why the per-workflow `Card lifecycle` lines in each workflow file name explicit state transitions rather than function calls — workflows ARE state-machine paths, and reading them as paths makes the architecture's failure modes legible (a stuck card is a stuck workflow).

## Role × stage matrix

The matrix is split by pipeline. Upstream stages are human-pace and per-source; downstream stages are project-pace and per-deliverable. Two separate matrices keep each readable.

### Upstream

| Role | Find | Ingest | Classify | Discuss | Distill | Promote |
| --- | --- | --- | --- | --- | --- | --- |
| **Human** | Sets corpus boundary | Resolves ambiguity | **Lead** | Owns the thinking | Authors the claim | **Lead** |
| Librarian | **Lead** | **Lead** | Provides metadata | Read-only | Seed inputs only | Carry evidence forward |
| Mapper | Read-only | Read-only | Read-only | Comparative-brief on new sources | Read-only | Read-only |
| Socratic | No claim | No claim | No claim | **Lead** (human invokes; write-denied) | No claim | No claim |
| Writer | Reads context | Reads context | Reads context | No claim (human switches profiles) | **Lead** | Revise after review |
| Verifier | No claim | No claim | No claim | No claim | Filing-time similarity-check | Read-only |
| Linter | Structure check | Schema check | Dry-run validation | Surfaces stale discuss queue | Catch broken links | Gate before canon |

### Downstream

| Role | Assess | Frame | Outline | Draft | Verify | Revise | Export |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Human** | Decides readiness | Chooses the framing | Writes the outline | Writes the prose | Reviews findings | **Lead** | Decides to ship |
| Librarian | Source context | Read-only | Read-only | Read-only | Picks up gap cards | Read-only | Read-only |
| Mapper | **Lead** (via `scope-project`) | Read-only | Read-only | Read-only | Read-only | Read-only | Read-only |
| Socratic | No claim | Optional (human switches profiles) | No claim | No claim | No claim | No claim | No claim |
| Writer | Reads corpus map | **Lead** (via `counter-outline`, scratch-only) | **Lead** | **Lead** | Read-only (Verifier executes `cite-check`) | Revises (human-led) | Read-only |
| Verifier | Read-only | Read-only | Read-only | Read-only | **Lead** (cite-check, claim-trace) | Tracks open findings | Confirms verify-clean |
| Coder | No claim | No claim | No claim | No claim | No claim | No claim | **Lead** (runs Pandoc) |
| Linter | Validates corpus-map shape | Validates framing folder | Catch broken links | Catch broken links | Catch lingering findings | Catch unrevised flags | Validates export readiness |

**Reading the matrix.** **Lead** marks the single role that *executes* each stage — for Discuss, that's **Socratic** in the run-the-questioning sense, though the human owns the thinking itself. (This is deliberately distinct from the **Primary owner** column in the pipeline stage tables above, which names *decision authority* — for Discuss, the **Human**. A stage can have an agent Lead and a human owner.) The **Human** row is the person running the system: Lead for the stages no agent executes — Classify, Promote, and Revise — and elsewhere it names the judgment they keep while an agent runs the mechanics. The **Coder** appears downstream only (its work — running the Export pipeline — has no upstream counterpart).

Rule (both pipelines): **finding and filing** are Librarian work; **mapping the corpus** is Mapper work; **questioning without producing** is Socratic work; **durable writing and prose** are Writer work; **verifying claims trace** is Verifier work; **running the export pipeline** is Coder work; **structural safety and audit-trail housekeeping** are Linter work. The human owns the judgments — what counts as ready, which framing wins, whether a gap matters — that none of these profiles is allowed to make.

**Maintenance is cross-cutting, not a stage** — it's omitted from both matrices (see [Maintenance](#maintenance-cross-cutting)). The Linter is its primary owner; the Verifier runs the retraction sweep.

**No Orchestrator row.** Routing is encoded in lane-overrides and Kanban dispatch — see [profiles/README.md](../profiles/README.md#routing-without-an-orchestrator).

## Workflow inventory

Workflows are listed in execution order within each pipeline. Refer to a workflow by name (e.g., "Discuss workflow", "Verify workflow") — there are no numeric IDs.

### Upstream (8)

| Workflow | Goal | Main owner |
| --- | --- | --- |
| [Zotero Capture](upstream/zotero-capture.md) | Make Zotero the source of truth for references and PDFs. | Human |
| [Find](upstream/find.md) | Find related papers, tools, people, venues worth adding. | Librarian surfaces; human confirms |
| [Ingest](upstream/ingest.md) | Create the right note in the right folder with enrichment. | Librarian; human resolves ambiguity |
| [Classify](upstream/classify.md) | Promote agent-proposed fields into canonical metadata. | Human |
| [Discuss](upstream/discuss.md) | Sharpen thinking on a classified source via Socratic conversation before writing a claim note. | Human (Socratic profile, write-denied) |
| [Distill](upstream/distill.md) | Distill literature into durable claims. | Human authors; Writer drafts |
| [Promote](upstream/promote.md) | Turn stable claim notes into reference pages. | Human; Linter may flag candidates |
| [Archive](upstream/archive.md) | Preserve deprecated material without deleting it. | Human only |

### Downstream (8)

| Workflow | Goal | Main owner |
| --- | --- | --- |
| [Assess](downstream/assess.md) | Map the corpus for a project; decide if it's ready to write. | Mapper (`scope-project`); human decides |
| [Frame](downstream/frame.md) | Generate 2–3 competing project framings before drafting; commit to one. | Writer (with `counter-outline`, scratch-only); Socratic (with `lens-reading`); human chooses |
| [Write](downstream/write.md) | Build manuscripts from synthesized knowledge (umbrella). | Human; Hermes assists |
| [Verify](downstream/verify.md) | Trace draft claims back to claim notes; spawn gap cards for failed traces. | Verifier; human decides |
| [Revise](downstream/revise.md) | Close the gap-loop: address verification findings before export. | Human |
| [Export](downstream/export.md) | Run Pandoc to produce the final, frozen deliverable. | Coder runs Pandoc; human decides to ship |
| [Query](downstream/query.md) | Get a cited synthesis from the vault. | Writer / Librarian; human verifies |
| [Code](downstream/code.md) | Treat code as a research output with provenance. | Human; Coder scaffolds (external agent implements) |

### Maintenance (2)

| Workflow | Goal | Main owner |
| --- | --- | --- |
| [Lint](maintenance/lint.md) | Keep structure, links, queues healthy. | Linter; human decides on fixes |
| [Refactor](maintenance/refactor.md) | Keep notes atomic and remove duplication. | Hermes identifies; human decides |

### Workflow ↔ stage mapping

Most workflows share a name with the pipeline stage they implement — Find, Ingest, Classify, Discuss, Distill, Promote, Archive (upstream) and Assess, Frame, Verify, Revise, Export (downstream) — so the shared name *is* the mapping. The exceptions:

- **`link` and `corroborate`** are *maturity transitions*, not workflows — they advance a claim note's `maturity` (`seedling` → `budding` → `evergreen`) without a named operation.
- **`arrange`, `outline`, and `draft`** have no standalone workflow; the **Write** umbrella owns them.
- **Zotero Capture** feeds the pipeline (before `find` / `ingest`) but isn't itself a stage.
- **Query** and **Code** are downstream *branch* workflows — off the main `assess → export` spine.
- **Lint** and **Refactor** are *cross-cutting* maintenance, not pipeline stages.

## How the human triggers work: the Command Palette

The board state machine, profile contracts, and lane policies are all *passive* — they describe what *can* happen. Workflows still need a *trigger*. For the daily human, that trigger is the Obsidian Command Palette.

The authoritative catalog of `Memoria:` commands (capture, processing, interactive retrieval, project, maintenance, lens-reading) and their implementation lives in [surfaces/command-palette.md](../surfaces/command-palette.md). This section describes the control flow that fires whenever any command is invoked.

### Control flow

Every command follows the same six-step shape:

1. **User invokes the command** from the palette.
2. **Plugin reads frontmatter** from the active note. No business logic; just field extraction.
3. **Plugin calls the Hermes API** with a small JSON payload. No persistent state.
4. **The Hermes API translates** the call into one or more MCP tool invocations.
5. **MCP servers** validate, record, and dispatch. Policy MCP checks permissions; tasks MCP updates the board.
6. **Audit log is appended.** The decision and result are durable.

See [architecture/control-plane.md](../architecture/control-plane.md) for the layer architecture, and [surfaces/command-palette.md](../surfaces/command-palette.md) for per-command implementation (QuickAdd / Templater / Hermes API / agent-client).

If a step fails, the failure is visible at exactly one layer — the plugin shows a status notice, the Hermes API returns an HTTP error, or the MCP returns a `deny` / `dry_run` payload. There is no place for a silent failure to hide.

### Hotkey discipline

Most commands are palette-only — `Cmd-P → M → <2–3 letters>` is the primary input mode. A small set of high-frequency commands earn dedicated hotkeys or [Commander](../plugins/optional/cmdr.md) buttons; see the [recommended Commander set in command-palette.md](../surfaces/command-palette.md#setting-up-the-bindings) for the recommended top five. Hotkey real estate is scarce; everything outside that set stays palette-only.

### What this section is not

- **Not a plugin spec.** The TypeScript implementation and HTTP endpoint schemas live outside the design — there is no custom Memoria HTTP code. Command dispatch goes through Hermes's built-in API (`hermes gateway`), and vault read/write goes through the `obsidian-local-rest-api` community plugin. See [architecture/control-plane.md](../architecture/control-plane.md) for the layer architecture and the Hermes MCP reference for the wire formats.
- **Not the only trigger.** Cron jobs, the discovery loop (see [roadmap/future-directions.md](../roadmap/future-directions.md#the-discovery-loop)), and Hermes-side `delegate_task` all also fire workflow steps. The Command Palette is the *human-facing* channel; the others are scheduled or agent-driven.

## Research directions (steering input)

`00-meta/research-directions.md` is a single human-edited Markdown file that tells Hermes what to weight. It is not a rule (`AGENTS.md`) or a vocabulary (`schema-decisions.md`) — it's a *steering* file. The agent reads it at session start and uses it to prioritize discovery queries and flag relevance during classify.

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
- **In `find`** the Librarian weights candidate generation toward priorities and listed authors.
- **In `classify`** the human weights `_proposed_classification` proposals against listed projects and topics. (The Librarian already proposed the proposed fields; weighting them against priorities is a human judgment.)
- **In `draft`** the Writer cites it as the active research agenda — answer notes that don't address any listed priority are flagged for human review.
- **In `scope-project`** the Mapper treats the priorities list as a relevance signal when computing the corpus map.

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
- **Coding agents own code execution.** The Coder scaffolds and coordinates; the implementation may be delegated to Kilocode, Aider, Claude Code, or the human depending on the task.
- **Nothing canonical is overwritten automatically.** The agent may propose, draft, or flag, but human review is the promotion gate.

## Default operating model

Memoria runs on rhythms layered by frequency. The skeleton cadence: **daily** capture and inbox skim; **twice a week** classify partial paper notes and run discovery; **weekly** the [ritual](maintenance/lint.md#weekly-ritual); **per-project** draft from `30-synthesis/01-claims/`, arrange in Canvas, and export via Pandoc. What that feels like hour-to-hour:

**Pre-morning, on phone over coffee (optional).** If overnight cron produced anything worth pushing — a retry threshold hit, a drift alarm, a substantive ingest summary — Telegram pushed a notification before the human opened the vault. Example: *"Overnight: 3 sources ingested, 12 link suggestions ready, 1 retry threshold hit on card-2026-05-26-042 (Librarian timeout fetching DOI for Tanaka 2024)."* The human glances, notes anything blocking, deals with it at the desk. If nothing pushed, nothing demands attention before opening the vault — that's the design.

**Morning glance (5–10 minutes).** Open the vault. The Human workspace appears by default (`Cmd-1`). Glance at [Daily Health](../dashboards/daily-health.md): is any section red? Today's queue, drift signals, lane health, cron status. Most days nothing is red, and Daily Health closes again. If link suggestions accumulated overnight from Librarian's enrichment work, bulk-approve the ones that look right via `Memoria: approve all link suggestions` (see [command-palette.md](../surfaces/command-palette.md#maintenance-3-commands)). For retries flagged in the Telegram push, drop to CLI: `hermes kanban show <card-id>` to inspect, fix the underlying issue, `hermes kanban unblock <card-id>` to release it back to `ready` (re-dispatch resets the retry count). Total time: under ten minutes unless something demands attention.

**Reading session (1–2 hours, when scheduled).** Switch to the Reading & Processing workspace (`Cmd-2`). Open [`discuss-queue.md`](../dashboards/discuss-queue.md): which paper note is ripest for processing? Read the source with the `[!brief]` callout in mind. When ready to process, ask Socratic about the active note (`Cmd-P → Memoria: ask about this note`) — the ACP pane opens on the right with the Socratic profile, which is architecturally write-denied. The conversation runs; the human writes the claim note themselves in `30-synthesis/01-claims/` (in their own words, in the left pane) as the conversation progresses. Save. The git hook fires; Librarian's enrichment runs overnight; the link suggestions appear in tomorrow's morning glance.

**Walking (whenever a thought hits).** Telegram: `/fleeting <thought>` drops the text into `10-inbox/01-fleeting/` with a timestamp. The thought is captured; the human doesn't have to remember it through to the next desk session. It surfaces in tomorrow's [`discuss-queue.md`](../dashboards/discuss-queue.md) (or the weekly fleeting-triage step) for action. Source-URL capture works the same way: paste the link, get a confirmation, the actual ingest happens overnight.

**Writing session (project work).** Switch to the Drafting workspace (`Cmd-3`). Open the draft. The `[!verification]` callout at the top shows Verifier's last claim-trace report. Write. The Writer ACP pane is available on the right if local critique is wanted, but it's optional — most drafting happens in the human's head, with the pane silent. On significant edits, save and `git commit`. Verifier picks up the draft automatically ([Verify](downstream/verify.md)). Gap cards from Verifier appear in tomorrow's morning queue. If the gap loop suggests more reading before the draft can stabilize, switch back to a reading session.

**Friday ritual (90 minutes).** The [weekly ritual](maintenance/lint.md#weekly-ritual). Top to bottom in `weekly-review.md`.

**Evening (passive).** If the Linter's weekly drift sweep ran on schedule, Telegram pushes a short confirmation: *"Drift report for the week is ready."* If verdict is `PASS` (green), the human glances and ignores. If `REVIEW` or `FAIL`, the message is the signal to open [`drift-watch.md`](../dashboards/drift-watch.md) next desk session. The push exists so the human knows the sweep happened — silence would be indistinguishable from a missed cron run.

**What the human does NOT do:**

- Manage agent state by hand. The Kanban dispatches; the human approves or redirects via card transitions.
- Re-decide policy questions. The lane-overrides and the policy MCP encode them once.
- Hunt for files. Folders encode lifecycle; search finds the rest.
- Switch between mental models for "what tool am I using." One vault, one keyboard, one set of Memoria commands.

If after three months of use the human's mouse hand barely moves and they've stopped consciously tracking which workspace they're in, the rhythm is right. The board surfaces what's stalled; the dashboards surface what's overdue.

## Anti-patterns

- **Skipping classification.** Letting auto-proposed classification become canonical without review. Symptom: drift between what the vault claims and what the literature actually says.
- **Distilling without source citation.** Claim notes that don't trace to a paper note. Symptom: the vault remembers claims it cannot defend.
- **Promoting before stability.** Moving a note to `30-synthesis/02-reference/` before it's evergreen. Symptom: reference pages that keep changing.
- **Treating archive as delete.** Archived notes are preserved for traceability; they should not be removed.
- **Running find without a corpus boundary.** Generates noise that never gets classified. Symptom: `10-inbox/03-candidates/` grows without bound.
- **Crossing pipelines mid-task.** Mixing upstream (classify, distill) work with downstream (draft, export) work in the same session. Each pipeline has its own rhythm; mixing them produces drafts that re-litigate filing decisions.

<!-- memoria-nav -->

---

[← Previous: Linking patterns](../vault/linking-patterns.md)

[Next: Zotero Capture →](upstream/zotero-capture.md)
