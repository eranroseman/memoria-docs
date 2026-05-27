# Timeline and phases

The week-by-week ramp from the [minimum viable system](../07-roadmap.md#minimum-viable-system) to production, plus the six implementation phases that structure the work.

The roadmap is ordered so each phase produces a working, useful slice — you should not need to complete phase 4 before phase 1 is paying off.

## Implementation timeline

A concrete week-by-week ramp from MVS to production.

### Weeks 1–2 — Infrastructure

- Set up Zotero + BBT with the citekey format `auth.lower + year + shorttitle(1,0)` (see [ADR-4](decisions/04-citekey-naming-convention.md)).
- Create the MVS folder structure.
- Install Hermes and configure terminal access.
- Drop in two templates: `source-note.md`, `claim-note.md`.
- **Ingest 5 pilot papers.** Verify ingest works end-to-end.
- **Triage all 5.** Verify `_draft_classification` proposals look reasonable.

Exit: a new note from ingest lands in the right folder with the right frontmatter.

### Weeks 3–4 — Seed the corpus

- Ingest your **30–50 most important papers**.
- Triage them all — this builds your corpus profile and validates the vocabulary.
- Write **5–10 claim notes** from the most synthesis-ready literature.
- Run the weekly ritual twice (even if minimal) to confirm 30-minute target.

Exit: triage feels routine; claim notes have started to form a graph.

### Month 2 — Deepen synthesis

- Complete triage on the full initial corpus.
- Write claim notes consistently — target **2–3 per week**.
- Run `hermes run find-duplicates` for the first time — establish the merge discipline.
- Expand schema beyond MVS where you feel the absence (`pub_status`, `full_text_reviewed`, etc.).

Exit: the claim note layer is dense enough that you start linking across papers.

### Months 3–4 — Activate advanced features

- Create the **first MOC** when a topic crosses 15 claim notes.
- Activate lag metrics in the weekly dashboard (once `triage_completed` is populated).
- Begin Canvas sessions for chapter planning.
- Start systematic `hermes run discover` for whatever scoping work you have active.
- Add items / entities folders if you're consistently citing tools and people.

Exit: the vault stops being a place you build and starts being a place you write from.

### Month 5+ — Production use

- Migrate to a multi-device setup if the Chromebook becomes a regular working device or batch ingest needs to run overnight — see the [Deployment options](deployment-options.md) matrix.
- Build child MOCs as topic clusters become dense (`>20` claim notes on a branch).
- Begin drafting from the wiki layer.
- Promote from mode-based to four-profile (or six-profile) when the bottleneck appears.

The migration path always points the same direction: **start narrow, expand when the corpus demands it.** Avoid adding structure pre-emptively.

## Implementation phases

### Phase 1 — Naming and schema (rename + contract)

**Goal.** Establish Memoria as the canonical name and lock down the schema contract that Hermes and the board will share.

**Steps.**

1. Rename the system in the vault documentation and any AGENTS-style schema files. Keep `research-wiki` as an alias for migration.
2. Define the canonical card states, lane-to-profile mapping, and review gate rules in one schema document so Hermes and the board share the same contract.
3. Confirm note-type names match the canonical set in [05-notes-folders.md](../05-notes-folders.md): `fleeting-note`, `synthesis-note`, `source-note`, `item-note`, `entity-*`, `claim-note`, `moc`, `reference-note`, `project-note`, `code-note`, `canvas`, `draft`, `deliverable`. Keep any older names as deprecated aliases during transition.
4. Commit the design document set in the separate `memoria-docs/` repo.

**Exit criteria.** The schema is the single source of truth. Anyone (including the agent) can read it and know what the states, types, and folders mean.

### Phase 2 — Vault structure (folders + templates)

**Goal.** Ensure the vault is structurally ready for the workflows.

**Steps.**

1. Confirm the folder structure matches [05-notes-folders.md](../05-notes-folders.md). Create any missing folders.
2. Drop the 11 templates into `00-meta/01-templates/`.
3. Migrate any existing notes whose folder no longer matches their type.
4. Set up `00-meta/index.md` and `00-meta/weekly-dashboard.md` with the queries from [06-surfaces.md](../06-surfaces.md).
5. Confirm Zotero + Better BibTeX exports to `00-meta/03-config/library.bib`.

**Exit criteria.** A new note created from a template lands in the right folder with the right frontmatter. Dashboards open and show real data.

### Phase 3 — Hermes profiles (seven contracts)

**Goal.** Configure the seven Hermes profiles with their permissions, commands, skills, and MCPs — on the **primary** machine. The full seven only need to live on the canonical dispatcher; secondary devices install a narrower set (see [Per-device install sets](#per-device-install-sets) below).

**Steps (primary machine).**

1. Create Hermes profiles for researcher, cartographer, socratic, writer, verifier, coder, linter.
2. For each profile, drop its AGENTS.md into the Hermes configuration. The drafts are in [profiles/](../profiles/).
3. Attach only the commands, skills, and MCPs the profile needs (see [02-profiles.md](../02-profiles.md)).
4. Test each profile in isolation with a small task that exercises its lane.
5. Confirm that permission boundaries hold: the Researcher cannot write to `30-synthesis/01-permanent/`; the Socratic profile cannot write anywhere; the Cartographer can only write to project-scratch corpus-map paths; the Linter cannot silently move files; etc.

#### Per-device install sets

For multi-device deployments (Options A+ and C — see [Deployment options](deployment-options.md) and [Secondary-device patterns](deployment-options.md#secondary-device-patterns-a-and-c)), not every machine compiles the full seven profiles. The right install set depends on the device's role:

| Device | Profiles to compile | Why |
| --- | --- | --- |
| **Primary** (single workstation under A; desktop under A+; VPS under C) | All seven | This is the canonical Hermes dispatcher. All cron, all queue dispatch, all overnight loops run here. |
| **Secondary, reader role** (laptop under A+; PI's laptop under C) | `memoria-socratic` only (baseline); add Cartographer / Writer / Verifier only if a specific use case justifies the discipline obligations | Structural enforcement: profiles not installed cannot be invoked. Socratic's lane policy (`policy.allow.write: []`, `routing.invocation: interactive_only`) means it's architecturally safe with zero operator discipline. |
| **Secondary, developer role** (dev laptop under C) | All seven, with **mandatory** `HERMES_HOME` isolation and pointed at a *test vault* (not the canonical synced vault) | Two Hermes dispatchers can coexist *only* if they target different vaults. Under C this is mandatory because the VPS is reachable on the same Tailscale as the dev's machine — pointing the dev's Hermes at the production vault while the VPS is dispatching against it is the most likely real-world incident class. The dev iterates against fixtures; the production vault is touched by exactly one Hermes (the VPS's). |

The principle: **install only what the device's role can safely run.** This is the "structural enforcement over behavioral enforcement" pattern — `hermes -p memoria-researcher discover` returning "profile not found" on a reader laptop is stronger than the operator remembering not to invoke it. See [Secondary-device patterns](deployment-options.md#secondary-device-patterns-a-and-c) for the per-profile risk breakdown.

**Exit criteria.** Each profile can do its job end-to-end on a small example on the primary. Permission violations fail loudly, not silently. Secondary devices have the right per-role install set; running a profile that wasn't installed returns a clean "profile not found" rather than a silent run.

### Phase 4 — Kanban board (state machine)

**Goal.** Stand up the board as the orchestration layer.

**Steps.**

1. Stand up the **Hermes native Kanban** (see [03-board.md](../03-board.md) for the canonical choice and the alternatives that were considered).
2. Configure the nine states: `pending`, `ready`, `active`, `blocked-on-human`, `awaiting-review`, `rejected`, `retry-needed`, `approved`, `done`.
3. Configure the schema fields: `status`, `assignee`, `lane`, `blocked_reason`, `retry_count`, `handoff_note`, `last_updated`, `canonical_target`, `review_state`, `review_owner`, `review_requested_at`, `reviewed_at`.
4. Configure dispatch logic so cards in `pending`, `awaiting-review`, or `blocked-on-human` are not claimable by non-review workers.
5. Configure retry behavior so failed work reuses the same card and increments `retry_count`.
6. Configure the Kanban dispatch rules to advance cards (no Orchestrator profile); configure Verifier and Linter to set `review_state` as recommendations the operator approves.
7. **Stand up a mock bridge for board-layer testing.** A small in-memory or fixture-backed server that implements the same HTTP endpoints as the real Obsidian-side bridge (see [01-architecture/control-plane.md](../01-architecture/control-plane.md)) but writes to a scratch directory instead of the live vault. The board's dispatch logic, retry rules, and state transitions are exercised end-to-end against the mock without Obsidian running — this is also what CI runs against. The thin-layers discipline (one job per layer, no business logic in the bridge) is what makes the mock cheap to build and maintain.

**Exit criteria.** A card can flow from `ready` through review and back to a worker. Retries reuse the same card. The review gate blocks dispatch. The state machine has a regression test suite that runs without live Obsidian.

### Phase 5 — Pilot corpus (small end-to-end run)

**Goal.** Run a small corpus through the full pipeline and refine the lane rules, review states, and handoff notes before scaling.

**Steps.**

1. Pick 5–10 sources representative of your active research.
2. Run them through ingest → triage → synthesize → promote.
3. For each card, exercise the review gate explicitly. Watch for any state that "leaks" past review.
4. Note handoff friction: where did the next worker have to re-derive context? Improve handoff notes.
5. Note routing friction: where did the dispatch rules misfire? Improve lane-override rules.
6. Refine the schema if a field is missing or unused.

**Exit criteria.** A new source can move through the pipeline without surprises. The weekly dashboard meaningfully reflects state.

### Phase 6 — Scale and automate

**Goal.** Apply Memoria to the full research corpus and add automation at the edges.

**Steps.**

1. Migrate the existing corpus into the new structure (one folder at a time, with linter dry-runs at each stage).
2. Add scheduled tasks: nightly enrichment refresh, weekly lint, monthly stale-note check.
3. Add a board summary dashboard if the board is dense enough to warrant it.
4. Set up Pandoc export pipeline for `40-workbench/02-drafts/`.
5. Configure session logging to write to `00-meta/04-logs/` and commit weekly.

**Exit criteria.** The system runs without daily babysitting. The human shows up for review and synthesis; everything else flows.
