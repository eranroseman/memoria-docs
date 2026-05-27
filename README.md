# Memoria design documents

Memoria is a research operating system that turns sources into durable knowledge through explicit states, specialized Hermes profiles, and a Kanban board that preserves review, retries, and handoffs.

This folder is the canonical design document set.

## How to read this

Read the spine docs in order if you are new. Jump to the section that matches your task if you are not. The **Mode** column follows the [Diátaxis][diataxis] framework: *Explanation* (concepts and rationale), *Reference* (precise contracts and tables), *How-to* (operational recipes), *Tutorial* (learning by doing).

[diataxis]: https://diataxis.fr/

### Spine documents

| # | Document | Mode | Audience | What it covers |
| --- | --- | --- | --- | --- |
| 00 | [Vision](00-vision.md) | Explanation | Anyone | Purpose, design goals, naming rationale, what Memoria is and is not. |
| 01 | [Architecture](01-architecture.md) | Explanation | System designer | The three-layer model: Kanban board, Hermes workers, vault. Full depth in [01-architecture/](01-architecture/). |
| 02 | [Profiles](02-profiles.md) | Reference + Explanation | Operator configuring Hermes | Seven Hermes profiles, their permissions, commands, skills, MCPs, and delegation. |
| 03 | [Board and states](03-board.md) | Reference | Operator | Kanban states, worker lanes, review gate, board schema fields. |
| 04 | [Workflows](04-workflows.md) | How-to | Daily user | Stage-gated pipelines (9 upstream stages, 8 downstream + optional Canvas). Per-workflow detail in [04-workflows/](04-workflows/). |
| 05 | [Notes, folders, linking](05-notes-folders.md) | Reference | Daily user | Note types, folder roles, templates, linking patterns. |
| 06 | [Operator surfaces](06-surfaces.md) | Explanation + Reference | Daily user | Four surface types (persistent, modal, inline, ambient); per-type detail in [06-surfaces/](06-surfaces/). |
| 07 | [Implementation roadmap](07-roadmap.md) | How-to | Implementer | Phased plan, MVS, graduated start. Subsection detail in [07-roadmap/](07-roadmap/). |

### Subfolders

| Folder | Mode | What it holds |
| --- | --- | --- |
| [profiles/](profiles/) | Reference + Explanation | Seven per-profile **design summaries** (researcher, cartographer, socratic, writer, verifier, coder, linter) covering each profile's mission, boundary with adjacent profiles, and load-bearing design decisions. The full runtime prompts (SOUL.md files) live separately at `.memoria/profiles/memoria-<name>/SOUL.md` in the starter vault — the design summary points there for the runtime contract. |
| [dashboards/](dashboards/) | Reference + Explanation | Eleven per-dashboard **design summaries** (index, board-state, drift-watch, fleet-observability, weekly-dashboard, audit-log, reading-pipeline, process-queue, open-questions, schema-hygiene, skill-lifecycle) covering each dashboard's mission, boundary with adjacent dashboards, and load-bearing design decisions. The runtime Dataview queries live at `00-meta/05-dashboards/<name>.md` in the starter vault. |
| [reference/](reference/) | Reference | Terse reference docs the spine links to for detail: [glossary.md](reference/glossary.md), [policy-mcp.md](reference/policy-mcp.md), [profile-compilation.md](reference/profile-compilation.md) (**deferred**), [schema.md](reference/schema.md), [command-palette.md](reference/command-palette.md), [linking-patterns.md](reference/linking-patterns.md), [design-system.md](reference/design-system.md), [computational-toolbox.md](reference/computational-toolbox.md). |
| [rationale/](rationale/) | Explanation | The WHY behind load-bearing design choices: [computational-methods.md](rationale/computational-methods.md) (LLM-vs-deterministic boundary), [pattern-provenance.md](rationale/pattern-provenance.md) (borrow / adapt / ignore table), [coder-external-agent.md](rationale/coder-external-agent.md) (two-agent boundary). |
| [operations/](operations/) | How-to | Operational concerns: [failure-modes.md](operations/failure-modes.md) (Detect/Fix/Verify recipes), [obsidian-plugins/](operations/obsidian-plugins/) (per-plugin configuration, 20+ files). |
| **(runtime, in [memoria-vault](https://github.com/eranroseman/memoria-vault))** | Runtime artifact | The 11 dashboards live at `00-meta/05-dashboards/` in the starter vault. The 15 note templates live at `00-meta/01-templates/`. The runtime SOUL.md prompts for the seven profiles live at `.memoria/profiles/memoria-<name>/SOUL.md`; the linter's structural detectors live at `.memoria/profiles/memoria-linter/M-detectors.md` alongside its SOUL.md. |
| [01-architecture/](01-architecture/) | Explanation | Spillover from the architecture spine: [on-disk-layout.md](01-architecture/on-disk-layout.md), [memory-tiers.md](01-architecture/memory-tiers.md), [control-plane.md](01-architecture/control-plane.md), [capability-stack.md](01-architecture/capability-stack.md), [why-no-autonomous-synthesis.md](01-architecture/why-no-autonomous-synthesis.md). |
| [04-workflows/](04-workflows/) | How-to | One file per workflow (18 total). The spine has the narrative + role-matrix + inventory; each file has the steps + owners + commands. |
| [06-surfaces/](06-surfaces/) | Reference + How-to | Per-surface-type detail: [persistent.md](06-surfaces/persistent.md) (dashboards), [modal.md](06-surfaces/modal.md) (workspaces), [inline.md](06-surfaces/inline.md) (callouts), [ambient.md](06-surfaces/ambient.md) (status bar). |
| [07-roadmap/](07-roadmap/) | How-to + Explanation | Phased plan detail: [timeline.md](07-roadmap/timeline.md), [deployment-options.md](07-roadmap/deployment-options.md), [secret-management.md](07-roadmap/secret-management.md), [sync-and-coordination.md](07-roadmap/sync-and-coordination.md), [skill-governance.md](07-roadmap/skill-governance.md), [standard-cron-tasks.md](07-roadmap/standard-cron-tasks.md), [future-directions.md](07-roadmap/future-directions.md), [autonomy-progression.md](07-roadmap/autonomy-progression.md), [success-metrics.md](07-roadmap/success-metrics.md), [design-tensions.md](07-roadmap/design-tensions.md), plus [decisions/](07-roadmap/decisions/) (18 ADRs) and [pilots/](07-roadmap/pilots/). |
| [tutorials/](tutorials/) | Tutorial | Step-by-step walkthroughs that take you from zero to a working setup. Currently: [01-set-up-from-zero.md](tutorials/01-set-up-from-zero.md). |

## Core idea, in one paragraph

The board (Kanban) is the control plane. The workers (Hermes profiles) are the execution layer. The vault (Obsidian folders) is the durable knowledge store. Cards on the board carry state; profiles claim cards within their lane and execute work; outputs land in the vault only after the human review state changes to `approved`. This keeps workflow state, execution, and final knowledge separate but connected through explicit handoffs.

## Glossary

If you hit an unfamiliar term mid-document — *task packet, lane-override file, verdict band, trust score, hybrid pattern, method class, restrictive skill, …* — look it up in [reference/glossary.md](reference/glossary.md). About 40 terms grouped by domain (system, board, policy MCP, deployment, computational methods, pipeline stages, note types, future-direction terms). The most-linked-from terms also have inline glossary references at their first-mention site in the spine docs.
