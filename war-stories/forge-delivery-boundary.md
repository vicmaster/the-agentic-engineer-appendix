# forge / The Delivery Boundary Pattern

## Date / Version Context

- **Dates:** 2026-04-22 to 2026-04-24, week of Forge's first leadership-facing Slack DMs going live.
- **Project:** Forge — internal Rails 8 / Postgres / Sidekiq app at MagmaLabs that lets non-technical leadership configure and run Virtual Employees (supervised, capability-scoped agents that read company data and deliver back via Slack and email).
- **Tool surface for this story:** Slack DM ingress (user → `Slack::DispatchMessageJob` → `TaskRouter` → capability handler) and Slack DM egress (capability handler → `SlackDelivery` → posted message).
- **Glossary, used in this writeup:** *Capability* = a bounded unit of agent work in Forge (e.g. `bsc_analyst`, `kpi_watcher`, `data_source_chat`), each backed by an `AgentCapability` row, gated by `AccessPolicy`, and routed through `TaskRouter`. *Delivery boundary* = the layer between the capability's output and the channel that renders it (Slack DM, email, future Teams/Discord). *Channel adapter* = the code at the delivery boundary that translates the capability's output into the channel's dialect.

## What Was Being Attempted

Ship Forge's first leadership-facing Slack experience. Capabilities generate LLM-summarized output (BSC briefs, KPI summaries, free-form Q&A). The output gets posted to Slack as DMs, with users replying in-thread for follow-up. The flow felt straightforward: capability produces text; the delivery layer posts it.

The "delivery layer" was, at first, a thin wrapper that called `chat.postMessage`. That thinness is what produced the three incidents.

## What Went Wrong (Three Incidents in Three Days)

### Incident 1 — CommonMark in a `mrkdwn` channel

**2026-04-23, commit `545ecc3`.** Claude produced ATX headings, `**bold**`, pipe tables, `[text](url)` links — standard CommonMark output the model defaults to. Slack DMs render `mrkdwn`, which is similar but not the same: `*bold*` not `**bold**`, no headings, no pipe tables, links are `<url|text>`.

Leadership saw `## Risks` with the hashes still showing. They saw `**revenue**` with the asterisks intact. They saw pipe-table dash separators on their own lines because Slack didn't recognize the format and rendered it literally. The brief was technically delivered. It looked broken.

Discovery: a leader read the brief and asked whether the bot was misconfigured.

### Incident 2 — Silence between question and answer

**2026-04-23, commit `9acead0`.** Slack DMs flowed through `Slack::DispatchMessageJob` → `TaskRouter` → async `RunTaskJob` → eventual reply. Between the user's question and the bot's answer (often 5–15 seconds for a Sonnet round-trip), the channel was silent. No "typing…" indicator, no acknowledgment, no signal that the message had been received.

Users assumed the bot was dead and re-asked. Or just left.

Discovery: leadership reported "the bot doesn't always answer" — which was true behaviorally even though every message had been processed.

### Incident 3 — Thread context lost across follow-up DMs

**2026-04-24, commit `0a8ba66`.** First message in a Slack thread worked. Second message in the same thread ("what did you mean by X?") arrived at the LLM with no history — each task was independent, and `Context::Loader` built its snapshot from the data source only. The model couldn't see what it had said two messages ago.

Discovery: leadership followed up on a brief, got an answer that ignored the conversation, and reasonably stopped trusting the bot.

## How They Were Discovered

Three different discovery channels in three days:

- Incident 1: surfaced by a human reading the rendered output and noticing it looked broken.
- Incident 2: surfaced by a human assuming the bot was broken because the channel was silent.
- Incident 3: surfaced by a human catching that the bot's answer ignored prior context.

All three discoveries are *humans noticing a degradation*. None of the bugs would have been caught by capability-level tests, because the capabilities themselves were producing correct output. The bugs lived strictly at the delivery boundary — between "what the capability produced" and "what the human saw."

## What Fixed Each Incident

Each fix landed at the delivery boundary, not in the capability.

