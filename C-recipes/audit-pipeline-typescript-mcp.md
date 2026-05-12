# Audit Pipeline — TypeScript MCP Server (presentation-studio-mcp)

**Source aside:** Ch. 5 (eval loops are the product).
**Source incident:** `war-stories/presentation-studio-mcp-feedback-loops.md`.
**Last reviewed:** 2026-05-12.
**Valid as of:** Q2 2026, against `presentation-studio-mcp` v0.1.0 (commit `ee61fa1`, 2026-04-09).

## The body principle

Ch. 5, Principle 5: *the audit is the eval.* Probabilistic systems can't produce error signals on their own behalf, so the verification step has to live in the production loop — same shape as a unit test, run on every call, severity-graded, structured enough to act on.

For an artifact-generation tool with a probabilistic caller, the operational form is: **the audit step runs after render and before delivery, on every call, and the audit report is structured enough for the agent to iterate on without a human in the loop.**

## The recipe at one glance

`presentation-studio-mcp` is a TypeScript MCP server that turns a `DeckSpec` (JSON) into a `.pptx` file. The agent supplies meaning (bullets, narrative); the tool supplies shape (layouts, geometry, typography, branding). Hard split.

Every render call runs through twelve audit rules. Each rule emits structured `AuditIssue` objects with severity, category, and pointer to the offending slide and field. The agent reads the report on its next turn and iterates until findings are resolved.

The audit's job isn't to replace the renderer's `try/catch`. The audit's job is to surface the *valid-looking but bad* slides the renderer would happily produce — overflowing word counts, missing closing CTAs, hallucinated layouts that fell back silently. The renderer never refuses; the audit always tells.

## The durable shape (pseudocode)

The pattern, language-agnostic, in five moving parts.

```
# Pseudocode: the audit pipeline contract

# 1. The Finding shape — structured, severity-graded, pointer-bearing
type AuditIssue:
    severity      enum(warning, error)
    category      enum(density, layout, contrast, missing-field, ...)
    slideId       string
    field         string?    # which field on the slide
    message       string     # human-readable
    ruleName      string

type AuditReport:
    issues        list of AuditIssue
    summary:
        errorCount    int
        warningCount  int
        slidesAffected  int
    passed        bool       # errorCount == 0

# 2. The Rule shape — one file per rule, same interface
type Rule:
    name       string
    category   string
    apply(deck) -> list of AuditIssue

# 3. The audit pipeline — deterministic order, runs every rule, every slide
function auditDeck(deck, rules):
    issues = []
    for rule in rules:                   # deterministic order — test stability
        for issue in rule.apply(deck):
            issues.append(issue)
    return AuditReport(
        issues=issues,
        errorCount=count(i for i in issues if i.severity == "error"),
        warningCount=count(i for i in issues if i.severity == "warning"),
        slidesAffected=size(distinct(i.slideId for i in issues)),
        passed=(errorCount == 0),
    )

# 4. Severity escalation — threshold * 1.4 → error
function classifyDensity(wordCount, layoutThreshold):
    if wordCount > layoutThreshold * 1.4:
        return ERROR
    if wordCount > layoutThreshold:
        return WARNING
    return PASS

# 5. The render-then-audit pipeline — audit runs every call, before delivery
function renderAndAudit(deck):
    pptx = render(deck)                  # never refuses; falls back on unknown
    report = auditDeck(deck, allRules)
    return RenderResult(
        pptx=pptx,
        report=report,
    )
```

Five moving parts. None of them are exotic. The thing most teams skip is part 4 — the severity threshold that says *some warnings deserve to escalate, and the multiplier is a judgment baked into the code, not a runtime parameter.*

## The TypeScript implementation

> **Real implementation, as of writing 2026** — appendix B has the current SDK shape for the MCP server framework. Date context for the snapshot below: `presentation-studio-mcp` v0.1.0, TypeScript 5.x, Zod 3.x.

The Zod schema is the single source of truth:

