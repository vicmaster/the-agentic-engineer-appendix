# LLM

**One-line:** Large language model — the probabilistic component the book is built around.

**First introduced:** Ch. 1, *The Shift*.
**Load-bearing in:** every chapter; the book's central probabilistic component.

## The book's use

*LLM* is shorthand for "large language model": a transformer-class model trained on text that, given an input prompt, samples a probability distribution to produce an output. The book treats every LLM call as a *probabilistic* call site — same input, different output, every time — and that property is what drives most of the design decisions the chapters argue for.

## What the term deliberately includes and excludes

- **Includes:** any hosted or self-hosted large language model, regardless of provider, modality (text, vision, code), or size. The principles in the body don't depend on which.
- **Excludes:** smaller deterministic models (rule-based NLP, classical classifiers, embedding-only models). Those are deterministic enough that the failure shapes this book names mostly don't apply.

## Why the book doesn't name specific models

By editorial choice (see Ch. 2, *The Reader Contract*). The frontier of LLMs moves every few months; naming a model in the body would date the book within a year. Specific models, their stop reasons, their default budgets, their pricing, and their SDK shapes live in appendix A and B.

## Last reviewed

2026-05-12.
