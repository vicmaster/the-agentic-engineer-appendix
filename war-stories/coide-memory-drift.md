# coide / Memory That Drifted From Code

## Date / Version Context

- **Date:** Caught 2026-05-08 during a working session in coide. The drift had existed since the architectural change — `node-pty` to `child_process.spawn` — landed at some earlier commit. The exact commit isn't load-bearing for this story; the load-bearing fact is that *the change happened in code and didn't propagate to memory*.
- **Project:** coide v0.26.x — desktop GUI wrapper around the Claude Code CLI. The relevant surface is *how coide spawns and talks to the `claude` binary*. Originally that was via `node-pty` (a pseudo-terminal). It is now via `child_process.spawn` with stream-json input/output (a structured JSON protocol over stdio).
- **Surface for this story:** the auto-memory system at `~/.claude/projects/<project-slug>/memory/`. A folder of Markdown files, one per memory, indexed by `MEMORY.md`. Memories can describe user preferences, project state, feedback, or external references. They persist across sessions and are loaded into the agent's context whenever the project is opened.
- **Glossary, used in this writeup:** *Memory* = a persistent fact stored in the auto-memory directory, available to the agent in future sessions. *Recall* = the agent retrieving and using a memory's content. *Drift* = the gap between what memory says and what the code currently is. *Verification* = the act of checking memory's claim against the current state of the code or system. *Snapshot* = a memory understood as a point-in-time observation, not a live source of truth.

## What Was Being Attempted

Use the agent's memory the way it was designed to be used: as a substrate for cross-session continuity. A previous session had captured an architectural fact about coide — *coide spawns the Claude Code CLI via `node-pty`* — and saved it as a memory. Future sessions could pick up that fact without re-reading the codebase, which is exactly the kind of knowledge accumulation memory is for.

In a working session on 2026-05-08, the architecture of coide's CLI subprocess came up as relevant context. The agent recalled the memory. The memory said `node-pty`.

The code did not.

## What Went Wrong

The architecture had changed. coide had moved from `node-pty` to `child_process.spawn` with stream-json as the input/output protocol. The change was meaningful — different control-flow model, different lifecycle semantics, different testing surface, different failure modes. Anything an agent might say about coide's CLI handling depended on which of the two architectures was in play.

If the recall had been treated as live state — *memory says `node-pty`, therefore I will reason about the code as a `node-pty` consumer* — every architectural recommendation downstream would have been wrong. Wrong in the load-bearing way: not contradicted by the code on read, but built on a foundation the code had moved away from. The recommendations would have looked confident and been quietly off.

Examples of what could have shipped wrong:

- *"The PTY's `resize()` call should be propagated when the host window changes."* — coide doesn't have a PTY anymore; the resize concern doesn't apply.
- *"Use the `data` event handler to capture child output."* — `node-pty` and `child_process.spawn` both have `data` events but with different framing semantics; advice about parsing that data depends on which.
- *"The terminal's color codes will need to be stripped before display."* — `node-pty` emits ANSI; stream-json emits structured JSON. The stripping concern is irrelevant in the new architecture.

None of these are corner cases. They're the kind of observations a senior engineer would make when reviewing the subsystem. They would have all been wrong.

The bug isn't *memory was wrong*. Memory is allowed to be wrong; that's a property of any persistent claim across time. The bug is *acting on memory without verifying it against the current code*. The first move makes memory into a snapshot; the second move makes it into a source of truth. The second move is what produces wrong architectural advice with confidence.

## How It Was Discovered

While reading the memory in the course of session work, the recall surfaced the `node-pty` claim. The verification habit kicked in — *check the current code before recommending anything based on this* — and a quick read of the spawn path showed `child_process.spawn` with stream-json wiring. The memory was wrong by an architectural generation.

This is the load-bearing detail of the story: the discovery channel was the verification habit, not a downstream failure. If the habit hadn't been there, the agent would have produced confident advice about a `node-pty` architecture that no longer existed, and the discovery would have happened later — possibly through a confused human, possibly through a recommendation that didn't match the code, possibly through code being written against the wrong abstractions before someone noticed.

The verification cost was small. A grep, a file read, ten seconds of attention. The cost of skipping it is unbounded because the failure surfaces downstream of any number of decisions made on the bad foundation.

## What Fixed It

Two-part fix, both small:

1. **Update the memory.** The claim about `node-pty` was rewritten to reflect `child_process.spawn` with stream-json. The new memory is correct *as of 2026-05-08* and will drift again the next time the architecture changes. That's not a defect; it's a property.
2. **Reaffirm the verification habit.** The session-level habit — *recall is a snapshot, verify before acting* — was already part of the agent's working pattern. The incident reinforced it. There's nothing structural to "fix" beyond keeping the discipline.

