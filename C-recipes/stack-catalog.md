# Stack Catalog — Where the War Stories Come From

**Last reviewed:** 2026-07-31.
**Valid as of:** Q3 2026.

## What this entry is

A reader hits a war story in the body, asks *what stack did this happen on, and
does the lesson port to mine?* — this is where they land. The body keeps stack
details out of prose where they're not load-bearing; this is the index.

Unlike the other C-recipes, this entry doesn't carry implementations. The
implementations live in `audit-pipeline-typescript-mcp.md`,
`observability-across-stacks.md`, `team-primitives-cross-vendor.md`, and
`runtime-trust-patterns.md`. This entry is a routing table: *here's the project,
here's the stack, here are the war stories, here are the chapters they anchor.*

## How to read the catalog

Each project lists, in order:

- **Stack** — language, framework, runtime, deploy target.
- **Date range** — when the incidents in the corpus happened.
- **Repo state** — public or private, where the source lives.
- **War-story files** — the specific writeups in `war-stories/` keyed to the project.
- **Anchors** — the chapters those stories show up in.

If you're trying to *find* a war story by topic, use [`../war-stories/README.md`](../war-stories/README.md)
— the collection's index, organized by project. If you're trying to *understand
the production envelope* a story ran in, this catalog is the routing layer.

## Stacks

### Ruby on Rails + Sidekiq

#### Forge — Virtual Employee Platform

- **Stack:** Ruby on Rails 8, PostgreSQL, Sidekiq, Heroku. Slack as the primary delivery channel; Anthropic API for LLM calls.
- **Date range:** 2026-04 to present. Pre-launch arc Apr-15 → Apr-29; production launch 2026-04-29.
- **Repo state:** Private. MagmaLabs internal.
- **War-story files:**
  - `forge-max-tokens.md` — the half-sentence brief (silent truncation at `max_tokens`).
  - `forge-prompt-cache-minimum.md` — the cache that wasn't (a silent infrastructure no-op below the cache floor).
  - `forge-delivery-boundary.md` — three Slack-rendering bugs unified as the *delivery boundary* pattern.
  - `forge-layered-status.md` — *the function returned ≠ the work completed* (routing-vs-execution layer conflation).
  - `forge-async-adapter-smoke-test.md` — the runner that returned before the job (dev `:async` adapter silently dropped `perform_later` jobs).
  - `forge-audience-constraint.md` — the audience constraint as leverage (scoping to leadership pre-decided three product layers away).
  - `forge-channel-id-regex.md` — the regex that made up Slack's contract (legacy `G…` channel IDs rejected).
  - `forge-idempotency-stealth-state.md` — the routine that said it ran (no operator-visible deduped-vs-fired signal).
  - `forge-keyword-router-natural-language.md` — the keyword router met the Spanish question (LLM fallback for natural language).
  - `forge-ops-analyst-launch-attack.md` — the launch attack that found no authority to hijack (injection/jailbreak containment on a read-only capability).
  - `forge-redaction-never-fired.md` — the gate that hasn't fired (a security control that's spec-covered but never exercised in production).
- **Anchors:** Ch. 1 (The Shift), Ch. 3 (Context as a Resource), Ch. 5 (Eval Loops Are the Product), Ch. 6 (Failure Modes Catalog), Ch. 7 (Orchestration Patterns), Ch. 8 (Observability for Probabilistic Systems), Ch. 12 (Security and Trust Under Adversarial Input).
- **Related appendix entries:**
  - [`../B-sdks/anthropic-prompt-caching.md`](../B-sdks/anthropic-prompt-caching.md)
  - [`observability-across-stacks.md`](observability-across-stacks.md) (Implementation 1 + 2)
  - [`runtime-trust-patterns.md`](runtime-trust-patterns.md) (Layer 3 source)

#### leads-crm — Sales Pipeline CRM

