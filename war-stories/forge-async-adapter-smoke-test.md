# forge / The Runner That Returned Before the Job

## Date / Version Context

- **Date:** The behavior is structural to Rails' default dev adapter — there's no specific incident-day commit because the loss is silent. The pattern got named during the 2026-05-09 self-interview from accumulated smoke-test friction across the project's 2026-04 to 2026-05 build arc. Production launched 2026-04-29; the smoke-test friction predates and postdates the launch.
- **Project:** Forge — Ruby on Rails 8 / Postgres / Sidekiq / Heroku Eco. Production `queue_adapter` is `:sidekiq` (Heroku worker dyno running `bundle exec sidekiq`). The dev `queue_adapter` defaults to `:async` unless explicitly overridden in `config/environments/development.rb` — which Forge did not do in the early build arc.
- **Surface for this story:** `bin/rails runner` invocations used as smoke-tests. The `RoutineRunner` and `TaskRouter` pipeline both call `perform_later` heavily — `RunTaskJob.perform_later(task.id)` is the load-bearing line that hands off execution from the synchronous routing layer to the async execution layer. Half the codebase's interesting paths route through `perform_later` somewhere.
- **Adjacent surface:** rake tasks. Same shape — a rake task that calls `perform_later` and exits has the same dev-only silent-drop behavior because the rake process also exits before AsyncAdapter flushes.
- **Glossary, used in this writeup:** *Queue adapter* = the `ActiveJob` backend that turns `perform_later` calls into actual job execution. Rails ships several: `:inline` (run synchronously in the calling process), `:async` (enqueue on a thread pool inside the calling process), `:sidekiq` (push to Redis, Sidekiq workers execute). *Smoke test* = a fast end-to-end probe an operator runs locally to confirm a code change didn't break the production path. *Adapter parity* = the property that dev and prod adapters have the same lifecycle semantics — specifically, that *"`perform_later` returned"* means *"the job will run"* in both environments.

## What Was Being Attempted

Verify a routine works locally before pushing.

The orchestrator's loop: change `RoutineRunner` or a capability handler, then probe the change end-to-end with `bin/rails runner "ScheduledRoutine.find_by(name: 'leadership-brief').run_now"`. The expected behavior: routine fires, `TaskRouter` enqueues `RunTaskJob`, the job runs the capability, the capability calls Claude, the brief gets written, the artifact persists, the operator can `tail -f log/development.log` and see the whole chain.

This is the cheap-smoke-test discipline the project leaned on heavily. Heroku CI round-trips are slow. The dev loop has to be tight or the build pace breaks. `bin/rails runner` is the natural fit — load Rails once, fire one routine, see what happens. No web server, no Sidekiq process, no Redis. Just Ruby and Rails and the routine's code path.

The expected output: log lines showing the routine firing, the task being created, the job picking it up, the capability running, the artifact landing.

## What Went Wrong

The runner returned before any of that happened.

`bin/rails runner` is a one-shot process. It loads Rails, evaluates the expression, exits. `ScheduledRoutine#run_now` calls `TaskRouter.call`, which creates a `Task` row and calls `RunTaskJob.perform_later(task.id)`. `perform_later` returns truthy — the job has been "enqueued." The expression evaluates. The runner returns. The runner exits.

`AsyncAdapter`, by design, enqueues the job onto an in-process thread pool and lets the calling thread continue. When the calling thread is a long-lived server process (Puma, a Rails console, a Sidekiq worker), the thread pool has time to drain and the job runs. When the calling thread is `rails runner`, the calling thread *is* the process. Process exit kills the thread pool. The job's enqueue record was an in-memory queue entry on a thread pool that no longer exists.

The job is gone. No error. No warning. No log line. `perform_later` returned a job handle; the runner returned cleanly; the operator sees a clean exit. The task row is still in the database with `status: :queued`, frozen forever because the job that would advance it through `running → succeeded` never started.

The operator's mental model: *I ran the routine. The runner exited cleanly. The routine fired.* The system's actual behavior: *You enqueued a job onto a thread pool that died before the job could start. The task row says `queued` because the work was queued. The work was queued and lost.*