What didn't get attempted: making memory automatically refresh from code. There's no clean mechanism for that, and there shouldn't be — memory's value is *cross-session persistence*, which depends on it being a stored fact, not a live derivation. The trade-off is intrinsic. You either have memory (which can drift) or you don't (which loses cross-session continuity). The fix is the discipline, not the mechanism.

## The Durable Lesson

Memory is a point-in-time snapshot, not live state. Verification must be paired with recall — especially for memories that name specific files, functions, flags, or architectural choices.

The lesson sounds tautological stated this way. *Of course* memory can be stale. Everyone using a persistent-memory system intuitively knows that. The reason it's worth a chapter-length writeup is that the *discipline* — verifying every recall before acting on it — is the part that doesn't propagate intuitively. Memory feels declarative. The agent recalls something, the recall feels factual, the temptation to act on the recall is large. Verification is a tax on that temptation, and the tax has to be paid every time.

> **Heuristic.** Any memory that names a specific file, function, flag, or architectural choice is a claim about the state of the code *at the time the memory was written*. Treat the recall as evidence, not as truth. If the user is about to act on a recommendation that depends on the recall, verify the named entity exists and behaves as the memory describes. The verification is cheap; the cost of skipping it is unbounded.

The deeper observation: this is the *context* version of a broader failure family. Other members:

- **`forge-max-tokens.md` (silent truncation):** the LLM's output is a snapshot of what it produced this turn; treating it as complete output without verifying `stop_reason` is the analog.
- **`forge-layered-status.md` (the function returned ≠ the work completed):** a status field is a snapshot of the routing layer's state; treating it as the execution layer's state is the analog.
- **`leads-crm-org-scoping.md` (UI-layer invariants leaking):** the UI's enforcement is a snapshot of the rule at the controller boundary; treating it as the system-wide invariant is the analog.

All four bugs share the shape: *a value that's true in one context, treated as true in a wider context where it isn't*. Memory drift is the version where the *time dimension* is the context. The memory was true when it was written. It's not necessarily true now. Acting on it without re-checking the time dimension is the bug.

The shape generalizes well past memory:

- **Cached architectural claims in CLAUDE.md.** A repo-level config file describing "the system uses Postgres with the `pg` adapter" is a memory in a different store. If the system migrates to Trilogy, the file is wrong and any agent reading it acts on stale info.
- **Recorded conversation summaries.** A "context summary" produced by compaction is a snapshot of what mattered at compaction time. The session keeps moving; the summary doesn't.
- **External references in agent prompts.** A skill that says "check the dashboard at grafana.example.com/d/api-latency" is fine until the dashboard's URL changes; the skill is wrong, the agent acts on it, the verification step is the only thing that catches the staleness.

In every case the rule is the same: *recall is a starting point, not an answer*. Verification is what turns recall into truth.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *memories that name specific entities the code can change*. It doesn't apply to:

- **Memories about user preferences.** "User prefers terse responses" doesn't drift the way "the code uses `node-pty`" does. The user's preference can change, but it's not the kind of thing that gets contradicted by reading source files. Verify when in doubt, but don't burn cycles re-checking preference memories on every recall.
- **Memories about durable, cross-session facts.** "User's email is X" is unlikely to drift; verifying it against source on every recall is a tax with no payback. The lesson is about memories *whose subject is the code*, not memories whose subject is the user or the world.
- **Cases where the cost of acting on stale memory is small.** A memory that says "this project uses TypeScript strict mode" is wrong in one direction (project moved to non-strict) only if someone bothered to change it; the cost of acting on the stale claim is low because most TS projects converge on similar conventions. Verify when stakes are high; skip when stakes are low.
- **One-shot memory uses where the user will catch errors immediately.** If the recall is part of a chat reply the user reads in real-time, the user is the verification. The lesson matters most for *agent-internal* uses of memory where no human is watching the recall happen.

The signal: *if this recall is wrong, what happens?* If the answer is "the agent makes a recommendation downstream of the bad fact and someone acts on it," verify. If the answer is "the user reads it and corrects me," skip the verification.

## What This Story Is *Not* Evidence For

- **Not evidence that memory is broken.** Memory works as designed. The drift is a property of any persistent claim across time, not a defect of the memory system. The fix is in the *use* of memory, not in the system.
- **Not evidence that memory should be auto-refreshed.** Auto-refresh would make memory live state, which would lose its cross-session value. The trade-off is intrinsic; the discipline is the right level of fix.
- **Not evidence that all memory recalls need verification.** User-preference memories rarely drift. World-facts about durable entities (a user's name, a company's domain) rarely drift. The lesson is about *code-naming* memories — those that mention specific files, functions, flags, or architectural choices.
- **Not evidence that this is unique to coide or to Claude Code's auto-memory system.** Any system with persistent agent memory has this property. The lesson generalizes to CLAUDE.md, agent prompts, skill descriptions, cached external references, conversation summaries, and any other store of facts the agent reads back later. The discipline scales with the surface.
