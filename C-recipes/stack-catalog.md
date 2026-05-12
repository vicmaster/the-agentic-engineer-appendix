# Stack Catalog — Where the War Stories Come From

**Source aside:** Ch. 6 (failure modes catalog).
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026.

## What this entry is

A reader hits a war story in the body, asks *what stack did this happen on, and does the lesson port to mine?* — this is where they land. The body keeps stack details out of prose where they're not load-bearing; this is the index.

Unlike the other C-recipes, this entry doesn't carry implementations. The implementations live in `audit-pipeline-typescript-mcp.md`, `observability-across-stacks.md`, `team-primitives-cross-vendor.md`, and `runtime-trust-patterns.md`. This entry is a routing table: *here's the project, here's the stack, here are the war stories, here are the chapters they anchor.*

## How to read the catalog

Each project lists, in order:

- **Stack** — language, framework, runtime, deploy target.
- **Date range** — when the incidents in the corpus happened.
- **Repo state** — public or private, where the source lives.
- **War-story files** — the specific writeups in `war-stories/` keyed to the project.
- **Chapters anchored** — where each story shows up in the body.

If you're trying to *find* a war story by topic, use `war-stories/index.md` in the manuscript repo — it's organized by failure shape and principle. If you're trying to *understand the production envelope* a story ran in, this catalog is the routing layer.

## Stacks

### Ruby on Rails + Sidekiq

#### Forge — Virtual Employee Platform

- **Stack:** Ruby on Rails 8, PostgreSQL, Sidekiq, Heroku. Slack as the primary delivery channel; Anthropic API for LLM calls.
- **Date range:** 2026-04 to present. Pre-launch arc Apr-15 → Apr-29; production launch 2026-04-29.
- **Repo state:** Private. MagmaLabs internal.
- **War-story files:**
  - `forge-max-tokens.md` — the half-sentence brief (silent truncation).
  - `forge-prompt-cache-minimum.md` — the cache that wasn't (silent infrastructure no-op).
  - `forge-delivery-boundary.md` — three Slack-rendering bugs unified as the *delivery boundary* pattern.
  - `forge-layered-status.md` — *the function returned ≠ the work completed* (routing-vs-execution layer conflation).
- **Chapters anchored:**

  | Chapter | Story |
  |---|---|
  | Ch. 1 (The Shift) | half-sentence brief |
  | Ch. 3 (Context as a Resource) | half-sentence brief, cache that wasn't |
  | Ch. 5 (Eval Loops Are the Product) | half-sentence brief (real-shaped inputs) |
  | Ch. 6 (Failure Modes Catalog) | layered status, silent truncation |
  | Ch. 7 (Orchestration Patterns) | delivery boundary pattern |
  | Ch. 8 (Observability) | layered status, cache verification record |

- **Related appendix entries:**
  - [`../B-sdks/anthropic-prompt-caching.md`](../B-sdks/anthropic-prompt-caching.md)
  - [`observability-across-stacks.md`](observability-across-stacks.md) (Implementation 1 + 2)
  - [`runtime-trust-patterns.md`](runtime-trust-patterns.md) (Layer 3 source)

#### leads-crm — Sales Pipeline CRM

- **Stack:** Ruby on Rails 8, PostgreSQL, Hotwire, Devise + Google OAuth + Pundit, RSpec. Heroku-deployed. Remote MCP endpoint at `/mcp`.
- **Date range:** 2025-11-26 (repo init) to present. ~63 commits across ~23 weeks at the time of writing.
- **Repo state:** Private. MagmaLabs internal.
- **War-story files:**
  - `leads-crm-org-scoping.md` — UI-enforced invariants leaking when the second surface lands.
  - `leads-crm-mcp-second-surface.md` — the 1-line HTTPS fix that revealed *the tool surface is a second product*.
  - `leads-crm-codex-pr1.md` — cross-vendor agent disagreement as a review primitive.
  - `leads-crm-vision-as-agent-memory.md` — `VISION.md` as a gating artifact + `/ship-feature` skill.
