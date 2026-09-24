# Observability — Verification Records Across Stacks

**Source aside:** Ch. 8 (observability for probabilistic systems).
**Source incidents:** `war-stories/forge-prompt-cache-minimum.md`, `forge-layered-status.md`, `coide-memory-drift.md`, `leads-crm-mcp-second-surface.md`.
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026, against Forge (Rails 8 / Postgres / Sidekiq), leads-crm (Rails 8 / Postgres / Hotwire), coide (Electron / TypeScript wrapping an agent CLI).

## The body principle

Ch. 8, Principle 8: *log the claim, not the call.* In probabilistic systems, the runtime can't raise on most failures, so observability has to be constructed at the seams where claims could quietly stop being true. For every contractual claim your code is making — *this layer succeeded, this provider feature applied, this stored fact still matches reality* — log the verification, not just the call's return value. Alert on the verification breaking, not on the call failing.

## The recipe at one glance

Four implementations of *log the claim* across two stacks, each born from a real incident. Together they form a usable observability discipline for a probabilistic system: every claim your code makes about an invisible layer has a *verification step*, the verification's output is a *structured record*, and the alert fires when the record says the claim stopped holding.

This is the operational complement to Ch. 5's *audit is the eval*. The audit catches bad output at the call site. Observability catches bad output (and bad infrastructure-acceptance, bad layer-transitions, bad stored facts) across many calls, over time, where no single call's failure looks loud enough to notice.

## The durable shape (pseudocode)

The pattern, language-agnostic, in four moving parts.

```
# Pseudocode: the verification-record contract

# 1. The Claim — what your code is asserting about an invisible layer
# Examples:
#   "this cacheable block was actually cached by the provider"
#   "this routine's last_run_status reflects the execution layer"
#   "this stored fact about the codebase is still true"
#   "this MCP tool call reached the underlying API"

# 2. The Verification — a check that, when run, says whether the claim holds
function verify(claim):
    actualLayer  = readActualLayerState(claim)
    expected     = claim.assertion
    return VerificationRecord(
        claim=claim.name,
        held=(actualLayer matches expected),
        observed=actualLayer,
        expected=expected,
        timestamp=now(),
        tags=claim.tags,
    )

# 3. The Record — emitted on every claim-bearing call, structured for aggregation
function emitVerification(record):
    metricBackend.emit(
        name="claim.verified",
        value=1 if record.held else 0,
        tags={
            "claim": record.claim,
            "observed": record.observed,
            **record.tags,
        },
    )
    if not record.held:
        logger.warn("Claim broken", record=record)

# 4. The Alert — fires on a state the system should have left, not on what it entered
function expectedEventAlert(claim, window, expectedCount):
    actualCount = metricBackend.query(
        name="claim.verified",
        tags={"claim": claim.name, "held": True},
        window=window,
    )
    if actualCount < expectedCount:
        alert("Claim '{claim.name}' has not held {expectedCount} times in {window}")
```

Four moving parts. The thing most teams skip is part 4 — the expected-event alert. Standard alerting fires on bad events; probabilistic-system alerting fires on the *absence* of good events. Different shape; different design discipline.

## Four implementations

Each subsection shows one claim, the verification step, the structured record, and (where relevant) the alert that would catch the failure mode.

### Implementation 1 — Cache-bucket classifier (Rails, Forge)

**Claim:** *every cacheable block we send is actually being cached by the provider.*

**Source incident:** `forge-prompt-cache-minimum.md`. Wiring landed in commit `d6e9076` (2026-04-15). For a week, the wiring claimed caching was on; the provider was silently declining because blocks were below the floor. Verification arrived a week later in commit `4204787` (2026-04-22).

> **Real implementation, as of writing 2026** — appendix B has the current Anthropic SDK shape; see [`../B-sdks/anthropic-prompt-caching.md`](../B-sdks/anthropic-prompt-caching.md) for the field names. Date context for the snapshot below: Forge `data_source_chat.rb` at commit `4204787`, Ruby 3.x, Rails 8.

The bucketing classifier:

```ruby
# app/services/virtual_employee/capabilities/data_source_chat.rb

CACHE_FLOOR_CHARS = 4096 # ~1024 tokens via 4-chars-per-token heuristic

# Bucket every cacheable call into one of four states.
# Surfaces as result_meta[:cache_status] on every turn.
def compute_cache_status(result_meta, cacheable_blocks)
  creation_tokens = result_meta[:cache_creation_tokens].to_i
  read_tokens     = result_meta[:cache_read_tokens].to_i
  block_chars     = cacheable_blocks.sum { |b| b.to_s.bytesize }

  return :cached      if read_tokens.positive?
  return :seeded      if creation_tokens.positive?
  return :below_floor if block_chars < CACHE_FLOOR_CHARS
  :none # above floor, both counters zero — the failure shape
end
```

