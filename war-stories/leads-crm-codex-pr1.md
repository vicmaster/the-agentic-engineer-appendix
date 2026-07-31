# lead-crm / Cross-Vendor Agent Disagreement as a Review Primitive

## Date / Version Context

- **Date:** 2026-04-29. Codex opens PR #1 against `magma-labs/lead-crm` from branch `codex/fix-settings-toggle-autosave`. Two commits land via the merge: `73cae83 Fix settings boolean toggle persistence` and the merge commit `3b28c03 Merge pull request #1 from magma-labs/codex/fix-settings-toggle-autosave`. The diff touches `app/views/settings/index.html.erb` and `spec/controllers/settings_controller_spec.rb` — a single false-path bug and its regression test.
- **Project:** lead-crm — Rails 8 / PostgreSQL / Hotwire internal CRM. Built primarily by Victor with one committed Claude agent definition (`.claude/agents/rails-developer.md`, sonnet, 6 tools). 63 commits across ~23 weeks at the time of the PR. Codex is a separate agent vendor (OpenAI's coding agent), pointed at the same repo independently.
- **Prior state:** Before PR #1, the Settings form had a boolean-toggle persistence bug. The toggles looked like they should save and didn't reliably — the unchecked-checkbox case wasn't handled. Rails's `form.check_box` semantics require a hidden field for the false path; the Claude-built form omitted it. The bug had been in production long enough to matter; Victor had noticed the symptom but not isolated the cause.
- **Surface for this story:** the *PR itself*, not the bug. The bug is a recurring Rails-shaped mistake. The story is about who caught it and how that catch became visible afterward.
- **Glossary, used in this writeup:** *Agent vendor* = the company providing the agent's underlying model and harness (Anthropic for Claude, OpenAI for Codex, etc. — distinguished from *agent instance* which is a separate run of the same vendor's agent). *Cross-vendor review* = review of one vendor's output by an instance of a different vendor. *Branch-name provenance* = the practice of using descriptive branch names (e.g. `codex/fix-x`) so the originating agent or system is traceable from `git log` alone, without external metadata.

## What Was Being Attempted

Use Codex to do a small repo cleanup pass while Claude was being used for ongoing feature work.

The setup wasn't a planned cross-vendor experiment. Codex was being used loosely — pointed at the repo, asked to look for things to improve. The expectation was something between *quality-of-life nits* and *let's see what it notices*. The Settings form bug had been a low-grade annoyance for weeks; Victor had assumed the autosave behavior was a Rails edge case he'd circle back to.

What Codex did with the cleanup pass: opened the form file, recognized the false-path issue in Rails check_box semantics, generated the fix plus a regression test, and committed it on a branch named `codex/fix-settings-toggle-autosave`. The branch name was the agent's default — Codex's harness conventionally prefixes its branches with the agent identifier. No discipline was being applied here; the audit trail was a byproduct.

The PR opened. The fix was correct. The merge happened with no special fanfare. From outside, the commit graph looks like a normal external contribution.

## What Went Wrong

Nothing — and that's exactly the story. The interesting fact isn't a failure mode. It's that **one agent (Claude) had built a form that shipped a bug, and a different agent (Codex) reading the same code caught it.**

This needs to be sat with. The Rails false-path bug is the kind of mistake any agent would make once and any agent would catch once — it's a well-known Rails idiom. Claude has the knowledge; Claude wrote the form anyway with the bug in it; the form went through whatever review existed (Victor's own glance, the existing test pass) without anyone noticing. Codex, looking at the same code from the same starting point, surfaced it. The difference wasn't capability. The difference was *perspective at the moment of reading*.

The structural observation: **same-vendor review is closer to self-review than to peer review.** When Claude reviews Claude-written code, the model's read of "what should be here" overlaps heavily with "what is here" — the failure modes Claude built into the form are the same failure modes Claude is biased to miss when reading the form. The bias doesn't go away by spinning up a second Claude instance to look at it. Two Claude instances are still working from the same training data, the same priors about what Rails forms typically look like, the same patterns of *what counts as suspicious*.

A different-vendor agent has a different bias surface. Codex is trained by a different team on different data with different reinforcement on what's worth flagging. It sees the Rails form with a slightly different read of *what's missing here*. That's not necessarily better — Codex would have its own blind spots Claude wouldn't share. The point is the *non-overlap*, not the superiority. Two different bias surfaces catch more bugs than two instances of the same one.

