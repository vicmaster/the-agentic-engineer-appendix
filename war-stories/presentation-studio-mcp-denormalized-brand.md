# presentation-studio-mcp / The Brand That Wasn't There

## Date / Version Context

- **Date:** The original design used `brand.id` references; the denormalization fix landed at `buildDeckSpec.ts:35` (the line that copies brand properties into the deck spec). The specific commit isn't recorded, but the fix predates the project's stable build state — the surprise benefit (portability across registry drift) became visible only after the denormalization was in place for long enough to outlive a brand-registry change.
- **Project:** presentation-studio-mcp — local MCP server that renders structured DeckSpecs into `.pptx` files. The architecture has *the agent decides what to say; the tool decides geometry, spacing, typography, branding* (see the companion story `presentation-studio-mcp-agent-tool-split.md` for the broader framing). The brand registry was originally a server-side concept — the agent passed a brand id, the tool looked up the brand at render time.
- **Surface for this story:** the DeckSpec / brand boundary. Pre-fix: DeckSpec has `brand: { id: "magma-2026" }` and the tool resolves `id → full brand object` at render time. Post-fix: DeckSpec has `brand: { id: "magma-2026", name: "...", colors: {...}, fonts: {...}, logo: {...} }` — a full snapshot of the brand at the moment the deck was built.
- **Glossary, used in this writeup:** *Normalized reference* = an artifact that stores a foreign-key-shaped pointer (an id, a URL, a slug) into a separate source of truth. The artifact is small; the lookup happens at consumption time. *Denormalized snapshot* = an artifact that stores a copy of the referenced data inline. The artifact is larger; no lookup is needed at consumption time. *Registry drift* = the source of truth changing (additions, removals, mutations) after the referencing artifact was built. *Agent-side state* = the data the agent has access to when it builds an artifact. *Server-side state* = the data the tool has access to when it consumes the artifact. The two can diverge silently.

## What Was Being Attempted

Build a deck spec that the renderer can turn into a branded `.pptx`.

The product framing was straightforward. The agent writes a DeckSpec — slides, layouts, content, branding. The renderer consumes the spec, applies the layouts, fills in the content, applies the brand (colors, fonts, logos), produces the `.pptx`. The brand was conceptually separate from the deck — multiple decks could share a brand, the brand's properties shouldn't be repeated across every deck spec, normalization is the obvious shape.

The original design followed the normalized-reference pattern. The brand registry — a server-side table or JSON file — held the brand objects keyed by id. Each brand object had its full set of properties (colors, fonts, logo URLs, type scale, spacing tokens). The DeckSpec held only `brand: { id: "magma-2026" }`. At render time, the tool resolved the id against the registry, pulled the full brand object, applied it to the slides.

Three reasons this design *looked* clean:

- **Storage is smaller.** A DeckSpec with `brand: { id: "magma-2026" }` is tiny. A DeckSpec with the entire brand object inline is dozens of fields larger.
- **Updates propagate.** Change the brand's primary color in the registry, and every deck that references the brand id renders with the new color on next render. The single-source-of-truth instinct.
- **The agent doesn't have to know brand internals.** The agent passes the id; the tool handles the lookup. The agent's mental model is *"this deck uses the Magma 2026 brand,"* not *"this deck uses these specific colors and fonts."*

Each of these reasons has a counter-argument that became visible only after the system was in production.

## What Went Wrong

The agent wrote a deck spec referencing a brand id that wasn't there at render time.

The exact scenario is one of two structurally identical paths, both of which reduce to the same thing: *the agent wrote a deck spec that referenced a brand id that wasn't registered (or got dropped).*

**Path 1 — never registered.** The agent named a brand it had been told about (in a prompt, a system message, a prior conversation turn) but that the registry had never had. The agent's mental model of the brand catalog drifted from the registry's actual contents. The DeckSpec's `brand.id` referenced a brand that existed only in the agent's belief, not in the server's database.

**Path 2 — got dropped.** The brand was once in the registry but had been removed, renamed, or migrated away by the time the deck rendered. The DeckSpec's `brand.id` was correct at the moment the agent wrote it; the registry diverged afterwards.

Either way, the render-time lookup returned nothing. The renderer's two options at that point:

- **Fail loudly.** Throw an exception, refuse to render, surface the missing brand back to the agent. This is the strict-reject pattern — the same one `presentation-studio-mcp-feedback-loops.md` names as the *wrong* pattern for probabilistic callers. The agent passes a broken DeckSpec; the strict-reject doesn't help the agent recover, doesn't produce a usable artifact, doesn't even produce a reviewable preview.
- **Fall back silently.** Use a default brand (some bland Magma fallback, or a generic *"unbranded"* template). This produces a render but the deck looks wrong, and the agent has no signal that anything went sideways — the render succeeded; the artifact came back.

Both options are bad. The strict-reject breaks the agentic loop; the silent fallback ships wrongness. The underlying issue is the same: **the deck spec's correctness depended on registry state the agent couldn't see.** The agent did everything right by its own lights — wrote a valid spec, referenced a brand by its known id — but the spec's validity was a function of *the registry at render time*, which is a different system from the one the agent has visibility into.