```ts
// packages/schema/src/audit.ts
import { z } from "zod";

export const Severity = z.enum(["warning", "error"]);
export type Severity = z.infer<typeof Severity>;

export const Category = z.enum([
  "density",
  "layout",
  "contrast",
  "missing-field",
  "bounds",
  "consistency",
]);
export type Category = z.infer<typeof Category>;

export const AuditIssue = z.object({
  severity: Severity,
  category: Category,
  slideId: z.string(),
  field: z.string().optional(),
  message: z.string(),
  ruleName: z.string(),
});
export type AuditIssue = z.infer<typeof AuditIssue>;

export const AuditReport = z.object({
  issues: z.array(AuditIssue),
  summary: z.object({
    errorCount: z.number().int().nonnegative(),
    warningCount: z.number().int().nonnegative(),
    slidesAffected: z.number().int().nonnegative(),
  }),
  passed: z.boolean(),
});
export type AuditReport = z.infer<typeof AuditReport>;
```

The rule shape is one file per rule, all sharing an interface:

```ts
// packages/core/src/audit/types.ts
import type { DeckSpec, AuditIssue, Category } from "../../../schema/src";

export interface Rule {
  name: string;
  category: Category;
  apply(deck: DeckSpec): AuditIssue[];
}
```

One rule, fully worked, to show the shape:

```ts
// packages/core/src/audit/rules/textDensity.ts
import type { Rule } from "../types";

// Per-layout thresholds — "too dense" is not one number.
// Numbers as of Q2 2026 against PptxGenJS default fonts.
// Update via pixel-diff regression when typography changes.
const DENSITY: Record<string, number> = {
  "hero-cover": 40,
  "two-column-text": 140,
  "logo-wall": 30,
  "comparison-table": 140,
  // ... fifteen layouts total
};

const ERROR_MULTIPLIER = 1.4; // judgment baked in — see architecture.md decision #6

export const textDensityRule: Rule = {
  name: "textDensity",
  category: "density",

  apply(deck) {
    const issues = [];
    for (const slide of deck.slides) {
      const threshold = DENSITY[slide.layout];
      if (threshold === undefined) continue; // layoutUnknown rule handles this

      const wordCount = countWords(slide);
      if (wordCount > threshold * ERROR_MULTIPLIER) {
        issues.push({
          severity: "error",
          category: "density",
          slideId: slide.id,
          message: `Slide has ${wordCount} words; layout '${slide.layout}' caps at ${threshold} (error above ${Math.round(threshold * ERROR_MULTIPLIER)})`,
          ruleName: "textDensity",
        });
      } else if (wordCount > threshold) {
        issues.push({
          severity: "warning",
          category: "density",
          slideId: slide.id,
          message: `Slide has ${wordCount} words; layout '${slide.layout}' caps at ${threshold}`,
          ruleName: "textDensity",
        });
      }
    }
    return issues;
  },
};

function countWords(slide: DeckSpec["slides"][number]): number {
  // Implementation detail — counts across title, bullets, body, captions.
  // Omitted for brevity.
  return 0;
}
```

The audit driver — deterministic rule order, simple loop:

```ts
// packages/core/src/audit/auditDeck.ts
import type { DeckSpec, AuditReport } from "../../../schema/src";
import { allRules } from "./rules";

export function auditDeck(deck: DeckSpec): AuditReport {
  const issues = allRules.flatMap((rule) => rule.apply(deck));
  const errorCount = issues.filter((i) => i.severity === "error").length;
  const warningCount = issues.filter((i) => i.severity === "warning").length;
  const slidesAffected = new Set(issues.map((i) => i.slideId)).size;
  return {
    issues,
    summary: { errorCount, warningCount, slidesAffected },
    passed: errorCount === 0,
  };
}
```

The rule list — explicit order, no reflection-based discovery, no magic:

```ts
// packages/core/src/audit/rules/index.ts
// Order is deterministic so test fixtures stay stable.
// See auditDeck.ts line 32-43 for the canonical sequence.
import { layoutUnknownRule } from "./layoutUnknown";
import { titleLengthRule } from "./titleLength";
import { textDensityRule } from "./textDensity";
import { tooManyBulletsRule } from "./tooManyBullets";
import { imageMissingRule } from "./imageMissing";
import { imageDistortedRiskRule } from "./imageDistortedRisk";
import { elementOutOfBoundsRule } from "./elementOutOfBounds";
import { lowContrastRiskRule } from "./lowContrastRisk";
import { emptySlideRule } from "./emptySlide";
import { closingCtaMissingRule } from "./closingCtaMissing";
import { duplicateSlideIdRule } from "./duplicateSlideId";
import { inconsistentBrandUsageRule } from "./inconsistentBrandUsage";

export const allRules = [
  layoutUnknownRule,
  titleLengthRule,
  textDensityRule,
  tooManyBulletsRule,
  imageMissingRule,
  imageDistortedRiskRule,
  elementOutOfBoundsRule,
  lowContrastRiskRule,
  emptySlideRule,
  closingCtaMissingRule,
  duplicateSlideIdRule,
  inconsistentBrandUsageRule,
];
```

