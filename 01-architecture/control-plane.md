# Control plane: how a human request reaches Hermes

The board defines *what state* a card is in. The policy MCP defines *where* a worker may write. But the system also needs a daily-use surface for the human to *trigger* discrete actions — queue a task, run an active card, audit a write, push registry state into a note. The control plane has three thin layers between the human and Hermes:

```text
Obsidian Command Palette  ──►  Local HTTP bridge  ──►  MCP servers  ──►  Hermes
       (UI)                       (gateway)            (policy + tasks)   (worker)
```

Each layer has exactly one job. None of them owns business logic except the MCP servers.

| Layer | What it does | What it doesn't do |
| --- | --- | --- |
| **Command Palette** (Obsidian plugin) | Reads frontmatter from the active note. POSTs a small JSON payload to the local bridge. Shows a status notice. | No database access. No policy evaluation. No task state changes. |
| **Local HTTP bridge** | Wraps the MCP tools behind HTTP endpoints (`/tasks/run`, `/policy/check`, `/sync/push`). Translates plugin requests into MCP tool calls. | No persistent state. No business rules. |
| **MCP servers** | `policy_mcp` checks permissions, writes the audit log. `tasks_mcp` updates the board, records handoffs, dispatches to Hermes. | No UI. No direct vault writes outside their declared tool surface. |
| **Hermes** | Runs the actual skill, writes the vault note, returns a structured result. | No board management — reports completion through the task MCP. |

## Why thin layers

- **Each layer is trivially replaceable.** Swap the Command Palette for a CLI, or the HTTP bridge for stdio MCP, without touching the policy or task logic.
- **Failure isolation.** If the plugin breaks, the policy MCP still runs against direct CLI invocations. If the bridge crashes, Hermes can still execute via its own MCP client.
- **One source of truth per concern.** Policy is in the MCP, not the plugin. Task state is in the MCP, not the bridge. The plugin can't accidentally cache state because it has none.

The discipline that makes this work: **never put business logic in the plugin or the bridge.** Both should be small enough that rewriting them from scratch in an afternoon is reasonable. When the system gets a new behavior, it goes in the MCP, not the UI.

## Fail-closed startup

The HTTP bridge is the smallest layer but the one with the largest blast radius if misconfigured — anything that can reach it can dispatch tasks and trigger writes. Two rules at startup:

- **Default to loopback only.** Bind to `127.0.0.1` unless explicitly configured otherwise. The vast majority of deployments (Options A, A+, and B; see [07-roadmap/deployment-options.md](../07-roadmap/deployment-options.md)) never need anything else. Under Option C and the micro-Option-C variant of A+ (where the laptop reaches the desktop's Hermes via Tailscale-bridged SSH or API), the binding may need to extend to a network interface — but only with `HERMES_BRIDGE_TOKEN` set.
- **Refuse non-loopback without authentication.** If the bridge is configured to bind a non-loopback interface and `HERMES_BRIDGE_TOKEN` (or equivalent shared secret) is not set, the bridge must refuse to start with a clear error. Failing closed at startup is what makes Deployment Option C (VPS-side bridge reachable from the desktop) viable without a long-running unauthenticated endpoint.

The discipline is binary: the bridge either runs in a trusted zone (loopback) or it runs with authentication. There is no third state, and there is no "I'll add the token later" — a misconfiguration here is invisible until someone notices traffic in the audit log they didn't expect.

## MCP server registration

MCP servers are registered **per profile**, following the Hermes-canonical pattern. Each profile carries an `mcp.json` at `~/.hermes/profiles/memoria-<name>/mcp.json` listing the servers that profile may talk to. The same information can also live in `config.yaml` under `mcp_servers:` — the two shapes are interchangeable and Hermes reads both. Under direct profile management, both files are hand-authored in `.memoria/profiles/memoria-<name>/` and copied verbatim by the installer (with `{{VAULT_PATH}}` substituted in `mcp.json`).

The shape, per server: `command` or `url`, `env`, `enabled`, `timeout`, and `tools.include` / `tools.exclude` filters. Memoria registers three servers across the relevant profiles:

- **`obsidian`** — vault read/write via the Obsidian Local REST API. All profiles.
- **`policy`** — the Memoria policy MCP (described above), reading lane-overrides from `.memoria/lane-overrides/`. All profiles.
- **`tasks`** — the task registry MCP that fronts the Hermes Kanban. All profiles.

Not every profile gets every tool from every server — `tools.include` filters narrow the surface per profile (e.g., the Socratic profile gets the `obsidian` server but only the read-side tools). The exact config file shape follows the [official Hermes MCP config reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference) — we don't redefine it here.

## Related

- Policy MCP (the rules being enforced): [reference/policy-mcp.md](../reference/policy-mcp.md)
- Board state machine (the cards being dispatched): [03-board.md](../03-board.md)
- Deployment options (when non-loopback binding becomes relevant): [07-roadmap/deployment-options.md](../07-roadmap/deployment-options.md)
