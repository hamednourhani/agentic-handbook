---
title: "Technique: Deciding Where a Piece of Information or Task Should Live"
tags: [context-window, skill-design, memory, technique, agent-architecture]
status: draft
---

# Technique: Deciding Where a Piece of Information or Task Should Live

Any agentic runtime with a context window offers several places to put a given piece of information
or a given repeatable task — a global config file, a persistent memory system, a skill, a delegated
subagent, an external tool connection, a full multi-agent orchestration, or nothing at all (just
letting it ride in conversation history until compacted). Picking the wrong home is the single
biggest driver of an agent that feels bloated or expensive for no clear reason. This is an ordered
checklist for making that call deliberately instead of by default.

## The checklist

1. **Is it true on literally every turn, regardless of task?**
   → A global, always-loaded config file. Keep it small — everything here is a permanent per-turn tax
   on every future conversation, forever, whether or not it's relevant to what's actually being worked
   on right now.

2. **Is it a durable fact about the user, the project, or past feedback — but not needed every turn?**
   → A structured, persistent memory entry. A one-line index entry stays always-loaded and cheap; the
   full content only gets pulled in when the current task is judged relevant to it.

3. **Is it a bounded, repeatable procedure for a specific, recognizable situation?**
   → A skill. Nearly all of the design effort belongs in the trigger description — the only part paid
   unconditionally on every turn. Keep the body itself lean, and push bulk justification or edge-case
   handling into reference material loaded only when actually needed.

4. **Is it a large, self-contained sub-task whose intermediate grind shouldn't permanently bloat the
   main conversation?**
   → A delegated subagent. An isolated child session absorbs the bulk (searches, exploratory reads,
   dead ends); only the spawn instruction and the final result cross back. Skip this for genuinely
   trivial one-offs — the fixed spin-up and re-explaining cost can exceed what delegation saves.

5. **Is external system access needed only for a bounded stretch of work?**
   → An external tool connection, held only for the span of work that actually needs it. Disconnect
   once that domain of work is finished, if the connection is broad/chatty and only one specific
   capability was ever used, or if context pressure is mounting and compaction is firing repeatedly.

6. **Are there many independent items needing deterministic fan-out, or does the task benefit from
   multi-pass quality checks (independent verification of each finding, a panel of differing
   approaches, repeating discovery until nothing new turns up)?**
   → A full orchestrated workflow — reserved for work at that scale, with genuine opt-in given its
   cost at scale. Default to a pipelined structure over a barrier-heavy one unless a stage genuinely
   needs every prior result together before proceeding.

7. **Is the conversation itself just large and mostly settled/resolved dead weight?**
   → Compact it. This is safe at essentially any point — it only changes what gets rebuilt and resent
   on future turns, it never touches the underlying, permanent conversation record.

## The idea that repeats at every layer

**Progressive disclosure.** A cheap, always-loaded index — a short config file, a one-line memory
pointer, a skill's trigger description, a tool registry entry — gates something expensive that's
pulled in only on actual demand. Every option on this checklist is a variation of the same shape; the
questions above are really just "which index should gate this, and how cheap can that index be kept."

**The recurring cost model: permanence, not proportionality.** Once something lands in a running
conversation, it costs the same on every future turn whether it was used once or fifty times. That's
why the *gate* deserves more design effort than the content behind it — a broad, imprecise gate that
fires rarely is worse than a narrow, precise one that fires often, for the same eventual amount of
real use.

## Worked example: applying this to designing a diagnostic skill

See [Capstone Skill Case Study](../case-studies/agents/claude-code/capstone-skill-case-study.md) for
a full walkthrough of this checklist applied to a single real deliverable, including two precision
gaps that only surfaced after the skill was actually used in practice.
