# Glossary

Agent Runtime
: The "body" for an LLM "brain." It's the code that manages the ReAct loop, provides tools, handles memory, and communicates with the LLM API.

Immutable Prefix
: The stable, unchanging beginning of a conversation history that can be "memorized" by the server when using Prompt Caching.

Prompt Caching
: An advanced performance optimization where the LLM provider's server caches the immutable prefix of a conversation, avoiding the need for the client to re-send it and the server to re-process it on every turn.

RAG (Retrieval-Augmented Generation)
: A technique for providing an agent with long-term memory. The agent *retrieves* relevant information from a knowledge base (like a vector database of past conversations) to *augment* its current prompt before *generating* a response.

ReAct Loop
: The fundamental operational cycle of an agent: **Reason** (the LLM thinks and chooses an action) -> **Act** (the runtime executes the action, usually a tool) -> **Observe** (the runtime captures the result and adds it to the history).

System Prompt
: A set of instructions given to an LLM at the start of a conversation to define its persona, goals, and rules. It's what turns a generalist model into a specialist agent.

Tool / Function Calling
: The mechanism by which an LLM can request that the agent runtime execute a function. The LLM is provided with a "menu" of available tools and can respond with a structured command to invoke one.
