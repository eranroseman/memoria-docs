# Workflow 3: Discover

**Group.** Upstream
**Goal.** Find related papers, tools, people, or venues worth adding.

## Steps

1. Hermes (researcher) runs forward citation discovery, backward citation discovery, or concept-driven search.
2. Returns candidates to `10-inbox/03-candidates/`.
3. Human reviews the queue.
4. Human includes candidates in Zotero (which feeds back into ingest) or excludes them with a reason.

## Owners

Hermes for candidate generation. Human for the corpus boundary decision.

## Commands

- `hermes run discover --source {citekey} --depth 1`
- `hermes run discover --query "..." --limit 20`

## Example

The researcher wants forward citations from the Mamykina paper. Runs `hermes run discover --source mamykina2010sense --depth 1` → returns 18 candidate citations (papers that cite Mamykina) to `10-inbox/03-candidates/` → human reviews the queue weekly → includes 3 in Zotero (which triggers ingest on the next loop), excludes 15 with reasons like `not-empirical` or `wrong-population`.

## Related

- **Profile:** [profiles/researcher.md](../profiles/researcher.md)
- **Candidate schema:** [ADR-21 shared candidate frontmatter](../07-roadmap/decisions/21-shared-candidate-frontmatter.md)
- **Pre-ingest screening at scale:** [ADR-19 pre-ingest screening](../07-roadmap/decisions/19-pre-ingest-screening.md)
- **Gap cards from #17 Verify** land here as `type: gap-candidate`; see [#17 Verify](17-verify.md).
