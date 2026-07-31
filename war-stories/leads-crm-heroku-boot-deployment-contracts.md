# leads-crm / The Rails App That Ran Until It Deployed

## Date / Version Context

- **Date:** 2026-04-13 — the first Heroku deploy attempt for leads-crm. Several commits landed that day fixing the three boot issues incrementally. The dev environment had been running for weeks (repo start was 2025-11-26); this is the first time production assumptions were tested.
- **Project:** leads-crm — Rails 8 / Postgres / Hotwire / Devise + Pundit / RSpec. Sales-qualification CRM. The Heroku deploy was driven by the broader push to expose the CRM as a tool surface for external agents (the REST API + MCP server work that landed 2026-04-04, three days before the directory-linking commit `1a3669f`).
- **Surface for this story:** the boundary between *the dev environment as the developer sees it* and *the Heroku environment as the production runtime enforces it*. Three boot-time failures, three different sub-surfaces.
- **Glossary, used in this writeup:** *Deployment-only contract* = a behavior or configuration the production environment requires that the dev environment doesn't enforce, leaving a silent gap until first deploy. *Boot-time failure* = a failure that prevents the app from starting, distinct from runtime failures that occur during request handling. *Procfile* = the Heroku-specific declaration of process types (`web: bin/rails server`, `worker: bundle exec sidekiq`, etc.) — the production launcher's contract with the application.

## What Was Being Attempted

Deploy the working dev application to Heroku.

The dev environment was healthy. The Rails app booted with `bin/rails server`, the test suite was green, the MCP server worked locally against the dev database, the REST API endpoints responded. Every internal correctness criterion was satisfied. The deploy was the next logical step — production-running so external agents could call the MCP tools, leadership could use the web UI, and the team could start exercising the system under real load.

Heroku's deploy contract is well-documented but easily under-met. A working Rails app needs to satisfy several invariants at boot time on Heroku:

- The `Procfile` declares the process types Heroku will spawn. If it's missing or has syntax errors, Heroku falls back to a default that may not match the app's needs.
- The seed data has to be able to run in production mode. Validation that's relaxed in dev (or relies on env-specific defaults) fails when production mode is stricter.
- Asset pipelines have to compile cleanly during slug compilation. Missing assets, broken JS bundles, or sass/scss errors fail the build.
- Active Storage has to have a production-grade adapter (S3, GCS, Azure Blob, etc.) — Heroku's filesystem is ephemeral, so the local `:local` adapter doesn't survive a dyno restart.
- Environment variables for secrets, API keys, third-party credentials all have to be set via Heroku config vars; the dev `.env` file doesn't deploy.

Each of these is a *contract the production environment enforces that the dev environment doesn't*. The agent that built the app, working entirely in dev, has no signal that these contracts exist unless the operator names them up front.

## What Went Wrong

Three independent boot failures, one deploy.

The Heroku deploy on 2026-04-13 surfaced all three at roughly the same moment because Heroku attempts to boot the application as part of the deploy process — and the first error halts the boot. The three failures had to be addressed one at a time.

**Failure 1 — seed validation rejected production-environment fixtures.** The `db/seeds.rb` file had been written in dev mode where some validations were skipped or had relaxed defaults. In production mode, validations that touch missing-required-fields, missing-association-targets, or missing-env-var-defaults rejected the seed records. The app couldn't seed the initial data; without seed data, the first deploy couldn't be exercised end-to-end. Local seeds had worked silently because dev provided the gaps the validations cared about (a default org, a default user, env vars from `.env`).

**Failure 2 — Procfile interpolation didn't expand under Heroku's process launcher.** The `Procfile` contained shell-syntax interpolation (e.g., `web: bundle exec rails server -p $PORT -e $RAILS_ENV`) that worked when run from a developer shell but didn't expand under Heroku's launcher in the way the file assumed. The specific syntax mismatch isn't unique to this project — Heroku's launcher has specific rules about variable expansion that differ from a standard shell. The dev environment never tested the Procfile because dev uses `bin/rails server` directly, not via Procfile.

**Failure 3 — Active Storage config was missing the production adapter.** `config/storage.yml` had `local` and `test` adapters defined; production wasn't named. Heroku's filesystem is ephemeral, so even if production had defaulted to local, file uploads would have been lost on every dyno restart. The dev environment used `local` storage and never surfaced the issue because the dev filesystem persists.

Three failures, three different layers, one moment. The dev environment had *worked around all three* by providing defaults the production environment doesn't:

- Dev's relaxed validation context  →  production's stricter mode rejected the seed.
- Dev's shell-direct boot  →  production's Procfile launcher couldn't interpolate.
- Dev's persistent local filesystem  →  production's ephemeral filesystem needed a real storage adapter.

