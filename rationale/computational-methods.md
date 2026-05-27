# Computational methods: when Memoria uses LLMs vs classical methods

Every task Memoria performs is classified as **deterministic**, **hybrid**, or **generative**. The class determines the implementation method, the cost profile, and the audit shape. Per-agent specifics live in [profiles/](../profiles/).

## Classification

| Class | Definition | Method | Examples |
| --- | --- | --- | --- |
| **Deterministic** | Has a single right answer derivable from rules, regex, math, or graph algorithms. | Scripts, regex, classical ML, vector search, graph walks. | Citation token extraction, schema-version check, link suggestion ranking, similarity-check, find-duplicates, M1–M8 detectors, dashboard queries. |
| **Hybrid** | A deterministic step narrows the problem; an LLM handles the residual judgment on the narrow result. | Classical pre-filter + LLM on top. | `_draft_classification` proposals (classifier first, LLM fallback), `cite-check` (embedding similarity first, LLM on ambiguous claims), `[!brief]` candidate selection + prose. |
| **Generative** | No fixed output; quality is judged subjectively or requires open-ended composition. | LLM-required. | Socratic conversation, Writer's `draft`, `counter-outline`, comparative-brief prose, synthesis writing. |

**The default placement test:** if there is a regex, a graph algorithm, or a similarity threshold that would produce the right answer most of the time, the task is **deterministic** or **hybrid** — not pure LLM. The LLM enters only where the residual judgment genuinely requires generation.

## Why this classification

A research vault accumulates tasks of two qualitatively different kinds. Memoria treats them differently — and naming the boundary explicitly is what keeps the design honest about cost, determinism, and what it can actually test. Without the classification, every task is reflexively routed to an LLM "because it's text"; with it, the LLM's role contracts to where its judgment is actually load-bearing.

## Decision criteria

When designing a new workflow or skill, choose the method class by asking five questions in order. The first "yes" pins the class:

1. **Can a regex, schema rule, or graph algorithm produce the answer?** → Deterministic. (Examples: citation token extraction, schema-version-mismatch detection, orphan note detection.)
2. **Can vector similarity over note embeddings produce a useful ranking?** → Deterministic. (Examples: similarity-check, find-duplicates, link-suggestion candidate ranking.)
3. **Can a small classifier trained on past human decisions produce a calibrated proposal?** → Deterministic with an LLM fallback for low-confidence cases. → **Hybrid.** (Examples: `_draft_classification`, future triage confidence scoring.)
4. **Can a deterministic step narrow the candidate set before an LLM judges the remainder?** → Hybrid. (Examples: `cite-check` with embedding-pre-filter on claim-source matches.)
5. **Is the task open-ended generation — prose, dialogue, alternative outlines, creative comparison?** → Generative. LLM-required. (Examples: Socratic processing, draft synthesis, counter-outline.)

If none of the first four applies and the task isn't open-ended generation, you've found a gap — the task probably doesn't need to be automated at all.

## The hybrid pattern

The most common Memoria pattern is **deterministic narrowing + LLM enrichment**. It keeps LLM cost bounded and most decisions auditable while preserving judgment-quality output where it matters:

```text
Deterministic step:  Reduce N candidates to K (where K ≪ N)
                              ↓
LLM step:            Generate, judge, or compose over the K
                              ↓
Result:              Audit-trail = deterministic step's output; final = LLM's
```

Concrete instances in Memoria:

- **`[!suggestions]` callout.** 5,000 notes → top-10 candidates by (embedding similarity + shared-citation graph + topic-tag overlap, weighted) → optional LLM prose for *why* each candidate is suggested. LLM works on 10 candidates, not 5,000.
- **`cite-check`.** A draft has 80 claims → citekey-resolution gives 80 candidate sources → embedding similarity gives a quick score per (claim, source) pair → LLM judges only the middle band (similarity 0.4–0.75); above is auto-clean, below is auto-fail.
- **`[!brief]` comparative read.** New source → top-5 most-comparable existing sources by shared citations + embedding similarity → LLM composes the comparative narrative across 5 sources, not the entire corpus.
- **`_draft_classification`.** A new source-note → small multi-label classifier produces topic/methods/study_design proposals → if classifier confidence > 0.85, accept; else fall back to LLM proposal.

The benefit isn't just cost. The deterministic step's output is *auditable* (you can show which sources contributed to the rank, what the similarity score was, why the candidate was selected). The LLM's output is qualitative — useful but opaque. The hybrid keeps the audit trail load-bearing.

