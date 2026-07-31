# presentation-studio-mcp / The Agent/Tool Split: Keep the Worker Boring

## Date / Version Context

- **Date:** Repo init `ee61fa1`, 2026-04-09. Five commits same day, all from one author. Current version `0.1.0`. The codebase arrived fully-formed in the initial commit — 349 README lines, 15 layouts, 12 audit rules, 13 MCP tools, 9 Pillow ops, full docs, 30+ tests. The "agentic build approach" *is* the project. The follow-up commits were CI polish and README translation.
- **Project:** presentation-studio-mcp — a local, brand-aware engine that turns a `DeckSpec` (JSON) into a real, production-quality `.pptx` file, exposed as an MCP server so Claude Code / Codex can use it as their presentation backend. TypeScript monorepo (pnpm workspaces, Zod schemas, PptxGenJS renderer), Python+Pillow sidecar for image processing, all running locally over stdio. No external services.
- **Surface for this story:** the entire architecture. There is no single component to point at because the principle lives in *every* boundary the codebase draws. The 13 MCP tools, the 15 layouts, the 12 audit rules, the Pillow worker, the JSON-RPC fallback, the denormalized brand inside `DeckSpec` — all of them are instances of the same rule.
- **Glossary, used in this writeup:** *DeckSpec* = the JSON contract the agent produces and the tool consumes — slides, content, layout names, brand id (and now denormalized brand). *Layout* = a named, deterministic geometry template (`hero-cover`, `two-column-text`, `comparison-table`, etc.). *Audit rule* = one of 12 deterministic checks that runs over a deck and emits structured warnings or errors before render. *Worker* = the deterministic side of the system — the renderer, the audit pipeline, the Pillow sidecar. *Actor* = the probabilistic side — in production, the agent calling the MCP tools. *Boundary* = the line the architecture draws between what the actor decides and what the worker decides.

## What Was Being Attempted

Build a backend for agents to produce real `.pptx` files, where "real" means production-quality, brand-consistent, geometry-correct decks that don't embarrass anyone in a leadership meeting.

The trigger: *agents are good at writing slide content and bad at producing decks that look professional — they hallucinate fonts, blow past safe areas, generate "creative" colors that wreck brand consistency, and use whatever PPTX library they remember from training*. The realization was that *an agent doesn't need help writing slides — it needs a backend that refuses to let it touch geometry, typography, or branding.*

That sentence is the entire architectural thesis. The codebase is the working demonstration of the thesis. Every meaningful design choice in presentation-studio-mcp falls out of one rule: **agent decides what to say; tool decides how it looks.**

This is not a refactor that arrived after the fact. The line was drawn before the first commit. The proof is that every API in the codebase preserves it.

## What Went Wrong (Counterfactually)

There's no incident. The project hasn't shipped a bug because it hasn't allowed the failure mode to exist.

What this story is documenting is *what would have gone wrong in the absence of the boundary* — the architectures the project deliberately rejected, each of which has a name and a known failure shape. Naming them is what the writeup is for.

**Architecture A — *the smart renderer*.** A tool that accepts unstructured prompts ("make me a deck about Q3 risks") and decides everything: layout selection, content generation, geometry, typography, branding. Failure shape: every slide is a small mystery; the agent has no way to iterate on geometry without re-prompting; brand consistency is a coin flip; the system has no way to refuse bad input because it accepts everything as input.

**Architecture B — *the fancy renderer with agent-pluggable smarts*.** A tool that accepts structured input but lets the agent override layout decisions, typography, color choices, "for flexibility." Failure shape: every override is a place the agent's judgment leaks into geometry. The agent picks a "better" font because the prompt suggested formality; the font isn't installed on the recipient's machine; the deck renders broken. The boundary is theoretical; the implementation is porous.

**Architecture C — *the strict renderer that 4xx-rejects bad input*.** A tool that enforces the boundary by refusing anything that doesn't validate. Failure shape: agentic callers don't read errors well. They retry the same broken call. They hallucinate around the validation. The strict renderer produces nothing rather than something useful, and the agent has no signal it can act on. (This is the failure mode `presentation-studio-mcp-feedback-loops.md` documents directly.)

The architecture presentation-studio-mcp actually shipped — let's call it Architecture D — is the one that *normalizes, warns, renders the gap visibly, and emits structured feedback the agent can act on*. The boundary is enforced at the worker; the worker still produces output even when the input was sloppy; the warnings are how the agent learns. None of A, B, or C produces this combination.

