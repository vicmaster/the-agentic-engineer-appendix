# forge / The Function Returned ≠ The Work Completed

## Date / Version Context

- **First writeback fix:** commit `aea302b`, week of 2026-05-05. Fixed `last_run_status` getting stuck at `running` for routine writebacks. Landed on `:queued` as the new terminal value.
- **Second writeback fix:** PR #42, 2026-05-08. Mapped `:queued` to `:succeeded` on the writeback because `TaskRouter` returns synchronously after enqueuing `RunTaskJob` and the `Task` itself completes async. 7-line diff.
- **Project:** Forge — internal Rails 8 / Postgres / Sidekiq app. Virtual Employees execute via `RoutineRunner` → `TaskRouter` (policy gate, idempotency, audit) → capability handler (often LLM-summarized) → `Artifact` rows → delivery. Writeback is the routine-level summary status displayed in the admin UI: did this routine fire successfully or not?
- **Glossary, used in this writeup:** *Routine* = a `ScheduledRoutine` row, fired by `sidekiq-cron` or manually triggered, that produces one `Task`. *Task* = a row in the `tasks` table representing one unit of agent work; has its own status (`:queued`, `:running`, `:succeeded`, `:failed`). *`TaskRouter.call`* = the synchronous entrypoint that enqueues `RunTaskJob` and returns; its return value names the *routing* outcome, not the *execution* outcome. *`last_run_status`* = the column on `ScheduledRoutine` that captures the routine's last-fire status for the admin UI. *Routing layer* = code that decides which Task to create, applies policy, dedupes, returns a routing-level status. *Execution layer* = code that actually runs the Task and updates its status to `succeeded` or `failed`.

## What Was Being Attempted

Show the operator whether a routine fired successfully.

The admin UI lists `ScheduledRoutine` rows and displays each one's `last_run_status` next to the next-fire timestamp. Operators look at this column to confirm "did the BSC analyst run this morning?" It's a single-column status, intentionally simple — green/yellow/red, no drill-down required.

The writeback is the code that, after firing a routine, updates that column. The first version was naive: call `TaskRouter.call`, take whatever it returned, write that into `last_run_status`. The status would round-trip through the routing pipeline and the column would reflect the truth.

That assumption is the bug. There were two truths, on two rows, in two layers. The writeback was looking at one and writing the other.

## What Went Wrong

This is a two-act bug because it took two writeback fixes to surface what was actually wrong.

### Act 1 — `last_run_status` stuck at `running` (`aea302b`)

The first version of the writeback was reading `Task#status` *before* the Task had finished running. `TaskRouter.call` enqueued `RunTaskJob` (Sidekiq async), returned to the writeback, the writeback read the Task row, the row said `running`, and that's what got written to `last_run_status`.

The Task would complete a few seconds later. Its row would update to `succeeded`. But the writeback had already happened — `last_run_status` was permanently `running` for that routine until the next fire.

In the admin UI, every routine looked permanently in-flight. Operators couldn't tell whether yesterday's run had succeeded; the column was a lie.

The fix in `aea302b` was to stop reading `Task#status` synchronously and instead use the value `TaskRouter.call` returned — which, for the happy path, was `:queued` (because `TaskRouter` had successfully enqueued the job and was returning that fact). The column now correctly reflected what `TaskRouter` knew at return time.

The Act 1 fix landed. The column was no longer stuck at `running`. The "false-failure UX" — operators seeing a permanent in-flight state and assuming routines were broken — went away.

### Act 2 — `last_run_status` stuck at `queued` (PR #42)

What replaced it was *the false-pending UX*.

