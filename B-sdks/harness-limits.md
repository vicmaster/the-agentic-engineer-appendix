# Harness Limits — MCP Lifecycle, Tool Timing, Delegation Depth, Context, Permissions

**Source asides:** Ch. 4 (the bookmark that didn't load; AskUserQuestion auto-resolution; supervisor-orchestrator), Ch. 7 (orchestration patterns aside).
**Source incidents:** `war-stories/canvas-mcp-viewer-lifecycle.md`, `coide-askuserquestion.md`, `markdown-toolkit-subagent-depth.md`.
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026, against Claude Code (the harness used through the manuscript). Comparison entries for other harnesses noted inline.

## What the body says

Three Ch. 4 war stories turn on harness behaviors the application layer didn't author — and Principle 4 (*the contract enforced at the agent boundary is the real contract*) names the shape. The body deliberately doesn't lock in the harness's name or the exact numbers; this entry has the specifics for Claude Code as of Q2 2026, the war-story-anchored limits, and a comparison table across the other major agent harnesses.

## How to find limits in a harness

Three layers, in roughly the order you should read:

1. **The release notes and changelog.** Limits surface here first because they're often the things teams complain about after a release. `claude-code/releases` is more honest than the intro docs.
2. **The harness's issue tracker.** The delegation-depth limit (`markdown-toolkit-subagent-depth.md`) was named in an issue thread, not the docs. Search the tracker for `limit`, `maximum`, `depth`, `cannot`, `error`.
3. **The intro docs.** Often the most-stale source for limits because the docs lead with capabilities, not constraints. Useful for finding what's *supposed* to work; less useful for finding what *won't*.

The fastest pre-flight check before drafting an orchestration plan: open the release notes, search for the keywords above, then stress-test the limits the search surfaced.

## Claude Code — Q2 2026

### MCP server lifecycle

**Limit:** MCP servers spawned by Claude Code are *session-bound*. The harness starts them at session begin and shuts them down at session end. Resources the MCP server owns (HTTP servers, background workers, file handles) die with the session.

**War story:** `canvas-mcp-viewer-lifecycle.md`. The canvas-mcp server printed `http://localhost:3001/canvas/abc` URLs in tool responses. A bookmark from yesterday returned `ECONNREFUSED` because the Express server inside the MCP process had died with the session.

**Implication for tool authors:** anything in a tool response that names a resource (URL, file path, job ID, session token) needs its lifecycle to match the resource's, not the call's. Either separate the resource's lifecycle from the call's (split the viewer into a standalone process), or stop printing the resource, or annotate the resource with `lifetime: "session"` so callers can decide.

### AskUserQuestion auto-resolution

**Limit:** `AskUserQuestion` auto-resolves about **89 milliseconds** after the model emits the call, in the absence of an external answer over the protocol. The auto-resolution result is an *error* result, not a default-answer result.

**War story:** `coide-askuserquestion.md`. coide's interactive-picker plan would have shipped a workaround for the 89ms window. The 15-minute research detour caught the limit before the architecture committed.

**Implication for GUI authors:** there is no honest window for an external UI to render the question, capture a user click, and post a response over the protocol. The read-only card is the right shape; the interactive picker requires an upstream change.

### Sub-agent delegation depth

**Limit:** Claude Code's `Agent` tool — the thing that spawns sub-agents — is *only available inside the top-level session*. Sub-agents do not receive the `Agent` tool, which means **delegation depth is 1**. A supervisor-orchestrator pattern is structurally impossible.

**War story:** `markdown-toolkit-subagent-depth.md`. The first orchestration playbook had a `supervisor-orchestrator` sub-agent at the top of the org chart. The pattern was structurally impossible; the fix was a doc rewrite that moved the orchestrator role into the top-level session.

**Implication for orchestration design:** under this limit, the orchestrator role lives in the top-level session. No org chart at the sub-agent layer. Sub-agents do bounded work and return; they do not coordinate.

**Recheck (2026-09):** later Claude Code versions appear to make the spawning tool available to sub-agents as well, so depth = 1 may no longer hold in the version you're running. Run pre-flight item 1 below (a throwaway sub-agent that tries to spawn another) before designing around either answer. Even where nesting works, the Ch. 7 advice to keep the orchestrator's seams in one place still applies.

### Context window

**Limit:** Claude Code sessions hold the full conversation history, system prompt, tool definitions, and tool outputs in the context window. The window size depends on the model — as of Q3 2026, the current frontier models (the Claude 5 family, Opus 4.8, Sonnet 4.6) support roughly **1M tokens** of context, some via a `[1m]` model variant (e.g. `claude-opus-4-8[1m]`); Haiku 4.5 caps at roughly **200K tokens**.

**Implication for long sessions:** the harness automatically compacts long histories when the window fills, summarizing older turns. Compaction is lossy — load-bearing details get dropped. See Ch. 6's *context collapse* sidebar. Mitigation: return pointers, not bytes, from your tools (Ch. 3); start fresh sessions for unrelated work rather than accreting history.

### Tool count per session

**Limit:** A practical ceiling on the number of tools the harness can expose to the model in one session. Above ~50 tools, model performance degrades — tool selection becomes unreliable, latency increases, and the prompt cache hit rate drops because the tool list grows. The hard cap is higher (~200) but the *useful* cap is much lower.

**Implication for MCP design:** if your MCP server exposes 30 tools, adding a second MCP server with 30 more pushes against the soft ceiling. Either consolidate tools (one `database_query` tool that takes an action enum, rather than 12 narrow tools) or split into multiple sessions with different tool subsets.

### File operations

**Limit:** The `Read`, `Edit`, `Write`, `Glob`, `Grep` tools operate in the working directory and its subdirectories. Reading outside the working directory is allowed if the path is absolute, but operations are gated by the harness's permission system. See [`harness-affordances.md`](harness-affordances.md) for the permission flags.

**Read tool quirks:**
- Default read window: 2,000 lines starting from the beginning of the file. Use `offset` and `limit` for larger files.
- Images (PNG, JPG, etc.) are returned as visual content, not text.
- PDFs over 10 pages require the `pages` parameter (e.g. `"1-5"`).
- Empty files return a system reminder, not empty content.

**Edit tool quirks:**
- Edits fail if the file hasn't been read in the current conversation.
- The `old_string` must be unique in the file unless `replace_all: true`.
- Line numbers in the Read output are prefixes (e.g. `42\t<content>`); they're not part of the file content and must not appear in `old_string`/`new_string`.

### Bash tool

**Limit:** Bash commands have a default timeout of **120,000 ms (2 minutes)** and a maximum of **600,000 ms (10 minutes)**. Background commands (`run_in_background: true`) don't count against the timeout but tie up a process slot.

**Common gotchas:** `find`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`, `echo` should use the dedicated tools (`Glob`, `Grep`, `Read`, `Edit`, etc.) — they're optimized for the harness's permission model and provide better output. Bash is reserved for shell-only operations.

## Comparison across other harnesses

The Claude Code limits above are *specific*; every harness has its own version. The table below is approximate as of Q2 2026 — check each harness's current docs.

| Limit | Claude Code | OpenAI Agents SDK | Cursor (agent mode) | Anthropic API (direct loop) |
|---|---|---|---|---|
| MCP server lifecycle | Session-bound | Caller-managed | Editor-bound | Caller-managed |
| Async-input window | 89ms (AskUserQuestion) | N/A (no equivalent tool) | Caller-defined | Caller-managed |
| Delegation depth | 1 (`Agent` tool top-level only) | Configurable (handoffs) | N/A (single agent) | Caller-defined |
| Context window | 200K / 1M | current GPT-tier (provider-dependent) | Editor-managed | Model-dependent |
| Tool count (soft) | ~50 useful | ~64 useful | Editor-curated | Caller-managed |
| Permission system | settings.json (allow/ask/deny) | Per-tool function | Workspace-scoped | None (caller-managed) |
| Auto-memory | Per-project MEMORY.md | None native | None native | None native |
| Sub-agents | Top-level only | Handoff-based agent graph | None | Caller-defined |

The pattern across all rows: **the limit names a constraint the application layer doesn't enforce, the harness does.** The discipline (read the boundary, don't prototype) ports across harnesses; the specific numbers don't.

## Pre-flight checklist for a new orchestration plan

Lifted and expanded from Ch. 4 / Ch. 7. Run this *before* drawing boxes:

1. **What's the maximum delegation depth?** Read the issue tracker, not just the intro docs. Stress-test the deepest delegation you actually need with a throwaway sub-agent that tries to spawn another sub-agent.
2. **What's the lifecycle of resources my tools return?** If a tool emits a URL, file path, or session token, what happens to that resource between calls and across sessions?
3. **Are there auto-resolution windows on tools I'm depending on?** Specifically: any tool that *looks* like it should support an external response (`AskUserQuestion`, picker dialogs, confirmation prompts). The window may be too narrow to be useful.
4. **What's the context budget across a typical session?** If your work fans out into many tool calls, sum the expected tool-output sizes. Compaction starts kicking in around 70-80% of the window.
5. **What's the tool count once all my MCP servers are loaded?** Sum across all servers. Above ~50 tools, plan for either consolidation or session-splitting.
6. **What's the permission model?** What requires explicit operator consent, what defaults to allowed, what defaults to denied? See [`harness-affordances.md`](harness-affordances.md).

A 15-minute pre-flight against this checklist is the cheapest insurance against the Ch. 4 class of failure. See `war-stories/coide-askuserquestion.md` for the canonical small-up-front-research-saves-the-architecture story.

## What survives the harness changing

The principle from Ch. 4: *the contract enforced at the agent boundary is the real contract, regardless of what the docs or the application layer claim.* Every harness will have its own version of these limits. The specific numbers (89ms, depth=1, 1M tokens) will change; the discipline of reading the boundary before drafting against it doesn't.

## Cross-references

- Ch. 4 (Tool Surfaces and Trust Boundaries) — full chapter treatment.
- Ch. 7 (Orchestration Patterns) — the pre-flight check that asks delegation depth before drafting.
- `war-stories/canvas-mcp-viewer-lifecycle.md` — MCP server lifecycle anchor.
- `war-stories/coide-askuserquestion.md` — auto-resolution timing anchor.
- `war-stories/markdown-toolkit-subagent-depth.md` — sub-agent delegation depth anchor.
- [`harness-affordances.md`](harness-affordances.md) — the harness's *affordances* (skill flags, auto-memory, consent gates), as opposed to its limits.
- [`../C-recipes/team-primitives-cross-vendor.md`](../C-recipes/team-primitives-cross-vendor.md) — Primitive 3 (role-separated playbook) is constrained by the delegation-depth limit.
