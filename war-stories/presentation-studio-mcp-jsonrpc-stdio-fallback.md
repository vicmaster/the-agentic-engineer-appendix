# presentation-studio-mcp / Sixty Lines of Insurance

## Date / Version Context

- **Date:** The SDK breakage incident was during the project's development arc — in the dev loop, not in production. The fallback code at `apps/mcp-server/src/index.ts:144-204` is the artifact; the specific commit isn't recorded, but the fallback became *load-bearing* the moment the SDK broke and the project's development continued through it.
- **Project:** presentation-studio-mcp — local MCP server that renders DeckSpecs into `.pptx` files. The server is invoked via stdio (the standard MCP transport for local servers); the agent client talks to it via JSON-RPC framed over stdin/stdout. The official MCP SDK provides Server/Tool/Resource abstractions that wrap the raw protocol.
- **Surface for this story:** the MCP server's bootstrap path. Pre-fallback: `import { Server } from '@modelcontextprotocol/sdk'` and use the SDK's abstractions. Post-fallback: try-import the SDK; if it fails (version mismatch, missing install, breaking API change), fall through to a hand-rolled loop that implements the four core methods directly against the JSON-RPC-over-stdio protocol.
- **Glossary, used in this writeup:** *MCP* = Model Context Protocol, the JSON-RPC-shaped protocol agents use to call external tools and read external resources. The protocol itself is small and stable; the SDKs that abstract it are larger and change more. *JSON-RPC over stdio* = the wire protocol for local MCP servers — newline-delimited JSON-RPC messages on stdin/stdout. *Fallback loop* = a hand-rolled implementation of a protocol's core methods, used when the official SDK isn't available or is broken. *Core methods* = the minimal set of JSON-RPC methods an MCP server needs to be useful — for presentation-studio-mcp, that's `tools/list`, `tools/call`, `resources/list`, `resources/read`.

## What Was Being Attempted

Build an MCP server using the official SDK.

The product framing was obvious. MCP servers exist in an ecosystem with SDKs in multiple languages (TypeScript, Python, others); using the official SDK is the obvious starting point. The SDK provides:

- Server lifecycle (initialize, register tools, register resources, run the main loop).
- Tool registration with typed input schemas (often Zod-shaped) and handler functions.
- Resource registration with URI schemes and read handlers.
- JSON-RPC framing on stdio (or HTTP, depending on transport).
- Error handling, request/response correlation, the rest of the protocol's mechanical layer.

Without the SDK, every MCP server author would have to implement all of this from scratch. The SDK is *meant* to be the abstraction that lets server authors focus on their tools, not on the protocol. presentation-studio-mcp's first commits used the SDK exactly as intended — register a `render_deck` tool, register a `read_deck_spec` resource, hand off to the SDK's main loop.

The SDK worked. The server worked. The agent client (Claude) called the server's tools via the standard MCP protocol, the SDK handled framing, the renderer ran, the `.pptx` files came back.

## What Went Wrong

The SDK broke.

The specific breakage was a version mismatch — a breaking change between MCP SDK minor versions, or a peer-dependency conflict, or an update that changed the SDK's public API in a way that wasn't backwards-compatible with the server's existing code. (Which one it was isn't the point, and neither is the precise SDK error message — the *class* of failure is.) From the operator's perspective, the symptom was identical to *the server is broken*:

- Run the server.
- Server fails to start (or starts and immediately exits with an exception).
- Stack trace from the SDK's internals (or a missing-export error, or a peer-dependency warning that escalated to a runtime error).
- Agent client can't connect.
- The whole dev loop — agent calls tool, tool runs, agent iterates — halts.

The mental-model question that hits in that moment is *who owns this failure*. The natural reflexes:

- *"My code is wrong."* — but the diff is small; the code that was working an hour ago shouldn't suddenly be wrong.
- *"The SDK has a bug."* — possible, but the fix is *file an issue and wait*, which doesn't keep the dev loop moving.
- *"I need to pin the SDK version."* — works for the immediate problem, doesn't solve the structural one (the next SDK change will surface again).
- *"I need to not depend on the SDK for the critical path."* — the structural fix, but feels heavyweight.

