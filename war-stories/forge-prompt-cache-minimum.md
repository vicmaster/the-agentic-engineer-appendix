# forge / The Cache That Wasn't

## Date / Version Context

- **Wiring date:** 2026-04-15, commit `d6e9076` (T030–T042 of `001-forge-mvp`). `ClaudeClient#build_system` learned to wrap `cacheable_blocks` with `cache_control: { type: "ephemeral" }`; `build_meta` learned to surface `cache_creation_input_tokens` and `cache_read_input_tokens` from the response usage block (`app/services/virtual_employee/llm/claude_client.rb:48-49`, `:79-80`).
- **First operator-readable status layer:** 2026-04-22, commit `4204787` ("`feat(003-data-source-chat): capability — DataSourceChat + prompt v1 + view + initializer`"). Introduced `CACHE_FLOOR_CHARS = 4096` and `compute_cache_status` in `data_source_chat.rb:17`, `:206-214`.
- **Repo state:** pre-launch. Production didn't go live on Heroku until 2026-04-29 — the entire wire-it/watch-zeros/verify arc happened in pre-prod. Leadership had not yet seen any briefs.
- **Capabilities that used `cacheable_blocks:` first:** `bsc_analyst` (`bsc_analyst.rb:35`) and `meeting_prep` (`meeting_prep.rb:46`), both leadership-brief capabilities that pack a presenter-rendered context block above the user message. `data_source_chat` joined a week later and is the chat-shape capability that drove the verification work — turn 2+ in the same thread is the strongest cache-hit case.
- **Glossary, used in this writeup:** *Prompt caching* = an Anthropic API feature where content blocks marked with `cache_control: { type: "ephemeral" }` are stored on the provider's side and reused on subsequent calls within the cache TTL, billed at a discounted rate. *Cache seed* = the first call where the provider stores the block (`cache_creation_input_tokens > 0`). *Cache hit* = a subsequent call within the TTL that reuses a stored block (`cache_read_input_tokens > 0`). *Cache floor* = the minimum block size below which `cache_control` is silently a no-op, ~1024 input tokens for Sonnet 4.6 / Haiku 4.5 at the time, encoded in Forge as `CACHE_FLOOR_CHARS = 4096` via a 4-chars-per-token heuristic. *Silent no-op* = the failure shape where the call succeeds, the response shape is identical to a successful cache call, but neither cache counter advances.

## What Was Being Attempted

Turn on prompt caching for capabilities that pack the same large data block into the prompt across runs.

The `bsc_analyst` capability serializes the BSC context — perspectives, KPIs, snapshots — into the system prompt every fire. The `meeting_prep` capability does the same with meeting context. Weekly cadence, mostly-unchanged underlying data — exactly the shape `cache_control: { type: "ephemeral" }` is for. For `data_source_chat`, the case is sharper: chat shape means turn 2 in the same thread is paying for the same data block as turn 1, every time, forever, unless the cache is doing what the docs claim.

The 003 spec encoded this as a US2 acceptance criterion in spec language: "the platform demonstrates [caching] by recording a non-zero cache-read count on the task execution" (`specs/003-data-source-chat/spec.md:26-32`). The cost-savings claim was the spec, not just an aspiration.

The wiring was easy. `cache_control: { type: "ephemeral" }` is two tokens of Ruby. The `ClaudeClient` code that does it is six lines. The capability call sites that pass `cacheable_blocks:` are one line each. The whole feature reads like a config flip.

## What Went Wrong

The cache silently didn't cache.

`cache_control: { type: "ephemeral" }` has a documented minimum block size — about 1024 input tokens for Sonnet 4.6 and Haiku 4.5 at the time. Below that floor, the API still accepts the directive. The call still succeeds. The response still comes back. `stop_reason` is `:end_turn`, `usage` populates, the conversation works exactly as if caching were working.

What's different — the only thing that's different — is that `cache_creation_input_tokens` and `cache_read_input_tokens` both come back zero. No warning. No header. No annotation in the usage block. No error returned by the SDK. The block was not cached, and the only signal is two counters in `usage` that have to be checked against an expectation that nothing in the system was holding.

For a small fixture sheet or a stub data source, the cacheable block is sub-floor. The seeding turn returns zero on `cache_creation_input_tokens`, the next turn has nothing to read, and the operator sees a successful call where caching plainly isn't working — without anything in the system saying so.

