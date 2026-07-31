# Adversarial input

**One-line:** Input that is not merely uncooperative but hostile — and "wearing the same clothes as the data," indistinguishable in form from legitimate content.

**First introduced:** Ch. 12, *Security and Trust Under Adversarial Input*.
**Load-bearing in:** Ch. 12 (the assumption the chapter drops that the rest of the book made).

## The book's use

*Adversarial input* names the assumption Ch. 12 drops. Every chapter before it treated the model as cooperative but fallible — a well-meaning coworker who tries to do the right thing and sometimes produces valid-looking invalid output. That model is correct for most of what you'll build, and it's why the rest of the book is about verification rather than defense.

Security asks you to hold a second model of the same system at the same time. Here the input isn't just uncooperative, it's **adversarial** — and it's *wearing the same clothes as the data*. The attacker's instructions arrive looking exactly like the content the model is supposed to read; there's no reliable way to tell the steering text from the legitimate content, because both travel in the same channel: natural language.

That's the shift the whole chapter turns on: the agent is no longer a fallible coworker but an **influenceable actor that reads untrusted content while holding authority.** Someone else's text — a Slack message, a retrieved document, a web page, a tool's output, a row of community-authored data — flows into the model's context and tries to steer what the agent does next with the authority it holds.

## Why the term gets a name

The runtime won't refuse the attacker for you, and the model won't reliably refuse either. What refuses an adversarial input is the architecture you built before the attacker showed up — the authority you declined to grant, the reads you scoped, the destinations you bound, the human you kept on the irreversible action. Naming *adversarial input* as a distinct assumption is what lets the chapter treat the worst realistic input as the design point, not an edge case.

## Common cross-references

- [prompt-injection](prompt-injection.md) is the specific mechanism by which adversarial input reaches authority.
- [authority](authority.md) is what the input is trying to steer; [blast-radius](blast-radius.md) is what it can reach if it succeeds.
- [least-privilege](least-privilege.md) is how you bound the worst realistic case in advance.
- The *cost-of-being-wrong* axis (Ch. 4, *Tool Surfaces and Trust Boundaries*) bites hardest here, because the adversarial case is the worst realistic input by definition.

## Last reviewed

2026-07-31.
