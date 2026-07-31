# forge / The Audience Constraint as Leverage

## Date / Version Context

- **Date:** The audience constraint was decided pre-repo (2026-04 product framing); the repo started 2026-04-14 (commit `c401af0`) with the constraint already encoded in the seed data (`db/seeds.rb:49` — the Magma Virtual Ops Analyst is explicitly leadership-only). The structural anchor commit is `004-admin-access-control` (2026-04-25), which made *"non-leader"* a first-class denied state in the auth layer. Production launched 2026-04-29 to ~5 leadership users; the audience boundary has held through 127 commits and ~3.5 weeks of post-launch operation.
- **Project:** Forge — MagmaLabs Virtual Employee Platform. Ruby 3.3 / Rails 8 / Postgres / Sidekiq / Heroku Eco. Supervised, capability-scoped agents that read company data (BSC, KPIs, Slack threads), summarize with Claude, and deliver back to leadership via Slack and email.
- **Surface for this story:** the product as a whole. There's no specific feature or commit where the constraint surfaced as a problem; the constraint surfaces as *the absence of features that would have been built without it*. The structural-anchor commit (`004-admin-access-control`) is where the constraint moved from an implicit design assumption to an enforced first-class denied state in the auth layer.
- **Audience boundary, in writing:** the seed data names the audience exactly. `db/seeds.rb:49` creates the Magma Virtual Ops Analyst as a `VirtualEmployee` with four capabilities (`bsc_analyst`, `kpi_watcher`, `meeting_prep`, `data_source_chat`), all bound by `CapabilityBinding` rows to a single audience scope. The audience scope is *leadership* — explicitly named, explicitly small, explicitly Spanish-default. Every routine, every artifact, every Slack DM, every email rollup flows to this audience and only this audience.
- **Glossary, used in this writeup:** *Audience constraint* = the deliberate, structural decision about who the system's output is for, named early and enforced in code rather than left to operator discretion. *First-class denied state* = an authorization outcome that is a named, expected, tested result of the policy, not an exception or fall-through. *Asymmetry product* = a product where the population that consumes the output and the population that produces the input are different by design, and the difference *is* the product's shape.

## What Was Being Attempted

Build an AI system that leadership would actually read.

The product framing was operational, not technical. MagmaLabs leadership had a Balanced Scorecard, a set of KPIs, and a stack of monthly leadership meetings — and the standard pattern for *how does leadership stay current?* was *a PM navigates dashboards and rolls up status into a slide deck*. The roll-up was slow, the data was stale by the time leadership read it, and the load was on the PM rather than on the data sources. The product question was *can we get leadership reading the actual numbers, with the actual context, without anyone navigating a dashboard or assembling a deck?*

The natural product instinct — and the one most teams would have shipped — was a *general-purpose AI assistant for the company*. Everyone gets an account. Everyone can ask questions. Everyone gets summaries. The product is *AI for the team*; leadership is one population among many.

The audience constraint took a different shape: **leadership-only, ~5 users, Spanish-first.** Non-leaders are *not* a denied audience in some narrow sense — they're not an audience at all. Forge isn't a *team product with a leadership feature*; it's a *leadership instrument with a hard audience boundary*. The asymmetry isn't a permissions filter; it's the product's shape.

The decision was load-bearing. Once the audience was *only leadership*, the entire downstream design fell into place:

- **No per-user UI.** Five users don't need a self-serve admin panel for managing their notifications, preferences, or routing rules. An operator (the human at the keyboard) configures everything in the admin UI.
- **No real-time anything.** Leadership reads monthly briefs and daily routines. There's no chat-shaped product surface that needs WebSocket plumbing, presence indicators, or sub-second response budgets.
- **No provisioning layer.** Five users don't need SAML, role onboarding wizards, invitation flows, or self-serve account creation. The five leadership users are seeded into the database.
- **Spanish-first copy.** Leadership reads Spanish; English is a translation path, not the default. The prompts, the templates, the Slack messages, the email subjects — all Spanish by default, English derived.

