# Blast radius

**One-line:** What a compromise can reach, reveal, and set in motion — bounded by the authority you granted, not by whether the model refuses.

**First introduced:** Ch. 12, *Security and Trust Under Adversarial Input*.
**Load-bearing in:** Ch. 12 ("no write tool" is not "no blast radius").

## The book's use

*Blast radius* is the measure of a compromise: if untrusted input fully steered the agent on the turn that used a tool, a data source, or an output channel — what could it *reach*, *reveal*, and *set in motion*? The book insists you ask those three separately, because the read-only agents are exactly the ones where teams remember "reach" and "set in motion" but forget "reveal."

The chapter's sharpest correction lives here: **"no write tool" is not "no blast radius."** The Ops Analyst survived a write jailbreak because it had no write tool — a real win, but a narrow one, proving only the *integrity* case. The same architecture would not have stopped an exfiltration: a read-plus-emit agent, steered by a poisoned document, can pull confidential fields and push them out through a URL it constructs or the text of its reply, never calling a write tool. Every control that held at launch stays green while confidential data leaves the building. The authority being abused was *read plus emit*, not *write* — so the blast radius was never zero.

The book sorts blast radius into three tiers to decide how an agent may act:

- **Bounded** — damage capped by design (a read scoped to non-sensitive data whose output can only reach the requester; a write scoped to the caller's own rows; a trivially reversible action). Safe to fire autonomously.
- **Gated** — the blast radius is real, but a human stands between the model and the consequence (money, external communication, irreversible mutation, disclosure of sensitive data). Let the agent propose; make a human authorize the immutable action.
- **Unbounded** — a hijacked model could do arbitrary damage (raw database access, unrestricted shell, credentials, cross-tenant reach, an output channel that can egress anything it can read). Design this combination out of existence before launch.

## Why the term gets a name

It reframes the security question from "can the model be jailbroken?" (yes, eventually) to "when it is, what does the attacker get?" Blast radius is bounded by the authority you enforce outside the model, which is why the book's advice is to bound every channel on purpose and measure exposure by the flows you enforce — not by how convincingly the model can be talked to.

## Common cross-references

- [authority](authority.md) is what sets the blast radius; a compromise can reach exactly as far as the authority you granted.
- [least-privilege](least-privilege.md) caps the blast radius but does not make compromise harmless.
- [prompt-injection](prompt-injection.md) and [adversarial-input](adversarial-input.md) are how a compromise starts.
- The *cost-of-being-wrong* and *reversibility* axes (Ch. 4, *Tool Surfaces and Trust Boundaries*; Ch. 10, *When to Trust the Output*) are the general-case version of the reach/reveal/set-in-motion question.

## Last reviewed

2026-07-31.