## How It Was Discovered

The architecture was decided *before* the project existed. There's no "discovery" in the incident sense. The discovery is in the *artifact* — the codebase is the demonstration that the boundary works, and every API in the codebase is evidence. To name what was discovered, you have to walk through how the boundary shows up in code.

Six pieces of evidence, each one a place where the boundary could have leaked and didn't:

1. **The 15 layouts are a fixed enum.** Layout names are a closed set: `hero-cover`, `two-column-text`, `comparison-table`, `logo-wall`, etc. The agent picks from the set. The agent does *not* pass arbitrary geometry. If the agent wants a layout the system doesn't have, it has to either pick the closest existing one or accept the fallback. The boundary is enforced by the type system, not by reviewer discipline.
2. **Unknown layouts fall back; they don't error.** `renderSlide.ts` rule, paraphrased: *any unknown layout falls back to `two-column-text` with a warning*. This is the load-bearing four-line rule that lets the boundary stay strict without trapping the agent on hallucinated layout names. The fallback is not an escape hatch; it's the same boundary, drawn around the failure case.
3. **Per-layout density thresholds.** `hero-cover` allows 40 words; `two-column-text` allows 140; `logo-wall` allows 30. *Too dense* is not one number — it's a per-layout judgment baked into the audit code. The agent doesn't get to decide how dense is too dense; the layout's threshold decides. The agent's job is to write content that fits, and the audit is how the agent learns whether it fit.
4. **Denormalized brand inside `DeckSpec`.** The deck stores a *copy* of the brand, not just `brand.id`. The agent never has to know the registry's state; the deck is portable across registry drift. The boundary protects the deck artifact from the agent's incomplete view of server state. This is the same pattern as `leads-crm-org-scoping.md`'s lesson — *don't make persisted artifacts depend on registry state the agent can't see* — applied at the artifact-design layer rather than the multi-tenant data layer.
5. **`normalize_deck_spec` is a separate tool from `render_deck`.** The agent can iterate on a noisy spec without burning a render cycle. Normalization is its own contract: structured input → structured output + warnings. The agent's loop is: normalize, read warnings, fix the spec, render. The boundary preserves *the agent's iteration loop* as a first-class concern, not as an afterthought.
6. **No-auto-fill rule.** If a required field is empty, render leaves the space blank and emits a warning. The agent sees the gap on inspection and fills it correctly, instead of accepting a hallucinated default. Auto-filling would teach the agent that missing fields are fine; the rule explicitly refuses to teach that lesson.

Six pieces. Six different APIs. One rule. The rule is the writeup.

## What Fixed It

There's nothing to fix. The project is *the fix* applied as a design choice from day one.

What this writeup contributes — what makes it worth writing down — is the *naming* of the rule that organizes all six pieces, in a way that's portable to other systems where the same line needs to be drawn.

## The Durable Lesson

When a system has a probabilistic actor (anything generative, autonomous, or under-specified) sitting above a deterministic worker, keep the worker boring.

Push every meaning-decision up to the actor; let the worker only handle "how do we shape this exact input into this exact output." If you can't draw the line before writing code, you don't have a tool — you have a confused agent in a trench coat.

