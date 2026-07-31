# presentation-studio-mcp / Feedback Loops: When the Caller Doesn't Read Errors

## Date / Version Context

- **Date:** Artifact landed 2026-04-09 (commit `ee61fa1`); writeup captured 2026-05-09.
- **Project:** presentation-studio-mcp v0.1.0 — a local MCP server that turns a `DeckSpec` (JSON) into a `.pptx` file. Stack: TypeScript monorepo, Zod schemas, PptxGenJS renderer, Pillow sidecar.
- **Tool surface:** 13 MCP tools (`render_deck`, `audit_deck`, `normalize_deck_spec`, etc.), all stdio, all called by agents (Claude Code, Codex). 15 layouts, 12 audit rules, 7 templates.
- **Glossary, used in this writeup:** *agentic caller* = a probabilistic caller (LLM agent) invoking the tool, as distinct from a human caller invoking the same surface manually. *Strict-reject* = the conventional API discipline of returning a 4xx-shaped error for any input that fails validation. *Warn-and-degrade* = the alternative discipline this writeup names: normalize the input to something usable, produce output, and surface a structured warning the caller can act on.

## What Was Being Attempted

Build a tool that lets an agent generate professional slide decks. The agent decides what to say. The tool decides geometry, typography, branding. Hard split — no overlap.

The interesting question, once the split is drawn: *what is the tool's feedback discipline when the agent gets something wrong?* Specifically, when the agent passes:

- A layout name the tool doesn't know (`hero-stacked-cinematic`, `four-column-callout`, anything plausible-looking the agent invented).
- A required field left empty (a slide title, a body block, a closing CTA).
- A deck spec that overflows the canvas (35 bullets on a 5-bullet layout, 200 words on a 40-word title slide).

The default move — the one a human-facing API would pick — is strict-reject: return a 4xx-shaped error, refuse to render, force the caller to fix the input. The agent will read the error and try again.

Except agents don't read errors well.

## What Went Wrong (The Generalization)

This is the architectural-pattern part of the story. There isn't one incident; there's a class of incidents the human-API discipline produces under agentic callers, and a class of design decisions the project made to avoid each one.

The shape:

1. **The agent retries the same broken call.** A strict-reject error to a human is a stop signal. To an agent, it's a noisy retry condition. The agent re-runs the failed call with minor variations, burns tokens and tool budget, and often arrives at the same fail.
2. **The agent treats hallucinated defaults as valid.** When a tool helpfully fills in missing fields with sensible defaults, the agent never sees the gap. A blank `closingCtaMissing` slot the renderer auto-filled with "Thank you" reads as "the system handled it" — and the agent moves on without ever writing the right CTA.
3. **The agent renders and prays.** Without per-slide feedback before the artifact is shown to a human, the agent has no signal that the deck looks bad until the human opens it. By then the iteration loop is too long to close cheaply.

Each of these is a way "tool designed for humans" fails when the caller is probabilistic. None show up as a single incident — they show up as ambient drift, as decks that look plausible and aren't.

## How It Was Discovered

Two routes, both visible in the artifact:

1. **By design constraint.** The project's load-bearing rule is *the agent decides what to say; the tool decides how it looks.* That rule, taken seriously, forces the question: *what does the tool tell the agent when "what to say" produces something the tool can't render?* Strict-reject is the easy answer that doesn't survive the rule. Warn-and-degrade is the harder answer that does.
2. **By observed failure modes during build.** The defensive choices in the codebase — the layout fallback, the audit pipeline, the no-auto-fill rule — read as the codified residue of "we tried strict-reject and the agent didn't behave the way a human would." The mechanisms they explain are real, shipped code.

## What Fixed It (The Three Mechanisms)

The project encodes one principle in three concrete mechanisms. Each one answers a specific failure mode of the human-API discipline.

### Mechanism 1 — The Layout Fallback

`renderSlide.ts` contains a four-line rule, documented in `architecture.md`: *any unknown layout falls back to `two-column-text` with a warning.*

No crash. No 4xx. No rejection.

The agent gets a working `.pptx` it can iterate on, plus a structured warning that says "you asked for `hero-stacked-cinematic`, I rendered `two-column-text` because I don't know that layout." The agent can read the warning, pick a real layout, and re-call. The deck is never blocked on the agent's hallucinated enum value.

The same shape repeats throughout the codebase. Hidden slides → warning + skip. Missing images → warning, render placeholder. Missing CTA on a closing slide → audit warning, not render error. The project is consistently a *warning-emitting system rather than a gate-and-reject one.*

### Mechanism 2 — The Audit Pipeline as Eval Loop

The audit pipeline is 12 rules, each in its own file, each producing structured `AuditIssue` objects with severity. The rules know things like:

- Per-layout density thresholds (`hero-cover`: 40 words, `two-column-text`: 140 words, `logo-wall`: 30 words). "Too dense" is not one number.
- Title length, bullet count, image presence, contrast risk, layout validity, element bounds.
- Severity escalation: `> threshold * 1.4` → error, otherwise warning.

The audit runs *before* render is shown to a human. Every slide that's going to look bad surfaces in the audit report with a category, severity, and pointer to the offending field.

The agent reads the report. The agent fixes the slides. The next render passes audit cleanly.

