# 07 — Implementation roadmap

This document is the phased plan for taking Memoria from design to running system. It assumes the design choices in [00-vision.md](00-vision.md) through [06-surfaces.md](06-surfaces.md) are settled.

The roadmap is ordered so each phase produces a working, useful slice. You should not need to complete phase 4 before phase 1 is paying off.

## How this folder is organized

The narrative spine (minimum viable system, graduated start, expansion-threshold rule, what to defer, success metrics) stays here. The substantive content — timeline, decisions, deployment, future directions, pilots — lives in dedicated files under [`07-roadmap/`](07-roadmap/).

| File | What it covers |
|---|---|
| [timeline.md](07-roadmap/timeline.md) | Week-by-week ramp from MVS to production; the six implementation phases (naming, vault structure, profiles, board, pilot corpus, scale) |
| [deployment-options.md](07-roadmap/deployment-options.md) | Four deployment patterns (A, A+, B, C); secondary-device patterns; per-device install sets |
| [secret-management.md](07-roadmap/secret-management.md) | `.env` vs Bitwarden Secrets Manager; when not to centralize |
| [sync-and-coordination.md](07-roadmap/sync-and-coordination.md) | The 5–15 second sync window; `.agent-lock` discipline; bib watcher |
| [skill-governance.md](07-roadmap/skill-governance.md) | Skill state machine (intake → archived); registry; onboarding checklist; bridge-to-dedicated graduation |
| [standard-cron-tasks.md](07-roadmap/standard-cron-tasks.md) | The four standard scheduled tasks; `cron_mode` migration order |
| [design-tensions.md](07-roadmap/design-tensions.md) | Five tensions that won't resolve cleanly (autonomy/control, schema strictness/flexibility, etc.) |
| [future-directions.md](07-roadmap/future-directions.md) | Overnight loop, propagation debts, LLM-judge gate, retry reflection, literate code-notes, open-design integration, and more — each with adopt-when triggers |
| [autonomy-progression.md](07-roadmap/autonomy-progression.md) | The 12 within-boundary autonomy patterns from the [paper survey](../rationale/pattern-provenance.md), organized as a 5-layer dependency roadmap. Cross-cuts `future-directions.md`. |
| [success-metrics.md](07-roadmap/success-metrics.md) | Time-from-capture-to-claim, promotion rate, review backlog, orphan rate, reuse rate |
| [decisions/](07-roadmap/decisions/) | 21 architecture decision records (ADRs); see [decisions/README.md](07-roadmap/decisions/README.md) for the index |
| [pilots/](07-roadmap/pilots/) | Active pilots with explicit rollback criteria. Currently: [E1 — Open Notebook as comparative-brief back-end](07-roadmap/pilots/E1-open-notebook.md) |

## Minimum viable system

If the full setup feels heavy, start with this subset. It delivers most of the long-term value with a fraction of the configuration overhead.

| Layer | Minimum viable | Defer until needed |
| --- | --- | --- |
| **Tools** | Zotero + Better BibTeX, Obsidian, Hermes (terminal only), Git | Kanban board, ACP plugins, Scite, ASReview, K-Dense skills |
| **Folders** | `00-meta/`, `10-inbox/`, `20-sources/01-literature/`, `30-synthesis/01-permanent/`, `30-synthesis/02-wiki/` | items, entities, projects, drafts, deliverables, archive |
| **Templates** | `source-note.md`, `claim-note.md` | All others (11 more) |
| **Schema fields** | `type`, `triage_status`, `maturity`, `topic`, `projects`, `added` | `pub_status`, `full_text_reviewed`, `_draft_classification`, `_enrichment`, MOCs |
| **Workflow** | Ingest → triage → claim note | discover, enrich, MOCs, code artifacts, Canvas, deliverables |
| **Dashboard** | One Dataview query for triage debt | The full weekly dashboard |
| **Profile** | Mode-based Hermes (one tool, per-run mode) | Four-profile, six-profile |

**The rule:** add complexity only when you feel the absence of the missing piece. If you're not running into a problem, you don't need the solution yet.

## Implementation paths: graduated start

Don't start with seven profiles if seven profiles will block you from starting. The system supports three graduated configurations:

| Path | Profiles | When to use |
| --- | --- | --- |
| **Mode-based** | One Hermes, with per-run `mode: ingest \| synthesize \| verify \| maintain`. | Single tool, single config. Simplest start. Good for exploration; weakest safety properties. |
| **Four-profile minimal** | Researcher + Writer + Verifier + Linter (see [02-profiles.md](02-profiles.md)). | Solo workflow at low volume. Cartographer and Socratic capabilities fold into Researcher and Writer respectively, with care. |
| **Seven-profile canonical** | The full design: Researcher, Cartographer, Socratic, Writer, Verifier, Coder, Linter. | When volume makes the operator's review queue a bottleneck, or you want strong architectural separation between thinking (Socratic, write-denied) and producing (Writer, review-gated). |

The migration path is *up*: start with mode-based or four-profile; promote to the canonical seven once the bottleneck makes the cost worth paying.

## Expansion threshold discipline

A general rule for adding structure:

> Add new structure only when the corpus justifies it. Introduce new MOCs, subagents, dashboards, vocabularies, or note types only when recurring work makes the added complexity worthwhile.

Concrete thresholds:

- **New MOC**: when a topic has ≥ 10 source notes and ≥ 3 claim notes that share it.
- **New subagent / profile**: when an existing profile is consistently overloaded *and* the overload comes from a distinct concern.
- **New dashboard**: when a recurring question is answered by ad-hoc grep more than 3 times.
- **New vocabulary or note type**: when at least 5 existing notes are forcing themselves into a wrong-shaped slot.

Below these thresholds, prefer to live with the friction.

## What to defer

Things that *sound* useful but should not be in the initial build:

- Cross-vault federation (multi-vault sync) — adds complexity disproportionate to benefit.
- Auto-generated reference notes from claim clusters — too easy to produce polished-but-untrusted content.
- Tree-search experiment automation — wrong domain for a knowledge-work system.
- A web UI — Obsidian is the UI.
- Real-time multi-user collaboration — Memoria is single-user by design.

If any of these become genuinely needed, they belong in a future major version, not in the initial roadmap.

## Next

- For the timeline and the six implementation phases: [07-roadmap/timeline.md](07-roadmap/timeline.md).
- For the open and resolved architectural decisions: [07-roadmap/decisions/](07-roadmap/decisions/).
- For deployment, sync, and secret management: [07-roadmap/deployment-options.md](07-roadmap/deployment-options.md), [07-roadmap/sync-and-coordination.md](07-roadmap/sync-and-coordination.md), [07-roadmap/secret-management.md](07-roadmap/secret-management.md).
- For the day-one workflows: [04-workflows.md](04-workflows.md).
- For the profile contracts to drop into Hermes: [profiles/](profiles/).
- For a reminder of the architecture this roadmap builds: [01-architecture.md](01-architecture.md).
