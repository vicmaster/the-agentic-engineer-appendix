# Harness Affordances — Skills, Sub-agents, Memory, Permissions, Hooks

**Source asides:** Ch. 9 (agent vs. script vs. human), Ch. 11 (building the team).
**Source incidents:** `war-stories/coide-ship-feature.md`, `magmalabs-delegated-skill-decay.md`, `leads-crm-vision-as-agent-memory.md`.
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026, against Claude Code (the harness used through the manuscript).

## What the body says

Ch. 9 and Ch. 11 lean on a small set of harness affordances to make their patterns concrete: per-skill config flags, sub-agent registration, an auto-memory system the agent reads and writes across sessions, consent gates that wrap irreversible actions, and hooks that fire on tool calls. The body keeps these dated specifics out of prose; this entry names them.

This is the *affordances* companion to [`harness-limits.md`](harness-limits.md). The limits are what the harness won't do; the affordances are what it offers. The pre-flight discipline is to read both before drafting.

## Skills

A *skill* is a slash-command-invokable workflow defined as a Markdown file with frontmatter. Skills live in `.claude/skills/<skill-name>/SKILL.md` (project-scoped) or `~/.claude/skills/<skill-name>/SKILL.md` (user-scoped).

### Skill definition shape

```markdown
---
name: ship-feature
description: Flip a VISION.md checkbox and ship the feature in one commit
disable-model-invocation: true
allowed-tools:
  - Read
  - Edit
  - Bash(git add:*)
  - Bash(git commit:*)
  - Bash(git push:*)
---

Read VISION.md. Find the first unchecked item matching $ARGUMENTS.
If none, stop and ask the user to add one (in its own commit) or pick existing.
Flip the matching `- [ ]` to `- [x]`. Stage all changed files.
Commit with a message that describes the work (not the checkbox flip). Push.
```

### Skill frontmatter keys (as of Q2 2026)

| Key | Type | Purpose |
|---|---|---|
| `name` | string | The slash-command name. Required. Should match the directory. |
| `description` | string | One-line summary shown in the skill picker. |
| `disable-model-invocation` | bool | If `true`, the skill can only be invoked by the user via `/<name>`. The model cannot call it via the `Skill` tool. |
| `allowed-tools` | list | Tools the skill is allowed to use. Pattern-matched (`Bash(git push:*)` allows `git push` invocations but not arbitrary bash). |
| `model` | string | Override the session model for this skill's execution (e.g. `claude-haiku-4-5` for cheaper skill runs). |
| `argument-hint` | string | Hint shown in the slash-command UI for what `$ARGUMENTS` should contain. |

### `disable-model-invocation` — the durable-side-effect flag

From `coide-ship-feature.md` — the canonical anchor for this flag. A skill with autonomous side effects (commits, pushes, sends, deletes) should be user-invoked only. Setting `disable-model-invocation: true` is the structural gate; the discipline pairing (rules-with-why in memory) is the behavioral one. Either alone leaks. Both together is the right shape for any irreversible side effect.

**Rule of thumb:** if the skill name's verb is one of `ship`, `send`, `delete`, `deploy`, `purchase`, `commit`, `push`, `publish`, set the flag.

### `allowed-tools` patterns

Tool restrictions support glob-style patterns. Common shapes:

```yaml
allowed-tools:
  - Read                     # full Read tool access
  - Edit                     # full Edit tool access
  - Bash(npm:*)              # only `npm <anything>`
  - Bash(npm test)           # only the exact command `npm test`
  - Bash(git:*)              # any `git` invocation
  - Bash(git push:*)         # `git push` to any remote
  - Glob(**/*.ts)            # only matching TypeScript files
  - mcp__forge__*            # any tool from the `forge` MCP server
  - WebFetch(domain:github.com)  # WebFetch restricted to GitHub
```

The pattern is matched against the *full invocation*, not just the tool name. `Bash(rm:*)` won't match `Bash(sudo rm:*)` because the prefix is different.

### `$ARGUMENTS` and arg-style invocation

A skill invoked as `/ship-feature bulk delete actions` receives `$ARGUMENTS = "bulk delete actions"`. The skill body can pattern-match against `$ARGUMENTS`, search files, or refuse if the argument shape doesn't fit.

## Sub-agents

Sub-agents are defined the same way as skills but live in `.claude/agents/<agent-name>.md`. They're invoked by the `Agent` tool (only available in the top-level session — see [`harness-limits.md`](harness-limits.md)) and run in isolated contexts.

### Sub-agent definition shape

```markdown
---
name: rails-developer
description: Senior Rails engineer specialist
model: claude-sonnet-4-6
tools:
  - Read
  - Glob
  - Grep
  - Edit
  - Bash
---

You are the senior Rails engineer on this project. Follow Rails 8 conventions.
Read routes/schema/models before implementing. Run tests after every change.

Your output is code. Do not narrate the implementation; show it.
```

