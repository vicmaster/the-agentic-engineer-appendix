# Observability — Verification Records Across Stacks

**Source aside:** Ch. 8 (observability for probabilistic systems).
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026.

## The body principle

Ch. 8, Principle 8: *log the claim, not the call.* Every contractual claim your code is making — about layer transitions, provider behavior, stored facts, tool surfaces — needs a verification record that names whether the claim still holds.

## What this recipe captures

Ch. 8's aside names four implementations of the *log the claim* discipline across two stacks:

- **Cache-bucket classifier** (Ruby on Rails, Forge) — see [`B-sdks/anthropic-prompt-caching.md`](../B-sdks/anthropic-prompt-caching.md) for the SDK field shape; this entry shows the metric emission and alerting pattern in Rails.
- **Layered-status writeback** (Ruby on Rails, Forge) — the `routing_status` vs. `execution_status` split that fixed the column-was-lying bug.
- **Memory-verification habit** (Node/TypeScript, coide) — the read-current-code-before-acting-on-recall pattern from Ch. 6's *plausible drift*.
- **Per-tool-surface metrics** (Ruby on Rails, leads-crm) — the MCP-layer observability gap that the HTTP-to-HTTPS one-line patch surfaced.

## The shape (from Ch. 8)

For every infrastructure feature, persisted status, stored fact, or tool surface your system depends on, the verification record has the same shape:

- A **claim** the code is making (in prose, written down).
- A **verification step** that, when run, says whether the claim still holds.
- A **structured record** of the verification (metric, log line, audit row).
- An **alert on the verification breaking**, not on the call failing.

## TODO (subsequent passes)

This recipe currently sketches the four implementations and points back to the aside. To be fleshed out:

- The Rails Sidekiq integration for the layered-status writeback (after-commit hooks, callback wiring).
- The metric backend conventions (Prometheus? StatsD? Datadog?) — pick one example and show the verification-record emission.
- The TypeScript pattern for memory-verification — how the agent's working pattern reads a stored fact and emits the verification log.
- The MCP surface observability sketch — per-tool call rates, transport-level signals, auth-flow metrics. The leads-crm `77a026b` story is the anchor.

## Expected-event alerting examples

From Ch. 8:

- *Cache-hit count for capability X has been zero for ninety minutes during business hours.*
- *Routine Y's `last_run_status` has been `queued` for more than ten minutes after the scheduled fire time.*
- *Audit finding Z has been open for more than three iteration loops without a re-render attempt.*

The alert shape: fire on a state the system should have left and didn't, not on a state the system shouldn't have entered.

## What survives the metric stack changing

The principle: *log the claim, not just the call; alert on what didn't happen, not just on what did.* The metric backend will rotate underneath. Every probabilistic system has its own version of these claims; the discipline is finding them and instrumenting them, not memorizing any specific stack's affordances.

## Cross-references

- Ch. 8 (Observability for Probabilistic Systems) — full chapter treatment.
- Ch. 3 (Context as a Resource) — origin of the cache-bucket classifier shape.
- Ch. 5 (Eval Loops Are the Product) — the audit pipeline as the runtime cousin of observability.
- [`B-sdks/anthropic-prompt-caching.md`](../B-sdks/anthropic-prompt-caching.md) — the SDK fields the cache-bucket classifier reads.
- [`runtime-trust-patterns.md`](runtime-trust-patterns.md) — when the verification record *is* the trust signal at runtime.
