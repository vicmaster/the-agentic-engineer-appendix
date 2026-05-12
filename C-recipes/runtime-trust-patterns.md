# Runtime Trust Patterns — Tool Commands, Audit Pipelines, Telemetry Counters

**Source aside:** Ch. 10 (when to trust the output).
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026.

## The body principle

Ch. 10, Principle 10: *treat approval as a side-effect, not the terminal state.* The agent's stopping, the reviewer's approval, and the SDK's acceptance are all side-effects of the work. The terminal state is whatever structural check confirms the output meets the standard. Trust the check, not the claim.

## What this recipe captures

Ch. 10's aside names three runtime trust mechanisms implemented across three stacks. Each mechanism is the structural check that survives any single party's claim:

- **Tool-command acceptance criteria** — markdown-toolkit's reviewer-sweep fix. The acceptance criterion changed from *the reviewer confirms* to *`grep -r '\.tool-item-' src/` returns zero*. A tool command, not a human or agent observation.
- **Per-call audit pipelines** — presentation-studio-mcp's twelve-rule audit running on every render call. See [`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md) for the implementation sketch.
- **Telemetry counters distinguishing infrastructure-feature application from silent no-op** — Forge's four-bucket cache classifier reading `cache_creation_input_tokens` / `cache_read_input_tokens`. See [`../B-sdks/anthropic-prompt-caching.md`](../B-sdks/anthropic-prompt-caching.md) for the field shapes and [`observability-across-stacks.md`](observability-across-stacks.md) for the metric emission.

## The shape (from Ch. 10)

Every runtime trust mechanism the chapter argues for has the same three-part structure:

1. A **claim** the agent (or the SDK, or the reviewer) is making — *I produced a deck*, *I cached this block*, *I deleted the dead classes*.
2. A **structural check** that lives outside the claim — a tool command, an audit rule, a telemetry counter, a verification record.
3. A **trust signal that's the check's pass**, not the claim's stop.

The pattern:

> For sweeps, use a tool command. For artifacts, use a per-call audit. For infrastructure, use telemetry verification. For irreversible actions, gate the action.

## TODO (subsequent passes)

This recipe currently captures the three mechanisms and points back to the aside and the related entries. To be fleshed out:

- The exact `grep -r` patterns markdown-toolkit's playbook uses, and how the acceptance criterion is wired into the wave-3 reviewer artifact.
- The presentation-studio audit-rule structure (see also [`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md) — these two entries should converge as both fill out).
- The Forge cache-bucket emission to the metric stack (Rails ActiveSupport instrumentation? Custom middleware?).
- A worked counter-example: a system whose trust shape was *agent declares done* and what happened.

## What survives the tooling rotating

The principle: *trust the check, not the claim.* The specific tools (ripgrep, the audit-rule directory layout, the four-bucket classifier with current field names) will rotate. The structural-check-outside-the-claim shape doesn't.

## Cross-references

- Ch. 10 (When to Trust the Output) — full chapter treatment.
- Ch. 4 (Tool Surfaces and Trust Boundaries) — the gate/verify/trust framework at design time.
- Ch. 5 (Eval Loops Are the Product) — the audit pipeline as the trust signal at runtime.
- Ch. 8 (Observability for Probabilistic Systems) — the verification record as the observable form of the trust signal.
- [`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md), [`observability-across-stacks.md`](observability-across-stacks.md), [`../B-sdks/anthropic-prompt-caching.md`](../B-sdks/anthropic-prompt-caching.md) — the three implementation anchors.
