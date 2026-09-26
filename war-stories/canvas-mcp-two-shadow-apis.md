# canvas-mcp / Two Subagents, Two Shadow APIs

## Date / Version Context

- **Date:** 2026-03-21. Commits `fc8cdb5` (22:28) and `f8485bc` (22:50) — 22 minutes apart on the same evening. The contradiction wasn't visible until ~1 week later.
- **What the commit history shows:** only the reconciled end state. `f8485bc` introduces the structured `shadows` array and keeps the Phase 1 `shadow` string as a legacy fallback behind one precedence rule (`shadows` wins). The collision itself happened during the session (author's recollection) and was papered over before it was committed. The later cost is on record: a CSS string passed to `shadows` crashed the renderer (`5e6c03d`, 2026-05-23), and writing `shadow` on a node that already had `shadows` silently did nothing until a warning was added (`3acc3d1`, 2026-07-27).
- **Project:** canvas-mcp v0.1.0 — a single-package Node/TypeScript project; about 1,650 lines of `src/` TypeScript after Phase 3 (about 2,250 counting test scripts), about 2,400 by Phase 4.
- **Build context:** Two phases shipped that evening. Phase 2 = components + icons + presets + exports. Phase 3 = gradients + shadows + responsive + diff. ~1,500 LOC of new code across eight subtasks, fanned out across parallel subagents and merged into two commits 22 minutes apart.
- **Glossary, used in this writeup:** *Fan-out* = splitting one task into N parallel subtasks executed by separate subagents. *Merge commit* = the act of collapsing those subtasks into a single coherent change in the codebase. *Structured API* = an API that takes typed objects (`shadows: [{x: 4, y: 4, blur: 12, color: "#000", opacity: 0.15}]`). *String API* = an API that takes a single formatted string (`shadow: "0 4px 12px rgba(0,0,0,0.15)"`).

## What Was Being Attempted

Ship Phase 2 and Phase 3 in one evening using subagent fan-out. The eight subtasks were assumed to be mostly disjoint:

- Phase 2: components (in `renderer.ts`, `operations.ts` and `types.ts`; there is no `components.ts`), icons (`icons.ts`), presets (`presets.ts`), exports (`screenshot.ts`).
- Phase 3: gradients, shadows, responsive, diff.

The disjointness was real for six of the eight subtasks. Icons, presets, and exports each owned distinct files. Gradients and responsive each touched the renderer in surgically different spots. The fan-out felt safe.

The merge happened as two commits 22 minutes apart, with no integration step in between. Tests for each phase passed. The build was green. Both commits were authored, both reviewed (informally), both pushed.

## What Went Wrong

Two of the eight subtasks both touched `renderer.ts` and `types.ts`. Both shipped a shadow API. Neither knew the other existed.

Phase 2's "components" subtask had defined components as composable bundles that could include shadows, and shipped a structured shadow API: `shadows: [{x, y, blur, spread, color, opacity}]`, with an array because a node could have multiple shadows stacked.

Phase 3's "shadows" subtask, working on the renderer in parallel, had shipped a string-shorthand API: `shadow: "0 4px 12px rgba(0,0,0,0.15)"`, the CSS-style format you copy out of a design tool.

Both worked in isolation. Each subagent's tests covered its slice. Each subtask shipped clean.

In integration: the render path now had two shadow code paths, two type definitions, two sets of validation, two sets of edge cases. A node could legally specify either or both. The interfaces were structurally incompatible (you can't reduce a stacked array of shadows to one CSS string without losing information; you can't expand a CSS string to a structured array without parsing).

The contradiction wasn't visible at merge time because the two commits were pushed without a step where someone read both diffs side-by-side.

## How It Was Discovered

About a week later, while debugging a gradient bug, the user discovered both APIs in the render path. The structured `shadows: [{...}]` API was being processed by one branch; the legacy `shadow: "..."` string was being processed by another; some nodes had one, some had the other, some had both.

Discovery channel: stepping through `renderer.ts` line by line and noticing two unrelated shadow handlers.

## What Fixed It

Nothing.

That's the lesson, and it's the part of this story that's permanent.

The fix that *would have worked* — collapse to one API, deprecate the other, rewrite the calling code — was rejected because both APIs were already in use. The structured API was load-bearing for components (which need stacked shadows). The string API was load-bearing for the simpler call sites the Phase 3 subagent had built around it. Picking one would have meant rewriting the other's tests and re-deriving its design choices.

The decision: support both, document the redundancy, move on.

The README now reads (paraphrased): *the legacy `shadow` string property still works for simple cases.* The structured `shadows` array is the current API. Both are validated. Both are exercised by tests. Both will be supported until canvas-mcp ships a major version that breaks back-compat — which it hasn't.

The dual-path support is a permanent tax. Every future change to the renderer's shadow handling has to consider both paths. Every new node-level feature has to decide whether to integrate with `shadows`, `shadow`, or both. The cost compounds.

## The Durable Lesson

Parallel subagents converging in the same file converge on interfaces that look fine in isolation and conflict on contact.

When two subtasks both ship "the right thing" for their slice, you don't get two right things — you get one codebase carrying two design choices for the same problem. The cost of supporting both rarely shows up at merge time. It shows up every time someone touches the area for the rest of the project's life.

The shape:

> **Heuristic.** Two reasonable answers to the same question is worse than one mediocre answer. The first is permanent in the API surface; the second is a place you can iterate. Fan out only when the subtasks own different files — *check on paper, before kicking them off, that no two will touch the same module.* If two of them will, either serialize them, or pre-assign which one owns the shared file.

Three rules that fall out of the lesson:

1. **The merge is a commit, not a code-review session.** If the subtasks' outputs need to be reconciled by a human reading both diffs and adjudicating overlaps, the parallelism win has already been eaten by merge cost.
2. **One named owner of the merge commit.** That person has the authority to throw one branch's API away when two of them collide. Without that authority, you ship both.
3. **22 minutes between merge commits is not "shipping fast" — it's "shipping without sleeping on it."** Make the merge a separate, deliberate act. Sleep on it. Read both diffs cold the next morning.

## Counter-Example — When Fan-Out Is Right

The lesson is *not* "don't parallelize." It's "parallelize only when the subtasks are honestly disjoint."

Fan-out works when:

- Each subagent can be assigned a named file or module before they start, and the assignment is checked.
- The integration seams are pre-defined types or interfaces that the subagents agree on up front.
- Every parallel branch produces a throwaway test harness that confirms its slice in isolation, so the integration test isn't the first time the parts meet.

The right examples in the same evening: icons landed in a new `icons.ts`, presets in a new `presets.ts`, and the capture features in `screenshot.ts`. Those are the file-disjoint pieces.

The wrong example was two subtasks that *both decided to add a shadow API to the renderer* because each thought it was the natural place to put one. The fan-out was right for the rest of the work and wrong for the renderer.

## What This Story Is *Not* Evidence For

- **Not evidence that fan-out is bad.** Most of canvas-mcp's parallel work merged cleanly. The lesson is about *which subtasks to fan out*, not whether to fan out.
- **Not evidence that the fix is "always serialize."** Serializing the eight Phase 2/3 subtasks would have made the build dramatically slower without solving the underlying issue. The fix is *check disjointness on paper before kicking off*, not *don't kick off*.
- **Not evidence that the dual API is wrong.** Given the constraint (both APIs already in use, rewriting one is expensive), supporting both was the right operational call. The wrong call was *not catching it before both were in use.* The lesson is at the merge step, not at the support step.
