---
mode: how-to
audience: operator
topic: obsidian-ui
---

# Home — the startup front door

`Home.md` (vault root) is the note Memoria opens on launch via [obsidian-homepage](../obsidian-plugins/recommended/obsidian-homepage.md) ([ADR-25](../decisions/25-homepage-front-door.md)). It is a **launchpad**, not a dashboard: it *surfaces* the dashboards rather than being one of the twelve. Role split — [Daily Health](../dashboards/daily-health.md) is the health glance; Home leads with it and adds navigation plus quick actions, so a session starts at one deliberate place.

## What Home contains

- **Lead glance:** the Daily Health red signals, linked (or a block-embedded) so opening Home *is* the morning health check.
- **Navigate:** links to the board and the knowledge dashboards ([open-questions](../dashboards/open-questions.md), [contradictions](../dashboards/contradictions.md), [reading-pipeline](../dashboards/reading-pipeline.md)).
- **Quick actions:** command-palette entries (`Memoria: …`) for the common moves (new paper, find, discuss) — see [command-palette.md](command-palette.md).
- **Recent activity:** a short Dataview list of recently-touched notes.

## Design rules

- **Thin, and a pure consumer.** Home embeds/links existing surfaces; it never becomes a second place that *computes* health or questions. If a query belongs to a dashboard, it lives in that dashboard and Home links it.
- **A note, not a plugin UI.** Home is Dataview-in-a-note — git-tracked, lintable, embeddable. This is exactly why a plugin-rendered start page was rejected ([ADR-25](../decisions/25-homepage-front-door.md)).
- **Graceful degradation.** With the homepage plugin absent, `Home.md` is still an ordinary note the human can pin — nothing depends on auto-open.

## Runtime scaffold

A thin starting point for the starter vault's `Home.md` (Dataview; trim or extend to taste). Adjust links to the vault's actual dashboard filenames:

````markdown
---
type: dashboard
---
# 🏠 Memoria — Home

**Health:** [[index|Daily Health]]  ·  **Board:** [[board-state]]

## Knowledge

[[open-questions]] · [[contradictions]] · [[reading-pipeline]]

## Quick actions

`Memoria: New paper note` · `Memoria: Find` · `Memoria: Discuss current note`

## Recently touched

```dataview
TABLE file.mtime AS "Updated"
FROM "30-synthesis" OR "20-sources"
SORT file.mtime DESC
LIMIT 8
```
````

The Daily Health dashboard ships at `00-meta/01-dashboards/index.md`; link it as `[[index|Daily Health]]` or block-embed a red-signals section if Daily Health exposes one.
