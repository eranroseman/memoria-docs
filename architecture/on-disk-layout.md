---
mode: reference
audience: system-designer
topic: architecture
---

# On-disk layout: starter vault and Hermes runtime

> [!warning] Scaffold vs. Complete Install
> This layout shows the target on-disk structure. Several items marked below are not yet present in the v0.1 starter vault. Items annotated `# v0.2` or `# deferred` are planned but not yet created.

The system spans **two filesystem locations**: the starter vault (versioned, distributable, holds all install material) and the user's Hermes runtime (per-user, holds installed profile directories copied from the vault by the installer).

## Starter vault (versioned, distributable)

The starter vault is the **single distributable artifact**. The human clones one repo and runs one script; that script sets up everything else.

The **vault root folder is human-defined**. The distribution lives at a repo conventionally called `memoria-vault`, but the human clones it into any folder name they want (`git clone <url> my-research-vault`) and can move it anywhere on disk. Memoria's design is agnostic to the root folder's name and location — `install.ps1` detects its own location at runtime via `$PSScriptRoot` and uses that absolute path everywhere a vault path is needed (notably the `{{VAULT_PATH}}` substitution in each profile's `mcp.json`). Only the *contents* of the root — the `00-meta/`, `.obsidian/`, `.memoria/` shape — are fixed by the design.

Numbered-prefix subdirectories (e.g., `01-templates`, `02-csl`, `01-papers`) are the standard naming convention — see [vault/README.md](../vault/README.md#folder-structure) for the full taxonomy and per-folder role table. Numbers encode display order in Obsidian's file explorer (alphabetical sort puts them in semantic-priority order) and give workflows stable paths to reference.

```text
<vault-root>/                           # any folder name; human picks at clone time
├── README.md                           # human-facing: clone, run install
├── install.ps1                         # Windows installer
├── install.sh                          # macOS/Linux — v0.2
│
├── 00-meta/                            # vault skeleton (human-visible in Obsidian)
│   ├── 01-dashboards/                  # 11 shipped dashboards (Daily Health as index.md, board-state, contradictions, drift-watch, audit-log, …; skill-lifecycle deferred)
│   ├── 02-logs/                        # audit.jsonl, board-state.jsonl, lint-findings.jsonl, cron-history.jsonl
│   ├── 03-templates/                   # 15 note templates (claim-note, paper-note, …)
│   ├── 04-reference/                   # human-facing reference notes (design-system, schema-reference, …)
│   ├── 05-eval/                        # vault-eval gold tasks (per-workflow gold sets) — ADR-23
│   ├── 08-metrics/                     # fleet + eval metrics (telemetry rollups, eval/ run results) — Post-MVS
│   ├── index.md                        # vault landing page (pinned in sidebar)
│   ├── research-directions.md          # Librarian's session-start input
│   └── system-status.md                # runtime health snapshot
├── 10-inbox/
│   ├── 01-fleeting/                    # raw captures
│   ├── 02-answers/                   # answer drafts awaiting review
│   └── 03-candidates/                  # discovery leads and screening queue
├── 20-sources/
│   ├── 01-papers/                     # one note per citable paper (specialized item)
│   ├── 02-items/                       # tools, repos, packages, products, standards, datasets, software
│   └── 03-entities/                    # 01-people / 02-organizations / 03-venues
├── 30-synthesis/
│   ├── 01-claims/                   # claim notes — human-owned
│   ├── 02-reference/                        # reference notes
│   └── 03-moc/                         # Maps of Content
├── 40-workbench/
│   └── <project>/                      # one folder per project: 01-map/ 02-framing/ 03-canvas/ 04-drafts/ 05-verification/ 06-code/
├── 50-deliverables/
│   ├── 01-manuscripts/                 # papers, articles, preprints
│   ├── 02-presentations/               # slides, talks, posters
│   ├── 03-media/                       # figures, infographics, web
│   └── 04-releases/                    # datasets, models, code, supplementary
├── 90-assets/                          # Marker extracts, figures, supplementary materials (PDFs stay in Zotero)
├── 95-archive/                         # superseded notes
│
├── .obsidian/                          # Obsidian config (auto-hidden by Obsidian)
│   ├── plugins/
│   │   ├── obsidian-linter/data.json   # post-v0.1 (reference-only per ADR-24; not installed)
│   │   ├── obsidian-citation-plugin/data.json
│   │   ├── agent-client/data.json.example
│   │   ├── obsidian-local-rest-api/data.json.example
│   │   └── callout-manager/data.json.TODO
│   └── snippets/
│       └── memoria-link-colors.css
│
└── .memoria/                           # Memoria tooling (dot-prefix: auto-hidden by Obsidian)
    ├── profiles/                       # the seven Hermes profile directories, hand-authored
    │   ├── memoria-librarian/
    │   │   ├── SOUL.md                 # the actual system prompt
    │   │   ├── config.yaml             # model, commands, tool filters
    │   │   ├── mcp.json                # MCP server connections (with {{VAULT_PATH}} placeholders)
    │   │   ├── cron/                   # scheduled tasks for this profile
    │   │   ├── skills/                 # profile-bundled skills
    │   │   ├── .env.EXAMPLE            # required env vars, commented out
    │   │   └── distribution.yaml       # name, version
    │   ├── memoria-mapper/
    │   ├── memoria-socratic/
    │   ├── memoria-writer/
    │   ├── memoria-verifier/
    │   ├── memoria-coder/
    │   └── memoria-linter/             # also holds M-detectors.md alongside SOUL.md
    ├── profile-memory/                 # learned MEMORY.md/USER.md, junctioned into ~/.hermes (opt-in multi-machine sync) — created on first multi-machine sync opt-in
    │   └── memoria-<name>/
    ├── mcp/                            # Memoria-specific MCP servers (Python)
    │   ├── policy_mcp.py               # not yet authored (v0.2)
    │   ├── tasks_mcp.py               # not yet authored (v0.2)
    │   └── requirements.txt
    ├── lane-overrides/                 # YAML files the policy MCP reads at startup (named {lane}.yaml)
    │   ├── librarian.yaml              # not yet authored (v0.2)
    │   ├── mapper.yaml                 # not yet authored (v0.2)
    │   ├── socratic.yaml               # not yet authored (v0.2)
    │   ├── writer.yaml                 # not yet authored (v0.2)
    │   ├── verifier.yaml               # not yet authored (v0.2)
    │   ├── coder.yaml                  # not yet authored (v0.2)
    │   └── linter.yaml                 # not yet authored (v0.2)
    ├── csl/                            # Pandoc citation style files
    ├── library.bib                     # Zotero BibTeX export
    └── tool-registry.yaml              # machine-read tool config
```

Engineering documentation (architecture, workflows, decisions (ADRs), profile design summaries, dashboards, roadmap, and topic-distributed reference material — ~125 files under `memoria-docs/`) lives in a **separate repository**. Documentation is not part of the runtime; the human's vault doesn't need it to function.

## Runtime install (per-user, not in repo)

When the human runs `install.ps1` / `install.sh`, the installer copies the seven profile directories from `.memoria/profiles/` to Hermes's standard location at `~/.hermes/profiles/` (or `%USERPROFILE%\.hermes\profiles\` on Windows; both honor `HERMES_HOME` to override). Profiles are prefixed `memoria-` to keep them separable from other agents on the same machine:

```text
~/.hermes/profiles/
├── memoria-librarian/                 # copied from .memoria/profiles/memoria-librarian/
│   ├── SOUL.md                         # author-owned, overwritten on every install
│   ├── config.yaml                     # author-owned, overwritten on every install
│   ├── mcp.json                        # author-owned, {{VAULT_PATH}} substituted with real path
│   ├── cron/
│   ├── skills/
│   ├── .env.EXAMPLE                    # author-owned, overwritten on every install
│   ├── .env                            # user-owned, preserved across installs
│   └── distribution.yaml
├── memoria-mapper/
├── memoria-socratic/
├── memoria-writer/
├── memoria-verifier/
├── memoria-coder/
└── memoria-linter/
```

Per-profile structure follows the [Hermes profile-distribution shape](https://hermes-agent.nousresearch.com/docs/user-guide/profile-distributions) — `SOUL.md`, `config.yaml`, `mcp.json`, `skills/`, `cron/` — so Memoria profiles are compatible with `hermes profile list` and the standard `hermes -p memoria-librarian chat` invocation surface.

## How the installer works

1. **Detect prerequisites.** Verify Hermes is installed and Python is available for the MCP servers; fail with a clear message if either is missing.
2. **Stage each profile.** Copy `.memoria/profiles/<name>/` to a temporary directory. In each copy's `mcp.json`, substitute `{{VAULT_PATH}}` with the absolute path of this vault (forward-slash form for cross-platform JSON safety).
3. **Install.** For each staged profile, run `hermes profile install <staged-path> --alias memoria-<name> --force --yes`. Hermes copies author-owned files (SOUL.md, config.yaml, mcp.json, skills/, cron/) into `~/.hermes/profiles/memoria-<name>/`, leaves `.env` alone (user-owned).
4. **Bootstrap secrets.** For each installed profile, if `.env` doesn't exist, copy `.env.EXAMPLE` to `.env`. Human fills in real values.
5. **Clean up staging.** Delete the temporary directory.

The script is **idempotent and always-overwrite**: re-running after `git pull` rewrites every author-owned file from the vault source, preserves every human-owned secret. Humans who tuned `config.yaml` through `hermes config set` will see those tweaks reset; the trade is predictability — every install matches the source exactly.

## Why this structure

- **One artifact, one install.** The human's primary interaction is with the Obsidian vault. Bundling install material with the vault gives a single unified UX: clone one repo, run one script, everything is in place.
- **Dot-prefix for tooling.** Both `.obsidian/` (Obsidian's own config) and `.memoria/` (Memoria's install material) use the dot-prefix convention so Obsidian's vault scanner auto-hides them from the file explorer, search, graph view, and Dataview queries. Human sees content; tooling stays out of the way without any per-vault exclusion config.
- **Direct profile management (no compiler).** The seven profile directories under `.memoria/profiles/` are hand-authored, not generated. Shared content (audit-log behavior, common policies, common MCP connections) lives in seven places and is maintained by hand. Drift between sibling profiles is caught by the Linter rather than prevented at build time. See [the deferred compiler vision](../roadmap/profile-compilation.md) for the alternative that may become relevant if drift becomes painful at the seven-profile scale.
- **Source = runtime.** Under direct management, the file in `.memoria/profiles/memoria-librarian/SOUL.md` is exactly what gets installed to `~/.hermes/profiles/memoria-librarian/SOUL.md`. No build step means no "compiled vs source" duality — what's in the repo is what the agent reads at runtime.
- **MCP servers ship with the vault.** The Python sources for `policy_mcp.py` and `tasks_mcp.py` live at `.memoria/mcp/`. Each profile's `mcp.json` points at this location via the `{{VAULT_PATH}}` placeholder, substituted at install time. Humans don't need a separate clone of an MCP-source repo.

## Version control

**Git is Memoria's authoritative version-history layer.** Every reversibility property the system relies on — recoverable mistakes, audited diffs, the ability to roll a worker's write back — assumes a git history exists. This is non-negotiable; it's why the [policy MCP audit log](README.md#permission-enforcement-the-policy-mcp) records `before_hash` and `after_hash` for every write, and why the [Linter's `vault-hash-drift` detector](../profiles/linter.md) treats a non-MCP write as CRITICAL.

- **In Git** (starter vault repo): the vault skeleton (`00-meta/`, `10-inbox/`, …); `.obsidian/` config; all of `.memoria/` (profiles, MCP source, lane overrides); installers; README. Optionally also `.memoria/profile-memory/` — see the profile-memory note below.
- **Out of Git**: `.env` files in `~/.hermes/profiles/`, `auth.json`, secrets, the raw audit log (`00-meta/02-logs/audit.jsonl` — append-only log of human-touchable writes), and sessions / the session database (`~/.hermes/state.db`). These are local runtime state. Profile memory (`MEMORY.md`/`USER.md`) at `~/.hermes/profiles/<name>/memories/` is also local by default — but can be promoted into the vault for non-concurrent multi-machine use via the [`memories/` junction](../roadmap/sync-and-coordination.md#syncing-profile-memory-across-machines-the-memories-junction), which stores the files at `.memoria/profile-memory/memoria-<name>/` and junctions Hermes's path onto them.
- **Generated locally, not committed**: `~/.hermes/profiles/memoria-*/` directories. These are recreated from the vault source on every `install.ps1` run.

**Sync ≠ version history.** Memoria's [deployment options matrix](../roadmap/deployment-options.md) gives three sync choices (manual git pull/push, Obsidian Sync, or Syncthing) — these are about *how the vault propagates between devices*, not about *how history is recorded*. Git is always the history layer regardless of which sync layer the human picks. Running git's pull/push **as the sync layer** (the local-only option) at the same time as Obsidian Sync is a misconfiguration: two sync mechanisms race for the same files (most painfully `.obsidian/workspace.json`) and one will lose. Pick one sync layer; let git be history-only behind it.

## Related

- Vault folder taxonomy: [vault/README.md](../vault/README.md)
- Lane-override YAML format: [profiles/README.md lane-override files](../profiles/README.md#lane-override-files)
- The deferred compiler design (not currently in use): [roadmap/profile-compilation.md](../roadmap/profile-compilation.md)
