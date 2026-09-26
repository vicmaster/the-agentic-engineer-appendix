# canvas-mcp / The Human Watching Is the Customer

## Date / Version Context

- **Date:** The mental-model flip crystallized at the viewer commit `a3cf691` (2026-03-25). The base64-PNG design predates it (the original `screenshot()` return shape was bytes-in-response from Phase 1, 2026-03 onward); the standalone viewer is what made the alternative concrete.
- **Project:** canvas-mcp — open-source MCP server for AI design canvas. Renders structured design specs into visual canvas artifacts. The operator's interactive loop is *agent calls tool, tool produces a canvas, human reviews the canvas, agent iterates based on the operator's feedback*. The human review step is load-bearing — the operator is in the loop, not downstream of it.
- **Surface for this story:** the response shapes of the tools the agent calls while designing. Pre-`a3cf691`: the only visual output was `screenshot()`'s base64 PNG. Post-`a3cf691`: `canvas_create` and `batch_design` also return a URL like `http://localhost:3001/canvas/<id>` into a live viewer (served from the MCP server's in-memory canvases; disk persistence came with `3e4201f` on 2026-04-11), and the operator opens it in a browser and sees the actual canvas.
- **Glossary, used in this writeup:** *Agent-as-customer* = the mental model where the only stakeholder of an MCP tool's output is the calling agent. *Human-watching-the-session* = the operator running the agentic flow, observing the conversation as it unfolds, and reviewing intermediate artifacts. *Agentic ≠ agent-only* = the principle that agentic systems usually have at least two customers (the agent and the operator), and tool design should serve both.

## What Was Being Attempted

Make `screenshot()` return enough information for the agent to reason about the canvas it just rendered.

The product framing was correct in the obvious sense. The agent designs canvases by calling MCP tools (`canvas_create`, `batch_design`, `set_variables`, `apply_preset`, etc.), and the agent needs *some* signal about what the canvas currently looks like to decide its next move. Returning the rendered PNG as base64 bytes was the obvious shape: the agent gets a literal image of the canvas, can describe it, can decide what to change.

In a vacuum, the design works. The agent calls `screenshot()`, gets back a PNG, reasons about the visual, calls the next tool. The MCP protocol supports image-typed responses; Claude's vision is good at reading rendered designs; the loop closes correctly from the agent's perspective.

The framing's blind spot was *who else is in the loop*. The operator running canvas-mcp isn't watching the conversation from outside — the operator is *actively reviewing* what the agent produces. The operator's review surface should be *the rendered canvas itself*, not *the agent's verbal description of the canvas*, and not *an image that lives inside a tool result*, which (depending on the client) the operator may not see rendered at all, and can't open in a browser, bookmark, or keep up on a second screen.

The agent-as-only-customer framing missed this. The tool worked for the agent. The tool worked terribly for the human watching the agent.

## What Went Wrong

The agent could see the canvas. The human couldn't.

The operator's actual session loop, with the pre-`a3cf691` `screenshot()`:

1. Operator asks the agent to make a canvas.
2. Agent calls `canvas_create`, sets variables, etc.
3. Agent calls `screenshot()`.
4. The tool result is an image the model reads. What the operator sees of it depends on the client; it isn't a page they can open, resize, or come back to. The agent adds its narrative description.
5. Operator wants to *see* the canvas to decide what to ask next.
6. Operator either (a) takes the agent's description at face value, (b) asks the agent to `export` the canvas to a PNG file and opens it separately, or (c) ignores the screenshot entirely and asks the agent to *describe* the canvas in more detail.

Each of (a), (b), (c) is friction the operator pays for the *tool's design assumption that only the agent is the customer*. None of the three is acceptable for a real review loop:

- **(a) Take the agent at face value.** Agents are reasonable narrators of what they generate but not reliable enough that *"I made a 3-column dashboard"* is interchangeable with *seeing* the dashboard. Visual review is *visual*; verbal substitutes aren't.
- **(b) Export and open a file.** The agent calls `export` to write a PNG to disk; the operator finds the file and opens it. An extra tool call and a context switch per review. The operator's review velocity drops to whatever the slowest step is.
- **(c) Skip the screenshot.** The screenshot becomes purely decorative — the operator works from the agent's text descriptions and reviews the final output once. Loses the per-step review cadence the agentic loop should support.

The structural shape: **the tool's design only served the agent's loop. The operator's loop was an afterthought.** The viewer commit treats the operator's loop as load-bearing — the tools return a URL the operator can click and see the canvas. The agent no longer needs a PNG in the conversation just so someone can see the design; the operator gains a real review surface. Same fix, two distinct lessons.

## How It Was Discovered

By being the operator and noticing the friction.

