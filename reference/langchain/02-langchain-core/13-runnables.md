# runnables

> **Module** in `langchain_core`

📖 [View in docs](https://reference.langchain.com/python/langchain-core/runnables)

LangChain **Runnable** and the **LangChain Expression Language (LCEL)**.

The LangChain Expression Language (LCEL) offers a declarative method to build
production-grade programs that harness the power of LLMs.

Programs created using LCEL and LangChain `Runnable` objects inherently support
synchronous asynchronous, batch, and streaming operations.

Support for **async** allows servers hosting LCEL based programs to scale bette for
higher concurrent loads.

**Batch** operations allow for processing multiple inputs in parallel.

**Streaming** of intermediate outputs, as they're being generated, allows for creating
more responsive UX.

This module contains schema and implementation of LangChain `Runnable` object
primitives.

## Properties

- `RunnableMap`

## Methods

- [`import_attr()`](https://reference.langchain.com/python/langchain-core/runnables/import_attr)
- [`chain()`](https://reference.langchain.com/python/langchain-core/runnables/chain)
- [`ensure_config()`](https://reference.langchain.com/python/langchain-core/runnables/ensure_config)
- [`get_config_list()`](https://reference.langchain.com/python/langchain-core/runnables/get_config_list)
- [`patch_config()`](https://reference.langchain.com/python/langchain-core/runnables/patch_config)
- [`run_in_executor()`](https://reference.langchain.com/python/langchain-core/runnables/run_in_executor)
- [`aadd()`](https://reference.langchain.com/python/langchain-core/runnables/aadd)
- [`add()`](https://reference.langchain.com/python/langchain-core/runnables/add)

---

[View source on GitHub](https://github.com/langchain-ai/langchain/blob/65bbd47cb2721c51ef8638f9e7da35247c4bfdde/libs/core/langchain_core/runnables/__init__.py)