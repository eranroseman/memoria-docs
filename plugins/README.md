---
mode: explanation
audience: operator
topic: plugins
---

# Obsidian plugins — configuration reference

Memoria depends on a small set of Obsidian community plugins. Each plugin's configuration lives in its own `data.json` under `vault/.obsidian/plugins/<plugin-id>/data.json`. The configs are human-edited but version-controlled — the vault ships with reviewed defaults and the human changes them deliberately, not by exploring the plugin's settings UI.

This folder documents which plugins Memoria uses, what their `data.json` should contain, and the **load-bearing settings the human should not change** without understanding the consequences. The human-facing summary lives in `00-meta/04-reference/obsidian-config.md` (vault skeleton note); this folder is the engineering source it points at.

## Plugins Memoria depends on

The filesystem sorts these alphabetically, but **the priority order is what matters when setting up the vault**. Install plugins in the order below; skip optional ones until you feel their absence.

### Required (8) — Memoria breaks without these

| Plugin | Purpose |
| --- | --- |
| [obsidian-local-rest-api](required/obsidian-local-rest-api.md) | Exposes the vault to Hermes for read/write via HTTP. Required for the [control plane](../architecture/control-plane.md). **Its `data.json` contains secrets — gitignore it.** |
| [agent-client](required/agent-client.md) | Implements ACP (Agent Client Protocol) inside Obsidian. Routes human conversations with Hermes (and optionally Claude Code, Codex, Gemini CLI, Kilo Code) through a chat pane attached to the active note. Makes the Socratic profile invocable from a reading session. |
| [dataview](required/dataview.md) | Powers every dashboard. Without it, the persistent-surfaces layer in [surfaces/README.md](../surfaces/README.md) is non-functional. |
| [templater-obsidian](required/templater.md) | Runs the safe-and-unambiguous frontmatter scripts the Linter relies on (see [linter.md](../profiles/linter.md#implementing-safe-and-unambiguous-fixes-via-templater)). |
| [quickadd](required/quickadd.md) | Registers Memoria's command palette entries (see [command-palette.md](../surfaces/command-palette.md)). |
| [obsidian-citation-plugin](required/obsidian-citation-plugin.md) | Inserts citations from the BibTeX/BibLaTeX file Better BibTeX exports, and creates paper notes from a configured template. Memoria's primary Zotero-to-vault path for note creation today. |
| [pdf-plus (PDF++)](required/pdf-plus.md) | Deep-linking from notes to specific PDF passages. Foundational for claim-level citation. |
| [callout-manager](required/callout-manager.md) | Defines the `[!brief]`, `[!suggestions]`, and `[!verification]` callout types used by the [inline agent surfaces](../surfaces/inline.md). |

### Recommended (5) — install when the friction is felt

| Plugin | Purpose |
| --- | --- |
| [obsidian-Linter](recommended/obsidian-linter.md) | Frontend formatting Linter. **Has a load-bearing footgun.** |
| [smart-connections + markdb-connect](recommended/smart-connections.md) | Vector search across notes; Zotero ↔ vault linking. |
| [supercharged-links + obsidian-style-settings](recommended/supercharged-links.md) | Color-code internal links by note `type` so a link to a paper-note looks different from a link to a claim-note. |
| [hover-editor](recommended/hover-editor.md) | Preview wikilinked notes in a popup without leaving the current note. |
| [tag-wrangler](recommended/tag-wrangler.md) | Bulk-rename, merge, and inspect tags across the vault. Useful given Memoria's controlled-vocabulary discipline. |

### Deployment-conditional (1) — install if the deployment option calls for it

| Plugin | Purpose |
| --- | --- |
| [obsidian-git](optional/obsidian-git.md) | Auto-commits vault changes. Required under the local-only and obsidian-sync deployment options as the sync (the local-only option) or history (the obsidian-sync option) mechanism. |

### Optional (4) — install only if a specific use case justifies it

| Plugin | Purpose |
| --- | --- |
| [obsidian-kanban](optional/obsidian-kanban.md) | Visual Kanban rendering for `board-state` dashboard. |
| [cmdr (Commander)](optional/cmdr.md) | Bind frequently-used Memoria commands to physical buttons in the ribbon or status bar. |
| [obsidian-outliner](optional/obsidian-outliner.md) | Outline-aware editing for nested lists. |
| [obsidian-excalidraw](optional/obsidian-excalidraw.md) | Hand-drawn diagrams stored as `.excalidraw` files. |

### Future migration (1) — held in reserve

| Plugin | Purpose |
| --- | --- |
| [zotlit](optional/zotlit.md) | Reads Zotero's SQLite database directly — faster for bulk imports. **Held as a future migration target, not currently used.** |

Plus [visual style discipline](ui-discipline.md) — restraint about how the vault *looks*, independent of any specific plugin.

## The `data.json` convention

Each plugin stores its settings as a JSON file at `.obsidian/plugins/<plugin-id>/data.json`. The format is plugin-specific — there's no shared schema. Memoria ships canonical `data.json` files (or `.example` / `.TODO` variants for plugins that need per-human setup) directly in the starter vault's `.obsidian/plugins/` tree — they ship to the human as part of the vault, no separate template-copy step. The lifecycle doc lives at [`plugin-configs-lifecycle.md`](plugin-configs-lifecycle.md) alongside this README. The discipline:

- **Ship a reviewed `data.json` per plugin** in the vault. The human clones the vault, opens it, and the plugin settings are already correct.
- **Commit `data.json` files to Git by default** — they are configuration, not state, and belong in version control. **Exception: any plugin whose `data.json` contains secrets must be gitignored.** Today the known offender is [obsidian-local-rest-api](required/obsidian-local-rest-api.md), which stores a generated `apiKey` and an RSA private key directly in `data.json`. Before committing a plugin's `data.json` for the first time, open it and confirm there are no keys, tokens, certificates, or other credentials inside; if there are, gitignore the file and ship a sanitized `data.json.example` instead.
- **Never edit `data.json` while Obsidian is running.** Obsidian writes the file on settings change, so external edits race. Quit Obsidian, edit, restart.

### `.gitignore` snippet for known-secret plugins

```text
# Plugins whose data.json contains secrets — regenerated on first launch
vault/.obsidian/plugins/obsidian-local-rest-api/data.json
```

Add new entries here whenever a plugin update introduces secret storage. The first sign is usually a `crypto:`, `apiKey:`, `token:`, `secret:`, or `privateKey:` field appearing in `data.json` after a plugin upgrade.

## Operational notes

- **Disabling a required plugin breaks Memoria.** The control plane, dashboards, and templates all assume their plugin is installed and configured. If a plugin update breaks something, downgrade rather than disable.
- **Plugin updates can change `data.json` schema.** When updating a plugin version, diff the new `data.json` against the committed version before accepting changes. Schema migrations have caused silent setting loss in past Obsidian releases.
- **`.obsidian/plugins/*/main.js` is plugin code, not config.** Don't commit it (it changes on every update and bloats Git history). The `.gitignore` should include `.obsidian/plugins/**/main.js` and `.obsidian/plugins/**/styles.css` while keeping `data.json` and `manifest.json`.

## When this doc is wrong

Plugin behavior changes across versions. If a setting documented here doesn't exist in the installed version, the plugin has either added it (the doc is behind) or removed it (the doc is stale). Update the doc the same session you notice the drift — see the same discipline in [failure-modes.md](../operations/failure-modes.md).

<!-- memoria-nav -->

---

[← Previous: Session logging](../operations/session-logging.md)

[Next: obsidian-local-rest-api →](required/obsidian-local-rest-api.md)