- **Stack:** Ruby on Rails 8, PostgreSQL, Hotwire, Devise + Google OAuth + Pundit, RSpec. Heroku-deployed. Remote MCP endpoint at `/mcp`.
- **Date range:** 2025-11-26 (repo init) to present.
- **Repo state:** Private. MagmaLabs internal.
- **War-story files:**
  - `leads-crm-org-scoping.md` — UI-enforced invariants leaking when a second surface lands (every non-UI write path bypassed org scoping).
  - `leads-crm-mcp-second-surface.md` — the 1-line HTTPS fix that revealed *the tool surface is a second product*.
  - `leads-crm-codex-pr1.md` — cross-vendor agent disagreement as a review primitive.
  - `leads-crm-vision-as-agent-memory.md` — `VISION.md` as a gating artifact plus a `/ship-feature` skill.
  - `leads-crm-directory-linking-backfill.md` — ninety-one files and no backfill (a feature commit that shipped no migration for existing rows).
  - `leads-crm-heroku-boot-deployment-contracts.md` — the Rails app that ran until it deployed (deployment-only contracts the dev env hid).
  - `leads-crm-settings-toggle-false-path.md` — the invisible false case (a Rails `check_box` missing its hidden field; unchecked toggles silently didn't persist).
- **Anchors:** Ch. 4 (Tool Surfaces and Trust Boundaries), Ch. 6 (Failure Modes Catalog), Ch. 7 (Orchestration Patterns), Ch. 8 (Observability for Probabilistic Systems), Ch. 11 (Building the Team Around Agentic Systems).
- **Related appendix entries:**
  - [`observability-across-stacks.md`](observability-across-stacks.md) (Implementation 4)
  - [`team-primitives-cross-vendor.md`](team-primitives-cross-vendor.md) (Primitives 1 + 2)

#### magma-core — Internal Hiring / Recruiting Platform

- **Stack:** Ruby on Rails, PostgreSQL. Candidate funnels, BambooHR sync, screening. The incident is about the multi-agent *review harness* run against the repo (Claude Code fan-out), not the product code.
- **Date range:** The orphaned-sub-agent incident surfaced 2026-07-06 during a PR-review session.
- **Repo state:** Private. MagmaLabs internal.
- **War-story files:**
  - `magma-core-orphaned-subagents.md` — the Five-Hour Ghost (a fan-out review left two sub-agents running 5+ hours after the parent finished; teardown-ownership via a self-propagating lifecycle contract).
- **Anchors:** Ch. 7 (Orchestration Patterns) — teardown/lifecycle ownership; Ch. 8 (Observability for Probabilistic Systems) — *unaccountable by default*.
- **Related appendix entries:**
  - [`observability-across-stacks.md`](observability-across-stacks.md) (lifecycle/cost-side observability)

### Node / TypeScript

#### canvas-mcp — AI Design Canvas (open source)

- **Stack:** TypeScript monorepo, pnpm workspaces, Zod schemas, MCP SDK over stdio, local viewer over HTTP. Vanilla canvas-rendering, no third-party design library.
- **Date range:** Phase 1 in 2026-03; Phase 2 + 3 on 2026-03-21. The viewer-lifecycle bug surfaced 2026-04-11.
- **Repo state:** Public. Open source.
- **War-story files:**
  - `canvas-mcp-viewer-lifecycle.md` — *the tool that lies about state it doesn't own* (viewer URLs 404 after the session ends).
  - `canvas-mcp-two-shadow-apis.md` — parallel sub-agents shipping incompatible APIs that now have to be supported forever.
  - `canvas-mcp-base64-png-context.md` — the bytes that stayed in the conversation (base64 PNGs compounding in context; return URLs instead).
  - `canvas-mcp-human-watching-is-customer.md` — the human watching is the customer (the operator is a co-consumer of the tool's output).
  - `canvas-mcp-half-built-evaluate.md` — 629 lines of implication (an uncommitted `evaluate.ts` implies a scorer the project doesn't ship).
  - `canvas-mcp-feature-multiplicative-renderer.md` — phase 1 mockups after phase 3 (per-phase tests never cross-validate older output).
  - `canvas-mcp-design-md-colors-parser.md` — box shadows in the colors map (a parser that validated shape, not meaning).
- **Anchors:** Ch. 3 (Context as a Resource), Ch. 4 (Tool Surfaces and Trust Boundaries), Ch. 5 (Eval Loops Are the Product), Ch. 6 (Failure Modes Catalog), Ch. 7 (Orchestration Patterns), Ch. 9 (Agent vs. Script vs. Human).

#### coide — Desktop GUI for Claude Code

- **Stack:** Electron + TypeScript wrapping the Claude Code CLI. Child processes managed via `child_process.spawn` with stream-json input.
- **Date range:** Ongoing project. The AskUserQuestion incident landed 2026-05-08; memory-drift surfaced shortly after.
- **Repo state:** Private. MagmaLabs internal.
- **War-story files:**
  - `coide-askuserquestion.md` — the picker that couldn't exist (the CLI auto-resolves the tool in ~89ms).
  - `coide-ship-feature.md` — the skill that auto-invoked (autonomous side-effects without a consent gate).
  - `coide-memory-drift.md` — persistent memory drifting from current code.
- **Anchors:** Ch. 3 (Context as a Resource), Ch. 4 (Tool Surfaces and Trust Boundaries), Ch. 6 (Failure Modes Catalog), Ch. 7 (Orchestration Patterns), Ch. 8 (Observability for Probabilistic Systems), Ch. 9 (Agent vs. Script vs. Human).
- **Related appendix entries:**
  - [`../B-sdks/harness-limits.md`](../B-sdks/harness-limits.md) (AskUserQuestion auto-resolution window)
  - [`observability-across-stacks.md`](observability-across-stacks.md) (Implementation 3)

#### markdown-toolkit — Chrome Extension Build Playbook

- **Stack:** WXT (Manifest V3 Chrome extension), TypeScript, React 19. Spreadsheet-to-markdown parser under unit test. Six role-specialized Claude Code sub-agents in `.claude/agents/`.
- **Date range:** Side project. The orchestration playbook was first exercised 2026-05-04.
- **Repo state:** Private. Solo developer.
- **War-story files:**
  - `markdown-toolkit-subagent-depth.md` — sub-agents can't spawn sub-agents.
  - `markdown-toolkit-reviewer-sweep.md` — *the reviewer sweep that wasn't* (a grep-zero acceptance criterion).
  - `markdown-toolkit-role-separation-solo.md` — role separation, not team size.
  - `markdown-toolkit-cellText-vestigial-selector.md` — the selector no one questioned (a vestigial `tr` that survived every review).
  - `markdown-toolkit-diagnostic-badge.md` — the badge that made the bug landable (*"9 rows · 1 col"* user-side observability).
- **Anchors:** Ch. 4 (Tool Surfaces and Trust Boundaries), Ch. 6 (Failure Modes Catalog), Ch. 7 (Orchestration Patterns), Ch. 8 (Observability for Probabilistic Systems), Ch. 10 (When to Trust the Output), Ch. 11 (Building the Team Around Agentic Systems).
- **Related appendix entries:**
  - [`../B-sdks/harness-limits.md`](../B-sdks/harness-limits.md) (sub-agent delegation depth)
  - [`team-primitives-cross-vendor.md`](team-primitives-cross-vendor.md) (Primitives 1 + 3)
  - [`runtime-trust-patterns.md`](runtime-trust-patterns.md) (Layer 1 source)

#### presentation-studio-mcp — DeckSpec Renderer (open source)

- **Stack:** TypeScript monorepo (pnpm workspaces), Zod schemas, PptxGenJS renderer, Pillow Python sidecar for image processing, MCP SDK over stdio.
- **Date range:** Repo init 2026-04-09. v0.1.0 shipped same day in five commits — the large-monorepo-arrives-fully-formed pattern.
- **Repo state:** Public. Open source.
- **War-story files:**
  - `presentation-studio-mcp-feedback-loops.md` — *the audit step is the eval loop* (warn-and-degrade, not strict-reject, for probabilistic callers).
  - `presentation-studio-mcp-agent-tool-split.md` — *agent decides what to say; tool decides how it looks* (the worker-stays-boring principle).
  - `presentation-studio-mcp-pillow-spawn-per-operation.md` — the leak the agent couldn't see (spawn-per-operation vs. a leaky persistent Pillow worker).
  - `presentation-studio-mcp-jsonrpc-stdio-fallback.md` — sixty lines of insurance (a hand-rolled JSON-RPC fallback for when the SDK breaks).
  - `presentation-studio-mcp-denormalized-brand.md` — the brand that wasn't there (denormalizing brand data so artifacts survive registry drift).
- **Anchors:** Ch. 4 (Tool Surfaces and Trust Boundaries), Ch. 5 (Eval Loops Are the Product), Ch. 6 (Failure Modes Catalog), Ch. 9 (Agent vs. Script vs. Human), Ch. 10 (When to Trust the Output).
- **Related appendix entries:**
  - [`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md) (deep implementation)
  - [`runtime-trust-patterns.md`](runtime-trust-patterns.md) (Layer 2 source)

### Other

#### magmalabs-assistant — Personal COO Workflow

- **Stack:** Not a codebase. The "build" is a Claude Code configuration — custom skills, registered sub-agents, and an MCP stack into Google Workspace, Slack, and Grain, plus a memory layer (`MEMORY.md` and per-skill notes) that carries the operator's rules across sessions.
- **Date range:** Skills accreted over months; the atrophy incidents surfaced across a several-week window in early 2026.
- **Repo state:** Private. Personal configuration; no public artifact.
- **War-story files:**
  - `magmalabs-delegated-skill-decay.md` — *what you delegate is what decays* (operator-side skill atrophy).
- **Anchors:** Ch. 8 (Observability for Probabilistic Systems), Ch. 9 (Agent vs. Script vs. Human), Ch. 11 (Building the Team Around Agentic Systems).
- **Related appendix entries:**
  - [`../B-sdks/harness-affordances.md`](../B-sdks/harness-affordances.md) (auto-memory layer, consent gates)
  - [`team-primitives-cross-vendor.md`](team-primitives-cross-vendor.md) (Primitive 4 — operator-atrophy discipline)

## The frontier model

The body deliberately doesn't name the model in prose (the Reader Contract in
Ch. 2). The appendix does. The war stories ran against current frontier hosted
models from Anthropic: Forge's leadership briefs called Claude Sonnet 4.6, and
the Claude Code harnesses (coide, markdown-toolkit, magma-core review, the
operator stack) ran on whatever the CLI defaulted to across the 2026-03 →
2026-07 window. The specific model IDs, per-capability defaults, and current
frontier lineup live in [`../A-models/frontier-models-2026.md`](../A-models/frontier-models-2026.md),
dated so the specifics stay current as the lineup turns over.

## A note on stack diversity

The corpus deliberately covers Ruby on Rails + Sidekiq and Node / TypeScript as
the two primary stacks. Python is absent on purpose — most AI/agent books
default to Python, and the body's principles are designed to hold across
ecosystems. If a pattern only makes sense in Python, it's a Python idiom dressed
up as a principle.

The two stacks were also chosen for what they're *good and bad at*:

- **Ruby on Rails + Sidekiq** is mature, opinionated, and async-by-convention. The half-sentence brief, the layered-status bug, the org-scoping leak, the MCP-second-surface bug, and the deployment-only contracts all surface failure modes that ride on Rails's strengths — the framework's helpfulness can hide a layering bug, and the implicit conventions can mask a defect until production.
- **Node / TypeScript** is composable, less opinionated, async-by-default. The viewer-lifecycle bug, the two shadow APIs, the half-built `evaluate.ts`, the AskUserQuestion auto-resolution, the memory drift, and the audit pipeline all surface failure modes that ride on Node's strengths — smaller building blocks, more composable, easier to wire up parallel sub-agent work.

Both stacks are *load-bearing examples*, not the point. The point is that the
principles port. If the body claims something works the same way on Rails and on
Node, the war stories from both stacks are the evidence.

## What survives the stack catalog changing

The specific stacks will rotate; the family of failure modes — layering, gaps
between layers, lack of runtime signal, unowned lifecycle, unenforced authority
— shows up in every probabilistic system. The catalog's value is the lens, not
the stack list. A reader in 2030 looking at this catalog should be able to
substitute their own stack for any of the projects above and the lessons should
still apply.

## Cross-references

- [`../war-stories/README.md`](../war-stories/README.md) — the collection's index, organized by project. This catalog is the *stack-routing* view; the war-stories index is the *by-project* view.
- [`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md) — deep recipe grounded in presentation-studio-mcp.
- [`observability-across-stacks.md`](observability-across-stacks.md) — deep recipe spanning Forge, leads-crm, and coide.
- [`team-primitives-cross-vendor.md`](team-primitives-cross-vendor.md) — deep recipe spanning leads-crm, markdown-toolkit, and magmalabs-assistant.
- [`runtime-trust-patterns.md`](runtime-trust-patterns.md) — deep recipe spanning markdown-toolkit, presentation-studio-mcp, and Forge.
- [`../A-models/frontier-models-2026.md`](../A-models/frontier-models-2026.md) — the models the war stories ran against.
