# templater-obsidian

Templater is the templating layer Memoria uses for safe-and-unambiguous frontmatter fixes (see [linter.md](../../profiles/linter.md#implementing-safe-and-unambiguous-fixes-via-templater)) and for capture commands invoked via [quickadd](quickadd.md).

Load-bearing settings:

- `templates_folder: "00-meta/01-templates"` — must match the canonical templates location in [05-notes-folders.md](../../05-notes-folders.md). Memoria uses numbered subfolders consistently throughout the vault (`00-meta/01-templates`, `00-meta/02-csl`, `00-meta/03-config`, etc.); diverging just for the templates folder breaks the convention. Note: research-wiki uses `00-meta/templates` because research-wiki's schema doesn't number `00-meta`'s children — different vault, different convention.
- `trigger_on_file_creation: false` — **Memoria default is off**, not on. Setting this to `true` causes Templater to fire on every newly created file, including files agents create through the policy MCP. Since agents already populate frontmatter through their own templates (e.g., the Researcher uses [obsidian-citation-plugin](obsidian-citation-plugin.md)'s `literatureNoteContentTemplate`), auto-triggering Templater on top of agent writes either races with or overwrites the agent's frontmatter. Keep it off; operators invoke templates explicitly via the command palette (`Cmd-P → Templater: Insert template`) on the rare files they create by hand.
- `enable_system_commands: false` — keep system commands off unless you have a specific need; the linter's `safe-and-unambiguous` Templater scripts don't need them.

Inline `data.json`:

```json
{
  "templates_folder": "00-meta/01-templates",
  "auto_jump_to_cursor": true,
  "use_system_commands": false,
  "trigger_on_file_creation": false,
  "enable_folder_templates": false,
  "syntax_highlighting": true,
  "enabled_templates_hotkeys": [],
  "startup_templates": []
}
```