- **Chapters anchored:**

  | Chapter | Story |
  |---|---|
  | Ch. 4 (Tool Surfaces) | MCP second surface, org scoping (secondary) |
  | Ch. 6 (Failure Modes Catalog) | org scoping (UI-enforced invariants) |
  | Ch. 7 (Orchestration Patterns) | VISION.md as the *specialist + surface* shape (secondary) |
  | Ch. 8 (Observability) | MCP second surface (tool-surface observability) |
  | Ch. 11 (Building the Team) | cross-vendor PRs, VISION.md gating artifact |

- **Related appendix entries:**
  - [`observability-across-stacks.md`](observability-across-stacks.md) (Implementation 4)
  - [`team-primitives-cross-vendor.md`](team-primitives-cross-vendor.md) (Primitives 1 + 2)

### Node / TypeScript

#### canvas-mcp — AI Design Canvas (open source)

- **Stack:** TypeScript monorepo, pnpm workspaces, Zod schemas, MCP SDK over stdio, local viewer over HTTP. Vanilla canvas-rendering, no third-party design library.
- **Date range:** Phase 1 in 2026-03; Phase 2 + 3 on 2026-03-21. The viewer-lifecycle bug surfaced 2026-04-11 (commit `3e4201f`).
- **Repo state:** Public. Open source.
- **War-story files:**
  - `canvas-mcp-viewer-lifecycle.md` — *the tool that lies about state it doesn't own* (viewer URLs surviving past their session).
  - `canvas-mcp-two-shadow-apis.md` — parallel sub-agents shipping incompatible APIs that now have to be supported forever.
- **Chapters anchored:**

  | Chapter | Story |
  |---|---|
  | Ch. 3 (Context as a Resource) | base64 PNGs in tool responses (to-mine stub) |
  | Ch. 4 (Tool Surfaces) | viewer URL lifecycle (the canonical example of *tool contracts include resource lifecycle*) |
  | Ch. 5 (Eval Loops Are the Product) | half-built `evaluate.ts` (to-mine), pixel-diff regression (to-mine) |
  | Ch. 6 (Failure Modes Catalog) | viewer URL, two shadow APIs |
  | Ch. 7 (Orchestration Patterns) | two shadow APIs (canonical fan-out failure) |
  | Ch. 9 (Agent vs. Script vs. Human) | half-built `evaluate.ts` (to-mine) |

#### coide — Desktop GUI for Claude Code

- **Stack:** Electron + TypeScript wrapping the Claude Code CLI. Child processes managed via `child_process.spawn` with stream-json input (post-2026-04 architecture).
- **Date range:** Ongoing project. The AskUserQuestion incident landed 2026-05-08; memory-drift surfaced shortly after.
- **Repo state:** Private. MagmaLabs internal.
- **War-story files:**
  - `coide-askuserquestion.md` — the picker that couldn't exist (89ms auto-resolution).
  - `coide-ship-feature.md` — the skill that auto-invoked (autonomous side-effects without consent gate).
  - `coide-memory-drift.md` — persistent memory drifting from current code.
- **Chapters anchored:**

  | Chapter | Story |
  |---|---|
  | Ch. 3 (Context as a Resource) | memory drift (secondary — memory as decaying context) |
  | Ch. 4 (Tool Surfaces) | AskUserQuestion, ship-feature consent gate |
  | Ch. 6 (Failure Modes Catalog) | memory drift (plausible drift along the time axis) |
  | Ch. 7 (Orchestration Patterns) | AskUserQuestion (research delegation that changed scope) |
  | Ch. 8 (Observability) | memory drift (Implementation 3 in `observability-across-stacks.md`) |
  | Ch. 9 (Agent vs. Script vs. Human) | ship-feature (autonomous side-effects) |

- **Related appendix entries:**
  - [`../B-sdks/harness-limits.md`](../B-sdks/harness-limits.md) (AskUserQuestion 89ms window)
  - [`observability-across-stacks.md`](observability-across-stacks.md) (Implementation 3)

