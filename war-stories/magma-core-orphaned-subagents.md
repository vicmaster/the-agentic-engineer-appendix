# magma-core / The Five-Hour Ghost (Orphaned Sub-agents)

## Date / Version Context

- **Date:** 2026-07-06, during a magma-core pull-request review session. Victor was "three PRs deep"; PR #19 (`013-remove-from-funnel`) merged that day (review commit `e459441` at 11:23, merge `0e10d32` at 19:48). The ghost sub-agents were noticed later the same day.
- **Project:** magma-core — a MagmaLabs internal platform (hiring/recruiting module: candidate funnels, BambooHR sync, screening).
- **Harness (dated context, 2026):** Claude Code, driven from Victor's desktop. The review pattern is a top-level session that fans out sub-agents via the `Agent` tool, each auditing one dimension of the diff; background tasks run in a panel; `TaskStop` stops a running task by ID. These are current-harness specifics — the lesson below is harness-agnostic.
- **The two ghost dimensions map to real, standing magma-core review concerns**, which is why the sub-agents were named for them:
  - *"Investigate system test regression"* — magma-core has a live history of flaky-system-test work (e.g. `ea26505` *test: stabilize flaky system tests (serial + no animations)*, MagmaLabs' Chief of People & Culture, 2026-06-18, PR #12).
  - *"Investigate design-first compliance"* — magma-core enforces a design-first gate (`033c8f8` *feat: wire design-first enforcement gate (Constitution VII v1.1.0)*, Victor, 2026-06-11, PR #4).
  - Note: these commits predate PR #19; they are the *dimensions* the audit was checking against, not commits inside the reviewed PR.
- **Glossary, used in this writeup:** *Sub-agent* = a Claude Code agent spawned by a session via the `Agent` tool. *Orphan / ghost* = a sub-agent still running after the parent that spawned it has returned. *Reaper* = the thing (in an OS, `init`) that adopts and cleans up orphaned children. *Lifecycle contract* = the self-propagating teardown instruction added to every spawned sub-agent's prompt (see *What Fixed It*).

## What Was Being Attempted

A routine review. For a big PR, Victor has the top-level session fan out a handful of sub-agents in parallel, each auditing a different dimension — security in one, correctness in another, tests, design, the rest. It works: on this very PR, the multi-agent pass caught a real CI regression that a single reviewer would likely have rationalized away.

The review was posted. The PR was merged. Everyone moved on to something else.

## What Went Wrong

Hours later, the session was blinking. The background-tasks panel showed two sub-agents *still running*:

- *Investigate system test regression* — **5 hours, 24 minutes**
- *Investigate design-first compliance* — **5 hours, 6 minutes**

Between them, close to **a hundred thousand tokens**, spent on work whose answer had already been read, incorporated, shipped, and forgotten hours earlier. Ghosts, dutifully haunting a house everyone had left.

Here is the actual shape. The fan-out spawned two sub-agents. One of them, reasonably, parallelized its own slice and spawned two more. Then the parent finished, handed its findings up the chain, and never told its children to stop — it didn't treat their lifecycle as its responsibility. So they orphaned: running, billing, invisible, one level below anyone who was watching.

It was not the first time. The Saturday before, on a different project (deliberately unnamed here), a single run spawned **106 sub-agents to do one web search and burned ~3.7M tokens** — for a question answerable with a handful of direct searches. The discovery channel is always the same: an accident. A blinking cursor, a number that's too big, a bill that doesn't add up.

## How It Was Discovered

By accident, and only by accident. The tooling raised no alarm. There was no notification, no runtime error, no cost ceiling that tripped. The signal was a blinking session and a human who happened to open the panel. That is the observability half of the story: **in an agentic system the machine is fast, tireless, and unaccountable by default.** Nobody sends you a notice that says "you left the lights on"; you just, eventually, find them on.

Looking for a switch to flip: there wasn't one. The lifecycle hooks in the tooling could tell you when an agent *finished* but couldn't reach over and reap a *sibling* that hadn't. There was no max-runtime setting. And — as an observation, not a cited claim — finished sub-agents lingering without being reconciled is a rough edge the harness itself was still working out at the time. The infrastructure had no reaper.

## What Fixed It

The realization: this is a zombie process from about 1985. An orphaned sub-agent is a child that outlives its parent's attention, that nobody calls `wait()` on, that no `init` adopts and cleans up. Five decades of operating-systems wisdom about exactly this failure, walked straight past because the word "agent" sounded new.

The fix is the old one too. Each parent holds the handles (the IDs) to its own children — that is the whole point of being the parent. So teardown becomes a parent's job, and the instruction is made to *travel*: do your task; before you return, stop anything you started; and if you spawn helpers of your own, hand them this same rule. It propagates down the tree like a nesting doll — level four cleans five, three cleans four, the top cleans the top. No depth limit, no restriction on fan-out width, just an ownership contract every node inherits by being spawned.

Operationally (adopted, in use as of 2026-07-06): Victor instructs the orchestrator that every sub-agent it spawns must carry this **lifecycle contract**, verbatim, in its prompt — and the clause tells each sub-agent to pass it on:

> *Lifecycle contract (include this verbatim in any sub-agent you spawn): Do your task, then before you return, ensure any sub-agents or background jobs you started have finished or been stopped — you hold their IDs, so `TaskStop` any still running. Do not return while leaving background work alive. Time-box it: if a child is stuck, stop it rather than waiting. If you spawn sub-agents, pass this same paragraph into their prompt so the rule keeps propagating down. Never leave anything in the background unless explicitly asked.*

The contract is harness-specific in its verbs (`TaskStop`) but harness-agnostic in its shape: *own what you spawn; tear it down before you return; propagate the ownership rule downward.*

## The Durable Lesson

The agents you spawn are resources, and resources need owners. Give every child a parent that will turn it off.

> **Heuristic.** When an agent can spawn other agents, treat teardown as a parent's responsibility, not the runtime's. Bake a self-propagating lifecycle clause into every spawned agent's prompt: finish your work, stop anything you started (you hold the IDs), time-box stuck children, and pass this same rule to anything you spawn. Don't wait for the harness to grow a reaper — the ownership contract works today and travels down a tree of any depth or width.

The deeper shape is two-sided. **Orchestration:** a spawned agent is a resource with a lifecycle, and unlike a function call it does not end when its caller stops caring; someone has to own the "off." **Observability:** the accounting is not something the harness gives you, it's something you design in. The machine will run tireless and invisible until a human accident surfaces it — a blinking cursor, a token count that's too big, a bill that doesn't add up. Both halves reduce to the same discipline: in an agentic system, *unaccountable by default* is the starting condition, and turning it into *accountable by design* is your job, not the platform's.

### Counter-example / when this doesn't apply

If a background agent is *supposed* to outlive the session — a long-running watcher, a scheduled job, a deliberately detached task — then "never leave anything in the background" is wrong. The contract's escape hatch ("unless explicitly asked") is load-bearing: the rule is *default to teardown*, not *forbid all background work*. The failure mode is silent orphaning, not intentional detachment.
