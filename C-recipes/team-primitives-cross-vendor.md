# Team Primitives — Cross-Vendor PRs, Gating Checklists, Role-Separated Playbooks

**Source aside:** Ch. 11 (team structure for agentic systems).
**Source incidents:** `war-stories/leads-crm-vision-as-agent-memory.md`, `leads-crm-codex-pr1.md`, `markdown-toolkit-role-separation-solo.md`, `magmalabs-delegated-skill-decay.md`.
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026, against the Claude Code harness used through the manuscript.

## The body principle

Ch. 11, Principle 11: *the structure is the team.* In agentic systems, the team's leverage is the structural primitives you build — gating artifacts, cross-vendor review surfaces, recognition reflexes the operator keeps practicing — not the headcount. The membership rotates faster than the structure. Engineer the structure; the team follows.

## The recipe at one glance

Four primitives, each from a different project, each surviving the *team-size* property: every one works for a solo developer plus one or more agents, and scales up cleanly. Together they implement the chapter's three-legged stool — reviewing AI-written code, who owns the eval harness, skill atrophy — plus a fourth (role separation without headcount) that the chapter pulls in from `markdown-toolkit`.

This is the operational complement to the body. The chapter argues *why* these primitives matter; this recipe shows *what they look like in commit-able artifacts*.

## The four primitives share three properties

From Ch. 11's heuristic. Strip any one and the primitive degrades back into informal coordination:

1. **Cheap enough to read fully on every loop.** The artifact is short or scannable, not a sprawling document.
2. **Gating — updated before the work, not after.** The artifact's update is a *precondition* for the work, not a record of it.
3. **Human-owned — the agent maintains compliance; the human decides what's on the list.** Roles are bright: the agent flips checkboxes, the human writes the checkboxes.

The four implementations below each enact these properties differently. The properties are what make the primitives load-bearing.

## Primitive 1 — Cross-vendor review with branch-name provenance

**Claim:** *agent-written code has been read by a bias surface distinct from the one that wrote it.*

**Source incident:** `leads-crm-codex-pr1.md`. Claude built a Settings form whose boolean On/Off toggles highlighted the saved value instead of the clicked control, and saved nothing unless the user found a separate *Save All Changes* button, so flipped toggles looked ignored and didn't persist. The bug shipped and sat. Codex, pointed at the same repo independently, opened PR #1 from branch `codex/fix-settings-toggle-autosave` and caught it. Two commits landed: `73cae83 Fix settings boolean toggle persistence` and the merge `3b28c03`.

The *catch* matters more than the bug. Same-vendor review is closer to self-review than peer review — two Claude instances share training data, priors, and the same patterns of *what counts as suspicious*. Two different vendors have different bias surfaces; the *non-overlap* is the value, not the superiority.

### The durable shape (pseudocode)

```
# Pseudocode: cross-vendor review as a structural primitive

# 1. Branch-name provenance — encode the agent vendor in git history
function branchName(vendorTag, scope):
    return f"{vendorTag}/{scope}"
    # e.g. "codex/fix-settings-toggle-autosave"
    # e.g. "claude/feature-pipeline-bulk-delete"

# 2. Spend the review budget on perspective diversity, not repetition
function reviewBudget(diff):
    biasSurfaces = []
    biasSurfaces.append(claudeReview(diff))    # producer's perspective
    if isHighStakes(diff):
        biasSurfaces.append(codexReview(diff)) # different bias surface
    biasSurfaces.append(humanGlance(diff))     # third surface, cheap
    return biasSurfaces

# 3. Treat cross-vendor catches as observability about the producer
function logCrossVendorCatch(producer, reviewer, finding):
    memory.append(
        slug=f"{producer}-blind-spots",
        body=f"{producer} missed {finding.class} in {finding.context}; "
             f"caught by {reviewer}. Pre-flight check before merging {finding.context} changes."
    )
```

### The implementation

The cheapest possible audit-trail discipline is **branch-name prefixes**. The cost is a few keystrokes per branch; the value is reconstructible multi-agent history from `git log` alone.