Each of these decisions, on its own, would have been a multi-week build for a general-audience product. Together, they would have been the first three months of Forge's roadmap *before any actual capability work shipped*. The audience constraint replaced that work with *not doing it*.

## What Went Wrong

Nothing — but that's the wrong question for this story.

The conventional war-story shape is *what broke, when, and how was it fixed*. This story's shape is *what didn't break because we decided not to build it*. The reverse question gives the same lesson, viewed from the opposite direction: **what would have broken if we hadn't shipped the constraint at day one?**

Three concrete answers, each one a sub-incident that almost-but-didn't-happen:

**The provisioning layer that wasn't built.** A general-audience version of Forge would have needed SAML integration (MagmaLabs uses Google Workspace; SAML through Workspace is doable but has its own SDK gymnastics), invitation flows, role onboarding wizards, and a UI for the operator to manage who has which capability. Realistically: 3-4 weeks of work, plus the ongoing tax of *every new feature needs to think about per-user authorization*. Without the audience constraint, this work would have shipped before the first real capability — meaning the first month of the project would have been infrastructure, not product.

**The role onboarding layer that wasn't built.** Beyond provisioning, a general-audience product needs per-role UX: a designer's view of the dashboard is different from a sales rep's view, which is different from a director's view. The product team's natural instinct (write personas, design per-role surfaces, ship them) was eliminated by the constraint — there's effectively one persona to design for, MagmaLabs' leadership, not a spectrum of roles. The avoided work: 2-3 weeks of design plus probably 4-6 weeks of front-end engineering.

**The real-time / chat layer that wasn't built.** *AI for the team* almost always becomes *a Slack bot anyone can DM* or *a web chat anyone can open*. That product surface needs WebSocket infrastructure, presence handling, message-order semantics, streaming response rendering, conversation state — all of which Forge avoided by being a *push* product (routines fire on schedule, leadership reads the output) rather than a *pull* product (anyone can ask anything). The avoided work: 4-6 weeks of infrastructure, plus the ongoing tax of *the chat surface is now a permanent product surface*.

Three avoided layers, ~10-13 weeks of work, none of which would have served the actual audience. The wish-I'd-known, in one line: *"If we'd shipped 'for everyone' first, we'd have built three layers (auth provisioning, role onboarding, UX redesign) we never needed. Forge would be 6 weeks behind."*

The conservative number is 6 weeks. The realistic number is higher, once the ongoing tax of *every feature now thinks about all those layers* is counted.

## How It Was Discovered

The constraint wasn't discovered after the fact; it was named at day one.

The discovery channel for this story is *the moment the operator considered the alternative and rejected it*. That moment isn't anchored to a commit because rejection-of-an-alternative doesn't produce a commit — it produces *the absence of a feature*. The closest commit anchor is `004-admin-access-control` (2026-04-25), which is when the audience constraint moved from *implicit design assumption* to *enforced first-class denied state in the auth policy*. Before that commit, *"only leadership uses this"* was a design intent. After it, *"non-leader"* was a named denied audience that the auth layer would reject with a specific error.

That commit is the anchor because it's the moment the constraint became *structural* rather than *implicit*. An implicit constraint is one any future feature could violate (*"oh, this small admin tool, let's just expose it to managers"*). A structural constraint is one the auth layer enforces at every request, with a tested-and-named denied path, that future features have to actively work around to violate.

The lesson worth pausing on: **discoveries that don't have a commit anchor are still discoveries.** This one is anchored to a *thought* — the moment the operator thought *I could ship this for everyone, and we'd lose six weeks doing it.* The thought happened pre-repo (the audience framing predates `c401af0`); the commit just made the thought structural.

The retrospective discovery channel — when the constraint *paid off* in the most visible way — is each time a new feature was scoped and the operator's first question was *"is this feature for leadership?"* If the answer was no, the feature didn't ship. The constraint did the design work before the feature got built. The record is full of moments where the constraint pre-decided the shape — mrkdwn-vs-CommonMark at the delivery boundary, thread-history reconstitution, IntentParser fallback to LLM — each of these decisions was *easier* because there were five users, all leadership, all Spanish-speaking, all reading the same shape of brief.

