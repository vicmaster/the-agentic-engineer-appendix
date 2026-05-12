# Team Primitives — Cross-Vendor PRs, Gating Checklists, Role-Separated Playbooks

**Source aside:** Ch. 11 (building the team around agentic systems).
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026.

## The body principle

Ch. 11, Principle 11: *the structure is the team.* The team's leverage is the structural primitives you build — gating artifacts, cross-vendor review surfaces, recognition reflexes the operator keeps practicing — not the headcount. The membership rotates faster than the structure.

## What this recipe captures

Three structural primitives implemented across three different stacks:

- **Cross-vendor PRs with branch-prefixed agent identifiers** — a Rails CRM (`leads-crm`) where multiple AI vendors' agents push branches with prefixes like `claude/`, `codex/`, `cursor/`, and the human review surface is the PR list. Each agent's bias surface is different; the cross-vendor review catches what a single vendor's review would miss.
- **Gating Markdown checklists wired to per-commit skills** — the `VISION.md` pattern from leads-crm. A Markdown checklist at the repo root, owned by the human; a `/ship-feature` skill that reads it, finds an unchecked item, flips it, and commits the flip alongside the feature code. The artifact gates the work *before* it starts.
- **Role-separated orchestration playbooks producing named file artifacts per wave** — the markdown-toolkit Chrome extension build playbook. Each role (planner, architect, engineer, reviewer, QA) produces a named file artifact (`plan.md`, `root-cause.md`, `qa.test.ts`). The artifacts are the role separation; without the artifact, the role doesn't exist.

A fourth primitive worth naming from the aside but not yet detailed:

- **Operator-side approval gates encoded inside skill definitions** — the personal COO automation stack (`magmalabs-assistant`). Certain skill actions require explicit operator approval before they fire; the gate lives in the skill's YAML/Markdown header, not at the call site.

## The shape (from Ch. 11)

Each primitive shares three properties (Ch. 11's "Heuristic for the eval harness" — generalized):

1. **Cheap enough to read fully on every loop.** The artifact is short or scannable, not a sprawling document.
2. **Gating — updated before the work, not after.** The artifact's update is a precondition for the work, not a record of it.
3. **Human-owned — the agent maintains compliance; the human decides what's on the list.** Roles are bright: the agent flips checkboxes; the human writes the checkboxes.

Strip any of those three and the pattern degrades into informal coordination.

## TODO (subsequent passes)

This recipe currently captures the shape and points back to the aside. To be fleshed out:

- The exact branch-prefix convention used in leads-crm (the agent-ID-to-prefix mapping; how it interacts with the GitHub UI).
- The `VISION.md` file format, the `/ship-feature` skill's pseudocode (and Ruby implementation), the commit-pair pattern (`Update vision for X` lands first, `Add X` lands second).
- The markdown-toolkit playbook's wave structure (wave-1 fan-out, wave-2 implementation, wave-3 verification), with the artifact list per wave.
- The magmalabs-assistant operator-gate pattern — the YAML header convention, the consent-gate UI flow.

## What survives the vendors and harnesses changing

The principle: *the structure is the team.* Every agentic-team system will have a version of these primitives. The specific vendors (`claude/`, `codex/`, `cursor/`) will rotate; the cross-vendor review discipline doesn't. The specific harness affordances (`/ship-feature` syntax, consent-gate YAML) will rotate; the gating-artifact + role-separation patterns don't.

## Cross-references

- Ch. 11 (Building the Team Around Agentic Systems) — full chapter treatment.
- Ch. 9 (Agent vs. Script vs. Human) — the executor-matching framework the operator-gate primitive serves.
- Ch. 10 (When to Trust the Output) — the trust/gate/verify framework at runtime.
- [`../B-sdks/harness-affordances.md`](../B-sdks/harness-affordances.md) — the Claude Code harness's `disable-model-invocation` flag, consent-gate mechanics, and auto-memory layer the recipes lean on.
