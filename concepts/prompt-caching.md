---
title: "Performance: Prompt Caching"
tags: [prompt-caching, performance, cost-optimization, concept]
status: final
---

# Performance: Prompt Caching

Prompt Caching is an advanced performance optimization used by agent runtimes to reduce latency and cost.

## The Problem
By default, an LLM is stateless. On every ReAct cycle, the agent runtime must send the *entire* conversation history. The LLM provider has to re-process this full history every time, even the parts that haven't changed.

## The Solution: Caching the "Immutable Prefix"
The solution is to identify the stable, unchanging beginning of the conversation history (the "immutable prefix") and have the server "memorize" it.

The immutable prefix is the entire conversation history as it existed at the end of the previous completed user turn.

## The Mechanism
Prompt Caching is a two-step API protocol:

1.  **Create Cache (First call of a turn):** The agent runtime sends the immutable prefix inside a special `cache_control` block. This tells the server to process this text but also save the resulting internal state to a temporary cache.

2.  **Use Cache (Subsequent calls in the turn):** For all future ReAct cycles within that same turn, the runtime sends a much smaller request. It sends **only the new messages** along with a flag like `use_cache: true`. It does **not** re-send the immutable prefix. The server sees the flag, retrieves the cached state, and combines it with the new messages before processing.
