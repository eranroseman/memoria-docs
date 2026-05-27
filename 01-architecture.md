# 01 — Architecture

## Three layers

Memoria has three layers. They are connected through explicit handoffs but never collapsed into one.

```text
┌─────────────────────────────────────────────────────────────────┐
│  Board layer (Kanban) — orchestration and memory of active work │
│  states: pending → ready → active → awaiting-review →           │
│          approved → done                                        │
│  (rejected, retry-needed, blocked-on-human are off-path)        │
└────────────────────────────┬────────────────────────────────────┘
                             │ assigns lane / advances state
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Worker layer (Hermes) — seven profiles execute work in lanes  │
│  researcher · cartographer · socratic · writer · verifier ·    │
│  coder · linter                                                │
└────────────────────────────┬────────────────────────────────────┘
                             │ every write checked by the policy MCP
                             │ (allow / allow_with_log / deny / dry_run)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Vault layer (Obsidian) — durable knowledge by lifecycle stage  │
│  00-meta · 10-inbox · 20-sources · 30-synthesis · 40-workbench  │
│  50-deliverables · 90-assets · 95-archive                       │
└─────────────────────────────────────────────────────────────────┘
```

## Why three layers, not one

A single-agent or single-document system blurs three different concerns: *what work is in progress*, *who is doing it*, and *what stable knowledge has accumulated*. Memoria treats each as its own layer with its own semantics.

- **The board never holds knowledge.** It tracks work. Cards die at `done`; knowledge lives in the vault.
- **The workers never hold permanent state.** They claim cards, act, and release. Continuity comes from the board (in-flight) or the vault (settled).
- **The vault never schedules work.** It is the destination, not the orchestrator.

This separation is what makes retries safe, handoffs lossless, and review enforceable.

### Thin control over thick state

A one-line characterization of this design, borrowed from Chen et al. 2026 (*Toward Autonomous Long-Horizon Engineering for ML Research*): **thin control over thick state.** The orchestrator and the workers carry as little persistent context as possible; the durable knowledge — plans, claim notes, drafts, audit traces — lives in files. Workers re-ground on those files between steps rather than relying on conversational handoffs.

This is not just a Memoria-internal preference. Chen et al.'s ablation removes their *File-as-Bus* protocol (the same shape as the Memoria vault layer) and measures the consequence: PaperBench drops by 6.41 points and MLE-Bench Lite by 31.82 points. The same conclusion is reached independently by AgentRxiv (Schmidgall & Moor 2025), which shows that agents reading prior agent-generated reports gain ~11% over isolated agents on MATH-500. A third independent confirmation comes from PARNESS (Wang & Luan 2026), whose entire design rests on naming "no existing tool persists cross-run knowledge in a form that can be retrieved into a finite LLM context" as one of the field's five structural problems and addressing it with a persistent knowledge layer. Three unrelated systems, three architectures, one finding: long-horizon agent work fails when state lives in chat and succeeds when state lives in files.

Memoria's three-layer split is the structural form of that finding. See [rationale/pattern-provenance.md](rationale/pattern-provenance.md) for the borrow/adapt/ignore mapping that places each of these systems against Memoria's design choices.

## Layer 1: Board (Kanban)

The board is the control plane. It persists every task as a card with state, assignee, blocker, retry count, and handoff note, and it keeps the card alive until the human review gate is passed.

Recommended states:

- `pending` — card created, specification in progress; dispatcher ignores.
- `ready` — specified and dispatchable.
- `active` — owned by one profile.
- `blocked-on-human` — needs a human decision the worker cannot make.
- `awaiting-review` — ready for operator review (or Linter dry-run report).
- `rejected` — operator declined this review pass; revise and return.
- `retry-needed` — failed but reusable (same card, not a new one).
- `approved` — human accepted.
- `done` — canonical, archived, or shipped.

The key rule: `awaiting-review` is a real state, not a label. A card can be worked on, blocked, returned, and retried without losing history or confusing completion with approval.

See [03-board.md](03-board.md) for full board schema, state transitions, and the review gate rules.

## Layer 2: Workers (Hermes profiles)

