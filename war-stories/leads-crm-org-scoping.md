# leads-crm / Org Scoping Bypassed by Every Non-UI Surface

## Date / Version Context

- **Date:** Fix landed 2026-02-14, commit `79f0f41`. The bug had existed since the early API/import paths went in. The MCP work on 2026-04-04 (`9d8d1f6`) inherited the fixed model.
- **Project:** Lead CRM — Rails 8 / PostgreSQL / Hotwire / Devise + Google OAuth / Pundit / RSpec. Multi-tenant: every meaningful record belongs to an `organization_id`. Pipeline visibility, lead access, reports, and team boundaries are all scoped per org.
- **Surface for this story:** the entire write boundary. Every place a `Lead` (or related record) could be created or modified. Web UI clicks, API requests, MCP tool calls, CSV imports, rake-task backfills, console scripts.
- **Glossary, used in this writeup:** *Org scoping* = the invariant that every record belongs to one organization, and visibility/edit access is gated by that membership. *Pipeline visibility* = the rule that a user only sees leads in their org's pipeline. *Orphaned record* = a row whose `organization_id` is missing or wrong, making it invisible-but-present (visible to no UI, still real in the database). *Hard invariant* = enforced at the model/database/policy layer regardless of which surface produced the write. *UI convention* = the looser version: the rule is enforced by the controller's `before_action` and the view's `current_organization` scoping, but writes that bypass the controller bypass the rule.

## What Was Being Attempted

Build a multi-tenant CRM. Every user belongs to an organization. Every lead, contact, task, email, and report belongs to an organization. Users only see their org's pipeline. Cross-org visibility is forbidden.

The early implementation followed the easy Rails path: scope queries with `current_organization`, gate controllers with Pundit policies, render views with the current user's org in the controller context. It worked for the web UI. Every click was wrapped in the right scope. The pipeline view filtered correctly. Cross-org leakage didn't happen *via the UI*.

That phrasing — "via the UI" — is the whole story.

## What Went Wrong

Pipeline visibility broke for a subset of leads. Users in Org A couldn't see leads they should have been able to see; some leads showed up nowhere. A pipeline visibility debug spec, written to investigate, surfaced the underlying mechanic: there were `Lead` rows with `organization_id` either missing entirely or set to the wrong org. They had been created by paths that didn't run through the UI's scoping logic.

The candidate paths include:

- **The REST API.** Bearer-token auth, but the token was associated with a user; the create path used `current_user.organization_id` *if* the controller was wired that way. Some early endpoints weren't, or were forgiving of the parameter being passed in the request body. An API client could send a `lead[organization_id]=N` and the model accepted it.
- **CSV imports.** A bulk-import path that read a row, built a `Lead`, and saved it. The org wasn't necessarily set from the import context — it was easy to write the import path assuming "the importer's org is implicit" and miss that the path could be invoked outside of a per-user context.
- **Migrations and backfills.** Rake tasks and one-off scripts that populated records during early development or data shape changes. Org context wasn't carried through; some rows landed without one.
- **Console scripts.** `rails console` writes by an operator are by definition outside the UI. Anything written there is the operator's responsibility — but the model didn't refuse to save records without org, so the operator could miss it without an error.

None of these paths showed a runtime error. The records saved. The UI showed what the UI's scoped query returned. The orphaned records existed in the database, invisible to the pipeline view, occasionally surfacing in reports or in cross-cutting queries that didn't apply the scope.

The bug wasn't *one* surface getting it wrong. It was *every non-UI surface* getting it wrong, because the rule was enforced at the UI layer and nowhere below it.

## How It Was Discovered

A pipeline visibility debug spec. The discovery channel wasn't a user complaint or a production alert — it was a deliberate investigation written when the visibility behavior didn't match expectations. The spec found the orphaned rows, traced the scoping rules, and surfaced the gap between "what the controller enforced" and "what the model allowed."

This is a better discovery channel than the alternative. The alternative would have been: a customer in Org A views their pipeline, doesn't see lead X, calls support, support traces the row, finds it's missing `organization_id`. By the time that happens, the orphaned-record population is unknown — could be five rows, could be five hundred — and every fix has to include a backfill to repair the existing damage.

The fix in 2026-02-14 included exactly that: enforce the invariant *and* repair the orphaned records. The repair was the cost of letting the rule live at the UI layer for too long.

## What Fixed It

Two things, landed together as `79f0f41`:

1. **Enforce org requirements below the UI.** The `Lead` model (and related models) require `organization_id`. Validation refuses to save without it. Pundit policies double-check on the read path. The controllers still set the org from `current_user`, but now if any non-UI path tries to save without one, the save fails — no silent invisible row.
2. **Repair the existing orphans.** A migration and backfill task that walked the existing data, found rows with missing/wrong `organization_id`, and either set them correctly (where the right org was inferable from associations) or surfaced them for manual review.

