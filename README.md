---
mode: reference
audience: anyone
topic: general
---

# Memoria design documents

Memoria is a research operating system that turns sources into durable knowledge through explicit states, specialized Hermes profiles, and a Kanban board that preserves review, retries, and handoffs.

This folder is the authoritative design document set. It is organized **by topic** (one folder per concept), with [Diátaxis][diataxis] mode (`explanation`, `reference`, `how-to`, `tutorial`) declared in each file's front-matter rather than in the folder path.

[diataxis]: https://diataxis.fr/

**Three complementary ways in:** read **by topic, in order** (below) if you're new; jump **by mode** if you want a particular *kind* of doc; or scan **the map** for where a given topic lives. They're lenses on the same set, not competing tables of contents.

## Reading by topic — if you're new, read these in order

Each topic folder has a `README.md` with the conceptual overview. The reading order builds the system from the inside out: what it is, then how it's structured, then how it operates.

1. [vision.md](vision.md) — Purpose, design goals, what Memoria is and is not.
2. [architecture/](architecture/) — Three-layer model (board, workers, vault). Why every layer.
3. [board/](board/) — The control plane: states, review gate, handoff schema.
4. [profiles/](profiles/) — The seven Hermes workers: missions, permissions, command catalog.
5. [vault/](vault/) — The vault: folder taxonomy, fifteen note types, frontmatter schema, linking.
6. [workflows/](workflows/) — How work flows through the system: upstream, downstream, maintenance pipelines.
7. [surfaces/](surfaces/) — How the human interacts: persistent, modal, inline, ambient.
8. [roadmap/](roadmap/) — Phased plan, deployment, autonomy progression, design tensions.

## Reading by mode — if you have a specific need

The Diátaxis modes ([explanation, reference, how-to, tutorial][diataxis]) describe what *kind* of writing a doc contains. Every doc declares its mode in front-matter; use this when you want a particular kind of content.

- **Want to try Memoria from scratch?** → [tutorials/](tutorials/).
- **Need a recipe to do something?** → [workflows/](workflows/), [operations/](operations/), [plugins/](plugins/).
- **Looking something up?** → [glossary.md](glossary.md), and the precise reference files inside each topic folder ([board/states.md](board/states.md), [board/card-schema.md](board/card-schema.md), [vault/frontmatter-schema.md](vault/frontmatter-schema.md), [vault/templates.md](vault/templates.md), [vault/linking-patterns.md](vault/linking-patterns.md), [profiles/profile-commands.md](profiles/profile-commands.md), [surfaces/command-palette.md](surfaces/command-palette.md), [architecture/policy-mcp.md](architecture/policy-mcp.md), [architecture/computational-toolbox.md](architecture/computational-toolbox.md), `surfaces/*.md`, `dashboards/*.md`).
- **Want to understand the why?** → each topic's `README.md`, plus [decisions/](decisions/) (ADRs) and the topic-local `why-*.md` files ([architecture/why-no-autonomous-synthesis.md](architecture/why-no-autonomous-synthesis.md), [architecture/why-computational-methods.md](architecture/why-computational-methods.md), [architecture/why-pattern-provenance.md](architecture/why-pattern-provenance.md), [profiles/why-coder-external-agent.md](profiles/why-coder-external-agent.md)).

## The map — what lives where

