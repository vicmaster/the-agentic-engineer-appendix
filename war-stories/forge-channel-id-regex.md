# forge / The Regex That Made Up Slack's Contract

## Date / Version Context

- **Date:** Fixed in PR #41, merged 2026-05-08. The regex itself shipped much earlier — somewhere in the Slack-integration arc of 2026-04. The gap between *shipped with a bug* and *surfaced the bug* was at least three weeks; the second team onboarding was what triggered the surfacing.
- **Project:** Forge — Ruby on Rails 8 / Postgres / Sidekiq / Heroku. The engagement validator lives in the admin UI flow: admins configuring a new VE engagement enter the Slack channel ID where the VE should post; the validator runs server-side on form submission; rejection produces a "this doesn't look like a Slack channel ID" inline error.
- **Surface for this story:** the `Engagement` form's `slack_channel_id` field validation. Pre-fix: regex `/^C[A-Z0-9]+$/`. Post-fix: regex `/^[CG][A-Z0-9]+$/` plus a spec exercising both prefixes against representative IDs.
- **Glossary, used in this writeup:** *Synthetic validation* = a deterministic check whose correctness is measured against test fixtures the developer wrote, not against the external system the check is meant to model. *External oracle* = the system that actually decides whether a value is valid (here: Slack's API). *Resolve-at-source* = the discipline of asking the external system whether a value is valid rather than guessing from format alone.

## What Was Being Attempted

Validate Slack channel IDs at form submission so admins get fast inline feedback.

The motivation was reasonable. The form had a free-text input where admins paste a Slack channel ID. A typo in that field would produce a downstream `channel_not_found` error from Slack's API hours later when the VE first tried to post — not at form submission. A client-side or server-side validator could catch typos immediately, save the round-trip, and give the admin a usable error before the data even hit the database.

The implementation was straightforward. Slack channel IDs follow a known format: a `C` prefix followed by uppercase alphanumeric characters. The regex `/^C[A-Z0-9]+$/` matched. The team wrote the regex from memory; the fixtures in the spec were `Csomething`, `C123ABC`, `CABC456`. Every fixture passed. Every fixture failed when prefixed with the wrong letter. The validator was correct against its own test data.

It shipped. The first team onboarding worked. The validator did its job — caught one typo during setup, displayed a friendly error, the admin corrected the channel ID, the engagement saved. The team felt good about the validator. Three weeks went by.

## What Went Wrong

The second team's announcements channel had ID `G…`, not `C…`.

The admin pasted the ID into the form. The validator rejected it. The inline error read *"This doesn't look like a Slack channel ID — please check the value."* The admin stared at the screen. They knew the ID was correct. They had just copied it from Slack's "Copy Channel ID" affordance. They re-pasted. Same error. They tried a different channel — the team's general channel — `C…` prefix, accepted. The general channel was the wrong destination; the engagement was for announcements, and announcements was `G…`.

The admin escalated. The escalation path was *Slack DM the Forge team, ask what's going on.* The Forge team's first instinct was *the admin pasted the wrong thing.* Twenty minutes of back-and-forth later, somebody on the Forge side actually clicked into Slack's API docs and noticed the entry that wasn't in the team's memory: **Slack issues `G…` IDs for legacy private channels** — the prefix is short for "Group," the format Slack used before private channels migrated to the unified channel API. The IDs are stable; channels that were created before the migration still carry their `G…` prefix forever. There's no automatic upgrade path; Slack's docs note the variation in a sentence most people scan past.

The bug shape:

- **The regex was correct against its mental model** of Slack's channel ID format. The mental model was wrong.
- **The tests passed** because the fixtures were generated from the same mental model. The fixtures were synthetic — written by someone who knew what they expected channel IDs to look like, not pulled from a real list of channels.
- **The first team's onboarding worked** because their channels were all created after the unification migration. No `G…` IDs in the first team's namespace. The validator passed because its blind spot didn't intersect with the first team's reality.
- **The second team hit the blind spot** because they were on Slack longer; their workspace had legacy private channels with `G…` prefixes.

This is the structural pattern: **a validator whose correctness is measured against test fixtures the developer wrote, deployed into a world where the actual input shape is different.** The test data and the regex come from the same mental model; both can be wrong in the same direction; the tests can't surface the wrongness because they're testing the model, not the world.

## How It Was Discovered

A user couldn't get past form validation, escalated, and the Forge team eventually clicked into Slack's docs.

The discovery channel matters. The validator's error was *honest by the validator's own model* — the message said "this doesn't look like a channel ID," which was true relative to the regex. The user had no way to know that the validator's model was wrong; from their POV the message read as either a typo accusation (they hadn't made one) or a system bug (the more likely conclusion). The path from *user hits error* to *team discovers regex is too narrow* required: a user who didn't give up, an escalation channel that surfaced the friction, a Forge team-member willing to actually read Slack's docs end-to-end, and the specific historical knowledge that Slack ever used `G…` for groups.

