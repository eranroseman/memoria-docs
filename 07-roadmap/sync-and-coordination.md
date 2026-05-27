# Sync and write coordination

Under multi-machine [deployment options](deployment-options.md), agent writes and human writes need coordination. This file covers the three discipline mechanisms: the sync window mental model, the `.agent-lock` file, and the bib watcher (Option C only).

## Sync window: the 5–15 second mental model

When the agent runs on a VPS and the human is reading the same vault locally in Obsidian, writes propagate through Syncthing in roughly 5–15 seconds: ~1–2s for filesystem watcher detection, ~2–8s for network transfer, ~1–3s for Obsidian to reload. The full chain:

```text
Hermes writes file (VPS)
  → Syncthing detects change:   ~1–2s
  → Network transfer to desktop: ~2–8s
  → Obsidian detects new file:  ~1–3s
─────────────────────────────────────
Total:                          ~5–15s typical
```

What this means operationally:

| Scenario | What you see | How to handle |
| --- | --- | --- |
| Both sides idle for >30 seconds | Both nodes converged; identical view. | Normal case for chat-style queries. |
| You just edited a note in Obsidian | Hermes won't see it for 5–15s. | Wait a moment before asking Hermes about it. |
| Hermes just finished a batch | New files appear in Obsidian over 10–40s. | Visually obvious; watch the file explorer. |
| Hermes is mid-batch ingest | Mixed state — some files updated, some not. | Don't chat during active ingest; it confuses the conversation. |

Syncthing uses atomic file replacement on delivery, so Obsidian never reads a half-written file. The 5–15 second window is *clean lag*, not torn-write lag. The system doesn't need explicit coordination rituals for normal use.

## Agent write coordination (`.agent-lock`)

Under [Option C](deployment-options.md), the race between agent and human writes is **frequent, not rare** — the VPS is always-on and runs cron tasks, overnight loops, scheduled enrichment, and the bib watcher continuously. The operator on the desktop or laptop reads (and occasionally writes) the same vault while the VPS is mid-batch. Syncthing produces a recoverable conflict copy rather than silently overwriting, but conflicts compound over weeks. **`.agent-lock` is required discipline under C, not an optional habit.**

Under [Option A+](deployment-options.md), the race is rare-but-real — the primary desktop only writes when the operator is actively using it, and the desktop typically has long idle periods. `.agent-lock` is still recommended but the failure mode is much less frequent.

The mechanism: a `00-meta/.agent-lock` file. Hermes creates it before starting a batch write and removes it when finished. The human's Obsidian-side workflow (a Templater script, a [`board-state` dashboard](../dashboards/board-state.md) glance, or a manual habit) checks for the file before triggering its own writes.

```text
00-meta/
└── .agent-lock          # presence means "agent is mid-batch; defer your writes"
```

The discipline tightens with deployment option:

| Option | When to check `.agent-lock` | Why |
| --- | --- | --- |
| **A** (single machine) | Never — only one machine writes | No race possible |
| **A+** (desktop + laptop) | Before editing hot zones (`10-inbox/`, `40-workbench/01-projects/<active>/`) on the laptop while the desktop is on | Race exists but only when both machines are active |
| **B** (Obsidian Sync) | Same as A+ if you have a VPS for cron; otherwise no | Depends on whether anything else is writing concurrently |
| **C** (VPS + desktops + laptops) | Before any edit in zones the VPS is actively touching — `10-inbox/`, project-scratch for active projects, the current verify-cycle draft | VPS writes continuously; assume an edit collision is the default state unless you've checked |

Not airtight — a determined writer can ignore the lock — but adequate for a single-user workflow where the operator's main risk is *forgetting* the agent is running, not *deliberately* colliding with it. Under C specifically, the operator should treat checking the lock the same way they treat checking the dispatcher status before any edit in active zones.

## Bib watcher (Option C only)

When Zotero runs on the desktop and Hermes runs on the VPS, the VPS needs a current `library.bib` to ingest sources. Better BibTeX auto-exports the bib to `00-meta/03-config/library.bib` on every Zotero change; Syncthing delivers the file to the VPS within the standard sync window.

If Syncthing isn't between Zotero and the VPS (e.g., Option A migrating to a VPS without adopting Syncthing), use a small file watcher on the desktop that commits the bib to Git on change:

```bash
# 00-meta/03-config/watch-and-push-bib.sh
#!/usr/bin/env bash
set -euo pipefail
inotifywait -m -e modify "$VAULT/00-meta/03-config/library.bib" |
while read; do
  cd "$VAULT"
  git add 00-meta/03-config/library.bib
  git commit -m "bib: auto-export" || true
  git push
done
```

The VPS pulls before every ingest. This is the lowest-cost way to get a fresh bib to the VPS without committing to Syncthing — useful as a bridge between Option A and Option C.