The dev environment is, structurally, a *more permissive* environment. It satisfies invariants the production environment enforces. An app built entirely in dev — by an agent or a human — accumulates dependencies on those default-satisfactions, and the dependencies become visible only when the stricter environment runs the app.

For an agent-built Rails app, the gap is wider. The agent's natural mode of work is *implement features, run tests, ship*. The agent doesn't read deployment-target docs unless prompted; doesn't think about Procfile semantics unless the spec mentions them; doesn't reach for `config/storage.yml` for production-adapter setup because the dev work doesn't require it. The contracts are *visible* in the deployment-target documentation, but they're not in the *immediate work surface* the agent is operating on. Without the operator naming them, they go unmet.

## How It Was Discovered

By Heroku's boot logs, one failure at a time.

The discovery channel is *the deploy command's output*. The first deploy on 2026-04-13 produced a boot failure with a specific error message (the seed validation rejection). The operator read the error, fixed the seed file, redeployed. The redeploy produced the next failure (Procfile interpolation). The operator fixed the Procfile, redeployed. The redeploy produced the third failure (Active Storage adapter). The operator fixed `storage.yml`, redeployed. This time the app booted.

The friction of *three sequential redeploys to surface three independent issues* is the structural property of boot-time failures: they don't compose. The first failure halts the boot; the second failure is invisible until the first is fixed. You can't see your full deployment-debt in one shot; you see it one log line at a time.

The non-discovery channel — and the part worth pausing on — is that *none of these failures would have surfaced from the test suite, the dev server, the CI run, or any agent-driven verification short of an actual deploy attempt.* The test suite ran in test mode, which has its own permissive context (often even more permissive than dev). The dev server didn't use the Procfile. The local Active Storage adapter worked fine. The standard agentic-development verification stack was *blind* to the deployment-time contract.

This makes the discovery channel *deployment itself*. The orchestrator who treats deployment as *"a thing that happens after the app works"* misses that deployment is the first time the production environment's contract is enforced. The orchestrator who treats deployment as *"the first integration test against the production environment"* expects deploy-time discoveries and budgets for them.

## What Fixed It

Three commits, one Procfile syntax change, one `storage.yml` addition, one seed adjustment. Each fix was small once the failure was named.

The structural shape of the fixes:

**Seed validation fix.** The seed file was adjusted to satisfy production-mode validations directly rather than relying on dev defaults — explicit field values where dev had defaulted, explicit env-var fallbacks where dev had `.env`, explicit org/user creation where dev had been seeded from a fixture. The seed now runs in any environment.

**Procfile interpolation fix.** The Procfile syntax was rewritten to match Heroku's launcher expectations — typically dropping the shell-direct invocation in favor of a launcher-compatible form. The specific syntax change is well-documented in Heroku's Procfile guide; the issue was simply that the original Procfile had been written from dev-shell intuition rather than from Heroku's reference.

**Active Storage adapter fix.** `config/storage.yml` got a `production:` block with the chosen production adapter (an S3-compatible service in this case), and Heroku config vars got the corresponding credentials. The app now has explicit production storage, not a fallback.

What's load-bearing isn't the individual fixes — they're standard Rails-on-Heroku gotchas. What's load-bearing is *the orchestration discipline that catches them earlier next time.* The discipline that emerged: **deploy-as-first-integration-test**. The next feature that significantly affects the deployment shape (a new dependency, a new asset type, a new external service) is preceded by a deploy attempt before the feature is considered complete. The orchestration plan includes *deploy* as a phase, not as an after-the-fact step.

What's owed but not yet shipped: a `.specify/deployment-checklist.md` template that the spec-kit flow reads before a feature is considered ready to merge — a list of deployment contracts the operator confirms have been considered. Lives in operator habit at the moment; worth writing down.

## The Durable Lesson

An agent can build a locally coherent Rails app while still missing deployment-only contracts. **The dev environment is structurally more permissive than the production environment, and the agent's natural completion criterion (*the app runs and the tests pass*) has no signal for the gap.**

The mental-model flip the lesson rests on: *the deploy* is not *"running the app in a different place"*; it's *"the first time the production environment's contract is enforced on the application."* The dev environment is a working approximation of the production environment, but the approximation has known asymmetries — relaxed validations, persistent filesystem, dev-shell semantics, fallback defaults — that the production environment doesn't honor. Building entirely in dev accumulates dependencies on those asymmetries; deploying surfaces them all at once.

For agentic systems specifically, the cost compounds. The agent's verification stack (run the tests, exercise the dev server, ship the MCP tools locally) is fully inside the dev environment's permissive context. The agent has no automatic signal for *"this works because dev provides X; production won't provide X."* The signal has to come from the operator, and the cheapest form of that signal is **deploy early, deploy often, treat deploy as the first integration test**.