The structural cost is precise: *the MCP SDK will break at the worst possible moment.* SDKs break during dev when you're iterating fast (the worst time for a tooling failure), during demos when you're showing the system to someone (the most embarrassing time), during production rollouts when you can't easily revert (the most expensive time). The breakage isn't *if*; it's *when*.

For an agentic system whose entire value proposition is *the agent can call the tool reliably*, the tool's uptime depending on a third-party SDK's API stability is a *quiet-limit* failure mode — the kind of dependency-surface assumption that doesn't fail until it does, and when it does, fails the whole system at once. Same shape as `forge-async-adapter-smoke-test.md` at the queue-adapter layer, `leads-crm-heroku-boot-deployment-contracts.md` at the deployment-environment layer, and the harness-limit stories elsewhere in this collection. Different surfaces, same lesson: *the orchestrator who hasn't named the dependency surface can't see the failure coming.*

## How It Was Discovered

When the server stopped starting.

The discovery channel was the dev loop's break-glass moment. The operator ran the server, the server failed, the error pointed at the SDK, the operator's flow halted. The broader pattern: *the MCP SDK will break at the worst possible moment* — meaning the discovery is reliably *during active work*, not during a scheduled maintenance window.

The non-discovery channel — the part worth pausing on — is that *standard test discipline doesn't catch SDK breakage*. The tests run against the SDK that's currently installed; if the SDK is fine in CI's pinned version, the tests pass. The breakage shows up when the SDK changes upstream — an `npm install` that pulls a new minor, an automatic upgrade in a CI environment with looser pinning, a dev machine that's behind on package updates. The standard verification stack is *blind* to upstream-SDK changes between runs.

This is the *dependency-surface-as-quiet-limit* class. Same structural shape as deployment-time contracts and dev/prod adapter mismatches: the failure mode is *the environment changed in a way the code's tests don't model*. For deployment, the change is environment-strictness; for adapters, it's adapter-lifecycle-semantics; for SDKs, it's SDK-API-stability. All three are *outside* the code's normal verification surface; all three surface only when the change hits production-shaped use.

## What Fixed It

A 60-line JSON-RPC stdio loop covering the four core methods.

The structural shape of the fix at `apps/mcp-server/src/index.ts:144-204`:

**Try-import the SDK; fall through on failure.** The bootstrap path attempts to import the official SDK; if the import throws (because the SDK isn't installed, isn't compatible, or has a breaking API change), the catch block invokes the fallback path. The server runs *either* way.

