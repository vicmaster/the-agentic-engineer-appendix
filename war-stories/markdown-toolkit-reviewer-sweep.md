# markdown-toolkit / The Reviewer Sweep That Wasn't

## Date / Version Context

- **Date:** 2026-05-04. The afternoon the orchestration playbook was first exercised on markdown-toolkit. The bigger ticket (budget-sheet parser fix) landed as `.architecture/parser-fix-2026-05-04.md` plus the parser changes plus QA fixtures plus reviewer sign-off. The smaller ticket — *delete dead CSS rules* — became `.tickets/2026-05-04-cleanup-v1.md`. The second pass spawned `.tickets/2026-05-04-cleanup-v2.md` ~2 hours later.
- **Project:** markdown-toolkit — WXT (Manifest V3) Chrome extension. One developer, one codebase, six role-specialized Claude Code sub-agents. The `.claude/ORCHESTRATION.md` playbook names the wave structure: PM + Architect (Wave 1, parallel), Engineer (Wave 2), QA + Reviewer (Wave 3, parallel).
- **Surface for this story:** the v1 cleanup ticket and the orchestration playbook's review wave. The technical surface (CSS class deletions in the side-panel UI) is incidental. The story is about *what counts as a complete sweep*.
- **Glossary, used in this writeup:** *Wave* = a phase in the orchestration playbook (e.g. Wave 3: QA + Reviewer). *Reviewer wave* = the third wave, where `code-reviewer` and `qa-automation` run in parallel against the engineer's changes. *Nit* = a reviewer-flagged item that doesn't block approval — typically minor, often a cleanup or rename. *Sweep* = the act of removing all instances of a class of code (here: dead CSS rules) from a codebase. *Cleanup ticket* = a follow-up ticket that captures reviewer nits as a separate piece of work, so the original PR can land and the nits land separately.

## What Was Being Attempted

Use the orchestration playbook to do a small, well-defined cleanup.

The budget-sheet parser fix was the marquee work of the day — three independent defects diagnosed, four implementation steps, fixtures landed, reviewer approved. As part of the reviewer's sign-off, seven nits were noted. The biggest of the seven (Nit 5) was "delete these dead CSS classes": three names — `.brand-text`, `.tool-item-body`, `.tool-list-soon` — that the reviewer had grepped and confirmed were referenced nowhere.

The team's pattern, defensible and clean, was *don't block the original PR on the nits; spin up a small ticket for them*. So `.tickets/2026-05-04-cleanup-v1.md` got written. The engineer worked through the seven nits; the dead-CSS sub-task was three deletions plus the JSX nodes that referenced them. Five classes total when the engineer was done (the engineer found two more in the JSX during the work). The v1 reviewer approved on second pass.

That should have been the end of the cleanup story. The nits were enumerated, the engineer deleted what was named plus a couple more found in passing, the reviewer approved. The orchestration playbook had executed its review wave correctly.

Then the v2 ticket happened.

## What Went Wrong

The v1 reviewer, on the *second* approval pass after the deletions had landed, found two more orphan CSS rules — `.tool-item-name` and `.tool-item-desc` — that the v1 sweep had missed.

The discovery is small in stakes (two more lines of dead CSS) and large in implications. The v1 sweep had been declared complete. The reviewer had approved. The ticket had closed. And there were still dead classes in the codebase that matched the same shape as the ones the sweep had removed.

The two missed rules weren't subtle. They were in the same stylesheet. They had the same `.tool-item-*` prefix as the deleted `.tool-item-body`. A `grep -r '\.tool-item-' src/` would have surfaced every `.tool-item-*` class in one shot; the engineer had deleted what the reviewer named and left what the reviewer hadn't.

This is the structural defect. The sweep's *methodology* was *delete the classes the reviewer named*. That methodology produces a complete sweep only if the reviewer's enumeration was complete. The reviewer was a careful agent doing careful work — and still missed two rules out of an obvious cluster, because reviewers don't grep exhaustively; they read code, they notice dead stuff, they list what they noticed.

The v1 sweep wasn't wrong because the reviewer was careless. It was wrong because the methodology depended on reviewer enumeration being equivalent to a tooling pass, and reviewer enumeration *isn't*.

## How It Was Discovered

The reviewer's second pass on the same area, after the v1 deletions had landed, surfaced the missed rules. The discovery channel was the same role doing the same job a second time and noticing what they hadn't noticed the first time.

This is worth pausing on. The discovery wasn't an automated check. It wasn't a CI rule. It wasn't a grep run by a different agent. It was the same reviewer agent looking at the same code with a slightly different context — *the post-deletion state* — and catching what the pre-deletion read had missed.

The fact that the discovery happened at all is the only reason the lesson is namable. If the v1 reviewer hadn't done a second pass, the two orphan rules would have stayed in the codebase indefinitely, and the sweep methodology would have looked correct from the outside. Most reviewer-driven sweeps in most projects don't get a second pass. They land, they close, and the residue sits in the tree until someone unrelated stumbles on it months later.

## What Fixed It

`.tickets/2026-05-04-cleanup-v2.md` got written immediately. Its Acceptance Criterion 1 was the load-bearing change:

> **AC1.** Run `grep -r` for the relevant pattern across `src/` and `entrypoints/`. The ticket is not done until the grep returns zero results.

This is a different shape than v1's acceptance criteria. v1 said *delete the named classes and confirm they're gone*. v2 said *delete the class of dead code, with the grep as the terminal check*. The grep is a tool command, not a human/agent observation. It returns the same answer regardless of which reviewer is on duty.

The two orphan rules got deleted. The grep returned zero. The ticket closed.

What this fix earned, beyond closing the bug: a methodology change for future sweeps. Any cleanup ticket that depends on enumeration ("delete the dead X") now starts with a tooling acceptance criterion ("grep for X across these directories returns zero"). The reviewer's enumeration is allowed to be incomplete; the grep makes the completeness machine-checkable.

What didn't get attempted: trying to make reviewer enumeration *more* exhaustive. The lesson explicitly rejects that direction. Reviewers — human or agent — read code; they don't grep exhaustively. Asking them to do so is asking them to play tool, badly. The structurally correct fix is to put the tool where the tool belongs and let the reviewer focus on the stuff a tool can't do (judgment, taste, architectural fit).

## The Durable Lesson

A reviewer agent enumerating findings is not a complete sweep. Bake the sweep into acceptance criteria as a tool command, not as a human/agent observation. Treat approval as a side-effect, not the terminal state.

This is a structural rule about orchestration patterns, not a critique of the reviewer role. Reviewers — at every level of the stack, agent or human — are *enumerators of what they noticed while reading*. They are not *exhaustive scanners of the code's full surface*. The two are different jobs. Confusing them is a category error.

> **Heuristic.** When a ticket's acceptance criterion is *delete (or rename, or migrate, or audit) all instances of X*, the AC must include a tool command that returns the answer. Not "the reviewer confirms" — *the grep returns zero*. Not "QA verifies" — *the test asserts the count*. The reviewer's job is to catch the things the tool can't (intent, fit, naming); the tool's job is to catch the exhaustiveness the reviewer can't. Don't conflate the two.

The deeper observation: this is the *orchestration* version of a broader verification pattern. `forge-max-tokens.md` says *check `stop_reason` in code, don't trust the human reading the brief*. `coide-memory-drift.md` says *verify the memory's claim against current code, don't trust the recall*. This writeup says *verify the sweep with a tool command, don't trust the reviewer's enumeration*. Three different layers of the same pattern: the system-level discipline that catches the failure mode the agent (or model, or memory, or reviewer) can't catch on its own.

The shape generalizes:

- **Migration tickets.** A ticket that says "migrate all callers of X to Y" needs a grep-zero AC. The reviewer's enumeration of callers will miss some.
- **Rename tickets.** Rename `oldFn` to `newFn` everywhere. The grep for `oldFn` returning zero is the AC, not the reviewer's "looks complete to me."
- **Dependency removals.** "Remove all uses of `lodash`." The bundle analyzer or import grep is the AC. The reviewer's read of the diff isn't.
- **Eval rule additions.** "Every public endpoint should have rate-limiting." The check should be a static analysis pass that lists endpoints without rate limits, not a reviewer's "I checked the obvious ones."

In all four cases the reviewer is still useful — they catch what the tool can't see. The reviewer is not the ground truth for *exhaustiveness*. The tool is.

This pairs with `markdown-toolkit-subagent-depth.md`. That story is *the orchestration architecture had a hidden assumption about delegation depth*. This story is *the orchestration wave had a hidden assumption about what a reviewer enumerates*. Both are about quiet limits in the orchestration model that don't surface until you've shipped the architecture and watched it produce a wrong-but-plausible result. Both are corrected by making the load-bearing assumption explicit and machine-checkable, not by asking a human or agent to be more exhaustive.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *sweeps that depend on exhaustive enumeration*. It doesn't apply to:

- **Reviews of judgment, taste, or architectural fit.** A reviewer saying "this naming is confusing" or "this abstraction is leaky" or "this control flow is hard to follow" is operating in their actual job. There's no tool command that replaces it. Don't try to mechanize what the reviewer's role is for.
- **Tickets where the sweep is the whole job.** A migration ticket whose acceptance criterion is *all callers migrated* is a sweep ticket and needs the grep AC. A feature ticket whose body includes "and clean up the related dead code" probably shouldn't include the cleanup at all — split the cleanup into its own ticket with its own AC.
- **One-shot exploratory passes.** "Look around for any low-hanging cleanup" is a different shape. The reviewer's enumeration is the entire output; there's no claim of completeness. Don't apply the sweep methodology to discovery work.
- **Cases where the tool command is genuinely impossible.** Some sweeps don't have a clean grep target — *all places where this conceptual pattern exists* may not be machine-checkable. In those cases the lesson degrades: name the limitation in the AC explicitly ("reviewer enumeration; grep target unavailable for this kind of pattern") so the next reviewer knows the sweep is human-bounded.

The signal: *can my AC be expressed as a tool command that returns a specific value?* If yes, write it that way. If no, name the limitation explicitly.

## What This Story Is *Not* Evidence For

- **Not evidence that the reviewer agent is bad.** The reviewer did its job. The job description was wrong; the bug is in the methodology, not the role.
- **Not evidence that all reviews need tool commands.** Reviews of judgment, taste, and architectural fit don't have tool-command equivalents. The lesson is specific to *sweeps* — work that depends on exhaustive enumeration — not to reviews in general.
- **Not evidence that orchestration is broken.** The orchestration wave executed correctly; the wave's *output contract* (what counts as a complete review) was the load-bearing thing that needed adjusting. Same waves, same agents, just with the AC shape changed.
- **Not evidence that this is unique to multi-agent setups.** A solo developer running `git status`, eyeballing the diff, declaring it complete, and moving on has the same failure mode. The lesson generalizes to any sweep where exhaustiveness is implicit. Multi-agent setups make it sharper because the role separation creates an artifact (the reviewer's nit list) that *looks* exhaustive but isn't.
- **Not evidence that the v1/v2 cadence is wasteful.** v1 → v2 is the cadence working as designed. Reviewer found nits, ticket closed, follow-up ticket landed. The lesson isn't to avoid the cadence; it's to make each ticket's AC strong enough that the cadence converges, instead of chaining indefinitely.
