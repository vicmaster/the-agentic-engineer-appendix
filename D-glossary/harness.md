# Harness

**One-line:** The runtime that wraps an agent loop, mediating between the model, its tools, and the world.

**First introduced:** Ch. 4, *Tool Surfaces and Trust Boundaries*.
**Load-bearing in:** Ch. 4, Ch. 5 (eval loops), Ch. 6 (failure modes), Ch. 7 (orchestration), Ch. 8 (observability), Ch. 9 (executor matching), Ch. 11 (team), Ch. 12 (where authority and least-privilege are enforced).

## The book's use

A *harness* is the system that hosts an agent loop. It owns the things that aren't the model and aren't your tool — the parts most teams don't think about until they bite.

A harness is responsible for, at minimum:

- **Loop orchestration.** Calling the model, reading the output, routing tool calls, feeding results back into the next turn.
- **Tool dispatch.** Deciding which tool runs for which model-emitted call, with what arguments, against what permissions.
- **Lifecycle.** Process boundaries, session state, when subprocesses live and die, what state survives a session.
- **Protocol limits.** Timing, message order, auto-resolution windows, delegation depth, how deep sub-agents can spawn other sub-agents.
- **Architectural rules.** What graph shapes are allowed (Ch. 4, Ch. 7).

## Why the term gets a name

Every harness enforces a contract that's *partly* in the tool schemas you wrote, *mostly* outside them — and the part outside is what produces the failure modes Ch. 4 catalogs. *The contract enforced at the agent boundary is the real contract* (Principle 4) is a claim about harnesses.

In practice, a harness is one of: an agent CLI (Claude Code, Cursor's agent mode, an open-source agentic shell), an SDK's agent loop (Anthropic, OpenAI, the major frameworks), a custom in-process loop your team wrote, or a vendor's hosted "agents" runtime. The book stays neutral; the principles apply to all of them.

## What the term deliberately excludes

- **Just the model.** The model isn't the harness. The model produces text; the harness decides what to do with the text.
- **Just your tools.** Your MCP server or function-calling tool isn't the harness either. The harness *calls* your tools.
- **The whole product.** A user-facing agent app contains a harness; the app is not the harness.

## Common cross-references

- **MCP** (Model Context Protocol) is one protocol harnesses speak to tools — first used in Ch. 3.
- The *eval harness* (see [eval-harness](eval-harness.md)) is unrelated to the agent harness; the shared word *harness* is a coincidence of naming.

## Last reviewed

2026-05-12.
