# forge / The Routine That Said It Ran

## Date / Version Context

- **Date:** The dedupe behavior shipped with the first version of Forge's `RoutineRunner` (pre-launch arc, 2026-04). The `force_run_token` fix landed in commit `aea302b`, week of 2026-05-05 — the same commit that fixed the `last_run_status` getting-stuck-at-`running` bug from `forge-layered-status.md`. Both fixes shipped together because they both surfaced from the same sanity-check pass before the June 1 ramp.
- **Project:** Forge — Ruby on Rails 8 / Postgres / Sidekiq / Heroku. `ScheduledRoutine` rows are fired by `sidekiq-cron` on a daily cadence, each routine producing one `Task` per fire. Routines are also operator-triggerable manually from the admin UI for testing, debugging, and special-case re-runs.
- **Surface for this story:** the manual-run button on the `ScheduledRoutine` admin page. Pre-fix: clicking the button enqueues a routine fire that dedupes against the day's existing Task. Post-fix: two buttons — a normal *Run now* (still dedupes) and a *Force run* (generates a fresh `force_run_token`, bypassing the dedupe).
- **Glossary, used in this writeup:** *Idempotency key* = a value that determines whether a new execution should run or return a previously-completed result. *Dedupe bucket* = the time or value range that maps multiple potential executions to the same key (here, `YYYY-MM-DD`). *Stealth state* = system state that gates behavior but is invisible to the operator without explicit instrumentation. *Force path* = a deliberate operator-side escape hatch that bypasses idempotency, with a distinct UX from the normal-fire path so the operator can see they're using it.

## What Was Being Attempted

Prevent accidental double-fires of daily routines.

The design choice was correct. A `bsc_analyst` routine that fires daily at 8am produces one leadership brief per day. If `sidekiq-cron` flakes and tries to fire twice, or if an operator clicks the manual-run button while the scheduled fire is in-flight, or if any other race condition emerges, the system should produce *one* Task for the day, not two. Duplicate briefs landing in leadership Slack DMs would be confusing at best and embarrassing at worst.

The dedupe implementation was conventional. Each routine fire computes an idempotency key — `"#{routine_id}:#{Date.current.iso8601}"` — and `TaskRouter` checks for an existing Task with that key before enqueuing a new one. If one exists, the router returns the existing Task. If not, it creates a new Task and enqueues `RunTaskJob` async.

By every database-correctness criterion, the implementation was sound. Tasks are unique per routine per day. Leadership never received a duplicate brief. The dedupe did its job.

What the implementation didn't have, until `aea302b`, was an operator-visible signal that distinguishes *the routine fired and produced this Task* from *the routine deduped and returned the same Task you saw before.*

## What Went Wrong

The operator couldn't tell the difference between *fired* and *deduped*.

Two specific moments where the gap bit:

**Scenario 1 — manual re-run for testing.** Operator wants to confirm a routine works after a code change. Clicks *Run now*. Page reloads. The routine's `last_run_status` shows `:succeeded` and the timestamp shows this morning. From the operator's POV: *did my click do something?* The answer is *no, your click hit the dedupe path and returned the morning's existing Task*. There's no banner saying so. The button gave no immediate-feedback distinction between *new fire* and *deduped fire*.

The operator's mental model wobbles. *Maybe the button is broken? Maybe the routine ran my code change but is still showing the old timestamp? Maybe Sidekiq is misbehaving?* The standard debugging instinct — *click it again, see if anything changes* — produces the same result every time, because the dedupe key is sticky for the day. The operator either gives up or escalates to reading the Sidekiq dashboard, the database, or the logs to figure out what the click actually did.

**Scenario 2 — operator-triggered force-run after a partial failure.** A routine fired in the morning but its capability handler crashed midway. The Task row says `:failed`. The operator wants to re-fire the routine after the underlying bug got fixed an hour later. Clicks *Run now*. The dedupe path triggers because a Task already exists for today (the failed one). The system returns the failed Task. The operator sees the same `:failed` row, can't make the system try again, and has no way to indicate *"I know there's a Task for today; that's the one that's broken; please fire a fresh one."*

