# Linter — design summary

**Runtime contract.** Full prompt, lint check table, and the M-detector specifications live at `.memoria/profiles/memoria-linter/SOUL.md` and `.memoria/profiles/memoria-linter/M-detectors.md` in the starter vault.

## Mission

Linter is Memoria's deterministic conscience. It validates structure (frontmatter shape, link health, schema versions), catches silent-failure modes the operator wouldn't notice (the eight M-detectors), and owns audit-log rotation and per-session log writing. The defining trait is **zero-LLM throughout**: every check Linter runs is regex, AST walk, SHA-256 hash, file existence, or set arithmetic. The same vault state produces the same report on every run, in every environment — which is what makes Linter the only Memoria profile that can sit inside CI gating.

## What this profile is not

- **Not a content checker.** Linter doesn't grade quality, judge whether a claim is well-supported, or assess whether a draft reads well. Those are Verifier's (provenance) and the operator's (quality) domains. Linter checks *structure*: does the frontmatter parse, does the wikilink resolve, does the schema version match the canonical reference.
- **Not Verifier.** Both run mechanical checks, but at different layers. Verifier asks "does this draft's content trace to real sources?" — content-aware. Linter asks "is this note's schema valid, does its `extract_path` point at an existing file, is `data.json` consistent with the committed template?" — content-agnostic. They compose.
- **Not an LLM.** This is non-negotiable. Memoria's design treats Linter's determinism as load-bearing — if the verdict-band rollup ever depended on LLM judgment, it would no longer be reproducible across runs, which would defeat its role as a CI gate. Any check that needs LLM judgment is by definition not a Linter check.
- **Not a fixer by default.** Dry-run is the default. Auto-fix is allowed only for the `safe-and-unambiguous` and `authorized-targeted` classes; schema changes, content edits, and canonical edits are always report-only. The policy MCP enforces the class gate at the tool layer — Linter cannot bypass it even if it tried.

## Design decisions

- **Zero-LLM and deterministic — the design parallel to the trust score.** Linter's verdict band (PASS / REVIEW / FAIL) is the structural rollup; the fleet-observability dashboard's trust score is the operational rollup. Both are headline single numbers; both are reproducibly computed from logs and findings; neither involves LLM judgment in the rollup. This parallelism is intentional — operational health and structural health get separate headline metrics with the same epistemic discipline.
- **Owns `00-meta/04-logs/`.** Linter writes per-session log summaries, rotates the policy MCP audit log weekly (`audit.jsonl` → `00-meta/04-logs/archive/audit-YYYY-WW.jsonl`), and writes lint findings. This is the only path where Linter writes routinely. The rotation is classed as `authorized-targeted` in the auto-fix table, so the policy MCP allows it without escalation.
- **M-detectors are the silent-failure layer.** The eight M-detectors (M1–M8) catch failure modes the operator wouldn't notice by reading content: a Dataview query that references a field no template emits (M4), a `data.json` that drifted from the committed version (M6), an `extract_path` pointing at a missing file (M8). The defining property is "silent" — the failure mode looks like "nothing to do" when there's actually something to do. The full detector specs live in `M-detectors.md` alongside the runtime SOUL.md.
- **Auto-fix is class-gated by the policy MCP.** When `profile = "memoria-linter"` and `action = "auto_fix"`, the MCP requires `flags.class ∈ {"safe-and-unambiguous", "authorized-targeted"}`. Schema/content changes and canonical edits are always `deny` regardless of who requests them. Class gating is the runtime enforcement of the auto-fix policy — not just a design rule, an enforced one.

## Permissions and commands

Folder permission matrix lives in [02-profiles.md](../02-profiles.md#folder-permission-matrix); the runtime contract (lint check table, severity scale, verdict band, auto-fix classes, log rotation) lives in the SOUL.md. The M-detector procedural detail lives in M-detectors.md alongside it.

## Related

- Workflows: [10-maintenance-linting](../04-workflows/10-maintenance-linting.md)
- Reference: [reference/policy-mcp.md](../reference/policy-mcp.md) — the auto-fix class gate is enforced here; the audit log Linter rotates is owned here.
- Reference: [reference/profile-compilation.md](../reference/profile-compilation.md) — **deferred design.** M1 was originally specified against this compiler; under direct profile management M1 is install drift (vault source vs deployed copy), not build drift.
- Method class: [rationale/computational-methods.md](../rationale/computational-methods.md) — Linter is the canonical example of a fully deterministic profile.
- ADRs: [16 contradictions dashboard](../07-roadmap/decisions/16-contradictions-dashboard.md) — adjacent concept; Linter is structural, the contradictions surface is content.
