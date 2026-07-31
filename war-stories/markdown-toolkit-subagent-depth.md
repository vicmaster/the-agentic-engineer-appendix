# markdown-toolkit / Sub-agents Can't Spawn Sub-agents

## Date / Version Context

- **Date:** Playbook drafted 2026-05-04 ~14:00, the same afternoon the multi-agent pattern landed in markdown-toolkit. The hidden-assumption rewrite happened later that week after issue #4182 surfaced the limit.
- **Project:** markdown-toolkit — a WXT (Manifest V3) Chrome extension hosting a registry of single-purpose Markdown utilities; first tool is *Spreadsheet → Markdown*. One developer, one codebase, six role-specialized Claude Code sub-agents (`project-manager`, `team-leader-architect`, `product-designer`, `senior-engineer`, `qa-automation`, `code-reviewer`).
- **Surface for this story:** `.claude/ORCHESTRATION.md` — the playbook that names the team and the wave structure. Not product code; this is build-time orchestration.
- **Glossary, used in this writeup:** *Sub-agent* = a Claude Code agent invocation spawned by a session via the `Agent` tool. *Top-level session* = the originating Claude Code conversation; the only place that can call `Agent`. *Wave* = a phase of work in the playbook (e.g. Wave 1: PM + Architect in parallel; Wave 3: QA + Reviewer in parallel). *Delegation depth* = how many `Agent` calls you can chain — i.e. can a sub-agent call another sub-agent?

## What Was Being Attempted

Set up a multi-agent build playbook for a one-developer side project. The motivation is explicit and worth stating: *role separation forcing artifact production* — even solo, having `team-leader-architect` produce a separate file from `senior-engineer` forces a commit to the diagnosis before any code is written. That discipline was the goal, not headcount.

The first draft of `.claude/ORCHESTRATION.md` modeled the team as a hierarchy. There was a `supervisor-orchestrator` sub-agent at the top of the org chart whose job was to dispatch the team — Wave 1 ("send PM + Architect"), Wave 2 ("send Engineer with both artifacts"), Wave 3 ("send QA + Reviewer"). The top-level Claude Code session would invoke `supervisor-orchestrator`, the supervisor would invoke the team, the team would produce artifacts, and the supervisor would summarize back up.

The diagram looked clean. The org chart was familiar. The first ticket went to draft.

## What Went Wrong

The supervisor-orchestrator pattern was structurally impossible.

Claude Code's `Agent` tool is only available inside the top-level session. A sub-agent — anything spawned via `Agent` — does not get the `Agent` tool itself. Which means a sub-agent cannot spawn another sub-agent. There is no "supervisor calls the team" path. The only entity that can call the team is the top-level session.

This was not in the docs anyone reads first. The Claude Code agent docs describe sub-agent capabilities, configuration, and use cases. They do not lead with "delegation depth is 1." That fact lives in issue #4182 in the GitHub tracker, which is where the problem finally surfaced — not by reading the playbook, but by attempting to make the supervisor pattern work and watching it fail.

The first draft of the orchestration doc was, in effect, fiction. Plausible org chart, impossible delegation graph.

## How It Was Discovered

The discovery channel was reading issue #4182. The playbook had been written. Some of the wave logic had been exercised solo (top-level session manually playing both orchestrator and PM in sequence). The supervisor pattern would have surfaced the moment the playbook's "real" version was attempted — `Agent supervisor-orchestrator` would have returned a sub-agent that couldn't itself spawn the team — and that would have been a runtime failure with a useless error message.

Instead the failure was caught upstream: an existing GitHub issue in the Claude Code repo named the limit before a runtime exercise of the playbook hit it. The fix was a doc rewrite, not a code rewrite. The supervisor-orchestrator was deprecated; the orchestrator role moved explicitly to the top-level session.

This is a better discovery channel than the runtime would have offered. It's also the channel that requires you to know to *look there*. Issue trackers are not search-indexed in the docs.

## What Fixed It

The playbook was rewritten in two ways:

1. **Deprecate `supervisor-orchestrator`.** The role no longer exists in `.claude/ORCHESTRATION.md`. There is a note explaining why — sub-agents cannot spawn sub-agents — so future-Victor (or future-collaborators) don't reintroduce it.
2. **Move orchestration to the top-level session.** Wave 1, Wave 2, Wave 3 are all driven by the top-level session calling `Agent` directly. The session is the orchestrator. The role separation that was the original goal — having `team-leader-architect` produce a separate file from `senior-engineer` — is preserved, because that's a function of *who writes which file*, not *who dispatches whom*.

