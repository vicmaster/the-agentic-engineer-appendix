# canvas-mcp / The Tool That Lies About State It Doesn't Own

## Date / Version Context

- **Date:** The bug existed from ~2026-03-25 (commit `a3cf691`, the day the in-process viewer landed) until 2026-04-11 (commit `3e4201f`, the day the standalone viewer + JSON persistence shipped). Roughly seventeen days of leaked promises.
- **Project:** canvas-mcp v0.1.0 — an open-source MCP server that gives any AI assistant a visual design canvas. Stack: Node 20 / TypeScript / `@modelcontextprotocol/sdk` / Express / Puppeteer.
- **Tool surface:** 13 MCP tools over stdio. The relevant ones are `screenshot()` and `viewer_url()`, both of which return URLs like `http://localhost:3001/canvas/abc` pointing at an Express server that ran inside the MCP server process.
- **Glossary, used in this writeup:** *MCP server* = a process the agent's harness (Claude Code, Cursor, etc.) spawns to expose a set of tools. *Stdio transport* = the MCP protocol's primary transport — the harness writes JSON-RPC to the server's stdin and reads from stdout. *Session* = the lifetime of one harness invocation; closing the chat ends the session. *Side-effect emitter* = a tool that does more than transform input to output — it spawns processes, writes files, or implies the existence of resources that outlive the call.

## What Was Being Attempted

Two tightly-coupled things:

1. **Render canvases for an agent to see.** The MCP server takes a JSON description of a design (frames, text, icons), renders it via HTML/CSS in headless Chromium, and produces a PNG.
2. **Show the canvas to a human.** The base64 PNG in the tool response is technically sufficient — the agent can "see" the canvas — but agents aren't the customer. The human watching the agent is. So the project added an Express viewer at `localhost:3001` and started returning viewer URLs in tool responses (`http://localhost:3001/canvas/abc`) instead of (or in addition to) embedded bytes.

The viewer URL flow felt clean: the agent renders, the tool response includes a clickable URL, the human opens it in a browser tab and sees the canvas update live. Better than embedded base64 (which clutters context and can't be re-opened), better than a separate file path (which the human has to know how to render).

It worked in the chat. Designs rendered. URLs were clickable. The human saw the canvas. The agent could iterate.

## What Went Wrong

For about two weeks, every viewer URL the tool printed was a future-tense lie.

The Express viewer ran inside the MCP server process. The MCP server's lifecycle was bound to the Claude session. Close the chat — the harness shut down the MCP server — the Express viewer died with it — every URL pointing at port 3001 went 404.

A user came back the next morning, clicked the bookmark from yesterday's design session, and got `ECONNREFUSED`.

That moment is the load-bearing failure. The tool had been emitting URLs that *only worked while the session that produced them was still alive.* The agent didn't know. The user didn't know. The URL looked like every other URL — stable, addressable, openable later.

It wasn't.

## How It Was Discovered

The user opened a bookmark from a design session the previous day. The browser returned `ECONNREFUSED`. There was no error in any log, no failure surfaced in the chat, no signal anywhere that the URL had decayed. The discovery channel was *a human's expectation that yesterday's URL still worked.*

This is the worst possible discovery channel for a bug. There's no telemetry, no eval, no automated test that catches "the URL my tool returned yesterday is still resolvable today." The bug is invisible until a human exercises the only behavior that surfaces it.

## What Fixed It

Two changes, ~70 lines of new code total, shipped together as commit `3e4201f`.

1. **Split the viewer into a standalone process.** `viewer-standalone.ts` — a separate `bin` entry that runs the Express server *outside* the MCP server's lifecycle. The MCP server starts and stops; the viewer keeps running. The user can leave the viewer process up across sessions.
2. **Persist canvases to disk.** Add `~/.canvas-mcp/canvases/` as a JSON-on-disk store. The MCP server writes canvases there as it produces them. The standalone viewer reads from there to serve URLs. Any session can write; any viewer instance can read; URLs survive the session that produced them.

The fix is structural, not behavioral. The bug wasn't "the viewer crashed sometimes" — it was "the viewer's lifecycle was wrong." Splitting the lifecycle was the only honest fix; everything else would have been a workaround pretending the in-process version could be made durable.

## The Durable Lesson

An MCP tool is not a function. It's a side-effect emitter.

A function takes input, returns output, and the conversation moves on. The return value is the whole contract. Caller doesn't have to think about anything else.

A tool that prints `http://localhost:3001/canvas/abc` is making a *promise about state it doesn't own.* It's claiming that a long-lived server exists and will serve that URL. The return value is a small slice of what the tool is actually doing — under it, processes are being spawned, files are being written, ports are being bound, daemons are being expected to outlive the call.

When the lifecycle of those side-effects doesn't match the lifecycle of the resources the tool is *describing*, the tool is lying.

The durable shape:

> **Heuristic.** Any tool output that names a resource — a URL, a file path, a job ID, a session token — is a claim about that resource's lifecycle. If the lifecycle of what produced the resource is shorter than the lifecycle the resource is implying, the tool is making a promise it can't keep. The fix is structural: separate the resource's lifecycle from the call's lifecycle, or stop printing the resource.

The pattern shows up beyond MCP. Any RPC that returns a URL pointing at a server bound to the request's lifetime has the same problem. Job IDs from a system that loses jobs at restart. Session tokens from a server that doesn't persist sessions. File paths from a sandbox that gets torn down. The shape is universal.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *resources the caller is expected to use after the call returns.* It doesn't apply when:

- **The caller is using the resource immediately, in-band.** A tool that returns a temporary file path the caller passes to the next call within the same session is fine — the lifecycle matches.
- **The output explicitly names its lifetime.** A response shape like `{"url": "...", "expires_at": "...", "lifetime": "session"}` is honest about the constraint. The lie isn't the URL; the lie is the implicit-stable-forever framing of an unmarked URL.
- **The resource is genuinely scoped to the call.** A streamed response chunk, an in-flight job that completes within the call, a transient identifier the caller never persists. The lifecycle is single-call by design.

The signal: *would a reasonable user expect this resource to outlive the call?* If yes, the lifecycle has to match — or the response has to be explicit about the limit.

## What This Story Is *Not* Evidence For

- **Not evidence that all tool outputs should be self-contained.** Embedding the canvas as base64 inside the tool response is what canvas-mcp tried earlier and abandoned for separate context-economics reasons (see the companion story `canvas-mcp-base64-png-context.md`). The right answer to lifecycle mismatch is *match the lifecycle*, not *eliminate the resource*.
- **Not evidence that MCP servers should run forever.** The MCP server's session-bound lifecycle is correct. What was wrong was *binding a separately-lifecycled resource (the viewer) to that lifecycle.* The fix was to give the viewer its own lifecycle, not to extend the MCP server's.
- **Not evidence that this is unique to MCP.** It's the most visible in MCP because tool responses are the primary output channel and the harness lifecycle is short and obvious. The pattern shows up in any RPC system with mismatched resource and call lifetimes.