> **Heuristic.** For any Rails (or similar full-stack framework) project with a known production target, attempt a deploy *before* the first feature is considered complete. The deploy doesn't have to succeed — it has to surface the production-environment contract gaps. Treat the deploy log as the first integration test; iterate on the gaps until the app boots clean. Once the boot baseline is established, every significant feature gets a deploy attempt as part of its definition of done. The cost of the discipline is small (Heroku deploys are minutes); the cost of skipping it is *every deployment-only contract surfaces at the worst possible moment* — usually right before a launch, often blocking a demo, occasionally taking down production.

The shape generalizes beyond Rails-on-Heroku:

- **Node.js on Vercel / Render / Cloud Run.** The deployment target's process model differs from local `npm run dev`; cold-start semantics, environment-variable surfaces, ephemeral filesystems all surface only on first deploy.
- **Python on Lambda / Cloud Functions.** Cold-start cost, package size limits, ephemeral filesystem, container-mode vs. function-mode contracts.
- **Go binaries on Kubernetes.** Health-check semantics, graceful-shutdown signals, config-mounting via ConfigMap/Secret — all production-only contracts the dev binary doesn't exercise.
- **Static site generators on Netlify / Cloudflare Pages.** Build-time environment differs from dev-server environment; redirects, headers, function bindings all evaluated only at deploy.

In every case, the pattern is identical: **the production runtime enforces contracts the dev runtime doesn't, and the only way to surface the gap is to attempt the deploy.** The lesson is *deploy attempts are integration tests; budget for them, run them early, iterate.*

The pairing worth naming: this is the *deployment-shaped* version of the companion story `forge-async-adapter-smoke-test.md` — both stories are *the dev environment's permissive defaults hide production-environment contracts*. There, the missed contract was the queue adapter's lifecycle (dev's `:async` returns immediately; production's `:sidekiq` enforces durability). For this story, the missed contracts were three at once (seed validation, Procfile semantics, Active Storage adapter). Together they argue for a pre-flight *adapter parity / deployment parity* check as a load-bearing orchestration discipline, not a sidebar. See also the companion story `leads-crm-directory-linking-backfill.md` on *the agent's natural completion criterion doesn't include what the spec doesn't name* — there the missing piece was existing-data recovery; here it's deployment-environment parity. Both are *spec-doesn't-cover-the-world* stories with leads-crm-shaped specifics.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *production-environment contracts that aren't enforced in dev*. It doesn't apply to:

- **Apps with dev/prod parity from the start.** Some teams run Heroku-shaped dev environments (Docker Compose mimicking the production runtime, dev databases in Postgres rather than SQLite, dev workers running Sidekiq rather than `:inline`). Those teams pay the parity cost upfront and have fewer deploy-time surprises. The lesson is for projects that *don't* have parity, which is most projects most of the time.
- **Single-user local-only tools.** A CLI, a desktop app, an unhosted utility — these don't have a *deploy* in the relevant sense. The lesson is for apps with a remote production target.
- **Apps deployed to a fully-replicated staging environment.** Some shops have a staging environment that *is* the production environment, just with different data. Deploying to staging is the integration test; the lesson is mostly already practiced. The lesson is for projects without such an environment — which, for small teams and agentic builds, is most of them.
- **Trivial config changes.** A one-line bug fix doesn't need a fresh deploy-as-integration-test. The lesson kicks in for changes that meaningfully affect the deployment shape — new dependencies, new asset types, new external services, new env vars.
- **Pre-existing apps with a long deploy history.** An app that's been deploying to Heroku for years has already paid most of the deploy-time-contract cost; new features add small deltas. The lesson is highest-leverage for *first deploys* and for *changes to the deployment shape*.

The signal: *can the failure surface in dev?* If yes, dev tests catch it. If no (filesystem ephemerality, launcher semantics, production-only env vars, asset-pipeline contracts), only the deploy will.

## What This Story Is *Not* Evidence For

- **Not evidence that Heroku is uniquely demanding.** The three failures are standard production-environment contracts that *every* deployment target enforces; Heroku's are just well-documented. The lesson applies to any production target.
- **Not evidence that agent-built Rails apps are uniquely fragile.** Human-built Rails apps hit the same boot failures the first time they deploy to a new target. The lesson is about *agentic acceleration of the gap*, not about agents producing worse code.
- **Not evidence that dev/prod parity is the only answer.** Full parity is expensive; deploy-early discipline is cheap. The lesson is *make the deploy itself an integration test*, not *replicate production in dev*.
- **Not evidence that the existing test suite was inadequate.** The tests covered runtime behavior. The bugs are at the *deploy time* layer, which standard tests don't model. The structural follow-up is the deploy-checklist habit, not more tests.
- **Not evidence that each of the three failures was severe in isolation.** Individually they're standard config one-liners. The lesson is the *aggregate pattern* — three independent boot failures from one moment, all from the same root cause (the dev environment is more permissive than production).
