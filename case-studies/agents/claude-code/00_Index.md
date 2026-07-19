# Case Studies: Claude Code Context & Token Mechanics

A collection analyzing how a real, production agentic coding tool manages its context window, cost,
and tooling — grounded in direct empirical observation (live `/context`/`/cost` output, watching a
skill load in real time, verifying billing claims against current docs), not just restated
documentation.

## Reading order

Start with the foundations, then branch into whichever specific mechanic is relevant:

1. [Context Window Anatomy](./context-window-anatomy.md) — the fixed blocks vs. the one growing block
2. [Prompt Caching Mechanics](./prompt-caching-mechanics.md) — how caching actually works, breakpoints, KV-cache
3. [Prompt Caching Billing Reality](./prompt-caching-billing-reality.md) — the real multipliers, and a lesson in verifying dollar-specific claims
4. [Tool Loading Cost Model](./tool-loading-cost-model.md) — what's sent for tools, agents, and skills, and why they differ
5. [Skills Design Principles](./skills-design-principles.md) — writing a skill that stays cheap until triggered
6. [Agent Delegation Trade-offs](./agent-delegation-tradeoffs.md) — when a subagent saves tokens vs. adds overhead
7. [MCP Server Trade-offs](./mcp-server-tradeoffs.md) — why a connected-but-unused tool server isn't free
8. [Memory Architecture](./memory-architecture.md) — persistent memory vs. global config vs. conversation history
9. [Compaction Mechanics](./compaction-mechanics.md) — what compacting actually touches, and what it doesn't
10. [Session Telemetry](./session-telemetry.md) — three different numbers that look like the same thing
11. [Workflows Orchestration](./workflows-orchestration.md) — pipeline vs. parallel, when a full orchestration is worth it
12. [CLI Config Surface](./cli-config-surface.md) — health checks vs. context vs. cost commands, and where config actually lives
13. [Capstone Skill Case Study](./capstone-skill-case-study.md) — applying all of the above to design one real skill

## The throughline

Nearly everything here reduces to two ideas, repeated at every layer:

- **Progressive disclosure** — a cheap, always-loaded index gates something expensive pulled in only
  on demand (a skill's description gates its body, a memory index gates its files, a tool registry
  gates its schemas).
- **Permanence, not proportionality** — once something enters a running conversation, it costs the
  same on every future turn regardless of how many times it actually gets used, which is why the gate
  matters more than the content behind it.

See [Context Budget Placement Checklist](../../../techniques/context-budget-placement-checklist.md)
for the practical decision tree built on top of this collection.