1. **Incident 1 fix.** A `SlackMarkdown` helper module that runs only when rendering for Slack. It translates CommonMark to `mrkdwn`: `**` → `*`, headings → bold-line-break, `[text](url)` → `<url|text>`, pipe tables → fixed-width plain text. The capability still produces CommonMark. The translation happens at the edge.
2. **Incident 2 fix.** A reaction-ack cycle on the user's originating message: `👀` posted the moment the task is routed, replaced by `✅` when the response is delivered or `⚠️` if something failed. Cheaper than a placeholder message ("Voy a revisar..."), cleaner than nothing. The user gets a "I see you" within ~50ms of sending.
3. **Incident 3 fix.** Thread history derived from prior succeeded `data_source_chat` tasks joined on `thread_ts`. The data was already in `task.source_metadata` — just not being read back in. No schema change, no LLM-side change. The delivery boundary now stitches conversation history before passing the input down to the capability.

Three fixes. Three different mechanics. One thing in common: *none of them touched the capability handler.*

## The Durable Lesson

Keep the LLM-call code dialect-agnostic. Push every channel-specific concern to the delivery edge.

This is an architectural pattern, not a single technique. It says: *the capability decides what the answer is. The delivery boundary decides how the answer reaches the channel.* When a new channel (Email, Teams, web) ships, the capability isn't touched.

Three sub-rules fall out of it:

1. **Translate dialects at the edge.** The capability emits a single neutral format. The channel adapter translates to that channel's dialect (`mrkdwn`, HTML email, plaintext, whatever). The LLM prompt should never know it's "writing for Slack" — the moment it does, you've conflated content with formatting and lost cache hit rate.
2. **Render channel-shaped feedback at the edge.** Async-work UX (the reaction-ack cycle) is a channel concern, not a capability concern. The capability doesn't know whether it's being called from a synchronous CLI or an async Slack pipeline. The delivery boundary knows; it owns the "I see you" and the "I'm done."
3. **Reconstitute channel-shaped context at the edge.** Thread history is a Slack-specific construct. The capability shouldn't know what a thread is. The delivery boundary stitches `thread_ts`-shaped history into the capability's input, then unstitches the capability's output for posting.

> **Heuristic.** Every output channel is a translator. Every async channel is a feedback emitter. Every conversational channel is a context reconstituter. All three live at the delivery boundary, where the channel's shape is known and the capability's shape is unknown. If a capability handler ever has to know whether it's talking to Slack vs. email vs. CLI, the boundary is in the wrong place.

The shape generalizes. Replace "Slack" with "Discord" or "Teams" or "the web app." Replace "capability" with "model call" or "service handler" or "domain function." The pattern is the same: probabilistic content production above, channel-specific rendering below, with a hard boundary that prevents leakage in either direction.

## Counter-Example — When the Pattern Doesn't Apply

The pattern is for systems where (a) the capability is reused across channels and (b) channel-specific concerns are non-trivial. Don't generalize it to:

- **Single-channel systems where reuse isn't real.** A CLI-only tool with no plans to ship a UI doesn't need a delivery boundary; the CLI's rendering is the only rendering.
- **Channels where the capability's output is structurally identical to the channel's accepted format.** A capability that produces JSON for a JSON-API consumer doesn't need translation; the boundary is degenerate.
- **Cases where the channel's dialect is so simple it can be handled in the prompt.** Sometimes "respond in plain text only" in the system prompt is cheaper than a translation layer. The signal is whether the dialect concern is non-trivial enough to be worth a module.

The threshold for the pattern: *would shipping a second channel require touching the capability code?* If yes, the boundary isn't there yet. If no, the boundary exists and you can ship the second channel cheaply.

## What This Story Is *Not* Evidence For

- **Not evidence that all channel concerns belong at the boundary.** Some genuinely belong in the capability — for example, "this is a chat capability, not a brief capability" is a capability-level decision, not a delivery decision. The boundary is for *rendering, feedback, and context-reconstitution*, not for the capability's own behavior.
- **Not evidence that the boundary should be heavy.** The Forge delivery boundary is small: a Slack helper, a reaction-ack cycle, a thread-history join. ~200 lines total across the three fixes. The point is *where* the code lives, not *how much*.
- **Not evidence that this generalizes to single-call systems.** The pattern requires reuse across channels to pay back its cost. A one-shot script that prints to stdout doesn't need a delivery boundary.
