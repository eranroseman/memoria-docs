# Obsidian plugins — configuration reference

Memoria depends on a small set of Obsidian community plugins. Each plugin's configuration lives in its own `data.json` under `vault/.obsidian/plugins/<plugin-id>/data.json`. The configs are operator-edited but version-controlled — the vault ships with reviewed defaults and the operator changes them deliberately, not by exploring the plugin's settings UI.

This folder documents which plugins Memoria uses, what their `data.json` should contain, and the **load-bearing settings the operator should not change** without understanding the consequences. The operator-facing summary lives in `00-meta/obsidian-config.md` (vault skeleton note); this folder is the engineering source it points at.

## Plugins Memoria depends on

The filesystem sorts these alphabetically, but **the priority order is what matters when setting up the vault**. Install plugins in the order below; skip optional ones until you feel their absence.

### Required (8) — Memoria breaks without these

| Plugin | Purpose |
| --- | --- |
| [obsidian-local-rest-api](obsidian-local-rest-api.md) | Exposes the vault to Hermes for read/write via HTTP. Required for the [control-plane bridge](../../01-architecture/control-plane.md). **Its `data.json` contains secrets — gitignore it.** |
| [agent-client](agent-client.md) | Implements ACP (Agent Client Protocol) inside Obsidian. Routes operator conversations with Hermes (and optionally Claude Code, Codex, Gemini CLI, Kilo Code) through a chat pane attached to the active note. Makes the Socratic profile invocable from a reading session. |
| [dataview](dataview.md) | Powers every dashboard. Without it, the persistent-surfaces layer in [06-surfaces.md](../../06-surfaces.md) is non-functional. |
| [templater-obsidian](templater.md) | Runs the safe-and-unambiguous frontmatter scripts the linter relies on (see [linter.md](../../profiles/linter.md#implementing-safe-and-unambiguous-fixes-via-templater)). |
| [quickadd](quickadd.md) | Registers Memoria's command palette entries (see [command-palette.md](../../reference/command-palette.md)). |
| [obsidian-citation-plugin](obsidian-citation-plugin.md) | Inserts citations from the BibTeX/BibLaTeX file Better BibTeX exports, and creates literature notes from a configured template. Memoria's primary Zotero ↔ vault bridge for note creation today. |
| [pdf-plus (PDF++)](pdf-plus.md) | Deep-linking from notes to specific PDF passages. Foundational for claim-level citation. |
| [callout-manager](callout-manager.md) | Defines the `[!brief]`, `[!suggestions]`, and `[!verification]` callout types used by the [inline agent surfaces](../../06-surfaces/inline.md). |

### Recommended (5) — install when the friction is felt

| Plugin | Purpose |
| --- | --- |
| [obsidian-linter](obsidian-linter.md) | Frontend formatting linter. **Has a load-bearing footgun.** |
| [smart-connections + markdb-connect](smart-connections.md) | Vector search across notes; Zotero ↔ vault linking. |
| [supercharged-links + obsidian-style-settings](supercharged-links.md) | Color-code internal links by note `type` so a link to a source-note looks different from a link to a claim-note. |
| [hover-editor](hover-editor.md) | Preview wikilinked notes in a popup without leaving the current note. |
| [tag-wrangler](tag-wrangler.md) | Bulk-rename, merge, and inspect tags across the vault. Useful given Memoria's controlled-vocabulary discipline. |

### Deployment-conditional (1) — install if the deployment option calls for it

| Plugin | Purpose |
| --- | --- |
| [obsidian-git](obsidian-git.md) | Auto-commits vault changes. Required under Deployment Options A / B as the sync (Option A) or history (Option B) mechanism. |

### Optional (4) — install only if a specific use case justifies it

| Plugin | Purpose |
| --- | --- |
| [obsidian-kanban](obsidian-kanban.md) | Visual Kanban rendering for `board-state` dashboard. |
| [cmdr (Commander)](cmdr.md) | Bind frequently-used Memoria commands to physical buttons in the ribbon or status bar. |
| [obsidian-outliner](obsidian-outliner.md) | Outline-aware editing for nested lists. |
| [obsidian-excalidraw](obsidian-excalidraw.md) | Hand-drawn diagrams stored as `.excalidraw` files. |

### Future migration (1) — held in reserve

| Plugin | Purpose |
| --- | --- |
| [zotlit](zotlit.md) | Reads Zotero's SQLite database directly — faster for bulk imports. **Held as a future migration target, not currently used.** |

Plus [visual style discipline](visual-style.md) — restraint about how the vault *looks*, independent of any specific plugin.

## The `data.json` convention

Each plugin stores its settings as a JSON file at `.obsidian/plugins/<plugin-id>/data.json`. The format is plugin-specific — there's no shared schema. Memoria ships canonical `data.json` files (or `.example` / `.TODO` variants for plugins that need per-operator setup) directly in the starter vault's `.obsidian/plugins/` tree — they ship to the operator as part of the vault, no separate template-copy step. The lifecycle doc lives at [`plugin-configs-lifecycle.md`](plugin-configs-lifecycle.md) alongside this README. The discipline:

- **Ship a reviewed `data.json` per plugin** in the vault. The operator clones the vault, opens it, and the plugin settings are already correct.
- **Commit `data.json` files to Git by default** — they are configuration, not state, and belong in version control. **Exception: any plugin whose `data.json` contains secrets must be gitignored.** Today the known offender is [obsidian-local-rest-api](obsidian-local-rest-api.md), which stores a generated `apiKey` and an RSA private key directly in `data.json`. Before committing a plugin's `data.json` for the first time, open it and confirm there are no keys, tokens, certificates, or other credentials inside; if there are, gitignore the file and ship a sanitized `data.json.example` instead.
- **Never edit `data.json` while Obsidian is running.** Obsidian writes the file on settings change, so external edits race. Quit Obsidian, edit, restart.

### `.gitignore` snippet for known-secret plugins

```text
# Plugins whose data.json contains secrets — regenerated on first launch
vault/.obsidian/plugins/obsidian-local-rest-api/data.json
```

Add new entries here whenever a plugin update introduces secret storage. The first sign is usually a `crypto:`, `apiKey:`, `token:`, `secret:`, or `privateKey:` field appearing in `data.json` after a plugin upgrade.

## Operational notes

- **Disabling a required plugin breaks Memoria.** The control-plane bridge, dashboards, and templates all assume their plugin is installed and configured. If a plugin update breaks something, downgrade rather than disable.
- **Plugin updates can change `data.json` schema.** When updating a plugin version, diff the new `data.json` against the committed version before accepting changes. Schema migrations have caused silent setting loss in past Obsidian releases.
- **`.obsidian/plugins/*/main.js` is plugin code, not config.** Don't commit it (it changes on every update and bloats Git history). The `.gitignore` should include `.obsidian/plugins/**/main.js` and `.obsidian/plugins/**/styles.css` while keeping `data.json` and `manifest.json`.

## When this doc is wrong

Plugin behavior changes across versions. If a setting documented here doesn't exist in the installed version, the plugin has either added it (the doc is behind) or removed it (the doc is stale). Update the doc the same session you notice the drift — see the same discipline in [failure-modes.md](../failure-modes.md).
