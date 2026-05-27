# Command palette — Memoria's keyboard surface

The single biggest UX move in Memoria is binding every common operation to an Obsidian command, then driving the system from `Cmd-P` (`Ctrl-P` on Windows/Linux) instead of clicking through menus. Within a few weeks of consistent use, muscle memory replaces every UI click.

This document is the canonical catalog of Memoria commands, what each invokes, and where they're implemented. It's the reference the operator consults when configuring [QuickAdd](obsidian-plugins.md#quickadd-and-templater) (which registers the commands) and [Commander](obsidian-plugins.md#cmdr-commander) (which puts the top five on physical buttons).

## Naming convention

Every command begins with the prefix **`Memoria:`** so the operator can type `Cmd-P → "M"` and filter the palette to Memoria-only commands. After three months of use, this becomes the primary input mode for the system — the operator types Cmd-P, then 1–3 letters of the command name, then enter.

Two-tier discipline:

- The **canonical commands** below are the operational surface. They cover capture, processing, interactive retrieval, projects, maintenance, and lens-based reading.
- Anything beyond these is an *operator addition*, not part of the canonical Memoria UX. Operators add their own custom commands freely — but the canonical set is what every Memoria install ships with, what cross-machine portability assumes, and what the [Commander](obsidian-plugins.md#cmdr-commander) recommendation set is drawn from.

## The canonical commands

### Capture (3 commands)

| Command | What it does | Implementation |
| --- | --- | --- |
| `Memoria: capture fleeting` | Instant note to `10-inbox/01-fleeting/` with timestamp. No friction; the operator presses the hotkey, types one sentence, presses enter, and the thought is captured. | QuickAdd → Templater (fleeting-note template) |
| `Memoria: capture source from URL` | Prompts for a URL, creates `intake:source` card in the Research lane's queue. Researcher picks it up within 60 seconds and starts the ingest pipeline. | QuickAdd → POST to Hermes API (`hermes kanban add`) |
| `Memoria: capture from Zotero selection` | Reads the current Zotero selection (via the Zotero local API), creates an `intake:source` card with the citekey pre-populated. Useful when the operator is mid-reading in Zotero and wants the paper in the vault. | QuickAdd → reads Zotero API → POST to Hermes API |

### Processing (3 commands)

| Command | What it does | Implementation |
| --- | --- | --- |
| `Memoria: invoke Socratic on current note` | Opens the [ACP pane](obsidian-plugins.md#agent-client) with the Socratic profile loaded, passing the active note path as context. The conversation never modifies the note (Socratic's lane policy is `policy.allow.write: []`). **Persistent pane** — the conversation has its own lifecycle and can be resumed via `savedSessions[]`. | QuickAdd → invokes `agent-client` plugin's "switch agent" command with `defaultAgentId: memoria-socratic` and the active-note path |
| `Memoria: process this fleeting note` | Moves the current fleeting note to the process queue: creates a `process` card targeting the operator, and opens the Socratic ACP pane. Combines the previous two commands for the fleeting → claim transition. | QuickAdd composing two prior commands |
| `Memoria: write permanent note` | Opens a new note with the claim-note template, prompts for the source citekey, and pre-populates the frontmatter. Lands in `30-synthesis/01-permanent/` with `maturity: seedling`. | QuickAdd → Templater (claim-note template) |

### Interactive retrieval (3 commands — transient ACP)

These commands invoke a profile via a *transient* ACP session: the agent-client plugin opens a fresh chat session, the agent returns results, the session closes. No persistent pane, no `savedSessions[]` entry. They're for "ask one specific question and get an answer" — distinct from the persistent Socratic processing chat above, and distinct from the substantive card-based work in the Project section below (which produces durable artifacts via the Kanban).

One transient command per ACP-suitable profile in the picker (Cartographer / Writer / Verifier), each pointed at the kind of question the profile is shaped to answer.

| Command | What it does | Implementation |
| --- | --- | --- |
| `Memoria: find related notes` | Invokes Cartographer in transient ACP mode with the current note's content as the query. Returns a ranked list of related notes (top 5–10 by embedding similarity + shared citations + topic overlap) directly in the ACP chat. No file artifact written; no card created. The operator copies citekeys they want or dismisses the session. | QuickAdd → invokes `agent-client` plugin's "switch agent" command with `defaultAgentId: memoria-cartographer` and the active-note path; closes session after first response |
| `Memoria: counter-outline this section` | Invokes Writer with the `counter-outline` skill loaded in transient ACP mode. Returns 2–3 competing outlines for the current section directly in chat — no file artifact, no project-scratch write. Useful for quick brainstorming before committing to a framing direction. For the substantive *committed* counter-outline (which writes to `40-workbench/01-projects/<project>/framing/`), use `Memoria: frame this section` in the Project section below. | QuickAdd → invokes `agent-client` plugin's "switch agent" command with `defaultAgentId: memoria-writer` and the `counter-outline` skill flag; closes session after first response |
| `Memoria: similarity-check this claim` | Invokes Verifier in transient ACP mode with the current selection (or active note's body if no selection) as the query. Returns the top 3 most-similar existing claim-notes by cosine similarity over the embedding index, with the similarity score per result. No card created; no `near-duplicate-candidate` flag written. Useful before filing a new claim note to check for duplicates; if the operator decides to file anyway, the card-time `similarity-check` runs separately and produces the audit-trail entry. See [profiles/verifier.md `similarity-check`](../profiles/verifier.md#core-commands) for the substantive card-based version. | QuickAdd → invokes `agent-client` plugin's "switch agent" command with `defaultAgentId: memoria-verifier` and the active selection or note as input; closes session after first response |

The architectural distinction: **persistent ACP** (Socratic, default) is for long conversations during processing; **transient ACP** (Cartographer / Writer / Verifier via these commands) is for quick queries with no expected artifact. See [`obsidian-plugins.md#agent-client`](obsidian-plugins.md#agent-client) for the per-profile rationale.

**The transient commands don't replace their card-based counterparts.** Each profile has both surfaces:

| Profile | Transient ACP (this section) | Substantive card-based |
| --- | --- | --- |
| Cartographer | `Memoria: find related notes` — quick query, no artifact | `Memoria: scope this project` — produces `corpus-map.md`, Kanban-tracked |
| Writer | `Memoria: counter-outline this section` — quick brainstorm, no artifact | `Memoria: frame this section` — produces outlines in `40-workbench/01-projects/<project>/framing/`, Kanban-tracked |
| Verifier | `Memoria: similarity-check this claim` — quick lookup, no audit entry | (filing-time `similarity-check` fires automatically when a new claim-note is filed; produces the audit-trail entry and the `near-duplicate-candidate` flag if applicable) |

The distinction is *output discipline*: transient commands produce chat output the operator can copy or dismiss; substantive commands produce file artifacts tracked through the Kanban with audit trail.

### Project (4 commands)

| Command | What it does | Implementation |
| --- | --- | --- |
| `Memoria: new project` | Creates `40-workbench/01-projects/<name>/` with `brief.md` from the project template, then opens a `scope` card on the Cartography lane queue. The operator fills the brief; Cartographer picks up the card and produces the corpus map. | QuickAdd → Templater (project-brief template) → POST to Hermes API |
| `Memoria: scope this project` | Manually triggers Cartographer's `scope-project` skill on the current project folder. Produces `corpus-map.md` as a *file artifact* tracked as a card on the Kanban. For quick "what does my corpus say about X" queries without file commitment, use `Memoria: find related notes` (transient ACP) in the Interactive retrieval section above. | QuickAdd → POST to Hermes API (lane: cartography, task: scope-project, payload: project path) |
| `Memoria: frame this section` | Invokes Writer with the `counter-outline` skill loaded on the current project. Produces 2–3 competing outlines as *file artifacts* in `40-workbench/01-projects/<project>/framing/`, tracked as a card on the Kanban. For quick brainstorming without file commitment, use `Memoria: counter-outline this section` (transient ACP) in the Interactive retrieval section above. | QuickAdd → POST to Hermes API (lane: writer, skill: counter-outline) |
| `Memoria: verify this draft` | Manually triggers Verifier's `cite-check` on the current draft. Normally Verifier fires automatically on git commit; this command is for the case where the operator wants verification on uncommitted work. | QuickAdd → POST to Hermes API (lane: verify, task: cite-check, payload: draft path) |

### Maintenance (3 commands)

| Command | What it does | Implementation |
| --- | --- | --- |
| `Memoria: approve all link suggestions` | Bulk-approves every `awaiting-review` link-suggestion card. The operator should glance through the [`weekly-dashboard`](../dashboards/weekly-dashboard.md) first; this is the "if they all look right" shortcut. | QuickAdd → POST to Hermes API (bulk approve filtered cards) |
| `Memoria: lint this note` | Runs the Linter (dry-run) on the current note, displays the report inline. The Linter is otherwise scheduled (see [`07-roadmap.md` Standard cron tasks](../07-roadmap/standard-cron-tasks.md)); this is the manual on-demand check. | QuickAdd → POST to Hermes API (lane: linter, task: lint, payload: note path) |
| `Memoria: show lane status` | Opens [`index.md`](../dashboards/index.md) in the right sidebar. Useful when the operator wants to glance at lane health mid-task without leaving their current note. | QuickAdd → Workspace pane manipulation |

### Lens-based reading (parameterized; 4 examples)

Each lens is its own command. The commands invoke Socratic with the named `lens-reading` parameter. Adding a new lens means adding one more command — the parameter is the lens skill's slug.

| Command | Lens |
| --- | --- |
| `Memoria: read through Mamykina lens` | `mamykina-sensemaking` |
| `Memoria: read through Veinot equity lens` | `veinot-informational-justice` |
| `Memoria: read through Design Justice lens` | `design-justice-costanza-chock` |
| `Memoria: read through JITAI lens` | `jitai-receptivity-timing` |

Implementation pattern (same for all): QuickAdd → invokes `agent-client` plugin's "switch agent" command with `defaultAgentId: memoria-socratic` and an additional skill argument naming the lens.

**Why each lens is its own command rather than one parameterized command.** Memoria's discipline says lens-based reading is *deliberately constrained* — the operator picks a specific lens at the start of the session and stays in it, rather than switching mid-conversation. One command per lens makes the choice explicit before the conversation begins. A single parameterized command would invite mid-conversation lens-switching, which muddies whose questions are being asked.

## Setting up the bindings

The bindings are operator-side configuration, not part of Memoria's shipped vault. The convention:

1. **Install [QuickAdd](obsidian-plugins.md#quickadd-and-templater)**.
2. **For each command above, create a QuickAdd entry** with the same name (preserving the `Memoria:` prefix).
3. **Configure the underlying mechanism** per the Implementation column — Templater templates for capture commands, Hermes API calls for Kanban interactions, agent-client commands for ACP invocations.
4. **Optionally pin the top 5 to Commander** for physical-button access. The recommended Commander set (see [`obsidian-plugins.md`](obsidian-plugins.md#cmdr-commander)):
   - `Memoria: capture fleeting`
   - `Memoria: invoke Socratic on current note`
   - `Memoria: new project`
   - `Memoria: lint this note`
   - `Memoria: approve all link suggestions`

The `Cmd-P → "M"` filter convention works from the moment the first three commands exist. The operator builds the muscle memory as they go.

## What's deliberately NOT in this catalog

- **Card state management.** Approving, denying, rejecting individual cards happens through inline callout buttons (see [`06-surfaces.md`](../06-surfaces/inline.md)) or through the Kanban directly. There's no `Memoria: approve this card` command because card state changes need card context the palette doesn't provide.
- **Profile administration.** Reloading a profile, editing a lane-override, editing a profile source in `.memoria/profiles/memoria-<name>/` and redeploying with `install.ps1` — these are CLI operations, not palette operations. The palette is for *daily ops*; profile config is a *deployment concern*.
- **Search and retrieval (search-engine-style).** The vault's native search, Dataview queries, and `qmd` search are richer than any palette command could be. Memoria doesn't try to substitute its own search. The Interactive retrieval commands above are a different category — *conversational* retrieval where the operator asks a profile a question and gets a curated response, not full-text / vector / property search that returns ranked results from an index.
- **Composing custom workflows.** Operators add their own commands for their own workflows; this catalog is the *canonical Memoria set*, not an exhaustive operator surface.

## Related

- [`obsidian-plugins.md`](obsidian-plugins.md#quickadd-and-templater) — QuickAdd and Templater plugin details.
- [`obsidian-plugins.md`](obsidian-plugins.md#cmdr-commander) — Commander plugin for putting top commands on buttons.
- [`06-surfaces.md`](../06-surfaces.md) — workspaces, dashboards, callouts, and the command palette as one of the five Memoria surfaces.
- [`04-workflows.md`](../04-workflows.md) — workflows the commands trigger (workflow #14 Process, #15 Scope, #16 Frame, #17 Verify).