`last_run_status` was now permanently `queued` for every routine. Because `TaskRouter` always returned `:queued` for happy-path runs (that's its routing-layer success status), and the writeback was now writing whatever `TaskRouter` returned, the column read `queued` forever — even after the Task itself had succeeded async.

Operators looking at the admin UI saw "queued" next to every routine. The state never advanced. From the operator's perspective, jobs were stacking up in the queue and nothing was running.

This is *the same bug* as Act 1, with a different terminal value.

What Act 1 had done was move the status from "stuck at running" to "stuck at queued." Both are wrong for the same structural reason: the column was supposed to show *whether the work completed*, and it was being populated with *whether the routing succeeded*. Two different things. Two different rows. Two different layers.

The Act 2 fix in PR #42 was 7 lines. Map `TaskRouter`'s `:queued` return to `:succeeded` on the writeback, with the understanding that `:queued` is the routing layer's terminal success state and the routine's status column wants the *execution layer's* terminal success state. The Task itself, async, will complete or fail; the routine's writeback has to either know that's coming or short-circuit on the routing-layer success and trust that the Task row will record any later failure.

The 7 lines made the layering explicit in code. The mental-model flip — *`TaskRouter.call` returning `:queued` is terminal for the routing layer; the Task's own status is the execution layer's terminal state* — is what stops the next version of this bug from shipping.

## How It Was Discovered

Both bugs were caught by sanity-checking before the June 1 ramp.

This is worth naming explicitly: neither bug was caught by an alert, a status check, or a test. Both were caught because someone looked at the admin UI before a planned production rollout and noticed the column didn't say what it should.

The sanity check was *I'd better look at this before we double the volume*. The bug had been live for some non-zero amount of time before it was noticed. Operators using the UI day-to-day either hadn't noticed, hadn't reported it, or had quietly assumed the column was unreliable and stopped trusting it — which is its own failure mode. A status field nobody trusts is a status field nobody reads, which is a status field that won't catch the next regression.

The discovery channel is *human reads UI before scaling event, notices state never advances*. That's a worse-than-an-alert channel. It's also a better-than-no-channel channel, which is what a system without operator vigilance ends up with.

## What Fixed It

Two PRs, one week apart:

1. **`aea302b` (Act 1):** stop reading `Task#status` synchronously; use the value `TaskRouter.call` returned. Removed the "stuck at `running`" UX. Introduced the "stuck at `queued`" UX without anyone noticing yet.
2. **PR #42 (Act 2):** explicit mapping in the writeback — `TaskRouter`'s `:queued` (routing-layer success) writes as `:succeeded` (execution-layer's optimistic terminal). The Task row remains the source of truth for the execution layer; if the Task fails async, the operator-visible status diverges from the writeback, which is a known-and-handled secondary case.

The structurally correct fix would be to write the *execution layer's* status into `last_run_status` after the Task completes, via a post-execution callback or a Task-row-change trigger that updates the routine. PR #42 didn't do that. It accepted the optimistic mapping as the fix because (a) the failure case is already visible in the Task row, and (b) the writeback path doesn't have a natural async-completion hook.

That's a real trade-off, and it's worth naming honestly: PR #42 makes the happy path correct and accepts that the unhappy path (Task fails async) requires looking at the Task row, not the routine's column. For Forge's audience and operator pattern, that's acceptable. For a higher-volume or lower-trust system, it wouldn't be.

## The Durable Lesson

In agentic systems, "the function returned" is not "the work completed."

The lesson sounds obvious stated this way. It does not sound obvious *while you're writing the writeback*, because every Ruby method you've ever written returns when its work is done. That's the implicit contract of synchronous code: the call returns ⇒ the work happened. Async breaks that contract, and agentic systems break it twice — once for the Sidekiq enqueue, once for the LLM round-trip inside the capability.

Forge's bug is the cleanest single-incident articulation of this. The same column. The same writeback. Two PRs in a week. Two different wrong terminal values. Both bugs are *the same bug* at the structural level: the column was supposed to represent the execution layer's status, and it was being populated by the routing layer's status.

Routing-layer success says "I successfully handed this to the execution layer." Execution-layer success says "the work actually got done." They live on different rows because they answer different questions:

- *Did `TaskRouter` enqueue the job?* — `TaskRouter`'s return value, captured the moment `call` finishes.
- *Did the job complete?* — The Task row, updated by `RunTaskJob` async.

Conflating them is what produces *valid-looking invalid status*. Operators see a value in the column that looks like an answer; the answer is to a different question than they thought they were asking.

> **Heuristic.** Any persisted status field needs to know which layer it represents. If a system has a routing layer and an execution layer (or any version of "I dispatched the work" vs. "the work finished"), each layer needs its own status, and the operator-visible field has to be explicit about which one it's showing. The cheap version is naming the column unambiguously (`routing_status` and `execution_status` separately, never `last_run_status` ambiguously). The expensive version — and the one you'll regret skipping — is making the data model reflect the same distinction the code is already making.

The shape generalizes. This is the same pattern as:

- **HTTP 202 Accepted vs. eventual completion.** The 202 says "I accepted your request"; the work happens later. A client reading 202 as success is wrong unless the system has a follow-up status check.
- **Sidekiq enqueue vs. job completion.** Calling `MyJob.perform_later` returns immediately; the job runs minutes later. Status fields populated at enqueue time describe the wrong layer.
- **Database transaction commit vs. external side effects.** A transaction that includes "send an email" via an after-commit hook returns success when the row commits; the email might still fail. Status fields populated at commit time describe the wrong layer.
- **LLM call returning vs. downstream artifact landing.** An agentic system that calls a model, gets a response, kicks off a delivery job, and writes "succeeded" before delivery confirms is showing the wrong layer's status.

In all four cases, the developer-natural place to write the status is the place that's wrong. The right place is downstream of all the async hops, which is harder to wire and easier to forget.

This pairs structurally with `forge-max-tokens.md`. That story is *the LLM call returned valid-looking invalid output*. This story is *the routing call returned valid-looking invalid status*. The shape is the same: a layer producing a result that looks correct at the call site and is meaningfully wrong at the consumer's point of view. Both fail silently. Both require explicit verification. Both are members of the same family — the system has more layers than the operator can see, and the layers don't raise.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *systems with distinct routing and execution layers, where status visibility matters*. It doesn't apply to:

- **Truly synchronous systems.** A function that does its work in-band and returns when done has one layer. The return value is the answer. Don't add a routing/execution distinction to a system that doesn't have one.
- **Cases where only one layer's status matters to the operator.** A health-check endpoint that only needs to say "is the routing layer alive?" doesn't need execution-layer detail. Pick the layer that matches the question.
- **Systems where the layers are visible by other means.** If operators have a job dashboard (Sidekiq UI, Que, Solid Queue) showing async progress alongside the routine UI, the routine's column doesn't have to carry the execution layer's status — operators look at the right surface for each question. The bug here is that Forge's admin UI was supposed to be a one-stop status; once that's the design, the column has to mean what operators read into it.

The signal: *will an operator look at this status and assume it answers a question that crosses layers?* If yes, make the column mean the layer the operator's question lives in. If no, pick the cheaper layer and document the assumption.

## What This Story Is *Not* Evidence For

- **Not evidence that the routing-layer status is unimportant.** `TaskRouter`'s return value is correct and useful — it's the right answer to "did the routing succeed?" The bug is using it to answer a different question, not the value itself.
- **Not evidence that async architectures are bad.** The Sidekiq + `TaskRouter` + capability-handler pipeline is the right shape for Forge's workload. The lesson is about *status visibility across the async hops*, not about preferring synchronous designs.
- **Not evidence that the optimistic mapping in PR #42 is the perfect fix.** It's the pragmatic fix for Forge's audience. A higher-volume system would need a Task-completion callback that updates the routine, not an optimistic mapping. The lesson is about *naming the layering*; the right implementation depends on the operational stakes.
- **Not evidence that two PRs is always the path.** Sometimes the layering is obvious from day one and the writeback gets it right the first time. The two-PR shape here is the *evidence* — what it took to surface the layering — not a recommendation to always ship the broken version first.
