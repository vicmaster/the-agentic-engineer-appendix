# leads-crm / Ninety-One Files and No Backfill

## Date / Version Context

- **Date:** 2026-04-17 (commit `1a3669f`, the 91-file directory-workflow commit) and 2026-04-18 (commit `6c193f4`, the backfill task plus additional specs). Repo state at the time: ~30 commits in over ~21 weeks of work, mid-build, between the 2026-04-04 REST-API + MCP inflection point (`9d8d1f6`) and the 2026-05-06 remote-MCP shift (`27c04e6`). Production launch already in motion — the leads-crm app was being used by a live sales team during this build arc.
- **Project:** leads-crm — Ruby on Rails 8 / Postgres / Hotwire / Devise + Pundit / RSpec. Sales-qualification CRM with a Lead → Pre-Qualified → MQL → SQL → Prospect pipeline, scoring, promotion rules, reporting, team access control. By 2026-04-17, the app had been running in production with real lead records for weeks; the directory workflow was a *feature addition*, not a greenfield build, and that asymmetry is exactly what the story is about.
- **Surface for this story:** the `Lead` table and its relationship to new `Company` / `Contact` directory tables. Pre-change: `Lead` rows carried denormalized company/contact strings inline. Post-change: `Lead` rows have foreign keys into `Company` and `Contact`, with the directory tables as the canonical source. The new shape requires every existing lead to either be matched to an existing directory row or to have a fresh directory row created from its denormalized fields.
- **Glossary, used in this writeup:** *New-path commit* = the commit that adds the new feature, schema, and specs, assuming the data is shaped by the new path from the start. *Recovery commit* = the commit that walks existing production-shaped data into the new path — usually a backfill rake task, a data migration, or a one-off script with logging. *Orchestration phase* = a deliberate, named step in a multi-phase change that produces a named artifact (the new shape, the backfill task, the verification report) and gates the next phase.

## What Was Being Attempted

Add a company/contact directory model to leads-crm so that multiple leads sharing the same company or contact would link to a single canonical directory row instead of carrying duplicate denormalized strings.

The motivation was the usual normalization argument. Sales reps were entering "Acme Corp", "ACME Corp.", "Acme Corporation", "acme" as distinct strings on different leads. Reports counted them as different companies. The team wanted a directory table — one Company row, many Lead rows pointing at it — so that the pipeline view, the reports, and any future per-company analytics had a single source of truth.

The feature work was substantial. 91 files in a single commit: a `Company` migration, a `Contact` migration, the join tables, the Lead associations, the directory-management controllers, the directory-list and directory-edit views, the autocomplete on lead forms, the link-lead-to-directory flow, the unlink flow, the specs covering all of it, the policy updates, the seed data for the directory tables in test, the Hotwire bits for the autocomplete UX. By every internal correctness criterion, the feature was complete. CI was green. The seed-data path worked end-to-end. A fresh test database loaded with seeds had directory rows; new leads created through the UI linked to them; the reports counted companies correctly.

What the feature work didn't model: the existing production leads. Hundreds of `Lead` rows with denormalized `company_name` and `contact_name` strings, no directory association, sitting in the production database from before the feature shipped.

## What Went Wrong

The new-path commit was complete on the new path and silent on the old data.

The way this fails is asymmetric across the feature's surfaces:

- **The new-lead-creation path worked correctly.** A sales rep entering a new lead through the UI got the autocomplete, picked a directory row (or created one), and the lead's foreign key was set. The feature's spec covered this case exhaustively.
- **The new-lead-creation-from-API path worked correctly.** The REST API endpoint accepted `company_id` and `contact_id` on lead creation. The spec covered it.
- **The reports were broken for existing leads.** The new reports counted *directory rows* as companies, not denormalized strings. Existing leads with no directory association didn't show up in the per-company view at all. They had `company_name` text but no `company_id`. The report joined on `company_id` and silently filtered them out.
- **The directory-list view was missing most of the actual companies.** The list rendered `Company.all`. Existing companies that were only represented as denormalized strings on old leads weren't in `Company` at all. The directory looked sparse and inaccurate.
- **The unlink flow had nothing to unlink for existing leads.** The UI showed *"This lead is not linked to any company"* on every legacy row. From the operator's view this looked like the feature was broken — *"I thought we just added the directory? Why isn't this lead in it?"*

