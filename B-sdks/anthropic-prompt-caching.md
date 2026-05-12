# Anthropic Prompt Caching — Field Shapes and Floor

**Source aside:** Ch. 3 (the cache that wasn't a cache).
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026.

## What the body says

Ch. 3's opening story turns on a prompt-cache block that was below the provider's minimum size, so the directive was silently a no-op. The body names the failure shape (*infrastructure-claim test*) without locking in field names; this entry has the field names.

## The dated specifics

As of Q2 2026, the relevant Anthropic SDK shapes are:

- **Cache-control declaration on a request block:**

  ```
  cache_control: { type: "ephemeral" }
  ```

- **Response usage fields** that report whether the cache actually applied:

  - `cache_creation_input_tokens` — non-zero when *this* call seeded the cache.
  - `cache_read_input_tokens` — non-zero when *this* call read from cache.

- **Cache floor:** the minimum size of a cacheable block was around 1,024 input tokens. Below that, the `cache_control` directive is a silent no-op — the call succeeds, both counters come back zero, no warning, no header.

**TODO (subsequent pass):** confirm the exact 2026 floor per Anthropic's then-current docs, note any per-model variation (Sonnet vs. Haiku vs. Opus), and capture the SDK version this entry was last verified against.

## The pseudocode the body uses

From Ch. 3:

```
function classifyCacheCall(response, cacheableBlocks):
    creation = response.usage.cacheCreationTokens
    read = response.usage.cacheReadTokens
    blockSize = sum(sizeOf(b) for b in cacheableBlocks)

    if read > 0:
        return CACHED
    if creation > 0:
        return SEEDED
    if blockSize < CACHE_FLOOR:
        return BELOW_FLOOR
    return NONE
```

The pseudocode uses camelCase (`cacheCreationTokens`, `cacheReadTokens`) so it isn't locked to a single SDK's snake_case. The current snake_case fields are the ones listed above.

## What survives the SDK changing

The infrastructure-claim test: *for every provider feature you've turned on, ask what the response looks like when the feature silently doesn't apply.* The names will rotate; the test stays.

## Cross-references

- Ch. 3 (Context as a Resource) — full chapter treatment.
- Ch. 8 (Observability) — the bucketing pattern recast as a verification record (`provider.cache.bucket` metric).
- [`C-recipes/`](../C-recipes/) — cache-bucket classifier per stack.