```sh
# Codex's default — already prefixes by agent identifier
git push -u origin codex/fix-settings-toggle-autosave

# Claude — prefix manually when doing scoped work that warrants the trail
git push -u origin claude/feature-pipeline-bulk-delete

# Aider, Cursor, any other vendor — same shape
git push -u origin cursor/refactor-controllers
```

After PR merge, the `git log` carries the multi-agent history forever:

```
3b28c03 Merge pull request #1 from magma-labs/codex/fix-settings-toggle-autosave
73cae83 Fix settings boolean toggle persistence
```

Strip the `codex/` prefix and the same commits look like solo work. The prefix is the only audit trail.

The *capturing the signal* discipline: when Vendor B catches a bug Vendor A shipped, write a one-line entry into `MEMORY.md` (or its equivalent in the harness) naming the class of bug and the pre-flight check that would have caught it:

```markdown
- [Claude tends to wire UI state to stored values instead of the control](claude-ui-state-from-stored-value.md) — before merging form changes, click each new control and confirm it shows the change and saves it
```

That's the chapter's third leg surfacing here — the cross-vendor PR is a data point about the resident agent's blind spots, not just a bug fix. Throwing away the signal because the bug is gone leaves evidence on the table.

### What survives the vendors changing

The principle: **scale the team by perspective, not by hand-count.** The specific vendors (`claude/`, `codex/`, `cursor/`) will rotate; the *count the distinct bias surfaces in the review chain, not the number of review passes* heuristic doesn't.

## Primitive 2 — Living gating artifact (`VISION.md` + `/ship-feature`)

**Claim:** *every agent-authored commit references a checkbox the human added before the work started.*

**Source incident:** `leads-crm-vision-as-agent-memory.md`. lead-crm coordinates solo-plus-one-agent work on a Rails app without a task tracker. The coordination primitive is a single Markdown file at the repo root plus a six-line skill that reads it. After 63 commits over 23 weeks, the pattern holds.

The clearest evidence is the *bulk-delete commit pair*, 2026-04-15:

```
4dc9469 Update vision for bulk delete support
88e60f8 Add bulk delete actions for pipeline stages
```

The order is the point. The first commit adds `- [ ] Bulk delete actions for pipeline stages` to `VISION.md`. The second commit implements the feature and flips the checkbox. Strip the first commit and you can still *read* the implementation; you can't *read it as a decision*. The vision update is the decision; the implementation is the execution.

### The durable shape (pseudocode)

```
# Pseudocode: living gating artifact + skill that gates work on it

# The artifact — short, checklist-shaped, in-repo
file: VISION.md
content:
    # area headings
    ## Pipeline
    - [ ] Bulk delete actions for pipeline stages
    - [x] Per-stage filtering
    # ... more areas, ~50-100 items total at steady state

# The skill — refuses if no matching item exists
function shipFeature(arguments):
    items = readChecklist(VISION_MD)
    match = findUnchecked(items, matching=arguments)
    if not match:
        error("No matching unchecked item on VISION.md. "
              "Add the item first (in its own commit), then re-run.")
    markChecked(match)
    stageAllChanges()
    commitMessage = composeMessage(work=arguments, item=match)
    commit(commitMessage)
    push()
```

### The implementation

The artifact lives at the repo root as plain Markdown:

```markdown
# lead-crm — Vision

## Pipeline

- [x] Per-stage filtering
- [x] Bulk delete actions for pipeline stages
- [ ] Saved filter views
- [ ] Stage-transition audit log

## Reports

- [x] Weekly conversion dashboard
- [ ] Per-rep activity report
```

The skill is six lines of instruction. As of writing, `.claude/skills/ship-feature/SKILL.md` is 646 bytes. The mechanical version:

```markdown
---
name: ship-feature
description: Flip a VISION.md checkbox and ship the feature in one commit
---

Read VISION.md. Find the first unchecked item that matches $ARGUMENTS.
If none, stop and ask the user to add one (in its own commit) or pick from existing.
Flip the matching `- [ ]` to `- [x]`. Stage all changed files.
Commit with a message that describes the work (not the checkbox flip).
Push.
```