The MCP tool surface that exposes audit + render — the agent calls `render_deck`, which composes render and audit:

```ts
// apps/mcp-server/src/tools/renderDeck.ts
import { renderPptx } from "@psm/core";
import { auditDeck } from "@psm/core";
import type { DeckSpec } from "@psm/schema";

export async function renderDeckTool(input: DeckSpec) {
  // Render first — never refuses; falls back on unknown layouts.
  const pptxBuffer = await renderPptx(input);
  // Audit second — every call.
  const report = auditDeck(input);
  return {
    pptxPath: persist(pptxBuffer, input.deckId),
    report,
  };
}
```

> The shape is durable. The implementation is a snapshot. Don't memorize the Zod call surface or the rule-list module pattern; the patterns above will look different in three years. The five moving parts of the audit pipeline shape (Finding, Rule, deterministic driver, severity escalation, render-then-audit) are what stay constant.

## The twelve rules

Each rule lives in its own file in `packages/core/src/audit/rules/`. The shape repeats: same interface, focused scope, no cross-cutting logic.

| Rule | Category | What it checks |
|---|---|---|
| `layoutUnknown` | layout | Slide's `layout` field names a layout the renderer doesn't know. Emits warning; the renderer falls back to `two-column-text` (the layout-fallback rule). |
| `titleLength` | density | Title field exceeds per-layout title-length cap. |
| `textDensity` | density | Total slide word count exceeds per-layout threshold. 1.4× → error; over threshold → warning. |
| `tooManyBullets` | density | Bullet count exceeds layout's bullet cap (separate from word count). |
| `imageMissing` | missing-field | Slide references an image that wasn't provided. Renderer renders a placeholder; rule warns. |
| `imageDistortedRisk` | bounds | Image aspect ratio differs significantly from the layout's image slot. |
| `elementOutOfBounds` | bounds | Computed element position falls outside the slide canvas. |
| `lowContrastRisk` | contrast | Text color vs. background color WCAG contrast estimate falls below threshold. |
| `emptySlide` | missing-field | Slide has zero content fields populated. |
| `closingCtaMissing` | missing-field | Last slide of a deck doesn't have a closing CTA field populated. |
| `duplicateSlideId` | consistency | Two or more slides share an `id`. Breaks rendering tools that key on `id`. |
| `inconsistentBrandUsage` | consistency | A slide overrides brand defaults (color, font) in a way that diverges from the deck's brand registry entry. |

The twelve rules aren't sacred. The pattern is: **one rule per file, deterministic order, single-purpose, structured output.** New rules accrete easily; old rules retire cleanly.

## The severity escalation rule (the 1.4× cliff)

Every threshold-bearing rule has the same escalation logic:

- Value at or below threshold → no issue.
- Value above threshold, below `threshold * 1.4` → warning.
- Value above `threshold * 1.4` → error.

The 1.4× multiplier is a judgment baked into the code. It comes from observation: slides that exceed by ~10-20% are noticeably crowded but recoverable; slides that exceed by 40%+ are structurally broken.

The multiplier lives in code, not config, on purpose. **The threshold is the contract; the multiplier is the editorial call.** Tweaking the multiplier per-environment would erode the audit's role as the eval — the agent would start tuning to environment-specific quirks instead of writing decks that hold up everywhere.

## The agent iteration loop

What the agent does with the `AuditReport`:

1. Calls `render_deck` with a `DeckSpec`.
2. Receives `{ pptxPath, report }`.
3. If `report.passed === true`, ships the artifact.
4. If `report.passed === false`, reads `report.issues` and groups by `slideId`.
5. For each slide with issues, the agent edits the slide's spec to address the named issues (trim word count, pick a real layout, fill the missing CTA).
6. Re-calls `render_deck` with the updated `DeckSpec`.
7. Repeats until `report.passed === true` or it hits a turn budget.

The agent doesn't need to understand the rules' internals. The `message` field is human-readable enough that the agent can pattern-match it to "what should I change in the spec." The `slideId` and `field` are the pointers; the `message` is the explanation.

A small but important detail: the audit report says **what to fix**, not **how to fix it**. The agent has freedom to choose how — trim the bullets, pick a denser layout, split a slide in two. The audit is the rubric, not the solution.

