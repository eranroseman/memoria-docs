# `schema-hygiene` — design summary

**Runtime artifact.** This dashboard ships at `00-meta/05-dashboards/schema-hygiene.md` in the [starter vault](https://github.com/eranroseman/memoria-vault) and runs in Obsidian via Dataview. The summary below covers its design role; the runtime queries live in the vault file.

## Mission

Catch leftover junk that accumulates between rituals: filenames containing `TODO`, `tmp`, `draft`, or `untitled` that the agent or operator forgot to finish or rename. Run after ingest batches or whenever something feels off. The dashboard surfaces; the operator chooses the action per file (rename, finish, archive, delete).

## What this dashboard is not

- **Not [Linter M7 (orphan working files)](../profiles/linter.md).** M7 detects transient *patterns* (`.tmp.*`, `.bak`, editor backups, manual-rename leftovers) outside permitted zones. Schema-hygiene detects *filename keywords* that signal in-progress operator content. Different surface: M7 is automation leftovers; schema-hygiene is operator leftovers.
- **Not Linter's schema-version-mismatch check.** That check (data-hygiene tier, not M-rule) is surfaced in [`drift-watch`](drift-watch.md)'s schema-migration-progress section. Schema-hygiene catches naming-level junk, not version-level migration debt.
- **Not data-quality validation.** Empty frontmatter fields, missing wikilinks, broken references — those are Linter findings surfaced elsewhere. Schema-hygiene's mission is narrower: "files I clearly forgot to finish."

## Design decisions

- **Filename keyword detection, not content scanning.** The check is `contains(file.name, "TODO") OR contains(file.name, "tmp") OR contains(file.name, "draft") OR contains(file.name, "untitled")`. Operator habits create these filenames; surfacing them by name is cheap, deterministic, and false-positive-resistant (operators don't normally name finished notes "untitled-3.md").
- **Whole-vault scan (`FROM ""`).** No folder restriction. The point is to catch junk anywhere, including in canonical zones where it shouldn't be.
- **Sort by `file.mtime` descending.** Recent junk is more actionable than old junk (operator remembers the context). Old junk that's lingered is archive-or-delete territory.
- **Capability-light.** Works on day one with no prerequisites — pure filename inspection.

## Related

- [Linter design summary](../profiles/linter.md) — M7 (orphan working files) is the structural detector counterpart
- [`drift-watch`](drift-watch.md) — schema-version-mismatch and structural drift live there
- [`weekly-dashboard`](weekly-dashboard.md) — recommends opening schema-hygiene as part of the Friday ritual
