# Workflow 8: Writing and drafting

**Group.** Downstream
**Goal.** Turn synthesized knowledge into chapters, papers, or other drafts. This workflow is the **umbrella** for the downstream pipeline — [#15 Scope](15-scope.md), [#16 Frame](16-frame.md), [#17 Verify](17-verify.md), and [#18 Revise](18-revise.md) each own one stage; this workflow ties them together and owns the outline / draft / export / deliverable middle.

## Steps (mapped to downstream stages)

1. **Scope.** Run [#15](15-scope.md) to produce `corpus-map.md`. Decide if the project is ready to write or needs more reading.
2. **Frame.** Run [#16](16-frame.md) to generate 2–3 competing framings; commit to one in `framing/CHOSEN.md`.
3. **Canvas** *(optional)*. For chapter-sized work (8–15 claim notes), use Canvas to arrange them spatially. See Canvas → Draft sub-workflow below.
4. **Outline.** Derive heading scaffold from `framing/CHOSEN.md` (and Canvas groupings if used).
5. **Draft.** Write prose in `40-workbench/02-drafts/<project>/` with citekeys. Commit triggers the verify hook ([#17](17-verify.md)).
6. **Verify.** [#17](17-verify.md) fires automatically on draft commit. Read the `[!verification]` callout at the top of the draft.
7. **Revise.** [#18](18-revise.md) closes the gap-loop. Loop back to step 5 until verify returns clean (or remaining gaps are accepted-soft).
8. **Export.** Run Pandoc.
9. **Deliverable.** Final output lands in `50-deliverables/`.

## Owners

Human owns argument assembly and drafting (steps 3–5, 7). Cartographer owns step 1 via `scope-project`. Writer with `counter-outline` owns step 2 (framing — and Socratic with `lens-reading` for lens-based framings); Verifier owns step 6 (verify). The middle (outline, draft) and the end (export, deliverable) are operator-led with Hermes assistance.

## Export command

```bash
pandoc 40-workbench/02-drafts/{chapter}.md --citeproc \
  --bibliography 00-meta/03-config/library.bib \
  --csl 00-meta/02-csl/apa.csl \
  -o 50-deliverables/03-exports/{chapter}.docx
```

**Precondition:** the verify→revise loop (steps 6–7) has closed.

## Canvas → Draft sub-workflow

Canvas is the spatial layer between synthesis and writing. It is an argument map, not a canonical note. Use it when you have **8–15 relevant claim notes** and need to see how they fit together before drafting.

**1. Collect notes onto the canvas.** Drag claim notes from the file explorer onto a new Canvas file. Save as `40-workbench/04-canvas/{chapter-or-section-name}.canvas`.

**2. Arrange spatially.** Group notes by sub-argument. Place notes that support the same claim together. Draw arrows showing logical flow: claim → evidence → implication. Use text cards for transitional claims that aren't yet in any note.

**3. Identify gaps.** Any text card that points to an empty space is a gap. A claim with no supporting note needs either a new source note or a new claim note before drafting starts. **Do not draft past unsupported claims.**

**4. Build the outline.** Once the spatial arrangement is stable, write the outline directly from Canvas groupings. Either by hand or:

```bash
hermes run draft "outline the argument on {canvas topic}" \
  --context 30-synthesis/01-permanent/{note1}.md \
  --context 30-synthesis/01-permanent/{note2}.md
```

**5. Move to draft.** Create `40-workbench/02-drafts/{chapter-name}.md`. Open Canvas in a split pane. Write section by section, citing citekeys from notes on the Canvas.

**6. Archive the Canvas.** When the section is drafted, move the canvas to `95-archive/` with `status: frozen`. Canvases are scratchpads, not deliverables — but they have provenance value (the argument map you built).

**Conventions:**

- One Canvas per chapter section or argument cluster, not one per paper.
- Never embed a Canvas in a draft note — `.canvas` files don't export to Pandoc.
- Canvases live in `40-workbench/04-canvas/` while active; archived when frozen.

## Related

- **Stages owned elsewhere:** [#15 Scope](15-scope.md), [#16 Frame](16-frame.md), [#17 Verify](17-verify.md), [#18 Revise](18-revise.md)
- **Profile:** [profiles/writer.md](../profiles/writer.md)
- **Workspace layout:** [06-surfaces.md — Drafting workspace](../06-surfaces/modal.md)