**A note on the per-layout density decision specifically.** The obvious starting design for the density rule would be *one global word cap* — "no slide can exceed 100 words." It's the simplest implementation, the easiest threshold to communicate, the most familiar shape to humans coming from style-guide-shaped rules. It was also wrong in two directions simultaneously. A `hero-cover` slide at 60 words looks terrible (the layout is designed for a short, punchy headline; 60 words flood the visual breathing room the layout exists to provide). A `comparison-table` slide at 120 words looks fine (the layout is structured around dense side-by-side text). A global threshold either flags the comparison-table as too dense (false positive that the agent learns to ignore) or fails to flag the overflowing hero-cover (false negative that ships as a bad slide). Either way, the signal degrades. The fix is structural: the threshold lives *per layout*, encoded in `packages/core/src/audit/rules/textDensity.ts:9-25`, with `hero-cover: 40`, `two-column-text: 140`, `logo-wall: 30`, etc. Same rule, layout-aware threshold. The audit's false-positive and false-negative rates both dropped substantially after the split.

The principle the per-layout density decision names: *"too dense" is not one number*. Or generalized: **output-shape audits need shape-aware thresholds — the threshold is a function of the artifact's structure, not a global constant.** A rule that flags *"too many bullets"* needs to know whether the layout expects 3 or 12 bullets. A rule that flags *"text overflow"* needs to know the actual rendered space, not a synthetic word count. A rule that flags *"contrast risk"* needs to know the brand's palette, not a global luminance threshold. Threshold-as-constant is the easy-but-wrong shape; threshold-as-function-of-shape is the slightly-harder-but-right shape. The surprise worth naming: the per-layout split *made the audit signal stop flagging fine slides and stop missing bad ones* — both edges of the precision/recall trade-off improved at once, because the original signal was *miscalibrated to the shape* rather than too loose or too tight.

This generalizes well beyond DeckSpec rendering. Anywhere an audit rule fires against artifacts that come in multiple shapes (multiple layouts, multiple component types, multiple document templates), the threshold should be a function of the shape. The implementation cost is a small lookup table; the signal-quality benefit is the difference between an audit the agent trusts and an audit the agent learns to ignore.

That's an eval loop. Not in the conventional ML sense — there's no model under evaluation here — but in the architectural sense: *a per-call signal that lets the caller know whether its output meets the standard, before a human sees it.*

The thesis worth naming: *eval loops are the product*. The audit pipeline is what that thesis looks like for an artifact-generation tool: the audit step is the eval loop. Skipping the audit and trusting the render is the same mistake as skipping evals and trusting the prompt.

### Mechanism 3 — Don't Auto-Fill Missing Fields

`architecture.md` decision #4: *if a required field is empty, render leaves the space blank and emits a warning.*

The discipline reads as obvious in retrospect and is contrary to most rendering conventions. The PptxGenJS-shaped move is to fill the slot with something — placeholder text, a default title, an empty bullet that looks structurally complete. That's helpful for a human caller who can ignore the placeholder.

For an agent caller it's a bug. The agent never sees the gap. The deck ships with a "Thank you" closing slide the agent didn't intend, a body block with placeholder lorem-ipsum, a title that says "Untitled Slide" because the field was empty.

Render the gap *visibly*. Emit a warning. The agent sees the hole on its next pass, fills it correctly. The tool refuses to invent meaning the agent didn't author — that's the worker staying in its lane.

## The Durable Lesson

Two lessons, stacked.

### Lesson 1 — Agentic callers need feedback that doesn't break their loop

The human-API discipline assumes the caller can read a 4xx error and act on it. Agentic callers can technically read errors but practically don't act on them well. They retry, they accept defaults, they render and pray.

The discipline that survives the move from human to agent caller is *normalize, warn, and produce something usable, then surface a structured warning the caller can act on.* The agent gets a working artifact and a per-call signal. The signal is what closes the iteration loop.

> **Heuristic.** If your tool is being called by a probabilistic caller, your error path is no longer the rejection path — it's the structured-warning path. Reserve hard rejection for inputs that would corrupt state or violate a security boundary. For everything else: degrade to a usable output and surface what you couldn't honor.

### Lesson 2 — The audit step is the eval loop

For artifact-generation tools, the eval loop isn't a separate offline harness. It's the per-call audit that runs before the artifact is shown to a human. Per-call evaluation, structured findings, severity-graded — the same shape as a model eval, applied to the tool's output rather than the model's prompt.

Skipping the audit and trusting the render is the same mistake as skipping evals and trusting the prompt. Both produce confident wrongness.

This is the load-bearing claim phrased operationally: *eval loops are the product*, applied to a deterministic-tool-with-probabilistic-caller. The audit is what makes the iteration loop closeable without a human in the path.

## Counter-Example — When Strict-Reject Is Right

Don't generalize warn-and-degrade into "always be permissive." Strict-reject is the right discipline when:

- The input would corrupt persistent state (a malformed write to a shared store, a deck spec that would overwrite a registered brand).
- The input violates a security boundary (an agent trying to render a deck for an org it doesn't own).
- The semantic gap is large enough that no degraded version is honest (a request for a layout that has no structural cousin, where falling back to `two-column-text` would mislead more than help).

The signal: *would a degraded output mislead the human reading it?* If yes, reject. If no, degrade and warn.

The discipline isn't "be permissive." It's "match the rejection shape to what the caller can act on, and to whether degraded output is honest."

## What This Story Is *Not* Evidence For

- **Not evidence that all errors should be warnings.** Strict-reject still belongs at security and corruption boundaries. The discipline is about matching rejection shape to caller behavior, not removing rejection.
- **Not evidence that audit pipelines replace evals.** They are evals for one thing: artifact shape. Model behavior, prompt regressions, latency, and safety still need their own loops.
- **Not evidence that auto-filling is always wrong.** It's wrong when the caller can't see the fill. For human-facing tools where the user reviews the output before shipping, sensible defaults are fine. The caller's review discipline is the load-bearing variable.
