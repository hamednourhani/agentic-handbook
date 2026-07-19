---
title: "Case Study: Prompt Caching Mechanics in Claude Code"
tags: [prompt-caching, claude-code, case-study, performance, cost-model]
status: draft
---

# Case Study: Prompt Caching Mechanics in Claude Code

Claude Code, like any client of a stateless LLM API, must resend the entire conversation on every
single turn — the server keeps nothing between calls. Prompt caching is the mechanism that makes this
affordable. This case study covers how it actually works end to end, not just that it exists.

## The full pipeline, every single turn

```
[1] The client ASSEMBLES the full request body from scratch — nothing carried over server-side
    from before:
      block1: system prompt
      block2: tool schemas
      block3: custom agent descriptions
      block4: memory files
      block5: skills catalog
      block6: entire message history so far (every prior turn's message + response)
      block7: your new message
        │
        ▼   (travels as HTTP bytes — transport layer only, not billed, not "context")
[2] Sent over the wire to the model provider's API
        │
        ▼
[3] TOKENIZER converts blocks 1-7 into one long token sequence.
      → this sequence's length = "input tokens" = exactly what the context-window view shows
      → THIS is the context window: how many tokens this one sequence can be before the model
        has no room left. Tokenization is cheap/mechanical and runs fresh every request,
        cached or not — it's not the expensive part.
        │
        ▼
[4] The server checks the cache_control breakpoints the client placed inside that sequence
    (e.g. breakpoint A after block2, breakpoint B after block6). For each span:
      - span [start → A]: hash it. Match in the cache store, still warm → HIT, reuse the stored
        internal computation (skip the expensive attention pass) → cheap.
        No match / expired → MISS → compute normally → write a fresh entry → small premium.
      - span [A → B]: same check, independently.
      - span [B → end] (the newest message): never cached before → always fresh compute, plain price.
        │
        ▼
[5] The model runs inference over the WHOLE sequence (hit-spans reuse stored computation, miss/new
    spans computed fresh) and generates OUTPUT tokens. Output is never cacheable — it doesn't exist
    yet before the call — always billed at plain output rate.
        │
        ▼
[6] Response returns. The client appends the message + response to its growing history.
        │
        ▼
NEXT TURN → back to [1]. Nothing persisted server-side; the entire block list is reassembled and
resent in full. But blocks 1-2 and the old messages are now byte-identical to last time, so those
spans are very likely HITs this round — only the newest message is genuinely new.
```

**Where each concept lives**: context window = the total length of the sequence built in step [3],
every turn, full stop — caching never shrinks or bypasses that count. Tokenization = the mechanical
step in [3], always runs, always cheap, unrelated to caching. Caching = only step [4] — a cost/speed
optimization layered on top of a request that's already been fully assembled and fully tokenized; it
decides how much of the *expensive inference* over that already-counted sequence gets skipped, never
how much gets counted.

## What a "breakpoint" actually is

A request is a flat, ordered list of content blocks. A **breakpoint** is a marker attached to one
specific block, meaning "hash and cache everything from the start of this request through this
block, as one unit." A request can carry up to 4 such markers, letting the client mark several
nested "cache up to here" points at once.

Worked example, three turns, small numbers for intuition:

| Turn | Request assembled | Spans & what happens | Context tokens this call |
|---|---|---|---|
| 1 | `[SysPrompt+Tools](27,000)` → breakpoint A → `["hi"](2)` | span→A: never seen → **miss**, writes cache | 27,002 |
| 2 | `[SysPrompt+Tools](27,000)` → A → `[msg1+resp1](52)` → breakpoint B → `["how are you"](3)` | span→A: identical to Turn 1 → **hit** (cheap). span A→B: new → **miss**, writes cache. span B→end: new msg → plain price | 27,055 |
| 3 | same prefix + new turn appended | span→A: **hit**. span A→B: identical to Turn 2's write → **hit**. newest message: plain price | ~27,060+ |

Notice the **context-window column only ever grows** — caching never appears there. What changes
turn to turn is purely which spans are billed at hit-price vs. write-price vs. plain-price underneath
that same, ever-growing total. The general pattern: on turn N, everything through the end of turn
N−1 (already written last time) is a hit; the content added since then is new, gets a fresh trailing
breakpoint, gets written. The hit-prefix keeps extending forward, turn after turn, and only the
newest sliver is ever a write — which is exactly why a long session doesn't get proportionally more
expensive per turn even though the total keeps climbing.

