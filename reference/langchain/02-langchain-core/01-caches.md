# caches

> **Module** in `langchain_core`

📖 [View in docs](https://reference.langchain.com/python/langchain-core/caches)

Optional caching layer for language models.

Distinct from provider-based [prompt caching](https://docs.langchain.com/oss/python/langchain/models#prompt-caching).

!!! warning "Beta feature"

    This is a beta feature. Please be wary of deploying experimental code to production
    unless you've taken appropriate precautions.

A cache is useful for two reasons:

1. It can save you money by reducing the number of API calls you make to the LLM
    provider if you're often requesting the same completion multiple times.
2. It can speed up your application by reducing the number of API calls you make to the
    LLM provider.

## Properties

- `RETURN_VAL_TYPE`

## Methods

- [`run_in_executor()`](https://reference.langchain.com/python/langchain-core/caches/run_in_executor)

---

[View source on GitHub](https://github.com/langchain-ai/langchain/blob/65bbd47cb2721c51ef8638f9e7da35247c4bfdde/libs/core/langchain_core/caches.py)