The `ClaudeClient` populates `result_meta` from the response usage block:

```ruby
# app/services/virtual_employee/llm/claude_client.rb

def call(messages:, system:, cacheable_blocks: [])
  response = anthropic_client.messages.create(
    model: model_name,
    system: build_system(system, cacheable_blocks),
    messages: messages,
  )

  result_meta = {
    cache_creation_tokens: response.usage.cache_creation_input_tokens,
    cache_read_tokens:     response.usage.cache_read_input_tokens,
    # ... other usage fields
  }
  [response.content, result_meta]
end
```

The capability emits the bucket on every call. Today it lands in `result_meta[:cache_status]`; the structural fix (still queued) is to emit a metric on `provider.cache.bucket` so the failure shape becomes alertable:

```ruby
# Pseudocode for the metric emission that closes the loop:
ActiveSupport::Notifications.instrument(
  "provider.cache.classified",
  capability: capability_name,
  bucket:     compute_cache_status(result_meta, cacheable_blocks),
  block_chars: cacheable_blocks.sum(&:bytesize),
)
```

**The alert that catches the failure shape:**

> *Cache-hit count for `bsc_analyst` has been zero for ninety minutes during business hours, despite at least N turns having fired.*

Not "the cache call errored" — those aren't happening. The signal is *the `:cached` bucket has not appeared when it should have*.

**What survives the SDK changing:** the four-bucket classification (`:cached`, `:seeded`, `:below_floor`, `:none`) and the *bucket-the-call-into-states-and-emit* discipline. The exact field names will rotate; the buckets are the durable layer.

### Implementation 2 — Layered-status writeback (Rails, Forge)

**Claim:** *the column the operator reads as the routine's success status actually reflects the execution layer, not the routing layer.*