The structural shape is blunt: *don't separate "the agent's view of state" from "the tool's view of state" without a snapshot mechanism.* The pre-fix design had exactly that separation — the agent saw brand ids (its own naming surface); the tool saw the registry (the server's data layer); a brand id valid at agent-write time was not guaranteed to be valid at tool-render time, and the gap between the two was where the failure lived.

## How It Was Discovered

By a render failure on a deck the agent had just built.

The discovery channel is *the render breaks while the agent is iterating*. The agent runs `build_deck_spec` (producing a DeckSpec), then `render_deck` (producing the `.pptx`). The render fails because the brand id isn't in the registry. The agent gets back an error (or, worse, a wrong-looking deck). The operator notices.

The non-discovery channel — the part worth pausing on — is that *standard MCP-tool testing wouldn't have caught this*. The tests test the renderer against a fixture DeckSpec, and the fixture's brand id exists in the test registry. The tests pass. The bug only surfaces when *the agent's belief about the registry* diverges from *the registry's actual contents*, which is a runtime-environment property, not a code-correctness property. Tests can't model *what the agent might believe about a registry that doesn't include it in the test data*.

The conceptual root cause is similar to `leads-crm-directory-linking-backfill.md`: the spec doesn't cover the production state of a referenced system. In that story the referenced system was the existing-data shape (production already had rows the new feature didn't know about); here the referenced system is the brand registry (the registry may have drifted between when the agent wrote the spec and when the tool rendered it). Both stories are *the agent's completion criterion was satisfied by the spec, but the spec's correctness depends on a runtime-environment property the spec doesn't describe.*

## What Fixed It

Denormalize the brand into the DeckSpec.

The change to `buildDeckSpec.ts:35` is structural rather than line-counted. The function that produces a DeckSpec now does an additional step: *resolve the brand id at build time, copy the brand's full object into the deck spec.* The resulting DeckSpec contains:

```json
{
  "slides": [...],
  "brand": {
    "id": "magma-2026",
    "name": "Magma 2026",
    "colors": { "primary": "#...", "secondary": "#...", ... },
    "fonts": { "heading": "...", "body": "..." },
    "logo": { "url": "...", "width": ..., "height": ... },
    "spacing": {...},
    "typeScale": {...}
  }
}
```

