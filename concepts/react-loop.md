---
title: "The ReAct Loop"
tags: [react-loop, agent-fundamentals, concept]
status: final
---

# The ReAct Loop

**ReAct** stands for **Reason -> Act -> Observe**. It is the fundamental operational cycle for most modern AI agents.

## The Cycle

1.  **Reason (Think):** The agent's LLM brain analyzes the current goal and all available information (history, tool outputs) to decide on the best logical next step.
2.  **Act (Use a Tool):** The agent runtime executes the action chosen by the LLM. This is typically a tool call, like reading a file or querying a database.
3.  **Observe (See the Result):** The output from the tool is captured and added to the conversation history.

This cycle repeats, with each observation feeding the next reasoning step, until the agent has enough information to provide a final answer.

## Lifecycle

A single user prompt doesn't trigger just one loop, but rather a **chain of loops**. For example, a request to "summarize a file" might trigger:
- **Loop 1:** Reason to find the file, Act by using `ls`, Observe the file list.
- **Loop 2:** Reason to read the file, Act by using `read_file`, Observe the file's content.
- **Loop 3:** Reason to summarize the content, Act by generating the summary text, and deliver the final answer.
