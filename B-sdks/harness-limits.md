# Harness Limits — MCP Lifecycle, Tool Timing, Delegation Depth

**Source asides:** Ch. 4 (the bookmark that didn't load; AskUserQuestion auto-resolution; supervisor-orchestrator), Ch. 7 (orchestration patterns aside).
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026, against the Claude Code harness used through the manuscript.

## What the body says

Three Ch. 4 war stories turn on harness behaviors the application layer didn't author — and Principle 4 (*the contract enforced at the agent boundary is the real contract*) names the shape. The body deliberately doesn't lock in the harness's name or the exact numbers; this entry has the specifics for the harness the war stories came from.

## The dated specifics

### MCP server lifecycle

MCP servers spawned by the Claude Code harness are *session-bound*. The harness starts them when a session begins and shuts them down when the session ends. URLs printed by an MCP tool's response only work for the duration of the session that emitted them.

**Implication for tool authors:** anything in a tool response that names a resource (URL, file path, job ID, session token) needs its lifecycle to match the resource's, not the call's. The canvas-mcp viewer was the cleanest case — see `war-stories/canvas-mcp-viewer-lifecycle.md` in the manuscript repo.

### AskUserQuestion auto-resolution

The harness's `AskUserQuestion` tool auto-resolves about **89 milliseconds** after the model emits the call, in the absence of an external answer over the protocol. The auto-resolution result is an *error* result, not a default-answer result.

**Implication for GUI authors:** there is no honest window for an external UI to render the question, capture a user click, and post a response back over the protocol. Building an interactive picker against this tool ships a workaround that breaks the next time harness timing changes. The coide read-only card is the right shape — see `war-stories/coide-askuserquestion.md`.

### Sub-agent delegation depth

The harness's `Agent` tool — the thing that spawns sub-agents — is *only available inside the top-level session*. Sub-agents do not receive the `Agent` tool, which means **delegation depth is 1**. A supervisor-orchestrator pattern is structurally impossible in this harness.

**Implication for orchestration design:** the orchestrator role lives in the top-level session, full stop. No org chart at the sub-agent layer. See `war-stories/markdown-toolkit-subagent-depth.md`.

## TODO (subsequent passes)

- Confirm these limits against the current Claude Code release. They have already shifted once between the Ch. 4 incidents and the Ch. 7 writeup; expect further drift.
- Add equivalent entries for other harnesses (Cursor agent mode, Anthropic API agent loops, custom in-process loops). Every harness has its own version of these limits; the pattern in Ch. 4 is to *read the boundary*, not assume.

## What survives the harness changing

The principle from Ch. 4: *the contract enforced at the agent boundary is the real contract, regardless of what the docs or the application layer claim.* The numbers above will change; the discipline of reading the boundary before drafting against it doesn't.

## Cross-references

- Ch. 4 (Tool Surfaces and Trust Boundaries) — full chapter treatment.
- Ch. 7 (Orchestration Patterns) — the pre-flight check that asks delegation depth before drafting.
- [`harness-affordances.md`](harness-affordances.md) — the harness's positive affordances (skill flags, auto-memory), as opposed to its limits.
