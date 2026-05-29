---
mode: how-to
audience: operator
topic: plugins
---

# obsidian-excalidraw

Optional. Stores hand-drawn diagrams as `.excalidraw` files. Canvas (Obsidian core) handles most spatial argument-mapping needs (see [workflows/downstream/write.md Canvas → Draft sub-workflow](../../workflows/downstream/write.md#canvas--draft-sub-workflow)) — Excalidraw earns its keep specifically for *diagrams* (architecture sketches, mechanism-of-action schematics, conceptual figures for papers) where Canvas's note-link affordance is overkill.

Excalidraw files live alongside their parent notes (typically in `40-workbench/01-projects/<project>/figures/` or `90-assets/`). Memoria doesn't ship configuration for it; the defaults are fine.