The fix is structurally complete because it covers both halves of the problem: stop the bleed (the validation), and clean the wound (the backfill). Either alone would have been incomplete. Validation without backfill leaves the existing orphans hidden forever. Backfill without validation lets new orphans land tomorrow.

What the fix earned, beyond closing this bug: the API/MCP rollouts that came two months later (`9d8d1f6` on 2026-04-04 and `27c04e6` on 2026-05-06) inherited a model that already refused to save without org. The MCP tools couldn't accidentally create orphans — the model wouldn't let them. The 15 MCP tools and the remote `/mcp` endpoint were able to ship without re-litigating tenant scoping at every call site, because the rule lived below the call sites.

That's the load-bearing observation: the cost of moving the invariant below the UI is paid once. The cost of *not* moving it grows with every new surface that touches the data.

## The Durable Lesson

Tenant ownership must be a hard invariant, not a UI convention.

The web UI is one surface. APIs, agents, imports, backfills, console scripts, scheduled jobs, internal admin tools, and (eventually) MCP tool calls are all parallel surfaces. Each one of them is a write path. Each one of them either honors the invariant or doesn't.

When the invariant lives at the UI layer — *the controller scopes it, the view filters it, the user's session has it* — only the UI honors it. Everything else has to remember to. Memory across surfaces is not a thing. The first non-UI surface that ships will leak the invariant unless the developer building it remembers, and the developer building it usually doesn't, because the rule isn't visible at the model layer they're working in.

> **Heuristic.** Before you ship the second surface (API, agent, importer, anything non-UI), audit which invariants currently live at the UI layer. Move every invariant that needs to hold across surfaces *below* the UI — to the model, the policy, the database constraint. Do this *before* the second surface ships. Do not negotiate per-surface; negotiate once at the layer below all surfaces.

The shape generalizes. Any rule enforced by "the way users interact with the system" — admin/non-admin distinctions, draft/published states, owner-only edits, role-based capabilities — is a candidate for the same drift. The UI is the most-traveled surface, so the rule looks complete when only the UI exists. The rule looks complete *until the second surface lands*.

The order matters. Once you have orphans, you have a backfill cost. Once you have a backfill cost, you have to choose between blocking the second-surface ship and shipping it on top of dirty data. Both choices are bad. The cheap fix is moving the invariant before the dirty data exists, which is *before the second surface*, which is, almost always, *before you think you need to.*

This is the deepest version of the state-opacity lesson. The system has more layers than the operator can see. The bug isn't in the layers; it's in the gap between which layer enforces what, and the gap is invisible until a second actor exercises the same data through a different layer.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *invariants that need to hold across multiple write surfaces in a multi-tenant or multi-actor system*. It doesn't apply to:

- **Single-surface systems.** A CLI tool with no API, no agent layer, no import path, and no plans to ship one is fine enforcing rules at the input boundary. There's no second surface to leak through.
- **Invariants that are genuinely UI-only.** Form validation messages, optimistic UI states, accessibility-driven warnings — these belong at the UI because they exist to serve the UI's user. Pushing them into the model would make the model do work it shouldn't.
- **Truly stateless tool surfaces.** A read-only MCP server that exposes pre-computed reports has no write path to leak through. The lesson is about writes, not reads. (Though authz on reads is its own version of this lesson — see Pundit's policy_scope discipline.)
- **Cases where the cost of not moving the invariant is genuinely small.** A prototype with no real customer data, no compliance constraints, and a planned rewrite within the month is allowed to defer this. The orphan-repair cost is bounded by the rewrite. Don't generalize the principle into "every prototype needs DB constraints from day one"; that's how prototypes die.

The signal: *will this system have more than one write surface, and does the invariant need to hold across all of them?* If yes, push it below the UI before the second surface ships. If no, defer.

## What This Story Is *Not* Evidence For

- **Not evidence that Pundit/policy-layer enforcement is wrong.** Pundit policies are part of the fix; they live on the read path. The lesson isn't "skip policies, just use validations" — it's "have both, and have validations be the floor."
- **Not evidence that database constraints solve everything.** A `NOT NULL` on `organization_id` would have caught some of these orphans, but not the wrong-org cases (where the column was set, just to the wrong value). The fix is at the model and policy layer, with the database constraint as a backstop. Don't read this as "DB constraints are sufficient."
- **Not evidence that all multi-tenant bugs are scoping bugs.** Multi-tenancy has many failure modes — performance isolation, billing leakage, cross-tenant cache pollution, schema drift in shared infrastructure. This writeup is about *write-path invariant leakage*, which is one of them.
- **Not evidence that the UI layer should never enforce anything.** The UI absolutely should enforce — for ergonomics, for clarity, for the user's flow. The lesson is that the UI's enforcement is *insufficient on its own* the moment a second surface exists. UI enforcement plus model enforcement is fine. UI enforcement alone is the trap.
