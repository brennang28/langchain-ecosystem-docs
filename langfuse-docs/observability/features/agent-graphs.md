---
title: Agent Graphs
description: Visualize and analyze complex agent workflows with Langfuse's agent graph view.
sidebarTitle: Agent Graphs
---

# Agent Graphs

Agent graphs in Langfuse provide a visual representation of complex AI agent workflows, helping you understand and debug multi-step reasoning processes and agent interactions.

_Example trace with agent graph view ([public link](https://cloud.langfuse.com/project/cloramnkj0002jz088vzn1ja4/traces/8ed12d68-353f-464f-bc62-720984c3b6a0))_

## Get Started


> ℹ️ **Note:** The graph view is currently in beta, please feel free to share feedback.


There are currently two ways to display the graph.
First, have an observation with any observation type except for `span`, `event` or `generation` in your trace. Then, Langfuse interprets the trace as agentic and will show a graph. The graph is automatically inferred from the observation timings as well as their nesting.
Second, when you use the LangGraph integration the graph automatically shows in Langfuse.

**Observation Types**: See all available [Observation Types](/docs/observability/features/observation-types) and how to set them.
**LangGraph**: See the [LangGraph integration guide](/guides/cookbook/integration_langgraph) for an end-to-end example on how to natively integrate LangGraph with Langfuse for LLM Agent tracing.

## GitHub Discussions

