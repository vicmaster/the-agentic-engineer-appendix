# leads-crm / The Tool Surface Is a Second Product

## Date / Version Context

- **Project:** Lead CRM — Rails 8 / PostgreSQL / Hotwire app for sales lead qualification (Lead → Pre-Qualified → MQL → SQL → Prospect). Devise + Google OAuth + Pundit + RSpec. Internal at MagmaLabs. Repo seeded 2025-11-26; ~63 commits over ~23 weeks.
- **Three-act commit timeline:**
  1. **2026-04-04 — `9d8d1f6`** "feat: add REST API (/api/v1/) with token auth and standalone MCP server." 76 files, 2,128 insertions, 15 MCP tools, token auth, API specs, `.claude/agents/rails-developer.md`. Local stdio MCP only.
  2. **2026-05-06 — `27c04e6`** "Add remote MCP endpoint." Shifted from local stdio to a deployed remote endpoint at `/mcp`, API token UI in Settings, RackCrmClient, MCP request specs, Claude connector docs.
  3. **2026-05-08 — `77a026b`** "Use HTTPS for internal MCP API calls." One-line patch to the internal Rack request URL. Production-only failure surfaced after SSL was forced.
- **Tool surface for this story:** the MCP layer over the REST API. 15 tools (`search_leads`, `get_lead`, `create_lead`, `update_lead`, `delete_lead`, `promote_lead`, `recalculate_score`, `send_email`, `list_tasks`, plus reports, settings, activity logs).
- **Glossary, used in this writeup:** *MCP layer* = the Model Context Protocol surface that exposes Lead CRM capabilities to external agents (Claude, etc.) as callable tools. *Internal Rack request* = a Ruby-side HTTP call from the MCP handler back into the Rails API, originally written assuming localhost. *Second product surface* = the framing this story names — the tool layer is not glue between agent and existing app, it's a distinct product with its own auth, transport, contract, and failure modes.

## What Was Being Attempted

Expose the CRM's business logic to agents.

The REST API came first (2026-04-04), and it was clean: bearer-token auth, CRUD on every pipeline stage, reports, CORS, 67 specs. The MCP server was deliberately small — a stdlib-only Ruby process that wrapped the API and presented 15 tools. It avoided Rails boot for local use, kept the tool contract changeable while it was settling, and ran over stdio.

That worked. Local agent calls hit local stdio MCP, which called the local Rails API. Everything in one developer's process tree. No transport surprises.

Five weeks later (2026-05-06) the second act: a remote MCP endpoint at `/mcp`. Now external agents (Claude with a remote connector configured) could hit Lead CRM in production. API tokens gettable from the Settings UI. RackCrmClient handling the in-process side. Request specs covering the MCP path. The Claude connector docs spelled out how to configure the agent.

The plan was: the MCP layer is a thin adapter on top of the existing API. It re-uses the auth, the policies, the controllers. The hard work is in the API. The MCP layer is glue.

That framing is what produced the third commit.

## What Went Wrong

`77a026b`. One line. "Use HTTPS for internal MCP API calls."

The MCP handler in production was making an internal Rack request back into the Rails API. The URL was `http://...`. Rails was forcing SSL in production. The internal call went to HTTP, hit the SSL redirect, returned a 301, and the MCP tool call failed in a way that didn't look like a transport failure — it looked like the API was unreachable.

The fix was changing the URL scheme. The size of the diff is the size of the actual code change.

The size of the *gap the bug exposed* is the writeup.

The MCP layer had been built and deployed for two days before the failure surfaced. It looked fine in dev (no SSL forcing). It looked fine in CI (request specs hit the in-process Rails app directly, not via internal URL). It only broke in production, under the specific combination of: deployed remote MCP endpoint, SSL forced at the application layer, and the MCP handler doing an internal URL-based request rather than an in-process call.

The bug was invisible to every existing test because every existing test was running in a context where the internal URL matched the runtime context. Production was the first context where the internal URL had to *actually function as a network address*. SSL was the constraint that forced the contradiction.

## How It Was Discovered

Operationally — an agent call to a tool returned an error, traced through the production logs, and the SSL redirect was the proximate cause. The fix took less time to write than this paragraph.

What that discovery channel revealed is more interesting: the MCP layer had no observability of its own. It was being treated as a passthrough — call goes in, API responds, tool result returns. The transport between the MCP handler and the API was implicit, untested at the edge, and silently dependent on production assumptions (HTTP-only) that weren't true in production.

If the MCP layer had been instrumented with the same care as the REST API — request/response logs, error rates, transport-level alerting — the SSL redirect would have surfaced as a 301-rate spike rather than as a confusing "tool call failed" report.

## What Fixed It

The 1-line patch changed the internal URL scheme from `http` to `https`. Done.

