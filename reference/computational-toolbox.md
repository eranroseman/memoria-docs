# Computational toolbox: deterministic methods Memoria uses

The deterministic methods catalog and their practical implementation details. The *rationale* for choosing deterministic over LLM (the hybrid pattern, anti-patterns, cost implications) lives in [rationale/computational-methods.md](../rationale/computational-methods.md); this document is the reference for what each method does and how to deploy it.

## Methods Memoria uses

The deterministic toolbox, organized by what each is good for:

### Regex and rule-based scripts

For: parsing structured text (citations, frontmatter, wikilinks), pattern detection in filenames, deterministic transformations (e.g., normalize whitespace, sort YAML keys).

Used in: linter's M1–M8 detectors ([profiles/linter.md](../profiles/linter.md)), `cite-check` citation extraction ([profiles/verifier.md](../profiles/verifier.md)), schema validation, `ingest` type-detection dispatch table ([04-workflows.md workflow #2](../04-workflows.md)).

Cost: free. Latency: microseconds. Determinism: total.

### Vector embeddings + cosine similarity

For: finding similar notes, ranking candidate links, detecting near-duplicates, narrowing comparative-read candidates.

Used in: `similarity-check`, `find-duplicates`, `[!suggestions]` ranking, `[!brief]` candidate selection.

Implementation: a sentence-transformer model (e.g., `bge-small-en`, `all-MiniLM-L6-v2`) embeds note bodies. Embeddings stored in an HNSW index (FAISS, hnswlib, or `qmd` if used as the vector layer). Re-index incrementally as new notes arrive.

Cost: one-time embedding compute per note (~10ms). Per-query: <100ms across thousands of notes. Determinism: total (same model + same text → same vector).

### Classical clustering (HDBSCAN, k-means)

For: corpus density analysis, identifying conceptual clusters in a project's scope, gap detection.

Used in: Cartographer's `cluster-map`, `scope-project`, `gap-report` ([profiles/cartographer.md](../profiles/cartographer.md)).

Implementation: HDBSCAN over note embeddings (no need to pre-specify cluster count). UMAP for 2D projection if visualization is needed.

Cost: seconds to minutes for thousands of notes. Determinism: HDBSCAN is deterministic for fixed parameters; UMAP has a random seed but can be fixed.

### Topic modeling (LDA, NMF, BERTopic)

For: identifying topics that are underrepresented in the corpus, comparing topic distributions across projects, surfacing methodological themes.

Used in: `gap-report` thin-coverage detection (when topic modeling outperforms simple tag aggregation), corpus-evolution dashboards.

Implementation: BERTopic is the modern default (combines embeddings + clustering + class-based TF-IDF for topic labels). For smaller corpora, classical LDA over TF-IDF works.

Cost: minutes for thousands of documents (one-time per analysis). Determinism: same data + same params → same topics.

### Small classifiers (logistic regression, gradient boosting, fine-tuned BERT)

For: proposing `_draft_classification` labels, scoring whether a note belongs to a project, predicting reading priority.

Used in: `_draft_classification` proposal (with LLM fallback for low-confidence cases), reading-priority ranking when sufficient training data exists.

Implementation: `scikit-learn` for tabular features and simple text features (TF-IDF). For deeper text, a fine-tuned small BERT (e.g., DistilBERT). Train on the operator's *past triage decisions* — the human-confirmed labels are the training set.

Cost: training is occasional and offline; inference is sub-millisecond. Determinism: total once trained.

**The retraining loop matters.** As the corpus grows, the operator overrides the classifier's proposals; those overrides become new training data. Periodic retraining (monthly is plenty) keeps the classifier's calibration aligned with the operator's evolving vocabulary. This is operationally cheap and architecturally important — the classifier becomes more accurate over time, the LLM fallback gets used less, costs drop.

### Graph algorithms (BFS, PageRank, shortest path)

For: orphan detection, MOC hub identification, dependency walks (propagation debt), link density measurement.

Used in: linter's graph-analyze command ([profiles/linter.md](../profiles/linter.md)), future propagation-debt enumeration ([07-roadmap.md](../07-roadmap.md)).