The structural shape of the breakage: the feature *worked correctly under the assumption that all leads flowed through the new path*. Existing production leads hadn't flowed through the new path. They were in the database in the old shape. The new feature's logic didn't fail on them — it just didn't *see* them, because the new code paths joined on the new foreign keys.

This is the most insidious form of agent-built-feature breakage. The feature isn't wrong. The tests aren't wrong. The CI build isn't wrong. The feature's spec covers the new-data path completely, and the new-data path works. The failure is at the data layer that the spec didn't model — existing production data — and the failure mode is *invisibility*, not error.

From the agent's perspective, building the feature in 91 files in one commit, the work was complete. The spec said *"add a directory model with linkages from leads to companies and contacts"*; the work added the directory model with linkages from leads to companies and contacts. Spec met. Done. The omission was *"and walk the existing leads into the new shape"*, which the spec didn't name, and which the agent had no signal to write because the agent was building from the feature description, not from production-state inspection.

## How It Was Discovered

By looking at production after the feature shipped.

The day after `1a3669f` landed, the operator (the human at the keyboard, building this with agents) looked at the directory list in the deployed environment and saw the sparseness — most of the leads weren't linked to anything. The reports counted companies that were obviously wrong (small numbers; the actual count of distinct company strings across existing leads was much higher). The operator's first reflex was *did the feature even work?*, then on inspection *the feature works for new leads but did nothing for existing ones*.

The discovery channel is *the operator looking at deployed state with production-shaped data*. There's no test that would have caught this — the test suite ran on seed data, which the agent had freshly created, which already conformed to the new shape. Synthetic data in tests doesn't carry the historical asymmetry production data carries. The asymmetry is *what the world looked like before the feature shipped*, and the feature's spec doesn't describe it.

The non-discovery channel — and the part worth pausing on — is that CI, lint, type-checking, and every automated gate were silent. Nothing in standard agent-built-feature verification covers *"and what about the rows that already existed?"* The agent's natural completion criterion is *the feature works for the case the spec describes*. Existing-data is not in the spec.

This is also the discovery channel for the wider class. Any agent-built feature that adds a new model, a new association, a new required field, a new index, a new policy gate — anything that *changes the shape of an existing table or changes what counts as a valid row* — has the same gap. The new path works; the existing rows don't conform; nothing fails loudly; the operator notices in the deployed environment some hours or days later.

The leads-crm operator caught it within ~24 hours, which is fast for this failure mode. Teams without a habit of looking at deployed state with production-shaped data have shipped this shape and not noticed for *weeks*, with reports running wrong the entire time.

## What Fixed It

A second commit, the next day. Commit `6c193f4` (2026-04-18) added a staging backfill task plus additional specs.

The backfill task's structure:

1. **Iterate every existing `Lead` row** in some bounded chunk size (to avoid loading them all into memory or holding a long-running transaction).
2. **For each lead, read its denormalized `company_name` and `contact_name` strings.**
3. **Look up or create the corresponding `Company` and `Contact` directory rows** — `find_or_create_by(name: company_name)` semantics, with the inevitable edge cases (whitespace, case-normalization, the question of whether *"Acme Corp"* and *"ACME Corp."* are the same company, which is a *semantic* question the backfill can't answer without operator input).
4. **Set the lead's new `company_id` and `contact_id` foreign keys** to the resolved directory rows.
5. **Log every operation** — which rows were created, which were merged, which were skipped because the denormalized strings were blank, which were flagged for operator review because the normalization was ambiguous.
6. **Run on staging first** with a representative production snapshot, validate the output, only then run on production with explicit confirmation.

The "more specs" part of the second commit covered the backfill task's logic — the find-or-create paths, the deduplication, the logging, the dry-run mode. Importantly, the specs *also* covered the post-backfill state of the application: *given a database where the backfill has run, do the reports, the directory list, and the unlink flow behave correctly?* That post-state coverage is what closed the loop on the original feature's silent-on-old-data failure mode.

Two structural notes on the fix:

**The fix is shaped as a phase, not a patch.** The backfill is its own commit, its own rake task, its own specs, its own staging-then-production rollout. It's not bundled into the original `1a3669f` because by the time the operator noticed the gap, the feature commit was already deployed and being used by sales reps creating new leads. The backfill is *catching up to* the feature, not bundled with it. This is the orchestration-phase shape — a deliberate, named, separately-shipped phase with its own artifact.

**The "had I planned for it" version is different.** If the operator had identified the existing-data gap *before* the 91-file commit went out, the right shape would have been a three-phase orchestration: (1) add the schema and the new model without removing the denormalized fields, (2) backfill the directory rows from the existing data with a verified rake task, (3) wire the new model into the reports/UI/API and (optionally) deprecate the denormalized fields. Each phase ships independently; each is reversible; the production data is never in a half-migrated state when something else is also changing. The actual shape that happened — phase (1) and phase (3) bundled, phase (2) added the day after — is the *recovered* version of that orchestration. The lesson is that the recovery shape and the planned shape are the same shape; the only difference is whether you wrote the backfill before or after noticing the gap.

What's owed but not yet shipped: a project-level discipline rule that *any commit adding a foreign-key relationship to an existing table comes with a backfill commit on the same PR, not the day after*. Not yet written down as a rule; lives in operator habit at the moment.

## The Durable Lesson

Agents are good at creating new workflows. **Migrations of existing production-shaped data into those new workflows need a separate recovery story, and the orchestration plan has to name it as its own phase or it ships as commit-two-the-day-after.**

The mental-model flip the lesson rests on: the agent's natural completion criterion is *the feature's spec is satisfied*. The feature's spec describes the new path. Existing data is *the world the feature is landing on*, not part of the spec — and the agent has no signal to look at it unless the operator includes the look as an explicit step. The omission is structural to how feature specs are written, not to how agents read specs.

> **Heuristic.** For any agent-built feature that adds a new model, association, required field, or index to an existing schema, the orchestration plan has four phases by default: (1) the new shape ships without removing the old shape; (2) a backfill rake task walks existing data into the new shape, with a dry-run mode, logging, and staging verification; (3) the new path becomes load-bearing in reports, UI, and API; (4) the old shape is deprecated and eventually removed. Phase 2 is where the agent-built-feature failure mode lives. Skip it and you get the *"feature works for new leads, sparse for existing ones"* shape. The orchestrator's job at the seam is to ask, before the agent ships the first phase, *"what about the rows that already exist?"* If the answer is *"the new path doesn't see them"*, phase 2 has to be planned, not improvised.

The shape generalizes well beyond Rails:

- **Adding a NOT NULL column to an existing table** — the new column has no default for existing rows; either nullable-then-backfill-then-tighten or default-on-create. Same shape.
- **Adding a new index that the application queries depend on** — the index must be built before the queries run against it; in large tables this is a multi-phase deploy (build index concurrently, verify, ship the query change).
- **Promoting a denormalized field to a normalized association** — the leads-crm story exactly, in any framework.
- **Adding a new enum value with semantic meaning** — existing rows that should map to the new enum value need backfilling; existing rows that shouldn't, don't.
- **Adding a new permission scope to a Pundit-style policy system** — existing roles that should have the new scope need it; the policy code alone doesn't grant it retroactively.
- **Adding a new event type to an event-sourced system** — existing aggregates need to be replayed against the new event handler, or the historical state diverges from what the new code expects.

In every case, the pattern is identical: **a new shape ships against a database that already contains rows in the old shape, and the new code's correctness depends on the old rows being walked into the new shape.** The fix is structural: name the backfill as its own phase, write its rake task and specs alongside (not after) the new-shape commit, verify on staging with production-shaped data, then ship.

The pairing worth naming: this is the *existing-data* version of the *function-returned-doesn't-mean-work-completed* pattern. See the companion story `forge-async-adapter-smoke-test.md` on *agents miss the contract that lives in the environment, not the spec* — there the missed contract was the dev/prod adapter lifecycle, here it's the existing-data shape. See also the companion story `leads-crm-org-scoping.md` on *application-layer invariants violated by data that didn't go through the normal path* — there the second-surface data (APIs, MCP, imports) bypassed the UI-enforced org scoping; here the existing-data bypasses the new feature's spec entirely. Both stories are about *the gap between what the agent's code path sees and what the production data actually looks like*.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *new features added to schemas that already have production data*. It doesn't apply to:

- **Greenfield builds.** A brand-new app with no production data has no existing-shape problem. The first deploy creates the schema with the new model from the start; there's nothing to backfill. The lesson kicks in once production has accumulated rows that pre-date the feature.
- **Features that don't touch existing tables.** Adding a new unrelated table, a new microservice, a new optional integration — none of these need a backfill because they don't have an existing-shape to migrate. If the new model has no association with anything that already exists, there's nothing to walk.
- **Features that are purely additive on optional fields.** Adding a new nullable column that's only used by new flows, where existing rows being NULL is a valid and expected state, doesn't need a backfill. The lesson applies when the new code path *assumes* the new shape and *fails or hides* under the old shape.
- **One-off scripts, prototypes, throwaways.** Code that won't see production data doesn't have this problem because there's no production data. The lesson is a production-app discipline.
- **Features explicitly scoped to "new records only."** Some products legitimately deprecate the old shape — *"existing records keep their old format; new records use the new format; the two are kept side by side forever."* That's a deliberate choice; the lesson doesn't apply because there's no migration. The risk shifts to *"will the dual-shape code stay correct over time?"* which is a different lesson (closer to the companion story `canvas-mcp-two-shadow-apis.md`).

The signal: *does the new feature's correctness depend on existing rows conforming to the new shape?* If yes, backfill phase. If no, ship the feature alone.

## What This Story Is *Not* Evidence For

- **Not evidence that agents shouldn't write large commits.** The 91-file commit was correct work — the feature is real, the specs are real, the schema design is sound. The omission was a separate phase, not a problem with the commit's size. Splitting the commit into ten smaller ones wouldn't have surfaced the backfill gap; only thinking about existing data would have.
- **Not evidence that agent-built features are unreliable.** The feature works correctly for its specified scope. The lesson is about a *scope gap between feature specs and production reality*, which exists for human-written features too. The agent didn't fail at the work; the spec didn't include the work.
- **Not evidence that backfills should always ship in the same commit.** Sometimes the right shape is *deploy the schema and the backfill is a follow-up after staging verification*. The lesson is that the backfill needs to be *planned*, not that it has to ship at any particular moment. Day-after is fine if it was planned for day-after; day-after-because-we-noticed is the failure mode.
- **Not evidence that all production-data-touching changes need rake-task-shaped backfills.** Online-migration patterns (dual-write, dual-read, gradual cutover) are valid for changes where downtime is unacceptable or the table is too large for a single rake-task run. The shape of the backfill depends on the constraints. The lesson is *name the phase and pick a shape*; rake-task is one shape, not the only one.
- **Not evidence that the existing test suite was inadequate.** The tests covered the feature's behavior — given the new shape, the code works. The bug is at the *fixture / production-data-asymmetry* layer that standard tests don't model because test databases start from seed data. The structural follow-up is the same one named in the *Heuristic*: include the post-backfill assertions in the backfill commit's specs (which `6c193f4` did) so that the *combined* state — schema change plus backfill — is what gets verified, not just the schema change in isolation.
