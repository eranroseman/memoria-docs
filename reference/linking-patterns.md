# Linking patterns

The canonical reference for vault linking discipline: link types, the full rule set, the expected cross-link graph by note type, MOC creation thresholds, and slug-collision resolution. The summary lives in [05-notes-folders.md](../05-notes-folders.md#linking-patterns); this document is the full reference.

## Link types

| Type | Syntax | Use | Direction |
| --- | --- | --- | --- |
| `citekey-link` | `[[mamykina2010sense]]` | Link a claim to its source note. | `claim-note` → `source-note` |
| `concept-link` | `[[receptivity-decreases-under-high-cognitive-load]]` | Link related claims and build the knowledge graph. | `claim-note` ↔ `claim-note` |
| `moc-link` | `[[jitai-design-moc]]` | Place a note within a navigational hub. | Note → `moc` |
| `entity-link` | `[[mamykina-lena]]` | Connect people, organizations, venues, tools. | Any → `entity-*` / `item-note` |
| `agent-cross-link` | Generated in note body | Surface citation-graph and related-note candidates. | Proposed by agent, confirmed by human. |

## Linking rules

- Link for usefulness, not completeness.
- Every `claim-note` should trace to at least one `source-note` citekey.
- Every `source-note` should eventually connect to at least one relevant `moc`.
- Use concept links only when the relationship would matter in a later reading or writing session.
- Prefer one strong MOC link over many weak generic links.
- Preserve provenance direction: claims point to evidence, not the other way around.
- Never treat agent-generated cross-links as canonical until they have been reviewed.
- If a note becomes an orphan, add a MOC link or relevant concept links before considering it complete.
- Do not link a `source-note` to a `claim-note` as if they were peers; the claim should stand on its own with the source note as support.

## Required patterns

- **Source notes** — include `Cites` and `Cited by` sections when relevant, with agent-proposed links reviewed during triage.
- **Claim notes** — include source links in body text and keep a `Connections` section for meaningful conceptual neighbors.
- **MOCs** — use frontmatter `moc:` links and keep the MOC body focused on overview, curated entries, and gaps.
- **Reference notes** — link outward to the core source notes and claim notes that define the concept, method, or domain.

## Cross-link graph

The expected link topology — what links to what — by note type:

```text
source-note ({citekey})
  ↔ person-note               (author)
  ↔ venue-note                (published in)
  ↔ org-note                  (author affiliation)
  ↔ item-note                 (uses or evaluates)
  ↔ source-note (other)       (cites / cited by)

claim-note
  → source-note               (citekey, supports the claim)
  ↔ claim-note (other)        (concept link — supports, contradicts, refines)
  → moc                       (frontmatter moc:)

person-note
  ↔ source-note               (authored)
  ↔ org-note                  (affiliated with)

org-note
  ↔ person-note               (members, affiliates)
  ↔ source-note               (institutional output)

item-note
  ↔ source-note               (cited in or evaluating)
  ↔ org-note                  (maintained by)
  ↔ code-note                 (used in project)

code-note
  → source-note               (motivating evidence)
  → project-note              (project it belongs to)
  → item-note                 (dependencies)
```

Single arrows (`→`) point from the source note carrying the link. Double arrows (`↔`) are bidirectional — both notes carry pointers.

**Co-authorship is not a direct link.** Two `person-note`s linked because they co-authored a paper would duplicate the source note's author list and force every paper to spawn new edges. Instead, co-authorship is expressed *indirectly* through shared `source-note` authorship: querying "papers by A" ∩ "papers by B" surfaces the relationship without a maintained edge. Advisor/advisee or other persistent relationships *are* direct edges — they outlast any single paper.

## MOC creation thresholds

MOCs require enough mass to be worth navigating. Building them too early creates maintenance burden for no payoff.

| Threshold | Rule |
| --- | --- |
| **Do not create at init** | A new vault should not have MOCs. Dataview queries and search cover navigation until clusters form. |
| **Topic MOC** | Create when a topic has **≥ 15–20 notes** between `20-sources/01-literature/` and `30-synthesis/01-permanent/`. |
| **Domain MOC** | Create when you have **at least 3 topic MOCs** that share a domain. A domain MOC with one child topic is just a topic MOC with extra steps. |
| **Child MOC** | Split a parent when a branch has **> 20 claim notes and > 10 source notes**. |
| **Project MOC** | One per dissertation chapter or formal review. Create on project start; archive on project close. |

These thresholds prevent MOC sprawl (too many empty hubs) and MOC underuse (one giant MOC that should be three).

## Slug collision resolution

The naming conventions are designed so collisions are rare. When they do occur:

| Collision | Resolution pattern |
| --- | --- |
| Two researchers with same name | Append disambiguator: `smith-john-iowa.md` vs `smith-john-stanford.md` |
| Two labs with similar names | Use full institution: `hci-lab-iowa.md` vs `hci-lab-cmu.md` |
| Person and organization share surname | Person keeps bare slug; org gets `-org` suffix: `mamykina.md` vs `mamykina-org.md` |
| Same package on different registries | Registry prefix: `pypi-requests.md` vs `npm-requests.md` |
| Repo and person collide | Repos always have `{owner}-` prefix; no collision possible by construction |

When a new note triggers a collision, the linter flags it for human disambiguation — never silently merges or renames.
