---
mode: how-to
audience: operator
topic: operations
---

# Failure modes — Detect / Fix / Verify

Operational reference for failures that block core workflows. Each entry follows a **Detect / Fix / Verify** triplet: the symptom an human notices, the commands to recover, and the check that confirms the recovery actually worked. This is the authoritative companion to `00-meta/04-reference/safe-mode.md` (the vault-resident human note); use safe-mode for the *first response* and this doc for the *full recovery procedure*.

The structure is borrowed from prior-iteration operational practice — symptoms alone are not enough. A failure recovery isn't complete until the verify step passes.

## Critical failures — detect, fix, verify

### 1. Stale `.bib` — ingest can't find the citekey

**Detect.** `hermes -p memoria-librarian run llm-wiki ingest --source {citekey}` returns "citekey not found" or "not in library.bib".

```bash
grep {citekey} .memoria/library.bib   # should return one entry
```

**Fix.** Export manually from Zotero (File → Export Library → Keep Updated), then push:

```bash
git add .memoria/library.bib
git commit -m "manual: bib update"
git push
git pull --ff-only   # on the agent node
```

**Verify.**

```bash
grep {citekey} .memoria/library.bib            # entry present
git log --oneline .memoria/library.bib | head -3   # recent commit
hermes -p memoria-librarian run llm-wiki ingest --source {citekey} --dry-run   # no "not found" error
```

### 2. Missing `_proposed_classification` — classification can't proceed

**Detect.** Note exists in `20-sources/01-papers/` with `lifecycle: proposed` but no `_proposed_classification` comment block in the file body. The classification query in [weekly-dashboard](../dashboards/weekly-overview.md) surfaces the note but it has no classification proposal to review.

**Fix.**

```bash
hermes -p memoria-librarian run classify --source {citekey}
# Re-runs classify on a single note; re-proposes classification from the abstract
```

**Verify.** Open the note — `_proposed_classification` comment block now present with `study_design`, `methods`, `topic`, and `projects` fields populated. `lifecycle` remains `proposed` (correct — it only becomes `current` after the human promotes).

### 3. Broken frontmatter — note missing from Dataview queries

**Detect.** Obsidian Properties panel shows a YAML parse error. Note does not appear in Dataview queries that should include it.

```bash
hermes -p memoria-linter run lint --source {citekey} --dry-run
# Reports YAML structure issues
```

**Fix.** Open the raw file in an editor outside Obsidian (Obsidian masks the raw YAML). Common causes: unclosed string (`title: "Unterminated`), list indentation error, missing closing `---` delimiter. Fix manually, save.

**Verify.**

```bash
hermes -p memoria-linter run lint --source {citekey} --dry-run   # no YAML errors reported
```

In Obsidian: Properties panel shows no error; note appears in a Dataview query (`FROM "20-sources/01-papers" WHERE file.name = "{citekey}"`).

### 4. Stale `qmd` index — vault search returns empty or outdated results

**Detect.** `hermes -p memoria-writer run draft "known topic"` returns no vault results despite relevant notes existing.

```bash
qmd search "known term" --vault {vault-path}
# Returns empty or omits notes you know exist
```

**Fix.**

```bash
cd {vault-path}
qmd index .
# Full rebuild — 1–5 minutes for 500+ notes; 10–15 minutes for 2000+ notes
```

**Verify.**

```bash
qmd search "known term"   # returns expected notes
hermes -p memoria-writer run draft "known topic"   # returns vault results with citekeys
```

### 5. Profile install drift — deployed `SOUL.md` doesn't match vault source

**Detect.** Linter `profile-install-drift` reports a SHA-256 mismatch between `.memoria/profiles/memoria-<name>/<file>` (vault source) and `~/.hermes/profiles/memoria-<name>/<file>` (deployed copy).

**Fix.** Either (a) the source changed in the vault and `install.ps1` hasn't been re-run — execute `./install.ps1` to redeploy all seven profiles; or (b) someone hand-edited the deployed copy — `diff` the two files, then either copy the change back into `.memoria/profiles/memoria-<name>/` and commit (promoting the edit) or just re-run `install.ps1` (discarding the edit).

**Verify.**

```bash
hermes -p memoria-linter run health-report --detectors profile-install-drift
# No drift reported
```

## All failure modes

Sorted by severity (most urgent first), then by topic. The **Severity** column uses the same scale as the Linter's [verdict band](../profiles/linter.md#severity-scale): **CRITICAL** (vault integrity or security at risk) → **HIGH** (silent-failure mode; the system appears to work but doesn't) → **MEDIUM** (maintenance debt; works now, will bite later) → **LOW** (recoverable in one command; usually expected after a routine change).

