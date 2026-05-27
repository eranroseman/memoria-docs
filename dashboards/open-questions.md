# `open-questions` — design summary

**Runtime artifact.** This dashboard ships at `00-meta/05-dashboards/open-questions.md` in the [starter vault](https://github.com/eranroseman/memoria-vault) and runs in Obsidian via Dataview. The summary below covers its design role; the runtime queries live in the vault file.

## Mission

Turn the vault into a research agenda by surfacing every note that contains an explicit `# Open questions` section. Open this when planning the next research direction — what questions has past synthesis surfaced that haven't been answered yet? The dashboard reads claim notes and literature notes (the two places open-questions sections accumulate naturally) and provides a single aggregated view across the corpus.

## What this dashboard is not

- **Not a synthesizer.** It surfaces existing open-questions sections; it doesn't propose new questions, cluster them, or rank them by importance. That's the operator's job (or a future Cartographer skill).
- **Not the only place questions live.** Inline `# Open questions` sections inside claim notes and literature notes are the *durable* questions — the ones worth re-finding months later. Ephemeral session questions live in fleeting-notes and discover-pass output, not here.
- **Not auto-resolving.** When a question is answered, the operator manually removes or restructures the section. The dashboard doesn't track which questions have been answered (no `resolved:` state) — that would require a richer schema than free-form section content.

## Design decisions

- **Free-form section, not structured frontmatter.** Questions live in markdown body content (`# Open questions` heading) rather than a frontmatter `open_questions: []` list. The reasoning: questions are *prose* (often paragraphs with context), and constraining them to flat YAML strings would lose the framing. The cost is that the dashboard can't filter/group by question metadata — it just surfaces the notes that have such sections.
- **Two source folders: `30-synthesis/01-permanent/` and `20-sources/01-literature/`.** These are the two places where durable questions accumulate. Project pages might also have questions, but those are typically operational ("what should we do next") rather than research-direction ("what's still unknown").
- **Sort by `file.mtime` not by question count.** Recently-touched notes are likely the operator's current focus; surfacing those first matters more than ranking by question density.
- **No capability gate.** Unlike most dashboards, this one works on day one — any note with `# Open questions` appears immediately.

## Related

- [05-notes-folders.md](../05-notes-folders.md) — claim-note structure (Open questions is a recommended section)
- [`weekly-dashboard`](weekly-dashboard.md) — surfaces this dashboard as part of the Friday ritual
- [Researcher design summary](../profiles/researcher.md) — the upstream discovery direction is informed by aggregated open questions
