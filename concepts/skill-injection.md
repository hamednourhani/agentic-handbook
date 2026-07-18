# Skill Injection Patterns

"Skills" are a prompt engineering technique for providing an agent with specialized, just-in-time knowledge for a specific task. They are not a native feature of LLMs but are managed by the agent runtime.

## How It Works
When a user's prompt is received, the runtime can identify a relevant skill and inject its content into the prompt before sending it to the LLM. This gives the LLM detailed, task-specific instructions.

The skill content is typically injected as a new `system` message right before the user's prompt to give it maximum contextual relevance for the immediate task.

## Architectural Trade-offs

There are two main patterns for selecting a skill, each with a different trade-off.

### 1. Runtime-Led Selection (The `odek` approach)
- **Mechanism:** The agent runtime performs a semantic search comparing the user's prompt against a library of skill descriptions. If a high-similarity match is found, it automatically injects that skill's content.
- **Pro:** Fast and token-efficient. The LLM's context is not cluttered with irrelevant options.
- **Con:** The agent is dependent on the quality of the semantic search. The LLM has no autonomy to choose a different skill.

### 2. LLM-Led Selection (Alternative Design)
- **Mechanism:** The runtime provides the LLM with a *list* of available skill names and descriptions. The agent is also given a `load_skill(skill_name)` tool. The LLM then reasons about which skill is best and uses the tool to load it.
- **Pro:** More autonomous. The LLM's powerful reasoning makes the choice, which may be more accurate.
- **Con:** Slower and more expensive. It requires at least one extra ReAct cycle (Reason -> Act(`load_skill`) -> Observe) just to get the instructions.
