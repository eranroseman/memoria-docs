# Workflow 15: Scope

**Group.** Downstream (stage workflow, added 2026-05)
**Goal.** Map the corpus for a project: identify what's ready, what's thin, what's missing. Produce a corpus map the operator can use to decide whether to write now or read more first.

## Pipeline position

First downstream stage; precondition for [#16 Frame](16-frame.md).

## Steps

1. The operator creates a project folder: `40-workbench/01-projects/<project>/` with a `brief.md` describing the deliverable, audience, length, and framing constraints.
2. A `scope` card opens on the project. Cartographer claims it.
3. Cartographer runs `scope-project` (see [02-profiles.md](../02-profiles.md#lane-permissions-matrix)): retrieves all claim and reference notes matching the brief topic; computes cluster density, recency distribution, source diversity; identifies adjacent topics with thin coverage.
4. The output is written to `40-workbench/01-projects/<project>/corpus-map.md` as a structured report.
5. The card moves to `awaiting-review`. Operator reads the corpus map and decides: proceed to [#16 Frame](16-frame.md), or pause to read more (loops back to [#1 Zotero capture](01-zotero-capture.md) / [#3 Discover](03-discover.md)).

## Owners

Cartographer executes the scope retrieval (read-only across the vault, writes only to the corpus-map artifact). Human owns the "is this ready to write?" decision.

## Card lifecycle

`ready` if watcher-created (file-system watcher on new `brief.md`) OR `pending` if operator-created via `Memoria: new project` (operator transitions to `ready` after reviewing the auto-populated brief fields) → `active` (Cartographer claims) → `awaiting-review` with `corpus-map.md` written → operator reads, decides: `approved` (advances to [#16 Frame](16-frame.md)) or `rejected` (operator typically spawns new cards in [#3 Discover](03-discover.md) to read more first; the original closes with `outcome: superseded` when the revision card opens, or `outcome: discarded` if the scope is abandoned).

## Command

`hermes -p memoria-cartographer run scope-project --project <project-name>`.

## Why not skip straight to drafting

Operators almost always overestimate corpus readiness — they remember the notes they've recently touched, not the gaps they haven't. The corpus map surfaces the gaps. A scope step that takes ten minutes saves writing a chapter on thin evidence.

## Related

- **Profile:** [profiles/cartographer.md](../profiles/cartographer.md)
- **Next workflow:** [#16 Frame](16-frame.md)
- **Umbrella workflow:** [#8 Writing and drafting](08-writing-drafting.md)
