# Harness Affordances — Skill Flags, Auto-Memory, Consent Gates

**Source asides:** Ch. 9 (agent vs. script vs. human), Ch. 11 (building the team).
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026, against the Claude Code harness used through the manuscript.

## What the body says

Ch. 9 and Ch. 11 lean on a small set of harness affordances to make their patterns concrete: per-skill config flags, an auto-memory system the agent reads and writes across sessions, and consent gates that wrap irreversible actions. The body keeps these dated specifics out of prose; this entry names them.

## The dated specifics

### `disable-model-invocation` per-skill flag

The Claude Code harness supports a per-skill configuration that disables the model from invoking the skill directly — the skill is then only triggerable by the user via the slash-command UI. The flag's name as of writing is `disable-model-invocation`.

**Use case:** Ch. 9's *match the executor to the work* discipline reaches for this flag when a skill has durable side effects (sending email, posting to Slack, committing code) that should be user-invoked, never auto-fired by the agent loop.

**TODO (subsequent pass):** verify exact YAML key, current default, and whether the flag has been renamed or split into finer-grained variants.

### Auto-memory layer

The harness has a per-project auto-memory system. The agent writes facts into a memory store keyed to the working directory; subsequent sessions in the same directory load those facts on startup. Memory has structured fields (`name`, `description`, `metadata.type`) and a slug-based filing system.

**Use case:** Ch. 11's *the structure is the team* references the memory layer as one of the structural primitives — but only when paired with the verification habit from Ch. 6 (*plausible drift*). Memory writes a fact at a point in time; the world moves; verification on read is what keeps recall honest.

**TODO (subsequent pass):** name the exact memory schema, the filing conventions the harness enforces, and any per-memory-type behaviors (e.g., `feedback` vs. `project` vs. `reference`).

### Consent gates

The harness offers a consent-gate mechanism: certain tool calls (file writes, shell commands, network requests) can require explicit user approval before they fire. The approval lives in `settings.json` (allowed-list, ask-list, deny-list) and is per-tool, per-project.

**Use case:** Ch. 10's *treat approval as a side-effect, not the terminal state* uses consent gates as the canonical Gate mechanism in the trust/verify/gate framework. The gate exists; the discipline is naming what should be on the ask-list vs. the allowed-list.

**TODO (subsequent pass):** document the exact `settings.json` schema, where project vs. user settings merge, and how the `Permissions` UI surfaces decisions.

## What survives the harness changing

Every harness will have *some* version of these affordances — a way to keep certain actions user-invoked, a way to persist context across sessions, a way to gate irreversible actions. The names will rotate. The patterns won't.

The principle from Ch. 9: *match the executor to the work, anchored to cost-of-being-wrong.* The flag, the memory layer, and the consent gate are *implementations* of that principle in one harness. Future harnesses will have different implementations of the same idea.

## Cross-references

- Ch. 9 (Agent vs. Script vs. Human) — full chapter treatment of executor matching.
- Ch. 10 (When to Trust the Output) — gate/verify/trust framework at runtime.
- Ch. 11 (Building the Team Around Agentic Systems) — structural primitives the team's discipline rides on.
- [`harness-limits.md`](harness-limits.md) — the harness's *constraints* (delegation depth, MCP lifecycle), as opposed to its affordances.
