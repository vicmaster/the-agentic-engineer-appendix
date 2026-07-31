# canvas-mcp / 629 Lines of Implication

## Date / Version Context

- **Date:** Indeterminate — `evaluate.ts` has been on disk for months without a commit. The file was first noticed during the 2026-05-09 self-interview, but its mtime suggests it's been in this half-built state since at least Phase 2 of canvas-mcp's build (2026-03).
- **Project:** canvas-mcp — open-source MCP server for AI-driven design mockups. TypeScript monorepo. As of writing the public repo has v0.1 shipped with 13 MCP tools, 15 layouts, 12 audit rules (in `auditDeck.ts`, *not* in `evaluate.ts`), 7 templates.
- **Surface for this story:** `evaluate.ts` itself — its existence on disk, its line count, its sibling `test-evaluate.ts`, and the gap between *the file the working tree contains* and *the functionality the project actually ships*. The shipped audit pipeline (`auditDeck.ts` + the twelve rules in `audit/rules/`) is a different code path entirely. The half-built `evaluate.ts` was a parallel attempt at a richer scorer, never finished, never committed, never gated, never deleted.
- **Glossary, used in this writeup:** *Half-built eval surface* = a partly-implemented evaluation or scoring path that exists in the codebase but doesn't function as advertised. *Honest absence* = the state where a feature is plainly missing and any reader can see it's missing. *Confident wrongness* = the state where a feature appears to exist (file present, structure plausible, types resolve) but doesn't actually deliver what its presence implies. *Working-tree implication* = the inferences a future reader (human or agent) draws from files present in the checkout, regardless of whether they're in version control.

## What Was Being Attempted

A richer evaluator for canvas mockups, beyond the rule-based audit.

The shipped audit pipeline in `auditDeck.ts` checks structural facts — layout in the known enum, density per layout, missing required fields, contrast risk, element bounds. Twelve rules, each focused, each fast, each producing a structured `AuditIssue`. The audit pipeline is the *eval* in the truest sense — *the audit is the eval* — and it works.

`evaluate.ts` was an attempt at something more ambitious: a scorer that grades mockups across layout, typography, and contrast dimensions, emits issues with severities, and produces a quality signal richer than the audit's pass/fail-per-rule shape. The intuition was that the audit could tell you *what's wrong* but not *how good* — and a richer evaluator could close that gap, producing per-design quality scores the agent could iterate against.

The work started. It progressed far enough to draft 629 lines of TypeScript and a test file. It stopped — the way features stop when someone gets pulled to something more urgent. There's no incident behind the stop. There's a *vacancy*: 629 lines of typed code that almost-but-doesn't compile, alongside a test file that exercises functions that almost-but-don't exist.

The file has never been committed. The project ships v0.1 with the audit pipeline and not the evaluator. The README mentions audit rules; it does not mention evaluation. By every shipped signal the project has no scorer.

By every signal a reader of the working tree can see, the project *has* a scorer.

## What Went Wrong

Nothing operationally, and that's the story.

The file does not run. The shipped product does not import it. The CI does not test it. Production users of canvas-mcp (the agents calling its MCP tools) never touch it. By every measure that involves *executing the code*, the file is inert.

By every measure that involves *reading the code*, the file is load-bearing.

Three readers stand to be misled by the file's presence:

**The author's future self.** Coming back to canvas-mcp after a few months, the operator opens the project, sees `evaluate.ts` in the file tree, and reasons: *I built a scorer; it must work or be close to working.* That mental model — formed by glancing at the working tree — is wrong. The work to recover from the wrong model is at least an hour of reading the file end-to-end to discover the gap between what's there and what runs. Multiply this by every project where the operator left half-built features in the tree.

**A future contributor.** An open-source contributor cloning the repo sees `evaluate.ts`, assumes evaluation is a project feature, and submits a PR that integrates the (broken) evaluator into the shipped pipeline. The PR's author has spent real time on the integration before noticing the underlying file is unfinished. The PR's reviewer either accepts a broken integration or rejects it after the contributor has put in work that depends on the file being complete. Either outcome wastes someone's effort.

