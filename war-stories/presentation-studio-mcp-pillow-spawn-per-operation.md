# presentation-studio-mcp / The Leak the Agent Couldn't See

## Date / Version Context

- **Date:** The persistent-worker design was the original Phase-1 implementation (early in the project's arc, when image operations were first wired in). The spawn-per-operation fix landed at `pillowBridge.ts` — the specific commit isn't recorded, but the fix was driven by operator-observed symptoms during dev, not by a precautionary architectural choice.
- **Project:** presentation-studio-mcp — local MCP server that renders DeckSpecs into `.pptx` files. The Node/TypeScript renderer handles most of the rendering; Pillow is used for the specific image-processing operations Python does cleaner than Node (or that have stable Pillow APIs and noisier Node equivalents). The TS-Python bridge is `pillowBridge.ts`.
- **Surface for this story:** the TS↔Python boundary for image operations. Pre-fix: long-lived `python3` worker, IPC over stdin/stdout for each operation, ~10ms per call once warm. Post-fix: fresh `python3` spawn per operation, no IPC reuse, ~200-500ms per call, no state shared between calls.
- **Glossary, used in this writeup:** *Persistent worker* = a subprocess that stays alive across many operations, accumulating state in memory between calls. *Spawn-per-operation* = a subprocess that starts fresh for each operation and exits when done, with no state carried over. *Autonomous caller* = a caller (an agent, a scheduler, a queue worker) that doesn't watch system-level resource metrics or worker process health. *Cold-start cost* = the time it takes to initialize a subprocess (process spawn + interpreter init + library imports + the work itself, before any user code runs).

## What Was Being Attempted

Make image operations during render fast.

The product instinct was reasonable. Rendering a deck involves multiple image operations — cropping a brand logo, resizing photos to layout slots, format conversion (PNG ↔ JPEG ↔ WebP), color profile adjustments. A 12-slide deck might trigger 30-50 image operations during render. If each operation paid Pillow's full startup cost (~200-500ms), the render would take 6-25 seconds just for image processing, dominating the actual render work.

The standard fix for this shape of cost — *expensive setup, cheap per-operation work* — is to amortize the setup across many operations via a long-lived worker. Languages and frameworks across the stack have variants:

- Database connection pools amortize the TCP-handshake-plus-TLS cost.
- Browser pre-warming amortizes the Chromium-startup cost in scraping pipelines.
- ML inference servers amortize the model-load cost in prediction APIs.
- Python `multiprocessing` pools amortize the worker-init cost in parallel workloads.

A persistent `python3` worker for Pillow follows the same pattern. Start one `python3` process with Pillow imported; keep it alive; send each image operation over IPC; receive the result; reuse the process. First operation pays the startup cost; subsequent operations cost ~10ms each. The math works for any workload that does more than a few operations per worker lifetime.

The original `pillowBridge.ts` implemented this design. A TypeScript module spawned the worker once (lazily, on the first image operation), sent operations as JSON messages over stdin, received results as JSON on stdout, kept the worker alive across the whole render session.

By every workload-economics criterion, the design was right. Tests passed quickly; benchmarks showed the expected amortization; the dev loop was fast.

## What Went Wrong

The worker hung or leaked.

The specific symptoms — the persistent worker hung or leaked during development, and the operator saw the symptom:

**Hangs.** After some number of operations, the worker would stop responding. The TS code sent a request; the worker never replied. The TS code waited indefinitely (or hit its own timeout, which was longer than the operator's patience). The whole render pipeline stalled. The operator had to manually kill the worker process and restart the render.

**Leaks.** Memory usage in the worker process grew across operations. Pillow's internal state, image buffers held in caches, references that didn't get garbage-collected, file handles that didn't get closed — exact mechanism varies by Pillow version and operation type. After enough operations, the worker's RSS would balloon, the OS would start swapping, latency would degrade, and eventually the kernel would either OOM-kill the worker or the operator would notice the slowdown.

Either symptom is a *catastrophic* failure mode for an autonomous caller. The agent calling presentation-studio-mcp doesn't watch:

- The Python worker's process state.
- The worker's memory consumption.
- IPC timing across calls.
- Whether the worker has been alive for 5 operations or 5,000.

The agent's only visibility is *the response to its tool call*. If the response comes back, the agent assumes things are fine. If the response doesn't come back, the agent might retry (which doesn't help — the worker is still stuck), might fail the tool call (which still doesn't surface the underlying worker issue), or might just hang along with the worker. None of the agent's natural reflexes restore the worker to health.

The operator notices, eventually — *"the deck render is slower than usual"* or *"the agent's last several tool calls didn't return"* — but the discovery is *operator-side*, not *agent-side*. For a system designed to be called autonomously, that's the wrong place for the trust signal to live.

The structural shape: **the persistent-worker design assumed the caller would watch the worker's health and restart it as needed. The autonomous caller doesn't watch; the worker accumulates faults silently; the system degrades without alerting.** The amortization win turned into an unbounded failure surface.

## How It Was Discovered

By the operator watching renders get slower and stop, repeatedly, during dev.

The discovery channel is the same one the forge async-adapter story names: *accumulated friction* during dev that doesn't crystallize into a single-incident moment. The operator runs renders during iteration. Renders sometimes hang. Restarting the dev loop (which restarts the worker) makes things work for a while. The pattern eventually becomes named: *the worker is the unstable part*.

The non-discovery channel — the part worth pausing on — is that *no automated check on the system side would have surfaced this in CI*. CI runs each test against a fresh worker, so the worker never sees enough operations to leak meaningfully. Tests pass; benchmarks pass; the failure surfaces only under *real-shaped sustained use*, which is exactly the use the autonomous caller produces and that the test suite doesn't model.

The pattern is also adjacent to `canvas-mcp-feature-multiplicative-renderer.md` and `forge-redaction-never-fired.md` — both are *latent-state observations caught by deliberate retrospection or operator-observed friction, not by tooling.* This story is a third member of that meta-family: the test suite didn't see it, no metric tracked it, only the operator noticing *"why does this keep stalling"* surfaced it.

## What Fixed It

Spawn `python3` per operation.

The structural shape of the fix at `pillowBridge.ts`:

**Each image operation gets a fresh subprocess.** TypeScript spawns `python3` with the operation's args, the script imports Pillow, performs the operation, prints the result, exits. No reuse, no shared state, no IPC long-running.

**Cold-start cost is paid every time.** ~200-500ms per call. Slower than the persistent design, but bounded — every operation has the same cost shape.

**Process isolation eliminates accumulation.** Whatever Pillow does to its internal state during the operation goes away when the process exits. Memory is released by the kernel. File handles close. The next operation starts from a clean slate.

**The fix is small.** Removing the IPC layer and the worker-lifecycle management probably *reduces* the line count of `pillowBridge.ts`. Spawning a subprocess per call is structurally simpler than maintaining a long-lived one.

The trade-off, stated directly: *a slow-but-isolated subprocess beats a fast-but-leaky persistent one when the caller is autonomous and won't notice the leak.* The operator's job isn't *"watch the worker process for health"* — that's no one's job in an autonomous-call system. The operator's job is *"design defaults that don't require watching."*

What's load-bearing isn't the per-operation spawn cost (it's real but bounded — adds ~10-20 seconds to a full render, which is acceptable for a non-interactive render flow). What's load-bearing is the *isolation*. The leak surface is gone because there's nothing to leak into. The hang surface is gone because each call has its own process lifecycle. The system is *predictable* in a way the persistent design wasn't.

What's owed but not yet shipped: a *per-operation timeout* on the subprocess spawn — currently if a single Pillow call hangs (rare with fresh processes, but possible if Pillow itself misbehaves on a specific input), the spawned `python3` could still hang the TS caller. A timeout-and-kill discipline closes that residual surface. The current design is much better than the original; not yet perfect.

## The Durable Lesson

A slow-but-isolated subprocess beats a fast-but-leaky persistent one when the caller is autonomous and won't notice the leak. **Subprocess hygiene defaults should match the caller's actual attention surface, not the workload's optimal economics.**

The mental-model flip the lesson rests on: subprocess design choices are usually framed as *throughput vs. startup cost* — persistent for high-throughput, spawn-per-call for low-frequency. That framing assumes the *caller* is watching the subprocess's health and will restart it when needed. For an autonomous caller, the framing breaks: the caller isn't watching, the subprocess can't self-heal, and *fast-but-leaky* compounds without anyone noticing until the system stops working.

For agentic systems specifically, the cost compounds. The agent's view of the system is its tool responses. Subprocess health, memory consumption, process lifetimes — none of these are in the agent's response surface. The agent has *no signal* that the underlying worker is degrading. Standard tool-error patterns (returning `{"error": "timeout"}` or similar) tell the agent that *the call failed*, but don't tell the agent *why* — and don't help if the next call to the same broken worker also fails.

> **Heuristic.** For any agentic tool that spawns subprocesses or maintains long-lived workers, default to *spawn-per-operation* even when it's slower than amortized alternatives. Only adopt persistent workers when you have a *named accountability path* for monitoring and restarting them — a sidecar process, an operator-watched dashboard, a CI check that asserts worker uptime. Without that path, autonomous callers will accumulate hangs and leaks silently. The cost of the slower default is bounded (cold-start per call); the cost of the unmonitored persistent worker is unbounded (the failure mode is *eventual stop without alert*).

The shape generalizes far beyond Pillow:

- **Headless browser workers (Puppeteer, Playwright).** Pre-warmed browsers leak DOM state, cookies, memory across navigations. For autonomous callers (scrapers, screenshot agents), spawn-per-call survives better than persistent pools.
- **ML inference workers.** Persistent inference processes can accumulate GPU memory fragmentation, cache corruption, or graph-state issues across calls. For autonomous callers with bursty patterns, spawn-per-call (or spawn-per-N-calls with strict recycling) is more reliable.
- **PDF/document generators.** LibreOffice, wkhtmltopdf, similar tools — persistent instances leak temp files, font caches, internal state. Spawn-per-call is the default for agentic invocations.
- **Compilers, transpilers, build tools.** Persistent compiler daemons (TypeScript's `tsserver`, Java's `javac --daemon`) leak across sessions; for one-shot autonomous calls, spawn-per-call is the safer default.
- **Database connection pools called from autonomous workers.** Connection-pool leaks under autonomous callers are notorious. Spawn-per-call (or explicit short-lived connections) trades throughput for predictability.

In every case, the pattern is identical: **persistent workers amortize startup costs by carrying state across calls; carried state can leak or corrupt; autonomous callers can't see the leak; predictability beats throughput when no one is watching.** The fix is structural — pick subprocess defaults that match the caller's attention surface.

The pairing worth naming: this is the *subprocess-lifecycle* version of the *quiet-limit-under-autonomous-callers* pattern. Pairs with `forge-async-adapter-smoke-test.md` on *runtime lifecycle the call site doesn't see* (queue adapter vs. subprocess worker), `presentation-studio-mcp-jsonrpc-stdio-fallback.md` on *the tool's dependency surface should survive its dependencies misbehaving* (SDK breakage vs. worker hangs), and `canvas-mcp-viewer-lifecycle.md` on *side-effect emitters that the caller doesn't track*. Four stories, one underlying lesson: **for autonomous callers, the system has to default to predictability over optimization, because the caller doesn't catch the optimization's failure modes.**

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *workers feeding autonomous callers that don't watch worker health*. It doesn't apply to:

- **Human-watched dev loops.** A developer running renders interactively watches latency, notices hangs, restarts workers. Persistent workers are fine when a human is in the loop. The lesson kicks in for *autonomous* callers, not interactive ones.
- **Workers with strong self-monitoring.** A worker that emits health metrics, has a watchdog process, and auto-restarts on hangs can be persistent because the *system* is watching, even if the *caller* isn't. The lesson is for workers without that infrastructure.
- **Genuinely high-throughput workloads.** A worker serving thousands of requests per second can't pay 200-500ms cold-start per call; the math doesn't work. For those, persistent workers + active monitoring is the right answer, with the monitoring being non-negotiable.
- **Workers with strict, audited cleanup.** Some workers (Erlang OTP processes, Rust workers with strict lifetimes) have lifecycle guarantees that prevent the accumulation problem at the design level. The lesson is for workers in languages/frameworks where state accumulation is the default.
- **Stateless workers.** A pure-function worker that genuinely holds no state across calls (idealized, rare in practice) doesn't leak. If your worker really has no state to leak, the persistent design is safe. Most workers in practice have *some* state (caches, file handles, library globals), so this exception is narrower than it sounds.

The signal: *if this worker silently degraded over the next 1000 operations, who would catch it?* If the answer is *"no one until the system breaks visibly"*, the worker shouldn't be persistent.

## What This Story Is *Not* Evidence For

- **Not evidence that persistent workers are always bad.** They're great when a human watches them. The lesson is *predictability over throughput when no one is watching*, not *never use persistent workers*.
- **Not evidence that Pillow is poorly designed.** Pillow is fine. The pattern of *image-processing libraries accumulating state across operations* is common across the ecosystem; the lesson generalizes regardless of which library.
- **Not evidence that spawn-per-operation is always the right default.** It's the right default *for autonomous callers without lifecycle monitoring*. For interactive callers, throughput-optimized workers, or workers with self-monitoring, the calculus differs.
- **Not evidence that the persistent-worker design was a mistake at design time.** The original design optimized for the workload's economics — a reasonable starting point. The mistake was *not noticing the autonomous-caller dynamic until the worker started hanging*. The lesson is the *framing shift* (caller's attention surface matters more than workload economics), not *don't try persistent workers ever*.
- **Not evidence that the existing test suite was inadequate.** Tests covered correctness of each operation. The bug is at the *worker-lifetime* layer, which standard tests don't exercise (each test gets a fresh worker, so leaks never accumulate). The structural follow-up is *test-with-sustained-use* — run N operations against the same worker, assert resource usage stays bounded. Not yet shipped; lives in operator attention.
