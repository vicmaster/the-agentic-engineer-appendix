# magmalabs-assistant / What You Delegate Is What Decays

## Date / Version Context

- **Project:** magmalabs-assistant — Victor's COO workload at MagmaLabs. Not a codebase. The "build" is a Claude Code configuration: nine custom skills (`/cockpit`, `/wrap-up`, `/ceo-sync`, `/call-prep`, `/deal-status`, `/standup-notes`, `/weekly-updates`, `/grain-search`, `/follow-up-alerts`), four registered sub-agents (`grain-notes`, `slack-searcher`, `calendar-assistant`, `lead-crm`), and an MCP stack into Google Workspace, Slack, and Grain. Plus a memory layer (Claude Code's MEMORY.md plus per-skill notes) that carries the operator's rules across sessions.
- **Timeframe:** rolling. The skills accreted over months as recurring workflows compressed into single commands. The three incidents below surfaced across a several-week window in early 2026; specific dates aren't tracked the way commits are, because the artifacts of the work are Slack messages, emails, and CEO summaries — not a git history.
- **Surface for this story:** the *operator*, not the agent. The agent's behavior was, in each case, working as designed. The story is about what happened to the operator's skills while the agent worked.
- **Glossary, used in this writeup:** *Skill* = a Claude Code slash-command-invokable workflow definition (`/cockpit` opens the operator's daily start-of-day routine; `/wrap-up` runs end-of-day reflection plus CEO summary; etc.). *Sub-agent* = a registered cheap-model assistant the main session fans research out to (e.g. `grain-searcher` does transcript lookups so the main context window doesn't fill with raw meeting text). *Memory* = the persistent MEMORY.md layer that survives session boundaries. *Cockpit* = the daily start-of-day skill — a routine name from the operator's CLAUDE.md, not a generic term.

## What Was Being Attempted

Use the skill stack to absorb the repetitive judgment parts of a COO role so the operator's attention could move to the parts that don't compress.

The motivating problem was real: Monday mornings rebuilt context the operator had already had on Friday. Reconstructing the state of the deal pipeline, the standup notes the team had filed, the threads with each client, the open follow-up alerts — all of that took an hour or two of cognitive setup before any actual work happened. The skill stack collapsed that into single commands. `/cockpit` produced the day's brief. `/wrap-up` closed the day with a reflection and a CEO summary draft. `/standup-notes` aggregated the day's standup transcript into action items. `/follow-up-alerts` surfaced clients who had gone quiet past a threshold.

The model was simple: skills compress workflows the operator already knew how to do. Each skill wrapped a routine the operator had been doing manually for months. The agent didn't introduce a new capability; it removed the orchestration tax.

Which is the setup for the atrophy. The skills did exactly what they were supposed to do — and the operator stopped practicing the underlying routines that the skills wrapped.

## What Went Wrong

Three skills decayed silently, each in a different shape.

**Slack formatting muscle.** The operator used to instinctively flatten formatting before pasting into Slack — Slack's `mrkdwn` dialect is not CommonMark, and certain combinations (blockquotes containing nested bullets, in particular) get eaten by Slack's renderer. After months of letting the `/follow-up-alerts` skill draft and send messages, the operator stopped checking. The skill sent a teammate a draft for a CLIENT follow-up using a blockquote with nested bullets. Slack ate the three critical questions inside the blockquote. The teammate received a draft that *looked empty* — the body of the message was the blockquote, and the blockquote's content was the questions, and the questions were the message. From the teammate's side, the message was a render error masquerading as a sent message.

**Email triage instinct.** The operator stopped reading emails end-to-end because the agent surfaces summaries. Then the operator forwarded a contact's feedback to the team based on a truncated email preview field, and the actual feedback in the body was different. The agent had read the full email correctly; the operator had read the agent's summary header without scrolling past it. Had to re-send the forward, apologize for the misattribution, and clarify what the contact had actually said. The skill of "scroll to the bottom of the email before you act" had quietly disappeared.

**Knowing whose work is whose.** The `/standup-notes` skill auto-aggregates everyone's items from the meeting transcript. The operator caught themselves reporting other directors' wins in their own CEO summary multiple times before adding the "only my items" filter to memory. The reflex of asking "is this mine to report?" had decayed because the skill served everything on a plate, and a plate with one item on it doesn't prompt the same question as a folder full of competing claims would.