This is also where the "second-agent-instance" instinct goes wrong. The natural move when you want more review is to add another Claude review pass — more eyes, same eyes. The second pass costs tokens and yields little, because the bias is preserved. The structurally correct move is to add a *differently-biased* reviewer. Cross-vendor is one way. Cross-prompt-style (an explicit *adversarial* review prompt vs. a *standard* review prompt) is another. The principle: scale the team by perspective, not by hand-count.

## How It Was Discovered

The PR opened in GitHub. Victor saw it in the notifications. The diff was small. The fix was correct. He merged it.

The discovery channel is worth lingering on. There was no automated cross-agent CI. No scheduled second-vendor pass. No structured comparison of Claude-vs-Codex output. The discovery happened because someone happened to point Codex at the repo around that time, and Codex happened to notice the bug Claude had shipped.

The fragility of that discovery channel is the story's quiet implication. If Codex hadn't been pointed at the repo that week, the bug would have persisted. If the cross-vendor review pattern only fires when *someone remembers to invoke it*, it isn't a review pattern — it's a coincidence. A real cross-vendor review system would need scheduling, scoping, and a queue of repos that get the second-vendor pass on some cadence. That doesn't exist in lead-crm and probably doesn't exist in most projects. The catch was lucky.

What makes the catch *survive*, beyond the moment of discovery, is the branch name. `codex/fix-settings-toggle-autosave` is the artifact that proves a cross-vendor catch happened. Without the prefix, the commit looks like any other commit in the log. The agent identifier in the branch name is the only audit trail. Six months from now, `git log` will still show `Merge pull request #1 from magma-labs/codex/...` and the multi-agent history will still be reconstructible. Strip the branch name and the same commits look like solo work.

## What Fixed It

The PR merged. The bug was gone. That's the fix at the technical layer.

The fix at the *practice* layer is what came out of reflecting on the moment. Three discipline moves, all small:

**Branch-name provenance as a hard rule.** Any time a non-default agent works on the repo, the branch carries an identifier. Codex's default already does this. Claude's default doesn't, so when Claude is asked to do a specific scoped task — the kind that would be the equivalent of a Codex PR — the branch gets prefixed manually. The cost is a few keystrokes; the value is that a year from now, the multi-agent history is still readable from git alone.

**Second-vendor before second-instance.** When the work warrants more review, the move is to bring in a different vendor's agent, not a second instance of the same one. Concretely, for lead-crm this means: Claude builds the feature; Codex (or Aider, or whatever third-party tool is current) does a review pass before merge for anything touching forms, auth, or money. The frequency is per-judgment-call, not a blanket rule. The rule is: when you reach for "more review," reach for *different* review, not *more of the same review*.

**Treat the merge of a cross-vendor PR as an audit signal, not just a merge.** When Codex opens a PR that catches a Claude bug, that's evidence about the kind of mistakes Claude is making in this repo. Worth a one-line note in MEMORY.md or the equivalent ("Claude tends to miss Rails false-path semantics in form helpers; pre-flight check before merging form changes"). The cross-vendor PR is a data point about the resident agent's blind spots, not just a bug fix. Throwing away that signal because the bug is gone is leaving evidence on the table.

What did *not* get attempted: trying to make Claude more careful about Rails false-paths. The lesson explicitly rejects that direction. Asking the same model to be more careful about a class of bug it just demonstrably missed is asking the model to fight its own priors. The structural move is to bring in a different reviewer, not to plead with the existing one.

## The Durable Lesson

When you want more review of agent-written code, add a differently-perspectived agent before you add another instance of the same agent. Different training, different priors, different blind spots — different review. Same vendor doing it twice is closer to self-review than peer review.

> **Heuristic.** For agent-written code, count the *distinct bias surfaces* in the review chain, not the number of review passes. Two Claude reviews = one bias surface. Claude + Codex = two bias surfaces. Claude + Codex + a human glance = three. The bug rate falls with bias surfaces, not with passes. When the budget for review is fixed, spend it on perspective diversity, not on repetition.