In both scenarios, the system's behavior was *technically correct* — the dedupe was doing what it was designed to do. The operational gap was *the operator couldn't see the dedupe was the reason*.

This is a *state opacity* lesson at the routine-execution layer. The system has more state than the operator can see — specifically, the idempotency key cache, which is invisible from the admin UI. The button-click result depends on that state; the result's appearance to the operator doesn't tell them which state path was taken.

## How It Was Discovered

Sanity-checking before the June 1 ramp.

Same discovery channel as `forge-layered-status.md`. Both bugs were caught by a human looking at the admin UI before a planned scaling event, noticing something that didn't add up. For idempotency, the moment was *I want to test this routine; the button doesn't seem to do anything; let me look at the code to see what's happening.* The code review surfaced the dedupe-without-force-path gap, and the fix went out with the layered-status fix in the same `aea302b` commit.

The non-discovery channel — and the part worth pausing on — is that *no alert, no metric, no test failure would have caught this*. The dedupe works as designed. Every test that mattered passed. The bug lived in the gap between *the operator's mental model of what the button does* and *what the button actually does under conditions the operator could trigger*. That gap is invisible to standard QA disciplines because the system's behavior is correct; the *operator-visible signal* is what's missing.

Most operator-visible-signal bugs of this shape only surface when an operator is trying to do something and the system's silence frustrates them. The frustration is the discovery channel. A team where the operators are too senior to admit the system confused them, or too disengaged to escalate, would never surface this.

## What Fixed It

`aea302b` introduced two operator-side affordances:

1. **The `force_run_token` admin button.** A second button on the `ScheduledRoutine` admin page, distinct from *Run now*. Clicking it generates a fresh token (a UUID or timestamp-derived nonce) and the routine fire uses that token as part of its idempotency key — `"#{routine_id}:#{Date.current.iso8601}:#{force_run_token}"`. The fresh token guarantees the dedupe path doesn't match. A new Task is created. The operator gets a *new* fire, deliberately, with a UI affordance that names what they're doing.

2. **A banner on the `Run now` path that names the dedupe.** When *Run now* hits the dedupe path, the page surfaces *"This routine already ran today at 08:00:14. Returning the existing Task. Click [Force run] to fire a fresh one."* The operator now sees what happened. The dedupe is still doing its job; the operator's model of what the button did is now correct.

The fix is asymmetric: a *small* code change in the controller (~7 lines for the banner, ~15 for the force-token path), a *large* improvement in operator confidence. The structural shift is from *idempotency as invisible state* to *idempotency as visible state with an explicit override*.

What's owed but not yet shipped: a *force-run audit log*. Every force-run is now a deliberate operator action; recording who clicked the button, when, and why (a free-text field in the click flow) would close the loop on accountability for re-runs. The current implementation generates the force-token but doesn't record the actor; that's a follow-up for another day rather than a gap in this fix.

## The Durable Lesson

Idempotency keys are the most invisible state in routine-execution systems. **Anything an operator might re-trigger needs an explicit force path with a distinct UX, or the operator will assume the system is broken.**

The mental-model flip the lesson rests on: the operator and the system have different definitions of "did the click do something." The system's definition is *did execution happen* (it didn't, by design — dedupe). The operator's definition is *did the click cause an observable change* (it didn't, by accident — no banner). The gap between those two definitions is where every idempotency-without-force-path bug lives.

> **Heuristic.** For any system action that an operator might want to re-trigger — manual runs of scheduled work, retries after partial failures, force-resends of stuck messages, replay of dropped events — design two distinct UX paths from day one: the normal-fire path (dedupes against the key) and the force path (bypasses the key with a deliberate token). The normal path should surface a banner naming the dedupe when it hits. The force path should look visibly different and require an extra click. Both paths are correct; the operator-visible distinction between them is the bug fix.

The shape generalizes far beyond `RoutineRunner`:

