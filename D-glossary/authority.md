# Authority

**One-line:** Everything an agent can actually *do* through the layer around it — write, read, emit, influence, spend, delegate, persist — as opposed to what you can talk the model into *saying*.

**First introduced:** Ch. 12, *Security and Trust Under Adversarial Input*.
**Load-bearing in:** Ch. 12 (the central concept — an injected model can't exceed the authority you enforce outside it).

## The book's use

*Authority* is the answer to "what could this agent set in motion?" Ch. 12 is built to correct one narrow reading of the word — the assumption that an agent's authority is its write and execute tools, so "no write tool" means "no exposure." That reading is wrong, and correcting it is the point of the chapter.

The book deliberately broadens authority to every channel a compromise can travel through:

- **Write / act** — the obvious one: mutate the database, send the email, run the migration.
- **Read** — the confidentiality boundary. An agent that can retrieve sensitive data can leak it; reach is authority.
- **Emit** — the egress boundary. A response, a constructed URL, a search query, a tool argument, even a log field is a channel an attacker can push data out through.
- **Influence** — the influence boundary. Fabricated content that gets a human to approve spending or forward a recommendation causes a real action with no write tool involved.
- **Spend / delegate / persist** — compute and money, the ability to hand work to a more privileged actor, and the ability to write something into memory that steers a later, more trusted session.

The operational instruction that falls out of this: **measure your exposure by the authority the agent holds and the flows you enforce, not by how convincingly the model can be talked to.** The model's refusal is the second line, never the first — because an injected model exercises its authority through content that looks perfectly ordinary.

## Why the term gets a name

Principle 12 rides on it: *an injected model can't exceed the authority you enforce outside it — and authority is more than write tools.* A jailbroken model can only do what the deterministic layer around it lets it do. So the security work is bounding every kind of authority in the layer you control — scope what the agent can reach *and emit* to least privilege, authorize every action and destination at the moment it's used, and audit all of it — rather than trying to make the model harder to talk into things.

## Common cross-references

- [prompt-injection](prompt-injection.md) is the attack that tries to seize authority the agent holds; the durable defense is enforcing authority outside the model.
- [least-privilege](least-privilege.md) is the discipline of granting the *narrowest* authority that does the job.
- [blast-radius](blast-radius.md) is what a compromise can reach — bounded by the authority you granted.
- The [agent](agent.md) is the actor that holds authority; the [harness](harness.md) is where authority and least-privilege are enforced.

## Last reviewed

2026-07-31.
