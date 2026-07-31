# lead-crm / VISION.md as Agent Memory

## Date / Version Context

- **Project:** lead-crm — Rails 8 / PostgreSQL / Hotwire internal CRM for MagmaLabs sales qualification. Repo started 2025-11-26 (`86c83d4 Initial commit`). 63 commits across ~23 weeks at the time of writing (2026-05-11).
- **Coordination primitive:** `VISION.md` at repo root (5,283 bytes as of 2026-04-15). Plain-Markdown document with checklist items grouped by area. Edits visible in `git log -- VISION.md`. Five edits in the project's history at the time of writing — each one preceding or accompanying a substantial feature commit.
- **Skill anchor:** `.claude/skills/ship-feature/SKILL.md` (646 bytes, 11 lines). Reads VISION.md, finds an unchecked `- [ ]` matching the `$ARGUMENTS`, flips it to `- [x]`, stages all changed files, writes a commit message describing the *work* (not the checklist update), commits, pushes.
- **Agent anchor:** `.claude/agents/rails-developer.md` (3,783 bytes). One committed agent definition. Acts as the senior Rails engineer specialist — reads routes/schema/models, follows Rails conventions, implements migrations/controllers/views/services, runs tests. The only build-time agent in the repo's committed configuration.
- **Surface for this story:** the *coordination layer* — VISION.md + the `/ship-feature` skill + the rails-developer agent. The Rails app underneath is incidental to the lesson.
- **Glossary, used in this writeup:** *Living artifact* = a versioned document that both human and agent read and write across the project's lifetime, distinct from documentation (read-only-by-humans) and task trackers (read-only-by-the-system). *Gating artifact* vs. *documenting artifact* — the distinction matters: a gating artifact must be updated *before* the work it gates; a documenting artifact is updated *after*. *Specialist + Surface* = the orchestration shape of one committed build-time agent plus a wide product-level tool surface for external agents.

## What Was Being Attempted

Coordinate solo-plus-one-agent work on a Rails app without a separate task tracker.

The setup is the question this writeup answers, and the answer is more subtle than "no tracker." The project is small — one human, one Claude agent, occasional Codex passes. A real task tracker (Linear, Jira, Asana, Trello) is overhead the project doesn't earn. A pure todo list in a personal notes app would work, but only as long as the human reads it. The interesting constraint is *what does the agent read* — what artifact does Claude have access to that tells it both what the project is and what's not done yet.

The answer that emerged: VISION.md. The document started life as a roadmap — what the CRM was for, what it would have, what was deferred. Over time the checklist accreted. The `/ship-feature` skill formalized the loop: instead of marking items done manually, the skill does it as part of the feature commit. Once that loop existed, VISION.md became something *the agent reads to know what's worth working on*, not just *the human reads to know what they planned*. The shift was subtle but real: the document moved from documenting-after-the-fact to gating-before-the-fact.

By the time the pattern had been in use for a few weeks, the discipline was: every commit that adds a feature must reference a VISION.md item. If the item doesn't exist, add it to VISION.md *first*, in a separate commit, then build the feature. The `/ship-feature` skill enforces the bottom half (item gets checked in the feature commit) by mechanism. The top half (the item must exist before the work starts) is human discipline — and the moments it almost broke are the evidence the discipline matters.

## What Almost Went Wrong (And Why That's the Evidence)

This isn't a writeup about a failure; it's a writeup about a pattern that *held*. But the moments when the pattern *almost* broke are what prove it's load-bearing.

**The bulk delete moment, 2026-04-15.** The git log shows two commits on the same day in this order: `4dc9469 Update vision for bulk delete support` then `88e60f8 Add bulk delete actions for pipeline stages`. The order matters. The first commit added the line `- [ ] Bulk delete actions for pipeline stages` to VISION.md. The second commit implemented the feature and flipped the checkbox.

The temptation, every time, is to implement first and update VISION.md second — or not at all. The cost of doing it the other way is one extra commit and a few minutes of writing the line. The benefit is invisible at the moment of the commit and substantial six months later: the entire feature history is reconstructible from `git log -- VISION.md` because every feature has an item that was added before it. Strip the bulk-delete commit pair out of the history and look at just the implementation commit (`88e60f8`) — you can read it as a feature, but you can't read it as a *decision*. The VISION update is the decision; the implementation is the execution. The two-commit pattern keeps them visible as separate steps.