### Sub-agent frontmatter keys

| Key | Type | Purpose |
|---|---|---|
| `name` | string | The agent identifier. Used in `Agent` tool invocations. |
| `description` | string | Multi-line OK. The model sees this when deciding whether to invoke the agent. |
| `model` | string | The model the sub-agent uses. Cheap-model sub-agents for narrow tasks; expensive-model for judgment work. |
| `tools` | list | Tool access granted to the sub-agent. Same pattern syntax as skills. |

### Pattern: specialist + surface

From `leads-crm-vision-as-agent-memory.md` — lead-crm has one committed sub-agent (`rails-developer`) plus a wide product-level MCP tool surface for external agents. The orchestration shape isn't fan-out; it's *one specialist plus a fat tool surface*. The committed sub-agent does build-time work; the MCP surface serves arbitrary external agents at runtime.

**When to add a sub-agent:** the work has a stable, narrow specialization (Rails dev, code review, QA testing) and the savings of running it on a cheaper model pay back the configuration cost.

**When not to:** for one-off tasks. Sub-agents earn their place by being reused; a sub-agent invoked once is overhead.

## Auto-memory

Claude Code's per-project memory layer persists facts across sessions. Memory lives in `.claude/projects/<encoded-cwd>/memory/` and is loaded at session start.

### Memory file structure

```
.claude/projects/-Users-victor-Projects-myapp/memory/
├── MEMORY.md              # the index — one line per entry, loaded into context
└── <slug>.md              # the entry — frontmatter + body
```

`MEMORY.md` is the always-loaded index. Entries beyond ~200 lines get truncated, so it stays short. Each entry file is a separate document loaded on demand.

### Memory entry frontmatter

```markdown
---
name: feedback-testing-database
description: Integration tests must hit a real database, not mocks. Reason: prior incident where mock/prod divergence masked a broken migration.
metadata:
  type: feedback
---

Don't mock the database in integration tests.

**Why:** prior incident where mocked tests passed but the prod migration failed.
**How to apply:** any test marked `integration:` must hit the real PG; unit tests stay mock-friendly.
```

| Key | Type | Purpose |
|---|---|---|
| `name` | string | Kebab-case slug. Must match the filename (minus `.md`). |
| `description` | string | One-liner; surfaces in `MEMORY.md` index. |
| `metadata.type` | enum | `user`, `feedback`, `project`, `reference`. Determines lifecycle and use patterns. |

### Memory types

The four types from the auto-memory system:

- **`user`** — information about the human operator (role, goals, knowledge, preferences). Tailors agent behavior to the user.
- **`feedback`** — corrections and validations. *"Don't do X"* or *"yes, that approach was right."* Includes the *why* so future-you can judge edge cases.
- **`project`** — facts about ongoing work, deadlines, who-is-doing-what. Decays fast; expect to refresh.
- **`reference`** — pointers to external systems (Linear projects, Slack channels, dashboards). Stable but verify before recommending.

### The verification habit

From `coide-memory-drift.md` — recall is a starting point, not an answer. Memory was written at a point in time; the world moves; verification on read is what keeps recall honest. Before acting on a memory that names a specific entity (a file, function, flag), check the entity still exists and matches.

```
function actOnRecall(memory):
    namedEntities = extractEntities(memory.body)
    for entity in namedEntities:
        if not exists(entity):
            flag("Memory references entity that no longer exists")
            updateOrRemove(memory)
            return
        if not matches(entity, memory.body):
            flag("Memory's claim about entity is stale")
            updateOrRemove(memory)
            return
    proceed(memory)
```

## Permissions (consent gates)

The permission system lives in `settings.json` files at three levels (user, project, project-local), merged in that order.

### Settings file locations

```
~/.claude/settings.json                       # user-wide
<project>/.claude/settings.json               # project, committed
<project>/.claude/settings.local.json         # project, gitignored
```

The settings.local.json is for the operator's personal overrides; the committed settings.json is for project-wide defaults.