The fix is a doc fix because the bug was a doc bug. No code changed. The wave structure works fine when the top-level session is the dispatcher; it does not work at all when a sub-agent is supposed to be.

What didn't get attempted: workarounds. There is no shim that lets a sub-agent call `Agent`. Some users have proposed file-based message-passing (sub-agent writes a "please run X" file, top-level session polls and dispatches), but that's reinventing the orchestrator — only worse, because now you have a polling loop layered on top of the harness's actual control flow. The honest answer is *the orchestrator is the top-level session* and any architecture that needs more depth needs a different harness.

## The Durable Lesson

Architectural limits of the agent harness are not in the docs you'll read first.

The supervisor pattern would have read fine in any review. Senior dev looks at the playbook, sees a familiar org chart, says ship it. The limit isn't in the chart's shape. It's in the harness's enforcement, which is a different document from the one you read to learn how sub-agents work.

> **Heuristic.** Before you commit to an orchestration architecture, the question to ask is *what's the maximum delegation depth this harness allows?* Then verify it. The doc that names the answer is rarely the doc that introduces sub-agents — it's usually a release note, an issue thread, or a diagnostic you only run after the architecture's been drafted. Assume the playbook is wrong about delegation depth until you've stress-tested it.

The deeper shape: every agent harness is a constraint surface, and the constraints are unevenly documented. Some are loud (rate limits, context windows, tool counts). Some are quiet (delegation depth, message-order timing, auto-resolution windows). The quiet ones are the ones that invalidate architectures before they're built. They're also the ones you only find by either reading the spec end-to-end or shipping the architecture and watching it fail.

Three independent projects in this corpus discovered the same shape of quiet limit. coide hit AskUserQuestion's auto-resolution timing. canvas-mcp hit MCP server lifecycle bound to session lifetime. markdown-toolkit hit sub-agent delegation depth. None of these limits showed up in the first doc the developer read. All three showed up in the contract enforced *at the agent boundary*, not at the application layer where the developer was working.

That's the load-bearing observation: the contract enforced at the agent boundary is the real contract, regardless of what the docs or the application layer claim.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *architectures that depend on harness behavior you didn't author*. It doesn't apply to:

- **Architectures bound entirely to your application layer.** A multi-agent simulation you build inside your own process, where you wrote the dispatch loop, is fully under your control. Delegation depth is whatever you make it. The lesson here is about external harnesses (Claude Code, Cursor, Anthropic Agent SDK), not internal ones.
- **Patterns that don't rely on agent-spawning at all.** A team of human operators each running their own top-level Claude Code session and sharing artifacts via git is *not* a multi-agent architecture in the harness sense. The constraint doesn't apply because there's no `Agent` call chain at all — every "team member" is independently an orchestrator.
- **Cases where the limit's been verified.** Once you've checked the harness allows what your architecture needs, the lesson is paid. The point is to check before committing, not to assume permanent uncertainty.

The signal: *does my orchestration plan depend on a harness behavior I haven't verified?* If yes, verify before drafting the playbook. If no, ship.

## What This Story Is *Not* Evidence For

- **Not evidence that multi-agent orchestration is broken.** It's evidence that *one architecture* (supervisor sub-agent dispatching the team) is impossible in Claude Code. The wave-based pattern works; the orchestrator just has to be the top-level session.
- **Not evidence that the docs are bad.** It's evidence that harness limits live in many places at once (issue trackers, release notes, source code, diagnostic exercises) and the first doc you read is rarely complete. This is a property of all complex systems, not a Claude Code defect.
- **Not evidence that you should always read every issue before drafting.** The opposite — exhaustively reading issues for limits you may never hit is its own tax. The lesson is to verify the *load-bearing* assumptions of your architecture, not all assumptions.
- **Not evidence that role separation needs sub-agents at all.** The principle is *named artifact per role*, not *agent per role*. A solo top-level session that produces `parser-fix-2026-05-04.md` and `cleanup-v1.md` and `qa.test.ts` as separate files, in sequence, has captured the discipline. The agents are an enforcement mechanism, not the principle.
