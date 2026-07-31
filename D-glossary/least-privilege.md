# Least privilege

**One-line:** The narrowest authority that does the job — for *reading* as much as for writing and acting — enforced below the model, not requested of it.

**First introduced:** Ch. 12, *Security and Trust Under Adversarial Input*.
**Load-bearing in:** Ch. 12 (caps the blast radius; does not make compromise harmless).

## The book's use

*Least privilege* is an old security discipline that predates agents by decades; what Ch. 12 adds is only *where* you apply it — at the seam where a probabilistic planner chooses and sequences privileged calls. The default is the narrowest authority that does the job, and the chapter is insistent that this applies to **reading** as much as to writing: every source in context is attack surface, so a read scope is granted as tightly as a write scope.

The crucial calibration the book draws:

- **Least privilege caps the blast radius; it does not set it to zero.** Removing the Ops Analyst's write tool was the strongest possible version of least privilege — the escalation target didn't exist. But most attacks don't need escalation; they abuse *legitimate* authority. A scoped email tool can still email every permitted recipient a lie. A tenant-scoped delete can still delete everything in that tenant. A read tool can still enumerate confidential data. Least privilege lowers the ceiling on damage. It doesn't make compromise harmless.

Scoping is enforced deterministically, below the model — the write is scoped to the caller's own rows, the current tenant, the specific resource — so a fully-hijacked model calling a tool can only reach what this caller already could.

## Why the term gets a name

It's the practical lever behind Principle 12. You can't defend an agent by trusting the model, and you often can't defend it by removing the tool (the tool *is* the job). What you can do is scope the tool's *reach* — and its reads, and its output channels — to the minimum, so that owning the model buys the attacker the least possible ground.

## Common cross-references

- [authority](authority.md) is what least privilege rations; grant the narrowest kind that does the job.
- [blast-radius](blast-radius.md) is what least privilege caps but does not eliminate.
- [prompt-injection](prompt-injection.md) is the attack whose damage least privilege bounds.
- The *cost-of-being-wrong* axis (Ch. 4, *Tool Surfaces and Trust Boundaries*) decides how much authority a given tool warrants; the [harness](harness.md) is where the scoping is enforced.

## Last reviewed

2026-07-31.
