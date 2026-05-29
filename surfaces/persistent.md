---
mode: reference
audience: operator
topic: surfaces
---

# Persistent surfaces: dashboards

Dashboards are the operational view layer — Dataview-rendered notes that surface what is in flight on the board, what is overdue in the vault, and what needs human judgment.

Each dashboard lives in its own file under [`dashboards/`](../dashboards/). The vault implementation places these in `00-meta/01-dashboards/`.

## Dashboards by use frequency

Every dashboard exists in the vault. The question is how often you open it.

| Frequency | Dashboard | Purpose |
| --- | --- | --- |
| **Daily** | [`index`](../dashboards/README.md) | Always-on health monitor. Open before any large operation. |
| **Reading session** | [`discuss-queue`](../dashboards/discuss-queue.md) | "What should I think about today?" — paper notes classified but not yet processed via Socratic. The upstream-discipline dashboard. |
| **Weekly** | [`weekly-dashboard`](../dashboards/weekly-overview.md) | Weekly ritual entry point (see [workflows/README.md](../workflows/README.md) workflow Lint). |
| **Weekly** | [`reading-pipeline`](../dashboards/reading-pipeline.md) | "What to read next?" — papers by processing stage. |
| **Weekly** | [`schema-hygiene`](../dashboards/schema-hygiene.md) | Catch TODO / tmp / draft filenames and leftover junk. |
| **Weekly** | [`drift-watch`](../dashboards/drift-watch.md) | Linter's structural detector findings + [verdict band](../glossary.md#observability-and-verdicts). Open when the lint pass passed but something still feels off. |
| **Per board op** | [`board-state`](../dashboards/board-state.md) | Active cards, review queue, retry watch. |
| **Forensic** | [`audit-log`](../dashboards/audit-log.md) | Policy-MCP write decisions. Open when something feels off. |
| **Planning** | [`open-questions`](../dashboards/open-questions.md) | Research agenda view — surface all explicit Open Questions sections. |
| **Scale-dependent** | [`skill-lifecycle`](../dashboards/skill-lifecycle.md) | Hermes skill registry view. Deferred — maybe later if needed; see [roadmap/future-directions.md](../roadmap/future-directions.md#skill-governance). |
| **Scale-dependent** | [`fleet-observability`](../dashboards/fleet-observability.md) | Cost and reliability trends for the worker fleet. Post-MVS only — see [roadmap/future-directions.md](../roadmap/future-directions.md). |

The "scale-dependent" dashboards are designed to be present from day one but only carry meaningful data once the corpus is large enough that human eyes stop noticing slow regressions. Until then they're inert.

## Design rules for dashboards

- **One decision per query.** Each query should tell the human what to do next, not just what exists.
- **Filter out the boring cases.** Exclude already-promoted, already-archived, or seedling-stage notes from active queues.
- **Sort by oldest first** in maintenance queues — the longest-stalled items are the most urgent.
- **Keep each dashboard short.** If a single dashboard has more than ~10 queries, split it into specialized dashboards.

## Performance discipline

Dataview reads Obsidian's metadata cache. Queries are usually fast, but as the vault grows past a few thousand notes, broad queries become the dominant render cost. These rules keep dashboards responsive without restructuring the vault.

- **Scope queries with `FROM "folder"`** instead of `FROM ""` (vault-wide). Use the narrowest folder that holds the candidates — e.g., `FROM "10-inbox"` for triage queues, `FROM "30-synthesis"` for promotion backlogs.
- **Keep property types stable.** If `lifecycle` is sometimes a string and sometimes a list, sort and filter behave unpredictably. Use the Linter's `safe-and-unambiguous` Templater script to coerce types on every note touch (see [profiles/linter.md](../profiles/linter.md#implementing-safe-and-unambiguous-fixes-via-templater)).
- **Keep frontmatter compact.** Long prose belongs in the note body. Frontmatter is for fields you query.
- **`LIMIT` long lists.** A dashboard rarely needs more than the top 20–30 rows. Add `LIMIT 30` and trust the sort.
- **Reuse queries.** If two dashboards ask the same question, write the query once and embed it from both — duplicate queries scan the same metadata twice.
- **Prefer `TABLE` and `LIST`** over `dataviewjs` for plain frontmatter views. Drop to `dataviewjs` only when you need cross-record aggregation or external file I/O (e.g., reading `audit.jsonl`).

A useful sanity check: open the dashboard, time it, then run the same queries on a folder-scoped version. If the folder-scoped version is twice as fast, the original was the wrong shape.

## Graceful degradation

A query depends on a field, a folder, or a decision that may not exist yet — `candidate-note` requires [Decision 21](../decisions/21-shared-candidate-frontmatter.md), `lane-metric` requires the [fleet-observability aggregator](../roadmap/future-directions.md#fleet-observability), `_proposed_classification` requires the Librarian to have run. A query that throws or returns an unexplained empty table when its dependency is missing is worse than the missing feature itself: it teaches the human to distrust the dashboard.

The discipline:

- **Empty result + explanation beats error.** A `TABLE` that returns zero rows should be paired with a one-line markdown note explaining what would populate it. Example: "Empty until Decision 21 is adopted and a `candidate-note` template exists" (see [weekly-dashboard](../dashboards/weekly-overview.md)).
- **Detect dependency, not absence of data.** A query on a brand-new vault returns zero rows because there is no data yet — that is the right behavior. A query that returns zero rows because the *field doesn't exist anywhere* is a capability gap, not an empty queue. The explanatory note distinguishes the two cases.
- **`dataviewjs` should guard explicit reads.** Queries that load external files (`audit.jsonl`, `state.json`) must handle the missing-file case and render a placeholder, not throw a stack trace into the dashboard.
- **Surface dependencies at the top of the dashboard.** Each dashboard's frontmatter or intro paragraph should name the decisions, files, or aggregators it depends on. If the human already knows what's missing, an empty query isn't a mystery.

This is the dashboards' version of the policy MCP's `dry_run` decision: when the gate isn't met, the system stays useful and explains itself instead of breaking quietly. The cost — a few lines of placeholder prose per dashboard — is trivial relative to the trust it preserves.

## Reference: `00-meta/04-reference/dataview-cheatsheet.md`

For dashboard authors, the vault ships with a Dataview cheatsheet at `00-meta/04-reference/dataview-cheatsheet.md` (see the [vault skeleton](../vault/README.md#vault-skeleton-human-facing-notes)). It contains definitive examples of the patterns above — TABLE, LIST, TASK, FLATTEN, multi-field SORT, folder-scoped FROM, FILTER BY FIELD — each with a one-line "when to use" header and a runnable block.

The cheatsheet is the operational complement to this section. This document says *what discipline* to follow; the cheatsheet shows *what the syntax looks like* when you sit down to write a new dashboard. Update the cheatsheet whenever a new query pattern proves itself useful across two or more dashboards — that's the bar for promotion from "one-off" to "reference."

## What dashboards do not do

- **Not workflow automation.** They surface state; they don't change it.
- **Not the source of truth for review.** Review state is on the board, not in a query result.
- **Not a replacement for the Linter.** Lint reports identify structural issues the dashboard cannot detect (e.g., schema drift, broken links).

## Frontmatter compatibility note

The queries in each dashboard assume the frontmatter conventions in [vault/README.md](../vault/README.md). If you migrate type names or folder paths, update the queries to match — aliases (e.g., keeping `research-wiki` as a deprecated name) help during transition but the queries themselves must reflect the authoritative structure.

<!-- memoria-nav -->

---

[← Previous: Human surfaces](README.md)

[Next: Modal surfaces: workspaces →](modal.md)
