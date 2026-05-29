---
mode: explanation
audience: operator
topic: surfaces
---

# Human surfaces

Memoria's three layers (board, workers, vault) are the architecture; **surfaces** are how the human sees them and acts on them. A surface is any place in the human's field of view where Memoria state appears or Memoria action is taken. This document codifies the four kinds, each with its own latency, persistence, and purpose — and each owning one mode of interaction.

## The four surface types

| Type | What it is | Open / close cadence | Examples |
| --- | --- | --- | --- |
| **Persistent** | Dashboards — Dataview queries rendered as notes. Opened deliberately; read; closed. | Seconds to minutes per visit; multiple times per day | `index`, `weekly-dashboard`, `audit-log` |
| **Modal** | Saved Obsidian workspaces bound to a hotkey. Switched into for a task; switched out of when the task ends. | Whole sessions (reading, drafting, triage) | Human (`Cmd-1`), Reading & Processing (`Cmd-2`), Drafting (`Cmd-3`) |
| **Inline** | Callout blocks inside individual notes. Appear in place, where the relevant context lives. | Visible whenever the note is open | `[!brief]`, `[!suggestions]`, `[!verification]` |
| **Ambient** | Always-visible but minimal indicators. Surface status without demanding attention. | Continuously visible while Obsidian is open | Status-bar lint findings, badge counts |

These are progressively more **embedded** in the workflow: persistent surfaces interrupt to be read; ambient surfaces inform without interrupting. The right type for an agent output depends on what kind of decision the human faces:

- **A queue of notes to act on** → persistent (dashboards)
- **Which working mode the human is in** → modal (workspaces)
- **One note in front of them** → inline (callouts)
- **Whether anything needs attention at all** → ambient (status bar)

Each surface type is detailed in its own sub-file in this folder ([persistent.md](persistent.md), [modal.md](modal.md), [inline.md](inline.md), [ambient.md](ambient.md)), with design rules specific to that type.

## Cross-surface design rules

Two rules apply to every surface, not just to one type.

- **The surface is read; the work is elsewhere.** No surface contains fix logic. Surfaces surface issues; action happens in notes (for content) or commands (for state changes). This is what keeps state transitions deliberate — a clicked dashboard row, a typed command, an accepted callout suggestion all converge on the policy MCP, never bypass it.
- **Invisible during normal use, legible on failure.** A healthy day is a 30-second glance at [`index`](../dashboards/README.md) that shows nothing red and gets closed. Surfaces earn their keep only when something goes wrong — and at that moment they must make the breakage immediately legible. A surface visited for reassurance is friction; a surface visited for diagnosis is the architecture working as designed.

Type-specific rules live in each sub-file.

## Persistent surfaces: dashboards

Dataview-rendered notes that surface what's in flight on the board, what's overdue in the vault, and what needs human judgment. Eleven dashboards in total, organized by use frequency (Daily / Reading-session / Weekly / Per-board-op / Forensic / Planning / Scale-dependent).

**See [surfaces/persistent.md](persistent.md)** for the dashboard catalog by frequency, the design rules (one decision per query, filter boring cases, sort oldest-first), the Dataview performance discipline, and the graceful-degradation pattern for queries whose dependency is missing.

## Modal surfaces: workspaces

Three Obsidian workspaces bound to keyboard shortcuts (`Cmd-1` / `Cmd-2` / `Cmd-3`), one per cognitive mode (Human / Reading & Processing / Drafting). Mode shifts become muscle memory; cognitive modes stop mixing on the same screen.

**See [surfaces/modal.md](modal.md)** for the per-workspace layout table, the prerequisite (the Workspaces core plugin must be enabled), and the design rules (one mode per workspace, three is the working set, workspaces travel with the vault).

## Inline surfaces: callouts in notes

Three callout types — `[!brief]`, `[!suggestions]`, `[!verification]` — produced by Mapper, Librarian, and Verifier respectively. All three use the hybrid pattern: a deterministic step narrows the candidate set, an LLM step composes the prose. The audit trail is the deterministic step's output; the LLM's prose is the surface presentation.

**See [surfaces/inline.md](inline.md)** for the per-callout producer table, example shape, design rules (producer-owned + human-curated, collapsed vs expanded defaults, never overwrite human edits), and the per-callout deterministic/LLM split with weights.

## Ambient surfaces: status bar

The Obsidian status bar carries two ambient producers — lightweight Linter findings and Kanban queue counts — sharing one line. Glance-readable in under one second; click to drill down into the persistent surface.

**See [surfaces/ambient.md](ambient.md)** for the composite line shape, implementation notes (Dataview, not a custom plugin), and the design rules (show state not decisions, glance-readable in under a second, no interruptive transitions, two producers is the working set).

<!-- memoria-nav -->

---

[← Previous: Refactor](../workflows/maintenance/refactor.md)

[Next: Persistent surfaces: dashboards →](persistent.md)
