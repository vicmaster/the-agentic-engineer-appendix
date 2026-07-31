# markdown-toolkit / The Selector That No One Questioned

## Date / Version Context

- **Date:** 2026-05-04 (~14:15, during the FINANZAS bug investigation). The vestigial `tr` had been in the codebase for an indeterminate number of prior edits — the selector list had survived multiple revisions of `cellText` because no one had reason to question its specific contents.
- **Project:** markdown-toolkit — Chrome extension for clipboard-to-Markdown utilities, TypeScript / WXT framework. The `cellText` helper extracts text from table cells during Sheets-paste conversion, walking the cell's child elements via a selector list to handle nested block-level content (paragraphs, divs, list items).
- **Surface for this story:** the parser's text-extraction layer. Specifically `cellText`, the helper that determines how to render a table cell's inner content as a Markdown cell value. The bug was Defect B in `.architecture/parser-fix-2026-05-04.md` — the 372-line architect document that named three independent defects in the FINANZAS-payload arc. (Defect A was the header-row width mismatch; Defect C was `querySelectorAll('tr')` recursing into nested tables — see the companion story `markdown-toolkit-diagnostic-badge.md` for the user-facing surface of that arc.)
- **Glossary, used in this writeup:** *Vestigial code* = code that was correct in an earlier context and has been carried forward through edits without re-validation, even though the surrounding context has changed enough to make it incorrect. *Selector list* = a CSS-selector string passed to `querySelectorAll` or equivalent; the order and contents matter because they encode which elements the helper treats as targets. *Root-cause trace* = the discipline of walking a bug back through the code path to the line that produces the wrong behavior, instead of patching at the symptom site.

## What Was Being Attempted

Extract Markdown-friendly text from each cell of a pasted Sheets table.

`cellText` is called once per `<td>` during table conversion. Its job is to walk the cell's content, identify the block-level elements that need their own rendering (paragraphs, list items, divs that contain structured content), and return a string that's correct as a Markdown cell value. The selector list `'p, div, li, tr'` was the helper's enumeration of *"these are the block-level tags I might find inside a cell."*

Three of the four selectors are reasonable. `<p>`, `<div>`, and `<li>` are all valid children of a `<td>` and can appear in real-world Sheets paste payloads (especially when the source spreadsheet has rich text formatting or embedded lists). The helper needs to handle them.

The fourth — `<tr>` — is not a valid child of `<td>`. The HTML spec doesn't allow it; browsers don't render it; Sheets doesn't paste it. There is no real-world payload in which `cellText` would encounter a `<tr>` inside its target `<td>`.

But the selector list was carried forward through multiple edits. Each edit was small (a tweak to how nested blocks were flattened, a fix to how list items were joined, an adjustment to the trim semantics), and none of the edits had reason to revisit the *selectors themselves*. The list *looked* like a reasonable list of block-level tags to a casual reader. The `tr` blended in.

## What Went Wrong

The `tr` was harmless until it wasn't.

For most Sheets payloads, the vestigial `tr` produced no observable effect — `querySelectorAll('p, div, li, tr')` on a `<td>` whose only children are text and `<span>`s returns an empty list either way. The bug was *latent* for the majority of pastes.

For the FINANZAS payload, the vestigial `tr` interacted with Defect C (`querySelectorAll('tr')` recursing into nested tables) in a way that made `cellText` walk the wrong set of elements. The combined effect was a parser output that flattened nested-block structure incorrectly — *"greedy nested-block flattening"* in the architect's words. The single-cell content collapsed into a confused stream of partial paragraphs, partial list items, and partial what-the-parser-thought-was-rows.

The user-visible symptom (*"9 rows · 1 col"* in the diagnostic badge) was upstream of this defect — that symptom is Defect A's footprint, and is the subject of the companion story `markdown-toolkit-diagnostic-badge.md`. But the *fix* for the FINANZAS payload required catching all three defects, and Defect B (the vestigial `tr`) was the one nobody had ever flagged. It survived:

- **Multiple human edits.** The selector list passed every reviewer's eye because it *looked* like a reasonable list of block-level tags. Nobody traced what each entry was doing.
- **Multiple agent-assisted edits.** Claude (and Codex, in some prior commits) had touched the surrounding code multiple times; the selector list was load-bearing for the surrounding logic, so it was preserved by default. Agents don't question existing code unless asked.
- **The test suite.** `qa.test.ts` had ~27 unit tests before the FINANZAS incident, all against synthetic fixtures. Synthetic fixtures don't include vestigial-tag cases because the tests were written *by people who would not have introduced the vestigial `tr` to begin with*. The test bias and the code bias share an origin — the same mental model produces both, so the test can't catch the code's mistake.

