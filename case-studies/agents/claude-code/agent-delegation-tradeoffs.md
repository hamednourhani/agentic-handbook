---
title: "Case Study: When Delegating to a Subagent Saves Tokens vs. Adds Overhead"
tags: [agents, claude-code, case-study, delegation, cost-model]
status: draft
---

# Case Study: When Delegating to a Subagent Saves Tokens vs. Adds Overhead

## The saving mechanism

A subagent runs in its own isolated child session — whatever bulk work it does (file reads,
intermediate tool calls, exploratory dead ends) is absorbed there, not in the parent's conversation.
Only the spawn instruction and the final result cross back into the parent's own growing context. The
real payoff is preventing that bulk from ever entering the parent's Messages *permanently* (where it
would otherwise be resent on every future turn for the rest of the session) — not making any single
call individually cheaper.

**Concrete example**: delegating "find where X is defined in this codebase" to a search-focused
subagent costs the parent one short answer back, instead of a dozen full file reads sitting in the
parent's context forever, each one resent on every subsequent turn.

## Where it's pure overhead instead

1. **Fixed spin-up cost** — the child loads its own full system prompt and tool set from scratch. Not
   worth paying for a single quick lookup or one-line read.
2. **Re-explaining cost** — the child starts with zero parent conversation history; anything it needs
   to do the task well has to be spelled out explicitly in the spawn prompt. For anything that depends
   on nuance already established in the parent conversation, this both costs tokens to restate and
   risks a worse answer if something gets left out.
3. **No savings if the output itself is large** — if the deliverable is inherently big (e.g. "write a
   full report"), nothing was saved; the bulk simply moved from one context to another, then came back
   anyway.
4. **No parallelism benefit for a single small task** — running multiple subagents in parallel saves
   wall-clock time, not tokens. That's a separate axis from the context-isolation savings above; either
   one alone can justify delegating, but they shouldn't be conflated as the same benefit.

## The decision, compressed

Delegate when the *intermediate work* is large relative to the *final answer* — that's when isolation
actually pays for itself. Don't delegate a task whose full context, work, and result would all have
fit comfortably in the parent's own next few turns anyway.
