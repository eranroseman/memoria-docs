# agent-client

Implements ACP — the Agent Client Protocol — inside Obsidian. ACP is what makes [01-architecture.md's surface](../../01-architecture.md#surfaces-each-one-owns-one-mode) "Obsidian dashboards + ACP panes" actually exist. The plugin runs configured agents as subprocesses (Hermes, Claude Code, Codex, Gemini CLI, and custom commands like Kilo Code) and exchanges messages with them over stdio, attaching the conversation to a side pane keyed off the currently active note.

**Example with placeholder paths:** shipped at `.obsidian/plugins/agent-client/data.json.example` in the starter vault. The example uses `{{HOME}}` placeholders the operator must replace with absolute paths before saving as the working `.obsidian/plugins/agent-client/data.json`. After the first launch the plugin will append a `savedSessions[]` array to the file — that's normal operator-generated state.

Load-bearing settings:

- **`customAgents` — configure four Memoria profiles, not one.** The picker should contain `memoria-socratic` (default), `memoria-cartographer`, `memoria-writer`, and `memoria-verifier`. The plugin's `customAgents` array supports multiple entries; each is one profile available in the picker. See per-profile rationale below.
- `defaultAgentId: "memoria-socratic"` — Memoria's canonical ACP default is Socratic. The operator falls into it without choosing; the other three are switchable via the picker for transient queries.
- `autoAllowPermissions: false` — **never set to `true`.** When `false`, every tool call the ACP agent makes prompts the operator for approval before executing. Setting this to `true` bypasses operator approval on every ACP write, which is exactly the failure mode the policy MCP exists to prevent at the Hermes side. The two layers (ACP approval + policy MCP enforcement) compose; turning either off breaks the composition.
- `autoMentionActiveNote: true` — automatically passes the active note's path and frontmatter to the agent as context. This is what makes "invoke Socratic on the current note" work from one keystroke.
- `chatViewLocation: "right-tab"` — places the ACP pane on the right, matching the [Reading & Processing workspace](../../06-surfaces/modal.md) assumption.
- `windowsWslMode: true` — required when the agent commands live inside WSL (the typical Memoria deployment on Windows). The plugin translates Windows paths to `/mnt/c/...` style paths for the subprocess.

## Which Memoria profiles to configure in the picker

The agent-client plugin is a *profile switcher*, not a single-profile binding. Each entry in `customAgents` is one profile available in the picker; `defaultAgentId` is the one the operator falls into by default. Memoria's four ACP-suitable profiles:

| Profile | ID | Pattern | Why |
| --- | --- | --- | --- |
| **Socratic** | `memoria-socratic` | **Persistent pane (default)** | Long conversations during processing. Architecturally write-denied (`policy.allow.write: []`); safe on any device. The canonical ACP use case — open the pane in the Reading & Processing workspace, talk through a literature note via questioning. |
| **Cartographer** | `memoria-cartographer` | **Transient session** | Quick corpus-retrieval queries via command palette ("find related notes," "what does my corpus say about X"). Session opens, returns results, closes. Not for persistent chat. |
| **Writer** | `memoria-writer` | **Transient session** | Quick drafting chat via command palette ("counter-outline this section," "rephrase this paragraph"). Useful with discipline — interactive `chat` mode only, never auto-`run draft`. Session closes after response. |
| **Verifier** | `memoria-verifier` | **Transient session** | Pre-filing similarity check via command palette ("show top-3 similar notes to this claim"). Returns ranked list, session closes. |

Skip from the picker:

- **Researcher** — network-active; every `discover`-style chat costs external API calls. Better dispatched via cards (queued, audited, retryable). Add only if you specifically want interactive discovery and accept the cost characteristics.
- **Coder** — delegates substantive coding to an external coding agent (Claude Code, Codex, Aider). The Hermes-side Coder profile doesn't fit ACP chat.
- **Linter** — background-only by design ([profiles/linter.md](../../profiles/linter.md)). No interactive use case.

## Persistent vs transient ACP sessions

The picker contains four agents but they're used in two architecturally distinct ways:

- **Persistent ACP pane.** Socratic. The operator opens the ACP pane in the Reading & Processing workspace and has a long conversation while working in adjacent panes. The session has its own lifecycle, persists in `savedSessions[]`, can be resumed later. This is the canonical setup for [workflow #14 Process](../../04-workflows.md).
- **Transient ACP session.** Cartographer, Writer, Verifier. The operator invokes the profile via the command palette for one specific question. The agent-client plugin opens a fresh session, the agent responds, the session closes. No persistent pane to manage; no savedSessions entry to accumulate. See [command-palette.md](../../reference/command-palette.md) for the specific commands.

The distinction matters for `savedSessions[]` hygiene: persistent Socratic sessions accumulate and should be pruned periodically; transient sessions auto-close and don't build up.

## Per-device install discipline

Under the [per-device install discipline](../../07-roadmap/timeline.md#per-device-install-sets), secondary devices may have only some profiles installed. The `customAgents` array should match what's locally installed:

- **Primary device** (all seven profiles installed): all four ACP-suitable entries above.
- **Secondary, reader role** (typically only Socratic installed): just the `memoria-socratic` entry. Configuring Cartographer / Writer / Verifier would let the picker show them, but invoking would fail with "profile not found" on that device.
- **Secondary, developer role** (all seven installed against a test vault): all four. The dev's Hermes points at a test vault via `HERMES_HOME`, so ACP conversations don't touch production.

Operators editing the example for their device should *delete* entries for profiles not installed locally, rather than leaving them in the picker as broken options.

## Configuring the laptop for non-Socratic ACP

The default secondary-device configuration (Socratic only) covers most operator needs — persistent processing chat and lens-based reading. But three scenarios call for *persistent* chat with non-Socratic profiles: Cartographer for deep corpus exploration (multi-turn drilling into clusters, density, methodological breadth), Writer for sustained drafting dialog (paragraph diagnostics, voice calibration, transition-finding), and Verifier for systematic auditing (pre-submission claim-by-claim review). These are *override* patterns — the [picker table above](#which-memoria-profiles-to-configure-in-the-picker) names transient session as the default for these three profiles, and operators should treat persistent chat as a deliberate exception, invoked when one-shot retrieval isn't enough.

Try the [transient command-palette commands](../../reference/command-palette.md#interactive-retrieval-3-commands--transient-acp) first — they cover the one-question, one-answer cases that are most laptop-appropriate. If those don't suffice and you genuinely need sustained dialog, choose the path that matches your deployment.

### Path 1 — Local install with discipline (Option A+ default)

Under [Option A+](../../07-roadmap/deployment-options.md) (desktop + laptop, no VPS), the desktop sleeps; SSH-spawn into it is unreliable. The right path is to install the additional profile(s) locally on the laptop with the [discipline obligations from Phase 3](../../07-roadmap/timeline.md#per-device-install-sets):

1. Clone the same starter vault on the laptop and run `./install.ps1`. Under direct profile management, profile parity follows from cloning the same vault — same `.memoria/profiles/memoria-<name>/` source on every device produces the same deployed copy at `~/.hermes/profiles/memoria-<name>/`.
2. The installer registers each profile with `--alias`, so `memoria-<name>` works as a command shortcut on the laptop the same way it does on the desktop.
3. Verify `cron_mode: deny` in the profile's `config.yaml` (Memoria's default — should not need changes).
4. Add to the agent-client `customAgents` array with a `displayName` that signals the discipline:

```json
{
  "id": "memoria-writer",
  "displayName": "Writer (drafting chat) — laptop, chat-only",
  "command": "{{HOME}}/.local/bin/hermes",
  "args": ["-p", "memoria-writer", "acp"],
  "env": []
}
```

The `— laptop, chat-only` suffix in the display name is a discipline reminder — when you see it in the picker, it nudges away from card-creating commands. Small but useful.

The discipline obligations under this path:

- **Never enable cron** on the laptop (default deny; leave it).
- **Never run `hermes serve`** on the laptop. No dispatcher, no API server, no card claiming.
- **Use only `chat`-mode commands** via ACP. Never `run draft`, never full `cite-check` passes, never `scope-project` invocations that write project-scratch files.
- Treat the laptop's ACP sessions with these profiles as *information surfaces*, not *production surfaces*. The architectural protection (the policy MCP) catches some write attempts, but not all — a Writer `run draft` invoked on the laptop will write to the synced vault if you're not careful.

### Path 2 — SSH-spawn into the primary (Option C default; optional under A+)

Under [Option C](../../07-roadmap/deployment-options.md) (VPS + laptop), the VPS is always reachable and has all seven profiles. Configure the `customAgents` entries to spawn the Hermes process on the VPS over SSH:

```json
{
  "id": "memoria-cartographer-remote",
  "displayName": "Cartographer (corpus queries) — via VPS",
  "command": "ssh",
  "args": ["vps-host", "hermes", "-p", "memoria-cartographer", "acp"],
  "env": []
}
```

No Memoria profile needed on the laptop. The agent-client plugin spawns SSH; SSH connects to the VPS; the VPS runs Hermes with the right profile; stdio streams back to the laptop's ACP pane.

Prerequisites:

- SSH always reachable (Tailscale or VPN connecting laptop to VPS).
- VPS configured to accept SSH from the laptop's key.
- Tolerance for ~100–500ms latency per message (noticeable in fast back-and-forth, unnoticeable in research-paced conversation).

This pattern also works under A+ as a fallback when the desktop happens to be on, but it's not the recommended A+ default because desktops sleep. Under C, where the primary never sleeps, SSH-spawn is unambiguously preferred.

### Deployment-specific recommendations

| Deployment | Preferred path for non-Socratic ACP on laptop |
|---|---|
| **A** (single workstation) | N/A — there's no laptop |
| **A+** (desktop + laptop, Syncthing) | Path 1 (local install + discipline) — desktop reliability is too low for SSH-spawn to be the default |
| **B** (Obsidian Sync, no VPS) | Same as A+ — desktop reliability dictates Path 1 if there's a laptop; same discipline applies |
| **B + VPS** (Obsidian Sync + VPS for cron) | Path 2 (SSH-spawn into the VPS) — same as Option C in this respect, since cron requires an always-on machine |
| **C** (VPS + laptop) | Path 2 (SSH-spawn) — VPS is always-on, no local install needed |

If you're on A+ and find yourself frequently wanting non-Socratic ACP from the laptop, that's a signal worth taking seriously: the discipline cost compounds, and the per-laptop profile installs accumulate maintenance overhead. The architectural answer is **migrate to Option C** — the VPS's always-on availability is exactly what makes SSH-spawn work, and it eliminates the install-and-discipline overhead entirely on every secondary device. Memoria's design explicitly names this trajectory: *"Start with A; migrate to A+ when a second device enters the workflow; graduate to C when you need always-on automation."* Frequent non-Socratic laptop ACP is one of the workflows that pulls operators toward C.

The plugin also persists `savedSessions` (an array of past ACP conversations with title and timestamp) in its `data.json`. These are useful for resuming a Socratic session, but they grow without bound — operators on long-running vaults should periodically prune them via the plugin's session-manager UI to keep `data.json` small. Unlike [obsidian-local-rest-api](obsidian-local-rest-api.md), this `data.json` contains no secrets and **is** safe to commit (sessions are conversation pointers, not credentials).
