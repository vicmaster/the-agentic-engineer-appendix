# Eval harness

**One-line:** The test infrastructure most teams build to grade probabilistic outputs against a golden set, typically offline in CI.

**First introduced:** Ch. 5, *Eval Loops Are the Product*.
**Load-bearing in:** Ch. 5, Ch. 8 (observability — forward-pointers to replay), Ch. 11 (only as a *contrast* — see below).

## The book's use

An *eval harness* is the offline counterpart to the online audit pipeline Ch. 5 argues for. Traditional setup:

- A *golden set* of representative inputs.
- Expected outputs (or expected-shape rubrics) the inputs should produce.
- A runner that calls the model under test and scores each output against the rubric.
- A pass/fail or score threshold that gates promotion.

This is the pattern borrowed from ML evaluation. The book's argument in Ch. 5 isn't *don't build one*; it's that **the offline eval harness alone is not sufficient** for probabilistic systems where every production call is a fresh sample. The audit pipeline runs *online*, on every call. The eval harness runs *offline*, on representative inputs. Both have their place; treating either as the whole answer is the mistake.

## Why this gets a separate term from *harness*

The two senses collide because the word *harness* gets reused.

- **Agent harness** (see [harness](harness.md)) — the runtime that hosts an agent loop.
- **Eval harness** (this entry) — the test infrastructure that grades model outputs.

They are unrelated. When in doubt, the chapter context disambiguates: Ch. 4 and Ch. 7 use *harness* to mean the agent harness; Ch. 5 and Ch. 8 use *eval harness* (full phrase) for the testing infrastructure. (Ch. 11 mentions the term only to *reject* it — see below.)

## What Ch. 11 does with the term

Ch. 11 deliberately *separates* itself from *eval harness* rather than generalizing it. The team-side question it names is **who owns the definition of done** — and the artifact that answers it is a **work-authorization gate**, explicitly *not* an eval harness: it doesn't judge whether the output is any good, it decides whether the work is allowed to proceed. That gate can be a Markdown file the agent and the team both read. The two are related but distinct — an eval harness grades output *quality*; a work-authorization gate governs whether work *ships*. Ch. 11 uses *eval harness* only as the contrast that makes the distinction clear.

## Common cross-references

- The *audit pipeline* in Ch. 5 is the online complement to the offline eval harness.
- A *golden set* is dated to the version of the system that produced it; expect to refresh it as the system changes.

## Last reviewed

2026-05-12.