### Permission shape

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(npm test)",
      "mcp__forge__list_routines"
    ],
    "ask": [
      "Bash(git push:*)",
      "Bash(git commit:*)",
      "Edit"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(sudo:*)",
      "WebFetch(domain:*)"
    ]
  }
}
```

| Bucket | Behavior |
|---|---|
| `allow` | Tool fires without prompting the operator. |
| `ask` | Tool requires explicit operator approval each time. |
| `deny` | Tool refused; the model gets an error. |

Patterns match the full invocation, same as skill `allowed-tools`. More-specific patterns win over less-specific ones; `Bash(git push:*)` in `ask` beats `Bash(git:*)` in `allow`.

### Pattern: gate the irreversible

From `coide-ship-feature.md` and the team-primitives recipe. Anything with durable side effects should default to `ask`, not `allow`. The operator's intention (*always show me drafts before send*) is unreliable; the gate-in-settings is structural.

A reasonable starting set for `ask`:

```json
"ask": [
  "Bash(git push:*)",
  "Bash(git commit:*)",
  "Bash(git reset --hard:*)",
  "Bash(git rebase:*)",
  "Bash(rm:*)",
  "Bash(npm publish:*)",
  "Bash(heroku:*)",
  "Bash(railway:*)",
  "Bash(curl:*)",
  "WebFetch",
  "Edit"
]
```

Anything that touches production, mutates externally-visible state, or is hard to reverse goes here.

### Pattern: deny the unrecoverable

`deny` is for actions you don't want the agent to even *try*. The classic shape:

```json
"deny": [
  "Bash(rm -rf:*)",
  "Bash(sudo:*)",
  "Bash(curl * | bash)",
  "Bash(eval:*)"
]
```

These are the patterns where even a confirmation prompt is too low a bar — the operator is likely to confirm without reading.

## Hooks

Hooks let the operator configure shell commands that fire at specific lifecycle moments. Hook config lives in the same `settings.json`.

### Hook shape

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(git push:*)",
        "hooks": [
          { "type": "command", "command": "./bin/check-tests-pass" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          { "type": "command", "command": "./bin/format-file" }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "echo 'session ended' | terminal-notifier" }
        ]
      }
    ]
  }
}
```

### Hook events

| Event | When it fires | Common uses |
|---|---|---|
| `PreToolUse` | Before a tool call; can block if exit ≠ 0 | Gate-style: refuse git push if tests fail |
| `PostToolUse` | After a tool call completes | Format-on-save, lint-on-edit |
| `UserPromptSubmit` | Before the model sees the user's input | Inject context, normalize input |
| `Stop` | When the session ends | Notifications, log summarization |
| `Notification` | When the harness wants to notify the operator | Custom notification routing |

### Pattern: automated discipline

From `magmalabs-delegated-skill-decay.md` (Move 2 — *approval gates in the skill, not in intention*). When the operator wants something to happen on every tool call (formatting, linting, logging, alerting), the hook is the right place — it's the harness that executes the hook, not the model. The operator's intention doesn't have to overrule the model's speed.

**Rule of thumb:** if the operator finds themselves saying *"I always want X to happen when Y"*, it's a hook. Memory and skill-prompts are reminders the model may or may not honor; hooks are deterministic.

## MCP server registration

External capabilities (databases, APIs, custom tooling) integrate via MCP. Server registration lives in `~/.claude/mcp_servers.json` or per-project `.claude/mcp_servers.json`.

```json
{
  "mcpServers": {
    "forge": {
      "command": "node",
      "args": ["/path/to/forge-mcp/dist/index.js"],
      "env": {
        "FORGE_API_TOKEN": "${FORGE_API_TOKEN}"
      }
    },
    "leads-crm": {
      "url": "https://leads-crm.example.com/mcp",
      "transport": "http",
      "headers": {
        "Authorization": "Bearer ${LEADS_CRM_TOKEN}"
      }
    }
  }
}
```

The harness spawns stdio MCP servers per session (see the lifecycle limit in [`harness-limits.md`](harness-limits.md)). HTTP MCP servers are long-running and shared across sessions.

Tools exposed by an MCP server appear as `mcp__<server-name>__<tool-name>` (e.g. `mcp__forge__list_routines`). Use the prefix in `allowed-tools` and `permissions` patterns.

## What survives the harness changing

The principle: **every harness has its own version of these affordances.** The specific YAML keys, settings.json schema, hook event names, MCP registration shape — all rotate. The discipline survives:

- **Per-skill flags for durable side effects.** Every harness needs a way to keep certain skills user-invoked.
- **Memory with rules-and-why.** Every persistence layer needs the reasoning behind the rule, not just the rule.
- **Permission gates as structural, not behavioral.** Every harness needs `ask` for the irreversible.
- **Hooks as automated discipline.** Every harness needs deterministic shell-out for "I always want X."

The shapes recur. The names won't.

## Cross-references

- Ch. 9 (Agent vs. Script vs. Human) — `disable-model-invocation` and the executor-matching framework.
- Ch. 10 (When to Trust the Output) — permissions as the runtime gate mechanism.
- Ch. 11 (Team Structure for Agentic Systems) — memory + skills + sub-agents as the structural primitives.
- `war-stories/coide-ship-feature.md` — `disable-model-invocation` anchor.
- `war-stories/magmalabs-delegated-skill-decay.md` — rules-with-why in memory.
- `war-stories/leads-crm-vision-as-agent-memory.md` — specialist + surface pattern.
- [`harness-limits.md`](harness-limits.md) — the harness's *constraints* (delegation depth, MCP lifecycle), as opposed to its affordances.
- [`../C-recipes/team-primitives-cross-vendor.md`](../C-recipes/team-primitives-cross-vendor.md) — all four primitives use the affordances above.
