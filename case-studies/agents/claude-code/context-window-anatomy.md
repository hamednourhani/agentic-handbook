---
title: "Case Study: Context Window Anatomy in Claude Code"
tags: [context-window, claude-code, case-study, tokens]
status: draft
---

# Case Study: Context Window Anatomy in Claude Code

Every turn, the prompt sent to the model is assembled from fixed blocks, in a fixed order:

```
[System prompt]  [System tools]  [Custom agents]  [Memory files]  [Skills catalog]  [Messages...]
    ~9k               ~18k            ~1-2k             ~1-2k            ~8k          growing, unbounded
```

## What's in each block, and why the numbers land where they do

- **System prompt** — the agent runtime's baked-in instructions (identity, tone rules, safety
  protocol, "how to do tasks"). Identical for every user, every project. Cannot be shrunk — it's the
  fixed cost of using the tool at all.
- **System tools** — the JSON schemas for tools loaded on every turn without being asked (file read/
  edit/write, search, subagent delegation, etc.). Each schema costs tokens for its name, description,
  and parameter definitions; a handful of tools with especially broad descriptions drag the average
  up. This is usually the single largest fixed block.
- **Custom agents** — a one-line description of each configured subagent, so the router knows what
  exists. Critically, this is *not* each subagent's full system prompt — just enough for the model to
  decide whether to invoke one. The full instructions for a given subagent only load into that
  subagent's own isolated session when it's actually invoked (see
  [Agent Delegation Trade-offs](./agent-delegation-tradeoffs.md)).
- **Memory files** — global and project-level persistent instructions/config files, plus the index of
  any structured auto-memory system. These load in full, every turn, unconditionally, regardless of
  whether anything in them is relevant to the current task.
- **Skills catalog** — only the **frontmatter description** of every available skill, not its body.
  This is why a large library of skills only costs a small amount of tokens on average per skill — the
  expensive part (the full instructions) is deferred until a skill is actually triggered, at which
  point it gets injected directly into the ongoing conversation (see
  [Skill Design Principles](./skills-design-principles.md)).
- **Messages** — the actual transcript: everything said, every tool call and result, every skill/agent
  payload ever injected into the parent conversation. This is the **only** block that's cumulative and
  unbounded — it only shrinks when the conversation is explicitly compacted (see
  [Compaction Mechanics](./compaction-mechanics.md)).

**Notably absent from this list: connected external tool servers (MCP).** A newly connected server can
show up as "N tools, 0 tokens" — it sits in a deferred registry, genuinely free until a tool-search
step pulls a specific schema in, at which point it becomes a permanent line in Messages and stays
there for the rest of the session (see [Tool Loading Cost Model](./tool-loading-cost-model.md)).

## The mental model

Everything except Messages is a **fixed floor** — it doesn't change turn to turn, no matter how long
the conversation runs. Messages is the only lever under direct control, via three things: what gets
asked, which skills/agents get invoked (each injects its full body once, permanently, into Messages),
and when the conversation gets compacted.

## Design trade-off worth naming

Bundling "always loaded" (system prompt, tools, memory) against "only loaded on demand" (skill/agent
bodies, deferred tool schemas) is the same shape repeated at every layer of the runtime: a cheap,
unconditional index gates something expensive that's pulled in only when actually needed. This
pattern — **progressive disclosure** — recurs across skills, memory, and tool loading, and is the
single idea that explains most of the cost-control levers available in this kind of system. See
[Context Budget Placement Checklist](../../../techniques/context-budget-placement-checklist.md)
for the practical decision tree built on top of it.