The skill enforces *that an item exists at commit time*. It does NOT enforce *that the item existed before the work*. The human is on the hook for the top half — the discipline of adding items in their own commit, then implementing. In lead-crm the discipline holds because the human-who-set-up-the-pattern is the same human-who-cooperates-with-it. In a larger team, the weakness would surface fast.

A Ruby implementation of the same shape for teams that want it programmable rather than skill-prompted:

```ruby
# lib/tasks/ship_feature.rake
namespace :ship do
  desc "Flip a VISION.md checkbox and commit the feature."
  task :feature, [:scope] do |_, args|
    vision = File.read("VISION.md")
    pattern = /^- \[ \] .*#{Regexp.escape(args.scope)}.*$/i
    match = vision[pattern]
    abort "No matching unchecked item. Add to VISION.md in its own commit." unless match

    updated = vision.sub(pattern, match.sub("- [ ]", "- [x]"))
    File.write("VISION.md", updated)

    sh "git add -A"
    sh "git commit -m #{args.scope.inspect}"
    sh "git push"
  end
end
```

### What survives the format changing

The principle: **one living artifact, owned by the human, read and updated by the agent at every feature commit.** Markdown is incidental — YAML, JSON, a flat text file would all work. The discipline is the gating-vs-documenting distinction and the *cheap-enough-to-read-fully-on-every-loop* property. Trackers (Linear, Jira) defeat the *file in the same repo the agent is editing* property that makes the pattern cheap.

## Primitive 3 — Role-separated playbook (markdown-toolkit `.claude/ORCHESTRATION.md`)

**Claim:** *every wave of agentic work produces a named file artifact; if there's no file, the role wasn't separated, just narrated.*

**Source incident:** `markdown-toolkit-role-separation-solo.md`. A solo developer deployed a multi-agent orchestration playbook against a Chrome-extension bug. The expected yield was speedup. The actual yield was *discipline the headcount didn't justify*: the architect produced a 372-line root-cause document naming **three independent defects** when the same developer working solo would have shipped the fix for the most obvious one and called it done.

The timeline (2026-05-04 afternoon):
- 13:46–13:51 — `wxt.config.ts` initial feature work.
- 14:14 — Wave 1 fires. Both `.tickets/2026-05-04-bug-fixes-v1.md` (project-manager) and `.architecture/parser-fix-2026-05-04.md` (team-leader-architect) written in parallel.
- The architect's 372-line document names Defect A (header-row width mismatch), Defect B (greedy nested-block flattening), Defect C (`querySelectorAll('tr')` recursing into nested tables). Solo-mode would have caught C only.

### The durable shape (pseudocode)

```
# Pseudocode: role-separated waves with artifact-or-it-didn't-happen rule

# The playbook — three waves, each producing named artifacts
playbook:
  Wave 1 (parallel, no coordination):
    project-manager      -> .tickets/{date}-{scope}.md
    team-leader-architect -> .architecture/{date}-{scope}.md
    product-designer     -> .design/{date}-{scope}.md
  Wave 2 (serial):
    senior-engineer reads Wave 1 artifacts, implements, commits
  Wave 3 (parallel):
    qa-automation -> .qa/{date}-{scope}.test.ts
    code-reviewer  -> .reviews/{date}-{scope}.md

# The rule
function runWave(wave):
    artifacts = []
    for role in wave.roles:
        artifact = role.execute(brief)
        if not fileExists(artifact):
            error(f"Role {role.name} didn't produce {role.expectedArtifact}. "
                  "Artifact-or-it-didn't-happen — re-run the role.")
        artifacts.append(artifact)
    return artifacts

# Non-coordination is a feature
function wave1(brief):
    pm_future = async pm.execute(brief)           # both read same brief
    arch_future = async architect.execute(brief)  # neither sees the other
    return [await pm_future, await arch_future]   # converge after, not before
```

### The implementation

The playbook lives at `.claude/ORCHESTRATION.md`. Six role-specialized sub-agents are registered in `.claude/agents/`:

```
.claude/
├── ORCHESTRATION.md         # the playbook — three waves, artifact rules
└── agents/
    ├── project-manager.md
    ├── team-leader-architect.md
    ├── product-designer.md
    ├── senior-engineer.md
    ├── qa-automation.md
    └── code-reviewer.md
```

