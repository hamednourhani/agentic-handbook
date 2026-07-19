---
title: "Case Study: Designing a Skill That Stays Cheap Until Triggered"
tags: [skills, claude-code, case-study, skill-design]
status: draft
---

# Case Study: Designing a Skill That Stays Cheap Until Triggered

Following directly from [Tool Loading Cost Model](./tool-loading-cost-model.md): a skill's
frontmatter description is paid unconditionally, every turn, forever, for as long as the skill
exists. Its body is only paid once — but paid permanently — the moment it's triggered. Four design
principles follow from that cost shape.

## 1. The frontmatter description is the highest-leverage part of a skill

It's the only part paid on every single turn, and it's also the sole signal deciding whether the far
more expensive body ever fires at all. Too vague, and it causes both failure modes at once: false
positives (an irrelevant body gets injected into unrelated conversations, wasting the trigger cost)
and false negatives (a real, matching request doesn't trigger it, so the intended savings/help never
materializes). Precise descriptions spell out exact trigger phrasing *and* explicit skip conditions —
"do not use for X" is as valuable as "use this for Y," since it prevents the skill from hijacking
adjacent-but-different requests.

## 2. Keep the body lean; push bulk into on-demand reference files

A skill's main body should be the minimum needed to act — not a dump of every sub-case, edge case, and
piece of background justification. Bulk content that only applies to specific sub-situations belongs
in separate reference files the skill reads only when that sub-case actually comes up. This mirrors
the same progressive-disclosure idea seen in deferred tool loading: don't pay for material you don't
need this particular time.

## 3. One skill, one job

A broad skill that tries to cover many loosely-related situations injects one large body every time
any part of it fires — even for a narrow, specific ask that only needed a fraction of that content.
Smaller, single-purpose skills keep each trigger's cost proportional to what was actually needed.

## 4. No isolation means repeated invocation compounds

Unlike a delegated subagent (see [Agent Delegation Trade-offs](./agent-delegation-tradeoffs.md)), a
skill's body lands in the parent's own conversation every time it fires — invoking the same skill
twice in one session pays its full body cost twice, permanently, both times. If a skill's real job
involves substantial intermediate work (many file reads, a long exploration), consider having the
skill delegate that work to an isolated subagent internally, so only a summary — not the full grind —
lands in the parent's growing context.

## The instrument for applying this

A dedicated skill-authoring/benchmarking tool (where available) is the practical way to measure a
skill's actual triggering accuracy and token footprint against these four principles, rather than
guessing from the description alone.