## Companion patterns (from the same project)

Three patterns ship alongside the audit pipeline, all serving the same principle:

### The layout fallback

A four-line rule in `renderSlide.ts`, documented in `architecture.md`: *any unknown layout falls back to `two-column-text` with a warning.*

The agent gets a working `.pptx` plus a structured warning. The deck is never blocked on the agent's hallucinated enum value. The same shape recurs: hidden slides → warning + skip; missing images → warning + placeholder; missing CTA → audit warning, not render error.

Generalization: **for any tool input that's an enum, falling back to a default plus warning beats rejecting**, when the caller is probabilistic.

### Normalize-and-warn

A separate `normalize_deck_spec` tool that takes a partial deck, fills defaults, and emits warnings — *without rendering*. The agent can iterate on a noisy spec, see what the tool will assume, then call `render_deck` once it has a clean spec.

Generalization: **separate normalize from render** so the agent can cheaply preview the tool's interpretation of its input before paying the rendering cost.

### Don't auto-fill missing fields

If a required field is empty, the renderer leaves the space blank and emits a warning. It does not invent a placeholder. The agent sees the gap on its next pass and fills it correctly — instead of the deck shipping with "Untitled Slide" because the renderer was helpful.

Generalization: **render the gap visibly**. Auto-filling teaches the agent that missing fields are fine.

All three patterns ride on the same principle: the agent needs *visible signal* about what the tool couldn't honor, not a hidden defaulting of decisions the agent should be making.

## Porting to other stacks

The five moving parts (Finding, Rule, deterministic driver, severity escalation, render-then-audit) port unchanged.

**Ruby on Rails (Forge-shape sketch).** Findings as plain Hashes or Dry::Struct; rules as classes with `def apply(deck)`; a `Audit::Pipeline.run(deck)` aggregator; the same 1.4× cliff. ActiveSupport instrumentation emits `audit.deck.completed` with the report as the payload, picked up by the metric backend (see [`observability-across-stacks.md`](observability-across-stacks.md)). The Rails-side wiring is what changes; the audit's shape doesn't.

**Python.** Pydantic for the Zod equivalent; one module per rule; an explicit `RULES` list for deterministic order; the audit returns a dataclass `AuditReport`. The pipeline is a generator: `for rule in RULES: yield from rule.apply(deck)`.

**Any stack.** The non-negotiables are: rules in their own files (test stability), explicit rule list (no reflection-based discovery), structured findings (severity + category + pointer + message), deterministic order, and the render-then-audit composition. Anything else is stack-specific affordances.

## Counter-examples — when this pattern doesn't apply

- **Pure deterministic transforms.** A library that crops images, validates JSON, or formats currency doesn't need an audit pipeline. The transform either succeeds or raises; there's no "valid-looking invalid output" failure mode.
- **Human-only callers reviewing every output.** A design tool used by humans who see every artifact before shipping has the human as the audit. Engineering a structured audit pipeline is a tax with no payback.
- **Outputs without an external standard.** The audit pipeline works because there's a concept of "this slide is too dense for this layout" — a checkable standard. Outputs whose quality is purely subjective (creative writing, brand voice) need a different eval shape (rubric scoring, taste pairs, human-judged sets).

## What survives the implementation changing

The principle: *the audit is the eval, run on every call, severity-graded, structured enough to iterate on.* The Zod calls, the file paths, the rule names, even the 1.4× multiplier — all dated specifics. The five moving parts of the pipeline shape, the deterministic order, the per-call execution, the agent-readable report — those are what survive.

## Cross-references

- Ch. 5 (Eval Loops Are the Product) — full chapter treatment of the *audit is the eval* principle.
- Ch. 9 (Agent vs. Script vs. Human) — the deterministic-tool-with-probabilistic-caller shape this recipe is a worked example of.
- Ch. 10 (When to Trust the Output) — the audit pipeline as the runtime trust signal for artifacts.
- `war-stories/presentation-studio-mcp-feedback-loops.md` — the source incident behind the recipe.
- `war-stories/presentation-studio-mcp-source.md` — the verbatim self-interview that grounds these specifics.
- [`observability-across-stacks.md`](observability-across-stacks.md) — how to emit the audit report as observable telemetry.
- [`runtime-trust-patterns.md`](runtime-trust-patterns.md) — the trust-the-check-not-the-claim discipline this recipe operationalizes.
