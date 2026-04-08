# documents

> **Module** in `langchain_core`

📖 [View in docs](https://reference.langchain.com/python/langchain-core/documents)

Documents module for data retrieval and processing workflows.

This module provides core abstractions for handling data in retrieval-augmented
generation (RAG) pipelines, vector stores, and document processing workflows.

!!! warning "Documents vs. message content"

    This module is distinct from `langchain_core.messages.content`, which provides
    multimodal content blocks for **LLM chat I/O** (text, images, audio, etc. within
    messages).

    **Key distinction:**

    - **Documents** (this module): For **data retrieval and processing workflows**
        - Vector stores, retrievers, RAG pipelines
        - Text chunking, embedding, and semantic search
        - Example: Chunks of a PDF stored in a vector database

    - **Content Blocks** (`messages.content`): For **LLM conversational I/O**
        - Multimodal message content sent to/from models
        - Tool calls, reasoning, citations within chat
        - Example: An image sent to a vision model in a chat message (via
            [`ImageContentBlock`][langchain.messages.ImageContentBlock])

    While both can represent similar data types (text, files), they serve different
    architectural purposes in LangChain applications.

## Methods

- [`import_attr()`](https://reference.langchain.com/python/langchain-core/documents/import_attr)

---

[View source on GitHub](https://github.com/langchain-ai/langchain/blob/65bbd47cb2721c51ef8638f9e7da35247c4bfdde/libs/core/langchain_core/documents/__init__.py)