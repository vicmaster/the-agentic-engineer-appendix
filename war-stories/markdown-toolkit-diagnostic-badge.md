# markdown-toolkit / The Badge That Made the Bug Landable

## Date / Version Context

- **Date:** 2026-05-04 ~13:46–17:00. The orchestration playbook's first real exercise; the badge had been part of the side panel since the project's first feature work that morning.
- **Project:** markdown-toolkit — WXT (Manifest V3) Chrome extension, TypeScript, React 19. Pure parser in `src/tools/spreadsheet-to-markdown/convert.ts`. Vitest + jsdom unit tests in `convert.test.ts` (27 tests at the time of the bug).
- **Surface for this story:** the side panel UI's row/col count badge — a five-character string rendered near the converted Markdown output (*"9 rows · 1 col"*). Tiny piece of UI. Load-bearing piece of observability.
- **Glossary, used in this writeup:** *Diagnostic badge* = a small piece of UI in the product that names a fact about the system's state in user-visible terms. *budget-sheet payload* = a real-world Google Sheets paste containing a budget table with freeze-pane wrappers, `<style>` blocks, and nested `<tbody>` elements. *User-side observability* = the operator's view of the system's behavior, surfaced in product UI rather than dev-console logs.

## What Was Being Attempted

Convert a real Sheets paste to a Markdown table.

The tool's design rule was honest: the agent doesn't write the conversion logic, the parser does. Paste from Sheets; get a Markdown table; click Copy. Pure-function parser, jsdom-tested, no LLM in the runtime. By 2026-05-04 13:46 the parser had 27 unit tests and was passing all of them. The side panel had a small diagnostic badge that read the parsed result and showed the row × column count next to the rendered table — added not because anyone asked for it, but because the design instinct was *show the user what you parsed.*

The hypothesis was that synthetic tests were enough. Every test in `convert.test.ts` used a hand-crafted `<table>` fixture. The fixtures matched what a *minimal HTML table* looks like. The tests all passed. The parser was, by every check the codebase made, working.

## What Went Wrong

A user pasted a real budget table from Sheets. The badge said *"9 rows · 1 col"*. The Markdown output was 9 lines, each containing the same first-column value with newlines mashed together.

Three independent defects had aligned:

- **Defect A — header-row width mismatch.** The parser computed column width from the header row but the body rows had different widths. The mismatch produced a corrupted table where columns drifted.
- **Defect B — `cellText`'s greedy nested-block flattening.** The selector list `'p, div, li, tr'` recursed into nested blocks and concatenated their text without separators. `<tr>` should never have been in that list — there's no `<tr>` inside a `<td>` — but it had been carried forward through edits because *it looked plausible.*
- **Defect C — `querySelectorAll('tr')` recursing into nested tables.** Sheets's clipboard payload wraps the data table in a freeze-pane outer table. `querySelectorAll('tr')` from `document` grabbed rows from *both* tables. The selector silently collapsed nested-table rows onto the outer table's structure, producing the "9 rows · 1 col" result.

The 27 unit tests covered none of these. Each test used a synthetic single-table fixture without the freeze-pane wrapper, without `<style>` blocks, without `<colgroup>`, without nested `<tbody>`. Every defect was real; every test fixture lied about what real Sheets pastes look like.

The badge's *"9 rows · 1 col"* was the only visible signal that the parser had produced output the user knew was wrong. Without the badge, the user would have looked at the rendered Markdown — which contained the right *number* of characters but the wrong *shape* — and said "the table is broken." With the badge, the user said "I pasted a 9-row, 6-column table; the badge says 9 rows · 1 col; the parser thinks this is a single-column thing." That second framing makes the bug class-named, not just symptom-reported. Filing the bug took the user less than a minute.

## How It Was Discovered

The user pasted the budget-sheet payload, watched the badge, and filed a one-line ticket.

This is the discovery channel worth pausing on. The badge made the failure mode *legible in the product surface*. The user wasn't a developer running with the side panel's dev tools open; they were doing their actual work, hit the bug, saw the badge, and could describe the wrongness precisely.

The standard alternative — *user sees broken output, files a bug, developer reproduces, developer adds instrumentation, developer finds the failure* — is the multi-hour version of the same path. The badge collapsed it to under a minute. The point is worth naming plainly: the badge said 9 rows · 1 col, and that diagnostic is what made the bug landable in under a day.

The unit tests were a complete miss. All 27 of them passed against the broken parser. Synthetic fixtures lied about what real Sheets pastes look like, and unit tests built on synthetic fixtures couldn't catch the bug that synthetic fixtures couldn't reproduce. The badge caught what the tests couldn't because the badge was instrumented on *output shape*, not *input fidelity* — it reported what the parser actually produced, and the user could compare that against what they had actually pasted.

## What Fixed It

Two layers, in this order:

1. **The diagnostic badge that wasn't broken.** Already in production. Already pointing at the symptom. The badge required no change — it was working as designed. The story is that *the badge was there at all*, not that anyone fixed it.

2. **The parser, in four steps named by the architect role.** `pickDataTable` helper to skip freeze-pane wrappers, `directRows` helper to scope `querySelectorAll('tr')` to the data table only, `cellText` cleanup to drop the vestigial `'tr'` selector, wire-up. The `convert()` public signature stayed unchanged. Four Fixtures (A–D) added to `qa.test.ts`: the real budget-sheet payload, a freeze-pane wrapper, multiple `<tbody>` blocks, `<style>` + `<colgroup>` not polluting output. Test count climbed from 27 to 38.

