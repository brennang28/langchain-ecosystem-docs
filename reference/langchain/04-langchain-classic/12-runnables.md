# runnables

> **Module** in `langchain_classic`

📖 [View in docs](https://reference.langchain.com/python/langchain-classic/runnables)

LangChain **Runnable** and the **LangChain Expression Language (LCEL)**.

The LangChain Expression Language (LCEL) offers a declarative method to build
production-grade programs that harness the power of LLMs.

Programs created using LCEL and LangChain Runnables inherently support
synchronous, asynchronous, batch, and streaming operations.

Support for **async** allows servers hosting the LCEL based programs
to scale better for higher concurrent loads.

**Batch** operations allow for processing multiple inputs in parallel.

**Streaming** of intermediate outputs, as they're being generated, allows for
creating more responsive UX.

This module contains non-core Runnable classes.

---

[View source on GitHub](https://github.com/langchain-ai/langchain/blob/65bbd47cb2721c51ef8638f9e7da35247c4bfdde/libs/langchain/langchain_classic/runnables/__init__.py)