# forge / The Silent Truncation

## Date / Version Context

- **Date:** Week of 2026-04-22, fixed at commit `2151549`. The bug existed since the first leadership brief shipped — likely 2026-04-19 to 2026-04-22 — depending on which capability fired first.
- **Project:** Forge — internal Rails 8 / Postgres / Sidekiq app at MagmaLabs. Virtual Employees (supervised, capability-scoped agents) generate LLM-summarized output that gets delivered to leadership via Slack DM and email.
- **Surface for this story:** the `bsc_analyst` capability inside the Magma Virtual Ops Analyst (`db/seeds.rb:49`). The handler calls Claude Sonnet 4.6, packs the BSC context into the prompt, and asks for a structured leadership brief. The response goes through the delivery boundary and lands in a leader's Slack DM or inbox.
- **Glossary, used in this writeup:** *Brief* = the structured leadership-facing summary the capability produces (sections like "Risks", "KPI Movement", "Meeting Prep"). *`max_tokens`* = the hard ceiling on the model's response length, enforced by the Anthropic API. *`stop_reason`* = the field on the API response that names why generation halted: `end_turn`, `max_tokens`, `stop_sequence`, or others. *Truncation* = the model hitting `max_tokens` mid-output, with the response ending wherever the budget ran out.

## What Was Being Attempted

Ship the first leadership-facing brief. The `bsc_analyst` capability reads the Balanced Scorecard data (risks, KPI movements, meeting context), builds a prompt around it, calls Claude Sonnet 4.6, and returns a structured Spanish-default brief for leadership consumption.

The initial `max_tokens` was 4096. That number wasn't researched or tuned — it was a default that propagated from earlier prototypes. The reasoning, to the extent there was reasoning, was *4096 is plenty for a one-page brief.* And for a small-input case it was. For a leader reading two paragraphs of risk summary, 4096 was fine.

Then real BSC data started landing — multiple risks across several scorecards, KPI movement with quarter-over-quarter narrative, three meetings worth of prep notes — and the brief started reaching for the ceiling.

## What Went Wrong

A leader received a brief that ended mid-sentence.

Not "ended awkwardly" or "felt incomplete." Ended mid-sentence, mid-bullet, with no period, no structural close, no signal that anything had been cut. The brief looked like every other brief — same Slack rendering, same section headings, same prose voice — until the last line, which was a half-thought.

The model had hit `max_tokens` at 4096 and the API had returned `stop_reason: "max_tokens"`. Forge's handler had taken the response, run it through `SlackMarkdown` (see `forge-delivery-boundary.md`), and posted it. The truncation was visible in the API response. It just wasn't visible *to Forge*, because nothing in the handler checked `stop_reason`.

The bug isn't that 4096 was too small. The bug is that 4096 being too small produced *valid-looking invalid output*. The capability didn't fail. Sidekiq didn't retry. No log line said "this brief was truncated." The system was operating with full apparent success while shipping a broken artifact.

This is the load-bearing distinction. A deterministic system with the same defect would have raised an exception ("string too long for column") or thrown a parse error ("expected closing brace") and surfaced the failure at the call site. The probabilistic system swallowed the defect into the output shape, where the only signal was *a human reading it and noticing.*

## How It Was Discovered

Leadership read the brief. A leader noticed the brief ended mid-sentence and asked whether something was wrong. The discovery channel was a human in the output path.

This is the worst-case discovery channel for a class of bug that exists because the system can't catch the bug itself. There was no eval that asserted "the brief contains a closing line." No spec that ran `stop_reason` against expectations. No alerting on `:max_tokens` responses. No regression test on real-shaped BSC inputs. The truncation could have shipped to leadership for weeks before anyone noticed, depending on which leader read which brief on which day.

Two facts make this discovery channel worse than it sounds:

1. **The audience is small.** Forge has ~5 leadership users. A bug that a leader has to *read carefully* to catch is a bug that, in a 5-user system, may go unsurfaced for any single brief. Multiply that across 4 capabilities and weekly cadence and the latency between bug-introduction and bug-report can be substantial.
2. **The output is structured Spanish-language prose.** A reader skimming for "did the bot say anything useful" can absolutely miss a missing terminal period. The bug requires the leader to be paying enough attention to notice the brief ended *abnormally*, which is a much stronger signal than "the brief existed."

In a deterministic system the truncation would have been a stack trace. In Forge it was a stylistic anomaly that depended on a human caring enough to flag it.

## What Fixed It

Commit `2151549` doubled `max_tokens` from 4096 to 8192. That's the entire diff for the immediate fix.

What didn't land in the same commit, and is owed: the structural fix. Two options, both worth doing:

1. **Check `stop_reason` and surface truncation explicitly.** The API tells you when it truncated. The handler can read `response.stop_reason`, and on `:max_tokens`, either retry with a larger budget, append a "[TRUNCATED]" marker, or refuse to deliver and queue for retry. The fix is mechanical; the discipline is remembering to wire it in for every capability.
2. **Budget headroom you'll never use.** 8192 is more than any current brief needs. The point isn't right-sizing — it's giving up the right-sizing optimization in exchange for never hitting the ceiling under realistic input. For Forge's audience and cost profile, the extra tokens-per-call cost is invisible; the operational cost of one truncated brief reaching a leader is large.

The pragmatic fix is doing both. Budget the headroom *and* check `stop_reason`, because the headroom is for the typical case and the check is for the day a future capability ships with longer input than expected.

What `2151549` actually did was the headroom half. The check is open work. The lesson is half-paid.

## The Durable Lesson

Probabilistic systems fail silently in ways deterministic systems don't.

Deterministic code has a closed set of failure modes — exceptions, timeouts, type mismatches, validation errors. Each one has a place where it surfaces and a runbook for how to handle it. The system tells you when something went wrong, in the layer where it went wrong, often before anything reaches a user.

LLM calls don't have that property. The model can hit `max_tokens` and return a half-finished response. It can refuse to follow the format and return prose where you expected JSON. It can hallucinate a closing structure that's syntactically valid and semantically wrong. It can produce a response that's the right shape, the right length, and the wrong content. None of these failures raise an exception. All of them produce *valid-looking output that's actually broken.*

The implication is that the verification step has to live in your code, not in the API's error handling.

> **Heuristic.** For any LLM call whose output is consumed downstream (rendered, stored, sent to a user, fed to another tool), the handler must verify *both* that the call succeeded and that the call's output is shaped correctly. `stop_reason: "end_turn"` is the success signal; anything else is a failure mode you need to handle explicitly. Default to the maximum reasonable token budget *and* check the stop reason — belt and suspenders, because either alone leaks.

The deeper observation: this is a member of a family. `max_tokens` truncation is the most concrete instance, but the same pattern shows up everywhere LLM output meets downstream code:

- **JSON output that's almost-valid.** The model produces a response that parses as JSON in 99% of cases and emits a trailing comma in 1%. The handler has to validate, not assume.
- **Format-following that drifts under load.** A capability that asks for "Section A, Section B, Section C" gets all three for typical input; for edge-case input the model collapses Sections B and C into one. The downstream renderer assumes three sections.
- **Language that switches mid-output.** A Spanish-default capability getting partly-English content because the input had English data the model decided to echo. The brief is "valid prose," just half in the wrong language.

All four are *valid-looking invalid output*. None of them raise. All of them require the handler to check.

The audit-as-eval framing from `presentation-studio-mcp-feedback-loops.md` is the same principle applied at the tool-design layer. This writeup is the same principle applied at the call-site layer. The two together form a shared spine: probabilistic systems require explicit verification because they cannot produce error signals on their own behalf.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *LLM output that's consumed by downstream code or users*. It doesn't apply to:

- **Throwaway interactive use.** A chat where the human reads the model's output, notices it's truncated, and just asks again. The verification is the human; no system-level handling is needed.
- **Outputs where truncation is acceptable or even desired.** Streaming a response and stopping early on a stop sequence is the design. `stop_reason: "stop_sequence"` is success in that flow; treating it as failure would be wrong.
- **Cases where the cost of one truncated output is genuinely zero.** A back-of-house log entry that no one reads, a draft staging-only artifact, a non-customer-facing scratch generation. Don't engineer verification for outputs whose consumers don't care.

The signal: *what happens when this output is wrong?* If the answer is "a human reads it and possibly doesn't notice," verify. If the answer is "the human reading it will obviously catch it," skip the verification and save the complexity.

## What This Story Is *Not* Evidence For

- **Not evidence that 8192 is the right number.** It's the right number for Forge's current capabilities and audience. The lesson is about *the budgeting discipline*, not the specific value. Different capabilities need different budgets; the right number depends on input shape, output structure, and user tolerance for retries.
- **Not evidence that doubling fixes everything.** The doubling fixed the symptom for the typical case. The structural fix (`stop_reason` checking) is owed. A future capability with even longer input will hit 8192 and produce the same valid-looking invalid output — at which point the lesson reasserts itself, more expensively.
- **Not evidence that all truncation is bad.** Streaming, stop-sequences, deliberate budgets for cost control — these are designs where the truncation is intentional and the system handles it. The lesson is about *unintentional* truncation that leaks downstream.
- **Not evidence that the discovery channel is unique to Forge.** A human reading the output and noticing is the discovery channel for *most* probabilistic-system bugs in the absence of explicit verification. Forge's small audience makes the latency variable; larger audiences just shift the discovery from "weeks later" to "hours later" while preserving the underlying property — the human is the eval.
