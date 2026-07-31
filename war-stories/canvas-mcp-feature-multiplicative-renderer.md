# canvas-mcp / Phase 1 Mockups After Phase 3

## Date / Version Context

- **Date:** Latent throughout canvas-mcp's phase-2 / phase-3 arc. Phase 2 and Phase 3 shipped 22 minutes apart on 2026-03-21 (commits `fc8cdb5` and `f8485bc`) — together ~1,500 LOC across components, icons, exports, presets, gradients, shadows, responsive, and diff. The cross-validation gap was identified in the 2026-05-09 self-interview; pixel-diff baselines are still not in place as of writing.
- **Project:** canvas-mcp — open-source MCP server for AI-driven design mockups. TypeScript monorepo, vanilla canvas rendering. The renderer accretes node properties across phases — each phase adds new fields to the node schema (`gradient`, `shadows`, `backdropBlur`, `componentId`, `overrides`) and corresponding handlers in `renderer.ts`.
- **Surface for this story:** the renderer's test coverage. Per-phase tests exist (one test file per phase, exercising the new properties that phase added). What's missing: tests that take a representative *Phase 1 mockup* (no gradients, no shadows, no responsive) and verify it still renders identically after Phase 3's code has shipped.
- **Glossary, used in this writeup:** *Per-phase test* = a test file that exercises the properties added in one phase, with fixtures that use only those properties. *Cross-phase test* = a test that exercises an older fixture against the current code path, verifying that the older fixture still produces the same output. *Pixel-diff baseline* = a snapshot of rendered output, byte-for-byte or perceptually compared on every change. *Latent regression* = a bug introduced in one phase that affects fixtures from an earlier phase, surfacing only when someone exercises an older fixture.

## What Was Being Attempted

Build the renderer in phases, each phase adding properties and handlers.

The plan was sensible. canvas-mcp's renderer started small — basic shapes (rect, circle, text), basic styling (fill color, stroke color, font), basic layout (absolute positioning). That was Phase 1. The renderer worked. Tests passed. The project moved to Phase 2.

Phase 2 added gradient fills, presets, and icon nodes. The renderer's `applyFill()` function gained a branch for gradient gradients. New tests verified that nodes with `gradient: { ... }` rendered correctly. The Phase 2 tests passed.

Phase 3, shipped 22 minutes later, added shadow rendering (the structured `shadows: [{...}]` API), backdropBlur, componentId / overrides for design-system-style component instantiation, and responsive layout properties. The renderer's main rendering loop gained branches for each new property. New Phase 3 tests verified that nodes with `shadows: [...]`, with `backdropBlur: number`, with `componentId: string`, all rendered correctly. The Phase 3 tests passed.

After the Phase 3 commit, the renderer's test suite looked complete. Every property had tests. Every test passed. The renderer was, by every check the codebase made, working.

What the codebase didn't check: **does a Phase 1 mockup still render the same way it did before Phase 3 shipped?**

## What Went Wrong

Nothing yet, and that's the latent half of the story.

The Phase 3 code touched `renderer.ts` in ways that could plausibly affect Phase 1 fixtures. The render loop now has new branches it didn't have before; the order of operations changed in a few places; the way `applyFill()` handles `undefined` vs. `null` for the new `gradient` property was tightened in a way that *might* interact with Phase 1's solid-fill code path.

No test catches this. The Phase 1 test file uses Phase 1 fixtures — `{ type: "rect", fill: "#4a90e2" }` style, no `gradient`, no `shadows`, no responsive properties. The fixtures still pass the Phase 1 test's assertions because the assertions are about Phase 1 properties. What the assertions don't say is *"and this canvas renders the same bytes it rendered six weeks ago."* They can't say that, because nobody saved a baseline.

The regression shape, if one exists, would look like this:

1. A user has a Phase 1 mockup saved from before 2026-03-21. Nodes are basic shapes with solid fills. The mockup looks fine in the user's bookmarks/exports.
2. After Phase 3 ships, the user opens that mockup. The renderer runs the *current* code against the *old* fixture. Something subtle in the new render path interacts with the fixture's basic-shape rendering — maybe the shadow code now eagerly evaluates a default shadow on every node, and the default looks wrong on shapes that never had shadows. Maybe the responsive layout code re-computes positions in a way the absolute-positioned Phase 1 fixture didn't account for.
3. The mockup renders *almost the same* but not identically. Maybe a shadow appears where there shouldn't be one, or text shifts by a pixel, or a color hue drifts because of a new color-space conversion that wasn't there before.
4. The user notices the difference and either reports a bug (if they remember what the mockup used to look like) or doesn't (if they don't). Either way, the regression has been live since Phase 3 shipped.

