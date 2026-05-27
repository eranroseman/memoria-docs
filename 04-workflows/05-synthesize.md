# Workflow 5: Synthesize

**Group.** Upstream
**Goal.** Distill processed literature into durable claims in `30-synthesis/01-permanent/`. This workflow covers three pipeline stages: **synthesize** (initial claim at `maturity: seedling`), **develop** (cross-link to budding), **stabilize** (multi-source-supported to evergreen).

## Precondition

The source has been **processed** ([#14 Process](14-process.md)). Synthesize doesn't fire until the operator has thought about the source through Socratic conversation. If you're about to write a claim from a source you haven't processed yet, run #14 first.

## Steps

1. Open the source note (which has been processed).
2. Identify 1–3 durable claims from the processing.
3. Create or extend claim notes (one claim per note). New claims start at `maturity: seedling`.
4. Link claims to source citekeys and related notes. As cross-links accumulate, `maturity` advances to `budding`, then `evergreen`.
5. Keep each note to a single claim.

## Owners

Human owns authorship and judgment. Hermes (writer) can suggest links after the claim is written.

## Why processing is now a separate workflow

The `socratic-processing` capability used to be documented here as a pattern inside synthesize (loaded as a skill onto Writer). It's been promoted to its own stage and workflow ([#14](14-process.md)) — and the underlying capability is now owned by the dedicated Socratic profile, whose write-denied lane policy makes the protection architectural rather than skill-load-dependent. The queue of "literature notes thought about but not yet written into claims" becomes board-visible. Synthesize now strictly produces claim notes; processing strictly produces thinking.

## Command

`hermes run draft "..."` (after processing is done).

## Limit rule

A source note should generate **at most 2–3 claim notes**. If a single source spawns more, the note is being used as a summary dump rather than a synthesis source. Split the source note's reading into multiple sessions, or accept that some claims belong to another paper's claim note.

## Title rule

Claim note titles state the **claim**, not the **topic**. `jitai-timing-accuracy-depends-on-context-sensing.md` is a claim note title; `jitai-timing.md` is a topic stub and should not exist as a claim note.

## When to create vs. extend

- New claim that changes how you understand an existing concept → extend the existing claim note.
- New claim with no existing note → create a new claim note.
- Claim that contradicts an existing claim → create a new claim note documenting the tension.

## Pre-filing similarity check

Before a new `claim-note` is filed, `hermes -p memoria-verifier run similarity-check` surfaces the top 3 most-similar existing notes. This is a point-of-action duplicate guard — distinct from the monthly retrospective `find-duplicates` sweep. If similarity exceeds the threshold (~0.8), Verifier flags the card with `near-duplicate-candidate`; the operator decides to file, merge, or extend. The check is informational, never blocking — see [profiles/verifier.md](../profiles/verifier.md) for the full protocol.

## Example

Human reads `mamykina2010sense.md` and extracts one durable claim: "JITAI receptivity decreases under high cognitive load." Runs `hermes run similarity-check "receptivity decreases under cognitive load"` → no near-duplicate above 0.8 → creates `30-synthesis/01-permanent/receptivity-decreases-under-high-cognitive-load.md` at `maturity: seedling` → cites `[[mamykina2010sense]]` in the body, adds a one-line "Connections" section. Over the next month, two more sources confirm and one contradicts; cross-links accumulate → `maturity` advances to `budding`.

## Related

- **Previous workflow:** [#14 Process](14-process.md)
- **Next workflow:** [#6 Promote to wiki](06-promote-wiki.md) (when claim reaches `evergreen`)
- **Profile:** [profiles/writer.md](../profiles/writer.md)
- **Linking patterns:** [reference/linking-patterns.md](../reference/linking-patterns.md)
