# Agent

**One-line:** The loop around a model — code that takes a probabilistic output, decides what to do with it, and feeds the result back as input to the next call.

**First introduced:** Ch. 1, *The Shift*.
**Load-bearing in:** Ch. 4 (tool surfaces), Ch. 7 (orchestration), Ch. 9 (agent vs. script vs. human), Ch. 10 (when to trust the output), Ch. 11 (building the team), Ch. 12 (an agent's authority = what it can do, read, emit, and influence).

## The book's narrow use

An *agent* is the wrapping that turns a single model sample into multi-step work. The model produces text; the agent does the work. Concretely, an agent has at least:

- A model (the probabilistic core).
- A loop (multiple turns, each conditioned on prior output).
- Some way to take action between turns — call a tool, read a file, ask a human, write to the world.

When this book says *agent*, all three are implied. A one-shot model call with no loop and no tool access is *an LLM call*, not an agent.

## What the term deliberately excludes

- **Anthropomorphic claims.** The book never says an agent "wants," "decides," or "knows" in the human sense. Saying "the agent decides which tool to call" is a description of code behavior — the loop selects from a set of tools based on the model's output. No claim about cognition.
- **The "agent vs. AI" framing.** Outside this book, *agent* is often a marketing word for "an LLM product with some autonomy." The book ignores that usage.
- **Sub-agents and orchestrators are still agents.** Ch. 7 introduces sub-agent and top-level session vocabulary as orchestration roles; both are agents under this definition.

## Common cross-references

- An *agentic system* is a system where one or more agents sit in load-bearing paths.
- The *harness* (see [harness](harness.md)) is what hosts the agent loop and mediates between the agent, its tools, and the world.
- The *cost-of-being-wrong* (Ch. 4) is the axis the book uses to decide which work warrants an agent vs. a script vs. a human (Ch. 9).

## Last reviewed

2026-05-12.