The deck is larger. The renderer doesn't need to look anything up — every property it needs is in the spec it was handed. The agent doesn't need to know brand internals (the *build* step does the lookup once, on the agent's behalf, before the agent has even handed off the spec). The render-time lookup-against-registry path is gone.

Three benefits, in order of expectation:

**Predictability at render time.** The renderer can never fail-because-brand-isn't-found, because the brand is always in the spec. The strict-reject vs. silent-fallback dilemma vanishes.

**Portability across systems.** A DeckSpec is now a self-contained JSON document. You can save it, email it, commit it to a repo, replay it months later — even if the brand registry has changed in the meantime, the deck still renders the way it did the day it was built. The surprise: this was *not* the primary reason for the denormalization (the registry-drift incident above was), but it became visible as the *bigger* benefit once the denormalization was in place long enough for a registry change to occur.

**Snapshot semantics.** The deck encodes *what the brand was at the moment the deck was built*. This is the right semantics for an *artifact* (a thing that should be reproducible exactly) but not for a *live document* (a thing that should reflect the current state of the world). The DeckSpec is structurally an artifact; the denormalization makes that explicit.

What's owed but not yet shipped: a *brand-version field* in the denormalized snapshot, so future-Victor (or another reader) can answer *"which version of the Magma brand is this deck rendering against?"* without diffing against the registry. Currently the snapshot is brand-content but not brand-version. Worth adding for archival decks.

## The Durable Lesson

Don't make your persisted artifacts depend on registry state the agent can't see. **The agent's view of state and the tool's view of state can diverge silently when the artifact is a normalized reference; denormalize-at-snapshot-time closes the divergence.**

The mental-model flip the lesson rests on: the normalization-vs-denormalization choice is usually framed as a *storage / freshness* trade-off — normalize for less storage and propagating updates, denormalize for faster reads and no joins. That trade-off is real for *live data*. For *agentic artifacts*, a different trade-off dominates: **the agent's belief about state and the tool's access to state can diverge across the agent's write and the tool's read.** If the artifact references state by id, the divergence is invisible until render. If the artifact contains a snapshot, the divergence is impossible because there's nothing to look up.

For agentic systems specifically, the cost of normalized references compounds. The agent's mental model of *what's in the registry* drifts over time — through prompt updates, conversation context, training data staleness, partial information from prior turns. The registry's actual contents drift over time — through edits, migrations, removals, additions. The two drifts are uncoupled. A normalized reference assumes the two will stay aligned; they won't.

> **Heuristic.** For any persisted artifact an agent produces that references data from a separate registry / catalog / system-of-record (brands, tenants, users, projects, products, configurations), prefer *snapshot at artifact-build time* over *reference at artifact-render time*. The artifact stores a copy of the referenced data inline; the consumer doesn't look anything up. The cost is larger artifacts; the benefit is artifacts that survive registry drift, are independently reproducible, and don't have agent-belief-vs-server-state divergence as a failure mode. If you genuinely need updates-propagate semantics (the artifact should reflect the *current* brand, not the brand at build time), keep both — the snapshot for portability, the reference for refresh — but the snapshot is the canonical artifact; the reference is the optional re-resolve.

The shape generalizes far beyond brand registries:

- **User preferences embedded in a generated report.** Snapshot the user's preferences at report-build time. If the user changes preferences afterward, prior reports are still accurate to the moment they were built.
- **Org/tenant config in a billing artifact.** Snapshot the org's plan, pricing, and discount tier into the invoice itself. Don't make invoices look up the current plan, because plans change.
- **Product catalog data in an order record.** Snapshot the product's name, description, and price into the order. The catalog can change; the order is what was actually purchased.
- **Configuration in a build artifact.** Snapshot the config used to produce a build into the build's manifest. The config can change; the build is reproducible from its own manifest.
- **External-system data in an audit log.** Snapshot the external state the system saw at the moment of the audited event, not a pointer to where it can be re-fetched. External systems change; audit logs need to be stable.

In every case, the pattern is identical: **artifacts are stable promises about a moment in time; live registries are unstable representations of the current state. Don't conflate the two by making the artifact a pointer into the registry.** The fix is snapshot-at-build-time; the cost is artifact size; the win is artifact independence from registry drift.

The pairing worth naming: this is the *registry-drift* version of the *agent-spec-doesn't-cover-the-world* pattern. Pairs with `leads-crm-org-scoping.md` on *application-layer assumptions silently violated by data that didn't go through the normal path* — there the violation was UI-enforced invariants leaking when a second surface lands; here it's deck-render correctness depending on registry state the agent can't see. Both stories are *agent-side state vs. server-side state* divergence at different layers. Pairs also with `leads-crm-directory-linking-backfill.md` on *the agent's completion criterion doesn't include the production-state of a referenced system* — there it was existing rows in a table; here it's brand entries in a registry. Both are *spec-doesn't-cover-the-world* stories at different boundary layers.

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *persisted artifacts that reference state from a separate system-of-record*. It doesn't apply to:

- **Live documents that should reflect current state.** A user-profile page that always shows the user's current name and avatar shouldn't snapshot the name and avatar — it should read them live. The lesson is for artifacts (decks, reports, invoices, orders) that should be stable representations of a moment, not for live UI.
- **References to data that's immutable by design.** If a registry guarantees no changes (a hash-content store, a versioned schema, an immutable id), the reference is safe — the lookup at consumption time is guaranteed to return the same thing. The lesson is for mutable / drifting registries.
- **Cases where the artifact's size cost is unacceptable.** Embedding a 5-MB brand asset into every deck might not be a good trade. The fix is *embed the brand metadata; reference the assets by URL* (a hybrid). The lesson is to *think about what should be snapshot vs. referenced*, not to *snapshot everything blindly*.
- **Systems with strong eventual-consistency contracts between agent and registry.** If the registry is the agent's *only* source of truth and the agent always reads-then-writes, the divergence-on-write can be small enough not to matter. The lesson is loudest in systems where the agent's view comes from elsewhere (prompts, training data, prior conversation) and the registry is a separate source.
- **Cases where you want updates to propagate retroactively.** Some scenarios *want* the artifact to reflect later changes — a contract template whose footer text updates when legal changes the standard text, for instance. The lesson is for artifacts that should be stable; if you want propagation, normalize.

The signal: *is the artifact a promise about a moment in time, or a window into current state?* If a moment-in-time promise, snapshot. If a window-into-state, reference. Most agent-produced artifacts are moment-in-time promises (the deck *as the agent built it*, the report *as of the build*, the invoice *for this purchase*); reference-shaped storage misfits them.

## What This Story Is *Not* Evidence For

- **Not evidence that normalization is wrong.** Live operational data should usually be normalized — a user's current preferences, a tenant's current plan, a product's current price all benefit from single-source-of-truth semantics. The lesson is for *persisted artifacts* that should be stable, not for all data.
- **Not evidence that registries are bad.** The brand registry is fine. The issue was *how the deck spec referenced the registry*, not the registry's existence. The fix kept the registry; it changed how artifacts depend on it.
- **Not evidence that the agent was being careless.** The agent did exactly what the design asked — pass a brand id. The bug was in the *design assumption* that the id would resolve at render time, not in the agent's behavior. The fix changed the design assumption.
- **Not evidence that all DeckSpec fields need to be denormalized.** The lesson is *snapshot the registry-resolved fields, not every reference*. A DeckSpec field that points to a stable, deterministic resource (a versioned layout name, an immutable component id) can stay normalized. The lesson is about *registry-drift-vulnerable* references specifically.
- **Not evidence that the portability surprise was the primary motivation.** The registry-drift incident drove the fix; the portability benefit was a downstream surprise. The lesson is *the fix turned out to have a bigger benefit than the immediate bug it addressed* — which is a common shape for structurally sound fixes, but the *original* motivation was the immediate incident, not the eventual surprise.