If any of those failed, the bug would have stayed live. The validator would have silently turned away every new team with a legacy-channel destination. The onboarding funnel would have a hidden failure mode nobody could point at.

This pattern recurs widely. The team-internal tests pass; the production failure mode only surfaces when a user trips over it and refuses to walk away quietly.

## What Fixed It

PR #41 widened the regex.

Mechanically: `/^C[A-Z0-9]+$/` became `/^[CG][A-Z0-9]+$/`. The spec gained fixtures for both prefixes: `Cgeneral`, `C123ABC`, `Gannounce`, `G456DEF`. The form-level validation now accepts both. The second team's engagement saved on the next attempt.

The structural fix the PR explicitly didn't make: replacing the regex entirely with a Slack-API call. The validator could ask Slack `conversations_info` whether the channel ID resolves before saving; Slack's response would be the authority instead of the regex. Three reasons this didn't ship in PR #41:

1. **Cost.** An API call on every form submission is slower than a regex, and Slack rate-limits aggressively. The validation flow would have to handle rate-limit responses gracefully.
2. **Scope.** PR #41 was a hotfix for the second team's onboarding. The structural change is its own PR with its own design discussion.
3. **Failure mode.** A Slack-API-call-based validator fails differently from a regex-based one — it can fail because Slack is down, because the network is flaky, because the bot lacks permission to look up that channel. Each new failure mode needs an operator-facing message, and designing those needs time.

The TODO is open. The pragmatic fix shipped. The structural fix is owed. The lesson lives in the gap: **the regex catches typos; only resolving at the source catches *the regex was wrong*.**

## The Durable Lesson

When you validate an identifier from an external system, **the test that the format matches your assumption is weaker than the test that the value actually resolves at the source.** Cheap synthetic validation drifts from reality faster than you think.

The mental-model flip the lesson rests on: **regex is a stand-in for an oracle.** You're trying to answer "is this a valid Slack channel ID?" The oracle is Slack itself. The regex is a guess at what Slack would say. The guess can be wrong in ways your tests can't catch, because *your tests come from the same mental model as the regex*.