## Methods Memoria uses

Seven deterministic methods cover the bulk of Memoria's non-generative work: regex and rule-based scripts, vector embeddings + cosine similarity, classical clustering (HDBSCAN, k-means), topic modeling (LDA, NMF, BERTopic), small classifiers (logistic regression, gradient boosting, fine-tuned BERT), graph algorithms (BFS, PageRank, shortest path), and API calls.

**See [reference/computational-toolbox.md](../reference/computational-toolbox.md)** for the per-method catalog (what each is good for, where Memoria uses it, implementation notes, cost and determinism profile).

## Anti-patterns

These are patterns to avoid because they substitute LLMs for tasks that have deterministic answers:

- **"Ask the LLM if this paper is similar to that one."** Use cosine similarity over embeddings. LLM-as-similarity-judge is expensive, slow, and gives different answers on different runs.
- **"Ask the LLM to extract the citations from this draft."** Use regex `\[@[a-z0-9-]+(?:; ?@[a-z0-9-]+)*\]`. Citations are tokens, not prose.
- **"Ask the LLM to classify which topic this note belongs to."** Train a classifier on the operator's past classifications. The LLM's classification is generic; the classifier's is *yours*.
- **"Ask the LLM to find the orphan notes."** Graph walk. The wikilink graph either has an edge or it doesn't.
- **"Ask the LLM if frontmatter looks valid."** Run the schema-check. Frontmatter validation is structural.
- **"Ask the LLM to confidence-score its own classification."** LLM self-reported confidence is uncalibrated — a 0.9 from an LLM is not a 90% probability of being right. Use the classifier's softmax output instead; that *is* a probability.
- **"Use the LLM for the audit trail."** The LLM's output is a recommendation, not the audit record. The deterministic step's output (similarity score, classifier probability, graph distance) is what should be logged for verdict-band and dashboard queries.

The underlying mistake in all of these: asking an LLM to do work where the answer is *derivable*. LLMs are right for *generation*, not for *derivation*.

## Cost and audit implications

Adopting the hybrid pattern across Memoria changes the operational profile:

| Concern | LLM-everywhere | Memoria's hybrid approach |
|---|---|---|
| Per-action API cost | $0.01–$0.10 per routine task | $0–$0.01 per routine task; LLM only on ambiguous cases |
| Latency per routine task | 2–5s | <100ms for deterministic; same as LLM for the hybrid fallback |
| Test coverage | Hard — LLM outputs drift | Easy — deterministic steps have fixed expected outputs |
| Audit trail | "LLM said X" | "Similarity score 0.87, classifier confidence 0.92, three shared citations: [list]" |
| Calibration | LLM self-reported confidence is unreliable | Classifier softmax is a true probability |
| Privacy | Prompts leave the machine | Routine work runs entirely local |
| Reproducibility | Different runs give different outputs | Same input → same output, always |

The overnight loop's $1–3/day budget ([07-roadmap.md](../07-roadmap/future-directions.md#the-overnight-loop-proactive-discovery-pattern)) is set assuming LLM-heavy ingest. Under the hybrid pattern, the same volume costs $0.20–$1/day. The savings compound: across a year, that's $300–$700 in API spend not paid.

## Per-task classification

Memoria's components, classified by method:

### Deterministic (no LLM)