The structural fix that follows from this story isn't in the parser; it's in the *test discipline*. The four new fixtures are real-world payloads captured from production, committed to the repo. The discipline that follows is blunt: synthetic unit tests are a smoke test; the safety net is a fixture captured from a real Sheets paste and committed to the repo. Pure-and-tested without a real-world fixture is theater.

What's owed but not yet shipped: a captured-from-real-world fixture *for every shape of Sheets paste the tool needs to handle*, treated as the primary test set, with synthetic fixtures relegated to edge-case probes. The discipline is *capture real, test against real*.

## The Durable Lesson

**Make the diagnostic visible in the product, not the console.** A small piece of UI that names a fact about the system's state — a badge, a counter, a status pill — buys you orders of magnitude faster bug reports than logs in a dev console only developers can see.

The deeper observation: this is *user-side observability*, distinct from the dev-side observability that most conversations about the topic spend their time on. Both are real. Both matter. They serve different audiences and catch different classes of failure.

> **Heuristic.** For any product surface that renders a generated artifact, add a tiny diagnostic that names the *shape* of what was generated. Row count, character count, layout name, summary length, the number of items in a list. Render it next to the output where a user can see it. The cost is a few lines of UI. The value is a user-filed bug report that contains the failure shape, not just the symptom. *"9 rows · 1 col"* is a better bug report than *"the table is broken."*

The shape generalizes far beyond a parser:

- **A summary tool** that renders the source word count next to the generated summary. A user who sees *"summary of a 4,200-word document"* on a one-sentence output knows the failure shape immediately.
- **A deck-generation tool** (e.g. presentation-studio-mcp) that surfaces the slide count and per-slide layout next to the output. A user seeing a 30-slide deck of all `two-column-text` knows the layout-fallback ran for everything.
- **A code-review agent** that shows the file count and line count it actually read. A user seeing *"reviewed 3 files, 142 lines"* on a 12-file PR knows the agent didn't see most of the change.
- **A search tool** that surfaces the corpus size, filter shape, and result count. *"3 results out of 200,000 documents matching filter X"* tells the user something *"3 results"* doesn't.

In every case the badge is doing the same job: **rendering a checkable fact about the system's behavior in the surface the user is already looking at.** That's a different discipline from logging the same fact in a dev-only stream. The user-side surface is where bug reports get filed; instrumenting it is the cheapest possible observability investment.

The corollary worth naming: this pairs with `forge-layered-status.md` from the opposite direction. Layered-status is *a status column that lied about which layer it represented* — a diagnostic that surfaced the *wrong* fact and produced a false sense of correctness. The badge here is *a diagnostic that surfaced the right fact* and produced a fast, accurate bug report. **The discipline isn't add-diagnostics-everywhere; it's make-sure-the-diagnostics-name-the-layer-the-user's-question-lives-in.** Both stories converge on the same point: the operator-visible surface is where observability earns its keep.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *user-side observability for failure modes the user can recognize*. It doesn't apply to:

- **Failures that are invisible without specialist context.** A subtle latency regression, a cache hit-rate drop, a per-token billing change — these are dev-observability concerns. Putting them in user-facing UI would be noise. Reserve the badge for things the user can *recognize as wrong*.
- **Outputs where the user can't compare against ground truth.** A creative writing tool, a generated image, a brand-voice draft — the user can read it but can't say "this should have been a 6-column table." Without a ground-truth comparison, the badge has nothing useful to say.
- **High-frequency programmatic outputs.** A logging endpoint, a webhook target, an API consumed by another system. The "user" is code; the diagnostic belongs in structured logs or traces, not in product UI.
- **Mature, well-trodden flows where the failure rate is already low.** The badge earns its place when the failure modes are new, ambiguous, or hard to describe. For a parser that's been correct for two years, the badge is overhead.

The signal: *would a non-developer user, hitting a failure, be able to read this diagnostic and describe what's wrong?* If yes, add the badge. If no, the diagnostic belongs in dev-side observability instead.

## What This Story Is *Not* Evidence For

- **Not evidence that every product needs a row/col badge.** The badge fit because the parser's output had a shape a user could compare against the input. Products where the user can't easily ground-truth the output don't get the same lever; reach for it where the comparison is honest.
- **Not evidence that unit tests are unnecessary.** The 27 unit tests caught everything they were designed to catch. The bug they missed was *outside their design assumption*, not *inside it*. The fix is captured-from-production fixtures alongside the synthetic ones, not synthetic-tests-considered-harmful.
- **Not evidence that the parser was poorly designed.** The pure-function discipline is what *made the fix fast* once the bug was named. The architect could specify fixtures in the architecture doc and the engineer could land them as `qa.test.ts` cases without touching the UI. The parser's shape paid off precisely when the bug landed; criticizing the parser would miss the lesson.
- **Not evidence that this is a multi-agent story.** The orchestration playbook executed cleanly around this bug, and the architect's discipline (372-line root-cause doc naming three defects) caught two defects solo-mode would have missed (see the companion story `markdown-toolkit-role-separation-solo.md`). This story is about *the badge*, which existed before the orchestration playbook did. The orchestration story is its own writeup; conflating them dilutes both.
