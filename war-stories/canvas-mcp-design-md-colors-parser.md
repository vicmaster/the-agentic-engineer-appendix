# canvas-mcp / Box Shadows in the Colors Map

## Date / Version Context

- **Date:** Latent throughout canvas-mcp's lifetime. The parser shipped with the first version of the `import_design_md` tool (Phase 1, 2026-03). The bug was caught by eyeballing a screenshot during dogfooding sometime in Phase 2 or 3. Still pending as of the 2026-05-09 self-interview — captured in `VISION.md` as `[ ] DESIGN.md parser: filter out non-color values (e.g. full box-shadow strings) from colors map`.
- **Project:** canvas-mcp — open-source MCP server for AI-driven design mockups. TypeScript monorepo, vanilla canvas rendering. The `import_design_md` MCP tool ingests a `DESIGN.md` file from a community-curated design repository (a separate GitHub project that aggregates design specs from 55+ contributors). The parser converts the Markdown spec into the `Canvas` type the renderer consumes.
- **Surface for this story:** the `DESIGN.md → Canvas` parse step. Specifically, the `colors` section's value-shape: the parser accepts `Record<string, string>` (any string-to-string map under the `colors` key) without semantic validation of whether the values are actually colors.
- **Glossary, used in this writeup:** *Import boundary* = the point where external input enters the system's typed world (here: the `DESIGN.md` parser turning text into typed structures). *Format-level validation* = a check on the input's structural shape (is this a string? is this a map?). *Semantic-level validation* = a check on the input's meaning (is this string actually a CSS color?). *Community input* = data produced by multiple authors who don't share a single mental model of the format's intent.

## What Was Being Attempted

Ingest a community-curated `DESIGN.md` into canvas-mcp's renderer.

The motivation was scale. canvas-mcp's first version had 15 layouts and 7 templates, hand-authored. The next version was supposed to scale by accepting design specs from a community repo — the AI-friendly-design-systems project, with 55+ contributors each maintaining a `DESIGN.md` for their own design system (Material, iOS, Bootstrap, in-house brand systems, etc.). The `import_design_md` tool ingests one of those files and produces a `Canvas` the renderer can use.

The format was reasonable. `DESIGN.md` has named sections — `## Colors`, `## Typography`, `## Spacing`, `## Components` — each containing a structured block of key-value pairs. The Colors section follows a convention like:

```markdown
## Colors

- primary: #4a90e2
- secondary: #50e3c2
- text-primary: #2d3748
- text-secondary: #718096
- error: #e53e3e
```

Straightforward. Hex codes, named keys. The parser converts this to `{ "primary": "#4a90e2", "secondary": "#50e3c2", ... }` and the renderer uses it for fill colors, stroke colors, text colors.

The implementation matched the format. The parser walks the section, splits each line on `: `, and produces a `Record<string, string>` for `colors`. No further validation. The format said *string-to-string map*; the parser produced a *string-to-string map*. The contract held.

## What Went Wrong

Some `DESIGN.md` authors put box-shadows in the colors map.

A real `DESIGN.md` had entries like:

```markdown
## Colors

- primary: #4a90e2
- text-primary: #2d3748
- button-shadow: 0 4px 12px rgba(0,0,0,0.15)
- card-elevation: 0 2px 4px rgba(0,0,0,0.08)
```

`button-shadow` and `card-elevation` are CSS box-shadow strings — multi-value shorthand declarations with offsets, blur radius, and color. They're not colors. They're *shadows* that happen to live in the "colors" section because the author's mental model was *colors is the section for visual-style values*, not *colors is the section for color values*.