**The catch — exact match, not semantic diff.** The server hashes the literal byte content up to a
breakpoint; it does not "diff" or understand that only one file changed by one character. Suppose,
between two turns, a single character changes in a memory file that sits *before* breakpoint A in the
request order — the server does a literal string hash of everything up to A, gets a different hash,
full stop. That one edit means breakpoint A misses, and *everything from that point in the request
onward* gets recomputed and rewritten fresh, even though the overwhelming majority of the actual
content is unchanged. Caching is exact-match, not semantic-diff.

## Output is cached too — just one turn later

Apparent contradiction worth resolving explicitly: "output is never cached" is true only for the
single turn in which a token is *being generated*. A response is OUTPUT during the turn it's
produced — freshly generated, billed at output rate, no caching concept applies to generation at all.
But the moment the next turn begins, the client reassembles the request and drops that prior response
back in as plain historical text inside the message-history block — at that point it's no longer
"output being generated," it's ordinary INPUT content, exactly as eligible for caching as anything
else. Same tokens, different role depending on which turn you're looking at: output once, then
forever after, ordinary cacheable input.

## Time-to-live: a sliding window, not a fixed schedule

A cache entry has a TTL — 5 minutes by default. Every **hit** resets that TTL back to its full
duration, so during an active back-and-forth (turns closer together than the TTL), an entry never
actually goes cold. It only expires if the conversation goes quiet longer than the TTL: the next
request after that gap is a full miss, billed and computed as if it were the first call ever, and a
fresh entry gets written from there. The entire accumulated prefix — not just the newest sliver — is
what's lost when this happens; the conversation history itself is completely unaffected, only the
caching benefit built up over the session evaporates, and the hit-chain has to start building forward
again from scratch.

Anthropic also offers a longer, opt-in 1-hour TTL at a higher write cost (see
[Prompt Caching Billing Reality](./prompt-caching-billing-reality.md) for the trade-off).

**Scope of a cache entry**: entries are isolated per workspace/account — never shared across
different customers' data, though they can be shared across a single account's own concurrent
sessions if the content up to a breakpoint happens to be byte-identical within each other's TTL
window (in practice, only the outermost, universally-identical blocks like the system prompt are
likely to actually overlap this way; anything deeper diverges per-project almost immediately).

## Why this is even safe: causal masking

Inside a transformer, every token is converted at each layer into a **Key** and **Value** vector —
its computed "meaning," available for later tokens to look up. A new token being generated needs to
*attend to* (look up) the Key/Value vectors of every token before it, plus compute its own.
Critically: a token's Key/Value vectors, once computed, never change based on what comes after it —
a direct consequence of **causal masking**: these models are decoder-only, meaning a token is only
ever allowed to attend to itself and tokens *before* it, never tokens after. That invariant is what
makes caching valid at all — if a token's representation could still be altered by later context,
reusing a stored copy of it would be unsafe.

The **KV-cache** is literally that stored set of Key/Value vectors, at every layer. On a cache hit,
the model does **not** re-run the old tokens through all the transformer's layers again — it skips
that entire recomputation and just loads their already-computed Key/Value vectors from storage. The
*new* tokens (whatever's past the last breakpoint) still get fully processed layer by layer as
normal, and as part of that processing they attend to the loaded (not recomputed) vectors of
everything before them. So: old tokens aren't "not passed" — conceptually the new tokens still need
everything before them to attend to — but the *expensive part* (re-deriving each old token's
representation through every layer) is what gets omitted. What's reused is the *result* of that work,
not the raw tokens.

This is also the direct reason generation itself is never a caching target: it's the one part of the
pipeline computing something that has never existed before, one token at a time, with nothing to
reuse.

## Practical takeaway

Caching turns "every turn costs proportional to total history so far" into "every turn costs
proportional to *what's new this turn*, plus a steep discount on everything old." The one thing that
breaks this: editing content that sits **before** an early breakpoint. Anything loaded unconditionally
at the start of every session (global configuration, standing instructions) is exactly the kind of
content where a small edit has an outsized cache-invalidation cost, since it sits ahead of everything
else in the request order.

**Sources**: [Anthropic prompt caching docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching).
