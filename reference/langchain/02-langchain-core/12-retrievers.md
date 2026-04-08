# retrievers

> **Module** in `langchain_core`

📖 [View in docs](https://reference.langchain.com/python/langchain-core/retrievers)

**Retriever** class returns `Document` objects given a text **query**.

It is more general than a vector store. A retriever does not need to be able to
store documents, only to return (or retrieve) it. Vector stores can be used as
the backbone of a retriever, but there are other types of retrievers as well.

## Properties

- `RetrieverInput`
- `RetrieverOutput`
- `RetrieverLike`
- `RetrieverOutputLike`

## Methods

- [`ensure_config()`](https://reference.langchain.com/python/langchain-core/retrievers/ensure_config)
- [`run_in_executor()`](https://reference.langchain.com/python/langchain-core/retrievers/run_in_executor)

---

[View source on GitHub](https://github.com/langchain-ai/langchain/blob/65bbd47cb2721c51ef8638f9e7da35247c4bfdde/libs/core/langchain_core/retrievers.py)