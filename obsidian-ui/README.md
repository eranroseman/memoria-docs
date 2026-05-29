---
mode: explanation
audience: operator
topic: obsidian-ui
---

# The Obsidian UI

Obsidian is Memoria's **primary UI** — the focused desktop surface where nearly all daily work happens. (Reaching Memoria from *outside* Obsidian is a separate question — the **CLI** and **Telegram** channels, plus the non-human **API**; see [architecture/channels-overview.md](../architecture/channels-overview.md).) This document covers what lives *inside* Obsidian: the UI components, what each is for, and the render discipline that decides which component an agent output belongs in.

There is no separate abstract "surface taxonomy" stacked on top of these. A component *is* its concrete thing — a dashboard, a callout — not a category. Earlier docs gave each one two names ("Persistent (dashboards)", "Modal (workspaces)", "Inline (callouts)", "Ambient (status line)"); that dual naming is retired. This file uses the concrete name and treats the *behavior* — how often a component opens, whether it interrupts — as a property, not a label.

## The Obsidian UI components

| Component | What it is | Cadence | Detail |
| --- | --- | --- | --- |
| **Dashboards** | Dataview queries rendered as notes. Opened deliberately, read, closed. | Seconds to minutes per visit; multiple times per day | [persistent.md](persistent.md) |
| **Workspaces** | Saved Obsidian layouts bound to a hotkey (`Cmd-1/2/3`); one per cognitive mode. | Whole sessions (reading, drafting, triage) | [modal.md](modal.md) |
| **Callouts** | Agent output rendered in-place inside a note, where the context lives. | Visible whenever the note is open | [inline.md](inline.md) |
| **Status line** | A glanceable Dataview widget in a pinned note — **not** the OS status bar. | Continuously visible while Obsidian is open | [ambient.md](ambient.md) |
| **Command palette** | The keyboard component — every Memoria action bound to `Cmd-P → Memoria:`. | Instant; many times per day | [command-palette.md](command-palette.md) |
| **Agent Client pane** | The ACP chat pane — the UI for conversing with a Hermes profile on the active note. | Per conversation; persistent for Socratic, transient for the others | [agent-client.md](../obsidian-plugins/required/agent-client.md) |
| **Other plugin UI** | UI contributed by the other community plugins Memoria uses (PDF++, Kanban, citation, …). | Varies | [obsidian-plugins/](../obsidian-plugins/README.md) |

## Which component for which output

The first four components form a gradient by how much they **interrupt**: a dashboard interrupts to be read; the status line informs without ever demanding attention. Callouts and workspaces sit between. Pick the component by the kind of decision the human faces:

- **A queue of notes to act on** → a dashboard.
- **Which working mode the human is in** → a workspace.
- **One note in front of them** → a callout.
- **Whether anything needs attention at all** → the status line.
- **A state change to issue** → the command palette.
- **A conversation about the active note** → the Agent Client pane.

That decision list *is* the design discipline the old persistent / modal / inline / ambient names used to carry — kept as a rule of thumb instead of a vocabulary the reader has to memorize.

## Cross-component rules

Two rules apply to every component, not just one type.

- **The UI is read; the work is elsewhere.** No component contains fix logic. Components reveal issues; action happens in notes (for content) or through the command palette (for state changes). This is what keeps state transitions deliberate — a clicked dashboard row, a typed command, an accepted callout suggestion all converge on the policy MCP, never bypass it.
- **Invisible during normal use, legible on failure.** A healthy day is a 30-second glance at [Daily Health](../dashboards/daily-health.md) that shows nothing red and gets closed. The UI earns its keep only when something goes wrong — and at that moment it must make the breakage immediately legible. A component visited for reassurance is friction; one visited for diagnosis is the architecture working as designed.

## What each detail file covers

Each component has its own file with the rules, catalogs, and layouts specific to it:

- **[Dashboards — persistent.md](persistent.md):** the dashboard catalog by frequency (eleven in total: Daily / Reading-session / Weekly / Per-board-op / Forensic / Planning / Scale-dependent), the design rules (one decision per query, filter boring cases, sort oldest-first), the Dataview performance discipline, and the graceful-degradation pattern for queries whose dependency is missing.
- **[Workspaces — modal.md](modal.md):** the per-workspace layout table (Human / Reading & Processing / Drafting on `Cmd-1/2/3`), the prerequisite (the Workspaces core plugin must be enabled), and the design rules (one mode per workspace, three is the working set, workspaces travel with the vault).
- **[Callouts — inline.md](inline.md):** the per-callout producer table (`[!brief]`, `[!suggestions]`, `[!verification]` from Mapper, Librarian, and Verifier), example shape, design rules (producer-owned + human-curated, collapsed vs expanded defaults, never overwrite human edits), and the per-callout deterministic/LLM split with weights.
- **[Status line — ambient.md](ambient.md):** the composite line shape, implementation notes (Dataview, not a custom plugin), and the design rules (show state not decisions, glance-readable in under a second, no interruptive transitions, two producers is the working set).
- **[Command palette — command-palette.md](command-palette.md):** the standard command catalog, the `Memoria:` naming convention, and the session-persistent-vs-transient invocation discipline.
- **[Agent Client — agent-client.md](../obsidian-plugins/required/agent-client.md):** the ACP pane configuration, the four profiles in the picker, mode-switching hotkeys, and the persistent-vs-transient session distinction.

The [design system](design-system.md) file is the exception: it isn't a component, but the visual-style source every component renders against.
