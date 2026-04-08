> ## Documentation Index
> Fetch the complete documentation index at: https://docs.langchain.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Test

> Strategies for testing LangChain agents, including unit tests, integration tests, and trajectory evaluations.

Agentic applications let an LLM decide its own next steps to solve a problem. That flexibility is powerful, but the model's black-box nature makes it hard to predict how a tweak in one part of your agent will affect the whole. To build production-ready agents, thorough testing is essential.

There are a few approaches to testing your agents:

* **Unit tests** exercise small, deterministic pieces of your agent in isolation using in-memory fakes so you can assert exact behavior quickly and deterministically.
* **Integration tests** test the agent using real network calls to confirm that components work together, credentials and schemas line up, and latency is acceptable.
* **Evals** use evaluators to assess your agent's execution trajectory, either via deterministic matching or an LLM judge.

Agentic applications tend to lean more on integration because they chain multiple components together and must deal with flakiness due to the nondeterministic nature of LLMs.

Mock chat models and use in-memory persistence to test agent logic without API calls.
  
Test your agent with real LLM APIs. Organize tests, manage keys, handle flakiness, and control costs.
  
Evaluate agent trajectories with deterministic matching or LLM-as-judge evaluators.
  

***


  
> ℹ️ **Note:**
>
> [Edit this page on GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/test/index.mdx) or [file an issue](https://github.com/langchain-ai/docs/issues/new/choose).


  
> ℹ️ **Note:**
>
> [Connect these docs](/use-these-docs) to Claude, VSCode, and more via MCP for real-time answers.