The honest version of this story: **we don't know if this regression exists. Nobody has rendered the Phase 1 fixtures against the Phase 3 code with the original output saved alongside.** The risk is latent because the tooling to detect it doesn't exist. Pixel-diff baselines would catch it. Per-phase tests don't.

The structural property worth naming: **feature growth in a renderer is not feature-additive; it's feature-multiplicative.** Each new property the renderer supports interacts with every existing property at the render path. Adding gradients in Phase 2 didn't just add a new branch — it changed the cross-product of *(N existing properties) × (1 new property) = N new interactions*. Phase 3 added five new properties, so the cross-product expanded by 5 × (previous N) plus 10 new pairs among the Phase 3 properties themselves. The interaction count grows quadratically; the per-phase test count grows linearly.

## How It Was Discovered

By writing the self-interview.

This is the discovery channel worth pausing on. The cross-phase gap wasn't caught by any test failing. Wasn't caught by any user reporting a bug. Wasn't caught by code review of the Phase 3 commit. The gap surfaced when the operator was reflecting on the renderer's phase-by-phase build and asked: *do my tests verify that older mockups still render the same way?* The answer was no — and the answer was visible only because the operator asked the question, not because anything had broken.

This is structurally identical to the companion stories `forge-redaction-never-fired.md` and `canvas-mcp-half-built-evaluate.md`: a *latent-state* observation caught by deliberate retrospection. The standard tooling (CI, manual QA, user reports) is blind to these gaps because the gap is *the absence of a check*, not *a failing check*. Tooling can't detect what tooling isn't measuring.

This pattern recurs in the corpus often enough to be its own meta-lesson: **periodic full-project retrospection is the only discovery channel that catches latent-tooling-gaps.** Not because retrospection is magic; because it's the one tool that asks *what's not being checked?* instead of *what's failing?*

## What Fixed It

Nothing yet. The fix is pixel-diff regression baselines, and they're not in place.

The mechanical shape would be:

1. **Capture representative fixtures.** A small library of canvas mockups from each phase — `phase1-rect-circle.json`, `phase2-gradient.json`, `phase3-shadows.json`, plus deliberately-mixed cases (`phase1-mockup-rendered-with-phase3-code.json`).
2. **Render each fixture, save the output as a baseline.** Save as PNG (or compute a perceptual hash for storage efficiency). Commit the baselines to the repo.
3. **On every renderer change, re-render each fixture and compare against the baseline.** Any difference is a *finding* the operator has to explicitly bless (with `update-baselines.sh` if intentional) or reject (if regression).
4. **The CI step fails on un-blessed differences.** Differences are not the same as failures — they're *questions the operator has to answer*. The CI fails because the question is unanswered, not because the answer is "broken."

The implementation is small — ~150 lines for the test harness, plus the baseline images themselves. The hard part isn't writing the code; it's curating the fixture library so it actually covers the cross-product of properties the renderer supports.

What's owed but not yet shipped: the entire pipeline above. canvas-mcp's `VISION.md` has the item open. The structural rationale is in the source self-interview; the code is the next step.

What didn't get attempted: trying to extend per-phase tests to cover cross-phase cases. The lesson rejects that direction. Per-phase tests use property-shaped assertions (*"this node has a gradient"*); cross-phase regression requires output-shaped assertions (*"this canvas's pixels match the baseline"*). The two are different categories of test. Adding more property-shaped tests doesn't catch what output-shaped tests catch.

## The Durable Lesson

**Feature growth in a renderer is not feature-additive; it's feature-multiplicative.** Each new property interacts with every existing one. Per-phase tests confirm each phase in isolation. They don't catch the regression where Phase N's logic interacts with Phase 1's behavior to produce a corrupted render that *neither phase's tests would touch.*

The mental-model flip the lesson rests on: **per-phase tests are *unit tests for the property*; pixel-diff baselines are *integration tests for the renderer.*** The two are different categories. Unit tests can't catch interaction bugs because they don't exercise the interactions. Integration tests can't catch property-level bugs because they don't isolate. You need both, and the discipline of *what gets each* is what most teams miss.

> **Heuristic.** For any system whose output shape is hard to assert with traditional unit tests — renderers, generated SQL, formatted documents, structured emails, multi-step plans, generated images, agentic outputs — snapshot the output of representative inputs and diff every change against the snapshot. The diff isn't necessarily a fail; it's a question the operator has to answer. *"Yes, that change is intentional"* or *"no, that's a regression."* Pixel-diff baselines (or their equivalent for non-image outputs) are how you make integration-level interaction bugs visible at CI time instead of letting them sit latent until a user reopens an old artifact.

