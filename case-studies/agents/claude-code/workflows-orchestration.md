---
title: "Case Study: Pipeline vs. Parallel, and When a Full Workflow Is Worth It"
tags: [workflows, orchestration, claude-code, case-study]
status: draft
---

# Case Study: Pipeline vs. Parallel, and When a Full Workflow Is Worth It

Multi-agent orchestration tools typically offer two ways to fan work out across many independent
items, and confusing them costs real wall-clock time even when the token cost is identical.

## The mechanical difference

**A parallel-barrier call** runs every item concurrently but waits for *all* of them before
continuing to the next stage. Chaining several parallel-barrier stages means the slowest item in
stage A blocks *every* item's stage B — even items that finished stage A almost instantly.

**A pipeline call** runs each item through all stages independently, with no barrier between stages.
Item A can be on stage 3 while item B is still on stage 1. Wall-clock time approaches the slowest
single item's *full chain*, not the sum of every stage's slowest item.

**Concrete example**: reviewing 5 files, then verifying whatever findings turn up. With a
parallel-barrier per stage, all 5 files must finish review before *any* verification can start — even
if 4 of them finished in seconds and one took minutes. With a pipeline, the fast files' findings start
verifying immediately while the slow file is still being reviewed — the same total work, finished
sooner.

## Default to pipeline

Reach for a real barrier only when a later stage genuinely needs *all* of the previous stage's results
together — deduplicating findings across the full result set before expensive downstream work, an
early exit when a total count comes back at zero, or a stage whose prompt needs to explicitly compare
against "the other findings." Anything short of that reason, using a barrier between stages is wasted
wall-clock time, not a real requirement of the task.

## When a whole Workflow (vs. a single delegated task) is worth the overhead

The same shape of trade-off as
[Agent Delegation Trade-offs](./agent-delegation-tradeoffs.md), one level up in scale. Justified when:

1. **The work is inherently broad or repetitive across many independent items** — migrating dozens of
   files, auditing every endpoint in a system — not a one-off task.
2. **Deterministic control flow is needed** — loops, conditionals, fan-out — rather than a single
   judgment call a lone agent could make in one pass.
3. **The task benefits from multi-pass quality patterns** that a single pass wouldn't perform on its
   own — independent adversarial verification of each finding, a panel of differently-angled attempts
   scored and synthesized, or repeating a discovery step until several consecutive rounds turn up
   nothing new.

If a task fits comfortably in one delegated call, or could just be done directly, standing up a full
orchestration script is pure overhead — the setup and coordination cost isn't earned back by anything
the task actually needed.