**Implement the four core methods directly against JSON-RPC.** The fallback path reads newline-delimited JSON from stdin, parses each line as a JSON-RPC request, dispatches by method name to the project's own handlers, writes the response JSON to stdout. Four methods are enough: `tools/list` (return the server's tools and their schemas), `tools/call` (dispatch a tool invocation to its handler), `resources/list` (return the server's resources and their URIs), `resources/read` (dispatch a resource read to its handler). Other methods (`initialize`, `notifications/*`, etc.) get minimal stub responses or are silently ignored — they're not on the critical path for the agent's actual work.

**Keep the protocol-level error semantics.** JSON-RPC error codes, `id` correlation, jsonrpc-version field, request/response framing — all of these match what the official SDK produces. The agent client can't tell whether it's talking to the SDK or the fallback. Same wire protocol, different implementation.

The 60-line size of the fallback is the load-bearing detail. It's small enough that any developer can read it end-to-end in a few minutes; it's small enough that maintenance is trivial (the JSON-RPC protocol itself doesn't change). It's just large enough to cover the actual surface the agent uses. There's a temptation to either *do the whole protocol* (200+ lines, ongoing maintenance burden, no payoff) or *skip the fallback* (zero lines, zero protection). The 60-line shape is the right size: cover what you use, ignore the rest.

The surprise worth naming: *this is the kind of "robustness shim" you usually regret.* The instinct against fallback paths is correct in many contexts — they accumulate, drift from the primary path, become their own source of bugs. For this specific case, the fallback survived because:

- **The fallback exercises the same wire protocol as the SDK.** The two implementations agree on what they emit and consume. There's no protocol-drift surface area.
- **The fallback's surface is small (4 methods).** Drift would have to happen across one of those four; the SDK changes that affected those methods would also affect the fallback's exposure to them, surfacing immediately.
- **The fallback is testable.** A simple integration test (start the server with the fallback path, send a `tools/list` JSON-RPC request, assert the response) catches regressions cheaply.

What's owed but not yet shipped: an *active drift detector* — a CI job that runs both the SDK path and the fallback path against the same test suite and asserts they produce identical responses. Currently the fallback is only exercised when the SDK is broken; a continuous drift test would catch the SDK-vs-fallback divergence proactively. Not load-bearing yet (the fallback's small surface keeps drift unlikely), but worth adding as the project grows.

## The Durable Lesson

When an SDK is the only path to your tool, it becomes a single point of failure. **Keep a hand-rolled escape hatch for the methods you actually use.**

The mental-model flip the lesson rests on: SDKs are *abstractions over protocols*. The abstraction's value is convenience (don't write framing, don't handle errors, don't dispatch by method name); the protocol's value is *stability* (the wire format changes much less often than the SDK's API). For *protocols you control or that have rough stability* (JSON-RPC, HTTP, gRPC's wire format), the protocol is the durable interface and the SDK is the convenience layer. Treating the SDK as the only path conflates the two — your tool's reliability ends up depending on the SDK's API stability, which is much weaker than the protocol's stability.

For agentic tools specifically, the cost compounds. The agent client uses the protocol (not the SDK); the tool's compatibility surface with the agent is the *protocol*. The SDK is internal to the tool. If the SDK breaks, the tool's external compatibility is unchanged — the protocol is still the same — but the tool's *bootstrap* fails because it can't load the SDK. The fallback restores the bootstrap path; the agent never notices.

> **Heuristic.** For any tool whose external interface is a stable protocol (JSON-RPC, HTTP, etc.) and whose internal implementation depends on a third-party SDK, build a small fallback that implements the protocol's core methods directly. The fallback should cover *only* the methods your tool actually uses (not the entire protocol surface). The fallback should be small enough to read in one sitting (~60-100 lines) and testable with a simple integration test. Activate the fallback when the SDK isn't available or fails to load. The cost is one-time (write the fallback once); the savings are bounded by *how often the SDK breaks*, which is *more often than you expect*. Treat the fallback as insurance — you hope to never use it; you're glad it's there when you do.

The shape generalizes far beyond MCP:

- **HTTP client SDKs.** A service that depends on a third-party SDK for HTTP requests can keep a `fetch`/`http`-shaped fallback for the specific endpoints it calls.
- **Cloud-provider SDKs (AWS, GCP, Azure).** The protocol layer (REST, gRPC) is stable; the SDKs change. A fallback that calls the REST endpoints directly for the few operations you actually use survives SDK breakage.
- **Database drivers.** The wire protocol (PostgreSQL's, MySQL's) is stable for years; the driver libraries change. For critical-path queries, a fallback that speaks the wire protocol directly is feasible (though usually overkill).
- **Webhooks frameworks.** The webhook payload format is fixed by the sender; the receiving framework can be replaced. A bare-HTTP receiver that handles the payload format directly survives framework changes.
- **Build tools / bundlers.** The output format is what downstream consumers care about; the build tool can be swapped if it breaks. (Not exactly a fallback, but the same principle — depend on the format, not the tool.)

In every case, the pattern is identical: **the external interface (the protocol, the wire format, the output shape) is stable; the SDK / framework / tool that produces it is mutable. Depending on the SDK as the only path makes the tool's reliability hostage to the SDK's API stability.** The fix is a thin fallback that implements the stable interface directly, sized to cover only what you actually use.

The pairing worth naming: this is the *third-party-SDK* version of the *quiet-limit* pattern. Pairs structurally with `forge-async-adapter-smoke-test.md` on *the runtime contract under the call site you can't see*, `leads-crm-heroku-boot-deployment-contracts.md` on *production environment enforces contracts dev doesn't*, `coide-askuserquestion.md` on *protocol auto-resolution windows you didn't budget for*, and `markdown-toolkit-subagent-depth.md` on *harness delegation depth limits in the agent runtime*. Five quiet-limit stories, five projects, one structural lesson: **the orchestrator who hasn't read past the introductory docs into the SDK's release notes, the runtime's lifecycle behavior, the deployment environment's strictness, the protocol's auto-resolution semantics, or the harness's depth limit will ship plans whose validity depends on assumptions the platform doesn't honor.** The fallback / parity-check / pre-flight-check / deploy-as-integration-test discipline is the structural counterweight.

Pairs also with `presentation-studio-mcp-agent-tool-split.md` — both are *keep the worker boring* lessons applied at different layers. That story is the agent/tool boundary (the agent decides what to say; the tool decides geometry). This entry is the tool/SDK boundary (the SDK abstracts the protocol; the tool can survive without the SDK). Both argue for *the boring layer should be replaceable; the protocol is what's durable.*

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *tools whose external interface is a stable protocol and whose internal SDK is on the critical path*. It doesn't apply to:

- **Tools with no stable external protocol.** A tool whose external interface is itself the SDK's surface (a library, an internal API that changes with the SDK) can't have a fallback — there's nothing stable to implement directly. The lesson is for tools that sit *behind* a protocol, not *as* the protocol.
- **Cases where the SDK is the protocol.** Some SDKs *are* the standard — protobuf's reference impl, OAuth library reference impls. There's no separate protocol stability to fall back to. The lesson is for cases where the SDK is *one implementation* of a protocol that has multiple.
- **Very young protocols.** A protocol that's itself unstable (still in draft, breaking changes every few months) doesn't provide the stability the fallback needs. The lesson assumes the protocol's stability exceeds the SDK's — true for MCP (the wire protocol is much more stable than the SDKs), not true for protocols mid-evolution.
- **Single-shot SDK uses with no critical-path dependency.** A tool that uses an SDK once at startup for a non-critical configuration step doesn't need a fallback — if it fails, the tool can warn-and-continue, no protocol surface to maintain.
- **SDKs from your own organization where you control the stability.** If you write both the SDK and the consumer, the SDK's stability is under your control; you can fix breakage at the SDK layer instead of fallback-ing past it.

The signal: *if this SDK breaks, does my tool's external interface still need to work?* If yes (the tool is on someone's critical path; the protocol is stable; the SDK is internal), build the fallback. If no, the SDK breakage is acceptable downtime and the fallback is overkill.

## What This Story Is *Not* Evidence For

- **Not evidence that SDKs are bad.** Most of the time, the SDK is the right path — convenient, well-maintained, fewer bugs than a hand-rolled implementation. The lesson is *have a fallback for when the SDK breaks*, not *don't use the SDK*.
- **Not evidence that the MCP SDK specifically is unreliable.** The MCP SDK is fine; the lesson is about *any* SDK being on a critical path. The same shape applies to any SDK that wraps a stable protocol.
- **Not evidence that fallbacks should be built for every dependency.** The lesson is specifically for SDKs that wrap *stable protocols* you control or that have rough stability. For dependencies whose value *is* the API (libraries with no underlying protocol), fallbacks aren't applicable.
- **Not evidence that the fallback was the primary motivation.** It's clear the fallback was built somewhat defensively before it was needed; it became load-bearing the moment the SDK broke. The lesson is *build the fallback before you need it*, not *build the fallback only after the SDK has broken once*.
- **Not evidence that the 60-line size is special.** Some tools' fallbacks will be larger, some smaller. The lesson is *size to cover what you actually use*, not *aim for 60 lines specifically*. The 60 here is what presentation-studio-mcp's surface required.