**An agent reading the codebase.** This is the angle worth pausing on. Agents working in agentic-development workflows read codebases. A Claude Code session pointed at canvas-mcp, asked to *explain the project's architecture*, sees `evaluate.ts` and reasons as if it's load-bearing. The agent's output — to the operator, to a teammate, to a downstream document — encodes the false claim that canvas-mcp has self-evaluation. The agent isn't wrong to draw the inference; the working tree implies it. The agent is wrong about the world, and the cause is the file's presence.

This last reader is the one that matters most for an agentic-development book. **Half-built features in working trees are a class of misinformation that agents propagate** without their authors ever noticing. The same file would mislead a human in the same way, but humans are slower to integrate the working-tree state into a confident architectural claim. Agents do it in one turn.

## How It Was Discovered

Writing the self-interview.

This is the discovery channel worth pausing on a second time. The file has been on disk for months. The operator has worked in the canvas-mcp directory many times since. No process flagged the file. No `git status` was suspicious — the file is tracked-by-absence (it's gitignored or simply uncommitted; either way `git status` says *clean working tree* or *untracked* in a way easy to skim past). No CI failed because the file isn't in any CI path. The lint passes because the file's syntax is valid TypeScript even if it doesn't link.

The discovery came from a deliberate full-repo retrospective: the self-interview asked *where did the project bite you?* and the operator scanned the codebase end-to-end for the first time in months. The 629-line file in the working tree stood out the moment someone looked.

This is the latent-discovery shape from the companion story `forge-redaction-never-fired.md` at a different layer. That story is a control that hasn't been exercised; this story is a file that hasn't been examined. Both are *invisible-to-tools* states caught only by deliberate operator attention. Both argue for periodic full-repo retrospectives — not because every retrospective will find something, but because the things they find are uniquely hard to find any other way.

## What Fixed It

Nothing yet, and that's worth being honest about. The discipline moves are queued, not landed:

1. **Decide the file's fate.** One of three honest outcomes: commit it as experimental behind a flag (`EXPERIMENTAL_EVALUATE_TS=true` gating the import, plus a README note that says *"This is incomplete, don't trust the output"*), finish the work and commit a working version, or delete the file. Sitting half-built in the working tree is not on the list of honest outcomes.

2. **A working-tree audit cadence.** Once a quarter, scan the project for files-not-in-git over a size threshold. The 629-line file would have surfaced months ago if anyone had been looking. The audit is cheap: `git status --porcelain` + a size filter. The expensive part is acting on what it surfaces, not finding it.

3. **A repo-readme expectations check.** Whatever the README says the project does, the working tree should not imply *more*. If the README doesn't mention scoring, files named `evaluate.ts` and `test-evaluate.ts` are misleading-by-presence. Either align the README with the working tree, or align the working tree with the README. The mismatch is the bug.

What didn't get attempted: trying to *finish* the evaluator quickly to validate it. The lesson explicitly rejects that direction. A half-finished feature being rushed to half-plus-half-finished is still half-built. The structural moves above (decide, audit, align) are operator discipline; finishing the work — if it's worth finishing — happens after the discipline is in place, not as a way to avoid the discipline.

## The Durable Lesson

**An agentic feature half-built ships as confident wrongness; an agentic feature unbuilt ships as honest absence.** The half-finished version is more dangerous than the unstarted one — its presence in the working tree implies the project has functionality it doesn't actually deliver. Readers (human or agent) infer from presence; the inference is wrong; the wrongness propagates downstream.

The mental-model flip the lesson rests on: **working-tree state is information, not implementation.** Files have evidentiary weight regardless of whether they run. A 629-line file named `evaluate.ts` *is the assertion that the project evaluates things* — independent of whether the assertion is true. Removing the assertion (delete the file, commit it behind a gate, name it `experiment-evaluate-WIP-do-not-use.ts`) is what reconciles the working tree with the truth.

> **Heuristic.** Every quarter, scan your projects for files-not-in-git over a size threshold and ask one question per file: *if a future reader saw this file, would they correctly infer what the project does?* If the answer is "no, they'd assume more than is true," the file is a misleader. Decide its fate explicitly — commit it gated, finish it, or delete it. Sitting in the working tree, half-finished, is not on the list.

The shape generalizes beyond evaluators:

- **Half-built migration scripts.** A `migrate-2025-q2.rb` in the working tree implies the migration ran; if it didn't, the implication misleads.
- **Half-written API clients.** A `clients/external-api/full-client.ts` half-built in the working tree implies external-API integration; if the shipped code uses a different path, the file is a misdirection.
- **Half-finished test files.** A `quality-spec.ts` half-built implies quality coverage; if it doesn't run, the coverage is theatre.
- **Stubbed-out feature directories.** A `features/billing/` directory with 12 files and no integration implies a billing module; if billing is shipped elsewhere or not at all, the directory misleads.
- **Half-built README sections.** *"## Evaluation"* with a paragraph and no implementation behind it implies an evaluation feature even when the working tree doesn't contain one.

In every case the pattern is the same: **the artifact's presence is an implicit claim about the project's behavior. Make the claim true (commit it gated, finish it, document its experimental status) or remove the artifact.** Working-tree implication is a class of communication, whether or not anyone wrote it intentionally.

The pairing worth naming: this is the *file-tree* version of the companion story `forge-redaction-never-fired.md`. That story is a control whose spec says it works but production hasn't tested. This story is a file whose presence says it works but the code hasn't run. Both are about the gap between *the inference a reader would draw* and *the reality of execution*. Both fix shapes are the same: deliberate operator attention, periodic retrospectives, structural disciplines that catch what dev-time tooling can't see. The two stories together make the case that **agentic systems require new categories of latent-state-audit that older systems didn't need** — because agents are confident inferrers, and confident inference on misleading-by-presence state propagates the misinformation through every downstream artifact the agent produces.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *large, structured files that imply functionality the project doesn't deliver*. It doesn't apply to:

- **Scratchpad files clearly named as such.** A `scratch.ts`, `playground/`, `wip-notes.md` — the naming convention signals the experimental status. Future readers know not to infer functionality from these. The cost-of-being-wrong is bounded by the naming.
- **Sub-100-line experiments.** A small file is closer to a snippet than a feature. A 30-line `evaluate-experiment.ts` doesn't propagate the same misleading implication as a 629-line one. The implication-weight scales with the file's heft.
- **Files actively in flight.** If the operator is working on the file *today* and intends to commit *this week*, the working-tree presence is normal in-flight state. The lesson applies to half-built files that have been *abandoned*, not in-progress.
- **Test fixtures or sample data.** A `fixtures/sample-eval-output.json` is data, not a claim about functionality. It might be misleading in other ways (does the project produce output like this?) but the lesson here is about code files specifically.

The signal: *if I left this project for six months and came back, would the file mislead me about what the project does?* If yes, decide its fate now. If no, the lesson doesn't fire.

## What This Story Is *Not* Evidence For

- **Not evidence that experiments shouldn't be tried.** They should. The lesson is about *abandoned experiments left in the working tree without status signal*, not about whether to try ambitious work. Finish the experiment, gate it behind an experimental flag with a README warning, or delete it — but trying it isn't the bug.
- **Not evidence that canvas-mcp's shipped audit pipeline is inadequate.** It works; it stands as the worked example of *the audit is the eval*. The half-built `evaluate.ts` was a parallel attempt at something richer, not a replacement for the audit. The shipped pipeline does its job.
- **Not evidence that the author was careless.** The file got abandoned the way features get abandoned in real projects — someone got pulled to something more urgent, the work paused, the file sat. The structural fix is *catching the abandoned state*, not *blaming the abandonment*.
- **Not evidence that all uncommitted files are misleading.** They aren't. The lesson is about *large, structured files that imply project functionality*. A `.env` file, a `node_modules/` dir, a `tmp/` scratch directory — all expected to be uncommitted, none of them imply load-bearing project behavior.
- **Not evidence that agents are uniquely vulnerable.** Humans are vulnerable too. The lesson is that *agents are vulnerable in one turn* — they integrate working-tree state into a confident output faster than a human would, which makes the misleading-by-presence shape sharper in agentic workflows. But the root cause is the working-tree state, not the agent.
