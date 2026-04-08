---
title: Overview
description: Langfuse is an open-source LLM engineering platform (GitHub) that helps teams collaboratively debug, analyze, and iterate on their LLM applications. All platform features are natively integrated to accelerate the development workflow.
---

# Langfuse Overview

Langfuse is an open-source LLM engineering platform ([GitHub](https://github.com/langfuse/langfuse)) that helps teams collaboratively debug, analyze, and iterate on their LLM applications. All platform features are natively integrated to accelerate the development workflow. Langfuse is open, self-hostable, and extensible ([_why langfuse?_](/why)).

## Observability [#observability]

[Observability](/docs/observability/overview) is essential for understanding and debugging LLM applications. Unlike traditional software, LLM applications involve complex, non-deterministic interactions that can be challenging to monitor and debug. Langfuse provides comprehensive tracing capabilities that help you understand exactly what's happening in your application.

- Traces include all LLM and non-LLM calls, including retrieval, embedding, API calls, and more
- Support for tracking multi-turn conversations as sessions and user tracking
- Agents can be represented as graphs
- Capture traces via our native SDKs for Python/JS, 50+ library/framework integrations, OpenTelemetry, or via an LLM Gateway such as LiteLLM
- Based on OpenTelemetry to increase compatibility and reduce vendor lock-in

Want to see an example? Play with the [interactive demo](/docs/demo).


> ℹ️ **Note:** Want to learn more? [**Watch end-to-end walkthrough**](/watch-demo) of Langfuse Observability and how to integrate it with your application.


Traces allow you to track every LLM call and other relevant logic in your app.

Sessions allow you to track multi-step conversations or agentic workflows.

Debug latency issues by inspecting the timeline view.

Add your own `userId` to monitor costs and usage for each user. Optionally, create a deep link to this view in your systems.

LLM agents can be visualized as a graph to illustrate the flow of complex agentic workflows.

See quality, cost, and latency metrics in the dashboard to monitor your LLM application.

## Prompt Management [#prompts]

[Prompt Management](/docs/prompt-management/overview) is critical in building effective LLM applications. Langfuse provides tools to help you manage, version, and optimize your prompts throughout the development lifecycle.

- [Get started](/docs/prompt-management/get-started) with prompt management
- Manage, version, and optimize your prompts throughout the development lifecycle
- Test prompts interactively in the [LLM Playground](/docs/prompt-management/features/playground)
- Run [Experiments](/docs/evaluation/features/prompt-experiments) against datasets to test new prompt versions directly within Langfuse


> ℹ️ **Note:** Want to learn more? [**Watch end-to-end walkthrough**](/watch-demo?tab=prompt) of Langfuse Prompt Management and how to integrate it with your application.


Create a new prompt via UI, SDKs, or API.

Collaboratively version and edit prompts via UI, API, or SDKs.

Deploy prompts to production or any environment via labels - without any code changes.

Compare latency, cost, and evaluation metrics across different versions of your prompts.

Instantly test your prompts in the playground.

Link prompts with traces to understand how they perform in the context of your LLM application.

Track changes to your prompts to understand how they evolve over time.

## Evaluation [#evaluation]

[Evaluation](/docs/evaluation/overview) is crucial for ensuring the quality and reliability of your LLM applications. Langfuse provides flexible evaluation tools that adapt to your specific needs, whether you're testing in development or monitoring production performance.

- Get started with different [evaluation methods](/docs/evaluation/overview): LLM-as-a-judge, user feedback, manual labeling, or custom
- Identify issues early by running evaluations on production traces
- Create and manage [Datasets](/docs/evaluation/features/datasets) for systematic testing in development that ensure your application performs reliably across different scenarios
- Run [Experiments](/docs/evaluation/core-concepts#experiments) to systematically test your LLM application


> ℹ️ **Note:** Want to learn more? [**Watch end-to-end walkthrough**](/watch-demo?tab=evaluation) of Langfuse Evaluation and how to use it to improve your LLM application.


Plot evaluation results in the Langfuse Dashboard.

Collect feedback from your users. Can be captured in the frontend via our Browser SDK, server-side via the SDKs or API. Video includes example application.

Run fully managed LLM-as-a-judge evaluations on production or development traces. Can be applied to any step within your application for step-wise evaluations.

Evaluate prompts and models on datasets directly in the user interface. No custom code is needed.

Baseline your evaluation workflow with human annotations via Annotation Queues.

Add custom evaluation results, supports numeric, boolean and categorical values.

```bash
POST /api/public/scores
```

Add scores via Python or JS SDK.

```python
langfuse.score(
  trace_id="123",
  name="my_custom_evaluator",
  value=0.5,
)
```


## Where to start?

Setting up the full process of online tracing, prompt management, production evaluations to identify issues, and offline evaluations on datasets requires some time. This guide is meant to help you figure out what is most important for your use case.

_Simplified lifecycle from PoC to production:_


![Langfuse Features along the development
  lifecycle](/images/docs/features-light.png)

![Langfuse Features along the development
  lifecycle](/images/docs/features-dark.png)

## Quickstarts

Get up and running with Langfuse in minutes. Choose the path that best fits your current needs:


}
    title="Integrate LLM Application/Agent Tracing"
    href="/docs/observability/get-started"
    arrow
  />
  }
    title="Integrate Prompt Management"
    href="/docs/prompt-management/get-started"
    arrow
  />
  }
    title="Setup Evaluations"
    href="/docs/evaluation/overview"
    arrow
  />

## Why Langfuse?

- **Open source:** Fully open source with public API for custom integrations
- **Production optimized:** Designed with minimal performance overhead
- **Best-in-class SDKs:** Native SDKs for Python and JavaScript
- **Framework support:** Integrated with popular frameworks like OpenAI SDK, LangChain, and LlamaIndex
- **Multi-modal:** Support for tracing text, images and other modalities
- **Full platform:** Suite of tools for the complete LLM application development lifecycle

## Community & Contact

We actively develop Langfuse in [open source](/open-source) together with our community:

- Contribute and vote on the Langfuse [roadmap](/docs/roadmap).
- Ask questions on [GitHub Discussions](/gh-support) or private [support channels](/support).
- Report bugs via [GitHub Issues](/issue).
- Chat with the community on [Discord](/discord).
- [Why people choose Langfuse?](/why)

Langfuse evolves quickly, check out the [changelog](/changelog) for the latest updates. Subscribe to the **mailing list** to get notified about new major features:

