# forge / The Gate That Hasn't Fired

## Date / Version Context

- **Date:** Latent throughout the project's pre-launch and early-production arc (2026-04 through present). The gap was named in the 2026-05-09 self-interview, not by any production signal.
- **Project:** Forge — Ruby on Rails 8 / Postgres / Sidekiq, Heroku-deployed. Virtual Employees execute via `RoutineRunner` → `TaskRouter` → capability handler → `Artifact` rows → delivery. Production launched 2026-04-29.
- **Surface for this story:** the access-control middleware between `TaskRouter` and the capability handler. Specifically, the redaction branch inside the Ops Analyst capability's policy check — the path that runs when a request *passes* the user-existence check but *fails* the sensitivity check.
- **Glossary, used in this writeup:** *Spec-covered* = the code path has unit/integration tests that exercise it and assert correctness. *Production-exercised* = the code path has actually run under at least one real request with real principal context, real auth state, and real downstream consumption. *Gated sensitivity* = a `default_sensitivity` field on a capability that determines who is allowed to see its full output vs. a redacted version. *Low-load gate* = a security control in a system whose production traffic is small enough that the control rarely or never fires.

## What Was Being Attempted

Ship the Ops Analyst capability with a redaction policy for sensitive output.

The design was conservative and sensible. Capabilities tag their output with a `default_sensitivity` level. The access-control middleware checks the requesting user against the capability's policy. Leaders see everything. Non-leaders, on capabilities with elevated sensitivity, see a redacted view — names of sections preserved, sensitive numbers and identifiers removed.

The policy got spec coverage early. Unit tests exercise the redaction branch with synthetic principals (a leader, a non-leader), confirm that the leader sees the full output and the non-leader sees the redacted view, and verify that the redaction logic doesn't leak fields it's supposed to mask. Every test passes. The redaction shows green on every CI run.

The intent of the spec coverage was honest: prove the redaction *works* before shipping. By the standard of *unit tests on the redaction logic*, the gate is verified. By the standard of *the gate has fired under a real request*, the gate is — to this day — untested.

## What Went Wrong

Nothing has gone wrong, and that's the story.

Two production conditions stack to keep the redaction path silent:

**Condition 1 — Forge's user population is tiny.** Production has ~5 leadership users. Leadership requests hit the *leader* branch of the policy, which returns the full output. The redaction branch is reachable only when a non-leader makes a request. In production, that almost never happens.

**Condition 2 — An earlier gate denies non-leaders before redaction runs.** The access-control middleware checks user existence and authorization order: *does this user exist in the system at all?* → *do they have any access to this capability?* → *what sensitivity level?* Non-leaders in production are usually not in the user table at all, because the system has been seeded with leadership accounts only. They hit *unknown user denial* and return with a generic auth error. The redaction branch — which runs only when a known non-leader requests a sensitive capability — is downstream of a check that filters out almost every realistic path to it.

The combined effect: the redaction code has never run in production. Not once. Every spec passes. No real request has tested the gate's behavior under live conditions — the actual session middleware, the actual logger, the actual downstream artifact serialization, the actual Slack rendering of a redacted artifact.

The gate *might* fire correctly the first time it runs. Or it might fail in any of several ways that specs can't see:

- A logger interceptor injects sensitive fields back into output before render.
- The artifact serializer flattens the redacted view into a shape that exposes what redaction stripped.
- The Slack delivery boundary's `mrkdwn` translation doesn't know about the redacted-view shape and renders raw redaction markers.
- An audit log captures the full pre-redaction output for "compliance" and the redaction never reaches that surface.

Each of these has happened in production systems we've seen. None would be caught by the unit specs. All would surface the first time a real non-leader hits a gated capability — which, in Forge's current production traffic shape, is approximately *never*.

## How It Was Discovered

By being named in a self-interview, not by any production signal.

This is the discovery channel worth pausing on. The standard discovery channel for security controls — *the control fires, somebody notices something's off, the bug surfaces* — is not available here. The control hasn't fired. There's nothing for anyone to notice.

The 2026-05-09 self-interview surfaced the gap by asking the question *which production-exercised paths give us confidence the system behaves correctly?* and noticing that the redaction path didn't appear on the list. The spec coverage was real. The CI signal was green. The production evidence was zero.

This is structurally identical to the failure pattern *expected-event alerting* defends against: **a state the system should have reached and didn't.** The state, here, is *the redaction code path has fired under a real request*. The alert isn't *the redaction failed*; it's *the redaction has not been exercised in N days/weeks*.

A team reading their dashboard would see normal traffic, no errors, all green. The gap is *what's not in the dashboard* — the absence of redaction-fire events. That gap is invisible until somebody asks the right question.

## What Fixed It

Nothing yet, and that's worth being honest about. The discipline moves are queued, not landed:

1. **A scheduled production-exercise test.** Cron-fire a synthetic non-leader request against a gated capability on a low-traffic cadence (daily? weekly?). The request runs through the full middleware stack, exercises the redaction branch, asserts on the *delivered* output (not just the policy's internal return value). This is closer to a *synthetic transaction* in the SRE sense than a unit test — it's a production probe that confirms the gate fires correctly *in the production environment*, with all its real adapters and downstream consumers.

2. **Expected-event alerting on the redaction code path.** If the synthetic-transaction probe is firing the gate daily, the absence of *any* redaction-fire event over a window (other than the probe) is a different signal: *real non-leader traffic is being denied earlier than the redaction layer.* Worth knowing — it confirms the architecture's intent but also confirms there's no live evidence the gate works for any real principal beyond the probe.

3. **A first-fire audit checklist.** The first time the gate fires for a real principal (not the synthetic probe), an audit runs: did the artifact get redacted? Did the audit log capture the right shape? Did the Slack render look right to the requesting user? Did any downstream system retain the unredacted view? This is the *production-readiness* version of the spec — the questions that only have honest answers once a real request has flowed through.

What didn't get attempted: trying to assert the redaction works without ever exercising it. The lesson rejects that direction. The spec is necessary; it isn't sufficient. *A security control you haven't seen fire is a security control you haven't seen fail.*

## The Durable Lesson

A gate that has never fired in production is a gate whose behavior under production conditions is unknown — regardless of how thoroughly it's spec-covered. Specs prove the gate is *internally consistent*; only firing the gate under real conditions proves it's *operationally correct*.

This is the runtime form of *the trust signal lives outside the claim*. The spec is the agent's-stopping equivalent for control code: it's the system saying *I produced a correct redaction*, evaluated against the system's own model of what correct means. The trust signal is something else: the gate firing under a real request, the audit confirming the artifact downstream matches what the policy promised, the requesting user seeing the redacted output and (ideally) it being indistinguishable from a permissions-deny in any way that would leak the existence of unredacted content.

> **Heuristic.** For any security control, gating logic, or policy enforcement: name the *production-fire condition*, count how often it fires in real traffic, and audit the gate's first real fire after any related code change. If the count is zero over your validation window, the spec is the only thing keeping you honest, and the spec doesn't know what the spec doesn't know. The fix isn't more spec coverage; it's a synthetic-transaction probe that exercises the gate end-to-end, in production, on a cadence you can alert against.

The shape generalizes beyond redaction:

- **Rate-limiting that's never been hit.** A rate limit in a system that never reaches the limit hasn't been exercised; the back-off code path, the retry semantics, and the user-visible behavior are all theoretical until they fire.
- **Circuit breakers in services with high uptime.** A breaker that hasn't tripped in 18 months works *if* its trip condition is configured correctly. Without a fire-drill, you don't know whether it does.
- **Fallback paths in degradation modes.** A read-replica fallback, a stale-cache fallback, a feature-flag-off fallback — all are spec-tested and production-untested in mature systems. The first real degradation event is the first real test.
- **Audit-log redaction in compliance-bound systems.** Same shape as Forge's gate but at the log layer. PII redaction in logs is spec-tested; the auditor's first real review under a regulator's request is the first real fire.

In every case the lesson is the same: **a control whose behavior is only proven by specs has a hidden surface — the gap between *the spec is green* and *the control has fired under live conditions*.** That gap is invisible until somebody asks. Synthetic transactions, fire-drills, and first-fire audits are how the gap gets closed.

The pairing worth naming: this is the *security-controls* version of `forge-prompt-cache-minimum.md`. The cache-that-wasn't was wiring claiming success while the provider silently declined. The redaction-that-hasn't-fired is a spec claiming correctness while the control hasn't been exercised. Both are *the system has a claim layer and a reality layer, and the gap between them is invisible without explicit verification*. Cache is verified by reading the response counters; redaction is verified by exercising the gate. Different mechanisms, same underlying discipline.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *gates whose production-fire count is low or zero in real traffic*. It doesn't apply to:

- **High-traffic gates that fire constantly.** A rate limit that trips a hundred times a day is exercised every day; the spec coverage plus the production trip rate together are sufficient evidence. Synthetic transactions would be redundant.
- **Gates whose mis-fire cost is bounded and recoverable.** A non-load-bearing feature flag, a cosmetic gating decision, an A/B-test branch — if the cost of the gate firing wrong is small and reversible, the spec-only verification is honest enough. Reserve the discipline for controls where the cost-of-being-wrong is real.
- **Pre-launch systems with no production traffic yet.** A spec-covered, never-fired gate in a system that hasn't shipped doesn't need a synthetic-transaction probe today; it needs one as part of the launch checklist. The discipline matters once there's production traffic to compare against.
- **Throwaway internal tooling.** A control in a CLI tool used by three engineers who would notice immediately if it misbehaved doesn't need production-exercise evidence. The audience is the validation.

The signal: *if this gate quietly stopped firing correctly tomorrow, how would anyone find out?* If the answer is *a regulator, a customer, or a security incident*, the cost-of-being-wrong justifies the synthetic-transaction discipline. If the answer is *a developer notices in real time*, the spec is enough.

## What This Story Is *Not* Evidence For

- **Not evidence that the redaction is broken.** No production signal supports that claim. The lesson is about *unverified*, not *failed*. The redaction might fire correctly the first time it runs; nobody can honestly say so today.
- **Not evidence that spec coverage is wasted.** The specs are necessary. They prove the gate is internally consistent and they catch the regressions specs are designed to catch. The lesson is that *necessary* isn't *sufficient* — adding the production-exercise discipline doesn't replace the specs, it complements them.
- **Not evidence that low-load systems can't ship security controls.** They can; they have to. The lesson is about *which validation signals are available* in a low-load system, not about whether to ship controls. Low-load systems need synthetic-transaction probes precisely because organic-traffic validation isn't available.
- **Not evidence that the audience problem is unique to Forge.** Most internal tools have small user populations. Most security controls in internal systems are spec-tested and production-untested for the same reason. The lesson is general; Forge happens to be the anchoring incident.
- **Not evidence that adding more spec coverage helps.** The gap isn't in the spec's coverage; it's in the difference between *spec passing* and *control firing live*. Adding integration tests inside CI doesn't close the gap — those tests don't run in production with production middleware, production loggers, and production downstream consumers. The synthetic-transaction probe runs *in production, against production*. That's the load-bearing distinction.
