---
title: Playground
description: Test, iterate, and compare different prompts and models within the LLM Playground.
sidebarTitle: Playground
---

# LLM Playground

Test and iterate on your prompts directly in the Langfuse Prompt Playground. Tweak the prompt and model parameters to see how different models respond to these input changes. This allows you to quickly iterate on your prompts and optimize them for the best results in your LLM app without having to switch between tools or use any code.


![LLM Playground](/images/docs/playground-overview.png)
## Core features

### Side-by-Side Comparison View

Compare multiple prompt variants alongside each other. Execute them all at once or focus on a single variant. Each variant keeps its own LLM settings, variables, tool definitions, and placeholders so you can immediately see the impact of every change.

### Open your prompt in the playground

You can open a prompt you created with [Langfuse Prompt Management](/docs/prompt-management/get-started) in the playground.

### Save your prompt to Prompt Management

When you're satisfied with your prompt, you can save it to Prompt Management by clicking the save button.

### Open a generation in the playground

You can open a generation from [Langfuse Observability](/docs/observability) in the playground by clicking the `Open in Playground` button in the generation details page.

### Tool calling and structured outputs

The Langfuse Playground supports tool calling and structured output schemas, enabling you to define, test, and validate LLM executions that rely on tool calls and enforce specific response formats.


> ℹ️ **Note:** Currently, Langfuse supports opening tool-type observations in the playground
>   only when they are in the OpenAI ChatML format. If you’d like to see support
>   for additional formats, feel free to add your request to our [public
>   roadmap](https://langfuse.com/ideas).


<iframe
  width="100%"
  src="https://www.youtube-nocookie.com/embed/IkSK5Kz-Pt8?si=ksiM2YzZC84HEJC_"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
  referrerpolicy="strict-origin-when-cross-origin"
  allowFullScreen
></iframe>

**Tool Calling**

- Define custom tools with JSON schema definitions
- Test prompts relying on tools in real-time by mocking tool responses
- Save tool definitions to your project

**Structured Output**

- Enforce response formats using JSON schemas
- Save schemas to your project
- Jump into the playground from your OpenAI generation using structured output

### Add prompt variables

You can add prompt variables in the playground to simulate different inputs to your prompt.


![Add prompt variables](/images/docs/playground-variables.png)

### Use your favorite model

You can use your favorite model by adding the API key for the model you want to use in the Langfuse project settings. You can learn how to set up an LLM connection [here](/docs/administration/llm-connection).


![Select the model you want to
  use](/images/docs/playground-model-selection.png)

Optionally, many LLM providers allow for additional parameters when invoking a model. You can pass these parameters in the playground when toggling "Additional Options" in the model selection dropdown. [Read this documentation about additional provider options](/docs/administration/llm-connection#advanced-configurations) for more information.

## Related Resources

- For structured regression testing of prompt and model variants on datasets, use [Experiments](/docs/evaluation/core-concepts#experiments).

## GitHub Discussions

