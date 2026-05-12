# Appendix C — Recipes

Where the body argues for a pattern, this section shows the pattern landing in a specific stack. The body has the *shape*; the recipes have the *implementation*.

## What lives here

- Audit-pipeline implementations per stack (Ruby on Rails, Node/TypeScript, others).
- Observability recipes — what to log, what to alert on, what the verification record looks like across metric backends.
- Cross-vendor team primitives — gating Markdown checklists, role-separated playbooks, branch-prefixed agent identifiers.
- Tool-specific patterns the body refers to in dated callouts.

## What doesn't live here

- Tutorials for getting started with a given framework — that's the framework's own docs.
- Generic best-practices content with no specific stack anchor — that's the body.
- Per-provider SDK call shapes — those live in `B-sdks/`.

## Organization

One entry per recipe per stack. Cross-stack patterns get one entry with sub-sections per stack.

## Contributions

Tool-specific recipes especially are the kind of thing that accumulates faster from a working community than from one author. Contributions follow three rules:

1. **Anchor every recipe to a body principle.** A recipe without a principle it serves is a snippet, not a recipe. State the principle the recipe is the implementation of (with a chapter cross-reference).
2. **Date the recipe.** Every entry has a `Last reviewed` line and a `Valid as of` date range. A recipe that no longer works in 2030 is still useful as historical context, but only if a reader can tell from the metadata.
3. **Include the pseudocode the body uses.** The recipe should re-state the body's pseudocode shape, then show the stack-specific implementation. The shape is the durable layer; the implementation is a snapshot.

## Entries

- [`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md) — from Ch. 5 (twelve-rule audit pipeline running on every render call in a TypeScript MCP server).
- [`observability-across-stacks.md`](observability-across-stacks.md) — from Ch. 8 (cache-bucket classifier, layered-status writeback, memory-verification habit, per-tool-surface metrics — implementations on Rails and Node/TypeScript).
- [`team-primitives-cross-vendor.md`](team-primitives-cross-vendor.md) — from Ch. 11 (cross-vendor PRs with branch-prefixed agent IDs, gating Markdown checklists wired to per-commit skills, operator-side approval gates, role-separated orchestration playbooks).
- [`stack-catalog.md`](stack-catalog.md) — from Ch. 6 (the stacks the war stories come from: Forge + leads-crm on Rails + Sidekiq; canvas-mcp + coide + markdown-toolkit on Node/TypeScript).
- [`runtime-trust-patterns.md`](runtime-trust-patterns.md) — from Ch. 10 (tool-command acceptance criteria, per-call audit pipelines, telemetry counters for infrastructure features — across markdown-toolkit, presentation-studio-mcp, Forge).