#### markdown-toolkit — Chrome Extension Build Playbook

- **Stack:** WXT (Manifest V3 Chrome extension), TypeScript, React 19. Parser in `src/tools/spreadsheet-to-markdown/convert.ts` (~38 unit tests at the time of the FINANZAS bug). Six role-specialized Claude Code sub-agents in `.claude/agents/`.
- **Date range:** Side project. The orchestration playbook was first exercised 2026-05-04 against the FINANZAS bug.
- **Repo state:** Private. Solo developer.
- **War-story files:**
  - `markdown-toolkit-subagent-depth.md` — sub-agents can't spawn sub-agents.
  - `markdown-toolkit-reviewer-sweep.md` — *the reviewer sweep that wasn't* (grep-zero AC pattern).
  - `markdown-toolkit-role-separation-solo.md` — role separation, not team size.
- **Chapters anchored:**

  | Chapter | Story |
  |---|---|
  | Ch. 4 (Tool Surfaces) | sub-agent delegation depth |
  | Ch. 6 (Failure Modes Catalog) | reviewer sweep (partial sweeps treated as complete) |
  | Ch. 7 (Orchestration Patterns) | sub-agent depth, reviewer sweep, role separation |
  | Ch. 10 (When to Trust the Output) | reviewer sweep (Layer 1 — tool-checkable verification) |
  | Ch. 11 (Building the Team) | role separation, cross-vendor branch-name pattern |

- **Related appendix entries:**
  - [`../B-sdks/harness-limits.md`](../B-sdks/harness-limits.md) (sub-agent delegation depth)
  - [`team-primitives-cross-vendor.md`](team-primitives-cross-vendor.md) (Primitives 1 + 3)
  - [`runtime-trust-patterns.md`](runtime-trust-patterns.md) (Layer 1 source)

#### presentation-studio-mcp — DeckSpec Renderer (open source)