## What Fixed It

The fix is the constraint itself. The structural work was making *non-leader* a first-class denied state in the auth policy, and the rest of the product flowing from that.

The shape of the fix, in three concrete moves:

**`004-admin-access-control` (2026-04-25) made *non-leader* a first-class denied audience.** The auth policy got a `default_audience` field on `VirtualEmployee` (set to `:leadership`) and a corresponding `User#role` (set to `:leader` for the five seeded users, `:contributor` for everyone else, `:nil` for users who haven't been classified). Requests from non-leaders to a leadership-scoped VE return *"unknown user denial"* — a named, tested, expected outcome. The denied path is the *normal* path for non-leaders, not an exception.

**The seed file (`db/seeds.rb:49`) names the audience exactly.** The Magma Virtual Ops Analyst is a `VirtualEmployee` with four capabilities, all bound to the leadership audience. The seed isn't aspirational — it's the actual production audience. Adding a leader is a seed change plus a deploy. Adding a non-leader-facing VE is a product decision, not a config one.

**Spanish-default copy is encoded in the prompts.** Every capability prompt is written in Spanish first, with English as a translation hint. The model produces Spanish output by default. The delivery boundary (`forge-delivery-boundary.md`) handles English-channel rendering when needed. The audience's language is baked into the prompts, not parameterized — the cost of changing the audience to an English-default population would be rewriting every prompt, which is intentional friction.

Three pieces, working together, make the constraint structural. The pieces survive scaling events (a new capability, a new VE, a new delivery channel) because each piece is encoded at the layer where new features have to acknowledge it.

What's owed but not yet shipped: an explicit *audience review* step in the capability-design playbook — *"before this capability ships, who is the audience, and is that audience in `default_audience`?"* Currently lives in operator habit. Worth writing down as a one-line check in the spec-kit template (`.specify/*-template.md`) so future-Victor doesn't accidentally ship a capability whose audience drifted.

## The Durable Lesson

A hard audience constraint named at day one is the cheapest form of product design. **Knowing who is *not* a user is more leverage than knowing who is.**

The mental-model flip the lesson rests on: most product framing names the *target audience* (*"this is for sales reps"*) and treats the rest of the company as a latent expansion market. The Forge framing names the *denied audience* (*"this is not for managers, not for ICs, not for the broader company"*) and treats the constraint as a structural feature, not a temporary scoping choice. The first framing keeps the product permeable; the second keeps it structural. Permeable products grow features they didn't need; structural products grow features that earn their cost.

> **Heuristic.** For any AI / agentic product, name the audience constraint at day one and encode it in three places: (1) the auth layer's `default_audience` (or equivalent) with a named, tested denied state for non-audience users, (2) the seed data that defines the production audience set, and (3) the prompts that bake in the audience's language and context (Spanish-first, English-first, technical-first, executive-first — whatever the audience needs). Three places, three encodings, structural enforcement. Without all three, the constraint is implicit and the next feature can violate it. With all three, future-you has to actively work around the constraint to ship a feature outside it — which is exactly the right amount of friction.

The shape generalizes beyond AI products:

- **Internal tools for a single team.** Most internal tools start as *"this is for our team, but maybe other teams will use it later."* The *maybe* is where the scope expands and the product gets worse. Naming *"only this team, no expansion path"* at day one buys focus.
- **B2B SaaS at the early stage.** Many failed B2B startups built a *"product for SMBs and enterprise"* because the founder didn't want to choose. The audience constraint *"only enterprise, only 50+ seat deals"* (or the reverse) shapes pricing, support, deployment, and feature priority. The constraint is leverage; trying to serve both halves the leverage.
- **Open-source projects with a stated non-audience.** Projects that say *"this library is for X, not for Y"* in their README cut their support burden by an order of magnitude. The non-audience is the load-bearing scope decision.
- **Conference talks, books, and other communication.** A talk titled *"AI agents for senior engineers"* tells juniors they can leave, which lets the talk go deeper. The audience constraint is the rhetorical equivalent of `default_audience`.

In every case, the rule is identical: **the population the system serves and the population it explicitly does not serve are both load-bearing; treat the denied audience as a first-class design decision, not as a scoping accident.**

The pairing worth naming: this is the *product-shape* version of the *constraint-yields-discipline* pattern that runs through the companion stories `magmalabs-delegated-skill-decay.md` (*what you delegate is what decays* — the operator's audience for their own skills) and `markdown-toolkit-role-separation-solo.md` (*role separation, not team size* — the orchestration audience). All three stories argue that **explicit constraints applied to the audience of the work produce discipline that softer scoping doesn't.** Forge's audience constraint is the same pattern applied to the product's user audience. The operator's skill atrophy is the same pattern misapplied to the operator's own skills (delegating what you don't practice is the cost of the trade). Role separation in solo work is the same pattern applied to the orchestration roles. Three layers of the same lesson — *who the work is for, and who it is not for, is the first design decision*.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *deliberate audience constraints in products that benefit from structural focus*. It doesn't apply to:

- **Products whose value comes from network effects.** A messaging platform, a social network, a marketplace — these products' value scales with audience breadth, not depth. Naming a denied audience cuts the network. The lesson is for products where *depth-for-an-audience* is the value proposition, not *breadth-across-audiences*.
- **Infrastructure and developer-tooling.** Libraries, frameworks, dev tools — the audience is usually *anyone with a relevant problem*, and naming a denied audience is more often a turf war than a design choice. The lesson is for product surfaces with end users, not for infrastructure layers.
- **Products in a market-defining stage where the audience is unknown.** Sometimes the right move is to ship to many audiences, watch which one converts, then narrow. The lesson assumes the operator *can* name the audience at day one; some discovery contexts genuinely can't.
- **Audience constraints that aren't structural.** Naming an audience and then not enforcing it in the auth layer is worse than not naming one — it gives the operator false confidence while the product permeates. The lesson is *named **and** enforced*, not just named.
- **Products where the audience is the entire org.** If the legitimate audience really is everyone-at-the-company, naming a denied audience is theater. The lesson is for cases where the audience is *narrower* than the operator's natural default. Forge's natural default would have been *the whole company*; the constraint narrowed it. A product whose natural default is *only marketing* doesn't need a *denied: not marketing* constraint because the constraint is implicit.

The signal: *does naming who the system isn't for produce focus, or just exclude?* If focus, the constraint is leverage. If exclusion without focus payoff, it's just gatekeeping. The Forge constraint produced focus — leadership-only meant Spanish-first, no UI for self-serve, no real-time, the asymmetry was the product. A constraint that produced *only* exclusion (without those downstream shape decisions) wouldn't have the same leverage.

## What This Story Is *Not* Evidence For

- **Not evidence that all products should be leadership-only.** The audience constraint is *Forge's* leverage; a different product would have a different audience constraint. The lesson is about *naming the constraint and enforcing it structurally*, not about the specific audience.
- **Not evidence that narrow audiences are always better.** Plenty of products earn their cost serving broader audiences. The lesson is *audience choice as a structural decision*, not *narrow audience as inherently superior*.
- **Not evidence that exclusion is virtuous.** Forge isn't excluding non-leaders for elitism's sake; non-leaders are *contributors* (Slack thread replies, BSC entries, KPI updates feed into the leadership-facing capabilities) but not *consumers* of the output. The asymmetry serves the product's shape, not a hierarchy preference.
- **Not evidence that the constraint can't evolve.** A general-audience Forge could ship later, after the leadership-only product has earned the right to expand. The lesson is about *naming the constraint at day one*, not about *the constraint surviving forever*. Evolution is fine; defaulting to *vague audience* at day one is the failure mode.
- **Not evidence that auth-layer enforcement is the only encoding.** Different products will encode the constraint differently — at the routing layer, the UI layer, the data-access layer, the prompts. The lesson is *three encodings, structural enforcement, not implicit*, not *every encoding has to be the auth layer*.
