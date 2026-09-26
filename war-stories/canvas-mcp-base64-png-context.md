# canvas-mcp / The Bytes That Stayed in the Conversation

## Date / Version Context

- **Date:** Slow-burn from canvas-mcp's first weeks (Phase 1 in 2026-03) through Phase 3 (2026-03-21). The fix landed in commit `a3cf691` (2026-03-25): a built-in browser viewer, a new `viewer_url` tool, and viewer links added to the `canvas_create` and `batch_design` responses. Disk persistence (`~/.canvas-mcp/canvases/`) and the standalone viewer process came later, in `3e4201f` (2026-04-11).
- **Project:** canvas-mcp — open-source MCP server for AI-driven design mockups. A single-package Node/TypeScript npm project (Zod for tool parameters) that renders a scene graph to HTML/CSS and screenshots it in headless Chromium via Puppeteer. 13 MCP tools when Phase 3 shipped, 16 by v0.1 (2026-04-11); `screenshot()` is one of three that return rendered images (with `screenshot_responsive` and `canvas_diff`).
- **Surface for this story:** the response shapes of the tools the agent calls while iterating. Before the fix: the only way to see a design was `screenshot()`, which returns a base64-encoded PNG as an image content block. After the fix: `canvas_create` and `batch_design` responses carry a short viewer link (`View live: http://localhost:3001/canvas/<id>`), and `viewer_url` returns a small JSON object of links. `screenshot()` still returns the PNG, for turns where the agent itself needs to look.
- **Glossary, used in this writeup:** *Tool-output that compounds* = an MCP tool whose response sits in the agent's conversation history for the rest of the session, getting re-billed at input rates on every subsequent turn. *Compounding cost* = the per-turn cost of conversation history that grows with each accumulated tool response, especially when the responses are large (images, full documents, dense JSON). *Context degradation* = the slow erosion of model attention and budget as compounding tool outputs accumulate.

## What Was Being Attempted

Show the agent what it just rendered.

The `screenshot()` tool's job is straightforward: take a `Canvas` definition (the agent's most recent design output), render it to PNG, return the image so the agent can iterate. The agent looks at what it produced, decides what to change, calls back into the design tool. Standard iterate-with-visual-feedback shape — the kind of loop that exists in every AI-assisted design workflow.

The first version of the tool returned the PNG directly. A base64-encoded image, embedded as a content block in the tool's response. MCP supports image content blocks natively; the agent receives the PNG, the model sees it as a vision input, the iteration loop closes. The implementation was about thirty lines. The contract was clean. The first three or four turns worked beautifully.

By turn ten, long sessions started slowing down for reasons the operator couldn't immediately name.

## What Went Wrong

The PNGs from earlier turns were still in the conversation.