The deceptive part is that none of the in-process steps lied. `TaskRouter.call` did its job (the row was created with the correct fields). `perform_later` did its job (the job handle was returned). `AsyncAdapter#enqueue` did its job (the job was placed on the pool). The breakage is between the pool and the process exit — and the lifecycle semantics there are *not* what the high-level call sites read like.

Production Sidekiq has none of this problem. The Sidekiq process is long-lived; it pulls jobs from Redis, executes them, marks them complete. `perform_later` in production pushes to Redis, where the job is durable, where a different process owns its execution lifecycle. The whole class of process-exit-during-job-run failures doesn't exist on production. It exists in dev because dev defaults to an adapter whose lifecycle assumes the calling process will outlive the work.

The asymmetry is the bug. The smoke test was probing the dev adapter's behavior, not production's. The dev adapter's behavior is *enqueue-and-pray-the-caller-stays-alive*; production's is *enqueue-and-Sidekiq-will-pick-it-up-eventually*. Same `perform_later` call site, two different lifecycle contracts.

## How It Was Discovered

Slowly, across many smoke tests, each one looking weirder than the last.

There is no single-incident moment for this one. The discovery channel is *accumulated friction* — runs that should have produced log output produced none, task rows that should have advanced stayed at `queued`, capability code that should have hit a breakpoint never did. The operator's first reflex on each instance is *the code I just edited must be broken*. The second reflex, after re-reading the diff and finding it sound, is *let me run it again, maybe Sidekiq blipped*. The third, after the third silent run, is *something underneath this is lying to me*.

The pattern crystallized into *"smoke-testing a routine locally with `runner` would silently lose the enqueue because the runner process exits before `AsyncAdapter` flushes"* when the operator stepped back from the per-incident frustration and asked the structural question: *what is `bin/rails runner` actually doing with `perform_later` calls?* The answer — *it doesn't wait for them to flush before exiting* — explained every silent smoke test in retrospect.

This is the discovery channel worth pausing on. There was no production incident here. No customer impact. No alert. The only signal was *my smoke tests aren't behaving the way I expect them to*, repeated across enough instances that the pattern named itself. A team where the operator pushed every change to staging and trusted CI to catch it would have shipped a thousand routine modifications without ever realizing the local smoke-test was theater. The operator's *I expected this to work, why didn't it* reflex is the discovery surface; without that reflex, the bug is invisible because the bug never fires in production.

## What Fixed It

A configuration change with a one-line shape. Several options, all structurally equivalent:

**Option A — `:inline` adapter in development.** Add `config.active_job.queue_adapter = :inline` to `config/environments/development.rb`. Every `perform_later` runs synchronously in the calling process. `bin/rails runner` blocks until the job completes; the runner exits after the work is done. The cost: synchronous behavior in dev means dev-time UX reflects synchronous work, which can hide async-only bugs (jobs landing in unexpected order, jobs that race with the request that enqueued them). For a routine-execution system where the production behavior is *async with durable queueing*, `:inline` is a useful smoke-test default but not a faithful dev environment.

**Option B — `:sidekiq` adapter in development with a local Redis.** Run Redis locally (`brew services start redis`), point dev at it, run `bundle exec sidekiq` in a second terminal. Dev now has full Sidekiq parity. `perform_later` from `rails runner` pushes to Redis, the local Sidekiq picks it up, the job runs. The cost: dev now requires Redis and Sidekiq running — more moving parts, more startup friction, more *something's not running* failure modes. Worth the cost when the project's interesting behavior is async-shape and dev needs to reflect that.

**Option C — `perform_now` explicitly in smoke-test scripts.** Skip the adapter question entirely; write the smoke test to call `RunTaskJob.new.perform(task.id)` instead of `perform_later`. The smoke test now bypasses the queue and runs the job inline regardless of adapter. The cost: the smoke test is no longer probing the enqueue path — it's probing only the execution path. The two are usually fine together, but if the bug is in *what `perform_later` sees when called from `TaskRouter`*, the `perform_now` smoke test won't catch it.

**Option D — flush the adapter before exit.** In `bin/rails runner` invocations, append `Rails.application.executor.shutdown` or an equivalent drain call. The runner waits for the pool to flush before exiting. The cost: more verbose smoke-test invocations, easier to forget, and the failure mode when forgotten is the same silent-drop.