| Folder | What it holds |
| --- | --- |
| [vision.md](vision.md) | Single-doc top-level explanation. Purpose, naming, what Memoria is and is not. |
| [glossary.md](glossary.md) | ~40 cross-cutting terms grouped by domain. Cross-cutting; lives at root. |
| [architecture/](architecture/) | Three-layer model, capability stack, on-disk layout, control plane, memory tiers. Includes `why-*.md` rationale docs (computational methods, pattern provenance, no autonomous synthesis). |
| [board/](board/) | `README.md` (concept) + `states.md` (state machine, lanes, review gate) + `card-schema.md` (Hermes card fields, metadata overlay, handoff). |
| [profiles/](profiles/) | `README.md` (the seven, lane permissions, delegation, anti-patterns) + `profile-commands.md` (operational command catalog) + per-profile design summaries + `why-coder-external-agent.md`. |
| [vault/](vault/) | `README.md` (folder taxonomy, promotion map, pitfalls) + `templates.md` (15 note types + lifecycles) + `frontmatter-schema.md` (frontmatter, controlled vocab) + `linking-patterns.md`. |
| [workflows/](workflows/) | `README.md` (the two pipelines, role matrix) + `upstream/`, `downstream/`, `maintenance/` (one how-to per workflow, 18 total). |
| [surfaces/](surfaces/) | `README.md` (four surface types) + per-type detail (`persistent`, `modal`, `inline`, `ambient`) + `design-system.md` (visual-style template) + `command-palette.md`. |
| [dashboards/](dashboards/) | `README.md` (Daily Health) + 10 further per-dashboard design summaries — 11 dashboards total. Runtime Dataview queries live at `00-meta/01-dashboards/` in the [starter vault](https://github.com/eranroseman/memoria-vault). |
| [operations/](operations/) | `README.md` + [failure-modes.md](operations/failure-modes.md) (Detect / Fix / Verify recipes). |
| [plugins/](plugins/) | `README.md` (priority-ordered overview) + per-plugin configuration split by lifecycle into `required/` (8), `recommended/` (5), `optional/` (6, includes deployment-conditional and future-migration), plus top-level `plugin-configs-lifecycle.md` and `ui-discipline.md` (Obsidian UI conventions). |
| [decisions/](decisions/) | 22 architecture decision records (ADRs) — 19 with their own file, 3 (ADR-1, 8, 11) index-only — + `_template.md` + `by-topic.md` (secondary index grouping ADRs by topic folder) + `adopt-on-demand-for-reviews.md` (shared rationale for the four deferred systematic-review ADRs 12 / 18 / 19 / 20). Cross-cuts all topics, so it sits at the top level. |
| [roadmap/](roadmap/) | `README.md` (phased plan) + per-section detail (`timeline`, `deployment-options`, `autonomy-progression`, `success-metrics`, `design-tensions`, `future-directions`, `profile-compilation` (deferred), etc.) + `pilots/`. |
| *(cross-cutting reference is distributed)* | Each cross-cutting reference doc lives next to what it explains: [surfaces/command-palette.md](surfaces/command-palette.md), [architecture/policy-mcp.md](architecture/policy-mcp.md), [architecture/computational-toolbox.md](architecture/computational-toolbox.md), [roadmap/profile-compilation.md](roadmap/profile-compilation.md) (**deferred**), [board/card-schema.md](board/card-schema.md), [vault/frontmatter-schema.md](vault/frontmatter-schema.md), [vault/linking-patterns.md](vault/linking-patterns.md), [profiles/profile-commands.md](profiles/profile-commands.md). |
| [tutorials/](tutorials/) | Step-by-step walkthroughs. Currently: [`01-set-up-from-zero.md`](tutorials/01-set-up-from-zero.md). |
| **(runtime, in [memoria-vault](https://github.com/eranroseman/memoria-vault))** | 11 dashboards at `00-meta/01-dashboards/`; 15 note templates at `00-meta/03-templates/`; 10 human-facing reference notes at `00-meta/04-reference/`; SOUL.md prompts at `.memoria/profiles/memoria-<name>/SOUL.md`; Linter detectors at `.memoria/profiles/memoria-linter/M-detectors.md`. |

## Core idea, in one paragraph

The board (Kanban) is the control plane. The workers (Hermes profiles) are the execution layer. The vault (Obsidian folders) is the durable knowledge store. Cards on the board carry state; profiles claim cards within their lane and execute work; outputs land in the vault only after the human review state changes to `approved`. This keeps workflow state, execution, and final knowledge separate but connected through explicit handoffs.

## Front-matter convention

Every `.md` file in this folder carries a YAML header declaring its Diátaxis mode and intended audience:

```yaml
---
mode: explanation        # one of: explanation | reference | how-to | tutorial
audience: operator       # anyone | operator | human | system-designer | implementer | contributor
topic: workflows         # the topic folder this file belongs to
---
```

Mode is what kind of writing this is. Audience is who it's for. Topic mirrors the folder (cross-cutting files at root use `general`). ADRs in [decisions/](decisions/) also carry `id`, `title`, `status`, `date_proposed` per the ADR template.

## Glossary

Unfamiliar term mid-document — *handoff payload, verdict band, trust score, method class, …*? It's in [glossary.md](glossary.md), with a [Disambiguations](glossary.md#disambiguations) section for the words used in more than one sense. The most-linked terms also carry inline references at their first-mention site in the topic READMEs.

<!-- memoria-nav -->

---

[Next: Vision →](vision.md)