The deeper fix — implicit, structural, owed but not landed in this commit — is the framing change: *the MCP server is not an integration layer; it's a second product surface with its own auth, transport, and failure modes*. The patch is what teaches you that lesson; the lesson is what stops you from shipping the next three versions of the same bug.

What hasn't landed yet (and is owed): MCP-specific request/response logging, alerting on tool-call error rates, and request specs that exercise the deployed remote-MCP path end-to-end (not just the in-process handler). The 1-line fix made the symptom go away. The mental-model flip is what would have prevented the symptom from existing.

## The Durable Lesson

When you expose business logic to agents via MCP, REST, or any tool protocol, you don't add a layer. You add a second product.

The "thin adapter" framing — *the MCP layer just wraps the API* — sounds like it's saving you work. It actually hides the work. A second product surface has its own:

- **Auth.** Even if the auth re-uses the same tokens, the *flow* is different. Humans get tokens via UI; agents get tokens via config. The expiration semantics, rotation cadence, and revocation paths are not the same problem.
- **Transport.** The MCP tool call goes over a different protocol than the human web UI. The transport's failure modes (network partitions, SSL mismatches, timeouts) are *new* failure modes for the same business logic.
- **Contract.** Agents don't read errors well. The 4xx that a human user would see and parse is, to an agent, an opaque blob it may retry, hallucinate around, or paper over. The error contract for the agent surface is its own design problem (see the companion story `presentation-studio-mcp-feedback-loops.md` for the corollary).
- **Observability.** Tool-call telemetry is a separate product surface. "The API is healthy" doesn't tell you "the agent layer is healthy." They can diverge silently — the agent layer can be returning 301s while the API is fine.

The 1-line HTTPS fix is the artifact that names the framing. Everything before it was *the API works locally and from CI, so the MCP layer must work*. Everything after it is *the MCP layer is its own production system, with its own deployment-shaped contracts*.

> **Heuristic.** When you ship a tool surface (MCP, REST-for-agents, GraphQL endpoint, function-call API), ask: *what would the runbook for this surface look like if it failed at 3am?* If the answer is "I'd debug the underlying API," you're treating it as glue. If the answer includes the surface's own auth flow, transport assumptions, error contract, and observability, you're treating it as a product. Only the second framing survives the first production incident.

The shape generalizes beyond MCP. Any agent-callable interface — function-calling on top of an LLM, tool plugins for ChatGPT, custom OpenAPI specs for an agent harness — is a second product surface. The cost of treating it as glue is invisible until production exposes one of the constraints (SSL, rate limits, network shape, error semantics) that the glue framing assumed away.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *agent-callable surfaces in production*. It doesn't apply when:

- **The agent surface is local-only and short-lived.** A stdio MCP server you run in dev to prototype tool contracts is genuinely glue. The transport is your shell, the auth is your filesystem, the failure mode is your terminal. No production constraints exist yet, and engineering for them is premature.
- **The surface is one-shot scaffolding for a single experiment.** If you're going to throw the integration away in two weeks, the second-product investment doesn't pay back. The 1-line HTTPS fix would be a "weird, fix it" annotation in your notes, not a structural lesson.
- **The agent and the underlying app share their entire production envelope.** A function the agent calls in-process in the same Rails app, with the same auth and the same logs, is not crossing a transport boundary. The MCP framing applies the moment a transport (or process, or deploy) gets between the agent's call and the app's response — not when there's literally one call stack.

The signal: *is my tool surface deployed independently, or hit over a network, or run in a different process than the underlying app?* If yes, it's a second product. If no, it's still possibly a second product (auth flows can diverge in-process), but the urgency is lower.

## What This Story Is *Not* Evidence For

- **Not evidence that thin MCP layers are bad.** The stdlib-only Ruby MCP server (Phase 1) was the right call for the period when the tool contract was changing weekly. Avoiding Rails boot kept iteration fast. The lesson is about *what changes when you deploy*, not about whether to keep the implementation small.
- **Not evidence that re-using API auth is wrong.** The bearer tokens are the right shape; what's missing is treating the *agent's auth flow* (token-from-Settings-UI, paste-into-connector-config) as a distinct UX surface. Re-use the mechanism, design the flow.
- **Not evidence that MCP-specific bugs are common.** SSL/HTTPS is a Rails-shaped trap; the lesson generalizes to any tool surface, not specifically to MCP. The reason MCP is on the label is that the *new* deployment-shaped contracts arrived at the same time as the MCP rollout — the layer made the constraints visible, but it didn't cause them.
- **Not evidence that the deeper fix has landed.** The 1-line HTTPS patch is in. The MCP-specific observability and the deployed-transport request specs are the structural follow-ups; they're owed and not yet shipped. This story is told mid-arc — the lesson is named, but the prevention infrastructure isn't fully built yet.
