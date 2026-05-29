---
mode: how-to
audience: operator
topic: plugins
---

# obsidian-linter — the footgun

The Obsidian Linter is a frontend formatting tool (not the **Memoria Linter** — a different thing: the Memoria Linter does structural validation; the Obsidian Linter does inline Markdown formatting).

**Authoritative config:** shipped at `.obsidian/plugins/obsidian-linter/data.json` in the starter vault — a ready-to-use `data.json` (see [the suffix conventions](../plugin-configs-lifecycle.md#the-three-suffix-conventions)). It encodes the `foldersToIgnore` list, the four on-save rules, and the `yaml-key-sort` key order that aligns with Memoria's frontmatter schema.

**Load-bearing settings:**

- `foldersToIgnore` — must exclude every folder where the agent maintains comment blocks (`_proposed_classification`, `_enrichment`). The Memoria-shipped `data.json` excludes:

  ```json
  "foldersToIgnore": [
    "00-meta",
    "10-inbox",
    "20-sources",
    "30-synthesis",
    "40-workbench",
    "50-deliverables",
    "90-assets",
    "95-archive"
  ]
  ```

  The Obsidian Linter runs **only on draft/canvas content the human directly writes** — and even that is conservative. The Linter does not support per-folder rules; whole-folder exclusion is the only knob.

- `ruleConfigs."yaml-key-sort".enabled: true` — enforces frontmatter key order on the folders that aren't excluded.
- `ruleConfigs."yaml-timestamp".enabled: false` — **must stay disabled.** Hermes writes notes frequently; if yaml-timestamp is on, every Hermes write resets the timestamp to "now," which destroys the human-vs-agent edit signal the Memoria Linter and dashboards rely on.

## Rule set: on-save vs manual invocation

The Obsidian Linter ships with 60+ rules. Enabling too many on save creates surprise — it touches prose every time the human saves, and rules that change content (capitalize-headings, auto-correct-common-misspellings, smart-quote substitution) can corrupt domain-specific terms before the human notices. The split: a **small set on save**, a **broader set on manual invocation only**.

**On save** (`lintOnSave: true`, these rules enabled):

- `yaml-key-sort` — frontmatter key ordering. The rule that matters most; out-of-order keys break Dataview indexing and the Memoria Linter's structural checks.
- `yaml-trailing-newline` — ensures the closing `---` has a newline after it (some YAML parsers stumble without it).
- `trailing-spaces` — strips trailing whitespace per line.
- `remove-multiple-spaces` — collapses runs of spaces in prose.

These four are mechanical, predictable, and never change semantic content. They're the minimum the research-wiki vault has been running with in production.

**Manual invocation only** (`Cmd-P → Linter: Lint current file`, otherwise disabled):

- All paste rules (`*-on-paste`): proper ellipsis, remove hyphens, remove leftover footnotes, remove multiple blank lines, etc. These prevent garbage from imported web/PDF content — useful when invoked after pasting, surprising when fired on every save.
- All spacing rules beyond the on-save four: `consecutive-blank-lines`, `paragraph-blank-lines`, `empty-line-around-code-fences`, `heading-blank-lines`. Useful for readability passes but produce diffs on every save that the human hasn't asked for.
- All content rules: `emphasis-style`, `strong-style`, `ordered-list-style`, `unordered-list-style`, `proper-ellipsis`, `quote-style`, `convert-bullet-list-markers`. Helpful for consistency sweeps but opinionated.
- All footnote rules: `move-footnotes-to-the-bottom`, `re-index-footnotes`. Useful before export, surprising mid-draft.
- All heading rules: `header-increment`, `headings-start-line`, `capitalize-headings`. The last one is especially prone to mangling domain terms.

**Always disabled** (the dangerous category):

- `auto-correct-common-misspellings` — has a track record of "fixing" technical terms wrongly (e.g., `JITAI` → `Just-In-Time-AI`).
- `yaml-timestamp` — see above.
- `insert-yaml-attributes` and `remove-yaml-keys` — schema-collision risk with Memoria's frontmatter contract.
- Any rule that strips HTML comments — see below.

Why this stratification: the on-save set is small enough that the human can keep all four rules' behavior in their head, which makes save-time changes predictable. The manual set is where the bulk of the value lives, but only when the human invokes it explicitly. The always-disabled category contains the rules that would silently corrupt content the human can't easily restore.

**Never enable any rule that strips HTML comments.** Memoria's `_proposed_classification` and `_enrichment` blocks (see [vault/README.md](../../vault/README.md) and the [obsidian-citation-plugin](../required/obsidian-citation-plugin.md) template) live as `<!-- ... -->` blocks inside paper notes. A Linter rule that removes HTML comments will silently delete every one of them on the next lint pass — and there is no undo. The agent has to re-enrich and re-classify every affected note.

**Version note.** As of Obsidian Linter `v1.31.2` (the version research-wiki runs against), no `remove-html-comments` rule exists in the bundled rule registry — a search of the plugin's `main.js` finds neither the rule name nor any equivalent transform. The warning here is forward-looking discipline rather than a current-version footgun: if a future version adds such a rule (or a community fork ships one), it must stay disabled. Before accepting any Obsidian Linter upgrade, diff the rule registry against the previous version and check for new rules whose name or description mentions "html", "comment", or "strip".