- **Stack:** TypeScript monorepo (pnpm workspaces), Zod schemas, PptxGenJS renderer, Pillow Python sidecar for image processing, MCP SDK over stdio. 13 MCP tools, 15 layouts, 12 audit rules, 7 templates.
- **Date range:** Repo init 2026-04-09 (commit `ee61fa1`). v0.1.0 shipped same day in five commits — the large-monorepo-arrives-fully-formed pattern.
- **Repo state:** Public. Open source.
- **War-story files:**
  - `presentation-studio-mcp-feedback-loops.md` — *the audit step is the eval loop* (the chapter's thesis on a plate).
  - `presentation-studio-mcp-agent-tool-split.md` — *agent decides what to say; tool decides how it looks* (the worker-stays-boring principle).
  - `presentation-studio-mcp-source.md` — verbatim self-interview; the source for the §6 hypotheses (flagged inferred-not-remembered).
- **Chapters anchored:**

  | Chapter | Story |
  |---|---|
  | Ch. 4 (Tool Surfaces) | normalize-and-warn (secondary — tool input surface) |
  | Ch. 5 (Eval Loops Are the Product) | audit pipeline as eval loop (primary anchor) |
  | Ch. 9 (Agent vs. Script vs. Human) | agent/tool split (primary positive anchor) |
  | Ch. 10 (When to Trust the Output) | audit pipeline (Layer 2 — per-call audit) |

- **Related appendix entries:**
  - [`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md) (deep implementation)
  - [`runtime-trust-patterns.md`](runtime-trust-patterns.md) (Layer 2 source)

### Other

#### magmalabs-assistant — Personal COO Workflow

- **Stack:** Not a codebase. The "build" is a Claude Code configuration: nine custom skills (`/cockpit`, `/wrap-up`, `/ceo-sync`, `/call-prep`, `/deal-status`, `/standup-notes`, `/weekly-updates`, `/grain-search`, `/follow-up-alerts`), four registered sub-agents (`grain-notes`, `slack-searcher`, `calendar-assistant`, `lead-crm`), and an MCP stack into Google Workspace, Slack, and Grain. Plus a memory layer (`MEMORY.md` and per-skill notes) that carries the operator's rules across sessions.
- **Date range:** Skills accreted over months. The three atrophy incidents surfaced across a several-week window in early 2026.
- **Repo state:** Private. Personal configuration; no public artifact.
- **War-story files:**
  - `magmalabs-delegated-skill-decay.md` — *what you delegate is what decays* (operator-side atrophy).
- **Chapters anchored:**

  | Chapter | Story |
  |---|---|
  | Ch. 8 (Observability) | skill decay (operator-side observability gap — secondary) |
  | Ch. 9 (Agent vs. Script vs. Human) | the corpus's only *use of agents* project (vs. *build of agents*) |
  | Ch. 11 (Building the Team) | skill atrophy (primary — the team-side primitive) |

- **Related appendix entries:**
  - [`../B-sdks/harness-affordances.md`](../B-sdks/harness-affordances.md) (auto-memory layer, consent gates)
  - [`team-primitives-cross-vendor.md`](team-primitives-cross-vendor.md) (Primitive 4 — operator-atrophy discipline)

## The frontier model

All war stories ran against a current frontier hosted text model from a major provider. The specific model name, version, and per-capability defaults live in [`../A-models/frontier-models-2026.md`](../A-models/frontier-models-2026.md). The body deliberately doesn't name the model in prose (Ch. 2's Reader Contract); the appendix names it with `Last reviewed` so the specifics stay dated.

## A note on stack diversity

The corpus deliberately covers Ruby on Rails + Sidekiq and Node / TypeScript as the two primary stacks. Python is absent on purpose — most AI/agent books default to Python, and the body's principles are designed to hold across ecosystems. If a pattern only makes sense in Python, it's a Python idiom dressed up as a principle.

The two stacks were also chosen for what they're *good and bad at*:

- **Ruby on Rails + Sidekiq** is mature, opinionated, and async-by-convention. The half-sentence brief, the layered-status bug, the org-scoping leak, and the MCP-second-surface bug all surface failure modes that ride on Rails's strengths (the framework's helpfulness can hide a layering bug, the implicit conventions can mask the bug until production).
- **Node / TypeScript** is composable, less opinionated, async-by-default. The viewer-lifecycle bug, the two-shadow-APIs, the half-built `evaluate.ts`, the AskUserQuestion auto-resolution, the memory drift, the audit pipeline — all surface failure modes that ride on Node's strengths (smaller building blocks, more composable, easier to wire up parallel sub-agent work).

Both stacks are *load-bearing examples*, not the point. The point is that the principles port. If the body claims something works the same way on Rails and on Node, the war stories from both stacks are the evidence.

## What survives the stack catalog changing

The principle from Ch. 6: *state opacity.* The specific stacks will rotate; the family of failure modes — layering, gaps between layers, lack of runtime signal — shows up in every probabilistic system. The catalog's value is the lens, not the stack list. A reader in 2030 looking at this catalog should be able to substitute their own stack for any of the projects above and the lessons should still apply.

## Cross-references

- Ch. 6 (Failure Modes Catalog) — the chapter the catalog anchors.
- `war-stories/index.md` in the manuscript repo — the canonical per-story index organized by failure shape and principle. This catalog is the *stack-routing* view; the war-stories index is the *failure-shape-routing* view.
- [`audit-pipeline-typescript-mcp.md`](audit-pipeline-typescript-mcp.md) — deep recipe grounded in presentation-studio-mcp.
- [`observability-across-stacks.md`](observability-across-stacks.md) — deep recipe spanning Forge, leads-crm, and coide.
- [`team-primitives-cross-vendor.md`](team-primitives-cross-vendor.md) — deep recipe spanning leads-crm, markdown-toolkit, and magmalabs-assistant.
- [`runtime-trust-patterns.md`](runtime-trust-patterns.md) — deep recipe spanning markdown-toolkit, presentation-studio-mcp, and Forge.
- [`../A-models/frontier-models-2026.md`](../A-models/frontier-models-2026.md) — the model all war stories ran against.
