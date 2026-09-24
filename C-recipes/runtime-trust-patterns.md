# Runtime Trust Patterns — Tool Commands, Audit Pipelines, Telemetry Counters

**Source aside:** Ch. 10 (when to trust the output).
**Source incidents:** `war-stories/markdown-toolkit-reviewer-sweep.md`, `presentation-studio-mcp-feedback-loops.md`, `forge-prompt-cache-minimum.md`.
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026, against markdown-toolkit (WXT Chrome extension), presentation-studio-mcp (TypeScript MCP server), Forge (Ruby on Rails 8 / Postgres / Sidekiq).

## The body principle

Ch. 10, Principle 10: *treat approval as a side-effect, not the terminal state.* In probabilistic systems, *the agent finished* and *the reviewer approved* and *the SDK accepted* are all side-effects of the work, not proof the work was done correctly. The terminal state is whatever structural check confirms the output meets the standard, independent of any single agent's, reviewer's, or SDK's say-so. **Trust the check, not the claim.**

## The recipe at one glance

Three runtime trust mechanisms — one per layer where the gap between *approval* and *actual correctness* can hide. Together they cover the full surface where probabilistic systems can lie quietly: in tool sweeps, in artifact-shape checks, and in infrastructure-claim verifications.

This recipe shows what each mechanism looks like in commit-able code. The other two C-recipes ([`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md) and [`observability-across-stacks.md`](observability-across-stacks.md)) carry the deep-dive implementations for Layers 2 and 3; this entry is the *runtime trust* framing — the cross-layer view that says *the trust signal lives outside the claim* at three different layers.

## The three layers share one shape

Each layer answers a different version of the same question: *what structural check survives when no single party's say-so is the trust signal?*

| Layer | The claim | The trust signal |
|---|---|---|
| 1. Tool-checkable verification | *the reviewer approved* | `grep` returns zero |
| 2. Per-call audit | *the agent finished* | the audit pipeline's structured findings |
| 3. Telemetry verification | *the SDK accepted* | the verification record's counters |

The pattern in one line: **the trust comes from a structural check that survives without any single party's say-so.**

## Layer 1 — Tool-checkable verification (the sweep)

**Claim:** *the reviewer approved; the sweep is complete.*
**Trust signal:** *the tool command returns the expected value.*

### The durable shape (pseudocode)

```
# Pseudocode: sweeps with tool-command acceptance criteria

# 1. Define the sweep as "delete/migrate/rename/audit all X"
sweep:
    scope: "all instances of <pattern> in <paths>"

# 2. Express the AC as a tool command, not as a human/agent observation
acceptanceCriteria:
    command: "grep -r '<pattern>' <paths>"
    expected: "no results"
    # NOT: "the reviewer confirms"
    # NOT: "the engineer eyeballed the diff"

# 3. Run the command. If non-zero, the sweep isn't complete.
function verifySweep(sweep, ac):
    output = run(ac.command)
    if output is not empty:
        return INCOMPLETE
    return COMPLETE
```

### Source incident: `markdown-toolkit-reviewer-sweep.md`

The reviewer enumerated three dead CSS classes: `.brand-text`, `.tool-item-body`, `.tool-list-soon`. The engineer deleted those plus two more found in passing. v1 reviewer approved on second pass.

Then the v1 reviewer on a *third* pass found two more orphan rules — `.tool-item-name` and `.tool-item-desc` — same prefix, same stylesheet, missed by both passes. A `grep -r '\.tool-item-' src/` would have surfaced all four `.tool-item-*` classes at once.

The structural defect: the sweep methodology was *delete the classes the reviewer named*. That methodology produces a complete sweep only if the reviewer's enumeration was complete. Reviewers — human or agent — read code and list what they noticed. They don't grep exhaustively.

### The implementation

The v2 cleanup ticket's load-bearing acceptance criterion:

```markdown
# .tickets/2026-05-04-cleanup-v2.md

## Acceptance Criteria

**AC1.** Run `grep -r` for the relevant pattern across `src/` and `entrypoints/`.
The ticket is not done until the grep returns zero results.

```sh
grep -rn '\.tool-item-' src/ entrypoints/   # expected: no output
grep -rn '\.brand-text' src/ entrypoints/   # expected: no output
grep -rn '\.tool-list-soon' src/ entrypoints/ # expected: no output
```
```

A reusable shape — sweep tickets get a templated AC section:

```markdown
## Acceptance Criteria

**AC1 (sweep completeness).** The following tool commands all return zero output:

| Command | Why |
|---|---|
| `grep -rn '<pattern>' <paths>` | All references to `<pattern>` are gone |
| `bundle exec rake unused_constants` | No unused constants remain in scope |
| `<your_tool> <args>` | <The structural claim the sweep depends on> |

The reviewer's role is to catch what the tool can't (intent, fit, naming).
The tool's role is to catch the exhaustiveness the reviewer can't.
```

### Other classes of work the same pattern serves

The grep-zero AC pattern is the template for any *delete-all*, *migrate-all*, *rename-all*, *audit-all* work:

- **Migration tickets.** *Migrate all callers of `oldFn` to `newFn`.* AC: `grep -rn 'oldFn(' src/` returns zero.
- **Dependency removals.** *Remove all uses of `lodash`.* AC: bundle analyzer output for `lodash` is `0 bytes`, or `grep -rn "from 'lodash'" src/` returns zero.
- **Eval rule additions.** *Every public endpoint should have rate-limiting.* AC: a static-analysis pass lists endpoints without rate limits, and the list is empty.
- **Security policies.** *No secrets in commit history.* AC: a repo-wide regex scan against the secrets pattern returns zero matches.

### What survives the tool changing

The principle: **the reviewer's job is to catch the things the tool can't (intent, fit, naming); the tool's job is to catch the exhaustiveness the reviewer can't. Don't conflate the two.** The specific commands (`grep`, `ripgrep`, `ast-grep`, a custom static analyzer) will rotate. The discipline of putting the exhaustiveness check into a tool — and making the tool's output the terminal state — doesn't.

## Layer 2 — Per-call audit (the artifact-shape check)

**Claim:** *the agent produced a deck; the artifact is ready.*
**Trust signal:** *the audit pipeline's structured findings are all-clear.*

### The durable shape (pseudocode)

```
# Pseudocode: per-call audit as the runtime trust signal

function renderAndAudit(input):
    artifact = render(input)
    report   = auditDeck(input)    # twelve rules, deterministic order
    return {
        artifact: artifact,
        report:   report,
        trustSignal: report.passed,   # the trust signal — NOT the agent's stopping
    }

# The agent's iteration loop reads the report.passed flag, not its own claim
function agentLoop(input):
    result = renderAndAudit(input)
    if not result.trustSignal:
        revisedInput = editSpec(input, result.report.issues)
        return agentLoop(revisedInput)
    return ship(result.artifact)
```

### Source incident: `presentation-studio-mcp-feedback-loops.md`

The full implementation lives in [`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md) — twelve audit rules, Zod schemas for `AuditIssue` / `AuditReport`, the deterministic driver, the 1.4× severity escalation cliff, and the agent's iteration loop.

The Ch. 10 angle on the same code: the audit pipeline's output is the *trust signal at runtime*. The agent saying *I produced a deck* doesn't earn trust. The audit's findings being clean is what does.

What the audit *doesn't* check matters as much as what it does: it doesn't check whether the slides are insightful, whether the message is correct, whether the audience will understand. Those are judgment calls upstream or downstream. The audit checks the *shape* — did the agent honor the contract the tool enforces? Layouts in the enum, density within bounds, required fields populated, every audit rule passing.

### The implementation — the runtime trust hook

The deep implementation is in `audit-pipeline-typescript-mcp.md`. The Ch. 10 framing surfaces one specific shape: the `trustSignal` field that callers can route on:

```ts
// apps/mcp-server/src/tools/renderDeck.ts
import { renderPptx, auditDeck } from "@psm/core";
import type { DeckSpec } from "@psm/schema";

export async function renderDeckTool(input: DeckSpec) {
  const pptxBuffer = await renderPptx(input);
  const report = auditDeck(input);

  return {
    pptxPath:    persist(pptxBuffer, input.deckId),
    report,
    trustSignal: report.passed,
    // The caller reads trustSignal. The agent's iteration loop uses this
    // boolean, not its own claim about whether the deck is ready.
  };
}
```

The general shape, beyond decks: **for any artifact your system produces that has a checkable structure, write the structural check and run it on every output. The check's pass is the trust signal; the agent's completion is a side-effect.**

Other classes of work the same pattern serves:

- **Generated SQL or code.** Parse the output, type-check it, run it against a known-shape test. The compile-clean is the trust signal, not "the agent said it was correct."
- **Structured emails or messages.** Validate the schema, check required fields, run the formatter. Schema-passes is the trust signal.
- **Multi-step plans.** Check that each step has the required fields, that dependencies form a DAG, that the count is within bounds.
- **Generated images.** Run a perceptual diff against a known-good baseline, or a structural validator that confirms the chart is well-formed.

### What survives the implementation changing

The principle: **the audit's findings being clean is the trust signal, not the agent's claim of done.** The specific audit rules will rotate as the tool's domain evolves; the *audit-runs-on-every-call-and-its-output-is-the-trust-signal* shape doesn't.

## Layer 3 — Telemetry verification (the infrastructure claim)

**Claim:** *the SDK accepted the call; the cache applied.*
**Trust signal:** *the verification record's counters confirm the provider's behavior.*

### The durable shape (pseudocode)

```
# Pseudocode: telemetry verification as the trust signal

function callProviderWithCache(messages, system, cacheableBlocks):
    response = providerSDK.call(messages, system, cacheableBlocks)
    bucket   = classifyCacheCall(response, cacheableBlocks)

    # Emit the verification record on every claim-bearing call
    metrics.emit("provider.cache.bucket", value=bucket, tags={...})

    return {
        content:     response.content,
        trustSignal: bucket in [CACHED, SEEDED],
        bucket:      bucket,
    }

# Bucketing — the structural check that survives the SDK's success path
function classifyCacheCall(response, cacheableBlocks):
    if response.usage.cacheReadTokens > 0:     return CACHED
    if response.usage.cacheCreationTokens > 0: return SEEDED
    if size(cacheableBlocks) < CACHE_FLOOR:    return BELOW_FLOOR
    return NONE  # above floor, both counters zero — the failure shape
```

### Source incident: `forge-prompt-cache-minimum.md`

The wiring claim was *we cached this block*. The provider's SDK accepted the call. The response shape was identical to a successful cache call. Two counters in the response said zero, and reading them was the only way to know caching had silently declined.

The full implementation lives in [`observability-across-stacks.md`](observability-across-stacks.md) — the Ruby `compute_cache_status` method, the four buckets (`:cached`, `:seeded`, `:below_floor`, `:none`), and the spec coverage that exercises all four states.

The Ch. 10 angle on the same code: the bucket *is* the runtime trust signal. The SDK accepting the call earns no trust about whether the work happened; the verification record's bucket does.

### The implementation — the runtime trust hook

The deep implementation is in `observability-across-stacks.md`. The Ch. 10 framing surfaces the shape callers route on:

```ruby
# app/services/virtual_employee/llm/claude_client.rb
def call_with_cache(messages:, system:, cacheable_blocks: [])
  response = anthropic_client.messages.create(
    model: model_name,
    system: build_system(system, cacheable_blocks),
    messages: messages,
  )

  bucket = compute_cache_status(
    cache_creation_tokens: response.usage.cache_creation_input_tokens,
    cache_read_tokens:     response.usage.cache_read_input_tokens,
    cacheable_blocks:      cacheable_blocks,
  )

  {
    content:      response.content,
    trust_signal: %i[cached seeded].include?(bucket),
    bucket:       bucket,
  }
end
```

The general shape, beyond caching: **for any infrastructure feature your system depends on, log the verification record on every claim-bearing call, and treat the verification's pass as the trust signal.** The SDK's lack of error is the *call returned* signal; the verification's pass is the *the work I asked for actually happened* signal. Conflating them is the failure shape Ch. 3's *infrastructure-claim test* defends against.

Other infrastructure claims the same pattern serves:

- **Rate-limit back-off.** Did your client back off when the provider asked? Log the headers, log the back-off, treat the verification as the trust signal — not "the call returned 200."
- **Model selection.** Did the provider route your call to the model you asked for? Some providers fall back silently. Log the actual model in the response.
- **Streaming behavior.** Did the stream complete cleanly, or did the connection drop and your handler glue the partial together? Log the stream's terminal state.
- **Tool routing in the harness.** If your harness can choose between local and remote tool execution, did it pick the path you expected? Log the actual path taken.

### What survives the SDK changing

The principle: **SDK acceptance is not infrastructure-acceptance.** The provider's field names, the four-bucket taxonomy, the specific cache floor — all dated. The discipline of bucketing every claim-bearing call into states-the-SDK-won't-distinguish, and routing on the bucket rather than on the SDK's success, doesn't.

## The trust ladder — operational matching of cost to mechanism

The three layers above are *mechanisms*. The decision of *which to use, when* is the **trust ladder** — three rungs that map to cost-of-being-wrong tiers from Ch. 4.

### The three rungs

```
# Pseudocode: the trust ladder, anchored to cost-of-being-wrong

function pickRung(output):
    cost = costOfBeingWrong(output)

    match cost:
        case IRREVERSIBLE:
            return GATE  # human authorization per output, always
        case HIGH_HARD_TO_UNDO:
            return GATE or STRICT_POST_HOC_VERIFY
        case MODERATE_REVERSIBLE:
            return POST_HOC_VERIFY  # structural check, no human-per-output
        case LOW_WITH_DRIFT_RISK:
            return SAMPLE  # review queue catches drift, not every wrong output
        case LOW_NO_DRIFT_RISK:
            return TRUST_AND_OBSERVE  # seam alerts catch aggregate failures
        case GENUINELY_ZERO:
            return TRUST  # human catches problems naturally
```

### The matching table

| Cost of being wrong | Rung | What's load-bearing |
|---|---|---|
| Irreversible | **Gate** | Human authorization per output |
| High, hard to undo | **Gate** or strict post-hoc verify | Either human-in-loop, or structural verification with rollback |
| Moderate, reversible | **Post-hoc verify** | The structural check (Layer 1/2/3 mechanisms) |
| Low, with drift risk | **Sample** | The review queue catching drift, not every output |
| Low, no drift risk | **Trust + observe** | The seam alerts (Ch. 8) catching aggregate failures |
| Genuinely zero | **Trust** | Nothing; the human catches problems naturally |

### Sampling: the under-engineered rung

Sampling is the rung that gets the least design attention and fails most quietly. Four under-engineered shapes from Ch. 10:

- **Sample rate too low.** A 5% sample of 100 outputs/day is five reviews. If the failure rate is 1/200, you might not catch it for months. Sample rate has to track the failure rate, not the volume you can review.
- **Reviewer queue becomes toil.** Once "reviews" outpace "catches," reviewers stop reading carefully. Sampling without fast feedback to the reviewer is the same as no sampling.
- **No drift signal.** Sampling that produces verdicts but no trend line is observation without learning. The trend dashboard is the design surface; the percentage isn't.
- **Sampling on the wrong axis.** Random sampling catches average-case drift. If the concern is *long-tail edge cases get worse*, targeted sampling — explicitly pulling outputs that match high-risk patterns — catches what random sampling won't.

Pragmatic version: **start with random sampling at a defensible rate (5–10% is typical), pair it with a trend dashboard, and add targeted sampling for any pattern your team has reason to worry about.**

## Three questions for the gate-vs-verify call

The hardest call in the trust ladder is the one between *gate* and *post-hoc verify* for moderately-high-cost outputs. From Ch. 10:

1. **Can the check be written in code?** If yes, post-hoc verify is a real option. If the check is genuinely a judgment call that requires reading novel context, gating is the honest answer. Don't fake a code check by using a model to make the judgment — your check then has the same probabilistic-failure problem as the original output.
2. **Does the system tolerate the verification's failure mode?** Post-hoc verify can fail by passing bad output (false negative) or by blocking good output (false positive). Both have costs. If false negatives are catastrophic, gate. If false positives are tolerable, verify.
3. **Is the cost-of-being-wrong asymmetric?** If a wrong output costs much more than a delayed good one, gate. If the cost is symmetric, verify with a good check beats gate.

## Counter-examples — when these patterns don't apply

- **Outputs whose structural shape isn't checkable in code.** Creative writing, brand voice, taste-bearing judgment calls. Layer 2's audit pipeline has nothing to grade against. Either move up to gating (human review per output) or accept that the trust signal is the human's read.
- **Reviews of judgment, taste, or architectural fit.** Layer 1's tool-command AC doesn't replace reviewer enumeration where the reviewer's actual job is intent, naming, abstraction quality. The lesson is specific to *sweeps* (work that depends on exhaustive enumeration), not to reviews in general.
- **Single-stage scripts with one obvious failure mode.** A 30-line script doing one thing in one place doesn't need the three-layer framework. The runtime's exception either fires or doesn't.
- **Cases where the cost of a wrong output is genuinely zero.** A back-of-house log entry, a draft generation reviewed before shipping, a sandbox experiment. The trust-ladder rung is *trust*, and the framework is overkill.
- **Workflows where the agent and operator both contribute, with the operator's part still requiring practice.** The operator's editing-the-draft reflex is the trust signal in this case. Adding a structural check on top would mostly be theater.

The signal: *if this output is wrong, what catches it?* If the answer is *a human reading it in real time*, the framework is overkill. If the answer is *a human downstream notices eventually, maybe weeks later, after damage*, pick a rung from the ladder anchored to cost-of-being-wrong.

## What survives the tooling rotating

The principle: *treat approval as a side-effect, not the terminal state; trust the check, not the claim.* The specific tools — `grep`, `ripgrep`, the audit-rule directory layout, the four-bucket cache classifier with current Anthropic field names — will all rotate. The three-layer framework and the trust ladder won't:

1. **Tool-checkable verification** for sweeps. The check is a tool command; the trust signal is its specific return value.
2. **Per-call audit** for artifacts. The check is a structural validator running on every output; the trust signal is the audit's `passed` flag.
3. **Telemetry verification** for infrastructure claims. The check is a verification record bucketed against expected states; the trust signal is the bucket.

Every probabilistic-output system has a version of this question; the discipline is making the structural check the load-bearing trust signal, not the agent's stopping.

## Cross-references

- Ch. 10 (When to Trust the Output) — full chapter treatment.
- Ch. 4 (Tool Surfaces and Trust Boundaries) — the gate/verify/trust framework at design time; this recipe is the runtime version.
- Ch. 5 (Verification in the Production Loop) — the audit-pipeline mechanism this recipe leans on for Layer 2.
- Ch. 6 (Failure Modes Catalog) — the *partial sweeps treated as complete* failure mode Layer 1 defends against.
- Ch. 8 (Observability for Probabilistic Systems) — the verification-record mechanism this recipe leans on for Layer 3.
- Ch. 12 (Security and Trust Under Adversarial Input) — the adversarial extension of these runtime-trust layers: the same verify-outside-the-model discipline, pointed at a hostile input.
- `war-stories/markdown-toolkit-reviewer-sweep.md` — Layer 1 source incident.
- `war-stories/presentation-studio-mcp-feedback-loops.md` — Layer 2 source incident.
- `war-stories/forge-prompt-cache-minimum.md` — Layer 3 source incident.
- [`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md) — Layer 2 deep implementation.
- [`observability-across-stacks.md`](observability-across-stacks.md) — Layer 3 deep implementation (and the *log the claim, not the call* discipline more broadly).
- [`team-primitives-cross-vendor.md`](team-primitives-cross-vendor.md) — the gates-in-skill primitive operationalizes the *gate* rung at the harness level.
- [`../B-sdks/anthropic-prompt-caching.md`](../B-sdks/anthropic-prompt-caching.md) — the SDK field shapes Layer 3 reads.
