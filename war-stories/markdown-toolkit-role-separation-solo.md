# markdown-toolkit / Role Separation, Not Team Size

## Date / Version Context

- **Date:** 2026-05-04 afternoon. The day the budget-sheet bug landed and the orchestration playbook (`.claude/ORCHESTRATION.md`) was first exercised. Mtimes confirm the timeline: `wxt.config.ts` first feature work at 13:46–13:51, first multi-agent flow at 14:14 (`.tickets/2026-05-04-bug-fixes-v1.md` from project-manager, `.architecture/parser-fix-2026-05-04.md` from team-leader-architect, written in parallel).
- **Project:** markdown-toolkit — WXT (Manifest V3) Chrome extension. One developer. Side project. First feature: spreadsheet-to-Markdown parser. The codebase is small (TypeScript + React 19, parser in `src/tools/spreadsheet-to-markdown/convert.ts`, ~38 unit tests at the time of the bug fix).
- **Team configuration:** six role-specialized Claude Code sub-agents registered in `.claude/agents/`: `project-manager`, `team-leader-architect`, `product-designer`, `senior-engineer`, `qa-automation`, `code-reviewer`. The playbook (`.claude/ORCHESTRATION.md`) names three waves: Wave 1 (PM + Architect + Designer in parallel), Wave 2 (Engineer reading Wave 1's artifacts), Wave 3 (QA + Reviewer in parallel).
- **Headcount reality:** *one human*. The "team" is the developer plus six specialist agents. No other contributors. No collaborators. The team exists entirely at the sub-agent level.
- **Surface for this story:** the *orchestration shape's yield*, not the bug. This writeup is about *what the multi-agent shape produced that solo work wouldn't have*.
- **Glossary, used in this writeup:** *Role separation* = the discipline of having distinct specialist roles each produce a named artifact, even when one person is performing all the roles. *Triangulation* = two roles working from the same brief independently arriving at the same conclusion (corroboration) or at non-overlapping conclusions (combined coverage). *Artifact-or-it-didn't-happen* = the rule that every wave in the orchestration must produce a named file; if there's no file, the role wasn't separated, just narrated.

## What Was Being Attempted

Use the orchestration playbook on a real bug, to see if the multi-agent shape did anything that solo work wouldn't have done.

The setup was deliberately small. The budget-sheet bug — a real Sheets-paste payload that the parser broke on — was a *first exercise* of the playbook. The developer had read the playbook, registered the sub-agents, and was looking for a piece of work that was *non-trivial enough to need the team* but *contained enough to fit a single afternoon*. The bug fit. The hypothesis being tested was implicit but clear: *does the team structure produce yield that the headcount of one doesn't justify?*

The setup-cost was real. Six sub-agents registered. A playbook to keep current. A wave structure to follow. Each wave producing artifacts (tickets, architecture notes, design specs) that wouldn't exist in solo mode. The developer was, in some sense, paying for headcount they didn't have — burning time and tokens on a coordination layer that solo work would skip.

The interesting question, going in, was whether the coordination layer paid for itself when there's no actual coordination problem to solve.

## What Happened

Wave 1 fired at 14:14. PM and Architect ran in parallel, both reading the same user-filed bug report. Twenty-ish minutes later, two artifacts existed:

- `.tickets/2026-05-04-bug-fixes-v1.md` from `project-manager` — the user-facing bug ticket, scoped, with acceptance criteria.
- `.architecture/parser-fix-2026-05-04.md` from `team-leader-architect` — a 372-line root-cause document naming **three independent defects**:
  - **Defect A:** Header-row width mismatch.
  - **Defect B:** `cellText` greedy nested-block flattening.
  - **Defect C:** `querySelectorAll('tr')` recursing into nested tables.

The 372-line document was the surprise. Solo-mode the developer would have looked at the budget-sheet payload, found Defect C (the most obvious one, the one that visibly caused the row-count problem), fixed it, shipped, and moved on. Defects A and B would have stayed in the parser. They would have surfaced as different bugs, later, with different payloads. The bug report would have closed with the right symptom-fix but the wrong root-cause taxonomy.

What forced the full taxonomy was the architect role *having to produce an artifact*. The architect wasn't allowed to verbally hand off "I think the issue is C" — the playbook requires a file. Writing a file made the architect look at the parser end-to-end, trace the budget-sheet payload through each stage, and notice that the problem cluster was three separate things, not one. Once the document existed, the engineer in Wave 2 had to address all three. Once the engineer addressed all three, QA added fixtures for all three. Once QA added fixtures, the reviewer signed off against the full taxonomy.

None of that would have happened in solo mode. The single-developer flow goes: see bug, find cause, fix cause, ship. The multi-agent flow goes: see bug, *produce a named document about the cause*, fix the document's full claim, *produce a named test artifact*, *produce a named review artifact*. The artifacts are the discipline. The roles are the scaffolding that demands the artifacts exist.

The triangulation finding from Wave 1 was the second surprise. PM and Architect, working from the same brief independently, both flagged the wrapper-table hypothesis without coordinating. They corroborated each other. *And* the architect's deeper trace ruled out a hypothesis the PM had wrong — PM had speculated about phantom rows as a cause; the architect's trace showed phantom rows were a symptom, not a cause. Two roles, same brief, independent reads — produced *combined coverage*, not redundant work. Solo mode collapses this into one read by one person, who has one bias surface and one set of priors.

The combined finding: the multi-agent shape produced two distinct yields, neither of which is about parallelism speedup.

1. **Artifact-forcing discipline.** Every wave's named file requirement meant every role had to *commit to a claim*, not just have an opinion. Commitments-on-paper catch root-cause taxonomies that opinions-in-head don't.
2. **Triangulation across roles.** Two independent reads of the same brief catch things one read misses, *even when both readers are the same person performing different roles via different agents*. The non-overlap is real because the role-prompt biases the read differently.

Neither yield requires more than one human. Both are unlocked by the role structure plus the artifact discipline.

## How It Was Discovered

The "discovery" here isn't of a bug — it's of the *shape of the yield*. The developer noticed it during Wave 2, when the engineer was implementing the fix and had three defects to address instead of one. The thought, paraphrased: *I would have shipped the Defect-C fix and called it done. The architect's doc is making me fix two more things I wouldn't have noticed.* That's the moment the orchestration value became visible.

The retrospective notice came at the end of the day, looking at the four artifacts produced: ticket, architecture note, test additions, reviewer sign-off. Each one was load-bearing — strip any one and the taxonomy collapses back to the symptom-fix. The team structure isn't valuable because there are more agents working; it's valuable because there are more artifacts being produced. *Six agents producing one artifact each beats one agent producing six artifacts*, because the role-prompted bias on each artifact is what creates the coverage.

The deeper retrospective notice came later, comparing this afternoon to similar bugs the developer had fixed solo on other projects. Solo-mode bugs close fast and ship the symptom-fix. The root-cause is usually right *for the symptom* and incomplete *for the system*. Three weeks later, a different payload triggers a different symptom from the same root-cause cluster. The team-mode bug closes slower (more artifacts to produce) and ships the symptom-fix plus two preventive fixes. Three weeks later, the related payloads don't trigger anything because the cluster got fixed.

The yield was measurable but not in the way the setup-cost framing predicted. Wall-clock time was *probably similar* to solo mode (the parallelism win was real but small at this codebase size). Token cost was *higher* (six agents, each with their own context). The yield was in the *future bugs that didn't happen* because Defects A and B were caught alongside C. That's the kind of value that doesn't show up in the moment.

## What Fixed It (Or Rather, What Made the Pattern Repeatable)

Three discipline moves emerged from the afternoon and got committed to the playbook:

**Artifact-or-it-didn't-happen, as a hard rule.** Every wave must produce a named file. No file → no role → no separation. This is the load-bearing rule. Without it, the orchestration collapses to narration ("the architect thinks it's a wrapper-table issue") which is just solo work with extra steps. The rule is in `.claude/ORCHESTRATION.md` and reinforced in the playbook's checklist.

**Wave-1 roles must be parallel, working from the same brief without coordinating.** This is what produces triangulation. If PM and Architect coordinate before producing artifacts, they collapse to one read with two names on it. The playbook explicitly says: *PM and Architect fire concurrently, neither reads the other's draft until both are filed*. The non-coordination is the feature, not a bug. (This is also where the `canvas-mcp-two-shadow-apis.md` failure mode lives in negative — parallel agents converging on the same *artifact* produce conflict; parallel agents converging on the same *brief* with separate artifacts produce triangulation. The artifact non-overlap is what makes the brief overlap work.)

**Wave 2 stays serial.** One engineer reads both Wave-1 artifacts. Critically, *not two engineers in parallel*. The corpus's sharpest fan-out failure (`canvas-mcp-two-shadow-apis.md`) is exactly this anti-pattern playing out: two parallel sub-agents both editing `renderer.ts` and shipping a permanent dual API. markdown-toolkit avoided that mode only because Wave 2 is one engineer alone. The playbook's serial Wave 2 is load-bearing — it's the rule that prevents the worst orchestration failure mode.

What didn't get attempted: trying to extend the team further. The temptation, after a successful afternoon, is to add more roles — a security reviewer, a performance specialist, a documentation writer. The decision was explicitly to *not* add roles unless a future bug surfaces a gap the current team didn't cover. Roles cost setup, coordination, and tokens. The cheapest team that produces the right artifacts is the right team. Add only when the missing role is named by a missed defect.

## The Durable Lesson

**Multi-agent orchestration is for role separation, not team size.** Even as one person, having distinct agent roles produce distinct file artifacts forces commitment to claims that solo mode lets you skip. The discipline is artifact-production, not headcount. The yield is bias diversity across roles, not parallelism across hands.

> **Heuristic.** Before deploying a multi-agent playbook, ask: *what artifacts will this produce that I wouldn't produce solo?* If the answer is "the same artifacts, just slower and more expensive," skip the playbook. If the answer is "a root-cause document I wouldn't write, a test taxonomy I wouldn't enumerate, a review note that catches what my first read would miss," the playbook earns its overhead. The cost of role-separation is paid in setup and tokens; the value is paid in future bugs that don't surface because the discipline caught their root-causes alongside the symptom.

The deeper observation: the pattern in this writeup, together with `leads-crm-vision-as-agent-memory.md` (single living artifact) and `magmalabs-delegated-skill-decay.md` (operator-side decay), forms a coherent frame for what a team is in agentic systems. All three are about *the team's structural primitives* — what artifacts exist, who owns them, what discipline keeps them alive. None of the three depends on team *size*. Each could be one human plus one or more agents. The lessons are about *structure*, not *headcount*. That's the thesis in compressed form: *team in agentic systems is about role separation and artifact discipline, not about how many humans are present*.

The shape generalizes outside agentic work:

- **Solo developers reviewing their own PRs.** A self-review pass that has to produce a *written checklist of concerns* before merging catches things a solo *glance-then-merge* misses. The artifact (the checklist) is the discipline.
- **Researchers writing methods sections before running analyses.** The pre-registration discipline in science is the same shape: forcing the artifact (the registered protocol) to exist before the work catches the "I'll fit the hypothesis to the result" failure mode.
- **Architects producing design documents before writing code.** RFC processes, ADRs, design docs — all the same shape. The artifact-before-implementation rule catches root-cause taxonomies that implementation-then-document misses.
- **Editors making outlines before writing.** The outline-before-prose discipline forces commitment to structure before commitment to language. Same shape.

In every case, the underlying principle is the same: *role separation forces artifact production; artifact production forces commitment to claims; commitment to claims catches things that uncommitted thinking misses*. The principle predates agents by decades. Agents make it operationally cheap to enforce, because the artifact-producing "role" can be a sub-agent prompt that costs tokens, not a teammate that costs headcount.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *non-trivial work where root-cause taxonomy matters*. It doesn't apply to:

- **Trivial work where the symptom-fix is the right fix.** Typo fixes, lint cleanups, single-file edits. The artifact-production cost outweighs any taxonomy yield. Don't deploy the playbook against work it doesn't fit.
- **Exploratory work where the brief is ill-defined.** Wave 1 triangulation requires a brief both PM and Architect can read independently. If the brief is "look around for things to improve," the parallel roles produce uncorrelated wandering, not triangulation. The pattern needs a fixed point of agreement to triangulate against.
- **Solo developers who already have the artifact discipline.** Some developers write design docs, root-cause analyses, and test taxonomies in solo work because they've internalized the discipline. For them, the multi-agent shape adds overhead without new yield. The pattern serves developers who need *external structure* to enforce internal discipline — which is most developers, most of the time, but not all of them.
- **Projects where the role-prompt biases don't actually differ.** If you configure six sub-agents but all six prompts are basically "be a careful software engineer," you don't get bias diversity — you get six copies of the same read. The role separation has to be real in the prompts (PM = scope, Architect = root-cause, QA = adversarial, Reviewer = consistency, etc.) or the triangulation is illusory.

The signal: *would the work, done solo, produce the artifacts the playbook would produce?* If yes, skip the playbook. If no, the playbook earns its overhead specifically by being a forcing function for those artifacts.

## What This Story Is *Not* Evidence For

- **Not evidence that solo developers should always use multi-agent playbooks.** The setup cost is real. Trivial work doesn't justify it. The pattern earns its overhead on non-trivial work with multiple-layered root causes. Use it where the cost is paid back.
- **Not evidence that more roles is better.** Six roles is the playbook in this project. Adding more (security reviewer, performance specialist, etc.) costs setup and tokens without clear additional yield until a missed defect names the gap. Grow the team in response to missed coverage, not in anticipation of it.
- **Not evidence that this replaces real teams.** A team of humans has dimensions (politics, accountability, learning, career growth) that a team of agents can't reproduce. The lesson is about *discipline* and *artifact production*, not about replacing humans. Use the pattern where headcount isn't available; don't use it as a reason to not hire.
- **Not evidence that all multi-agent setups produce triangulation.** Triangulation requires (a) parallel work, (b) non-overlapping artifacts, (c) prompts that bias the roles meaningfully differently. Configure any of those wrong and you get six copies of the same read. The pattern's yield is conditional on the configuration, not automatic from the headcount.
- **Not evidence that this generalizes to all project shapes.** The budget-sheet bug is a parsing problem with a layered root-cause cluster — exactly the shape where root-cause taxonomy matters. UI tweaks, copy changes, refactors, and many other work shapes don't have layered root-causes and don't benefit from the artifact-forcing discipline the same way. Match the pattern to the work shape.
