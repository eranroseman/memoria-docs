---
mode: how-to
audience: operator
topic: workflows
---

# Find

**Group.** Upstream
**Goal.** Find related papers, tools, people, or venues worth adding.

## Steps

1. The Librarian runs forward citation discovery, backward citation discovery, or concept-driven search.
2. Returns candidates to `10-inbox/03-candidates/`.
3. Human reviews the queue.
4. Human includes candidates in Zotero (which feeds back into ingest) or excludes them with a reason.

## Owners

Hermes for candidate generation. Human for the corpus boundary decision.

## Commands

- `hermes run find --source {citekey} --depth 1`
- `hermes run find --query "..." --limit 20`

## Example

The Librarian wants forward citations from the Mamykina paper. Runs `hermes run find --source mamykina2010sense --depth 1` → returns 18 candidate citations (papers that cite Mamykina) to `10-inbox/03-candidates/` → human reviews the queue weekly → includes 3 in Zotero (which triggers ingest on the next loop), excludes 15 with reasons like `not-empirical` or `wrong-population`.

## Related

- **Profile:** [profiles/librarian.md](../../profiles/librarian.md)
- **Candidate schema:** [ADR-21 shared candidate frontmatter](../../decisions/21-shared-candidate-frontmatter.md)
- **Pre-ingest screening at scale:** [ADR-19 pre-ingest screening](../../decisions/19-pre-ingest-screening.md)
- **Gap cards from Verify** land here as `type: gap-candidate`; see [Verify](../downstream/verify.md). (This is the current distinct type; the shared candidate frontmatter in [ADR-21](../../decisions/21-shared-candidate-frontmatter.md) is a separate, still-proposed unification.)

<!-- memoria-nav -->

---

[← Previous: Zotero Capture](zotero-capture.md)

[Next: Ingest →](ingest.md)
