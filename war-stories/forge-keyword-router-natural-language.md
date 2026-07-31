# forge / The Keyword Router Met the Spanish Question

## Date / Version Context

- **Date:** Pre-launch arc. `IntentParser` shipped with the first version of Forge's Slack-DM ingress (2026-04, early). The fallback fix landed `ee427e8` on 2026-04-22 — about a week before production launch on 2026-04-29. The bug never reached production users; it reached the *internal Forge team* dogfooding the leadership-brief flow against their own questions, which is how it surfaced.
- **Project:** Forge — Ruby on Rails 8 / Postgres / Sidekiq / Heroku. Slack-DM ingress flow: user message → `Slack::DispatchMessageJob` → `IntentParser` → handler lookup → `TaskRouter` → capability handler → response. The router sits between dispatch and execution; getting it wrong silently bottlenecks every downstream capability.
- **Surface for this story:** `IntentParser` specifically — the small class that took a free-text Slack DM and tried to derive a capability slug. Pre-fix: keyword-only. Post-fix: keyword-with-fallback-to-`data_source_chat`.
- **Glossary, used in this writeup:** *Capability slug* = the internal identifier for a Forge capability, like `kpi_summary`, `bsc_analyst`, `data_source_chat`. *Keyword router* = a deterministic matcher that maps user input to a capability slug via keyword presence (often slash-prefixed: `/kpi`, `/bsc`). *Strict-mode parser* = a deterministic routing layer that returns nil on ambiguous input rather than guessing. *LLM-as-fallback* = the discipline of routing unmatched input to a chat-shape capability that can handle arbitrary natural language, rather than refusing.

## What Was Being Attempted

Route Slack DMs to the right capability handler.

The design instinct was sensible. Forge had four or five named capabilities at the time — `kpi_summary`, `bsc_analyst`, `meeting_prep`, `data_source_chat`, `daily_brief`. Each one had a slug. A user typing *"what are this week's KPIs?"* probably wanted `kpi_summary`. A user typing *"how did the BSC meeting go?"* probably wanted `bsc_analyst`. The router's job was to take free text and pick the slug.

`IntentParser` chose keyword matching as the mechanism. Each capability's manifest listed trigger keywords — `/kpi`, `kpi`, `bsc`, `/bsc`, `prep`, `meeting`, etc. The parser scanned incoming messages for any keyword match and returned the corresponding slug. Anything that didn't match got `nil`, which routed to a static `HELP_REPLY` describing the available commands.

The implementation was clean. Forty lines of Ruby. Test coverage for every keyword. Zero ambiguity at the parser layer — the parser's contract was *exact-match-or-help*. This kind of design feels safe: the parser never *guesses*; it only routes when it's certain.

The design assumed users would learn the keywords.

## What Went Wrong

Users didn't learn the keywords. They typed Spanish prose.

The first real-team test was the Forge internal dogfooding: a leadership user pinged the bot with *"¿Cómo van las ventas este mes?"* — *how are sales going this month?* The message contained none of the trigger keywords. `IntentParser` returned `nil`. The bot responded with `HELP_REPLY` listing the available `/kpi`, `/bsc`, etc. commands.

The user read the help reply once. Didn't reply. Stopped using the bot.

Three more leadership users had similar experiences within the next two days. None of them complained. None of them filed bug reports. They simply *stopped trying*. The bot's adoption curve flatlined before launch.

This is the specific shape worth pausing on. The system was *technically correct* — the help reply listed the right keywords, the parser was deterministic, the response time was fast, no error fired. From every system-side measurement, the bot was operating correctly. The failure was at the layer the system can't see: **the user's interpretation of `HELP_REPLY` is *the bot is broken*, not *I should rephrase using a keyword*.** Users don't read help replies as instructions; they read them as confirmation that the bot doesn't understand them.

The assumption that broke: **keywords don't leak through the conversation. Users don't see the implementation; users see the failure shape.** Telling a leadership user "type `/kpi` to ask about KPIs" makes the bot feel like a CLI. CLIs are tools developers learn; leadership users don't. The keyword-as-implementation surfaced as friction the users wouldn't pay.

Three specific Spanish prose messages that hit the router during dogfooding, all leading to instant `HELP_REPLY` fall-through:

- *"¿Cómo van las ventas este mes?"* — should have routed to `kpi_summary` or `data_source_chat`.
- *"Resúmeme la reunión del BSC del martes"* — should have routed to `bsc_analyst` or `data_source_chat`.
- *"¿Qué hay que preparar para la junta de mañana?"* — should have routed to `meeting_prep` or `data_source_chat`.

Three different questions, three different capabilities they would have served, zero keyword matches across all three. The parser was honest; the parser was useless.

## How It Was Discovered

By dogfooding. A leadership user pinged the bot in Spanish; the response was `HELP_REPLY`; the user didn't follow up; the operator (who could see the conversation) recognized the shape immediately.

The discovery channel matters. If the bot had been launched to leadership users without internal dogfooding, the failure would have surfaced as *bot adoption is lower than expected*, with no clear path back to *the router is refusing real questions*. The operator's privileged view — seeing both the user's input and the bot's static-help reply on the same screen — was the only thing that made the cause visible. Aggregate metrics (DM count, response time, error rate) would have shown nothing wrong.

This is structurally similar to the operator-side observability gap in the companion story `magmalabs-delegated-skill-decay.md`: the agent's output gives no signal anything is wrong, and the only fix-path goes through deliberate operator attention. Standard observability instinct would have missed both.

## What Fixed It

`ee427e8` made `data_source_chat` the fallback capability.

The mechanical change was small. When `IntentParser` returns `nil`, instead of dispatching `HELP_REPLY`, the router checks whether the VE has a `data_source_chat` binding enabled. If yes, the raw message text routes to that capability — which is itself an LLM-shape handler that accepts arbitrary natural-language questions and produces a response grounded in the user's connected data sources. If no `data_source_chat` binding, `HELP_REPLY` is the last-resort fallback (but only for VEs that haven't enabled the chat capability).

In practice every leadership VE had `data_source_chat` enabled, so the static `HELP_REPLY` effectively stopped appearing. The bot now handles `/kpi` (keyword path → fast deterministic dispatch) and `¿Cómo van las ventas este mes?` (fallback path → LLM-shape chat) and `random question that doesn't fit any capability` (still fallback path → LLM tries, possibly returns *I don't have data on that, here's what I can answer*). All three work.

The structural shift the fix encodes: **the keyword parser is an optimization for clear cases, not a gate against unclear ones.** Keywords are a fast path that saves an LLM call when the user happens to use one. The fallback is the default; the keyword path is the exception. The original design had the polarities reversed — strict keyword matching as the default, help reply as the fallback — and the result was a router that refused real users.

What didn't get attempted: trying to *expand the keyword list* to cover more phrasings. The lesson rejects that direction. Adding *"how are sales"* and *"sales report"* and *"ventas"* and *"sales numbers"* and a hundred other phrasings would still miss the hundred-and-first phrasing. The structural fix is to stop relying on keywords for the unclear case; the LLM-as-fallback eats the long tail entirely.

## The Durable Lesson

A keyword router is a strong assumption about how users will speak to your agent. **The assumption is almost always wrong, because keywords leak the implementation.** Real users don't see your handler list; they see a chat input. They type the way they think, not the way your code parses.

The deeper observation: **design the fallback into the routing layer from day one. LLM-as-router is what lets you ship a strict-mode parser without trapping users when — not if — the parser misses.**

The mental-model flip the lesson rests on: the routing layer has two distinct jobs, and conflating them is the bug.

- **Job 1 — *Fast dispatch for clear cases.*** A user who types `/kpi` is signaling intent unambiguously. A deterministic parser is the right tool: fast, predictable, no LLM token cost, no semantic ambiguity.
- **Job 2 — *Graceful handling of unclear cases.*** A user who types Spanish prose is signaling intent ambiguously. A deterministic parser is the *wrong* tool here; it has no way to recover. The right tool is an LLM-shape handler that can absorb the ambiguity and either respond directly or ask for clarification.

The bug is when Job 1's tool is used to do Job 2's work — when the parser's *miss* condition becomes a refusal to handle the input at all. The fix is to let Job 2 have its own handler, with the parser's nil-return being the signal to delegate.

> **Heuristic.** For any user-facing input layer (chatbots, command palettes, search bars, voice assistants), assume the deterministic-parser path will be hit by only a fraction of real inputs. Design the fallback path *before* the parser path. The fallback is what determines whether your system feels usable. The parser is an optimization on top.

The shape generalizes:

- **CLI tools with a `help` subcommand.** A CLI that fails closed on unknown subcommands is the same shape as `IntentParser`. The fallback isn't usually an LLM (CLIs don't have that affordance), but the structural lesson holds: *the user will type something you didn't anticipate; the failure mode for that case matters more than the success mode for the cases you did.*
- **API endpoints that 404 on unknown paths.** Same shape. The 404 page is the surface the user sees when your routing didn't match. Investing in the 404 page is investing in the failure mode.
- **Tool surfaces in MCP servers.** A tool with strict input validation that 4xx-rejects unrecognized inputs is the same anti-pattern. See `presentation-studio-mcp-feedback-loops.md` for the longer treatment — *agentic callers don't read errors well*, and rejection isn't a usable feedback channel for an agent any more than `HELP_REPLY` is for a leadership user.
- **Search interfaces.** A search bar that returns zero results when the keywords don't match a known facet is failing in the same way. Modern search engines use vector similarity *as the fallback for unmatched keywords*, not *instead of keyword matching* — the polarity is the same as IntentParser's fix.

In every case the rule is: **the fast deterministic path is the optimization; the fallback is the load-bearing surface.**

The pairing worth naming: this is the *user-input* version of `presentation-studio-mcp-feedback-loops.md`. Both stories are about *strict-reject failure modes when the caller is probabilistic*. Presentation-studio's caller is an agent; Forge's caller is a leadership user typing into Slack. Both produce inputs the strict parser doesn't recognize. Both fail loudly when the right fix is to *normalize, accept, and let the deeper layer absorb the ambiguity*. The lesson generalizes across the human/agent line — *whoever your caller is, if their input is probabilistic and your parser is strict, the parser's miss-condition is your usability ceiling.*

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *user-facing routing where the input is natural-language-shaped*. It doesn't apply to:

- **Programmatic API routing.** A request from another service with a defined contract should be routed strictly. Mis-matches there are bugs to surface, not natural-language nuance to absorb. The lesson applies when the caller is human or LLM, not when the caller is code.
- **High-security routing where ambiguity is dangerous.** Authorization decisions, payment routing, anything where guessing-wrong has irreversible cost. Stay deterministic. The fallback to an LLM is exactly the wrong move for these surfaces; the fallback should be *refuse and require explicit re-authorization*, not *try to interpret*.
- **Cost-constrained internal tooling.** If the LLM-as-fallback would multiply your inference bill by 100×, the math may not favor it. Some internal tools that fire thousands of times a day can't pay for a fallback LLM call on every miss. The fix in those cases is different — better keyword coverage, or a structured input surface (forms, dropdowns) that doesn't accept free text in the first place.
- **Workflows where the user already learned the keywords.** A developer command palette with 200 commands and a power-user audience can stay keyword-only because the audience treats keyword-learning as part of the tool's value. The lesson is about *users who never agreed to learn your keywords*.

The signal: *if my user types something my parser doesn't recognize, what experience do I want them to have?* If the answer is *they should re-type using a known keyword*, you're designing for a user who'll do that — usually a developer. If the answer is *the system should still try to help*, you need the fallback.

## What This Story Is *Not* Evidence For

- **Not evidence that keyword routers are bad.** They're the right tool for clear cases. `/kpi` should still fast-path through the keyword parser; the LLM call is the optimization-trade-off you don't take when you don't have to. The lesson is about *the polarity of fast-path-vs-fallback*, not about removing keywords.
- **Not evidence that LLMs should handle every routing decision.** They shouldn't. Routes that are *deterministically clear* (a `/kpi` command, an internal API call with a defined path) should stay deterministic. The lesson is about *the unclear case*, where the LLM is the right tool.
- **Not evidence that the dogfooding caught everything.** The bug surfaced in dogfooding *because the operator was watching*. The same dogfooding pattern in a team where nobody's reviewing the conversation transcripts would have produced the same bot-abandonment without the diagnosis. Dogfooding is necessary, not sufficient.
- **Not evidence that Spanish is the bug.** Spanish is the specific shape this story took; English natural-language prose without keywords would have produced the same failure. The bug is *the parser's mismatch with natural-language input*, not *the language the input is in*.
- **Not evidence that `HELP_REPLY` was a bad message.** The message was clear, accurate, listed the right keywords. The bug is in *when the message fires* — the route to the message was triggered by a wrong predicate (parser-miss = user-needs-help), when the correct predicate is parser-miss = LLM-handles-it. The message stays; the predicate moves.
