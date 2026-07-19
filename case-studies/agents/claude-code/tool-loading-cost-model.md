---
title: "Case Study: What Actually Gets Sent for Tools, Agents, and Skills"
tags: [claude-code, case-study, tokens, tool-loading, cost-model]
status: draft
---

# Case Study: What Actually Gets Sent for Tools, Agents, and Skills

Four categories of "things the model can use" behave differently in terms of what actually enters
the request on every turn. Confusing them leads to wrong intuitions about where cost comes from.

| Category | What's sent every turn, unconditionally | What's deferred, loaded only on trigger |
|---|---|---|
| Built-in system tools | Full JSON schema (name + description + full parameter spec) | Nothing — always fully loaded |
| External tool servers (MCP-style) | Effectively nothing — sits in a zero-cost deferred registry | Full schema, once a specific tool is actually looked up — then permanent |
| Custom subagents | A compact catalog: name + one-line description each | The subagent's full system prompt — never enters the parent's context at all, only the subagent's own isolated session |
| Skills | Only the frontmatter description of every available skill | The skill's full body — injected directly into the *parent's own* growing conversation the moment it's triggered |

## Built-in system tools

These are loaded in full on every single request, no exceptions — this is usually the largest single
fixed cost in the context window. There's no lazy-loading lever available here; it's the fixed price
of the tool having these capabilities at all.

## External tool servers (deferred registry)

A newly connected external tool server can report as "N tools available, 0 tokens" — it sits in a
registry the model can search, but nothing about it has entered the actual token-costed request yet.
The moment a specific tool's schema is pulled in (via an on-demand lookup step), it becomes a
permanent line in the conversation and stays there — cache-eligible like anything else, but never
un-loaded again for the rest of the session. See
[MCP Server Trade-offs](./mcp-server-tradeoffs.md) for the full cost/benefit picture of this
category.

## Custom subagents: isolation, not deferral

Subagents are not deferred in the same sense — their one-line catalog entries are *always* present,
small as that cost is. What's different is that their **full instructions never load into the calling
conversation at all**. They only load into that subagent's own separate, isolated session at the
moment it's actually invoked, and only the final result crosses back. This is a structurally different
kind of savings from "not loaded yet" — it's "never loaded into *this* context, period." See
[Agent Delegation Trade-offs](./agent-delegation-tradeoffs.md).

## Skills: cheap catalog, expensive-and-permanent trigger

Skills follow the same cheap-catalog pattern as subagents on the surface — only a one-line description
per skill loaded every turn — but the crucial asymmetry is what happens on trigger. Invoking a skill
does **not** spin up an isolated child the way invoking a subagent does. Instead, the model requests
a skill by name, the runtime looks up that skill's file and returns its full body as a tool result —
which lands directly in the **same, parent** conversation history, growing it permanently, exactly
like any other tool result would. Two skills invoked in the same conversation both leave their full
bodies sitting in that conversation forever afterward.

**The asymmetry, stated plainly**: subagent invocation isolates cost elsewhere (the parent stays
lean). Skill invocation lands cost squarely in the parent's own growing context. This is the single
most important fact for deciding whether a piece of repeated work should be a skill or a delegated
subagent — see
[Context Budget Placement Checklist](../../../techniques/context-budget-placement-checklist.md).

## The recurring shape

All four categories are variations on the same idea: a cheap, always-loaded index (a schema, a
catalog line, a frontmatter description) gates something expensive, and the expensive part is either
never fully paid (system tools, forced to pay always) or paid once and then either permanent
(deferred tools once loaded, skill bodies) or contained elsewhere (subagent bodies). Recognizing which
bucket a given piece of functionality falls into is what makes it possible to predict its real cost
before using it.
