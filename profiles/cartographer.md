# Cartographer — design summary

**Runtime contract.** Full prompt and operational details live at `.memoria/profiles/memoria-cartographer/SOUL.md` in the starter vault.

## Mission

Cartographer is the lens on what the operator already has. It produces scope reports, gap reports, cluster density maps, and comparative briefs over the existing corpus. The defining trait is **read-only across canonical content**: Cartographer never proposes new sources, never composes new claims, never edits literature notes. Its value is in giving the operator a view of the corpus that would take hours to assemble by hand.

## What this profile is not

- **Not Researcher.** Researcher fetches new sources from the outside world; Cartographer maps what's already there. They share retrieval tooling but Cartographer cannot reach external APIs at all (`external_api_policy: blocked`).
- **Not Writer.** Cartographer produces maps; Writer produces arguments. A corpus-map says "you have 23 claim notes on JITAI receptivity, recency tilted 2024–2025." It does not say "you have enough to write" — that judgment is the operator's.
- **Not Verifier.** Both are read-only, but they serve different concerns. Verifier traces claims to sources (provenance check). Cartographer surveys clusters and gaps (corpus structure).
- **Not a chat surface for free-form questions.** Cartographer outputs are structured artifacts (`corpus-map.md`, `gap-report.md`, `[!brief]` callouts) with cited inputs, not conversational responses. For ad-hoc questions about the corpus, the operator switches to Socratic.

## Design decisions

- **Read-only by policy, even for scratch.** Cartographer writes only to `40-workbench/01-projects/*/corpus-map.md`, `gap-report.md`, `comparative-briefs/`, and `cluster-maps/`. Project scratch only — nowhere else. This protects against accidental canonical pollution from corpus-mapping operations (e.g., a stray write to `30-synthesis/` while computing a cluster).
- **Deterministic ML layer with LLM narrative composition.** Cartographer's value is the *reproducibility* of the maps — HDBSCAN cluster identification, embedding similarity, recency aggregation are all deterministic. The LLM only composes prose over those deterministic outputs. The pattern is documented as the canonical hybrid in [rationale/computational-methods.md](../rationale/computational-methods.md).
- **Output is a map, not an argument.** Every Cartographer artifact declares its inputs in frontmatter (`sources:` — which folders, what date range, which `qmd` index version). The operator can rerun and get an identical map; they can audit which slice of the corpus was scanned.
- **Comparative-brief drives the `[!brief]` inline callout.** Cartographer's only direct presence outside dashboards and project scratch is the comparative-read callout that appears at the top of new literature notes. It's the design's answer to "how does a new source relate to what I already have?" without requiring the operator to read the whole corpus first.

## Permissions and commands

Folder permission matrix lives in [02-profiles.md](../02-profiles.md#folder-permission-matrix); the runtime contract (full command list, deterministic/hybrid annotations per command, exit conditions) lives in the SOUL.md.

## Related

- Workflows: [15-scope](../04-workflows/15-scope.md), [11-merge-split-prune](../04-workflows/11-merge-split-prune.md), [02-ingest §`comparative-brief`](../04-workflows/02-ingest.md)
- Pilot: [E1 Open Notebook](../07-roadmap/pilots/E1-open-notebook.md) — active experiment using Open Notebook as the LLM back-end for narrative composition in `comparative-brief`.
- Method class: [rationale/computational-methods.md](../rationale/computational-methods.md) — Cartographer is the canonical hybrid profile.
- Reference: [reference/computational-toolbox.md](../reference/computational-toolbox.md) — the embedding, clustering, and search primitives Cartographer relies on.
