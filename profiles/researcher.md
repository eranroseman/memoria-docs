# Researcher — design summary

**Runtime contract.** Full prompt and operational details live at `.memoria/profiles/memoria-researcher/SOUL.md` in the starter vault.

## Mission

Researcher is the system's intake layer — the profile that decides what enters the vault. It fetches sources, enriches metadata, extracts PDFs, and proposes draft classifications. The defining trait is **optimistic-by-default**: when in doubt, Researcher includes the candidate and proposes the classification, letting Verifier and the operator do the gatekeeping at filing time. This is intentional — the cost of a missing source is invisible (you don't know what you don't have), while the cost of a candidate that gets reviewed and rejected is just one operator decision.

## What this profile is not

- **Not a synthesizer.** Researcher curates evidence; Writer composes claims from that evidence. Researcher never writes to `30-synthesis/01-permanent/` or `30-synthesis/02-wiki/`.
- **Not the gatekeeper.** Verifier is the system's quality bar — Researcher proposes optimistically, Verifier checks conservatively. The asymmetry is the design.
- **Not Cartographer.** Researcher fetches new sources from the outside world; Cartographer maps what already exists in the corpus. They share retrieval tooling (`qmd`) but differ in direction.
- **Not autonomous about classification.** The `_draft_classification` block Researcher writes is *proposal*, not fact. It lives in an HTML comment so it's invisible until the operator or Verifier promotes it.

## Design decisions

- **Optimistic vs conservative classification.** Researcher errs toward proposing classifications even when uncertain. The HTML-commented `_draft_classification` block makes proposals cheap to ignore — operators see them only when they choose to review.
- **Mostly-deterministic core with one LLM step.** Citation graph walks (OpenAlex), metadata enrichment (Crossref / Unpaywall / PubMed), PDF extraction (Marker), classification rule dispatch — all deterministic. The only LLM step is composing the candidate-note relevance description when surfacing to the operator. Keeps the cost surface small relative to the volume of work Researcher does.
- **Highest external API surface in the system.** Researcher is the only profile that talks to OpenAlex, Semantic Scholar, Crossref, PubMed, and Unpaywall regularly. Its lane-override gates external API calls via `external_api_policy: explicit_only`, requiring each skill to declare its API usage explicitly.
- **Ingest is one card per source, not one card per batch.** Researcher's unit of work is the source. A 30-paper ingest produces 30 cards. This keeps retries scoped (one broken PDF doesn't fail the batch), audit entries clean (one source = one trace), and policy decisions atomic.

## Permissions and commands

Folder permission matrix lives in [02-profiles.md](../02-profiles.md#folder-permission-matrix); the runtime contract (full command list, allowed/disallowed folders, exit conditions) lives in the SOUL.md.

## Related

- Workflows: [02-ingest](../04-workflows/02-ingest.md), [03-discover](../04-workflows/03-discover.md), [04-triage](../04-workflows/04-triage.md), [01-zotero-capture](../04-workflows/01-zotero-capture.md)
- ADRs: [19 pre-ingest screening](../07-roadmap/decisions/19-pre-ingest-screening.md), [17 retriever-scout profile](../07-roadmap/decisions/17-retriever-scout-profile.md), [21 shared candidate frontmatter](../07-roadmap/decisions/21-shared-candidate-frontmatter.md)
- Method class: [rationale/computational-methods.md](../rationale/computational-methods.md) — Researcher is on the hybrid side, with deterministic enrichment and an LLM-required classification-proposal step.