- **Sidekiq job uniqueness (`unique: :until_executed`, `:until_and_while_executing`, etc.).** Same shape — duplicate jobs dedupe, operators can't tell whether their enqueue hit the dedupe path. Force-path: a separate "queue without uniqueness check" button or admin action.
- **Webhook receiver dedupe by event ID.** Re-delivery of the same event from a provider dedupes; if the operator wants to *force* a re-process (because the first process failed silently downstream), they need a force path that bypasses the event-ID check.
- **Upload pipelines that dedupe by content hash.** A user uploading the same file twice gets the existing record by default — fine for most cases. The operator-side debug flow needs a force-upload that creates a new record with a fresh ID, separate from the dedupe.
- **Billing event dedupe by external ID.** Stripe and similar providers dedupe by `idempotency_key`. A finance-ops user wanting to *re-submit* a billing event for any reason needs a force path with a fresh key, plus an audit trail of why.
- **Cron jobs that check *"did this already run today?"* before firing.** Same pattern at the scheduler layer. If the operator manually triggers what they think is a missed run, the scheduler's day-bucket check might dedupe it. Force path: an admin button that bypasses the day-bucket.

In every case, the rule is the same: **idempotency without a force path is a correctness feature that ships an operator-experience bug.** The fix is structural and small.

The pairing worth naming: this is the *idempotency-layer* version of the companion story `forge-layered-status.md`. Both stories are about state in the routine-execution system that the operator can't see, where the system's behavior is *correct* but the operator-visible signal is *missing* or *wrong*. Both were caught by the same sanity-check pass and fixed in the same commit. Both belong to the same *state opacity* sub-family — the system has more layers than the operator can see, and a layer disagreeing silently with the operator's mental model is the recurring shape.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *idempotency that gates actions the operator might want to re-trigger*. It doesn't apply to:

- **Pure-side-effect-free idempotency.** A read-side cache, a memoized function, a content-hash dedupe in a context where re-running has *zero* side effects (no notifications, no billing, no state mutation downstream). The operator doesn't need a force path because there's nothing to force; the dedupe is a pure performance optimization.
- **Provider-side idempotency you don't control.** Webhook dedupe enforced by the provider before your code sees the request. The provider's dedupe isn't your operator-experience surface — you can't add a force path to it. The fix at your layer is different (alert on dropped events, manual re-fetch from the provider's API).
- **High-frequency programmatic re-triggers.** A retry loop in a worker that dedupes against a job key doesn't need an operator-visible banner — the operator isn't watching individual retries. Reserve the lesson for cases where a *human* might click and need feedback.
- **Idempotency tied to user identity, not action.** Login dedupe, signup dedupe (preventing duplicate accounts) — these have different operator-experience properties. The lesson applies to operator-initiated re-triggers of *system actions*, not to user-experience cases.

The signal: *if an operator clicks a button and the system dedupes silently, will the operator's next action make sense?* If yes, the dedupe is fine. If no (they'll assume something's broken, escalate, or hack around it), you need the force path.

## What This Story Is *Not* Evidence For

- **Not evidence that idempotency is bad.** It's necessary. The bug was the missing operator-visible signal, not the dedupe. Removing the dedupe would have shipped duplicate leadership briefs; that's a worse failure mode.
- **Not evidence that `YYYY-MM-DD` is the wrong dedupe bucket.** It's correct for daily routines. The lesson is about *the UX around the dedupe*, not the bucket's granularity.
- **Not evidence that operators should never re-trigger routines.** They should, often. The force-run path makes that re-trigger explicit and audit-able. The fix enables operator re-triggers, it doesn't suppress them.
- **Not evidence that all idempotency layers need force paths.** Pure-performance idempotency (memoization, read-side caching) doesn't. The signal is *would an operator want to bypass this?* If yes, force path. If no, the dedupe can stay invisible.
- **Not evidence that the standard test suite was inadequate.** The tests covered idempotency-as-correctness — same input, same output. The bug was at the operator-experience layer, which standard tests don't cover. Adding tests for *the banner appears when the dedupe fires* is the structural follow-up; it's a UX test, not a correctness test, and it lives in a different test file.