The corollary: if you're using multiple agent vendors, **the commit message and branch name are the only audit trail you'll have a year later.** Git log doesn't carry agent provenance the way it carries author identity. Without naming-discipline at the branch level, the multi-agent history collapses into a single Author=Victor stream that lies about who saw what. The branch-name prefix is the cheapest possible audit-trail discipline. Use it.

The deeper observation: this is the *cross-vendor* version of the structural-check principle that recurs across these stories. `markdown-toolkit-reviewer-sweep.md` says *the trust signal has to live outside the reviewer's enumeration*. `magmalabs-delegated-skill-decay.md` says *the trust signal has to live outside the operator's recognition reflex*. This writeup says *the trust signal has to live outside the originating agent's read of its own code*. Three different layers, same shape: the agent producing the work is not the right judge of the work; the structural check below or beside the agent is.

The shape generalizes:

- **Two-vendor code review.** Any agent-heavy workflow benefits from a second vendor pass before high-stakes merges. Cost: another tool subscription. Value: catches a different class of bug.
- **Cross-vendor static analysis.** Different linters, different type-checkers, different security scanners all have non-overlapping bias surfaces for the same reason. The principle predates agents; agents just make it sharper.
- **Editorial cross-reads.** A copy editor and a fact-checker reading the same draft catch different issues precisely because their training and priors don't overlap. Same shape.
- **External audit vs. internal review.** Internal reviewers share the org's blind spots. External auditors are a different bias surface, and the value is the non-overlap, not the expertise level.

In every case, the move is to scale by *different kinds* of reviewer, not by *more of the same* reviewer. The cost of perspective diversity is real (coordination, tooling, scope) but it's structurally distinct from the cost of repetition, and it buys something repetition can't.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *bias-driven misses* — bugs that a same-vendor reviewer is structurally less likely to catch because the bias matches the bug. It doesn't apply to:

- **Bugs the existing vendor would obviously catch on a second pass.** If a same-vendor instance flags the bug 90% of the time, paying for cross-vendor review is overkill. Reserve cross-vendor for the *systematic* blind spots, not the random ones.
- **Code where bias-driven misses don't matter much.** A prototype, a one-off script, a demo. The bug rate is acceptable; the cost of cross-vendor isn't paid back. Use the lesson where the cost of being wrong is real.
- **Workflows where the second vendor is structurally just-as-blind.** If two agents are trained on essentially the same data with essentially the same priors, calling them "different vendors" is marketing — they have the same bias surface. The signal isn't *named differently*; it's *trained differently*. Some commercial "alternative" agents are functionally identical for review purposes.
- **High-throughput automated review pipelines.** Cross-vendor inevitably costs more (two providers, two API surfaces, two failure modes). For high-volume work where the value of catching one extra bug per ten reviews is low relative to the operational complexity, single-vendor with deterministic linters as the second bias surface might be the right shape.

The signal: *am I looking at a class of bug where this vendor has a systematic blind spot?* If yes, cross-vendor pays back. If no, the cost-of-being-wrong is being managed by other things (tests, types, linters) and the perspective-diversity argument is academic.

## What This Story Is *Not* Evidence For

- **Not evidence that Codex is better than Claude (or vice versa).** The same shape would have played out in reverse — Claude reviewing Codex-built code would catch things Codex missed, in a different class of bug. The lesson is about *non-overlap of bias surfaces*, not about ranking vendors.
- **Not evidence that all PRs should be cross-vendor.** The lesson is about high-stakes or systematic-blind-spot work, not about routine changes. A typo fix doesn't need two vendors. The cost of perspective diversity is real; spend it where it pays back.
- **Not evidence that the bug-as-bug is interesting.** The Rails false-path bug is a Rails false-path bug. The interesting fact is *the catch*, not the bug. The test-discipline framing (for the false case) is a legitimate but different lesson from the same incident. Both can be true.
- **Not evidence that branch-name provenance is sufficient observability.** It's the cheapest possible audit trail, not a complete one. Real cross-agent observability would also include commit-message provenance, MEMORY.md entries about agent-specific blind spots discovered, and a queue of repos that get the cross-vendor pass on a cadence. Branch names are the floor, not the ceiling.
- **Not evidence that this scales infinitely.** The marginal value of a third vendor, a fourth, a fifth falls fast. Two distinct bias surfaces catches most of what one misses; three catches a little more; ten is mostly cost. The lesson is "more than one perspective," not "as many as possible."