The bulk-delete pair is one of several. The pattern repeats whenever a feature is bigger than a tweak. The discipline holds because the agent can't ship what isn't on the contract — `/ship-feature` literally errors out if there's no matching unchecked item, asking the human to either add one or pick from the existing list. The skill enforces the gate at the moment of commit. The human has to consciously *not* enforce it to skip the discipline.

**The almost-fail mode.** The pattern *would* break if the human got into the habit of adding items to VISION.md retroactively — implementing the feature, then opening VISION.md, then adding the item, then running `/ship-feature` against the just-added item. Mechanically possible, structurally identical to "no gating." Discipline matters more than the skill's enforcement because the skill only enforces *that an item exists*, not *that the item existed before the work*. This is the load-bearing weakness of the pattern. In lead-crm it hasn't broken yet because the human is the same person who set up the pattern and remembers why it matters. In a larger team, the weakness would surface fast.

The other risk that hasn't materialized: VISION.md drift. After 23 weeks, the document could in theory have become a graveyard of half-done checklist items, stale wishes, abandoned features. The reality is closer to the opposite — items get checked off or removed, the doc stays current. This is partly because it's small (5KB) and partly because reading it is part of the feature loop, so drift is visible immediately.

## What Fixed It (The Pattern Itself)

The fix at the architectural layer is the pattern: one living artifact, owned by the human, read and updated by the agent at every feature commit, with the gating discipline preserved by humans-not-doing-it-retroactively.

Concretely in lead-crm:

**The artifact.** VISION.md at repo root. Markdown. Checklist groups by area (e.g. *Reports*, *Pipeline*, *Auth*, *Operations*). Each item is a `- [ ]` or `- [x]` line short enough to grep against. The format is deliberate — a too-rich format (Linear-style ticket bodies, dependencies, priorities) defeats the cheapness that makes the pattern work. The artifact has to be small enough to read fully on every loop and dense enough to communicate intent in a single line.

**The skill.** `/ship-feature` is six lines of instruction (verbatim in the source file). It reads VISION.md, matches `$ARGUMENTS` to an unchecked item, flips it, commits, pushes. Two of those steps matter most: the *matching* step (which makes the skill refuse if no item exists, prompting the human to add one) and the *combined commit* step (which keeps the checkbox flip and the feature code in the same git commit, so future-`git log` shows them together).

**The agent.** One specialist (`rails-developer.md`). The skill itself is invocation-agnostic — anyone with `/ship-feature` access can run it — but in practice the rails-developer agent is the one that runs it inside its feature work. The combination is what makes the pattern *specialist + surface*: one committed build-time agent that knows Rails, plus a wide product-level MCP tool surface for external agents (15 tools across leads, scoring, reports, settings) for runtime agent use. The lesson lives in the build-time half.

**The discipline.** Every agent-authored commit must reference an item on the living spec, and the agent must update the spec in the same commit. The human must add new items *before* the work, not after. The skill enforces the bottom half; the human is on the hook for the top half.

What didn't get attempted: replacing VISION.md with a tracker tool (Linear, Asana, etc.). The cost of integration would be high, the value low at this project scale, and the *agent reads the file* property would be lost. The agent can read VISION.md without an API key, without a webhook, without rate limits — because it's just a file in the repo. That property is the cheapness that makes the pattern work; it doesn't survive the move to a hosted tracker.

## The Durable Lesson

**Use one living artifact that the agent reads and updates each turn, owned by the human, gating the work before it starts.** With one human and one specialist agent and one such artifact, ownership is unambiguous, history is reconstructible, and the coordination overhead is essentially zero.

> **Heuristic.** If you can't trace a change back to a checked-off intent on a shared artifact, you're not orchestrating — you're accumulating drift. The artifact has to be (a) cheap enough to read fully on every loop, (b) gating (updated before the work, not after), and (c) human-owned (the agent reads and flips checkboxes; the human decides what's on the list). Strip any of those three and the pattern degrades back into informal coordination.

The deeper observation: this writeup, together with the companion stories `leads-crm-codex-pr1.md` (cross-vendor review) and `magmalabs-delegated-skill-decay.md` (operator atrophy), forms a three-legged stool. Each names a different role the *team* plays in agentic systems, and each names a structural primitive that makes the role survive scale:

- **Cross-vendor review** is the *peer review* primitive — different bias surfaces catch what same bias surfaces miss.
- **Living artifact ownership** is the *coordination* primitive — the artifact is the team's shared memory, not anyone's recall.
- **Operator atrophy management** is the *practice* primitive — the operator must keep practicing what they can still spot when wrong.

