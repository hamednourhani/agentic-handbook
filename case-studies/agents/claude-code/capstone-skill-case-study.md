---
title: "Case Study: Designing a Diagnostic Skill From First Principles"
tags: [skills, claude-code, case-study, skill-design, capstone]
status: draft
---

# Case Study: Designing a Diagnostic Skill From First Principles

A worked example of applying every idea in this collection to a single concrete deliverable: a skill
that diagnoses an agentic coding session's own context/token health and recommends fixes, without
acting on them.

## Step 0 — confirm it should be a skill at all

Running it through
[Context Budget Placement Checklist](../../../techniques/context-budget-placement-checklist.md):
a bounded, repeatable procedure for a recognizable, recurring situation → a Skill, not a memory entry,
not a config-file rule, not a delegated agent task.

## Step 1 — the frontmatter description (the highest-leverage part, per [Skill Design Principles](./skills-design-principles.md))

Precise, example-laden, with an explicit negative clause: something like *"Use when the user asks why
the session feels slow or expensive, why context is growing fast, whether to compact now, whether an
external tool connection is worth staying connected, or where a piece of information should live.
Do NOT use for general application debugging unrelated to the session's own context mechanics."* The
negative clause exists specifically to stop the skill from hijacking unrelated requests just because
a word in the request ("context") is overloaded with an unrelated meaning.

Two more precision lessons learned only after the skill was actually used, both from the same class of
gap — a check being presented as more complete than it really is:

1. **Session-scoped vs. persistent checks weren't distinguished.** Some of the skill's checks are
   inherently only true of the *current* conversation (a live context breakdown; whether a given tool
   connection was invoked this session — there's no cross-session equivalent for either). Others read
   files that are shared and persistent across every session (a global config file's size; a memory
   folder's contents). Presenting both kinds of check uniformly implies a clean result on one says
   something about the other, when it doesn't. The fix: label each check's scope explicitly in the
   skill body, plus name the concrete blind spot (a tool connection idle *in this session* may still be
   essential in a different, unrelated session for the same project — this skill has no visibility into
   other sessions' history to check that).
2. **Overlap with an existing built-in health-check command wasn't addressed.** A health-check command
   answers "is this broken?" (bad settings, failed connections, invalid config). This skill answers a
   different question entirely: "is this *working* thing worth what it costs?" A tool connection can
   pass a health check cleanly and still be exactly what this skill should flag — healthy, but
   unused and quietly costing reasoning-surface. The fix: an explicit "do NOT use for X — that's the
   health-check command's job" clause added directly to the frontmatter, so the boundary travels with
   the skill automatically instead of depending on a person explaining it in conversation each time.

**Generalized lesson for skill design**: state what a skill does *not* cover, and how it differs from
anything with adjacent-sounding scope, proactively — before a reader has to discover the gap by
probing with follow-up questions.

## Step 2 — a lean body, diagnostic loop

1. Establish current state: pull the live context breakdown; check global-config size for content
   that isn't true on literally every turn; check which external tool connections are active and
   whether they were actually used this session; check whether structured memory exists and whether
   its index is bloated.
2. Match the symptom to a likely cause via a small lookup table (fast context growth → config bloat or
   a chatty tool connection; a repeated fact needing re-explanation → should be memory, isn't; a
   repeated task costing fresh tokens each time → should be a skill or delegated to an agent; and so
   on).
3. Report the diagnosis and the recommended fix plainly, with the *why* in one line — don't act on it.

## Step 3 — progressive disclosure for the justification material

The heavy reasoning behind each recommendation (the full cost model, the tool/agent/orchestration
trade-offs) is pushed into separate reference files the skill only reads on demand — not stuffed into
the main body that fires on every trigger. This mirrors [Skills Design Principles](./skills-design-principles.md)
point 2, applied to the skill's own construction.

## Step 4 — one job, no side effects

The skill diagnoses and recommends only — it does not itself trim files, disconnect tool connections,
or trigger compaction, since those are consequential actions that should stay under explicit
human approval rather than something a diagnostic skill does on its own initiative.

## Confirming "lazy loading" isn't a feature to build

"Loads lazily" isn't something that needed adding — it's simply what a skill body already does by
default (per [Tool Loading Cost Model](./tool-loading-cost-model.md)): only the frontmatter
description is unconditionally loaded every turn; the body and reference files stay unloaded until
the skill is actually triggered. This was confirmed empirically in practice: immediately after writing
the skill's frontmatter, the always-on skill listing reflected the new description text alone — the
body and reference files remained untouched until the skill was actually invoked.