Each screenshot is a full capture of the canvas (1440×900 by default, at 2× device scale), and it rides along as image input on every later turn. (The self-interview didn't record a per-image token count, and the exact cost depends on the client and the image size, so none is quoted here.) After three iterations on a mockup, three of those PNGs sat in conversation history. After ten iterations, ten PNGs. The agent had looked at each one once on the turn it was rendered, decided to iterate, and moved on. The PNGs themselves were *not useful* to subsequent turns — the agent didn't re-read them, didn't compare them, didn't reference them. They were inert.

But the model has no way to *forget* a previous tool response. Every subsequent turn re-submits the entire conversation history. Every subsequent turn paid input-token cost for all the previously-rendered PNGs the agent didn't need to see again. By turn ten, a single Sonnet round-trip was paying for ten previously-rendered canvases on its way to producing the eleventh.

The cost shape compounded:

- **Money.** Input tokens are billed on every turn. Ten iterations means ten images re-sent as input on every subsequent turn, billed at the input rate. Multiply by the number of turns left in the session. The math gets ugly fast.
- **Latency.** Larger inputs take longer to process. Long-running sessions noticeably slowed down across turns. The slowness tracked the prompt size, not the model.
- **Attention.** The most insidious cost. The model has finite attention per turn. Stuff the context with previously-rendered designs and the model spends attention on those tokens *at the expense of* the current task's tokens. Responses started drifting toward old designs — referencing colors from two turns ago, picking up patterns from a previous iteration the agent had explicitly moved past. Not because the model was confused; because the older designs were closer to the front of the input and the attention budget had to allocate somewhere.

The discovery channel: not a single incident. A slow accumulation of *long sessions are getting worse over time*. The operator noticed it during Phase 3, when working sessions on the renderer were lasting hours and the model's outputs were degrading in ways the operator could feel before they could name. The forensic moment was looking at the conversation buffer, scrolling up, and seeing a stack of previously-rendered PNGs sitting in the history that the agent had no business re-paying for.

The bug isn't *base64 is wasteful encoding*. The bug is *the tool's output shape was designed for a single-call interaction, and the actual workload is a multi-turn iteration loop*. A response shape that works for the typical call doesn't work for the typical *session*.

## How It Was Discovered

The operator noticed the cost shape, scrolled the conversation buffer, and saw what was in it.

This is the discovery channel worth pausing on. No alert fired. No dashboard surfaced the problem. The token-budget telemetry showed the session was approaching context limits, but the telemetry didn't *attribute* the budget consumption to a specific cause. The attribution required a human reading the conversation history and recognizing *those are old PNGs the agent doesn't need.*

A team running on the same shape without the operator's reflex would have noticed the cost spike at month-end on the invoice, not in the moment. By then the workflow has already been suboptimal for weeks.

This is structurally identical to a pattern worth naming: *the agent's output gave no signal anything was wrong* — the agent rendered, the tool returned, the model produced an iteration, the conversation continued. Every per-call signal looked fine. The cross-call pattern (token consumption growing turn-over-turn) was invisible from the per-call view.

## What Fixed It

The viewer.

Commit `a3cf691` (2026-03-25) made three changes that converged on a single architectural shift:

1. **A built-in viewer (`viewer.ts`)** — a small local HTTP server (Node's `http` module, no framework) started inside the MCP server process. It serves each canvas at its own URL, plus a gallery page, and polls every two seconds so the page follows the agent's edits. It moved into its own process, with disk persistence under `~/.canvas-mcp/canvases/`, in a later commit (`3e4201f`, 2026-04-11) to fix a related lifecycle bug; see the companion story `canvas-mcp-viewer-lifecycle.md`.
2. **A `viewer_url` tool** — returns the viewer's address and a link for every canvas as a small JSON object.
3. **Links in the working tools' responses** — `canvas_create` now returns a `viewerUrl` field, and `batch_design` appends `View live: http://localhost:3001/canvas/<id>`. A few tens of tokens, not an image. The `canvas_create` description tells the agent to always share that link with the user.

`screenshot()` itself was not changed. It still returns a PNG, in canvas-mcp and in its successor framesmith. What changed is that *seeing* the design no longer required one.

The human watching the agent work opens the URL in their browser and sees the canvas live. The agent gets a tiny URL string back from the tools it was calling anyway, and only needs `screenshot()` on the turns where it has to look for itself. Viewing stops costing conversation budget.

A second-order consequence worth naming, though nobody measured it at the time: the agent's iteration quality should *improve*. With less of the attention budget spent on stale PNGs, the model's output can stay focused on the current task. That's the self-interview's account, not a benchmark. If it holds, the fix isn't only a cost fix; it's a *correctness* fix for the workflow shape this tool supports.

What didn't get attempted: trying to make the agent *forget* older PNGs in some structured way (truncating the history, summarizing the previous turns, swapping older tool responses with placeholders). All of those mechanisms add complexity at the wrong layer — they're cleanups after the fact. The structural fix is at the tool's *response shape*, not in the conversation management.

## The Durable Lesson

**Return pointers, not bytes.** Anything a human or an agent will look at once and forget should not live in the conversation. It should live somewhere the conversation can refer to. URLs, file paths, identifiers, slot numbers — whatever the indirection is, the rule is that artifacts the agent doesn't need to *re-read* shouldn't take up space in the budget that gets re-billed every turn.

The mental-model flip the lesson rests on: tool output is *consumed twice* — once on the turn it's emitted (where the agent reasons about it) and on every subsequent turn (where the agent pays for it). Designing the response shape only for the first consumption ignores the second.

> **Heuristic.** For every MCP tool response, ask: *is this content the agent will need to re-read on a future turn?* If yes (a parsed configuration, a small extracted fact, a reference number the agent will compose with later), include it directly. If no (a rendered artifact, a screenshot, a full document the agent is going to digest once), return a pointer. The cost of indirection is one additional tool call on the rare turn the agent does need to re-read; the savings compound across the typical case.

The shape generalizes far beyond images:

- **Generated documents.** A tool that returns full document bodies in its response — search results, scraped pages, generated reports — pays the same compounding cost. The fix is the same: return identifiers, let the agent fetch the body on demand.
- **Large structured data.** API tool responses that return entire database query results when only a count or a summary was the question. The agent doesn't re-read the rows; the rows just sit in the history.
- **Verbose tool outputs.** Commands whose `stdout` is hundreds of lines of which the agent only needs the last few. Filter at the boundary; don't ship the whole dump.
- **Streaming content.** Audio transcriptions, video frame analyses, long-form synthesis. The artifact's *summary* belongs in the conversation; the *artifact itself* belongs at a URL.

In every case the discipline is the same: **tool responses should be shaped for the typical session, not the typical call.** A response that's reasonable for one turn becomes expensive across ten.

The pairing worth naming: this is the companion to the story in `canvas-mcp-viewer-lifecycle.md`. The lifecycle story is *the URL the tool returns has to outlive the call*. This story is *the tool should return a URL, not bytes, in the first place*. Both stories are about the same architectural shift — splitting the rendered artifact into a separate, persistent surface — but they motivate it from opposite directions. Lifecycle is about *what the URL guarantees*; context economics is about *why the URL exists instead of the bytes*. Together they argue for the canvas-mcp viewer as a load-bearing piece of architecture, not a presentation convenience.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *content the agent re-pays for across turns*. It doesn't apply to:

- **Tool outputs the agent genuinely needs to re-read.** A small parsed structure the agent will compose with on multiple subsequent turns belongs in the conversation. The cost is bounded; the savings of indirection don't pay back. The rule is *what the agent looks at once and forgets*, not *all tool output*.
- **Single-call interactions.** A REPL-shape interaction where the operator runs the tool once, reads the output, and ends the session. There's no compounding cost because there's no subsequent turn. Reach for indirection when the workflow shape is multi-turn.
- **Throwaway exploratory work.** Same logic as above. Pay the cost when the session is short enough that the math doesn't matter.
- **Outputs the agent needs the model to actually attend to in the future.** If the design depends on the model recalling the prior PNG on turn ten — say, for consistency checks across iterations — then the cost of re-billing the PNG is paying for the workflow you want. The lesson is about *unwanted* re-billing.

The signal: *am I paying for this content on every turn after the agent has looked at it once?* If yes, return a pointer. If no, the lesson doesn't fire.

## What This Story Is *Not* Evidence For

- **Not evidence that images don't belong in conversations.** They do — when the agent actually needs to re-read them across turns. The lesson is about *unwanted* re-billing of content the agent already digested, not about removing images from agent contexts categorically.
- **Not evidence that MCP's image content blocks are misdesigned.** They serve a real use case (vision-enabled iteration on a single turn). The bug was in *the response shape decision for this specific tool*, not in the underlying MCP affordance.
- **Not evidence that base64 is wasteful encoding.** Base64 *is* about 33% larger than the raw bytes, but that's not the bug. Even raw-bytes-as-MCP-image-content would have the same compounding-cost problem, just slightly cheaper per copy. The bug is *the bytes being in the response at all*, not the encoding.
- **Not evidence that pointers solve every context-budget problem.** They solve the specific shape where the agent doesn't need to re-read the content. Other context-budget problems (long-running conversations, deep tool-call trees, stuffed system prompts) have other shapes and other fixes. Reach for indirection where the compounding-tool-output shape fits; reach for other patterns elsewhere.
- **Not evidence that the viewer is the only fix.** Persisting to disk + returning a URL is one implementation. Returning an opaque ID the agent can pass back into a `fetch_canvas` tool would work the same way. Returning a slot number in a session-bound cache would too. The point is *the response carries a reference, not the bytes*; the specific reference mechanism is implementation choice.
