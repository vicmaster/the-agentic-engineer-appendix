# Anthropic Prompt Caching — Field Shapes, Floors, Gotchas

**Source aside:** Ch. 3 (the cache that wasn't a cache).
**Source incident:** `war-stories/forge-prompt-cache-minimum.md`.
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026, against the Anthropic Messages API (`/v1/messages`).

## What the body says

Ch. 3's opening story turns on a prompt-cache block that was below the provider's minimum size, so the `cache_control` directive was silently a no-op. The body names the failure shape (*infrastructure-claim test*) without locking in field names. This entry has the field names, the per-model floors, the cost shape, and the gotchas that bite teams who skip the verification half.

## What prompt caching is

Anthropic's prompt caching stores contiguous *prefix* segments of the request on the provider's side, billed at a discounted input rate when reused on subsequent calls within a TTL. The cache key is derived from the exact bytes of the cached blocks plus the model identifier; **any change to the prefix invalidates the cache**. The discount only applies to the cached prefix; tokens after the last cached block are billed at the regular input rate.

## The dated specifics

### Cache-control declaration

A block becomes cacheable by attaching `cache_control` to its end:

```json
{
  "type": "text",
  "text": "<contents>",
  "cache_control": { "type": "ephemeral" }
}
```

The marker applies to *everything up to and including this block*. Subsequent blocks without a `cache_control` marker are not part of the cached prefix.

You can mark up to **four** `cache_control` blocks per request. Each marker creates a cache breakpoint; the most-recently-hit breakpoint is what gets read on a cache hit.

### Response usage fields

The `usage` block on every Messages API response reports the cache state for that specific call:

```json
{
  "usage": {
    "input_tokens": 42,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 1100,
    "output_tokens": 200
  }
}
```

- `cache_creation_input_tokens` — non-zero when *this* call seeded the cache. Billed at ~1.25× the normal input rate (the seeding premium).
- `cache_read_input_tokens` — non-zero when *this* call read from cache. Billed at ~0.1× the normal input rate (the cached discount).
- `input_tokens` — the *uncached* tokens (everything after the last cache breakpoint). Billed at the normal input rate.

The relationship: `total_input_billed ≈ cache_creation_input_tokens × 1.25 + cache_read_input_tokens × 0.1 + input_tokens × 1.0` (multipliers as of Q2 2026; check the pricing page for current numbers).

### Cache floor per model

`cache_control` is silently ignored below per-model minimums. The call still succeeds, both counters return zero, no warning, no header — see the *cache that wasn't* story.

| Model | Minimum cacheable block | Notes |
|---|---|---|
| Sonnet 4.6 (`claude-sonnet-4-6`) | ~1,024 input tokens | The Forge story's anchor model. |
| Haiku 4.5 (`claude-haiku-4-5`) | ~4,096 input tokens | Highest floor in the lineup. Easy to miss. |
| Opus 4.8 / Opus 5 (`claude-opus-4-8` / `claude-opus-5`) | ~1,024 / ~512 input tokens | Frontier Opus-tier as of this review; the 5-family lowered the floor to ~512. |

The floors above are *Anthropic-documented minimums*. Below the floor the directive is a silent no-op. Just above the floor, cache behavior can be unreliable on the seeding turn — small blocks at the edge of the floor may or may not seed depending on the exact token count after tokenization. The conservative discipline: aim for **2× the floor** on every cacheable block, not 1×.

### Cache TTL

The default ephemeral TTL is **5 minutes** as of Q2 2026. The TTL refreshes on each cache hit, so frequently-accessed prefixes effectively don't expire while traffic flows. Idle prefixes expire and have to be re-seeded.

A longer TTL (1 hour) is available as `cache_control: { type: "ephemeral", ttl: "1h" }`. The 1-hour TTL carries a higher seeding cost (~2× the normal input rate); it pays back when the prefix gets reused enough times to amortize the seeding premium across the longer window.

### Char-to-token heuristic gotcha

The Forge codebase uses `CACHE_FLOOR_CHARS = 4096` based on a 4-chars-per-token heuristic. From the war story:

> *KPI tables tokenize denser than 4 chars/token, verbose Spanish prose tokenizes sparser, and the 4096-char floor is right on average and wrong at the edges.*

The tokenizer-true fix is a counter API call (`POST /v1/messages/count_tokens`) which returns the exact token count for a given content block. Use it when:

- The cacheable block is within 30% of the floor.
- The content shape is denser than English prose (code, JSON, tables, dense markup).
- The block contains content in a non-English language.

For typical English prose well above the floor, the 4-chars-per-token heuristic is fine.

## Real-implementation snippets

> The shapes below are dated. As of writing 2026, the official SDKs are `@anthropic-ai/sdk` (TypeScript/Node) and `anthropic` (Python). For Ruby, the Forge codebase uses the `anthropic-rb` community gem (also documented at the same Anthropic API endpoints).

### TypeScript

```ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const response = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  system: [
    {
      type: "text",
      text: "<long, mostly-stable system prompt — well above 1024 tokens>",
      cache_control: { type: "ephemeral" },
    },
  ],
  messages: [
    { role: "user", content: "What's the latest on Q3 KPIs?" },
  ],
});

const { cache_creation_input_tokens, cache_read_input_tokens, input_tokens } =
  response.usage;

// At this point, classify the call into one of the four buckets
// (see C-recipes/observability-across-stacks.md for the full pattern).
```

### Ruby

```ruby
# As of writing — anthropic-rb gem, snake_case throughout.
response = anthropic_client.messages.create(
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  system: [
    {
      type: "text",
      text: bsc_context_block,  # well above 1024 tokens
      cache_control: { type: "ephemeral" },
    },
  ],
  messages: [
    { role: "user", content: question },
  ],
)

usage = response.usage
# usage.cache_creation_input_tokens
# usage.cache_read_input_tokens
# usage.input_tokens
```

> The shape is durable. The implementation is a snapshot. The SDK call surfaces above will change; the field names and the four-bucket classification will not.

## Gotchas

### 1. The prefix is byte-exact

The cache key is derived from the exact serialized bytes of the cached blocks plus the model identifier. Any change — a re-ordered tool list, a swapped-in current date, an updated brand string — busts the cache.

**Symptom:** Cache hit rate inexplicably zero after a code change that "shouldn't have affected caching."
**Fix:** Order content from *most stable* (rarely changes) to *most volatile* (changes per call). Put the cache breakpoint at the boundary. Volatile content goes *after* the breakpoint, where it's billed at the normal input rate but doesn't bust the cache.

### 2. Block ordering matters

The cache covers a contiguous prefix. Inserting a new block in the middle of an already-cached prefix invalidates everything from the insertion point onward.

**Symptom:** A new feature shipped, cache hit rate dropped on the seeding turn.
**Fix:** Treat the cached prefix as an *append-only log*. New stable content goes after existing stable content (still inside the cached region); new volatile content goes after the cache breakpoint.

### 3. Tool definitions count

If `tools` is part of the request, the tool list is part of the prefix. Re-ordering tools, adding a tool, or changing a tool's description busts the cache.

**Symptom:** Caching works in dev, fails in prod after deploy.
**Fix:** Sort the tool list deterministically. Version the tool definitions so changes are explicit, not accidental.

### 4. Model version changes

Each model version has its own cache namespace. Switching from `claude-sonnet-4-6` to `claude-sonnet-5` requires re-seeding every cached prefix. The transition is silent — the call succeeds, the seeding turn just costs the seeding premium again.

**Symptom:** Cost spikes on the day of a model upgrade.
**Fix:** Plan the model upgrade as a re-seeding event. Run the seeding turns deliberately during low-traffic windows rather than letting them happen organically during peak load.

### 5. Streaming and caching coexist

Streaming responses (`stream: true`) participate in caching identically to non-streaming. The `usage` fields appear on the final `message_delta` event, not on intermediate `content_block_delta` events. Code that only inspects intermediate events misses the cache verification.

**Symptom:** Streaming code doesn't bucket calls into the four cache states; the verification half ships incomplete.
**Fix:** Inspect the `usage` block on the terminal `message_stop` event (or equivalent in your SDK's streaming abstraction).

### 6. Cache-aware retries

If you retry a failed call, the cache might be cold (TTL expired) or warm depending on timing. The retry's `cache_read_input_tokens` is informative: zero means the retry was a cold seed (paying the seeding premium); non-zero means the retry hit the cache. Surface this in telemetry so retries don't quietly become repeated seeding events.

## Common confusions

**"My cache hit rate is X% — is that good?"** Cache hit rate alone is a misleading metric. The right metric pair is *(hit rate) + (cached tokens per call)*. A 90% hit rate on 50 cached tokens is worse than a 50% hit rate on 5,000 cached tokens. Optimize for cached input volume, not hit count.

**"Should I cache the user's message?"** Almost never. User messages are the most volatile part of a conversation; caching them would mean a fresh seed on every turn. Cache the system prompt and tool definitions, where the content is mostly stable across calls.

**"Why is my cache_creation_input_tokens non-zero on every turn?"** You're busting the cache. Some part of your prefix is changing call-to-call. The forensics: print the cache-prefix bytes, diff between consecutive turns, find the volatile field.

**"My block is above the floor but both counters return zero."** This is the *cache that wasn't* failure shape from Ch. 3. Possible causes: (a) block is above the documented floor but below the model's actual operational floor (the floor is approximate), (b) the cache TTL expired between calls, (c) you're hitting a different region or load-balanced shard with a cold cache. Mitigation: aim for 2× the floor and instrument the four buckets so the `:none` case is visible.

## The pseudocode the body uses

The deep bucketing pattern lives in [`../C-recipes/observability-across-stacks.md`](../C-recipes/observability-across-stacks.md) (Implementation 1) and [`../C-recipes/runtime-trust-patterns.md`](../C-recipes/runtime-trust-patterns.md) (Layer 3). Re-stated here for completeness:

```
function classifyCacheCall(response, cacheableBlocks):
    creation = response.usage.cacheCreationTokens
    read     = response.usage.cacheReadTokens
    blockSize = sum(sizeOf(b) for b in cacheableBlocks)

    if read > 0:                    return CACHED
    if creation > 0:                return SEEDED
    if blockSize < CACHE_FLOOR:     return BELOW_FLOOR
    return NONE   # above floor, both counters zero — the failure shape
```

The pseudocode uses camelCase (`cacheCreationTokens`, `cacheReadTokens`) so it isn't locked to a single SDK's snake_case. The current snake_case fields are listed in the **Response usage fields** section above.

## What survives the SDK changing

The infrastructure-claim test from Ch. 3: *for every provider feature you've turned on, ask what the response looks like when the feature silently doesn't apply.* The field names, the floors, the TTL, the cost multipliers — all rotate. The four-bucket classification (`CACHED`, `SEEDED`, `BELOW_FLOOR`, `NONE`) and the gotchas (byte-exact prefix, block ordering, tool list, model version) are the durable layer.

## Cross-references

- Ch. 3 (Context as a Resource) — full chapter treatment.
- Ch. 8 (Observability) — the bucketing pattern recast as a verification record (`provider.cache.bucket` metric).
- `war-stories/forge-prompt-cache-minimum.md` — the source incident.
- [`../C-recipes/observability-across-stacks.md`](../C-recipes/observability-across-stacks.md) — Implementation 1 (the Ruby `compute_cache_status` method, the `:cached`/`:seeded`/`:below_floor`/`:none` buckets).
- [`../C-recipes/runtime-trust-patterns.md`](../C-recipes/runtime-trust-patterns.md) — Layer 3 (telemetry verification as the runtime trust signal).
- [`../A-models/frontier-models-2026.md`](../A-models/frontier-models-2026.md) — the model lineup the floors are pinned to.