The structural difference: Options A and B fix the *adapter mismatch* directly. C and D work around it at the call site. A and B are project-level fixes (one config line, applies everywhere). C and D are script-level (every smoke-test author has to remember).

The lesson is one level above the option choice: *pick an option, write it down, and make the dev environment's adapter contract explicit*. The default `:async` is the silent killer — not because async is wrong, but because the dev adapter's lifecycle contract assumes a long-lived calling process and `rails runner` is a short-lived one, and Rails doesn't warn you about the mismatch.

What's owed but not yet shipped (across most Rails projects that hit this): a `rails runner` extension or wrapper that emits a warning when `perform_later` is called and the configured adapter is `:async`. Something like *"Warning: you called `perform_later` from `rails runner` with `queue_adapter: :async`. The job may not execute before the runner exits. Consider `:inline` or `perform_now` for runner-shaped invocations."* This belongs upstream in Rails itself; until it lands, the discipline lives in project-level configuration and operator habit.

## The Durable Lesson

Dev/prod job-adapter parity is not a "nice to have" when half your codebase is `perform_later` from runners and rake tasks.

More structurally: **the smoke-test environment must match the production environment for the surfaces under test, or the smoke test is testing the wrong environment.** The operator's confidence in a smoke test is calibrated against the smoke-test environment's behavior. When that environment's behavior diverges from production in lifecycle semantics — *what happens when the calling process exits while work is pending* is exactly the kind of lifecycle divergence that bites — the operator's confidence is calibrated against the wrong thing.

The mental-model flip: `perform_later` is not a uniform abstraction. Its lifecycle contract depends on the adapter, and the adapters Rails ships have *different* lifecycle contracts. `:inline` runs the job before returning. `:async` returns immediately and runs the job later in the same process. `:sidekiq` returns immediately and runs the job later in a different process. The call site is identical; the contracts diverge at the lifecycle layer the call site can't see.

For agentic systems specifically, the cost compounds. Routines, capabilities, retries, error-summarization loops — all of them route through `perform_later` somewhere. The orchestration design assumes the enqueue is reliable. The smoke-test loop assumes the smoke test exercises the real path. When the dev adapter quietly drops enqueues, the orchestration design's most load-bearing assumption is silently false in the environment where the operator is iterating fastest.

> **Heuristic.** For any production system that uses async job execution, name the adapter contract in writing, set the dev adapter to match production's lifecycle semantics (`:inline` for synchronous fidelity, `:sidekiq` with local Redis for full parity), and never trust `bin/rails runner` with `:async` as a smoke-test surface. If the project's interesting behavior is async-shape, run dev async too. If the project's interesting behavior is the work itself (not the queueing), `:inline` is fine. The choice should be deliberate; the default `:async` is the silent killer because it looks asynchronous in the call site and behaves synchronously in the long-lived web process and not-at-all in the short-lived runner process. Three behaviors, one configuration value, no warning when you pick the wrong one for the context.

The shape generalizes far beyond Rails:

- **Node.js fire-and-forget Promises in CLI scripts.** A script that fires `doWork()` without `await`, then returns from `main()`. The process exits, the microtask queue is dropped, the work never happens. Same shape: short-lived caller, async work, no lifecycle handshake.
- **Goroutines launched without a `sync.WaitGroup`.** Main goroutine returns, the runtime kills outstanding goroutines, the work is silently lost.
- **Python `threading.Thread` without `join()`.** Daemon threads die when the main thread exits. Non-daemon threads block exit but only until they finish; if `main()` is `return`ing on first error, daemon-thread work is dropped.
- **Background tasks in cloud functions.** AWS Lambda / Cloud Run / Cloud Functions kill the container when the handler returns. Any work started with `setTimeout`, `setImmediate`, a Promise without `await`, a goroutine, a thread — gone. Same shape, harder to debug because the container is ephemeral.

In every case, the pattern is identical: **a short-lived calling context that spawns async work without a handshake guaranteeing the work's lifecycle outlives the caller's lifecycle.** The fix is structural: either the spawn returns a handle the caller awaits before exiting, or the work is pushed to a durable queue outside the caller's process before the spawn returns.

