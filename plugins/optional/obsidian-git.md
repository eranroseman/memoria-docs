---
mode: how-to
audience: operator
topic: plugins
---

# obsidian-git

Load-bearing settings (apply on every deployment):

- `commitMessage` — keep a stable template like `"vault: {{date}} {{numFiles}} files"`. Random commit messages clutter the history, and `{{numFiles}}` is genuinely diagnostic — a 200-file auto-commit means something unusual happened, and the human should notice in the git log.
- `autoBackupAfterFileChange: false` — don't auto-commit on every file change; that fights with Hermes writes and produces hundreds of commits per session. Use scheduled commits instead (`autoSaveInterval: 30` minutes is sensible).
- `pullBeforeCommit: true` — fetch and merge before committing. Defensive against multi-machine drift: catches the case where a different machine pushed changes since the last local pull, before the local commit creates a divergence.
- `pullBeforePush: true` — same defense at push time. The pair (`pullBeforeCommit` + `pullBeforePush`) is what makes multi-machine setups survive without merge conflicts becoming a daily ritual.
- `autoPullOnBoot: true` — pull when Obsidian starts. Catches "I opened the vault on a different machine and started writing without thinking about sync" — the most common multi-machine failure mode.

Deployment-conditional settings (vary by [deployment option](../../roadmap/deployment-options.md)):

- **local-only:** `autoPush: false`. There's no remote to push to. obsidian-git is used for local history, not sync.
- **obsidian-sync:** `autoPush: false`. Obsidian Sync handles cross-device sync; git is for history and auditing only. Push manually when a checkpoint is wanted.
- **always-on:** `autoPush: true`. Auto-push pairs with `autoPullOnBoot` for round-trip sync between desktop and VPS. The VPS-side instance pulls on its own cron; the desktop pushes as it works.

Per-machine override:

- `disablePush: true` on the VPS instance, even under the always-on option. The VPS should `git pull` only — letting it push would create writes that bypass the desktop's policy MCP for review and approval. Local-edit-and-push is the desktop's privilege.

Inline `data.json` — desktop instance, the always-on option:

```json
{
  "commitMessage": "vault: {{date}} {{numFiles}} files",
  "autoCommitMessage": "vault: {{date}} {{numFiles}} files",
  "autoSaveInterval": 30,
  "autoSave": true,
  "autoBackupAfterFileChange": false,
  "autoPush": true,
  "autoPushInterval": 0,
  "autoPullInterval": 0,
  "autoPullOnBoot": true,
  "pullBeforeCommit": true,
  "pullBeforePush": true,
  "syncMethod": "merge",
  "commitDateFormat": "YYYY-MM-DD HH:mm:ss",
  "disablePopups": false,
  "listChangedFilesInMessageBody": false,
  "showStatusBar": true,
  "updateSubmodules": false,
  "customMessageOnAutoBackup": false
}
```

For local-only / obsidian-sync desktop or the VPS-side instance, change `autoPush` to `false` and (on the VPS) set `disablePush: true`. No separate shipped template — the variation is small enough that one inline snippet plus the conditional table above is clearer than four parallel files.

<!-- memoria-nav -->

---

[← Previous: tag-wrangler](../recommended/tag-wrangler.md)

[Next: obsidian-kanban →](obsidian-kanban.md)