**Source incident:** `forge-layered-status.md`. Two writeback PRs in one week (`aea302b` then PR #42), both fixing `last_run_status`, both landing on the wrong terminal value before someone noticed *the column is supposed to mean execution-layer success, not routing-layer success.*

The bug shape:

```ruby
# WRONG — this is what shipped in aea302b
class WritebackService
  def call(routine)
    routing_status, _result = TaskRouter.call(routine.task_spec)
    routine.update!(last_run_status: routing_status)
    # routing_status is :queued for happy path — the column is permanently :queued
  end
end
```

The bug isn't `TaskRouter`'s return value — `:queued` is the *correct* answer to "did routing succeed?" The bug is conflating two layers' status into one column.

The pragmatic fix in PR #42:

```ruby
# app/services/forge/writeback_service.rb
class WritebackService
  ROUTING_TO_EXECUTION = {
    queued:    :succeeded, # optimistic — Task row records any later failure
    failed:    :failed,
    deduped:   :skipped,
  }.freeze

  def call(routine)
    routing_status, task = TaskRouter.call(routine.task_spec)
    execution_status = ROUTING_TO_EXECUTION.fetch(routing_status, :unknown)

    routine.update!(
      last_run_status: execution_status,
      last_run_task_id: task&.id,
    )
  end
end
```

The structural fix (not yet landed, named in the war story) would be a post-execution callback that updates the routine after the `Task` row transitions, rather than an optimistic mapping. The pragmatic mapping accepts that the unhappy path requires looking at the `Task` row.

The *observability* angle: the writeback should emit a verification record on every fire, naming which layer's status it just persisted, so the operator-visible column and the underlying truth can be cross-checked:

```ruby
ActiveSupport::Notifications.instrument(
  "routine.writeback.completed",
  routine_id:      routine.id,
  routing_status:  routing_status,
  execution_status: execution_status,
  task_id:         task&.id,
)
```

**The alert that catches the failure shape:**

> *Routine `Y`'s `last_run_status` has been `:queued` for more than ten minutes after the scheduled fire time.* (Stuck-state alert — the next state never came.)

The right shape: alert on a state the system should have left, not on a state the system shouldn't have entered. There's no exception, no error rate, no latency spike — the column was *set*. The signal is *the transition didn't happen.*

**What survives the data model changing:** the discipline of naming each layer's status separately and the heuristic *any persisted status field that crosses an async boundary is a candidate for this bug*. Whether the layers live in two columns, two rows, or two services is implementation detail; the layering is the principle.

### Implementation 3 — Memory-verification habit (TypeScript, coide)

**Claim:** *the architectural fact the agent's memory recalls is still true.*

**Source incident:** `coide-memory-drift.md`. The agent's persistent memory carried *coide spawns the CLI subprocess via `node-pty`*. Correct when written. The architecture later moved to `child_process.spawn` with stream-json input. The memory didn't update. The next session that recalled the memory would have produced confidently wrong architectural advice if the verification habit hadn't kicked in.

The observability angle is different from the first two implementations. There isn't a synchronous call site to instrument. The "call" is *the agent retrieves a memory and reasons about it*, and the gap between the memory and the world is the time axis itself.

What you instrument instead: the *verification* step that follows the recall.

```ts
// Pseudocode — memory verification as an observable event

interface MemoryRecall {
  slug: string;
  name: string;
  description: string;
  body: string;
  metadata: { type: string };
}

interface Verification {
  memorySlug: string;
  namedEntity: string; // a file, function, flag named in the memory
  observedExists: boolean;
  observedMatches: boolean;
  timestamp: string;
}

async function verifyMemory(recall: MemoryRecall): Promise<Verification> {
  const namedEntity = extractNamedEntity(recall.body);
  const observedExists = await fileOrSymbolExists(namedEntity);
  const observedMatches = observedExists
    ? await currentStateMatches(namedEntity, recall.body)
    : false;

  const verification = {
    memorySlug: recall.slug,
    namedEntity,
    observedExists,
    observedMatches,
    timestamp: new Date().toISOString(),
  };

  telemetry.emit("memory.verified", verification);

  if (!observedMatches) {
    telemetry.emit("memory.stale", { ...verification, body: recall.body });
  }

  return verification;
}
```

The corpus this builds: every time the agent acts on a recalled memory, a verification record names whether the recall still matches the world. Aggregating those records tells you *which memories drift fastest* and *which surfaces need a refresh discipline.*

**The alert that catches the failure shape:**

> *Memory `architecture-coide-cli-spawn` has been observed as stale 3 times in the last 24 hours.*

A high stale-rate on a specific memory is a signal to either refresh the memory or remove it; the agent is acting on a fact the codebase has moved past.

**What survives the harness's memory layer changing:** the discipline *recall is a starting point, not an answer.* Verification is what turns recall into truth, and the verification is the observable event. Whether the memory layer is YAML-on-disk, a vector store, or an in-process cache is implementation detail.

### Implementation 4 — Per-tool-surface metrics (Rails, leads-crm)

**Claim:** *the MCP layer in front of the API is healthy as a distinct surface.*

**Source incident:** `leads-crm-mcp-second-surface.md`. A 1-line HTTPS commit (`77a026b`) took five weeks to surface in production. The MCP layer had been built and deployed for two days before the SSL redirect bug bit; CI passed because request specs hit the in-process Rails app, not the deployed remote-MCP path.

The mental-model flip: **the MCP layer is not a glue layer; it is a second product surface with its own auth, transport, contract, and observability needs.** The API's metrics tell you the API is healthy. They don't tell you the MCP layer is healthy.

The four observability axes the surface needs:

```ruby
# app/middleware/mcp_telemetry.rb
class McpTelemetry
  def initialize(app)
    @app = app
  end

  def call(env)
    started = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    tool_name = env["mcp.tool_name"]

    status, headers, body = @app.call(env)

    duration_ms = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - started) * 1000).round

    ActiveSupport::Notifications.instrument(
      "mcp.tool_call.completed",
      tool_name:    tool_name,
      status:       status,
      duration_ms:  duration_ms,
      transport:    env["mcp.transport"], # stdio | http | https
      auth_token_id: env["mcp.auth_token_id"],
    )

    [status, headers, body]
  end
end
```

Four axes the instrumentation surfaces:

1. **Per-tool call rates and error rates.** Like API endpoints, but for the tool surface. Which tools are getting called, how often, at what failure rate.
2. **Transport-level signals.** If the surface goes over HTTPS, watch the HTTPS. The leads-crm bug was a 301-redirect that showed up as "tool call failed" because nobody was watching transport.
3. **Auth-flow metrics.** Token issuance, expiration, rotation. Agent auth has different cadence than human auth; instrument them separately.
4. **End-to-end specs that exercise the deployed transport.** Not just the in-process handler. In-process tests pass with `http://`; the deployed path forces HTTPS.

**The alert that catches the failure shape:**

> *3xx-rate on `mcp.tool_call.completed` for tool `search_leads` exceeded 5% over the last 15 minutes.*

A redirect-rate alert wouldn't fire on the underlying Rails API's dashboard because the API itself was returning 200s once the redirect resolved; the redirect was happening *between* the MCP handler and the API. The MCP surface's own metrics are what catch the failure.

**What survives the protocol changing:** the *tool surfaces are second products* framing. Every agent-callable interface (MCP, REST-for-agents, function-calling, OpenAPI specs for an agent harness) needs its own observability budget. The specific middleware shape will change; the discipline of treating the surface as a first-class product doesn't.

## Expected-event alerting (the negative-space pattern)

The four implementations share one alerting move that's worth pulling out separately: **alert on a state the system should have left and didn't, not on a state the system shouldn't have entered.**

Three concrete examples, restated from Ch. 8:

- **Cache-hit count = 0 for capability X over ninety minutes.** Not "the cache errored" — those aren't happening. The signal is the absence of expected cache hits.
- **Routine Y's `last_run_status` has been `:queued` for more than ten minutes after scheduled fire time.** Not "the routine errored" — the column was set. The signal is the transition that never came.
- **Audit finding Z has been open for more than three iteration loops without a re-render attempt.** Not "the audit failed" — the audit ran. The signal is the agent didn't act on the feedback.

The pattern across all three: define what success looks like as a *sequence* — a counter advancing, a state transitioning, a finding resolving — and alert when the sequence stalls.

A pragmatic starting point: pick the three claims your system depends on most, write one expected-event alert for each, ship them, expand from there.

## Porting to other stacks

The four moving parts (Claim, Verification, Record, Alert) port unchanged.

**Node / TypeScript.** Replace `ActiveSupport::Notifications.instrument` with whatever observability SDK is in play (OpenTelemetry, Pino structured logs, custom metric backend). The shape — emit a structured event keyed by claim name — survives the choice. The memory-verification implementation above is already TypeScript; the cache-bucket and layered-status patterns port to TS by replacing the Ruby blocks with `async` functions and the metric calls with the equivalent SDK.

**Python.** `logging.structured`, OpenTelemetry's Python SDK, or a custom emitter. Same shape: one event per claim, with the claim name, the observed value, the expected value, and the tags. Pydantic dataclasses for the record types.

**Go.** A small `claim` package exposing `Verify(ctx, claim)` and emitting through `slog` or whatever structured logger is in use. Channel-based metric emission for hot paths.

**Any stack.** The non-negotiables are: (1) the claim's name is part of the record so you can aggregate by it, (2) the verification's `held` boolean is the alertable field, (3) the record is structured (not free-text log lines), and (4) at least one alert fires on the absence of expected `held: true` events over a window — not just on `held: false` spikes.

## Counter-examples — when this pattern doesn't apply

- **Truly synchronous code with one layer.** A function that does its work in-band and returns when done has nothing to verify across layers. The return value is the answer; don't manufacture a verification record.
- **Throwaway prototypes.** A scratch script exploring what an agent can do doesn't need expected-event alerting. The verification is the human reading the output.
- **Internal tooling with a small, attentive audience.** Some agentic systems are used by three people who all use them every day and would notice within an hour if anything were off. The cost of a verification gap is small because the audience is the verification.
- **Cases where the cost of a wrong claim is genuinely zero.** A back-of-house log entry no one reads. A staging-only artifact. The verification investment has to earn its keep against the cost of the claim being wrong.

The signal: *if this claim quietly stopped being true, how would I find out?* If the answer is *a human notices in real time*, the framework is overkill. If the answer is *eventually, when something visible breaks*, the framework is the gap between failure and discovery — and closing that gap is what observability is for.

## What survives the metric stack changing

The principle: *log the claim, not just the call; alert on what didn't happen, not just on what did.* The metric backend will rotate underneath (Prometheus → Datadog → Honeycomb → something else); the four implementations' specifics will rotate (Ruby on Rails 8 → 9 → next-thing); the harness affordances will rotate. The four moving parts and the four named implementations as worked examples are the durable layer.

## Cross-references

- Ch. 8 (Observability for Probabilistic Systems) — full chapter treatment.
- Ch. 3 (Context as a Budget) — origin of the cache-bucket classifier shape.
- Ch. 5 (Verification in the Production Loop) — the audit pipeline as the runtime cousin of observability; see [`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md) for the artifact-side recipe.
- Ch. 6 (Failure Modes Catalog) — the layering shape that produces the *function returned ≠ work completed* family of bugs all four implementations defend against.
- `war-stories/forge-prompt-cache-minimum.md`, `forge-layered-status.md`, `coide-memory-drift.md`, `leads-crm-mcp-second-surface.md` — the four source incidents.
- [`../B-sdks/anthropic-prompt-caching.md`](../B-sdks/anthropic-prompt-caching.md) — the SDK fields the cache-bucket classifier reads.
- [`runtime-trust-patterns.md`](runtime-trust-patterns.md) — when the verification record *is* the trust signal at runtime.