Each role's agent definition includes its expected artifact path and the *brief* it operates from. A minimal `team-leader-architect.md`:

```markdown
---
name: team-leader-architect
description: Produces the root-cause architecture document for non-trivial work
tools: Read, Grep, Glob, WebFetch
---

You are the architect role in the orchestration playbook.

Your output is ALWAYS a file at `.architecture/{date}-{scope}.md`.
Do not narrate. Do not summarize verbally. If the file doesn't exist
when you finish, the role didn't happen.

Trace the brief end-to-end. Name every independent defect or design
decision, not just the most obvious. Solo-mode bugs ship one fix;
multi-agent bugs ship three. Your job is the taxonomy, not the patch.
```

The playbook's Wave 1 rule that produces *triangulation*: PM and Architect fire concurrently, **neither reads the other's draft until both are filed**. The non-coordination is the feature, not a bug. Two roles, same brief, independent reads — produce *combined coverage*, not redundant work.

The playbook's Wave 2 rule that prevents the worst orchestration failure mode (`canvas-mcp-two-shadow-apis.md`): **only one engineer reads the Wave 1 artifacts and implements**. Critically, *not two engineers in parallel*. Two parallel engineers touching the same file ship a permanent dual API; one engineer serial avoids the convergence-conflict.

### The artifact directory after a wave

After Wave 1 fires for the budget-sheet bug:

```
.tickets/2026-05-04-bug-fixes-v1.md          # PM's scoped ticket
.architecture/parser-fix-2026-05-04.md       # Architect's 372-line root-cause doc
```

After Wave 3 finishes:

```
.tickets/2026-05-04-bug-fixes-v1.md
.architecture/parser-fix-2026-05-04.md
.qa/2026-05-04-parser-budget-sheet.test.ts       # QA's fixtures for all three defects
.reviews/2026-05-04-bug-fixes-v1.md          # Reviewer's sign-off against the full taxonomy
```

The artifacts are the discipline. The roles are the scaffolding that demands they exist. Strip any one and the taxonomy collapses back to the symptom-fix.

### What survives the role count changing

The principle: **role separation forces artifact production; artifact production forces commitment to claims; commitment to claims catches things that uncommitted thinking misses.** Six roles is the right number for `markdown-toolkit`; a smaller project might need three (PM, engineer, reviewer); a larger one might need eight. The role count is per-project. The artifact-or-it-didn't-happen rule isn't.

## Primitive 4 — Operator-atrophy discipline (magmalabs-assistant)

**Claim:** *the operator's recognition reflex on a delegated routine has been deliberately kept warm, or the operator has consciously chosen to forget how to do it.*

**Source incident:** `magmalabs-delegated-skill-decay.md`. A senior operator delegated repetitive judgment work (daily summaries, message drafting, meeting prep, standup aggregation) to nine Claude Code skills and four sub-agents. Three skills decayed silently:

- **Slack formatting muscle.** The `/follow-up-alerts` skill sent a message with a blockquote containing nested bullets. Slack ate the three critical questions inside the blockquote. The teammate received what looked like an empty message.
- **Email triage instinct.** The operator forwarded a contact's feedback based on a truncated email preview, missing the actual feedback in the body. The agent had read the full email correctly; the operator had stopped scrolling past the agent's summary header.
- **Knowing whose work is whose.** The `/standup-notes` skill auto-aggregated everyone's items. The operator caught themselves reporting other directors' wins in the CEO summary multiple times before adding an "only my items" filter.

In all three cases, the agent did its job. The decayed muscle wasn't in the system; it was in the operator. Standard observability would have caught none of them.

### The durable shape (pseudocode)

