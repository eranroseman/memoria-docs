---
mode: how-to
audience: operator
topic: plugins
---

# quickadd

QuickAdd is the plugin that registers Memoria's [command palette catalog](../../surfaces/command-palette.md) entries. [Templater](templater.md) is the templating layer QuickAdd invokes for capture commands. The two together cover every Memoria command that writes a new note or modifies frontmatter.

Setup pattern: a QuickAdd "Macro" per Memoria command. The macro composes:

1. A Templater template (for capture / note creation), OR
2. An HTTP POST to the Hermes API on port 8642 (for Kanban interactions), OR
3. An [`agent-client`](agent-client.md) plugin command invocation (for ACP profile switches), OR
4. Combinations of the above.

The standard Memoria commands and their implementation patterns are in [command-palette.md](../../surfaces/command-palette.md). QuickAdd doesn't ship a `data.json` template here — human setup varies enough that one standard config would constrain rather than help.

Load-bearing settings:

- `macros[*].name` must use the `Memoria:` prefix exactly. The `Cmd-P → "M"` filter convention depends on it. A macro named `memoria: capture` instead of `Memoria: capture` won't appear in the filter.

<!-- memoria-nav -->

---

[← Previous: templater-obsidian](templater.md)

[Next: obsidian-citation-plugin →](obsidian-citation-plugin.md)