> **Heuristic.** For any identifier validation against an external system, decide explicitly: *do I need fast inline feedback, or do I need the value to actually resolve?* If both, do both — keep the regex as a cheap typo-catcher with a permissive shape, and call the oracle (Slack's `conversations_info`, the database's `find_by!`, the third-party API's `get_resource`) before committing the value. Don't let the regex be the only check; the regex catches typos. Only the oracle catches the regex being wrong.

The shape generalizes:

- **Email format regex.** Every project ships one. Every project's regex is wrong in some corner case. The fix that survives is *the cheap regex catches the obvious typos, then we send a verification email and wait for click-through.* The verification email is the oracle.
- **Phone number format regex.** Same pattern. Every country's format is its own thing. The structural fix is *parse with libphonenumber, then verify by SMS.*
- **Credit card BIN validation.** A regex can check that a card number looks plausible (Luhn checksum, length, prefix range). Only the payment processor can say the card actually works. Don't let regex-validity be the trust signal for card validity.
- **External resource IDs in general.** GitHub repo names, Stripe customer IDs, AWS resource ARNs, Linear issue keys — each system has its own format. Each system's format has historical edge cases the docs don't lead with. The regex is the optimization; the API call is the truth.

In every case, the rule is the same: **regex catches typos. API calls catch reality. Don't conflate them.**

The pairing worth naming: this is the *external-identifier* version of the companion story `forge-keyword-router-natural-language.md`. Both stories are *synthetic validation drifting from reality*:

- The keyword-router story — a keyword parser that assumed users would speak in keywords. Tests passed because the fixtures were keywords. Real users typed natural language. Parser rejected real input as unrecognized.
- This story — a regex that assumed Slack IDs would start with `C`. Tests passed because the fixtures were `C…` IDs. Real Slack issues `G…` IDs too. Regex rejected real input as malformed.

Same structural error in both. Both checks were validated against fixtures the developer made up, using the same assumptions that produced the check. **The synthetic test data matched the synthetic check because both came from the same wrong mental model.** Neither story's tests could have surfaced the bug because the tests were testing the model, not the world.

The fix-shape difference is worth noting: the keyword-router story's fix is *LLM-as-fallback* (delegate the unclear case to a smarter handler); this story's fix is *resolve-at-source* (ask the external oracle). Both fixes move the trust signal from a synthetic local check to a richer real-world check. The richer check costs more (LLM tokens / API call) but absorbs the long tail the synthetic check can't see.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *identifier validation against external systems where the external system is the source of truth*. It doesn't apply to:

- **Identifiers your own system mints.** A UUID you generate, a slug you compute, an ID from your own database — your local format check *is* the oracle, because your code is the system that decides validity.
- **Format checks that are advisory, not gating.** A "this doesn't look right, did you mean…" hint that doesn't block submission. The hint can be wrong without consequence; the user proceeds anyway. The lesson is about *blocking* validation.
- **Cases where the external API call is genuinely too expensive.** A high-frequency endpoint where every form submission validating against an external API would exceed rate limits or budget. Stay with regex, document the blind-spot, and add a recovery path for users who hit it (e.g., a "this ID was rejected — submit a ticket" affordance that bypasses the regex check after manual review).
- **Throwaway internal tools.** A scratch admin script used by three engineers who'd notice immediately if it rejected something valid. The discipline is for user-facing flows where the user can't easily escalate.

The signal: *if my regex rejects a value that the external system would accept, who notices and how?* If the answer is *the user, who concludes the system is broken and stops trying*, the regex is the wrong layer to validate at.

## What This Story Is *Not* Evidence For

- **Not evidence that regex validation is bad.** It's the right tool for catching typos cheaply. The bug isn't using regex; it's *using regex as the only check on identifiers the external system is the authority on*. Layer the regex with a source-resolution call; both are useful.
- **Not evidence that the Forge team was careless.** The mental model that Slack channel IDs start with `C` is correct for ~95% of channels created post-migration. The blind spot is one Slack's docs don't lead with. Most teams have this blind spot until it bites them.
- **Not evidence that Slack's API is poorly documented.** The `G…` prefix is documented; it's just not in the place a reader would expect. The lesson isn't *Slack should document better*; it's *don't rely on memory of an external system's format when the external system itself can answer the question.*
- **Not evidence that the regex-widening fix was wrong.** It was the right pragmatic fix for PR #41. The structural fix (resolve-at-source) is owed but doesn't undo the regex-widening — both can coexist. The lesson isn't *don't widen the regex*; it's *the regex isn't the trust signal*.
- **Not evidence that this only happens with external systems.** The same shape recurs internally — a database constraint check that assumes a specific format on a free-text column, a schema validation that encodes a model the producer doesn't match. External systems make the shape sharper because the cost of being wrong is higher (the user leaves), but the structural error is generic.
