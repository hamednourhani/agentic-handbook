---
title: "Case Study: Persistent Memory vs. Global Config vs. Conversation History"
tags: [memory, claude-code, case-study, persistence, cost-model]
status: draft
---

# Case Study: Persistent Memory vs. Global Config vs. Conversation History

Three mechanisms in an agentic coding tool all deal with "old information staying useful," and
conflating them produces real confusion. This case study separates them cleanly.

## The sticky-note vs. filing-cabinet analogy

**Global config file** (always-loaded project/user instructions) is a **sticky note on your
monitor** — read every single morning no matter what you're doing that day. It has to stay short,
because you see it *every time*, whether it's relevant to today's task or not.

**Structured memory** (a small index plus separate per-topic files) is a **filing cabinet with a
table of contents taped to the front**. The table of contents (the index) is short, cheap, and glanced
at every day. You don't drag every folder out each morning — only the one folder relevant to today's
task, and only when it's actually needed. The rest of the cabinet costs nothing until that day comes.

**The practical rule**: "Does this need to apply no matter what I'm doing?" → sticky note. "Is this
only useful when a specific topic comes up?" → filing cabinet. Example candidates for the cabinet: a
user's role or area of expertise, the root cause of a past incident, a project's operating currency or
deadline — true and useful, but not something that needs to be re-read every single turn regardless
of topic.

**Why the distinction matters**: stuff everything into the sticky note, and it stops being a quick
glance — it becomes a growing full page, read every turn even when almost none of it is relevant. The
cabinet can hold hundreds of folders without slowing the daily glance at the table of contents; cost
only hits the day a specific folder is actually opened.

## Structure of the filing-cabinet system

- **Index file** — one line per memory (`- [Title](file.md) — one-line hook`), capped in size by
  design, always fully loaded every session, regardless of relevance.
- **Individual memory files** — one per fact, each headed by a name, a short description, and a
  **type**. Only opened (read in full) when the current task is judged relevant to that file's topic,
  or when explicitly asked to recall — not auto-loaded.

**Four memory types**:

| Type | What it captures | Example |
|---|---|---|
| User | The person's role, expertise, working style | "Works primarily in Go, new to this project's frontend" |
| Feedback | A correction, or a confirmed-good approach | "Don't mock the database in integration tests — a past incident hid a broken migration" |
| Project | An ongoing fact, decision, or deadline | "Merge freeze begins on a specific date for a release cut" |
| Reference | A pointer to an external system | "Bugs are tracked in a specific external issue tracker" |

**What's explicitly excluded**: anything derivable from reading the code or git history directly,
one-off fixes with no lasting lesson, anything already covered by the always-loaded config file, and
purely ephemeral in-progress task state.

## How writing actually happens

There is no special "save to memory" message type. Mid-response, the assistant's own reasoning
(per its standing operating rules) judges that a fact is worth persisting, and then issues an ordinary
file-write tool call targeting the memory folder — the exact same mechanism as any other file edit.
The runtime treats it like any other tool call: a permission check, the filesystem write executes,
a result comes back. The runtime itself has no special awareness that "this write is memory" — that's
purely a path convention plus the model's own judgment call, made turn by turn, about whether a given
fact is worth surviving a full reset.

**Writing is not an every-turn event.** Most turns write nothing at all — memory-worthiness is a
judgment call matched against each type's trigger conditions (a correction happened, an approach was
just confirmed, a new durable fact about the user/project surfaced, an external-system pointer came
up), not a mechanical background scan running on every message.

## Why memory exists at all, precisely

Within a single, ongoing conversation, nothing needs memory to avoid being forgotten — the full
conversation history already holds everything until it's explicitly compacted. Memory exists for the
gap that compaction and session-end can't cover: once a session closes, its conversation history is
discarded entirely, and the next session starts from zero. So the precise framing isn't "memory keeps
a fact around longer" (a single ongoing session already does that for free) — it's "memory keeps a
fact forever, *across* sessions, because the conversation itself gets thrown away when the session
ends." Every memory write is a deliberate bet that a specific fact is worth surviving a reset that
would otherwise be guaranteed.

## Memory vs. conversation history — two independent mechanisms

These are separate systems that happen to both deal with "old info staying useful":

- **Conversation history** = the raw, complete transcript of *one session only*. It's temporary by
  default — a fresh session starts with none of it, unless that exact session is deliberately resumed.
  See [Compaction Mechanics](./compaction-mechanics.md) for how it's managed within a session.
- **Structured memory** = a small set of durable, curated facts stored separately on disk, scoped
  per-project, persisting across every future session regardless of what happens to conversation
  history or compaction.

**Compaction never reads from or writes to memory; memory-writing never depends on whether or when
compaction runs.** They are two independent clocks, not one feeding the other — a session can compact
many times without ever writing a memory, and a memory can be written in a session that never
compacts at all.

A manually-maintained notes file that a user asks the assistant to keep updating (a running log, for
instance) is a *third*, separate thing again — it survives on disk like structured memory does, but
it doesn't auto-load into a future session's context the way memory or global config do. It only
re-enters context if something explicitly reads it again.