Three different failure modes, one shape underneath: the operator stopped practicing the verification step that used to be muscle memory, because the skill had absorbed enough of the workflow that the verification step had no obvious place to live. Slack's flattening pre-check used to sit between "I have a draft" and "I paste it in" — the skill collapsed that gap. The email's scroll-to-bottom used to sit between "I read the subject" and "I act on it" — the agent's summary collapsed that gap. The attribution check used to sit between "I see the items" and "I credit them" — the auto-aggregation collapsed that gap.

The bug isn't in the skills. The bug is that compressed workflows compress *out* the practice that kept the operator's verification reflexes warm.

## How It Was Discovered

The Slack incident was caught because the teammate came back asking "what questions?" If they had received the message, assumed it was a real-but-confusing send, and answered around it, the broken render would have stayed invisible. The downstream human's confusion was the only signal.

The email-forward incident was caught because the team replied to the forward with a question that wouldn't have made sense given the contact's actual feedback. The operator went back to the original email, scrolled, and saw the mismatch. Again: downstream human signal, not anything visible in the agent's output.

The attribution incident was the slowest to surface. The CEO summary went out daily with subtly wrong items for several days before the operator caught the pattern themselves while editing. There's no evidence the CEO noticed, which is its own kind of evidence: if the operator hadn't caught it, the misattribution would have remained the historical record.

All three failure modes share a discovery pattern: the agent's output gave no signal that anything was wrong. The Slack message was sent. The email summary was accurate. The standup aggregation was complete. The check that would have caught each failure was *the operator's*, and the operator had stopped doing it.

Standard observability for agentic systems focuses on the agent's output — logging the claim, tracking the call, sampling the response. None of those would have caught any of these three incidents. The agent did its job in every case. The decayed muscle wasn't in the system; it was in the operator using the system. The observability question becomes: *who watches the operator?* The honest answer is: downstream humans, eventually, sometimes.

## What Fixed It

Three discipline moves, none of which involved using the skills less.

**Encode the rule and the *why* into memory.** "Don't include other directors' items" went into MEMORY.md, with the reason: each director reports their own. The skill reads memory at the top of every standup-notes run; the filter applies on every invocation. The "why" line is what makes the rule survive edge cases — a future revision of the skill, or a related skill that also aggregates from transcripts, can apply the same reasoning instead of needing the rule to be restated for each new surface.

**Move approval gates into the skill, not into intention.** "Always show me the CEO summary draft before sending" had been a thing the operator believed. After the Slack incident, the relevant gate got moved into the `/follow-up-alerts` skill's definition: drafts to a teammate that contain blockquotes or nested formatting must surface as a preview before send, not after. The gate now lives where the action happens, not in the operator's head where it was getting overrun by the skill's speed.

**Re-introduce practice deliberately for the skills the operator wants to keep sharp.** The CEO-summary editor reflex stayed intact because the operator edited every draft instead of approving without reading. The email-triage reflex didn't, because the operator stopped opening the full thread. The discipline move is to flag, per skill, whether the underlying workflow is one the operator wants to retain ability in — and if so, to schedule deliberate manual runs of the underlying workflow on some cadence. Not every skill needs this; some workflows the operator is happy to forget how to do. The decision needs to be conscious.

The deeper observation: none of these fixes is technical. The skill stack didn't change in architecture. What changed is the operator's understanding of which routines they were willing to lose ability in, and which they were paying to retain.

## The Durable Lesson

Skills don't replace skills — they replace *practiced* skills. The capability you delegate is the one that decays first. Pick what you delegate accordingly.

> **Heuristic.** Before delegating a routine to an agent or skill, ask two questions. First: *is this a routine I've practiced enough that I'd recognize a bad output if I saw one?* If no, delegate later, after the recall reps. Second: *if this routine decays in my hands, can the system catch it without me?* If no, you've delegated a check the system can't do — and the downstream signal will be a confused human, not a clean alert. Delegate anyway if the trade is worth it, but encode the why-of-the-trade somewhere durable, because future-you won't remember why this was an acceptable risk.

The mental-model flip the lesson rests on: knowing-by-recognition instead of knowing-by-recall. The operator's job stopped being "remember what the contact said" and became "be the person who notices when the summary of what they said is missing the part that matters." That's a load-bearing move for any operator scaling beyond what they can hold in their head — but it only works if the operator has done the recall version often enough to know what right looks like. Recognition is parasitic on prior recall. Delegate the recall before the taste is set, and you'll be a fast wrong instead of a slow right.