The structural lesson: **agentic workflows accelerate code that's already wrong as fast as code that's right.** The accelerant doesn't discriminate. An agent extending a helper with a wrong selector list extends it correctly *with respect to the wrong selector list*. The extension is consistent; the foundation is rotten.

## How It Was Discovered

By tracing the FINANZAS bug step-by-step.

The discovery channel was the architect role in the multi-agent wave on 2026-05-04. The architect's brief was *"explain why the FINANZAS paste produces wrong output"*, and the architect's method was *trace the parser end-to-end against the actual payload, line by line, until the wrong output is explained.* The 372-line root-cause document (`.architecture/parser-fix-2026-05-04.md`) is the artifact of that trace.

The moment the `tr` got flagged was when the architect, walking through `cellText`, asked the question *"why is `tr` in this selector list?"* — and noticed that no answer made sense. There's no scenario where a `tr` appears inside a `td` in real HTML. The selector was a leftover from an earlier context where the function did something different.

That moment is worth pausing on. Three things had to align for the catch:

1. **The trace was end-to-end, not symptom-anchored.** The architect wasn't patching the symptom (*"the parser outputs 9 rows when it should output more"*); the architect was *understanding the parser*. Symptom-anchored debugging would have added another rule (*"if the row count looks wrong, fall back to X"*) without noticing the vestigial selector.
2. **The architect asked *"why is this here"*, not *"is this correct"*.** The selector list *was* correct for the three valid selectors. The question that flagged the vestigial entry was *"what is this entry doing"* — a different question from *"is this code correct."* Correctness checks pass vestigial code because the surrounding logic still works; *"why is this here"* doesn't.
3. **The architect role was separated from the engineer role.** A combined role would have read the code as the person who would soon write the fix, and the fix-writer's bias is toward *"don't break what's there."* The architect's bias is toward *"explain what's there."* The bias separation made the question possible.

This is also the discovery-channel argument: **vestigial-code bugs are caught by *trace-style* root-cause work, not by *patch-style* symptom work.** Teams that operate exclusively in symptom mode accumulate vestigial code indefinitely. Teams that periodically run trace-style passes catch and remove it.

## What Fixed It

Drop `tr` from the selector. One character less.

The fix is the smallest possible code change in this corpus. `'p, div, li, tr'` → `'p, div, li'`. Three letters, one comma. Two of the FINANZAS-payload defects (A and C) needed substantive fixes (`pickDataTable` helper, `directRows` helper); Defect B's fix was a deletion.

What's load-bearing isn't the fix; it's *the discovery that enabled the fix*. The deletion is trivial; the architecture document that justified the deletion is 372 lines. The disproportion is the lesson — vestigial-code bugs are cheap to fix and expensive to find, which is why they accumulate.

The follow-up the architect document named (and the v2 cleanup ticket from the companion story `markdown-toolkit-reviewer-sweep.md` partially absorbed): **a periodic pass over selector lists, magic numbers, and similar dense-but-easily-skimmed constants, asking *"why is each entry here"* for each one.** Not as part of normal feature work — as a deliberate code-quality sweep. The pass is cheap if done regularly and expensive if put off, because vestigial entries compound.

## The Durable Lesson

Agentic workflows accelerate code that's already wrong as fast as code that's right. **Root-cause traces beat reflexive *"add another rule"* fixes.**

The mental-model flip the lesson rests on: most debugging is symptom-anchored. *"The output is wrong; what's the minimum patch that makes it right?"* That mental model works for *new* bugs introduced by the change you just made — you have a clear before/after and a small surface area to inspect. It breaks for *vestigial* bugs, where the symptom and the cause may be separated by years of edits, and where the cause looks like correct code in its surrounding context. Symptom-anchored debugging adds rules; trace-anchored debugging removes wrong assumptions.

For agentic workflows specifically, the cost compounds. When an agent is asked to *"fix the FINANZAS bug"* and the agent's natural unit of work is *"the smallest change that makes the failing test pass"*, vestigial code is invisible because the agent isn't questioning the existing code; it's extending it. The agent's tendency is the same as a human's tendency, but faster — which means the rate of *"add another rule on top of a wrong foundation"* edits goes up. Without a periodic trace-anchored counterweight, the codebase trends toward a state where every behavior is the sum of many small rules layered on top of vestigial assumptions, and no individual rule can be safely removed because the next-rule-down depends on it.

