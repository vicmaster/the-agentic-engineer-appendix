# Stack Catalog — Where the War Stories Come From

**Source aside:** Ch. 6 (failure modes catalog).
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026.

## What this entry is

A reader running into a war story in the body and asking *what stack did this happen on, and does the lesson port to mine?* should land here. The body keeps stack details out of prose where they're not load-bearing; this is the index.

## The stacks

### Ruby on Rails + Sidekiq

- **Forge** (Virtual Employee Platform). The leadership-brief story (Ch. 1, Ch. 3), the layered-status bug (Ch. 6, Ch. 8), the delivery-boundary pattern (Ch. 7), the prompt-cache-floor story (Ch. 3, Ch. 8).
- **leads-crm** (Rails CRM). The UI-enforced invariants leak (Ch. 6), the MCP-second-surface observability gap (Ch. 8), the `VISION.md` gating-artifact pattern (Ch. 11), the cross-vendor PR review pattern (Ch. 11).

### Node / TypeScript

- **canvas-mcp** (open-source MCP server for AI-driven design). The viewer-URL lifecycle bug (Ch. 4, Ch. 6), the two-shadow-APIs orchestration disaster (Ch. 7), the half-built `evaluate.ts` cautionary tale (Ch. 5, Ch. 9), the base64-PNG context-economics story (Ch. 3).
- **coide** (desktop GUI around an agent CLI). The AskUserQuestion auto-resolution protocol limit (Ch. 4, Ch. 7), the memory-drift / *plausible drift* story (Ch. 6, Ch. 8).
- **markdown-toolkit** (Chrome extension build playbook). The sub-agent delegation-depth limit (Ch. 4, Ch. 7), the reviewer-sweep partial-completeness failure (Ch. 6, Ch. 7, Ch. 10), the role-separated playbook with branch-prefixed agent IDs (Ch. 11).
- **presentation-studio-mcp** (DeckSpec renderer). The audit-pipeline-as-eval-loop pattern (Ch. 5), the normalize-and-warn discipline (Ch. 5, Ch. 9).

### Other

- **magmalabs-assistant** (personal COO automation). The use-of-agents pattern from Ch. 9 and Ch. 11.

## The frontier model

All war stories ran against a current frontier hosted text model. The model is named in [`../A-models/frontier-models-2026.md`](../A-models/frontier-models-2026.md).

## What survives the stack catalog changing

The principle from Ch. 6: *state opacity.* The specific stacks will rotate; the family of failure modes — layering, gaps between layers, lack of runtime signal — shows up in every probabilistic system. The catalog's value is the lens, not the stack list.

## Cross-references

- Ch. 6 (Failure Modes Catalog) — the chapter the catalog anchors.
- `war-stories/index.md` in the manuscript repo — the per-story incident writeups, with date and code references.
- Other entries in this appendix section index specific recipes per stack.