| Symptom | Severity | Cause | Fix |
| --- | --- | --- | --- |
| **Obsidian Linter corrupts frontmatter** | CRITICAL | Linter running on agent-maintained folders | Exclude `10-inbox/`, `20-sources/`, `30-synthesis/02-reference/` in Linter `foldersToIgnore` — see [obsidian-plugins/README.md](../plugins/README.md). |
| **`_proposed_classification` or `_enrichment` deleted** | CRITICAL | A Linter rule or plugin upgrade introduced an HTML-comment stripper that ran on save | **Never enable any rule that strips HTML comments.** Currently no such rule exists in obsidian-Linter v1.31.2, but the discipline is forward-looking — diff the rule registry before accepting any plugin upgrade. See [obsidian-plugins/README.md](../plugins/README.md). |
| Enrichment block empty after ingest | HIGH | API keys not set in environment (silent — ingest "succeeded" but with degraded data) | `echo $OPENALEX_EMAIL`; check per-profile `.env` |
| Dataview queries returning nothing | HIGH | `study_design` or `topic` vocabulary inconsistency — query returns empty table that looks like "nothing to do" | Check values in notes match the schema vocabulary exactly (see [frontmatter-schema.md](../vault/frontmatter-schema.md)). |
| `audit.jsonl` growing without bound | HIGH | Linter log rotation not running (silent until disk fills) | Check [linter.md log rotation section](../profiles/linter.md#log-rotation); the Linter rotates weekly. |
| Obsidian-agent-client can't connect | MEDIUM | ACP server not running or tunnel down | `systemctl --user status hermes-acp` and `hermes-tunnel` |
| `_proposed_classification` not appearing | MEDIUM | `classify` skill not installed or not in lane's allow list | `hermes skills install classify`; check `.memoria/lane-overrides/library.yaml` |
| Syncthing + `.bib` race condition | MEDIUM | VPS reads `.bib` while Syncthing is mid-transfer | Use Git pull for `.bib` distribution on the `always-on` option, not Syncthing — see [sync-and-coordination.md](../roadmap/sync-and-coordination.md#bib-watcher-always-on-only). |
| VPS tunnel drops on WSL2 restart | MEDIUM | systemd user service not auto-starting | `systemctl --user enable hermes-tunnel`. |
| Schema version mismatch in Dataview | MEDIUM | Notes on old schema version | `hermes -p memoria-linter run schema-migrate --dry-run` → review proposed field additions → run without `--dry-run` on a single folder first. |
| Cron job didn't fire overnight | MEDIUM | Sleep-prone host or stale `.env` | `always-on` option only (VPS); check `journalctl --user -u hermes-overnight` and the [discovery loop section](../roadmap/future-directions.md#the-discovery-loop). |
| Retry count climbing on same card | MEDIUM | Brittle prompt or broken tool | After `retry_count > 3` the card auto-escalates to `blocked-on-human` — see [board/README.md retry pattern](../board/README.md#retry-pattern). The human decides whether to revise the packet or close as infeasible. |
| Citekey not found at ingest | LOW | `.bib` not updated or not pulled | See failure mode #1 above. |
| `_enrichment` fields not queryable | LOW | `_enrichment` is a YAML comment block, not real frontmatter (design constraint, not a defect) | Use note modification date (`file.mtime`) as a proxy, or promote specific fields (e.g., `enriched_date`) to main frontmatter. |
| Pandoc + BBT DOCX corrupt | LOW | Known Pandoc/Better BibTeX issue with some citation styles | Rerun Pandoc; test on a single-citation document first to isolate. |
| High Scite contrast flag | LOW | Paper is actively disputed (a curation signal, not a system failure) | Open paper in Zotero → check what Scite classifies as contrasting → decide whether to include as primary or supporting evidence only. |
| Profile install drift after edit | LOW | Vault source changed but `install.ps1` not re-run | See failure mode #5 above. |
| Bitwarden bootstrap token rejected | LOW | Wrong region or revoked token | Re-run `hermes secrets bitwarden setup` and pick the correct region (US Cloud / EU Cloud / self-hosted) — see [roadmap/README.md secret management](../roadmap/secret-management.md). |

## When this doc is wrong

If a failure recurs and the Fix here doesn't work, treat that as a design issue. Either the symptom mapping is incomplete (the doc points at the wrong cause) or the fix is brittle (works sometimes but not always). In both cases the right response is to update this doc, not to memorize a workaround — the next human who hits it should find a recipe that actually works.

The discipline: never let a stale fix sit in this doc. If a command changed (e.g., a renamed flag) or a service was replaced (e.g., Bitwarden → 1Password), update or remove the entry the same session you noticed the drift.

<!-- memoria-nav -->

---

[← Previous: Operations](README.md)

[Next: Session logging →](session-logging.md)
