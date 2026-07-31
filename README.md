# Agentic Dev Book — Living Appendix

This is the dated half of *The Agentic Engineer*. The book's body holds durable principles; this appendix holds the time-sensitive specifics that motivate them — model names, SDK shapes, harness limits, current implementations, current pricing context.

The two halves live on different cadences. The body changes slowly because principles change slowly. The appendix changes whenever the field moves underneath it.

## Structure

- **A-models** — frontier model landscape: which models exist, default budgets, capabilities, limitations, current pricing context.
- **B-sdks** — SDK examples per provider; API field names; harness behaviors and limits; per-provider quirks the body refers to in dated callouts.
- **C-recipes** — tool-specific recipes, audit pipelines, observability stacks, team primitives — the runnable patterns that show how a body principle lands in a given stack.
- **D-glossary** — canonical, slightly-tighter-than-casual definitions for the book's load-bearing vocabulary.
- **war-stories** — the real incidents behind the book. Each `war-stories/*.md` pointer in the body names one of these self-contained writeups: what was attempted, what went wrong, how it was found, what fixed it, and the durable lesson. Companion narrative material, not dated reference — the tools in the stories are dated, the failure shapes aren't. See [war-stories/README.md](war-stories/README.md).

## How chapters cross-reference the appendix

Chapters never deep-link to appendix URLs. The cross-reference is always by section letter — "appendix B has the current SDK," "appendix C carries the audit-pipeline implementation per stack." Readers find the entry by topic.

Every `::: aside` "as of writing" callout in the body is a pointer into this appendix. If the body says *as of writing (2026), X was true*, the appendix is where you go to check whether X is still true.

## Conventions

- Entries are dated. Every entry has a `Last reviewed:` line, and where relevant, a `Valid as of:` date range.
- One entry per topic per provider. If a pattern works the same across three providers, it gets one entry with provider-specific subsections, not three entries.
- Code examples in entries lead with a one-line context note that names the stack, the date, and the SDK version. Readers should be able to tell at a glance whether the snippet is current to their setup.
- The appendix accepts contributions. See `C-recipes/README.md` for contribution guidance; tool-specific recipes especially are the kind of thing that accumulates faster from a working community than from one author.

## Status

This is a living document, last reviewed 2026-07-31, shipped alongside the book's first public release. The `war-stories/` collection is complete — 40 incident writeups across 8 projects. The **A-models**, **B-sdks**, **C-recipes**, and **D-glossary** sections are seeded from the manuscript's dated `::: aside` callouts and deepen over time; that's the point of splitting them from the body, so they update on their own cadence as models, SDKs, and harnesses move underneath. Every entry carries a `Last reviewed` date so you can tell at a glance how current it is.

## Access

This repository is public. The book's front matter carries the canonical URL; in a print copy it's on the inside back cover. Tool-specific recipes especially accumulate faster from a working community than from one author — see `C-recipes/README.md` for contribution guidance.