| Component | Method | Reference |
|---|---|---|
| Linter M1–M8 structural detectors | Hash compare, file existence, regex, graph walk | [profiles/linter.md](../profiles/linter.md) |
| Schema validation | Frontmatter rule check | [profiles/linter.md](../profiles/linter.md) |
| Citation token extraction in `cite-check` | Regex | [profiles/verifier.md](../profiles/verifier.md) |
| `similarity-check` | Cosine similarity over embeddings | [profiles/verifier.md](../profiles/verifier.md) |
| `find-duplicates` | Embedding clustering / pairwise similarity | [profiles/verifier.md](../profiles/verifier.md) |
| `retraction-check` | API call + DOI match | [profiles/verifier.md](../profiles/verifier.md) |
| `enrich` metadata fetches | API calls (OpenAlex, PubMed, etc.) | [profiles/researcher.md](../profiles/researcher.md) |
| `ingest` type detection | Rule-based dispatch table | [04-workflows.md workflow #2](../04-workflows.md) |
| Bib watcher, git hooks, cron triggers | Shell scripts | [01-architecture.md](../01-architecture.md) |
| Audit-log analysis and verdict band rollup | Aggregation stats | [profiles/linter.md](../profiles/linter.md) |
| `cluster-map`, `scope-project` density, `gap-report` topic identification | HDBSCAN / BERTopic / LDA over embeddings | [profiles/cartographer.md](../profiles/cartographer.md) |
| `[!suggestions]` candidate ranking | Weighted scoring (embedding + citation + topic) | [06-surfaces.md inline surfaces](../06-surfaces.md) |
| `[!brief]` candidate selection | Shared-citation + embedding similarity | [06-surfaces.md inline surfaces](../06-surfaces.md) |
| Dataview dashboards | SQL-like queries over frontmatter | [06-surfaces.md](../06-surfaces.md) |
| Drift propagation enumeration (future) | Graph walk over wikilinks | [07-roadmap.md propagation debts](../07-roadmap.md) |

### Hybrid (deterministic narrowing + LLM enrichment)

| Component | Deterministic step | LLM step | Reference |
|---|---|---|---|
| `_draft_classification` proposal | Multi-label classifier produces calibrated proposal | LLM fallback for low-confidence cases (probability < 0.85) | [profiles/researcher.md](../profiles/researcher.md), [07-roadmap.md Decision 11](../07-roadmap.md) |
| `cite-check` claim-source match | Embedding similarity per (claim, source) pair | LLM judges only middle band (0.4–0.75); above is auto-clean, below is auto-fail | [profiles/verifier.md](../profiles/verifier.md) |
| Inline callouts (`[!brief]`, `[!suggestions]`, `[!verification]`) | Candidate selection via weighted scoring (similarity + shared-citations + topic-tag) | Per-callout prose composition or explanation | [06-surfaces.md](../06-surfaces/inline.md#how-the-callout-content-is-produced-deterministic-narrowing--llm-enrichment) — canonical site for per-callout deterministic/LLM split, including weights |

### Generative (LLM-required)

| Component | Why LLM-required | Reference |
|---|---|---|
| Socratic conversation | Open-ended dialog, no fixed output structure | [profiles/socratic.md](../profiles/socratic.md) |
| Writer's `draft` command | Generative synthesis prose | [profiles/writer.md](../profiles/writer.md) |
| `counter-outline` | Creative diversity in outline alternatives | [profiles/writer.md](../profiles/writer.md) |
| `[!brief]` prose composition (within the hybrid) | Comparative narrative requires natural language | [profiles/cartographer.md](../profiles/cartographer.md) |
| `cite-check` ambiguous-band judgment (within the hybrid) | Semantic judgment when similarity is in the 0.4–0.75 middle | [profiles/verifier.md](../profiles/verifier.md) |
| Open-design rendering | Aesthetic and layout judgment over content | [rationale/coder-external-agent.md](coder-external-agent.md) |

## Implementation notes

For the practical details — embedding model selection (`bge-small-en` vs `all-MiniLM-L6-v2` vs `SPECTER2`), classifier training (when to start, when to retrain), the SKILL.md frontmatter where the method-class decision is encoded, and the `llm_backend` pilot-scoped pattern — see [reference/computational-toolbox.md](../reference/computational-toolbox.md#implementation-notes).

## Related design documents

- [profiles/researcher.md](../profiles/researcher.md) — uses classifiers + APIs + LLM-fallback for `_draft_classification`.
- [profiles/cartographer.md](../profiles/cartographer.md) — uses clustering + topic modeling; LLM only for narrative composition.
- [profiles/verifier.md](../profiles/verifier.md) — uses embeddings + regex; LLM only for ambiguous claim-source matches.
- [profiles/writer.md](../profiles/writer.md) — generative, LLM-required.
- [profiles/socratic.md](../profiles/socratic.md) — generative, LLM-required.
- [profiles/linter.md](../profiles/linter.md) — fully deterministic.
- [06-surfaces.md inline surfaces](../06-surfaces/inline.md) — hybrid pattern for `[!suggestions]` and `[!brief]`.
- [07-roadmap.md Decision 11](../07-roadmap.md) — confidence scoring on `_draft_classification`, resolved by the classifier approach.
- [01-architecture/capability-stack.md model routing](../01-architecture/capability-stack.md) — when LLM calls are needed, route synthesis to Claude and cheap tasks (embed, classify, summarize) to cheaper models or local inference.