> **Heuristic.** When a bug surfaces in a code path that has seen multiple prior edits — especially agent-assisted edits — open a root-cause trace before reaching for a patch. The trace's job isn't to find the symptom's cause; it's to ask *"why is each line here"* for the code path, and surface any line that doesn't have a current justification. Most won't be vestigial. The ones that are will look obvious in retrospect — and would have stayed invisible without the deliberate question.

The shape generalizes well beyond `cellText`:

- **Conditional branches kept "just in case."** `if (x == null)` branches that no longer have a code path producing a null `x`, kept because removing them feels risky.
- **Imports that nothing references.** Survived multiple file refactors; the linter catches some but not all (especially in dynamic-language codebases).
- **Configuration keys that nothing reads.** YAML/JSON config files accumulate keys that were once load-bearing and are now ignored; new contributors assume they matter because they're in the file.
- **Comments that describe code that no longer exists.** A comment that names a function or behavior that was renamed or removed; agents reading the file may treat the comment as a constraint.
- **Defensive `try/except` blocks around code that no longer throws.** Originally catching a real exception; the exception path has been removed but the wrapper survives.

In every case, the pattern is identical: **code that was correct in an earlier context, kept through later edits because removing it feels riskier than keeping it, accumulates faster under agent-assisted workflows because agents extend without questioning.** The fix is structural — periodic trace-anchored sweeps that ask *"why is each line here"* rather than *"is each line correct."*

The pairing worth naming: this is the *code-vestige* version of the *don't-trust-the-surface* pattern. See the companion story `coide-memory-drift.md` — both stories are *the artifact and the world have drifted apart, and the artifact's surface still looks correct*. Different artifacts (selector list vs. agent memory), same shape (correct-in-the-original-context, wrong-now, surface-still-plausible). See also the companion story `markdown-toolkit-reviewer-sweep.md` on the same incident from a different angle — it names the *partial-sweep failure* (the reviewer enumerates findings but doesn't catch what wasn't named); this story names the *vestigial-code accumulation* (the code carries forward correctly-looking lines that don't have a current justification). The two together cover the FINANZAS arc's structural lessons.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *code that looks correct in its surrounding context but isn't justified by current usage*. It doesn't apply to:

- **Genuinely new bugs introduced by the change at hand.** When the bug's symptom and the bug's introduction are within a few commits of each other, symptom-anchored debugging is the right tool. Trace-anchored work is overkill for a fresh regression.
- **Code that has a current justification, even if obscure.** Sometimes a line that looks vestigial is actually load-bearing for a rare code path. The fix isn't to delete on suspicion — it's to *find the justification* and either confirm it or delete the line. Trace work surfaces the question; the answer determines the action.
- **Documentation, not code.** Comments and docstrings can outlive the code they describe, but their *behavioral* impact is small. The lesson is for code whose presence affects runtime behavior.
- **Codebases too young to have vestigial accumulation.** A repo two weeks old doesn't have years of edits stacking up. The lesson kicks in once the codebase has seen enough revisions that *who put this here* is no longer a useful question for some lines.
- **Trivial or single-author code.** A small script written by one person who remembers every line has no vestige problem. The lesson is for collaborative codebases, especially those with agent-assisted edits.

The signal: *can you explain why each line in this code path is here, with current evidence rather than historical assumption?* If yes, no vestige. If no, run the trace.

## What This Story Is *Not* Evidence For

- **Not evidence that agents shouldn't extend existing code.** Extending existing code is most of what agentic work *is*. The lesson is that *trace-anchored review periodically* counters the *extending-without-questioning* default, not that the default itself is wrong.
- **Not evidence that linters are useless.** Linters catch many vestigial-code cases (unused imports, unreachable branches, dead variables). The lesson is about cases linters *don't* catch — semantically vestigial code that still parses and runs cleanly.
- **Not evidence that this specific defect was severe in isolation.** Defect B on its own produced no observable user-facing failure. It became material only in combination with Defect A and Defect C in the FINANZAS payload. The lesson is about the *accumulation pattern* (vestigial code compounds with other defects), not about this specific instance being a high-severity bug.
- **Not evidence that the multi-agent wave was unnecessary.** The architect role's separation from the engineer role is exactly what made the catch possible. Solo-mode would have shipped the symptom fix and missed Defect B. See the companion story `markdown-toolkit-role-separation-solo.md` for the broader argument.
- **Not evidence that the existing test suite was inadequate.** The tests covered the behaviors the team had imagined. The bug is at the *un-imagined* layer — code whose justification predates the team's current mental model. The structural follow-up is the periodic trace-anchored sweep, not more tests; tests can't be written for code paths nobody knows are wrong.
