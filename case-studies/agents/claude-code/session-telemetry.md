---
title: "Case Study: Three Different Numbers That Look Like the Same Thing"
tags: [claude-code, case-study, telemetry, cost-model, context-window]
status: draft
---

# Case Study: Three Different Numbers That Look Like the Same Thing

An interactive coding agent typically surfaces at least three different live numbers while it works.
They're easy to conflate because they all look like "how much have I used," but each measures
something different, resets on a different schedule, and answers a different question.

| # | What it measures | Resets when? | Grows across the session? |
|---|---|---|---|
| 1. Live "generating" counter | Output tokens streaming out for the **current turn** (which may internally chain several model calls if it uses tools along the way) | Every new user-facing turn | No — a fresh count each turn, not cumulative |
| 2. Context-window snapshot | Total tokens currently occupying the window, as of the last finished turn | Never on its own — only shrinks on an explicit compaction | Yes — climbs turn over turn as history grows |
| 3. Cumulative session cost | Sum of every input/output/cache-read/cache-write token from every call, the whole session | Never until a new session starts | Yes — strictly additive, the running total |

## Correcting a natural but wrong mental model

A plausible first guess is that the live counter resets on every individual model call inside a turn.
The more accurate model: a single user-facing turn can involve several chained model calls under the
hood (reason → call a tool → read the tool's result → reason again → maybe another tool call → ... →
final answer), and the live counter represents the **whole visible turn**, not each individual chained
call inside it.

- **Time**: the elapsed duration shown is the active-processing time for the **entire turn** — every
  chained call back-to-back, with no user idle time mixed in, since there's nothing for the user to
  wait on *between* those internal steps. Summed across every turn in a session, this is exactly the
  cumulative "API time" a session-cost report shows.
- **Tokens**: the live count is output tokens, and it keeps climbing across the internal steps of one
  turn precisely because each chained call can generate more output (another tool call, more
  reasoning, eventually the final reply) — all counted as part of that one turn's total. It goes static
  once the turn is fully done and control returns to the user, at which point the *next* turn starts
  its own counter from zero.

The correction worth internalizing: "resets every model call" is too granular. The right unit is *per
user-facing turn*, which may internally chain multiple model calls via tool use, not per individual
API round-trip within that turn.

## Worked example, three turns

- **Turn 1**: a question is sent. The live counter climbs from 0 as the model generates — say, up to
  a few hundred tokens over a few seconds of active processing (this is the turn's own **API time**).
  Once it finishes, the context-window snapshot updates to reflect the fixed floor plus this exchange.
  The cumulative cost total so far reflects this turn's input/output (nothing cached yet — it's the
  first call).
- *(The user reads the reply and responds ninety seconds later — that ninety seconds is invisible to
  the live counter and to the cumulative cost, it only shows up as part of real elapsed "wall" time.)*
- **Turn 2**: a new message is sent. The live counter **resets to zero** and climbs fresh for this
  turn's own output — it has no memory of Turn 1's number, which is gone; that number only ever meant
  "this turn, right now." The context-window snapshot updates again, now larger. The cumulative cost
  total adds Turn 2's numbers on top of Turn 1's — this is the only one of the three that's a
  straightforward running sum.
- **Turn 3**: same pattern repeats — live counter resets again, context-window snapshot climbs again,
  cumulative cost keeps accumulating.

## API time vs. wall time

**API time** — what the live counter's elapsed seconds represent, and what sums into a session's
total reported "API duration" — is only the seconds actually spent inside model calls; it's what's
billed compute-wise. **Wall time** is real clock time, including every idle gap between messages —
invisible to billing directly, but it's the thing that decides cache hit vs. miss (see
[Prompt Caching Mechanics](./prompt-caching-mechanics.md)): a wall-time gap longer than the cache TTL
means the next call's prefix comes back as a miss and gets rewritten at premium price, even though
nothing about the actual generation work changed.

## Why this matters practically

None of these three numbers substitute for another. The live counter tells you nothing about total
session spend; the context-window snapshot tells you nothing about cumulative cost (a fully-cached
turn can occupy just as many context tokens as an uncached one); and the cumulative cost total tells
you nothing about how close the current conversation is to its context ceiling. Reading the wrong one
to answer a given question — "am I about to run out of room?" vs. "how much has this session cost so
far?" — leads to wrong conclusions even though all three numbers are technically accurate for what
they actually measure.