The shape generalizes far beyond renderers:

- **Generated SQL.** A query builder that accretes optimization passes over time. Each pass adds a transformation. Per-pass tests verify the new transformation. Snapshot baselines of a representative query corpus, diffed on every change, catches the case where a Phase 3 optimization breaks a Phase 1 query.
- **Formatted documents.** A markdown-to-PDF renderer with growing feature support. The cross-product of features grows quadratically; per-feature tests don't catch it; baseline PDFs do.
- **Structured emails.** An email composer that supports text, HTML, attachments, calendar invites, signatures. Cross-product is real; baseline-corpus comparison is the fix.
- **Multi-step agentic plans.** A planner that accretes steps over time. A representative-plans corpus rendered after every change, with diffs against baselines, catches regressions in the plan shape that per-step tests can't.
- **Generated images from prompts.** A representative prompt-set with output baselines. Perceptual diffs (not byte-diffs) catch regressions in image generation pipelines.

In every case the rule is the same: **the unit-of-test should match the unit-of-output, not the unit-of-input.** Per-property tests have the wrong unit. Output-snapshot tests have the right unit. The transition from one to the other is the load-bearing eval discipline for any system whose features compound.

The pairing worth naming: this is the *test-shape* version of the companion story `presentation-studio-mcp-feedback-loops.md`. Both stories argue that **the unit-of-evaluation has to match the unit-of-output.** Presentation-studio audits per-deck on every call because the deck is the output unit. canvas-mcp would diff per-fixture on every change because the rendered canvas is the output unit. Different domains (decks vs. canvases), different timing (runtime vs. CI), same structural principle: **eval the output, not the inputs to the output.**

Also pairs with the companion story `canvas-mcp-half-built-evaluate.md`: same project, both stories about evaluation gaps. The half-built evaluator is a *richer scoring* that doesn't ship; the missing pixel-diff baselines are a *basic regression check* that doesn't ship either. canvas-mcp has audit-pipeline-shaped output (the rendered canvas is checkable) but has invested in neither layer of eval — neither the richer scoring nor the cheaper regression baseline. The two stories together make the project's eval gap concrete.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *systems whose features compound interactively at the output*. It doesn't apply to:

- **Systems where features are genuinely orthogonal.** A configuration system where every key is independent; a routing table where each route is isolated. Per-key tests are sufficient because there's no cross-product to test.
- **Outputs that aren't reproducible.** If your system's output is genuinely non-deterministic in a way that defeats byte-diffing (timestamps in the output, non-deterministic ordering, content that depends on real-time external state), pixel-diff baselines aren't useful. Either make the output reproducible at test-time (mocking the time, freezing the order) or accept that baseline-diffing isn't the right tool.
- **Small projects with few features.** A renderer with two properties and no plans for more doesn't need the discipline. The cross-product is two by two by one; per-property tests are enough. The lesson scales with feature count.
- **Prototypes you'll throw away.** Don't engineer pixel-diff infrastructure for code that's not going to ship. The discipline is for projects whose feature set grows over time.

The signal: *would a regression in feature N that affects an output from feature 1 surface in any of my current tests?* If yes, the per-feature tests already cover the interactions. If no, you have the gap this lesson is about.

## What This Story Is *Not* Evidence For

- **Not evidence that per-phase tests are wasted.** They're necessary. They catch the property-level bugs that pixel-diffs can't isolate. The lesson is about *adding* output-snapshot tests alongside, not *replacing* per-phase tests with them.
- **Not evidence that the regression I described actually exists.** The honest version is *we don't know.* The bug might not be there. The lesson is about the *gap in tooling* that would make a regression invisible if one existed — not about confirming a specific bug. Pixel-diff baselines are the discipline that closes the *knowability* gap.
- **Not evidence that canvas-mcp's renderer is fragile.** The renderer works for its supported use cases; users don't report cross-phase bugs (yet). The fragility this story names is *the project's ability to detect regressions if they happen*, not *the project's likelihood of producing regressions*.
- **Not evidence that pixel-diff is the only tool for this.** Perceptual diffs, structural diffs, snapshot comparisons of intermediate representations — all are valid choices. The discipline is *output-snapshot-and-diff*; the specific diff algorithm is implementation detail.
- **Not evidence that the lesson only applies to multi-phase agentic builds.** The same lesson applies to any team that ships features incrementally — single-developer, multi-team, AI-built, human-built. The shape is *feature-multiplicative growth*, not *agentic*. Agentic builds make the shape sharper because the feature-add-rate is higher, but the structural error existed long before agents.
