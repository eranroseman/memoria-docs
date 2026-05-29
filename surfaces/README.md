---
mode: explanation
audience: operator
topic: surfaces
---

# Human surfaces

Memoria's three layers (board, workers, vault) are the architecture; **surfaces** are how state is rendered where the human reads it. A surface is one of four presentation forms *inside Obsidian* — each with its own latency, persistence, and purpose, and each owning one mode of interaction. This document codifies those four. (Reaching Memoria at all is a separate question — that's *channels*: Obsidian, command palette, CLI, Telegram, API. The four surfaces all live within the Obsidian channel; see [architecture/channels-overview.md](../architecture/channels-overview.md) and the [glossary](../glossary.md#surfaces-and-channels) for how the two terms relate.) Two more files round out this folder: the [command palette](command-palette.md), the keyboard channel through which state changes are issued, and the [design system](design-system.md) that styles every surface.

## The four surface types

| Type | What it is | Open / close cadence | Examples |
| --- | --- | --- | --- |
| **Persistent** | Dashboards — Dataview queries rendered as notes. Opened deliberately; read; closed. | Seconds to minutes per visit; multiple times per day | `Daily Health`, `weekly-review`, `audit-log` |
| **Modal** | Saved Obsidian workspaces bound to a hotkey. Switched into for a task; switched out of when the task ends. | Whole sessions (reading, drafting, triage) | Human (`Cmd-1`), Reading & Processing (`Cmd-2`), Drafting (`Cmd-3`) |
| **Inline** | Callout blocks inside individual notes. Appear in place, where the relevant context lives. | Visible whenever the note is open | `[!brief]`, `[!suggestions]`, `[!verification]` |
| **Ambient** | Always-visible but minimal indicators. Surface status without demanding attention. | Continuously visible while Obsidian is open | A glanceable status line (a Dataview widget in a pinned note — not the OS status bar), lint findings, badge counts |

These are progressively more **embedded** in the workflow: persistent surfaces interrupt to be read; ambient surfaces inform without interrupting. The right type for an agent output depends on what kind of decision the human faces:

- **A queue of notes to act on** → persistent (dashboards)
- **Which working mode the human is in** → modal (workspaces)
- **One note in front of them** → inline (callouts)
- **Whether anything needs attention at all** → ambient (status line)

Each surface type carries design rules specific to it, detailed in its own sub-file (see [What each sub-file covers](#what-each-sub-file-covers) below).

## Cross-surface design rules

Two rules apply to every surface, not just to one type.

- **The surface is read; the work is elsewhere.** No surface contains fix logic. Surfaces reveal issues; action happens in notes (for content) or through the command palette (for state changes). This is what keeps state transitions deliberate — a clicked dashboard row, a typed command, an accepted callout suggestion all converge on the policy MCP, never bypass it.
- **Invisible during normal use, legible on failure.** A healthy day is a 30-second glance at [Daily Health](../dashboards/README.md) that shows nothing red and gets closed. Surfaces earn their keep only when something goes wrong — and at that moment they must make the breakage immediately legible. A surface visited for reassurance is friction; a surface visited for diagnosis is the architecture working as designed.

## What each sub-file covers

Each surface has its own sub-file with the rules, catalogs, and layouts specific to it:

- **[Persistent — dashboards](persistent.md):** the dashboard catalog by frequency (eleven in total: Daily / Reading-session / Weekly / Per-board-op / Forensic / Planning / Scale-dependent), the design rules (one decision per query, filter boring cases, sort oldest-first), the Dataview performance discipline, and the graceful-degradation pattern for queries whose dependency is missing.
- **[Modal — workspaces](modal.md):** the per-workspace layout table (Human / Reading & Processing / Drafting on `Cmd-1/2/3`), the prerequisite (the Workspaces core plugin must be enabled), and the design rules (one mode per workspace, three is the working set, workspaces travel with the vault).
- **[Inline — callouts](inline.md):** the per-callout producer table (`[!brief]`, `[!suggestions]`, `[!verification]` from Mapper, Librarian, and Verifier), example shape, design rules (producer-owned + human-curated, collapsed vs expanded defaults, never overwrite human edits), and the per-callout deterministic/LLM split with weights.
- **[Ambient — status line](ambient.md):** the composite line shape, implementation notes (Dataview, not a custom plugin), and the design rules (show state not decisions, glance-readable in under a second, no interruptive transitions, two producers is the working set).
- **[Command palette — the keyboard channel](command-palette.md):** the standard command catalog, the `Memoria:` naming convention, and the session-persistent-vs-transient invocation discipline.

The [design system](design-system.md) sub-file is the exception: it isn't a surface, but the visual-style source every surface renders against.

<!-- memoria-nav -->

---

[← Previous: Refactor](../workflows/maintenance/refactor.md)

[Next: Persistent surfaces: dashboards →](persistent.md)
