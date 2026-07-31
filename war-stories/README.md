# War Stories

These are the real incidents behind *The Agentic Engineer*. The book pulls each
lesson inline, right where it's told — these writeups are the receipts: the full
timeline, the commit, the numbers as they stood.

Every one is self-contained and follows the same shape: what was being
attempted, what went wrong, how it was discovered, what fixed it, the durable
lesson, and — deliberately — what the story is *not* evidence for. They're drawn
almost entirely from production systems at [MagmaLabs](https://magmalabs.io): a
virtual-employee platform, a Rails CRM, a set of MCP tools, and the author's own
operator stack.

You never need a writeup to follow the book. Reach for one when a story makes
you want the details.

---

## Forge

*MagmaLabs' virtual-employee platform — supervised, capability-scoped agents on
Rails 8 that deliver LLM-summarized briefs to leadership over Slack and email.*

- [The Silent Truncation](forge-max-tokens.md)
- [The Launch Attack That Found No Authority to Hijack](forge-ops-analyst-launch-attack.md)
- [The Gate That Hasn't Fired](forge-redaction-never-fired.md)
- [The Function Returned ≠ The Work Completed](forge-layered-status.md)
- [The Routine That Said It Ran](forge-idempotency-stealth-state.md)
- [The Cache That Wasn't](forge-prompt-cache-minimum.md)
- [The Runner That Returned Before the Job](forge-async-adapter-smoke-test.md)
- [The Keyword Router Met the Spanish Question](forge-keyword-router-natural-language.md)
- [The Regex That Made Up Slack's Contract](forge-channel-id-regex.md)
- [The Delivery Boundary Pattern](forge-delivery-boundary.md)
- [The Audience Constraint as Leverage](forge-audience-constraint.md)

## lead-crm

*A Rails CRM built solo-plus-agents, including a cross-vendor review workflow
and an MCP tool surface.*

- [Cross-Vendor Agent Disagreement as a Review Primitive](leads-crm-codex-pr1.md)
- [Org Scoping Bypassed by Every Non-UI Surface](leads-crm-org-scoping.md)
- [The Tool Surface Is a Second Product](leads-crm-mcp-second-surface.md)
- [The Rails App That Ran Until It Deployed](leads-crm-heroku-boot-deployment-contracts.md)
- [Ninety-One Files and No Backfill](leads-crm-directory-linking-backfill.md)
- [The Invisible False Case](leads-crm-settings-toggle-false-path.md)
- [VISION.md as Agent Memory](leads-crm-vision-as-agent-memory.md)

## canvas-mcp

*An MCP design/canvas tool — an agent decides what to say, the tool decides how
it looks.*

- [The Tool That Lies About State It Doesn't Own](canvas-mcp-viewer-lifecycle.md)
- [The Bytes That Stayed in the Conversation](canvas-mcp-base64-png-context.md)
- [The Human Watching Is the Customer](canvas-mcp-human-watching-is-customer.md)
- [Two Subagents, Two Shadow APIs](canvas-mcp-two-shadow-apis.md)
- [629 Lines of Implication](canvas-mcp-half-built-evaluate.md)
- [Phase 1 Mockups After Phase 3](canvas-mcp-feature-multiplicative-renderer.md)
- [Box Shadows in the Colors Map](canvas-mcp-design-md-colors-parser.md)

## presentation-studio-mcp

*An MCP tool that generates presentation decks under an autonomous caller.*

- [The Agent/Tool Split: Keep the Worker Boring](presentation-studio-mcp-agent-tool-split.md)
- [Feedback Loops: When the Caller Doesn't Read Errors](presentation-studio-mcp-feedback-loops.md)
- [The Leak the Agent Couldn't See](presentation-studio-mcp-pillow-spawn-per-operation.md)
- [Sixty Lines of Insurance](presentation-studio-mcp-jsonrpc-stdio-fallback.md)
- [The Brand That Wasn't There](presentation-studio-mcp-denormalized-brand.md)

## markdown-toolkit

*A markdown/table conversion tool, built with a role-separated solo
orchestration playbook.*

- [The Reviewer Sweep That Wasn't](markdown-toolkit-reviewer-sweep.md)
- [The Badge That Made the Bug Landable](markdown-toolkit-diagnostic-badge.md)
- [Sub-agents Can't Spawn Sub-agents](markdown-toolkit-subagent-depth.md)
- [Role Separation, Not Team Size](markdown-toolkit-role-separation-solo.md)
- [The Selector That No One Questioned](markdown-toolkit-cellText-vestigial-selector.md)

## coide

*A coding-agent project — harness behavior, agent memory, and skill autonomy.*

- [AskUserQuestion: When Research Delegation Changes the Build](coide-askuserquestion.md)
- [Memory That Drifted From Code](coide-memory-drift.md)
- [The Skill That Auto-Invoked](coide-ship-feature.md)

## magma-core

*A MagmaLabs internal platform; the story is about the multi-agent review
harness, not the product.*

- [The Five-Hour Ghost (Orphaned Sub-agents)](magma-core-orphaned-subagents.md)

## magmalabs-assistant

*The author's own operator stack — Claude Code skills and sub-agents wired into
daily knowledge work.*

- [What You Delegate Is What Decays](magmalabs-delegated-skill-decay.md)

---

*Forty stories across eight projects. The specific tools, models, and harnesses
are dated; the failure shapes are not. That's the whole bet of the book: the
principles in the body outlive the tools in these stories.*