The discovery channel is *the operator using their own product* and finding it worse than expected. There's no commit anchor for the *moment* the flip happened — the realization predates `a3cf691` by some amount, and crystallized as the viewer was being built. The way it got phrased later: *"a tool design that optimized only for the agent's needs would never have produced the viewer"* — a retrospective naming of *why this fix exists*, not a contemporaneous bug report.

The non-discovery channel — the part worth pausing on — is that *no agent could have surfaced this*. The agent's experience of `screenshot()` was fine; the agent had no signal that another stakeholder was being underserved. The discovery requires *being the operator and noticing*. Agents don't review their own tools' UX against a population they don't share.

This generalizes: **if you only let the agent test your agentic tools, you'll ship tools that work for the agent and not for the humans who watch.** The operator has to use the tool *as the operator*, with the operator's actual review needs, to notice the friction. Tool design that only consults the agent's experience produces tools that exclude the operator without anyone noticing — because no one is watching from the operator's seat.

## What Fixed It

The viewer commit `a3cf691`, 2026-03-25, and its follow-up `3e4201f`, 2026-04-11. Three layers of fix:

**Run a viewer.** A small HTTP server (the *viewer*) serves `http://localhost:3001/canvas/<id>` for each canvas, rendering it with the same HTML/CSS renderer the screenshots use and polling every two seconds so the page follows the agent's edits. In `a3cf691` it ran inside the MCP server process; `3e4201f` split it into a standalone process.

**Persist canvas state outside the conversation.** In `3e4201f`, each canvas got a JSON file at `~/.canvas-mcp/canvases/<id>.json` (short random IDs, not UUIDs), so the state outlives the MCP server process as well as any single tool call.

**Put the URL in the responses.** `canvas_create` returns a `viewerUrl`, `batch_design` appends a `View live:` link, and a new `viewer_url` tool lists a link for every canvas. A URL is a few dozen bytes. `screenshot()` still returns the PNG for the agent's own looking; the operator gets a clickable link.