```
# Pseudocode: three discipline moves for operator-side atrophy

# 1. Rules-with-why encoded into memory, surviving session boundaries
function encodeRule(rule, why):
    memory.append(MEMORY_MD,
        rule=rule,
        why=why,  # the load-bearing field — what makes the rule survive edge cases
    )

# Example:
encodeRule(
    rule="Don't include other directors' items in the CEO summary.",
    why="Each director reports their own. Auto-aggregation defeats attribution."
)

# 2. Approval gates move INTO the skill, not into operator intention
function defineSkill(name, definition):
    # WRONG — gate lives in operator's head
    operator.believes("Always show me drafts before sending")

    # RIGHT — gate lives in the skill's definition
    skill = {
        name: name,
        ...definition,
        gates: [
            condition: "draft contains blockquote or nested formatting",
            action: "preview before send, not after",
        ],
    }

# 3. Deliberate practice for skills the operator wants to retain ability in
function delegationDecision(routine):
    # Conscious decision per skill, not a default
    if operatorWantsToRetainAbility(routine):
        schedule(routine, cadence="manual_run_weekly")
    else:
        delegate(routine)  # accept that the muscle will atrophy
```

### The implementation

**Move 1 — Rules-with-why in memory.** The Claude Code harness has a per-project memory layer. Rules go in with their reasoning, so future revisions of the skill (or sibling skills that aggregate from the same source) apply the same logic:

```markdown
---
name: ceo-summary-attribution
description: Only the operator's own items appear in the CEO summary.
metadata:
  type: feedback
---

Don't include other directors' items in the CEO summary draft. Each director
reports their own; auto-aggregation defeats attribution. Filter the standup
transcript to items where the speaker is the operator (the operator's name
is in the harness's user profile).

**Why:** the operator caught themselves reporting other directors' wins
multiple times before adding this filter. Recognition reflex had decayed
because the skill served a one-item plate that didn't prompt "is this mine
to report?"
**How to apply:** runs at every `/standup-notes` invocation; CEO summary
draft must filter on speaker before output.
```

**Move 2 — Approval gates in the skill, not in intention.** A skill that has durable side effects (sending a message, posting to a channel, scheduling on a calendar) carries its own consent gate inside the skill definition:

```markdown
---
name: follow-up-alerts
description: Draft and send follow-up messages to teammates about quiet client threads
---

## Gates

Before sending any draft that contains:
- A blockquote
- Nested bullets inside another structural element
- More than 200 words

Surface a preview to the operator and wait for explicit approval. Do not
auto-send. The approval is operator-consent, not skill-decision.

## Body

... (draft generation logic)
```

The gate now lives where the action happens. The operator's intention ("always show me drafts") was getting overrun by the skill's speed; the gate-in-skill closes that gap structurally.

**Move 3 — Deliberate practice as a per-skill decision.** The harder discipline. For each skill, decide: *is this a routine the operator wants to retain ability in?* If yes, schedule deliberate manual runs of the underlying workflow on some cadence; if no, accept the trade.

```markdown
---
name: skill-retention-policy
description: Per-skill decision on whether to keep the underlying muscle warm
metadata:
  type: project
---

| Skill                | Retain ability? | Practice cadence                      |
|----------------------|-----------------|---------------------------------------|
| `/cockpit`           | Yes             | Manual rebuild once a month           |
| `/wrap-up`           | Yes             | Manual CEO summary once a week        |
| `/follow-up-alerts`  | No              | Operator is happy to forget           |
| `/standup-notes`     | Yes             | Read full transcript once a week      |
| `/grain-search`      | No              | Operator never had this muscle        |
| `/deal-status`       | Yes             | Manual rebuild before quarterly board |
```

**Why:** the CEO-summary editor reflex stayed intact because the operator edited every draft. The email-triage reflex didn't, because the operator stopped opening full threads. The discipline isn't *use the skills less*; it's *consciously decide which underlying muscles to keep warm.*

### The honest cost

Delegation means accepting you will be wrong occasionally and won't notice. The teammate's "what questions?" is the price of the trade. The right question isn't *how do I prevent that?* It's *is the rate of those incidents lower than the rate of mistakes I'd make doing it all myself, exhausted, at 11pm?* For most operators using agents to scale judgment work, the answer is yes — but the trade has to be conscious, and the cost has to be named.

### What survives the harness changing

The principle: **what you delegate is what decays.** The specific memory file format, the skill-definition syntax, the consent-gate UI — all dated. The three discipline moves (rules-with-why, gates-in-skill, deliberate-practice) survive. Recognition is parasitic on prior recall; if the operator hasn't done the recall version often enough to know what right looks like, they'll be a fast wrong instead of a slow right.

