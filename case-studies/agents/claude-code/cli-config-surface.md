---
title: "Case Study: The Config/Health/Cost Command Surface, and Where Config Actually Lives"
tags: [claude-code, case-study, configuration, settings, cli]
status: draft
---

# Case Study: The Config/Health/Cost Command Surface, and Where Config Actually Lives

A coding agent CLI typically exposes several commands that sound like they overlap but each answer a
genuinely different question. Keeping them straight avoids reaching for the wrong one.

## Three commands, three different questions

| Command-type | Question it answers | What it is not |
|---|---|---|
| A health-check command (e.g. `/doctor`) | "Is anything broken or misconfigured?" — validates settings files, checks hooks are wired correctly, checks external tool-server connectivity, scans for structurally broken configuration | Not a cost or token-efficiency profiler — it won't say "this costs too many tokens," only "this is broken/valid" |
| A context/window command (e.g. `/context`) | "What's occupying my window right now, and in which category?" | Not a health check, not a cumulative spend total |
| A cost command (e.g. `/cost`) | "How much has this whole session cost so far, cumulatively?" | Not a live per-turn counter, not a health check |

**Rule of thumb**: a health command asks "is anything broken/misconfigured"; a context command asks
"where are my tokens going right now"; a cost command asks "where has my money gone this session." A
purpose-built skill-benchmarking tool (where available) is the right instrument for "is this specific
skill well-designed" — none of the three built-in commands above measure that.

## `/permissions`

The interactive view for tool-permission rules — which tools or command patterns are auto-allowed,
auto-denied, or require asking each time. This is the in-session UI for the same rule set that would
otherwise be hand-edited directly in a settings file.

## A destructive maintenance command (e.g. `project purge`)

Wipes *all* state for a project — transcripts, task history, file history, its config entry. This is
destructive and irreversible; treat it like any other destructive operation (confirm before running,
don't reach for it casually).

## The reasoning-effort dial

A configurable "how much internal deliberation before answering" setting (low/medium/high/etc.).
Higher effort generally means better quality on hard, multi-step, or ambiguous work, at the cost of
more thinking tokens (billed as output) and more latency. Lower effort is faster and cheaper, and
perfectly fine for simple, mechanical requests.

**This is a trade-off dial, not a quality setting with one universally correct value.** A high global
default suits an architecture-heavy or exploratory workflow, but it isn't free — for trivial one-off
asks, it spends more than necessary. Where it matters most (e.g. inside a multi-agent orchestration
run), overriding effort *per individual call* rather than globally is the better lever — it lets a
single run mix cheap, mechanical steps with genuinely hard reasoning steps instead of paying the
global cost uniformly for everything.

## How global config files get discovered

A global, always-loaded instructions file lives at a **fixed, well-known path** the runtime always
checks at startup — discovery by convention, not by search. That file can also support an import
syntax (referencing another file by name from inside it), so anything referenced that way gets pulled
in and inlined too. The runtime typically also looks for a project-level version of the same file
(at the repository root) and combines it with the global one.

**A contrasting, easy-to-miss case**: a project that instead keeps its agent-facing instructions in a
differently-named file (this handbook's own `AGENTS.md` is a real example) does **not** get that file
auto-loaded into the always-on memory block at all — the runtime only checked for the conventional
name. Such a file only enters context when something — a person, or a tool — explicitly reads it as a
normal file. In short: the conventionally-named global config file (+ its imports) loads automatically
every session; any other file, however important, loads only if something explicitly reads it.

## Settings precedence, highest wins

A typical precedence order, from strongest to weakest: an organization/enterprise-managed policy →
flags passed for a single invocation → local, personal project-level settings (not shared/committed)
→ shared, team-wide project settings (committed) → global user-level settings. Later, more specific
scopes can only tighten or override what a broader scope allows, never silently ignore it.