The fix is asymmetric in cost. The persistence layer + viewer process is real infrastructure work (a few hundred lines, plus its own lifecycle concerns — see the companion story `canvas-mcp-viewer-lifecycle.md` for the *tool that lies about state it doesn't own* lesson that fell out of the *next* fix in this arc). The agent-loop change is one return-shape adjustment. Most of the cost was acknowledging the operator's loop as load-bearing, then building the surface to serve it.

What's load-bearing isn't the specific URL-not-bytes choice — that's the context-economics lesson (see the companion story `canvas-mcp-base64-png-context.md`). What's load-bearing here is the *framing shift*: from *the agent is the customer* to *the agent and the operator are both customers, and the tool design has to serve both.*

What's owed but not yet shipped: a *canvas-mcp design principle doc* that names the two-customers framing explicitly so future tools (a hypothetical `canvas_compare()` or `canvas_history()`) are designed with both stakeholders in mind from the start. Lives in operator habit at the moment; worth writing down at the project's `README.md` or `CONTRIBUTING.md` level.

## The Durable Lesson

Agentic ≠ agent-only. **The agent calling your tool is one customer; the human watching the agent call your tool is another. Tool designs that satisfy only the agent's needs ship a worse product than tool designs that satisfy both.**

The mental-model flip the lesson rests on: most MCP-tool-design literature implicitly frames the agent as the sole customer. Tool inputs are described as *"what the model passes"*; tool outputs as *"what the model receives back."* The framing has no place for *what the operator sees*, because the operator is treated as external to the tool's interface — a meta-observer of the conversation, not a stakeholder of the tool itself. That framing produces tools like the original `screenshot()`: technically correct for the agent loop, terrible for the operator's actual review needs.

The corrected framing names the operator as a co-customer. For tools that produce artifacts a human will review (canvases, decks, designs, generated reports, screenshots, sound files, code diffs), the tool's response shape has to include a surface the human can *see* — a URL, a file path that opens in the OS's default viewer, a rendered preview the conversation client can display. The agent gets a small textual reference; the human gets the artifact.

> **Heuristic.** When designing an MCP tool (or any agentic tool the operator will be present for), ask two design questions, not one: *"what does the agent need from this response to take the next action?"* AND *"what does the operator need from this response to review the agent's work?"* The two needs often have different optimal shapes (agent wants compact textual references; operator wants visual artifacts). Design the response to serve both — typically a small textual reference for the agent + a side-effect that produces something the operator can see (a URL, a file the OS opens, a rendered preview). If the tool can't serve both, the operator's needs should usually win, because the operator catches what the agent can't.

The shape generalizes beyond canvas-mcp:

- **A `screenshot()` MCP tool for any visual product.** Returns a URL or file path the human can open, alongside (or instead of) the bytes the agent reads.
- **A `diff()` MCP tool for code review.** The agent gets a small unified-diff text; the human gets a link to a rich diff view (GitHub PR, HTML diff, IDE-shaped surface).
- **A `query()` MCP tool that runs database queries.** The agent gets a summary or the first N rows; the human gets a link to a queryable result view they can sort and filter.
- **A `generate_audio()` or `generate_video()` MCP tool.** The agent gets a duration / metadata response; the human gets a playable file URL.
- **A `compile_report()` MCP tool.** The agent gets the summary fields; the human gets a rendered PDF or web page.

In every case, the pattern is identical: **artifact-producing tools have two consumers (agent + operator) with structurally different consumption modes (textual reasoning vs. visual / experiential review), and the tool's response should serve both.** The fix is asymmetric — the agent's part of the response is small; the operator's part is the side-effect that produces a viewable artifact. The cost is the persistence/serving infrastructure, which is real but bounded.

The pairing worth naming: this is the *customer-surface* version of the *context-economics* lesson from the companion story `canvas-mcp-base64-png-context.md`. Same commit (`a3cf691`), same fix, two distinct lessons. That story is the primary take on *return pointers, not bytes* — the cost is conversation-context bloat compounding across turns. This story is the take on *agentic ≠ agent-only* — the cost is operator-experience friction that makes the agentic loop worse than it should be. Read together, the two explain why the viewer was the *right* fix even though *just returning a URL* would have been a sufficient fix for the context-economics problem alone. The viewer earned its cost by serving the operator; the URL alone wouldn't have.

Pairs with the companion story `forge-audience-constraint.md` on *naming the customer is the load-bearing design decision*. That story names *who the system's output is for* at the product level; this one names *who the tool's output is for* at the agentic-tool level. Pairs with the companion story `magmalabs-delegated-skill-decay.md` on *the operator is the customer, not the agent* — different surface (operator-as-user-of-agents vs. operator-as-co-customer-of-agentic-tools), same underlying principle.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *tools whose outputs are artifacts a human will review*. It doesn't apply to:

- **Tools whose outputs are purely intermediate.** A tool that returns *"the count of records matching this query"* doesn't need a human-viewable surface — the count is textual, the agent reasons about it, the operator may or may not care. The lesson is for *artifact-producing* tools, not *fact-returning* tools.
- **Tools with no operator present.** A fully autonomous agent loop with no human in the review path doesn't have an operator-as-co-customer. The tool can be agent-optimized. (Caveat: most "autonomous" agents still have a human reviewing logs or replays after the fact — if so, that's still the operator's surface and the lesson still applies, just delayed.)
- **Tools running in pipelines where the artifact is consumed by another machine.** A `transform()` tool whose output feeds the next tool's input doesn't need a human-viewable surface. The artifact's consumer is the next step, which is also an agent.
- **Stateless computation tools.** `compute_hash()`, `count()`, `normalize()` — these don't produce artifacts to review. They produce values to compose.
- **Tools where the response *is* the artifact.** A `generate_text()` tool whose entire output is text the agent will consume directly doesn't have a separate operator surface — the operator reads the same text the agent does, in the same place.

The signal: *if the tool produces something a human would naturally want to look at (an image, a video, a deck, a 3D model, a complex table, a long document), the human's review surface is part of the design.* If the tool produces a textual value or a fact, the human's surface is the same as the agent's and no special accommodation is needed.

## What This Story Is *Not* Evidence For

- **Not evidence that MCP tools should never return bytes.** Sometimes the bytes are the right shape — small images, brief audio clips, anything that's genuinely consumed once and not reviewed by a human. The lesson is about *artifact-producing tools with a human review loop*, not about a blanket bytes-vs-URLs rule.
- **Not evidence that agents are bad reviewers of their own work.** Agents are *fine* reviewers in the in-conversation sense — they can describe what they did, reason about it, decide next moves. The lesson is about *the operator's review surface being structurally different from the agent's review need*, not about the agent being incapable of self-review.
- **Not evidence that the operator is more important than the agent.** Both are customers; both have legitimate needs. The fix served both. The lesson is *both, not one*.
- **Not evidence that every agentic tool needs a viewer process.** The viewer is canvas-mcp's specific fix because canvases are visual artifacts. A different tool with different output shape needs a different surface (a file path, a rendered HTML report, a downloadable archive). The shape of the operator's surface depends on the tool's output.
- **Not evidence that the original `screenshot()` design was a mistake at design time.** It was a reasonable first cut — the obvious shape, easy to implement, sufficient for an agent-only test. The mistake was *not noticing the operator was a co-customer until friction surfaced*. The lesson is the *framing shift*, not the *initial design choice*.
