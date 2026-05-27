# Workflow 14: Process

**Group.** Upstream (stage workflow, added 2026-05)
**Goal.** Transform a triaged source from "fully classified metadata" into "thought-about enough to write a claim from." No artifact is produced. The output is the operator's sharpened understanding.

## Pipeline position

Between [#4 Triage](04-triage.md) and [#5 Synthesize](05-synthesize.md).

## Steps

1. A `process` card opens automatically when a literature note's `triage_status` becomes `full`. The card targets the operator (no profile claims it).
2. The operator opens the source note, reads (or re-reads) the relevant passages, and switches to the **Socratic profile** via ACP: `Cmd-P → Memoria: invoke Socratic on current note`. The ACP pane on the right (see [06-surfaces.md](../06-surfaces/modal.md) — Reading & Processing workspace) opens the Socratic profile, whose lane policy is `policy.allow.write: []` (write-denied across the entire vault).
3. The Socratic profile runs the `socratic-processing` command: "What's the strongest single claim? What does it connect to? What would falsify it? What's the smallest version of this idea that stands alone?" — the questions are the profile's whole product. The operator answers in dialogue.
4. The operator either: (a) decides this source yields one or more claim notes → moves the card to `ready-for-synthesis` and runs [#5 Synthesize](05-synthesize.md), or (b) decides the source doesn't yield a claim → closes the card with `outcome: no-claim` and notes the reason in the source note's body.

## Owners

Human owns the thinking; Socratic owns the questioning. Architecturally the Socratic profile cannot author the claim — its lane policy denies all writes. That constraint is what makes processing remain the operator's cognitive work rather than something that gets quietly authored.

## Card lifecycle

`ready` (opens automatically when `triage_status` becomes `full`; the card is on the operator-targeted lane, not Kanban-dispatched) → operator invokes Socratic via ACP → Socratic conversation happens (no card state change, no writes) → operator writes the claim note in `30-synthesis/01-permanent/`, which closes the card via the git hook → `done` (with `outcome: claim-written`) OR operator manually closes with `outcome: no-claim`. The card never reaches `active` in the dispatcher sense — Socratic is `routing.invocation: interactive_only`.

## Command

`Cmd-P → Memoria: invoke Socratic on current note` (preferred) or `hermes -p memoria-socratic chat --command socratic-processing --source {citekey}` (CLI).

## Why the card auto-closes only on operator action

The system cannot tell whether thinking has happened. A literature note with `triage_status: full` and no `process` card closure is the dashboard's signal that the operator is behind on processing. Letting the agent close the card would defeat the visibility argument that made `process` a stage in the first place.

## Example

`mamykina2010sense.md` finishes triage at `triage_status: full`. A `process` card opens. Three days later it's still open — the `weekly-dashboard` surfaces it in the "process queue" view. The operator opens the note, invokes Socratic, and the profile asks: "What's Mamykina's strongest claim?" The operator answers ("receptivity depends on momentary cognitive load — strongest at idle / commute / waiting-room moments, weakest during task focus"). Socratic follows up: "What would falsify it?" → operator thinks → answers. After ten minutes, the operator closes the ACP pane and switches to Writer profile (or just writes in the editor without ACP) to author `30-synthesis/01-permanent/receptivity-decreases-under-high-cognitive-load.md` from scratch in their own words. The `process` card auto-closes on the new claim note's creation.

## Related

- **Profile:** [profiles/socratic.md](../profiles/socratic.md)
- **Dashboard:** [process-queue.md](../dashboards/process-queue.md)
- **Previous workflow:** [#4 Triage](04-triage.md)
- **Next workflow:** [#5 Synthesize](05-synthesize.md)
