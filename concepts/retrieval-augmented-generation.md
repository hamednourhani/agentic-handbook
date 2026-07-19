---
title: "Retrieval-Augmented Generation (RAG)"
tags: [rag, long-term-memory, vector-database, concept]
status: final
---

# Retrieval-Augmented Generation (RAG)

RAG is a technique used to provide an agent with a **long-term memory**. It separates the agent's memory into two types:

1.  **Working Memory:** The immediate, short-term conversation history sent with every prompt. It's fast but has a hard size limit (the context window).
2.  **Long-Term Memory:** A durable, searchable archive of all past conversations, typically stored in a **vector database**.

## The Process

RAG is a two-step process:

### 1. Filing (Storing Memories)
- **Chunking:** Past conversations are broken into meaningful chunks.
- **Embedding:** Each text chunk is converted into a numerical vector (an "embedding") that represents its semantic meaning.
- **Storing:** The vector and the original text are stored together in the vector database.

### 2. Retrieval (Recalling Memories)
This is not automatic. The agent must decide to search its memory.
- **Query:** The agent takes a new prompt or concept, creates a vector for it, and queries the database.
- **Search:** The database finds the stored text chunks with the most mathematically similar vectors.
- **Augment:** The agent takes these retrieved chunks and adds them to its current "working memory" before sending the prompt to the LLM. This gives the LLM relevant historical context to "augment" its reasoning.

### Key Trade-off
The quality of RAG depends entirely on the quality of the retrieval. If the search returns irrelevant information ("Garbage In"), the LLM's final answer will be of poor quality ("Garbage Out").