The pairing worth naming: this is the *environment-lifecycle* version of the *adapter-doesn't-mean-what-the-call-site-reads-like* pattern. Pairs structurally with the companion story `forge-prompt-cache-minimum.md` — both are *the SDK accepted the call; the infrastructure silently no-op'd*. Different layers (provider API vs. dev/prod adapter mismatch), same shape (*function returned ≠ work happened*). Pairs also with the companion story `forge-layered-status.md` on *the routing layer and the execution layer have different lifecycle contracts*.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *async job enqueues from short-lived processes against adapters whose lifecycle assumes a long-lived caller*. It doesn't apply to:

- **Long-lived dev processes.** A Rails server in development (`bin/rails server`) running with `:async` adapter is fine — the server process outlives any reasonable job's pool-drain time. The bug only fires when the calling process is short-lived. Web requests, Rails console sessions, and Sidekiq workers don't trigger it.
- **Synchronous-by-design infrastructure.** A script that calls `MyJob.perform_now` instead of `perform_later` is doing synchronous work explicitly. The adapter doesn't matter because the job runs in-process before the call returns. No lifecycle question.
- **Fully-durable queue adapters.** If dev uses `:sidekiq` (or `:resque`, or `:good_job`, or `:solid_queue` — any adapter where the enqueue persists to durable storage before the call returns), the calling process can exit safely. The work is durable; some other process will pick it up. The bug is specific to adapters that hold the queue in the calling process's memory.
- **Tests that explicitly assert on the enqueue, not the execution.** A spec that does `expect { ... }.to have_enqueued_job(MyJob).with(args)` is testing the enqueue, not the execution. Rails' test adapter (`:test`) tracks enqueues without running them. The behavior is intentional; the test is not a smoke test.
- **Production-only async surfaces.** Some teams treat dev as inline-only and only exercise the async path in CI or staging. That's a deliberate choice; the lesson still applies but the fix is "don't `runner` async work in dev, period" rather than "match the adapter contract."

The signal: *is the calling process expected to outlive the work it spawns?* If yes (web servers, Sidekiq workers, long-lived consoles), the dev adapter's lifecycle assumption holds. If no (runners, rake tasks, CLI scripts, cloud function handlers), the lifecycle assumption breaks and the work needs to be either pushed to durable storage before return or executed synchronously in-process.

## What This Story Is *Not* Evidence For

- **Not evidence that `AsyncAdapter` is bad.** It's reasonable for the in-process web-server case it was designed for. The bug is the mismatch between the dev default and `rails runner`'s lifecycle, not the adapter itself.
- **Not evidence that `bin/rails runner` is bad.** It's the right tool for short-lived Rails invocations — rake-task-shaped work, one-shot scripts, smoke tests. The bug is using it with `perform_later` against an in-process adapter.
- **Not evidence that all dev environments need full Sidekiq.** Many projects are fine with `:inline` in dev. The choice depends on whether the project's interesting behavior is async-shape. The lesson is *pick deliberately, don't accept the default*.
- **Not evidence that Rails should change the default.** Reasonable people disagree on the right default for `Rails.application.config.active_job.queue_adapter` in development. The lesson is project-level — name your project's needs, configure accordingly — not framework-level.
- **Not evidence that smoke tests are useless.** They're load-bearing for tight dev loops. The lesson is *the smoke test has to exercise the same lifecycle as production for the surfaces under test*, not *skip the smoke test*. A smoke test that probes the adapter mismatch and shows it fires (e.g. confirms the job ran by checking the task's `status` after a short sleep) is exactly the discipline the lesson asks for.
- **Not evidence that the test suite was inadequate.** The test suite tested the job's logic — given input, the job produces the right output. That coverage is correct. The bug is at the *invocation* layer, which standard tests don't exercise because the test adapter (`:test`) has neither dev's nor production's lifecycle contract. The structural follow-up is a smoke-test discipline — *every routine has a one-line runner invocation that confirms end-to-end execution in dev* — and a dev environment configured so that invocation behaves like production.