All three converge on the same shape: **the team's structural primitives must live outside any single party's say-so.** The reviewer can't be the agent that wrote the code. The plan can't be in anyone's head. The operator's quality bar can't be the operator's memory of their own quality bar. In each case the durable move is to put the load-bearing thing into an artifact that survives the moment.

The shape generalizes outside lead-crm:

- **Solo developer with one agent.** The same VISION.md pattern works for any project where the bottleneck is *coordination between today's-self and tomorrow's-self*. The agent is the consistent reader; the artifact is the consistent state. The pattern is small-team-friendly in a way most tracker tools aren't.
- **Tech-lead with a small reporting team.** Replace "agent" with "team member" and the same pattern works as a shared `ROADMAP.md` in the repo. The advantage over a hosted tracker: the artifact is versioned with the code, reviewable in PR diffs, and survives tool changes.
- **Editorial workflow with contributors.** A shared `OUTLINE.md` listing chapters, each as a checklist item, owned by the editor, read by writers. Same gating discipline: the item exists before the writing.
- **Research team with shared question doc.** One living `QUESTIONS.md`, owned by the lead researcher, read by analysts. New analyses trace back to items; new items added explicitly before the analysis begins.

In every case, the leverage is the *single living artifact*, not the headcount. The principle is older than agents — Conway's Law, separation-of-concerns, "the deliverable is the spec" — but agents make it operationally cheaper because the agent will actually read and update the artifact on every loop, where a human team member might let it drift.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *solo-plus-one-agent or very small team* coordination. It doesn't apply to:

- **Teams large enough to need real backlog management.** Once the team has more than ~3 contributors and the backlog exceeds ~30 active items, the lightweight Markdown-checklist pattern degrades. Real trackers exist for a reason — search, filtering, dependencies, priorities, status views. VISION.md's cheapness is also its ceiling.
- **Projects where the agent doesn't have repo read access.** The pattern relies on the agent reading VISION.md cheaply on every loop. If the agent is sandboxed away from the repo (only sees diffs, only sees prompts), the artifact has to live somewhere the agent *can* see, and the operational cheapness disappears.
- **Work where intent and implementation can't reasonably be split into checkable items.** Research, exploratory design, open-ended investigation. The artifact form (checklist of intended deliverables) doesn't fit because the deliverables aren't known up front. A research log or open-questions doc is the right artifact shape, not a checklist.
- **Compliance contexts where the audit trail needs to survive repo deletion or migration.** Git log + VISION.md is reconstructible only if the repo exists. For regulated work, a separate audit-grade tracker is the safer move — VISION.md is the working artifact, but the audit-grade record lives elsewhere.
- **Multi-team coordination across repos.** VISION.md is per-repo. Coordinating two repos that both depend on a single roadmap needs a cross-repo artifact, which then loses the *file in the same repo the agent is editing* property that makes the pattern cheap.

The signal: *is the artifact small enough to read fully on every loop, and is the team small enough that one document captures the active state?* If yes, the pattern earns its place. If no, the pattern is the wrong tool, and reaching for a real tracker is the correct move.

## What This Story Is *Not* Evidence For

- **Not evidence that you should rip out your tracker.** The lesson applies to small projects with small teams. If your team is larger or your backlog is real, keep the tracker. The lesson is about the *kind* of artifact that scales down to solo-plus-agent work, not about replacing tooling at scale.
- **Not evidence that Markdown is special.** YAML, JSON, a text file — the format doesn't matter beyond *the agent can read it and write it*. Markdown happens to be readable in PR diffs and renders nicely in GitHub. Use whatever format the agent already reads natively.
- **Not evidence that one agent is always enough.** lead-crm has one specialist build-time agent because the project's scope fits one specialist. Larger or more diverse projects need more agents. The lesson is about *the artifact*, not about *the team size*.
- **Not evidence that the pattern is self-enforcing.** The skill enforces *that an item exists at commit time*. It does not enforce *that the item existed before the work*. The human can defeat the gating discipline by retroactively adding items, and the pattern doesn't catch it. The pattern works because the human cooperates with it; it doesn't work because the mechanism is bulletproof.
- **Not evidence that VISION.md will scale forever.** After 23 weeks the document is current and useful. After 230 weeks, with three more contributors and a different agent vendor, it might not be. The lesson is about *the pattern's fit at this scale*, not about *the pattern's permanence*. Re-evaluate when the team or scope grows.