This is a load-bearing rule for agent/tool architecture. It's the cleanest single statement of *what's the right boundary between agent and tool* the corpus produces. Other stories about this boundary are cases where it was drawn wrong (`coide-ship-feature.md`'s autonomous side-effect skill) or wasn't drawn at all (canvas-mcp's half-built `evaluate.ts` stub). This story is what the boundary looks like when it's drawn right.

> **Heuristic.** Before you start building, on a whiteboard or in a sentence, name the line. *The actor decides X. The worker decides Y.* If you can't write that sentence with concrete X and Y, stop and figure out what the line is. Every fan-out point inside the worker is one more place the actor could have made the decision instead. Every "let the agent override the default" is a leak. Every "the tool decides what to write" is the wrong direction across the boundary. The rule scales: most failures of agent/tool architectures are failures to draw the line, not failures to implement it correctly.

The shape generalizes to many systems where a probabilistic actor sits above a deterministic worker:

- **Human teams.** Senior engineers (actor) decide what to build; juniors and shared infrastructure (worker) execute deterministic steps. Pairing breaks down the moment a junior is asked to make narrative decisions without a spec. The fix isn't more pairing; it's more clarity about which decisions belong to which side.
- **Microservice boundaries.** Services that decide intent (orchestrators, BFFs) shouldn't share memory with services that decide mechanics (storage, codec, rendering). The interface between them is exactly the audit-and-warn shape — the worker accepts structured input, normalizes, emits warnings, renders.
- **Contractor/vendor relationships.** The principal decides scope and acceptance criteria; the contractor produces the artifact. Contracts that let contractors decide scope mid-flight are the same failure mode as agents picking their own layouts. The rule scales identically: the boundary belongs in the contract, not in the conversation.
- **CI/CD pipelines.** Engineers (actor) decide what to deploy; the pipeline (worker) decides how. Pipelines that "helpfully" mutate config or "auto-detect" deployment targets are the same leak — the worker is making meaning decisions that belong upstream.
- **LLM-tool integrations of any kind.** The agent decides; the tool acts. Every tool that "gets smarter" by adding agent-like reasoning has, by definition, blurred the line. Sometimes that's the right move. More often it's the leaky one.

This pairs as the *positive* version of the rule for which `coide-ship-feature.md` is the negative version. coide built a tool whose body had durable side effects and let the agent invoke it autonomously — the boundary was drawn at the wrong layer. presentation-studio-mcp drew the boundary at the right layer and enforced it with code. Same line, opposite outcomes. Both stories belong side by side.

It also pairs structurally with `presentation-studio-mcp-feedback-loops.md`. That writeup is *agentic callers need feedback that doesn't break their loop* — the audit IS the eval, normalize-don't-reject, warn-don't-error. This writeup is *the boundary between agent and worker is the load-bearing architectural decision*. Together they form the two halves of the project's contribution to the book: the boundary is *what the architecture is*, and the audit pipeline is *how the agent learns within the boundary*.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *systems where a probabilistic actor sits above a deterministic worker*. It doesn't apply to:

- **Systems with no probabilistic component.** A pure deterministic pipeline (data ingestion, ETL, build systems before the AI era) has no actor in the agentic sense; the rule doesn't add anything. The line between "what the user asks for" and "what the system does" is already the rule, just at a different layer.
- **Systems where the worker is genuinely supposed to make meaning decisions.** A search engine ranking results, a recommender system picking what to show, a content moderation system deciding what to allow — the worker *is* the decider in those systems. The rule says *keep the worker boring*; if the worker is the meaning-maker by design, the rule doesn't apply.
- **Cases where the actor is also deterministic.** A code generator producing a deck spec from a database query is not a probabilistic actor; both sides are deterministic. The rule's value is in containing probabilistic behavior; without that, it's just "separate concerns," which is a different (older) rule.
- **Systems where the cost of "the worker decides" is genuinely low.** A throwaway script that prints whatever the LLM produces with no downstream consumer doesn't need the boundary. The cost of the failure mode is zero. Don't engineer a boundary for a system whose blast radius is one terminal window.

The signal: *is there a probabilistic actor whose decisions could land as durable, externally-visible artifacts?* If yes, draw the line and enforce it. If no, the rule is over-engineering for your situation.

## What This Story Is *Not* Evidence For

- **Not evidence that all systems should refuse to let the agent decide.** The lesson is about *meaning decisions that affect the artifact's quality*. There are systems where letting the agent decide is the whole point — agentic creative tools, exploration assistants, conversational interfaces. The rule is about *backends for agents*, not *interfaces for agents*.
- **Not evidence that the renderer should be feature-poor.** The 15 layouts and 12 audit rules are not minimal — they're the result of careful design within the boundary. The rule is *push meaning-decisions up*, not *make the worker featureless*. Within its responsibility (geometry, spacing, typography, branding), the worker is rich. It just doesn't reach beyond that responsibility.
- **Not evidence that this architecture is universally applicable.** It works for `.pptx` because there's a clean boundary between content (what to say) and presentation (how it looks). Some domains don't have that boundary cleanly available — a legal document where the wording *is* the presentation, or a piece of code where the abstraction *is* the meaning. The rule applies wherever the boundary is drawable; it doesn't help when it isn't.
- **Not evidence that the project has shipped at scale.** It's `0.1.0`. The architecture hasn't been stress-tested by months of agent traffic and hundreds of edge cases. The lesson is about *the design choice as articulation*, not about long-term operational vindication. The latter is owed; the former is what this writeup captures.
- **Not evidence that artifact-grounded writeups are second-class.** They aren't. The architecture-as-artifact is sometimes the cleanest evidence available. What's load-bearing is *naming what the artifact demonstrates* — which is what this writeup does — not pretending the architecture has incidents it doesn't.