The bug isn't the floor. The floor is documented; the number isn't a secret. The bug is that the floor is enforced *silently inside a successful response shape*, which is the same failure pattern as `forge-max-tokens.md` (`stop_reason: "max_tokens"` swallowed into valid-looking output) and `forge-layered-status.md` (`TaskRouter`'s `:queued` swallowed into a status column expecting execution-layer success). Three flavors of *the system has more layers than the operator can see, and the layers don't raise*.

The first version of Forge's caching code shipped the wiring and not the verification. The wiring half claimed "caching is on." The verification half — the layer that would tell an operator whether caching was *actually working* — didn't exist for a week.

## How It Was Discovered

The artifact-supported reading: the verification was added because it was needed.

The `compute_cache_status` method (`data_source_chat.rb:206-214`) buckets every turn into one of four states — `:cached`, `:seeded`, `:below_floor`, `:none` — and writes the bucket into `result_meta[:cache_status]`. The corresponding spec at `spec/services/virtual_employee/capabilities/data_source_chat_caching_spec.rb:131-176` exercises all four states. A comment in the same spec at lines 60–62 reads "*A large-enough parse result to push the serialized block above the ~4096-char cache floor*." That comment exists because the *initial* fixture wasn't large enough, and the assertion failed.

You don't write code that buckets `:none` (above-floor blocks returning zero counters) separately from `:below_floor` (sub-floor blocks correctly returning zero counters) unless someone has chased the difference. You don't pre-emptively size a test fixture against a cache-floor heuristic unless the previous fixture broke against that heuristic. You don't add a comment explaining the size of a fixture unless the size is load-bearing.

The 003 spec is even more explicit. Edge case enumeration at `spec.md:78` lists "Data source serialization below cache floor" with the rationale "follow-up debugging is not a mystery." Research note R4 reads "operators do not chase a bug that is actually just a small dataset." Both lines are reactive — they pre-empt confusion the author has already lived through.

The most likely discovery channel: a sanity check, in dev or against a small fixture, returned `cache_creation_input_tokens: 0` and `cache_read_input_tokens: 0` on what was supposed to be the seeding turn of a chat-shape capability. The expectation was "turn 1 seeds, turn 2 hits." The reality was "turn 1 silently doesn't seed because the block is sub-floor; turn 2 has nothing to read; both counters are zero on both turns, and the system has no opinion about whether that's correct or not." Writing `compute_cache_status` is the mechanical response to having had to figure that out by hand at least once.

What the discovery channel was *not*: a leader noticing. This was pre-launch, internal-only, and the bug lived entirely inside the gap between "the wiring is configured" and "the wiring is verified." That gap was a week. Most teams don't close it at all; the version of caching that ships in `research.md` and `spec.md` is the *post-mortem* version, where someone has been bitten and the discipline has been made explicit.

## What Fixed It

Two changes, one shipped, one half-shipped:

1. **`compute_cache_status` (`data_source_chat.rb:206-214`).** Reads `cache_creation_tokens` and `cache_read_tokens` out of `result_meta`, sums the byte length of the cacheable blocks, and returns one of `:cached` (read counter positive), `:seeded` (creation counter positive), `:below_floor` (block sum under `CACHE_FLOOR_CHARS`), or `:none` (above-floor block, both counters zero — the failure shape). Writes the bucket into `result_meta[:cache_status]`. Spec coverage at `spec/services/virtual_employee/capabilities/data_source_chat_caching_spec.rb:131-176` exercises all four states.
2. **What didn't propagate.** `compute_cache_status` only exists in `data_source_chat.rb`. `bsc_analyst` and `meeting_prep` both use `cacheable_blocks:` and neither surfaces a `cache_status`. `grep -rn "compute_cache_status"` returns one hit. The lesson landed for one capability and never got hoisted into `ClaudeClient`, where it would have applied to every capability automatically.

The structural fix that would close this whole class — the one that's still queued — is to make `ClaudeClient.call` itself emit a `:below_floor` signal in the response meta whenever any `cacheable_block` is below the threshold, and let any capability decide what to do with the signal. That hoists the discipline up one layer, where it can't be forgotten and where every new capability inherits it.

Two adjacent open items live in the same neighborhood. **No alert on `:none` with above-floor blocks** — today an operator has to grep `result_meta` to notice that a turn that *should* have hit the cache returned zeros. Pre-launch this was fine; post-launch it's the open back door, and the value of `compute_cache_status` is meaningfully smaller without an alert hooked to it. **The 4-chars-per-token heuristic is conservative-but-wrong-shaped** — KPI tables tokenize denser than 4 chars/token, verbose Spanish prose tokenizes sparser, and the 4096-char floor is right on average and wrong at the edges. The tokenizer-true fix is a counter API call and the cost/benefit hasn't been pulled.

## The Durable Lesson

Verify infrastructure claims with telemetry, not docs.

The wiring was easy. `cache_control: { type: "ephemeral" }` is two tokens of Ruby. The hard part — *did the cache actually cache* — wasn't visible in the code at all; it was visible only in the response telemetry, in two counters that nothing in the system was checking until a week after wiring.

This is the core thesis applied at the call site: every cached token is an assumption about a system you didn't author. Your code says "this block is cacheable." The provider decides whether to actually cache it, when to evict it, what counts as a prefix match for your next call, what minimum size qualifies, whether your model version even participates. Your code doesn't get to find out by introspection. The only feedback channel is the usage-block counters. If you don't read them, you're flying blind on infrastructure you've already paid for.

> **Heuristic.** Anywhere your code makes a contractual claim about provider behavior — *this prompt is cached*, *this token budget is enough*, *this rate limit won't apply here*, *this model version supports this feature* — the verification of the claim has to live in your code, not in the SDK's success path. SDK acceptance is not infrastructure-acceptance. The contract is what telemetry confirms, not what the SDK accepted. For caching specifically: bucket every turn with cacheable blocks into `:cached`, `:seeded`, `:below_floor`, `:none`, and surface the bucket where an operator can see it. `:none` with above-floor blocks is the failure mode that doesn't raise.

The shape generalizes beyond caching. Every provider abstraction has thresholds, behaviors, and silent no-ops the SDK won't surface. The vendor docs document the *number* (1024 tokens, this rate limit, this context window). They under-document the *silent no-op* — what your call looks like when you're below the threshold and the feature silently doesn't apply. OpenAI's, Anthropic's, Bedrock's, Vertex's docs all under-document silent-no-op modes. This is structural to provider abstractions, not vendor-specific.

The closer pairing among the Forge stories is `forge-layered-status.md`. Both are *the function returned ≠ the work happened*. Layered-status: `TaskRouter.call` returned `:queued`, the actual Task succeeded async, the writeback wrote the wrong value because routing-success was conflated with execution-success. Cache: the SDK returned a successful response, the provider didn't cache anything, the operator thought caching was working because the call succeeded. **Both are layering bugs where a successful return value at one layer hides a non-event at the next layer down.** Different stories, same shape underneath.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *load-bearing infrastructure assumptions* — features your code has decided to depend on that the application layer can't verify on its own. It doesn't apply to:

- **Features whose behavior is fully observable at the call site.** A function that returns a value you can check directly doesn't need a separate verification layer. The verification *is* the return value.
- **Decisions where the cost of being wrong is genuinely zero.** If you've turned on a provider feature for cleanliness but you don't actually depend on its behavior — caching where the workload is so small that the savings don't matter, retries where the failures are already tolerable without them — adding telemetry-and-alerting machinery is overhead for nothing.
- **Throwaway exploratory work.** A prototype where you're trying out an SDK feature to see if it does what you think. The verification is the human at the keyboard reading the response. Don't engineer telemetry for code that won't ship.

The signal: *what happens when this provider feature silently doesn't do what your code claims it's doing?* If the answer is "we keep paying for tokens, latency, dollars on the assumption it's free, fast, cached," verify with telemetry. If the answer is "nothing meaningful," the verification cost isn't earning its keep.

A general "always add telemetry for everything" reading of this story is over-applied. Telemetry has its own failure mode — signal-to-noise drops, the dashboard becomes useless, real anomalies hide in normal-noise. The discipline isn't *measure everything*; it's *measure the contractual claims your code is making that the SDK won't verify on your behalf*.

## What This Story Is *Not* Evidence For

- **Not evidence that Anthropic's docs are bad.** They aren't, particularly. The 1024-token floor is documented. What's *not* documented in any vendor's caching docs — and this is structural across provider abstractions, not Anthropic-specific — is *exactly what your call looks like when you're below the threshold and the cache silently doesn't apply*. The lesson is about silent no-op modes as a class, not about a particular vendor.
- **Not evidence that all caching is risky.** Caching is a real provider feature with real cost savings; turning it on is right. The lesson is about *shipping the wiring without the verification*, not about whether to ship the wiring. The wiring is fine. The wiring without the verification is the bug.
- **Not evidence that 4096 chars is the right floor.** It's the right floor for Forge's typical content shape and the 4-chars-per-token heuristic. A capability with denser content (numeric tables, code blocks) tokenizes denser and the heuristic over-estimates; a capability with sparser content (long-form prose) tokenizes sparser and the heuristic under-estimates. The right floor is tokenizer-true, not byte-estimated, and the heuristic is a known approximation.
- **Not evidence that operator-readable telemetry is always better than alerting.** `compute_cache_status` is a *readable* status field, not a *triggered* alert. Pre-launch, that's correct — alerts on a system nobody is operating yet are noise. Post-launch, the alert is the next thing owed. The lesson isn't "telemetry over alerts"; it's "telemetry first, alert when there's an operator who would act on it."
- **Not evidence that pre-launch verification always wins.** The lesson here happens to be "the verification arrived a week after the wiring, before any leader saw a briefing." That's fortunate timing, not a methodology. The reason this story isn't a production incident is sequencing — the `cache_status` layer happened to land before launch. A different sequencing — launch before the verification — would make the same wiring bug a production-visible cost surprise.
