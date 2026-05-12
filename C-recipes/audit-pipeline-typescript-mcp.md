# Audit Pipeline — TypeScript MCP Server (presentation-studio-mcp)

**Source aside:** Ch. 5 (eval loops are the product).
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026.

## The body principle

Ch. 5, Principle 5: *the audit is the eval.* The verification step lives in the production loop, same shape as a unit test, run on every call, severity-graded, structured enough for the agent to act on.

## The recipe at one glance

presentation-studio-mcp is a TypeScript MCP server that generates presentation decks. The agent supplies meaning (the bullets, the narrative); the tool supplies shape (layouts, geometry, typography, branding). Every render pass runs through twelve audit rules. Each rule emits a structured finding. The agent reads the report on its next turn and iterates until findings are resolved.

## The shape (from Ch. 5)

- One file per audit rule, each owning a single check.
- Each finding is `{ severity: warning | error, category: density | layout | contrast | missing-field, slide: <id>, field: <name>, message: <human-readable> }`.
- The audit runs *after* render and *before* delivery, on every call.
- The agent reads the report; if any finding has severity `error`, the agent re-renders. If all findings are `warning`, the agent decides whether to ship or iterate.
- The report's shape is the contract: structured enough that the agent can parse it, specific enough that the agent knows which slide to fix.

## TODO (subsequent passes)

This recipe currently captures the shape and the aside's claim. To be fleshed out:

- The twelve current rule names and what each one checks.
- Severity-threshold reasoning per rule.
- Per-layout density numbers (with `Last reviewed` per number — these will drift).
- The TypeScript types for `Finding`, `Rule`, `AuditReport`.
- A worked example: input + render + audit report + agent's iteration response.
- How the recipe ports to other stacks. Ch. 8 has a brief Rails sketch; expand here.

## Companion recipes

- The pixel-diff snapshot pattern from Ch. 5 (Property 2 — Cross-validation across changes). That deserves its own entry once captured.
- The `evaluate.ts` cautionary tale from canvas-mcp — *half-built eval surfaces ship as confident wrongness.* Capture as a counter-example entry under C-recipes.

## What survives the implementation changing

The principle: *the audit is the eval, run on every call, severity-graded, structured enough to iterate on.* The rule set will rotate as the tool's domain evolves. The pipeline shape (audit between render and delivery, structured findings, agent-readable report) survives.

## Cross-references

- Ch. 5 (Eval Loops Are the Product) — full chapter treatment.
- Ch. 10 (When to Trust the Output) — the audit pipeline as the runtime trust signal for artifacts.
- [`runtime-trust-patterns.md`](runtime-trust-patterns.md) — cross-stack version of the same trust-shape question.
