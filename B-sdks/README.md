# Appendix B — SDKs, APIs, Harness Behaviors

Where the body says *the API returns* or *the harness allows* or *the SDK accepts*, this section names the actual field, method, limit, and quirk in play at the time of writing.

## What lives here

- Per-provider SDK call shapes — initial request, response parsing, streaming, structured output.
- API field names — `stop_reason`, `usage`, cache-related counters, model-routing fields, rate-limit headers.
- Harness behaviors and limits — delegation depth, message-order timing, tool auto-resolution windows, MCP-server lifecycle.
- Per-harness affordances — config flags, skill systems, memory layers, gate mechanics.

## What doesn't live here

- Marketing comparisons across providers. The appendix is descriptive, not evaluative.
- Tutorials for getting started with a given SDK — providers' own docs cover that better than a book appendix can.
- Recipes that compose multiple SDK affordances into a working pattern — those live in `C-recipes/`.

## Organization

One entry per topic per provider (or per harness, for harness-specific behaviors). The naming convention is `<topic>-<provider>.md` when provider-specific, `<topic>.md` when stack-agnostic.

## Entries

- [`anthropic-prompt-caching.md`](anthropic-prompt-caching.md) — from Ch. 3 (`cache_creation_input_tokens`, `cache_read_input_tokens`, `cache_control: { type: "ephemeral" }`, the cache-floor minimum).
- [`harness-limits.md`](harness-limits.md) — from Ch. 4 and Ch. 7 (MCP server lifecycle, AskUserQuestion 89ms auto-resolution, sub-agent delegation depth = 1).
- [`harness-affordances.md`](harness-affordances.md) — from Ch. 9 and Ch. 11 (`disable-model-invocation` per-skill flag, auto-memory layer, consent gate mechanics).
