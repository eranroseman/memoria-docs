# Workflow 10: Maintenance and linting

**Group.** Maintenance
**Goal.** Keep the vault structurally healthy and queryable.

## Steps

1. Hermes (linter) runs lint checks on a schedule or post-ingest.
2. Reports orphans, stale enrichment, partial triage, broken links, schema issues.
3. Human reviews the report.
4. Decides whether to auto-fix or manually correct.

## Owners

Hermes for detection and reporting. Human for approval of fixes.

## Command

`hermes run lint --dry-run`.

## Weekly ritual

A practical weekly cadence that exercises this workflow end-to-end. Approximate time: 30–60 minutes once the vault is established.

| Order | Step | Goal | Approximate time |
| --- | --- | --- | --- |
| 0 | Read `research-directions.md` | Confirm or update the week's priorities. | 2 min |
| 1 | Open `weekly-dashboard.md` | Surface the queues. | < 1 min |
| 2 | Clear **unreviewed synthesis** | Review or discard inbox drafts. | 10–15 min |
| 3 | Process **discovery candidates** | Include or exclude with a reason. | 5–10 min |
| 4 | Resolve **triage debt** | Promote draft fields; mark `triage_status: full`. | 10–15 min |
| 5 | Promote **evergreen claim notes** | Move qualifying notes to `30-synthesis/02-wiki/`. | 5–10 min |
| 6 | Review **orphan notes** | Add MOC links, concept links, or archive. | 5 min |
| 7 | Inspect **stale literature and items** | Refresh enrichment or archive. | 5 min |
| 8 | Run lint dry-run | Catch structural issues. | 1 min + review |

The ritual isn't a separate workflow — it's the operating cadence that drives this workflow (and touches [#4 Triage](04-triage.md), [#6 Promote](06-promote-wiki.md), [#11 Merge/split/prune](11-merge-split-prune.md)). The dashboards in [06-surfaces.md](../06-surfaces.md) are the surfaces; this is the schedule.

## Daily rhythm

The weekly ritual is one of three rhythms. The other two — morning glance and per-session focus — are shorter and more frequent. Together they describe what using Memoria *feels like* day-to-day.

**Pre-morning, on phone over coffee (optional).** If overnight cron produced anything worth pushing — a retry threshold hit, a drift alarm, a substantive ingest summary — Telegram pushed a notification before the operator opened the vault. Example: *"Overnight: 3 sources ingested, 12 link suggestions ready, 1 retry threshold hit on card-2026-05-26-042 (Researcher timeout fetching DOI for Tanaka 2024)."* The operator glances, notes anything blocking, deals with it at the desk. If nothing pushed, nothing demands attention before opening the vault — that's the design.

**Morning glance (5–10 minutes).** Open the vault. The Operator workspace appears by default (`Cmd-1`). Glance at [`index.md`](../dashboards/index.md): is any section red? Today's queue, drift signals, lane health, cron status. Most days nothing is red, and the index closes again. If link suggestions accumulated overnight from Researcher's enrichment work, bulk-approve the ones that look right via `Memoria: approve all link suggestions` (see [command-palette.md](../reference/command-palette.md#maintenance-3-commands)). For retries flagged in the Telegram push, drop to CLI: `hermes kanban show <card-id>` to inspect, fix the underlying issue, `hermes kanban retry <card-id>` to release. Total time: under ten minutes unless something demands attention.

**Reading session (1–2 hours, when scheduled).** Switch to the Reading & Processing workspace (`Cmd-2`). Open [`process-queue.md`](../dashboards/process-queue.md): which literature note is ripest for processing? Read the source with the `[!brief]` callout in mind. When ready to process, invoke Socratic on the active note (`Cmd-P → Memoria: invoke Socratic on current note`) — the ACP pane opens on the right with the Socratic profile, which is architecturally write-denied. The conversation runs; the operator writes the claim note themselves in `30-synthesis/01-permanent/` (in their own words, in the left pane) as the conversation progresses. Save. The git hook fires; Researcher's enrichment runs overnight; the link suggestions appear in tomorrow's morning glance.

**Walking (whenever a thought hits).** Telegram: `/fleeting <thought>` drops the text into `10-inbox/01-fleeting/` with a timestamp. The thought is captured; the operator doesn't have to remember it through to the next desk session. It surfaces in tomorrow's [`process-queue.md`](../dashboards/process-queue.md) (or the weekly fleeting-triage step) for action. Source-URL capture works the same way: paste the link, get a confirmation, the actual ingest happens overnight.

**Writing session (project work).** Switch to the Drafting workspace (`Cmd-3`). Open the draft. The `[!verification]` callout at the top shows Verifier's last claim-trace report. Write. The Writer ACP pane is available on the right if local critique is wanted, but it's optional — most drafting happens in the operator's head, with the pane silent. On significant edits, save and `git commit`. Verifier picks up the draft automatically ([#17 Verify](17-verify.md)). Gap cards from Verifier appear in tomorrow's morning queue. If the gap loop suggests more reading before the draft can stabilize, switch back to a reading session.

**Friday ritual (90 minutes).** The [weekly ritual](#weekly-ritual) above. Top to bottom in `weekly-dashboard.md`.

**Evening (passive).** If the Linter's weekly drift sweep ran on schedule, Telegram pushes a short confirmation: *"Drift report for the week is ready."* If verdict is `PASS` (green), the operator glances and ignores. If `REVIEW` or `FAIL`, the message is the signal to open [`drift-watch.md`](../dashboards/drift-watch.md) next desk session. The push exists so the operator knows the sweep happened — silence would be indistinguishable from a missed cron run.

**What the operator does NOT do:**

- Manage agent state by hand. The Kanban dispatches; the operator approves or redirects via card transitions.
- Re-decide policy questions. The lane-overrides and the policy MCP encode them once.
- Hunt for files. Folders encode lifecycle; search finds the rest.
- Switch between mental models for "what tool am I using." One vault, one keyboard, one set of Memoria commands.

If after three months of use the operator's mouse hand barely moves and they've stopped consciously tracking which workspace they're in, the rhythm is right. The architecture is invisible during normal use; the dashboards light up only when something specific demands attention.

## Related

- **Profile:** [profiles/linter.md](../profiles/linter.md)
- **Dashboards:** [drift-watch.md](../dashboards/drift-watch.md), [weekly-dashboard.md](../dashboards/weekly-dashboard.md), [index.md](../dashboards/index.md)
- **Standard cron tasks:** [07-roadmap/standard-cron-tasks.md](../07-roadmap/standard-cron-tasks.md)