The harder pill in the lesson: delegating something means accepting that you will be wrong about it occasionally and won't notice. The teammate's "what questions?" is the price of the trade. The right question isn't *how do I prevent that?* It's *is the rate of those incidents lower than the rate of mistakes I'd make doing it all myself, exhausted, at 11pm?* For most operators using agents to scale judgment work, the answer is yes — but the trade has to be conscious, and the cost has to be named, or the first bad miss feels like betrayal instead of math.

The shape generalizes outside this project:

- **Lawyer with a contract-review agent.** Stops reading whole contracts. The agent's redlines are taste-based; the lawyer's eye for the unusual clause the agent didn't flag was the value-add. Atrophy is the eye, not the redlines.
- **PM with a roadmap-summary agent.** Stops re-reading stakeholder threads. The summary is faithful; the PM's sense of *what's not in the summary because no one said it out loud* was the value-add. Atrophy is the inference, not the summary.
- **Engineer with a code-review agent.** Stops reading whole diffs. The agent's findings are correct; the engineer's read for *what should be in this diff but isn't* was the value-add. Atrophy is the absence-noticing, not the line-by-line read.

In every case the pattern is the same. The skill compresses out the practice that kept the operator's verification reflex warm. The verification reflex was the thing worth keeping. The visible mechanics of the role were what got delegated; the invisible mechanics — the question the operator used to ask themselves at the boundary — were what got lost.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *practiced* skills that decay when delegated. It doesn't apply to:

- **Capabilities the operator never had in the first place.** The temporal cross-referencing the agent does — pulling threads from three months ago into the context of an email arriving today — isn't a skill the operator atrophied. It's a capability the operator didn't have. There's no decay because there was no muscle. Delegate freely; nothing to lose.
- **Skills where the agent's output is fully checkable downstream.** If the agent produces a number that a deterministic system will verify (the booking goes through or it doesn't; the migration succeeds or fails; the test passes or doesn't), the operator's recognition reflex doesn't need to stay warm. The system catches the failure. The atrophy lesson is specifically about *probabilistic outputs into ambiguous downstream surfaces* — drafts, summaries, judgments, attributions.
- **Routines the operator is happy to forget how to do.** Not every skill is worth retaining. The operator might genuinely be happy to never manually reconstruct a deal pipeline view again. That's fine — *consciously*. The lesson asks for a deliberate decision per skill, not a panic about losing every reflex.
- **Workflows where the agent and operator both contribute, with the operator's part still requiring practice.** The CEO summary editor reflex is the case-in-point: the agent drafts, the operator edits. The editing is the practice. As long as the operator keeps editing, the underlying judgment doesn't decay — it actually sharpens, because the agent's draft is a forcing function for noticing what the operator would have foregrounded differently. The lesson doesn't apply when the skill leaves the operator in the loop on the judgment-bearing step.

The signal: *am I delegating execution-and-judgment, or execution-only?* If only execution is delegated and judgment stays with the operator, atrophy is bounded. If both are delegated, the operator's recognition reflex is on borrowed time unless something else keeps it warm.

## What This Story Is *Not* Evidence For

- **Not evidence that delegation is bad.** Every incident in this writeup happened inside a stack of skills that the operator continues to use and would expand. The skills work. The lesson is about which routines are safe to compress, not about whether to compress any.
- **Not evidence that the operator should have caught these earlier.** The Slack render failure is invisible from the operator's side until the downstream human surfaces it. The email summary's accuracy depends on the agent reading correctly, which it did. The attribution issue is the only one that's arguably an operator-vigilance failure, and even that one was caught by the operator's own editing reflex on the CEO summary — slower than it should have been, but caught.
- **Not evidence that agents need more guardrails.** Two of the three fixes are *operator-side* discipline moves (encode rules with why; deliberate practice). Only the third (approval gate in the skill) is a system change. Treating this story as a brief for stricter agent constraints would miss the lesson — the agents are fine; the operator's relationship to the practiced routine is what needs the attention.
- **Not evidence that this only applies to non-engineering work.** The engineer-with-code-review-agent analogue in the Durable Lesson section is the same shape. Anywhere an agent absorbs a routine the operator used to do by hand, the underlying check the operator used to perform is at risk. Engineering is no exception; if anything, code-review atrophy is harder to catch because the downstream signal (a bug in production, a regression three weeks later) is more distant from the moment the practiced skill went unpracticed.
- **Not evidence that long sessions are the cause.** The five-week session that hit 51% tokens is a different problem — context-window economics, not skill atrophy. Long sessions rot in a different way. This writeup is about the skills the operator runs across many short sessions; the atrophy is in the operator, not the conversation.