Implementation: build the wikilink graph from frontmatter + body links; run standard algorithms (NetworkX in Python, or query Obsidian's link-graph cache via `dataviewjs`).

Cost: linear in graph size. Determinism: total.

### API calls (Zotero, OpenAlex, PubMed, CrossRef, GitHub)

For: metadata enrichment, retraction monitoring, citation graph traversal.

Used in: `ingest`, `enrich`, `retraction-check`, `discover`.

Cost: per-call API budget. Determinism: depends on the API (most are stable, some return ranked results that drift).

## Implementation notes

### Embedding model selection

The right embedding model depends on the operator's research domain. Three defaults:

- **`bge-small-en`** (33M params) — strong general-purpose retrieval. Good default for English-language research vaults.
- **`all-MiniLM-L6-v2`** (22M params) — smaller, faster, slightly weaker. Good for resource-constrained setups.
- **`SPECTER2`** — fine-tuned on scientific papers. Best for citation-similarity tasks specifically.

Re-embedding the entire vault on a model change is cheap (~10ms per note × thousands of notes = minutes). Don't pin a specific model in the design; let the operator choose. The vault stores embeddings keyed by (model_id, model_version) so multiple models can coexist during evaluation.

### Classifier training

The classifier for `_draft_classification` should:

- Train on `triage_status: full` source-notes (those the operator has triaged completely). Filter out `triage_status: partial` — they're not yet ground truth.
- Use multi-label (one-vs-rest) for `topic` and `methods` (a note can have multiple topics and methods). Use multi-class for `study_design` (one value).
- Retrain monthly or when the override rate (operator changes the proposed labels) exceeds 25%. Both are simple cron-triggered jobs.
- Log per-class precision/recall to inform when to retrain.

Training data starts small. The first few hundred notes have to be human-classified (no classifier yet). After 200–500 triaged notes, the classifier becomes useful. After 1,000, it's calibrated.

### Where the deterministic-vs-LLM decision lives in code

The decision is made at the *skill* level, not the agent level. Each Hermes skill in [profiles/*](../profiles/) declares its method class in its SKILL.md frontmatter:

```yaml
method_class: deterministic | hybrid | generative
deterministic_engine: regex | embedding | classifier | clustering | graph | api
llm_fallback_threshold: 0.85           # for hybrid skills only
llm_backend: generic | open-notebook   # for hybrid and generative skills
llm_backend_fallback: generic | none   # fallback when primary back-end unreachable
```

The linter can verify that the declared method class matches the implementation (skills declared `deterministic` shouldn't make LLM calls; skills declared `generative` shouldn't pretend otherwise). This is a future M-detector candidate, not yet shipped.

### LLM back-end choice

For `hybrid` and `generative` skills, the `llm_backend` field selects which LLM provider handles the LLM step. The default (`generic`) routes to whichever LLM the host profile is configured for in its `config.yaml`. Alternative back-ends route to specialized tools:

- **`generic`** — the host profile's default LLM (typically Claude for synthesis lanes, a cheap model for bulk-mechanical work). Used everywhere unless overridden.
- **`open-notebook`** — routes to a self-hosted [Open Notebook](https://github.com/lfnovo/open-notebook) instance for source-grounded RAG. The skill must assemble a source bundle (markdown export of relevant notes) and post-process Open Notebook's citations into Memoria wikilinks. Currently in pilot — see [07-roadmap.md Pilot E1](../07-roadmap/pilots/E1-open-notebook.md), which restricts the `open-notebook` back-end to one skill (`comparative-brief`) and provides explicit success and rollback criteria.

The pattern is *partial implementation* of broader skill-level back-end routing. Until a pilot succeeds and the operator chooses to expand it, only one skill at a time is allowed to specify `llm_backend: open-notebook`. Adding more would require:

1. The first pilot succeeding and the operator opting to expand.
2. The Hermes runtime gaining support for routing `llm_backend` field values to specific clients.
3. Updating this section to document the broader pattern.

Until all three are true, treat the `llm_backend` field as **pilot-scoped** — one skill, one back-end, explicit success criteria.

The `llm_backend_fallback` field is what protects daily workflow from infrastructure failures of the experimental back-end. If `open-notebook` is unreachable and the fallback is `generic`, the call routes through the generic LLM and logs the fallback. If the fallback is `none`, the call fails — use this only when testing the pilot's resilience.

## Related

- [rationale/computational-methods.md](../rationale/computational-methods.md) — the *why* this toolbox exists, the hybrid pattern, anti-patterns, cost and audit implications.
- [profiles/researcher.md](../profiles/researcher.md), [profiles/cartographer.md](../profiles/cartographer.md), [profiles/verifier.md](../profiles/verifier.md) — primary callers of the deterministic methods.
- [profiles/linter.md](../profiles/linter.md) — fully-deterministic profile; uses regex, hashing, graph walks throughout.