The parser accepted both. The format-level check (*string-to-string map*) passed. The renderer received `colors["button-shadow"] = "0 4px 12px rgba(0,0,0,0.15)"`. The renderer tried to use that string as a fill or stroke color. CSS parsers in the renderer accepted the string at the property-set level (the rendering library doesn't reject invalid color strings; it falls back to a default or renders nothing). The visual output was a silently broken node — a button with no shadow because the value was treated as a color and failed to render, or a card with a wrong fill because the renderer fell back to black when the color string didn't parse.

The shape of the bug:

- **Format-level validation passed** because the input shape matched (string-to-string map).
- **Semantic-level validation was absent** because no check asked *is this string actually a CSS color?*
- **The renderer didn't fail loudly** because rendering libraries are permissive — they'd rather show wrong output than crash.
- **The output was visually broken** but not structurally errored, so unit tests couldn't catch it (every test passed; the canvas still rendered).
- **The discovery channel was a human looking at a screenshot** and noticing the button looked wrong, then tracing back to the imported colors map.

The implicit trust the bug exposes: the parser was trusting 55+ `DESIGN.md` authors to share a single mental model of what `## Colors` means. They didn't. Some authors used "colors" for any visual-style key; others used it strictly for color values; a few used it as a catch-all for any string they didn't have a section for. The format's *intent* was clear-to-the-format-designer; the format's *interpretation by community authors* was not consistent.

## How It Was Discovered

By eyeballing a screenshot.

The operator imported a `DESIGN.md`, rendered a canvas with it, and noticed the visual output didn't match what the design system was supposed to look like. The screenshot looked off — a button with no shadow, a card with a wrong fill. The operator opened the imported `Canvas` object, looked at the `colors` map, and saw the box-shadow strings sitting alongside the hex codes. The cause was immediate once the operator looked.

The discovery channel is worth pausing on. Standard validation tooling caught nothing. The schema validation passed because the schema was *Record<string, string>*. The renderer didn't error because rendering libraries are permissive. CI tests passed because the synthetic test fixtures used `{ "primary": "#4a90e2", ... }` — no box-shadow strings, because the developer writing the tests had the *correct* mental model of what belongs in `colors`. Real-world authors had the *wrong* mental model; the tests couldn't see this because the tests came from the developer's mental model too.

This is the same shape as the channel-ID regex and the IntentParser stories: **synthetic test data and the synthetic check both come from the developer's mental model, so the tests can't catch the bug when real-world input doesn't match the model.** The fix-shape varies (LLM-as-fallback, resolve-at-source, semantic-validation-at-import); the bug-shape is identical.

## What Fixed It

Nothing yet, and that's worth being honest about. The fix is sitting in `VISION.md`:

```markdown
- [ ] DESIGN.md parser: filter out non-color values (e.g. full box-shadow strings) from colors map
```

The pragmatic fix would add a CSS-color regex at the import boundary: accept values that match `^#[0-9a-fA-F]{3,8}$|^rgb\(...\)$|^rgba\(...\)$|^hsl\(...\)$|^hsla\(...\)$|^[a-z]+$` (named colors), reject everything else. The rejected entries get a warning in the import log; the renderer never sees them. This is straightforward — half a day of work plus tests.

The structural fix is broader: **every section of `DESIGN.md` deserves semantic validation at the import boundary, not just colors.** Spacing values that should be CSS lengths but aren't. Typography that should be valid font shortcuts but isn't. Component definitions that reference shapes the renderer doesn't know. Each section has the same shape — format-level validation insufficient, semantic-level validation needed.

The longest-term fix is shifting from "permissive parser, permissive renderer" to "validating parser, strict renderer." The parser becomes the gate that accepts only well-formed semantic content; the renderer assumes its input is correct because the parser already filtered. Build the validation into the schema (CSS-color-typed values for colors, CSS-length-typed values for spacing, etc.) and reject mismatches with a clear import-time error that names *which entry from which author's DESIGN.md was rejected and why*.

What didn't get attempted: trying to make the renderer recover from bad colors. The lesson rejects that direction. A renderer that absorbs wrong input silently produces broken output silently. The fix has to live at the import boundary, where the cost of rejection is *one warning in the import log* and the cost of acceptance compounds across every render that depends on the corrupted map.

## The Durable Lesson

**Validate at the import boundary, not in the renderer.** A parser that accepts "the format" from a community-curated repo is implicitly trusting 55+ authors to agree on what a value means. They don't.

The deeper observation: **format-level validation catches type drift; semantic-level validation catches meaning drift. Community input always brings both.** The parser's job in a community-input system isn't *accept anything that matches the format*; it's *accept anything that the format intends*. The two are different, and the gap between them is where every community-input parser eventually breaks.

The mental-model flip the lesson rests on: **the format is a contract between the parser and the input's *intent*, not between the parser and the input's *shape*.** A `## Colors` section is contractually about color values, not just about string keys with string values. The parser that enforces only the shape is enforcing a weaker contract than the format intends. The user-visible failure is the gap between the two contracts.

> **Heuristic.** For any parser ingesting community-authored input, ask: *what does each section semantically mean, and what's the validation that confirms a value belongs there?* If the validation is *"it's a string"*, the section's name is doing the semantic work — and the section's name has no enforcement. Add an explicit semantic check at the import boundary: CSS-color regex for colors, length-regex for spacing, named-shape-enum for components. Warn on rejection. Don't let the renderer see entries the parser couldn't validate semantically.

The shape generalizes:

- **YAML configs with typed sections.** A `database:` block intended for connection strings that gets a feature flag pasted into it. Format-level (it's a string-to-string map) passes; semantic-level (it's a connection string) doesn't. The same fix shape applies.
- **JSON schemas accepting `additionalProperties: true`.** Permissive schemas let community authors put anything in. Strict schemas with `additionalProperties: false` plus per-field semantic validation catch the meaning drift at parse time.
- **Markdown frontmatter with extensible keys.** A blog platform's frontmatter that supports `tags`, `category`, `featured` — authors put random custom keys in because the platform accepts them. Half the keys are meaningful; half are typos or one-off ideas that never got cleaned up.
- **CSV imports with named columns.** Column `email` that contains phone numbers because the source spreadsheet had a misaligned column. Format-level (it's a string) passes; semantic-level (it's an email) doesn't.
- **REST API request bodies.** A field named `color` that accepts any string. Caller passes `"button-shadow"` because their mental model of what `color` means differs from yours.

In every case, the rule is the same: **the format names the intent; the validation has to enforce the intent, not just the shape.**

The pairing worth naming: this is the *community-input* version of the companion stories `forge-channel-id-regex.md` and `forge-keyword-router-natural-language.md`. All three are *synthetic-validation-drifts-from-reality* stories:

- **The IntentParser** (`forge-keyword-router-natural-language.md`) — a keyword parser assumed users speak in keywords. Tests passed because fixtures were keywords; real users typed natural language.
- **The channel-ID regex** (`forge-channel-id-regex.md`) — a regex assumed Slack IDs start with `C`. Tests passed because fixtures were `C…`; real Slack issues `G…` too.
- **This story (DESIGN.md parser)** — a parser assumed authors agree on section semantics. Tests passed because fixtures had the right kinds of values; real authors used "colors" for any visual-style value.

Three sources of synthetic mismatch — user input, external-system format, community-authored schema — all caught by the same structural error: **the synthetic test data and the synthetic check came from the developer's mental model.** The fix-shapes differ (LLM-as-fallback, resolve-at-source, semantic-validation-at-import), but the underlying discipline is the same: **don't let the deterministic check stand alone when the input source can produce things outside your model.**

## Counter-Example — When This Lesson Doesn't Apply

The lesson is about *parsers ingesting input from sources that produce semantically-varying values within a fixed format*. It doesn't apply to:

- **Parsers ingesting input from a single author you control.** A `config.yaml` written by the same team that wrote the parser doesn't need semantic-level validation at the parse boundary — the team's review process and tests already enforce the intent. The lesson applies when *the parser and the authors are different parties*.
- **Single-use one-off imports.** A scratch script that ingests one specific file once doesn't justify the validation layer. Just look at the output. The lesson is for parsers that ingest many files from many sources over time.
- **Parsers where invalid values fail loudly downstream.** If the renderer would crash on a box-shadow-as-color (instead of silently producing wrong output), the loud failure is its own validation. The cost of catch-it-downstream is bounded by the loudness; the lesson applies when downstream failure modes are *silent*.
- **Cases where semantic validation is genuinely too expensive.** A natural-language description field that the parser accepts as-is because validating "is this a meaningful sentence" requires an LLM call you don't want to pay for. The fix in those cases is acknowledging the absence — document the field as freeform, don't add it to a typed-section that implies otherwise.

The signal: *if a community author put something semantically wrong in this section, would the system fail loudly, fail silently, or accept it as valid?* If the answer is *fail silently or accept as valid*, the validation gap is where the bug lives.

## What This Story Is *Not* Evidence For

- **Not evidence that community input is bad.** Community-curated design systems are exactly the reason canvas-mcp can scale beyond hand-authored templates. The bug isn't accepting community input; it's accepting community input *without semantic validation at the boundary*.
- **Not evidence that the DESIGN.md format is flawed.** The format is fine for its intent. The bug is at the parser, not the format. Other parsers could handle the same format better.
- **Not evidence that the authors who put box-shadows in colors are wrong.** Their mental model — *colors is the section for visual-style values* — is internally consistent. The format didn't say it wasn't. The fix isn't to fight the authors' interpretation; it's for the parser to filter at the boundary so the renderer doesn't see semantically-mismatched values.
- **Not evidence that JSON Schema or other strict-schema tools would have caught this.** They would have caught it *if the schema declared `colors: Record<string, CSSColor>` instead of `Record<string, string>`*. The bug is the schema not encoding semantic types, not the absence of schema tooling. Strict tools with weak schemas have the same failure mode.
- **Not evidence that the renderer needs to be made stricter.** Making the renderer reject unknown color strings would surface the bug — but at the rendering layer, where the cost of rejection is higher (a broken canvas instead of a clean import warning). The structural fix lives at the import boundary, not the render boundary.
