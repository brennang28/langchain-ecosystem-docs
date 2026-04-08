# chains

> **Module** in `langchain_classic`

📖 [View in docs](https://reference.langchain.com/python/langchain-classic/chains)

**Chains** are easily reusable components linked together.

Chains encode a sequence of calls to components like models, document retrievers,
other Chains, etc., and provide a simple interface to this sequence.

The Chain interface makes it easy to create apps that are:

    - **Stateful:** add Memory to any Chain to give it state,
    - **Observable:** pass Callbacks to a Chain to execute additional functionality,
        like logging, outside the main sequence of component calls,
    - **Composable:** combine Chains with other components, including other Chains.

## Properties

- `importer`

## Methods

- [`create_importer()`](https://reference.langchain.com/python/langchain-classic/chains/create_importer)

---

[View source on GitHub](https://github.com/langchain-ai/langchain/blob/65bbd47cb2721c51ef8638f9e7da35247c4bfdde/libs/langchain/langchain_classic/chains/__init__.py)