Hermes is split into seven specialist profiles rather than one generalist agent. Each profile has narrow permissions, a focused command surface, the MCPs it actually needs, and a clear exit condition.

| Profile | Role |
| --- | --- |
| **Researcher** | Discovers, ingests, enriches, and classifies sources. Exits at `awaiting-review`. |
| **Cartographer** | Maps the corpus for a project: scope reports, gap reports, cluster maps. Read-only across vault except project scratch. |
| **Socratic** | Questions the operator about a literature note or framing. Write-denied across the entire vault. Invoked synchronously. |
| **Writer** | Synthesizes evidence into drafts, synthesis notes, and wiki-ready prose. Cannot canonize. |
| **Verifier** | Traces draft claims to claim notes; verifies citations; flags duplicates and retractions. Read-only across vault except verification reports. |
| **Coder** | Builds and maintains code artifacts, scripts, project scaffolding. Cannot edit canonical synthesis. |
| **Linter** | Validates structure, metadata, schema, link health; owns session and audit-trail housekeeping. Default is dry-run. |

There is no Orchestrator profile and no Reviewer profile. Routing lives in lane-overrides + Kanban dispatch (not a reasoning agent); review gates live in the policy MCP and the board state machine (not a dedicated profile). See [02-profiles.md](02-profiles.md#routing-without-an-orchestrator) for the routing rule, and [02-profiles.md](02-profiles.md) for per-profile detail.

## Layer 3: Vault (Obsidian folders)

The vault stores durable knowledge. Folders encode lifecycle stage, not subject area — see [05-notes-folders.md](05-notes-folders.md) for the canonical layout, including the full tree, subfolder roles, and access matrix.

| Folder | Role |
| --- | --- |
| `00-meta/` | Templates, CSL, config, logs, dashboards, schema. |
| `10-inbox/` | Fleeting captures, synthesis drafts, discovery candidates. |
| `20-sources/` | One zone for everything that describes the world: literature, items, entities. |
| `30-synthesis/` | One zone for everything that expresses your thinking: claim notes, reference notes, MOCs. |
| `40-workbench/` | Active work: projects, drafts, code, canvas. |
| `50-deliverables/` | Finished outputs: manuscripts, presentations, exports. |
| `90-assets/` | Attachments and binary assets. |
| `95-archive/` | Deprecated, superseded notes. |

The grouping is load-bearing. `20-sources/{01-literature, 02-items, 03-entities}` says "everything that describes something external." `30-synthesis/{01-permanent, 02-wiki, 03-moc}` says "everything that expresses your thinking." `40-workbench/{01-projects, 02-drafts, 03-code, 04-canvas}` says "things being worked on." This makes ownership and access policy easier to enforce.

See [05-notes-folders.md](05-notes-folders.md) for the full layout, note types, templates, and linking patterns.

## Surfaces: each one owns one mode

The three layers above are *what the system is*. Surfaces are *where the human sees it and acts on it*. Five surfaces cover the operator's interaction with Memoria. The discipline that keeps the system from feeling like a control panel: **each surface owns one cognitive mode.** Using a surface for the wrong mode (Telegram for desktop work, CLI for daily ops) produces the "every operation feels slightly off" drift that erodes a workflow.

| Surface | Mode | Use it for |
| --- | --- | --- |
| **Obsidian dashboards + ACP panes** | Desktop, focused, deliberate | Daily triage, reading, authoring, agent conversations on the active note. |
| **Command palette** (Obsidian) | Desktop, instant, frequent | The five-to-ten most-used actions: capture fleeting, invoke Socratic on current note, lint current note, new project. |
| **CLI** (`hermes …`) | Desktop, occasional, precise | Forensic queries against the audit log, profile administration, manual dispatch, anything benefiting from precision over discoverability. |
| **Telegram** | Mobile, async, lightweight | Fleeting capture on the go, source-URL capture, urgent push notifications (retry threshold hit, drift alarm, cron failure). Not for drafting or review. |
| **API server** (port 8642) | Programmatic, integration | File-system watchers, Zotero hooks, git post-commit, cross-machine dispatch. Never the surface for direct human action. |

Inside Obsidian, the four-type surface taxonomy (persistent dashboards, modal workspaces, inline callouts, ambient status bar) is the operator-facing companion to this table. The two views are complementary: this table answers "which channel," and [06-surfaces.md](06-surfaces.md) answers "what kind of thing appears where, inside Obsidian."

### Surface failure modes

The discipline is consciously picking the right surface for each operation. The failure modes happen when one surface gets used for another's job:

- **Telegram for things that need the desktop** → ignored notifications. Once Telegram has cried wolf with a message that should have been a dashboard query, the operator starts ignoring all Telegram messages — including the urgent ones.
- **CLI for things you do daily** → eroded habit. If `hermes kanban ...` is the operator's path to approving link suggestions, the friction of dropping to a terminal compounds across the day; eventually the suggestions stop getting approved.
- **Dashboards for things that need a script** → manual repetition that should be automated. A weekly "check this query, copy these card IDs, run this command" pattern is an API integration the operator hasn't written yet.
- **API directly when a command exists** → reinventing wheels. Curl-ing the API to do what `Memoria: capture fleeting` already does is technical debt disguised as flexibility.

The corrective is one question per operation: "what surface is this *kind* of work?" Most failures are answered by the table above.

Two principles:

- **Notifications must change what you'd do in the next 30 minutes**, or they shouldn't be notifications. Routine approvals wait for the dashboard; only hard blockers and drift alarms page Telegram.
- **An admin GUI, if ever built, is an Obsidian plugin — not a separate web app.** Extending Obsidian-as-interface composes cleanly with the existing five surfaces; adopting a peer system (e.g., a hackathon-grade Hermes-admin web UI) would create a sixth surface competing for the same modes, and a chat tab that bypasses the policy MCP entirely. The recommended shape — a read-only sidebar pane reaching the API on loopback — lives in [07-roadmap/future-directions.md](07-roadmap/future-directions.md#memoria-inspector-obsidian-plugin).

### CLI is the forensic surface

The CLI surface is for **rare, precise operations** — debugging stuck cards, inspecting audit trails, manual dispatch outside the normal workflow. The discipline: CLI is right for forensic work because forensic work is rare and benefits from precision. A UI for rare operations is mostly empty real estate; a CLI for rare operations is invisible until needed and then exactly the right shape.

Six use case categories cover what the CLI is for:

**1. Card inspection.** When a card has been sitting in `retry-needed` for two days, you want to know why. `hermes kanban show card-<id>` returns full state, retry count, blocker reason, and handoff notes in one screen — faster than navigating to the board dashboard and clicking through.

**2. Lane health checks.** `hermes lane status researcher` shows [trust score](reference/glossary.md#observability-and-verdicts), recent deny rate, last successful task, current queue depth. Run when something feels off but the dashboards haven't flagged anything yet.

**3. Audit forensics.** `hermes audit --card <id>` or `hermes audit --lane cartographer --since 24h` walks the audit log filtered to the slice you care about. Faster than opening the audit-log dashboard for narrow queries; the dashboard is for trends, the CLI is for specific traces.

**4. Manual dispatch.** `hermes dispatch --lane cartographer --task scope-project --project jitai-review` creates a card without waiting for a file-system trigger or cron. Useful when the operator wants to invoke something on demand outside the normal flow — for example, re-running a scope when new sources have been added mid-project.

**5. Profile administration.** Update lane-override files in `.memoria/lane-overrides/`, reload the policy MCP, edit profile sources in `.memoria/profiles/memoria-<name>/` and re-run `install.ps1` to deploy them, install new skills. All CLI operations, not dashboard ones, because they're rare and consequential — exactly the kind of operation that should require typing.

**6. Backup and migration.** Vault snapshots, audit log archival, profile config exports, schema migrations. CLI is the right home for anything that's "do once carefully."

Example commands across the categories:

```bash
# 1. Card inspection
hermes kanban show card-2026-05-26-042

# 2. Lane health
hermes lane status researcher

# 3. Audit forensics
hermes audit --card card-2026-05-26-042
hermes audit --lane verifier --since 7d

# 4. Manual dispatch
hermes dispatch --lane cartographer --task scope-project --project jitai-review

# 5. Profile administration
./install.ps1    # re-deploy all seven profiles from .memoria/profiles/
hermes profile reload memoria-linter

# 6. Backup / migration
hermes audit export --since 2026-01-01 --to logs/2026-archive.jsonl
```

These are examples, not a canonical catalog. The full Hermes CLI surface is upstream of Memoria (documented at [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com/)); Memoria pins specific commands only as illustrations. When the Hermes CLI changes between versions, Memoria docs may lag — the principle (CLI for forensic work) stays true even when the specific flag names shift.

**What CLI is NOT for.** Daily operations (capture, processing, drafting) go through the command palette inside Obsidian — see [`reference/command-palette.md`](reference/command-palette.md). Triage of approval queues belongs in the dashboards plus inline callout buttons, not the terminal. Reading content belongs in Obsidian; opening files in a terminal is hostile.

**Mental model.** CLI is the surgical tool. Sharp, precise, occasional. If the operator finds themselves using the CLI more than a few times a week, something else (the dashboards, inline UI, or command palette) needs improving — the friction of dropping to a terminal is a signal that a more frequent operation lacks its proper surface.

### Telegram has two distinct uses

Telegram is the asynchronous, mobile-reachable surface. It serves two distinct purposes that are worth keeping separate in the operator's mental model — they have different disciplines and different failure modes.

#### Use 1: Notifications that can't wait

The dashboards tell the operator what needs attention when they open them. Telegram pushes the operator when something can't wait until the next dashboard glance. The distinction matters because over-notifying turns Telegram into noise the operator ignores — which is strictly worse than not having notifications at all.

**Wire notifications for:**

- **Hard blockers.** A card hit retry threshold (>3 attempts) and auto-moved to `blocked-on-human`. The policy MCP denied an unexpected write (security signal). A skill broke catastrophically.
- **Time-sensitive workflows.** The operator triggered an overnight ingest of 30 papers and wants a morning summary when it's done, not on their next dashboard check.
- **Drift alarms.** Linter's M-series detectors found something at HIGH or CRITICAL severity. Audit deny rate spiked above baseline.
- **Cron failures.** A scheduled task (nightly hygiene, weekly drift report) didn't run, or ran but failed. The operator wants to know before the next day's cron cycle.

**Do not wire notifications for:**

- Anything that surfaces in the morning [`index.md`](dashboards/index.md) glance.
- Per-card events ("new card created", "card moved to active"). Volume kills signal.
- Routine approvals. Those wait until the operator chooses to do them; the queue surfaces them in the daily/weekly dashboards.

**The discipline, in one sentence.** If a notification doesn't change what the operator would do in the next 30 minutes, it shouldn't be a notification.

#### Use 2: Mobile capture and lightweight interaction

Telegram is where the operator reaches Hermes when they're not at their desk. The interactions that fit a phone screen:

- **Fleeting capture.** `/fleeting save: <thought>` drops the text into `10-inbox/01-fleeting/` with a timestamp. Zero friction; the operator types and the thought is captured.
- **Source capture from URL.** Paste a link, get back a confirmation that it's queued for ingest. The actual ingest happens overnight; the phone interaction is just the queue-up.
- **Quick lookups.** "What do I have on therapeutic alliance?" — Cartographer returns a short list. Useful when the operator is in conversation and wants to reference their own work.
- **Socratic conversations on the go.** The killer feature. Invoke Socratic on a literature note from the phone while walking, have the conversation, then come back to the desk and write the permanent note. The thinking happens in motion; the artifact happens at the desk.
- **Approval triage on long queues.** If 30 approvals have accumulated over a busy week, the operator can clear them on a train ride — card, quick summary, approve or reject, next. Phone screens are right-sized for binary decisions.

**Doesn't belong on Telegram:**

- Drafting. Phone keyboards are wrong for prose.
- Reviewing drafts. Too much context for the screen.
- Anything that needs the dashboard view. Use the desktop.

**Mental model.** Telegram is the always-available channel for things that fit on a phone and notifications that can't wait. Most of the value comes from a small number of well-chosen capture and Socratic flows, not from being a generic chat interface.

#### Telegram toolset is narrower than CLI

Hermes profiles can expose the same capability surface (browser, code execution, terminal, delegation, memory, skills, etc.) through any registered surface. Telegram is the **single surface where the toolset should be deliberately narrower** than the host profile. Mobile is for thinking and capture; running code, opening browsers, or shelling out from a phone is a footgun without commensurate value — the small screen makes auditing the action's effects unreliable, and the always-on connectivity makes accidental triggers easier than at the desk.

Memoria's recommended Telegram toolset per profile (subset of the full surface):

- `clarify` — ask the operator a follow-up question
- `memory` — read profile / lane / project memory
- `messaging` — send replies
- `todo` — read or update task lists (does *not* include scheduling external commitments; that's Todoist)
- `session_search` — look up prior conversations
- `skills` — invoke pre-approved skills and switch to read-only profiles (notably Socratic for on-the-go thinking via `lens-reading` with a chosen lens)

Explicitly **not** on Telegram: `code_execution`, `terminal`, `delegate_task`, `web_search`, `fetch_url`. These are desktop-CLI tools; routing them through Telegram is what produces the "I dispatched a long-running job from my phone and forgot about it" failure mode. The narrowing is enforced per profile in the Hermes runtime config, not just by convention.

#### Other messaging surfaces

Hermes can integrate with Discord, WhatsApp, Slack, Signal, Teams, and others. **Leave them off until there's a concrete need.** Each enabled surface is another channel that competes for the operator's attention; each one becomes a place notifications can land, and every channel demands its own discipline about what to wire. Telegram covers the mobile-async case adequately for a single-operator vault. If a collaborator joins, or a specific channel is the operator's existing primary (e.g., a Slack workspace), enabling that channel becomes worth the attention cost — but never as a parallel to Telegram. One mobile-async channel is the working set.

### API server: integration and automation

The API server on port 8642 is where Hermes connects to other systems. It's the least-used surface day-to-day but the most enabling for everything else — most of Memoria's "this just happens automatically overnight" magic relies on the API being reachable to triggers, watchers, and hooks.

Seven integration patterns cover what the API is used for:

**1. File-system triggers.** A watcher script monitors `10-inbox/00-pdfs/` as a side-channel capture surface — for PDFs the operator receives outside Zotero (downloads, colleague handoffs). New file appears → watcher imports it into Zotero (via the Zotero HTTP connector or `zotero-cli`) so Better BibTeX can assign a citekey → POST to the API → card created in Researcher's queue → standard ingest runs (Marker extract, source-note creation, frontmatter URIs). The dropped PDF leaves the vault during the Zotero-import step; the canonical PDF store remains Zotero (see [04-workflows/01-zotero-capture.md](04-workflows/01-zotero-capture.md)).

**2. Zotero hooks.** Better BibTeX can run a script on save. Wire it to POST to the API when a new entry is added. Researcher picks up the citekey and runs ingest. The Zotero workflow becomes the Memoria workflow with no extra effort.

**3. Email-to-Memoria.** A custom mail filter forwards arXiv alerts to a script that extracts the arXiv ID and POSTs to the API. Subscription feeds become candidate-discovery cards automatically.

**4. Git hooks.** On commit to `40-workbench/02-drafts/`, a `post-commit` hook POSTs to the API to create a `verify` card. Verifier picks it up; the operator gets a verification report by the time they next open the draft.

**5. Calendar integration.** If the operator uses a research-block calendar, a script can check the calendar and create a daily card with "today's reading queue" pulled from [`process-queue.md`](dashboards/process-queue.md). Useful for keeping rhythm against scheduled focus time.

**6. Cross-machine dispatch.** If the operator works across laptop and desktop, the API is how a command on one machine creates a card the other machine's Hermes instance picks up. Assumes a shared vault (Syncthing, git, or Obsidian Sync per the [deployment options](07-roadmap/deployment-options.md)).

**7. Custom dashboards or scripts.** Anything that needs to query Kanban state programmatically reads from the API. Cleaner than parsing the vault filesystem directly — the API returns structured data; the vault has notes that happen to encode state.

**What API is NOT for:**

- **Direct human interaction.** That's CLI, command palette, or Telegram. The API is for *programs* invoking Memoria, not operators.
- **Bypassing the policy MCP.** Every write that enters through the API still goes through the policy MCP. The API doesn't grant elevated permission; it's just another caller subject to the same lane-override rules.

**Security and binding.** Per the [fail-closed startup](01-architecture/control-plane.md#fail-closed-startup) rules, the API binds to `127.0.0.1` by default. Non-loopback binding requires `HERMES_BRIDGE_TOKEN` to be set. For a laptop that travels (coffee shops, conferences), loopback is the right default — local scripts can reach the API; nothing on the network can. Cross-machine integration earns its non-loopback binding only with explicit token configuration.

## On-disk layout

Memoria spans **two filesystem locations**: the starter vault (versioned, holds all install material including the seven hand-authored profile dirs under `.memoria/profiles/`), and the user's Hermes runtime (per-user, holds installed profile copies written by `install.ps1`) at `~/.hermes/profiles/memoria-*/`. Git is the canonical history layer; sync (Obsidian Sync, Syncthing, or manual git pull/push) is separate from history.

**See [01-architecture/on-disk-layout.md](01-architecture/on-disk-layout.md)** for the full tree, the vault/runtime relationship, the install flow, version-control rules (in / out of git), and the "sync ≠ history" discipline.

## Profile management

The seven profile directories under `.memoria/profiles/memoria-<name>/` are **hand-authored**. They are checked into the starter vault repo as authored, and `install.ps1` copies them verbatim into `~/.hermes/profiles/memoria-<name>/` (substituting `{{VAULT_PATH}}` placeholders in `mcp.json`). There is no build step; what you read in the vault is what the agent reads at runtime.

The trade-off is that shared content (audit-log behavior, common policy invariants, common MCP connections) lives in seven copies that must be kept in lockstep by hand. The linter's M1 install-drift detector (see the [Linter design summary](profiles/linter.md) and the runtime `M-detectors.md` alongside the Linter SOUL.md in the starter vault) catches one direction of drift (the deployed copy diverging from the vault source); inter-profile drift between the seven SOUL.md files relies on operator review during edits. See the [deferred compiler design](reference/profile-compilation.md) for the alternative that may become relevant if drift becomes painful at the seven-profile scale.

## How the layers interact

The recommended interaction pattern is:

1. A trigger (operator action, cron job, git hook, file watcher) creates a card on the board with a lane and an initial state. Routing is encoded in the trigger's rules — there is no profile that "decides" lane assignment.
2. **Specialist profile** (researcher, cartographer, writer, verifier, coder) claims a card from its lane and moves it to `active`. Socratic is invoked synchronously by the operator and doesn't appear in queue-based handoffs.
3. The worker executes the task, writes any provisional outputs (e.g., source notes, synthesis drafts) into the lane's declared write scope, and moves the card to `awaiting-review`.
4. The **operator** examines the work, then sets the card to `approved` or `rejected`. Some review decisions are partially automated — Verifier produces a `[!verification]` callout the operator reads — but the approval is always operator-driven.
5. If `approved`, the worker (or the next workflow trigger) advances the card to `done` and the output remains in its current location (or is moved to a canonical layer if promotion is part of the task).
6. If `rejected`, the operator chooses between two follow-ups: spawn a revision card on the same lane (carrying a `supersedes` link back to the original; original closes with `outcome: superseded`) or close the original entirely with `outcome: discarded`. See [03-board.md Post-rejection paths](03-board.md#post-rejection-paths).
7. **Linter** can act on any card structurally — usually before review — to flag schema, link, or orphan issues. It only ever produces reports, never silent fixes.

Cards never close on a worker's say-so. The card lives until the operator changes the review state.

## Core principles

- **The board is the shared state machine.** All long-lived work lives there.
- **Lanes are specialist execution paths.** Not separate boards.
- **Review is a state, not a comment.** It is queryable, ownable, and blocks dispatch.
- **Folders encode what a note is, not what it is about.** Topics belong in frontmatter and links.
- **Canonical synthesis is human-owned.** `30-synthesis/01-permanent/` is never auto-written.
- **Retries reuse the same card.** No duplicates; history is preserved in comments.
- **Handoff notes carry context.** The next worker should not need to re-read the conversation.
- **Every agent logs what it changed and why.** The policy MCP records every write decision to an audit log; the Linter rotates the log and produces the session summaries that make reversibility auditable.
- **Prefer extending the architecture over adopting peer systems.** A new tool that lives *inside* Memoria's existing surfaces — a plugin, a skill, a dashboard, a lane — composes with the policy MCP, the audit log, and the surface discipline. A new tool that lives *alongside* Memoria as a peer (its own UI, its own state, its own auth) creates boundary disputes, duplicates state, and usually ships a feature that bypasses the policy gate. Extensions compose; peers compete. When evaluating a new capability, look for the extension shape first.

## Memory tiers

Hermes memory operates at four scopes — **profile** (session-bound), **lane** (card-bound, travels via task packet), **project** (cross-lane, lives in `40-workbench/01-projects/<project>/`), and **episodic archive** (vault-wide, append-only, linter-owned). Confusing the scopes is the source of most "the agent forgot" and "the agent remembered something it shouldn't have" bugs.

**See [01-architecture/memory-tiers.md](01-architecture/memory-tiers.md)** for the full tier table, the rules that keep memory from bleeding across lanes, and why the tiering matters.

## Permission enforcement: the policy MCP

Profile permissions and lane scopes are not just documentation — they are enforced at the tool layer by a **policy MCP** that intercepts every vault write. Prompts are advisory; the policy MCP catches misrouted writes at the file-system level, before they happen.

The MCP guards eight distinct actions (`read`, `write`, `append`, `move`, `delete`, `mkdir`, `auto_fix`, `report`) and returns one of four decisions per request:

| Decision | Meaning |
| --- | --- |
| `allow` | Action proceeds; logged only if the lane requires it. |
| `allow_with_log` | Action proceeds; audit entry mandatory. |
| `deny` | Action blocked; the worker must escalate. |
| `dry_run` | Action logged and reported but not performed; surfaced for human approval. |

Two structural rules sit above the lane configuration:

- **Canonical zones are never auto-written.** Writes to `30-synthesis/01-permanent/`, `30-synthesis/03-moc/`, and `50-deliverables/` degrade to `dry_run` regardless of lane policy.
- **Linter auto-fix is class-gated** to `safe-and-unambiguous` or `authorized-targeted` (see [profiles/linter.md](profiles/linter.md#auto-fix-policy)).

Every allowed write produces an append-only audit entry in `00-meta/04-logs/audit.jsonl` with SHA-256 `before_hash` and `after_hash` for tamper detection and reversibility. The MCP — not the worker — computes the hashes; hash failures cause `deny`.

### Skill-conditional policy

Lane-overrides are the baseline policy a profile carries. A small set of skills tighten that baseline further — the currently-shipped example is `counter-outline` (Writer-loaded; narrows write scope to scratch-only). The mechanism, frontmatter contract, and composition semantics are in [reference/policy-mcp.md](reference/policy-mcp.md#skill-conditional-policy); the catalog of restrictive skills is in [02-profiles.md](02-profiles.md#skills-with-restrictive-policy).

**Full mechanism details** — decision protocol, request/response contracts, action vocabulary, audit log schema, SHA-256 implementation rules — are in [reference/policy-mcp.md](reference/policy-mcp.md). This summary covers what the architecture relies on; the reference doc is the canonical source.

## Control plane

The board defines *what state* a card is in. The policy MCP defines *where* a worker may write. The **control plane** is the daily-use surface that lets the human trigger discrete actions: three thin layers — Obsidian Command Palette → Local HTTP bridge → MCP servers — between the human and Hermes. None of them owns business logic except the MCP servers.

The HTTP bridge is fail-closed: binds to `127.0.0.1` by default, and refuses to start on a non-loopback interface unless `HERMES_BRIDGE_TOKEN` is set.

**See [01-architecture/control-plane.md](01-architecture/control-plane.md)** for the layer-by-layer responsibility table, fail-closed startup rules, and MCP server registration shape.

## Why Memoria refuses to autonomize synthesis

Karpathy's Autoresearch pattern works for ML experiments because three conditions hold: the metric is monotonic, changes are reversible, and experiments are independent. Knowledge work satisfies none of these. Memoria adopts the parts that *do* fit (overnight discovery loop, point-of-action similarity check) and refuses the parts that don't (autonomous keep/revert; scalar-metric-driven canonization).

**See [01-architecture/why-no-autonomous-synthesis.md](01-architecture/why-no-autonomous-synthesis.md)** for the three boundaries (cognition-bound vs compute-bound, no autonomous keep/revert, no scalar metric) and what they imply for scheduled operations, agent scope, and cost discipline.

## Pattern provenance

Memoria draws on a broad survey of contemporary AI-research systems (LitSearch, ResearchArena, AI-Scientist, LatteReview, OmegaWiki, Idea2Story, Karpathy Autoresearch, and others). The headline borrowed patterns are summarized in [00-vision.md](00-vision.md#contemporary-ai-research-systems); the autonomy boundary that rejects several wholesale is in [01-architecture/why-no-autonomous-synthesis.md](01-architecture/why-no-autonomous-synthesis.md).

**Full borrow / adapt / ignore table** — every pattern, every source repo, every rationale — lives in [rationale/pattern-provenance.md](rationale/pattern-provenance.md). The net design shift is from agent-assisted to **bounded, stage-gated knowledge production**: the agent becomes better at bookkeeping, retrieval, and drafting; the human remains the gatekeeper for meaning, promotion, and final structure.

## Capability stack

The minimum capability stack to operate Memoria is eight components: Hermes (seven profiles), the Hermes Kanban, Obsidian, Zotero + Better BibTeX, the external enrichment APIs (OpenAlex, Semantic Scholar, PubMed, Crossref, Unpaywall, ORCID, ROR), git, the Obsidian REST API or ACP for editor integration, and Pandoc for export.

On top of that base, **pre-built skills** (K-Dense `paper-lookup` / `pyzotero` / `citation-management`; Obsidian `obsidian-paper-note`; Hermes built-in `llm-wiki`) cover most enrichment and ingest work; the agent should use them rather than writing API clients from scratch. **Model routing** — Claude for synthesis, cheap models (via OpenRouter or similar) for bulk/mechanical tasks — keeps the overnight loop's `$1–3/day` budget achievable.

**See [01-architecture/capability-stack.md](01-architecture/capability-stack.md)** for the full skill catalog (K-Dense, Obsidian, Hermes built-in, the generic REST bridge escape hatch), the model-routing table, and the plugins/apps/external-tools list.

## Operating model

The architecture implies a specific operating model:

1. **Daily.** Capture new sources in Zotero. Let Hermes ingest them. Skim the inbox for synthesis drafts and discovery candidates.
2. **Weekly.** Run the [weekly dashboard](06-surfaces.md). Clear unreviewed synthesis. Promote evergreen claim notes. Triage partial source notes.
3. **Per-project.** Draft from `30-synthesis/01-permanent/`, arrange in `40-workbench/04-canvas/`, write in `40-workbench/02-drafts/`, export to `50-deliverables/` via Pandoc.
4. **Continuously.** The linter runs scheduled checks; cron and git-hook triggers advance cards through their lifecycle; the operator clears the approval queue.

This model is bounded, stage-gated, and human-in-the-loop by default. Autonomy is added at the edges (scheduled discovery, automatic enrichment) without weakening the review gate.

## Next

- For per-profile detail: [02-profiles.md](02-profiles.md).
- For board mechanics: [03-board.md](03-board.md).
- For the daily/weekly workflows: [04-workflows.md](04-workflows.md).
