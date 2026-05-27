# On-disk layout: starter vault and Hermes runtime

The system spans **two filesystem locations**: the starter vault (versioned, distributable, holds all install material) and the user's Hermes runtime (per-user, holds installed profile directories copied from the vault by the installer).

## Starter vault (versioned, distributable)

The starter vault is the **single distributable artifact**. The operator clones one repo and runs one script; that script sets up everything else.

The **vault root folder is operator-defined**. The distribution lives at a repo conventionally called `memoria-vault`, but the operator clones it into any folder name they want (`git clone <url> my-research-vault`) and can move it anywhere on disk. Memoria's design is agnostic to the root folder's name and location — `install.ps1` detects its own location at runtime via `$PSScriptRoot` and uses that absolute path everywhere a vault path is needed (notably the `{{VAULT_PATH}}` substitution in each profile's `mcp.json`). Only the *contents* of the root — the `00-meta/`, `.obsidian/`, `.memoria/` shape — are fixed by the design.

Numbered-prefix subdirectories (e.g., `01-templates`, `02-csl`, `01-literature`) are the canonical naming convention — see [05-notes-folders.md](../05-notes-folders.md#folder-structure) for the full taxonomy and per-folder role table. Numbers encode display order in Obsidian's file explorer (alphabetical sort puts them in semantic-priority order) and give workflows stable paths to reference.

```text
<vault-root>/                           # any folder name; operator picks at clone time
├── README.md                           # operator-facing: clone, run install
├── install.ps1                         # Windows installer
├── install.sh                          # macOS / Linux installer
│
├── 00-meta/                            # vault skeleton (operator-visible in Obsidian)
│   ├── 01-templates/                   # 15 note templates (claim-note, source-note, …)
│   ├── 02-csl/                         # Pandoc citation style files
│   ├── 03-config/                      # library.bib, tool-registry.yaml, etc.
│   ├── 04-logs/                        # audit.jsonl, board-state.jsonl, lint-findings.jsonl, cron-history.jsonl
│   ├── 05-dashboards/                  # 11 dashboards (index, board-state, drift-watch, fleet-observability, …)
│   ├── 06-schema/                      # schema-reference.md, design-system.md, AGENTS-style contracts
│   ├── 07-skills/                      # one skill-note per Hermes skill
│   └── 08-metrics/                     # lane-metric and skill-metric notes
├── 10-inbox/
│   ├── 01-fleeting/                    # raw captures
│   ├── 02-synthesis/                   # synthesis drafts awaiting review
│   └── 03-candidates/                  # discovery leads and screening queue
├── 20-sources/
│   ├── 01-literature/                  # one note per citable source
│   ├── 02-items/                       # tools, repos, packages, products, standards
│   └── 03-entities/                    # 01-people / 02-organizations / 03-venues
├── 30-synthesis/
│   ├── 01-permanent/                   # claim notes — human-owned
│   ├── 02-wiki/                        # reference notes
│   └── 03-moc/                         # Maps of Content
├── 40-workbench/
│   ├── 01-projects/                    # project coordination, scope, milestones
│   ├── 02-drafts/                      # manuscripts in progress
│   ├── 03-code/                        # code artifacts, scripts, notebooks
│   └── 04-canvas/                      # argument mapping, spatial planning
├── 50-deliverables/
│   ├── 01-manuscripts/                 # final manuscripts
│   ├── 02-presentations/               # slides
│   └── 03-exports/                     # submission-ready exports
├── 90-assets/                          # Marker extracts, figures, supplementary materials (PDFs stay in Zotero)
├── 95-archive/                         # superseded notes
│
├── .obsidian/                          # Obsidian config (auto-hidden by Obsidian)
│   ├── plugins/
│   │   ├── obsidian-linter/data.json
│   │   ├── obsidian-citation-plugin/data.json
│   │   ├── agent-client/data.json.example
│   │   ├── obsidian-local-rest-api/data.json.example
│   │   └── callout-manager/data.json.TODO
│   └── snippets/
│       └── memoria-link-colors.css
│
└── .memoria/                           # Memoria tooling (dot-prefix: auto-hidden by Obsidian)
    ├── profiles/                       # the seven Hermes profile directories, hand-authored
    │   ├── memoria-researcher/
    │   │   ├── SOUL.md                 # the actual system prompt
    │   │   ├── config.yaml             # model, commands, tool filters
    │   │   ├── mcp.json                # MCP server connections (with {{VAULT_PATH}} placeholders)
    │   │   ├── cron/                   # scheduled tasks for this profile
    │   │   ├── skills/                 # profile-bundled skills
    │   │   ├── .env.EXAMPLE            # required env vars, commented out
    │   │   └── distribution.yaml       # name, version
    │   ├── memoria-cartographer/
    │   ├── memoria-socratic/
    │   ├── memoria-writer/
    │   ├── memoria-verifier/
    │   ├── memoria-coder/
    │   └── memoria-linter/             # also holds M-detectors.md alongside SOUL.md
    ├── mcp/                            # Memoria-specific MCP servers (Python)
    │   ├── policy_mcp.py
    │   ├── tasks_mcp.py
    │   └── requirements.txt
    └── lane-overrides/                 # YAML files the policy MCP reads at startup
        ├── research.yaml
        ├── cartographer.yaml
        ├── socratic.yaml
        ├── writer.yaml
        ├── verifier.yaml
        ├── coder.yaml
        └── linter.yaml
```

Engineering documentation (architecture, workflows, rationale, ADRs, references — ~92 files under `memoria-docs/`) lives in a **separate repository**. Documentation is not part of the runtime; the operator's vault doesn't need it to function.

## Runtime install (per-user, not in repo)

When the operator runs `install.ps1` / `install.sh`, the installer copies the seven profile directories from `.memoria/profiles/` to Hermes's canonical location at `~/.hermes/profiles/` (or `%USERPROFILE%\.hermes\profiles\` on Windows; both honor `HERMES_HOME` to override). Profiles are prefixed `memoria-` to keep them separable from other agents on the same machine:

```text
~/.hermes/profiles/
├── memoria-researcher/                 # copied from .memoria/profiles/memoria-researcher/
│   ├── SOUL.md                         # author-owned, overwritten on every install
│   ├── config.yaml                     # author-owned, overwritten on every install
│   ├── mcp.json                        # author-owned, {{VAULT_PATH}} substituted with real path
│   ├── cron/
│   ├── skills/
│   ├── .env.EXAMPLE                    # author-owned, overwritten on every install
│   ├── .env                            # user-owned, preserved across installs
│   └── distribution.yaml
├── memoria-cartographer/
├── memoria-socratic/
├── memoria-writer/
├── memoria-verifier/
├── memoria-coder/
└── memoria-linter/
```

Per-profile structure follows the [Hermes profile-distribution shape](https://hermes-agent.nousresearch.com/docs/user-guide/profile-distributions) — `SOUL.md`, `config.yaml`, `mcp.json`, `skills/`, `cron/` — so Memoria profiles are compatible with `hermes profile list` and the standard `hermes -p memoria-researcher chat` invocation surface.

## How the installer works

1. **Detect prerequisites.** Verify Hermes is installed and Python is available for the MCP servers; fail with a clear message if either is missing.
2. **Stage each profile.** Copy `.memoria/profiles/<name>/` to a temporary directory. In each copy's `mcp.json`, substitute `{{VAULT_PATH}}` with the absolute path of this vault (forward-slash form for cross-platform JSON safety).
3. **Install.** For each staged profile, run `hermes profile install <staged-path> --alias memoria-<name> --force --yes`. Hermes copies author-owned files (SOUL.md, config.yaml, mcp.json, skills/, cron/) into `~/.hermes/profiles/memoria-<name>/`, leaves `.env` alone (user-owned).
4. **Bootstrap secrets.** For each installed profile, if `.env` doesn't exist, copy `.env.EXAMPLE` to `.env`. Operator fills in real values.
5. **Clean up staging.** Delete the temporary directory.

The script is **idempotent and always-overwrite**: re-running after `git pull` rewrites every author-owned file from the vault source, preserves every operator-owned secret. Operators who tuned `config.yaml` through `hermes config set` will see those tweaks reset; the trade is predictability — every install matches the source exactly.

## Why this structure

- **One artifact, one install.** The operator's primary interaction is with the Obsidian vault. Bundling install material with the vault gives a single unified UX: clone one repo, run one script, everything is in place.
- **Dot-prefix for tooling.** Both `.obsidian/` (Obsidian's own config) and `.memoria/` (Memoria's install material) use the dot-prefix convention so Obsidian's vault scanner auto-hides them from the file explorer, search, graph view, and Dataview queries. Operator sees content; tooling stays out of the way without any per-vault exclusion config.
- **Direct profile management (no compiler).** The seven profile directories under `.memoria/profiles/` are hand-authored, not generated. Shared content (audit-log behavior, common policies, common MCP connections) lives in seven places and is maintained by hand. Drift between sibling profiles is caught by the linter rather than prevented at build time. See [the deferred compiler vision](../reference/profile-compilation.md) for the alternative that may become relevant if drift becomes painful at the seven-profile scale.
- **Source = runtime.** Under direct management, the file in `.memoria/profiles/memoria-researcher/SOUL.md` is exactly what gets installed to `~/.hermes/profiles/memoria-researcher/SOUL.md`. No build step means no "compiled vs source" duality — what you read in the repo is what the agent reads at runtime.
- **MCP servers ship with the vault.** The Python sources for `policy_mcp.py` and `tasks_mcp.py` live at `.memoria/mcp/`. Each profile's `mcp.json` points at this location via the `{{VAULT_PATH}}` placeholder, substituted at install time. Operators don't need a separate clone of an MCP-source repo.

## Version control

**Git is Memoria's canonical version-history layer.** Every reversibility property the system relies on — recoverable mistakes, audited diffs, the ability to roll a worker's write back — assumes a git history exists. This is non-negotiable; it's why the [policy MCP audit log](../01-architecture.md#permission-enforcement-the-policy-mcp) records `before_hash` and `after_hash` for every write, and why the [linter's M2 vault-hash-drift detector](../profiles/linter.md#structural-detectors-and-verdicts) treats a non-MCP write as CRITICAL.

- **In Git** (starter vault repo): the vault skeleton (`00-meta/`, `10-inbox/`, …); `.obsidian/` config; all of `.memoria/` (profiles, MCP source, lane overrides); installers; README.
- **Out of Git**: `.env` files in `~/.hermes/profiles/`, `auth.json`, secrets, the raw audit log (`00-meta/04-logs/audit.jsonl` — append-only log of operator-touchable writes), sessions, conversation memory in `~/.hermes/profiles/<name>/memories/`. These are local runtime state.
- **Generated locally, not committed**: `~/.hermes/profiles/memoria-*/` directories. These are recreated from the vault source on every `install.ps1` run.

**Sync ≠ version history.** Memoria's [deployment options matrix](../07-roadmap/deployment-options.md) gives three sync choices (manual git pull/push, Obsidian Sync, or Syncthing) — these are about *how the vault propagates between devices*, not about *how history is recorded*. Git is always the history layer regardless of which sync layer the operator picks. Running git's pull/push **as the sync layer** (Option A) at the same time as Obsidian Sync is a misconfiguration: two sync mechanisms race for the same files (most painfully `.obsidian/workspace.json`) and one will lose. Pick one sync layer; let git be history-only behind it.

## Related

- Vault folder taxonomy: [05-notes-folders.md](../05-notes-folders.md)
- Lane-override YAML format: [02-profiles.md lane-override files](../02-profiles.md#lane-override-files)
- The deferred compiler design (not currently in use): [reference/profile-compilation.md](../reference/profile-compilation.md)