## Porting to other contexts

The four primitives port unchanged across stacks because they're git-and-Markdown-shaped, not code-shaped:

- **Cross-vendor review.** Any agent-heavy workflow with two or more vendors available. Branch-prefix discipline costs nothing.
- **Living gating artifact.** Any project where the team is small enough that a Markdown checklist captures the active state (the heuristic threshold: ~3 contributors and ~30 active backlog items).
- **Role-separated playbook.** Any non-trivial work with layered root causes. The artifact-or-it-didn't-happen rule is stack-agnostic.
- **Operator-atrophy discipline.** Any role where the operator has delegated judgment-bearing routines to an agent and the failure mode is invisible from the agent's output.

The shape also generalizes outside agentic work:

- **Solo developers reviewing their own PRs** with a *written* self-review checklist (Primitive 3).
- **Tech leads with a small reporting team** using a shared `ROADMAP.md` instead of a tracker (Primitive 2).
- **Lawyers with contract-review agents** who must keep reading whole contracts on some cadence (Primitive 4).
- **Cross-vendor static analysis** as the non-agent version of Primitive 1.

## Counter-examples — when these primitives don't apply

- **Teams large enough to need real backlog management.** Once a team has more than ~3 contributors and ~30 active items, the lightweight Markdown-checklist pattern degrades. Real trackers exist for a reason.
- **Trivial work where the symptom-fix is the right fix.** Typo fixes, lint cleanups, single-file edits don't justify role-separated playbooks. Don't deploy six sub-agents to fix a typo.
- **Bias-driven misses that don't matter much.** A prototype or a one-off script doesn't pay back the cost of cross-vendor review.
- **Routines the operator is happy to forget.** Not every skill needs deliberate practice. The decision has to be *conscious*, but the answer can legitimately be "let this one decay."
- **Compliance contexts where the audit trail must survive repo deletion.** `git log` + `VISION.md` is reconstructible only if the repo exists. For regulated work, a separate audit-grade tracker is the safer move.
- **Multi-team coordination across repos.** `VISION.md` is per-repo. Coordinating two repos that share a single roadmap needs a cross-repo artifact.

The signal across all four: *am I using these primitives at the scale they were designed for?* If yes, they earn their place. If no, reach for the tooling that fits the actual scale.

## What survives the vendors, harnesses, and skill systems changing

The principle: *the structure is the team.* The specific vendors (`claude/`, `codex/`), the specific harness affordances (`/ship-feature` syntax, consent-gate YAML, sub-agent registration), the specific Markdown formats — all dated. The four primitives' shapes are durable:

1. **Distinct bias surfaces in the review chain, audit-trail in branch names.**
2. **One living gating artifact, human-owned, agent-maintained.**
3. **Role-separated waves with artifact-or-it-didn't-happen as a hard rule.**
4. **Conscious delegation decisions with rules-with-why, gates-in-skill, and deliberate-practice cadences.**

Every agentic-team system will have a version of each. The discipline is naming the structural primitives that the team's work runs through, and engineering them deliberately instead of stumbling into them.

## Cross-references

- Ch. 11 (Team Structure for Agentic Systems) — full chapter treatment.
- Ch. 7 (Orchestration Patterns) — the *role-separated waves* primitive sits alongside the chapter's existing fan-out shapes.
- Ch. 8 (Observability for Probabilistic Systems) — the branch-name provenance and the operator-atrophy gap are observability-discipline arguments; see [`observability-across-stacks.md`](observability-across-stacks.md) for the verification-record version of the same shape.
- Ch. 10 (When to Trust the Output) — the gates-in-skill move is a runtime trust mechanism; see [`runtime-trust-patterns.md`](runtime-trust-patterns.md).
- `war-stories/leads-crm-vision-as-agent-memory.md`, `leads-crm-codex-pr1.md`, `markdown-toolkit-role-separation-solo.md`, `magmalabs-delegated-skill-decay.md` — the four source incidents.
- [`../B-sdks/harness-affordances.md`](../B-sdks/harness-affordances.md) — the Claude Code harness's `disable-model-invocation` flag, consent-gate mechanics, and auto-memory layer the recipes lean on.
