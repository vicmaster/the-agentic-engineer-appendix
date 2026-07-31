# coide — AskUserQuestion: When Research Delegation Changes the Build

## Date / Version Context

- **Date:** 2026-05-08
- **Product:** coide v0.26.x — a desktop GUI wrapper around the Claude Code CLI. coide spawns the `claude` binary as a subprocess and talks to it over stream-json.
- **Tool under investigation:** *AskUserQuestion* — a standard Claude Code tool the model calls to put a structured multiple-choice question in front of the user. In the terminal CLI, the user picks an option and the answer flows back to the model as a `tool_result`.
- **Glossary, used in this writeup:** *harness* = the runtime around the model that mediates tool calls (here, the Claude Code CLI). *Subagent* = a separate agent invocation spawned to do bounded work and report back.

## What Was Being Attempted

Two things, in order:

1. **Fix a small render bug.** AskUserQuestion's payload was showing up as `[object Object]` in coide's chat UI. Cosmetic, the kind of thing you knock out before lunch.
2. **Decide whether to make AskUserQuestion interactive in coide.** The terminal CLI lets the user pick and the choice flows back. coide was only *displaying* the question; nothing returned to the model. The natural next step seemed obvious: wire a real picker into the GUI, route the user's choice back over stream-json, and turn the read-only card into a proper interactive control.

The first task had a clear scope. The second was where the war story is.

## What Went Wrong

The render bug itself was trivial — JSON payload stringified the wrong way, fixed in a few lines.

The near-miss was the second task. Intuition said "we own the GUI, we own the subprocess, of course we can make this interactive." If that intuition had been allowed to drive the build, coide would have shipped a half-working interactive picker, fought a race condition it couldn't win, and either ripped it out or accumulated a maintenance tax for every CLI version bump.

What stopped that from happening was a 15-minute research detour.

## How It Was Discovered

While reading the codebase to figure out where to wire the picker, Victor delegated a parallel question to a research subagent (`claude-code-guide`):

> *What does the Claude Code CLI actually do with AskUserQuestion under stream-json?*

The subagent came back with two specific findings:

1. The CLI **auto-resolves** AskUserQuestion internally. It does not wait for an external `tool_result`.
2. The auto-resolve happens roughly **89 milliseconds after the `tool_use` is emitted**, and it resolves with `is_error: true`.

> 📅 **As of writing (2026-05-08):** the 89ms figure and the `is_error: true` resolution are observed behavior of the Claude Code CLI at this moment. Treat them as evidence for the story, not as a rule. Verify in your stack.

In other words: by the time coide could render the question, wait for the user to click, and post a `tool_result` back over stream-json, the CLI had already closed the loop and moved the conversation on. There was no window to inject a real answer. The constraint wasn't the GUI. It wasn't the subprocess model. It was the protocol the harness enforced before any of coide's code got a turn.

## What Fixed It

Nothing was "fixed" in the bug-fix sense — the larger feature simply didn't get built. The decisions were:

1. **Ship the read-only card.** The render bug got fixed. AskUserQuestion now displays cleanly as an informational card showing the question and options.
2. **Park the interactive picker.** Logged in `VISION.md` as blocked on an upstream change. Specifically: a CLI flag that lets the harness defer auto-resolution and accept an external `tool_result`.
3. **Don't build the workaround.** No racing the 89ms window. No PTY hijack. No shim around the protocol. All of those would have shipped something fragile that broke the next time the CLI changed its timing or its error semantics.

## The Durable Lesson

Two lessons, and they have to be stated separately because they live at different layers.

### Lesson 1 — Research delegation is most valuable when it can still change the scope

The default frame for subagent delegation is parallelism. Fan out, do more work in the same wall-clock time. That's real, but it's the smaller payoff.

The bigger payoff is **scope change**. A subagent that comes back with a fact you didn't have can move you from "build A" to "ship A-minus and park A." That move is worth more than any amount of parallel typing.

There's a catch. Scope change only happens if the research lands *before* you've committed to a design. Delegate after the architecture meeting and the finding becomes a footnote you reluctantly accommodate. Delegate during the codebase read, before you've drawn a single box, and the same finding rewrites the plan cleanly.

> **Heuristic.** If you're about to start building something whose feasibility depends on a system you didn't design, the cheapest move is to send a subagent to read the spec while you keep reading the code. The cost is one briefing prompt. The savings, when you're wrong about feasibility, are days.

### Lesson 2 — Protocol limits are not capability limits

The model was willing. The harness was willing. The GUI was willing. The protocol still said no.

That's a different shape of constraint than the ones engineers usually plan around. *Capability limits* ("the model can't reliably do X") show up in evals — you measure them, you work around them, you accept them. *Protocol limits* ("the message order or timing doesn't permit X") show up only when you read the contract, or when you ship and watch it fail in production.

You can't experiment your way to a protocol limit. You can only read your way there. Which means: when a feature depends on a tool boundary you didn't design, the first move is to read the spec, not to prototype.

This is the load-bearing point. Every tool an agent can call is a contract, and the contract is enforced *somewhere* — at the harness, at the CLI, at the provider. Not just at the model.

## Counter-Example — When This Lesson Doesn't Apply

Don't generalize this into "always research before building." For most features the protocol is well-trodden and the scope is obvious. Delegating research for a button or a list view is theater.

The signal that research delegation will pay off:

- The feature crosses a system boundary you didn't design (an upstream CLI, an external API, a third-party SDK).
- A small protocol detail could flip the build/don't-build decision.
- The cost of "build first, learn later" includes shipping something fragile or ripping it out.

If none of those are true, just build it. The research-first instinct is a tax when applied indiscriminately.

## What This Story Is *Not* Evidence For

- **Not evidence that subagents are always worth the briefing cost.** They aren't. The win here was specific: a system-boundary question with a binary build/don't-build implication.
- **Not evidence that you should never work around a protocol.** Sometimes you should. The case for parking it here was that the upstream owner (Anthropic) is the right place to fix it, not the wrapper.
- **Not evidence that read-only is the answer.** It was the answer *here*, because the interactive version had no honest implementation. In a different system with a different protocol, the call would go the other way.
