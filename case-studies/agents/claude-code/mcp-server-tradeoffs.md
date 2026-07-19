---
title: "Case Study: Why a Connected-but-Unused Tool Server Isn't Free"
tags: [mcp, claude-code, case-study, tool-servers, cost-model]
status: draft
---

# Case Study: Why a Connected-but-Unused Tool Server Isn't Free

Per [Tool Loading Cost Model](./tool-loading-cost-model.md), a connected external tool server shows
"0 tokens" until a specific tool schema is actually pulled in — but "0 tokens right now" is not the
same as "free to keep connected." Four reasons this matters, in increasing order of subtlety.

## 1. A small registry cost exists just to route lookups

Some lightweight per-server entry has to exist just so an on-demand tool-search step knows the server
and its tools exist at all. This floor is negligible per server, but it's not literally zero, and it
scales — mildly — with how many servers are connected at once.

## 2. A broad lookup against a "chatty" server can pull in more than intended

If a server exposes many tools and a lookup query is broad, several tool schemas can load at once —
each one becoming a permanent line in the conversation from then on, even if only one of them was
ever actually needed for the task at hand.

## 3. Permanence, not proportionality

A schema that's loaded once costs exactly as much on every future turn as one used fifty times —
loading is a one-time permanent tax, unrelated to how much value gets extracted from it afterward.
This means broad-but-rarely-used tool loading is strictly worse than narrow-but-frequently-used
loading, for the same eventual total usage.

## 4. The bigger, less visible cost: reasoning surface

Every additional loaded tool is one more option the model has to weigh on every future tool-choice
decision, for the rest of the session. This is a reasoning-quality cost, not a token-line cost — it
doesn't show up as a number in a context breakdown, but it's real, and it compounds the more
unrelated tools accumulate in a single session.

## When to disconnect, concretely

- The domain of work that needed this server is finished for the rest of the session (e.g. a
  migration against one system is done, no more calls to that system are expected).
- The server is broad/chatty and only one specific tool was ever actually needed — staying connected
  invites future lookups to sweep in its siblings unnecessarily.
- The session is near its context ceiling and automatic summarization keeps firing — trimming
  unused connections is one lever alongside compacting.

**Rule of thumb**: keep a server connected only for the span of work that actually needs it, not
connected by default for an entire session regardless of relevance.

## The caveat that matters most

"Unused this session" is a single-session snapshot, not a usage history. A server idle in *this*
conversation may be essential in a different, unrelated conversation for the same project — there is
no visibility from inside one session into whether the same server is load-bearing elsewhere. Treat
"hasn't been touched this session" as a prompt to ask before disconnecting, not as a standalone
verdict to act on unilaterally.
