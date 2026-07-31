# Frontier Models — 2026 Snapshot

**Source asides:** Ch. 1 (the half-sentence brief), Ch. 2 (worked example of the Reader Contract).
**Last reviewed:** 2026-07-31.
**Valid as of:** Q3 2026.

## What the body says

Both Ch. 1 and Ch. 2 reference *a frontier hosted model from a major provider* that produced the half-sentence brief in the opening story. The body deliberately doesn't name the model in prose; this entry is where the specifics live.

## The dated specifics

The half-sentence brief came out of **Forge**, whose leadership-brief capability called **Claude Sonnet 4.6** (`claude-sonnet-4-6`) on Anthropic's API. The default `max_tokens` budget on the call was 4,096. The fix that landed in Forge commit `2151549` doubled the budget to 8,192. The model's `stop_reason` on the truncated calls was `"max_tokens"` — the field that, if the handler had been reading it, would have caught the failure before the brief shipped.

The Claude Code harnesses elsewhere in the corpus (coide, markdown-toolkit, the magma-core review session, the operator stack) ran on whatever the CLI defaulted to across the 2026-03 → 2026-07 window — Opus- and Sonnet-tier models of that generation.

**Current frontier lineup (as of 2026-07-31):** the Claude 5 family — Opus 5 (`claude-opus-5`), Sonnet 5 (`claude-sonnet-5`), and Fable 5 (`claude-fable-5`, the most capable) — alongside Claude Opus 4.8 (`claude-opus-4-8`) and Claude Haiku 4.5 (`claude-haiku-4-5`). By the time you read this, that list has almost certainly turned over again — which is exactly why the body doesn't print it.

## Why the body doesn't name it

By editorial choice, restated in Ch. 2's Reader Contract:

> Twelve months later the prompts behave differently, the SDK has a new shape, the framework has been replaced or absorbed, and a different model is now the strongest.

Naming a model in the body dates the book within a year. Naming it here, with `Last reviewed`, is honest about how long the specifics stay valid.

## What survives

The principle from Ch. 1: *valid-looking invalid output*. Probabilistic systems can produce structurally correct, semantically wrong output without raising. The fact that this capability's default was 4,096 in 2026 is dated; the fact that *some* model's default will be the wrong number for *some* capability's real-shaped input is durable.

## Cross-references

- [`B-sdks/`](../B-sdks/) — current SDK shape for reading `stop_reason` per provider.
- [`C-recipes/`](../C-recipes/) — the headroom-test recipe (Ch. 3) for picking a budget that beats the typical-case fail.
