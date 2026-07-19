---
title: "Case Study: Prompt Caching Billing, Verified Against Live Sources"
tags: [prompt-caching, claude-code, case-study, billing, cost-model]
status: draft
---

# Case Study: Prompt Caching Billing, Verified Against Live Sources

[Prompt Caching Mechanics](./prompt-caching-mechanics.md) covers *how* caching works. This is about
the actual bill — and about a concrete lesson in checking dollar-and-date-specific claims live rather
than trusting a model's (any model's) training-time knowledge of pricing.

## The multipliers

| State | Rate (relative to base input price) | Why |
|---|---|---|
| First call / cache miss, 5-min TTL | ~1.25x | Paying a premium to *write* the cache entry |
| First call / cache miss, 1-hour TTL | ~2x | Larger premium for the longer-lived entry |
| Cache hit | ~0.1x (~90% discount) | Server skips recompute, passes the savings on |
| Uncached tokens (e.g. brand-new messages) | 1x | No cache involved at all |

Output tokens are never affected by caching — always billed at the normal output rate. There is also
a minimum chunk size (in the low thousands of tokens) below which a span isn't cache-eligible at all
— caching entries has its own overhead, so very small spans aren't worth caching individually.

**Break-even rule of thumb**: a 5-minute cache entry pays for itself after roughly 2 reads; a 1-hour
entry after roughly 12 reads (it costs more to write, so it needs more reuse to be worth it). Pick the
longer TTL only for workloads with long, sparse gaps between reuse (an automation that runs once an
hour); for anything continuous and chatty, the short TTL is already close to free-riding the whole
time, and paying the bigger write premium buys nothing extra.

## Individual/subscription use vs. raw API use

**Flat-rate subscription plans** (interactive coding-assistant use): usage is measured against a
quota/rate-limit, not a per-token bill you see directly. Caching is typically **on by default** — the
client software inserts the cache-control breakpoints automatically, with nothing to configure. The
benefit shows up indirectly: cached turns are cheaper and faster to serve, which is what lets a
flat-rate plan sustain the usage limits it offers — in practice, a quota stretches further the longer
a session stays continuous, per the caching mechanics above.

**Metered API / SDK usage** (building a custom integration, not going through a packaged client):
pay-as-you-go per token. Here, caching is **opt-in at the request level** — a developer must
explicitly add a cache-control block. Skip it, and every token is billed at plain base-input rate,
every call, no discount and no premium either.

## A worked reading of a real cost breakdown

A representative multi-hour session's cost summary might look like this:

```
duration (API):  ~15 min
duration (wall): several hours
By model:
model-small:  ~10k input, ~1.5k output, 0 cache read, ~50k cache write
model-main:   ~20k input, ~55k output, ~4.6M cache read, ~590k cache write
```

Reading this against the mechanics above:

- **A cache-read-to-cache-write ratio around 7-8:1** means every token in the "write" bucket got
  reused (hit) roughly 7-8 times on average, at ~90% off each time it was a hit — comfortably past
  the ~2-read break-even point, which is exactly why a long, heavily-edited session doesn't cost
  proportionally to its full accumulated context.
- **`duration (API)` far smaller than `duration (wall)`** is the pipeline from
  [Prompt Caching Mechanics](./prompt-caching-mechanics.md) made concrete: "wall" is real elapsed
  time including every idle gap between messages; "API" is only the time actually spent mid-inference.
  The model is bursty — it only consumes compute during the moments it's actively generating tokens.
- **Cache write isn't trivial even in a long, continuous session** for two reasons: large file
  edits/tool results resend substantial new content that's a write the first time it appears, no way
  around that; and any real-time gap between messages that exceeds the TTL turns what would have been
  a hit back into a full rewrite (a plausible occurrence across a long wall-clock session with
  thinking/reading time between turns).
- **A smaller helper model showing cache writes but zero cache reads** is the signature of a one-shot
  subagent or tool-use helper call: it never got reused within its own TTL window, so every one of its
  tokens landed in the write bucket with no chance to earn back the premium. A write that's never
  followed by a read is pure cost with no offsetting discount — the clearest failure mode in this
  whole cost model.

## The lesson: verify time-sensitive claims live, don't trust training-time recall

Pricing tiers, billing policy, and TTL defaults are exactly the kind of fact that drifts. Two concrete
examples surfaced by checking live sources instead of relying on recall:

1. **A claimed policy change turned out to have been announced and then reversed** before it ever took
   effect — a research answer that reported the announcement as settled fact, without registering the
   later reversal, would have been confidently wrong about current billing behavior.
2. **The default cache TTL was silently changed** (from 1 hour down to 5 minutes) at a point in time —
   a plausible-sounding "caching defaults to 1 hour" claim would have been correct once and stale later.

**Generalized rule**: for anything dollar-specific, date-specific, or policy-specific, treat a
confident-sounding claim (from any source, not just an LLM) as a hypothesis to verify against the
current live documentation before acting on it — especially before budgeting or automating around it.
The multipliers and rules of thumb in this document are a snapshot; check current pricing pages before
relying on exact numbers.

**Sources**: [Anthropic prompt caching docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching).
