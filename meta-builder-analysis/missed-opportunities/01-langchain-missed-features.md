# LangChain Missed Features: Deep Dive Analysis for meta-builder-v3

> **Version:** meta-builder-v3 v4.9.0  
> **Analysis Date:** April 2026  
> **Scope:** Features present in the LangChain ecosystem that meta-builder has not adopted

---

## Executive Summary

meta-builder-v3 uses LangChain as its underlying agent runtime but interacts with it primarily at the primitive level: raw chat model constructors, message types, and the `@tool` decorator. The framework has evolved substantially since meta-builder's core was designed. The gap is not cosmetic — it represents entire architectural layers that meta-builder reimplements manually, inconsistently, or not at all.

This document covers **ten missed capability areas** with full code examples, concrete pain points, integration targets, and effort/benefit assessments.

---

## Table of Contents

1. [Middleware System (`create_agent` + `middleware=[]`)](#1-middleware-system)
2. [Model Context Protocol (MCP) Integration](#2-model-context-protocol-mcp-integration)
3. [Context Engineering Framework](#3-context-engineering-framework)
4. [Long-Term Memory via LangGraph Store](#4-long-term-memory-via-langgraph-store)
5. [Agent Evals Framework (`agentevals`)](#5-agent-evals-framework-agentevals)
6. [`create_agent()` Factory](#6-create_agent-factory)
7. [Voice Agent Architecture](#7-voice-agent-architecture)
8. [Agent Chat UI](#8-agent-chat-ui)
9. [Generative UI Patterns](#9-generative-ui-patterns)
10. [Rate Limiters (`langchain-core`)](#10-rate-limiters-langchain-core)

---

## 1. Middleware System

### What It Is

LangChain's middleware system hooks into the agent execution loop at two precise points: **before and after the model call**. Middleware is registered via the `middleware=[]` argument on `create_agent()`. The system composes multiple handlers in declaration order. Each middleware can:

- Inspect and mutate `AgentState` (message history, custom state fields)
- Intercept and modify `ModelRequest` before it reaches the LLM
- Intercept and modify `ModelResponse` after the LLM returns
- Short-circuit the execution (e.g., jump to `"end"`)
- Access `langgraph.runtime.Runtime` for store, config, and context

LangChain ships a full suite of production-ready prebuilt middleware modules. Builders can also author custom middleware using decorator-based or class-based APIs.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    SummarizationMiddleware,
    HumanInTheLoopMiddleware,
    ModelCallLimitMiddleware,
    ToolCallLimitMiddleware,
    ModelFallbackMiddleware,
    PIIMiddleware,
    ToolRetryMiddleware,
    ModelRetryMiddleware,
    LLMToolSelectorMiddleware,
    ContextEditingMiddleware,
    ClearToolUsesEdit,
    TodoListMiddleware,
    ShellToolMiddleware,
    HostExecutionPolicy,
    FilesystemFileSearchMiddleware,
)
from deepagents.middleware.filesystem import FilesystemMiddleware
from deepagents.middleware.subagents import SubAgentMiddleware
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[your_tool],
    checkpointer=InMemorySaver(),
    middleware=[
        SummarizationMiddleware(model="openai:gpt-4.1-mini", trigger=("fraction", 0.8), keep=("messages", 20)),
        ModelRetryMiddleware(max_retries=3, backoff_factor=2.0),
        ToolRetryMiddleware(max_retries=3, initial_delay=1.0, jitter=True),
        ModelCallLimitMiddleware(thread_limit=100, run_limit=20, exit_behavior="end"),
        ToolCallLimitMiddleware(thread_limit=50, run_limit=10),
        ModelFallbackMiddleware("openai:gpt-4.1", "anthropic:claude-haiku-3-5"),
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        LLMToolSelectorMiddleware(model="openai:gpt-4.1-mini", max_tools=5),
        ContextEditingMiddleware(edits=[ClearToolUsesEdit(trigger=80000, keep=3)]),
        TodoListMiddleware(),
        ShellToolMiddleware(workspace_root="/workspace", execution_policy=HostExecutionPolicy()),
        FilesystemMiddleware(),
    ],
)
```

---

### 1.1 SummarizationMiddleware

**What it does:** Monitors the message history token count and automatically summarizes older messages when a threshold is reached. Runs in `before_model()` — the summary is generated before the next LLM call, not during.

**API reference:** [`SummarizationMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/SummarizationMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

# Token-count trigger (most common):
agent = create_agent(
    model="gpt-4.1",
    tools=[your_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-4.1-mini",          # cheaper model for summaries
            trigger=("tokens", 4000),       # summarize when >4000 tokens
            keep=("messages", 20),          # preserve 20 most recent messages
            summary_prompt="Summarize the conversation, preserving decisions, tool outputs, and outstanding tasks.",
        ),
    ],
)

# Fractional trigger (auto-adapts to the model's context window):
agent2 = create_agent(
    model="gpt-4.1",
    tools=[your_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-4.1-mini",
            trigger=("fraction", 0.8),     # when 80% of context window is used
            keep=("fraction", 0.3),         # keep 30% of context window
        ),
    ],
)

# OR-logic: trigger on EITHER condition:
agent3 = create_agent(
    model="gpt-4.1",
    tools=[your_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-4.1-mini",
            trigger=[("tokens", 3000), ("messages", 50)],
            keep=("messages", 20),
        ),
    ],
)
```

**Why meta-builder needs this:**

meta-builder's `Conductor` orchestrates multi-step pipeline execution. Each step appends tool calls, results, and planner reflections to the message history. There is no mechanism to cap this growth. For complex pipelines with 20+ nodes and iterative replanning, context overflow is not a hypothetical — it is an observed failure mode. The current workaround is `max_replans`, which aborts the pipeline rather than summarizing it.

`SummarizationMiddleware` would allow the Conductor to handle arbitrarily long pipelines gracefully, preserving only recent and relevant context while compressing older exchanges into structured summaries.

**Where to integrate:**

- `meta_builder/orchestration/conductor.py` — any `create_agent()` call that drives Conductor's multi-step loop
- `meta_builder/agents/vision_agent.py` — long multi-turn requirement extraction sessions
- `meta_builder/agents/evaluator_agent.py` — long evaluation sessions with many tool calls

**Integration plan:**

1. Add `SummarizationMiddleware` to Conductor's `create_agent()` call with `trigger=("fraction", 0.75)` and `keep=("messages", 30)`.
2. Write a custom `summary_prompt` that instructs the summarizer to preserve: current pipeline state, completed nodes, outstanding replanning decisions, and any error history.
3. Add a custom `token_counter` that matches the tokenizer of the primary model being used (avoids undercounting with the default character-based counter).

**Impact:** High benefit, low effort. Drop-in addition to `create_agent()` calls. No architectural changes required. Eliminates a class of silent context-overflow failures.

---

### 1.2 HumanInTheLoopMiddleware

**What it does:** Pauses agent execution before executing specified tool calls, yielding control to a human reviewer who can approve, edit the arguments, or reject the call entirely. Requires a `checkpointer` to persist state during the pause.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="gpt-4.1",
    tools=[read_file, write_file, deploy_to_prod, delete_database],
    checkpointer=InMemorySaver(),
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "deploy_to_prod": {
                    "allowed_decisions": ["approve", "edit", "reject"],
                },
                "delete_database": {
                    "allowed_decisions": ["approve", "reject"],  # no editing
                },
                "read_file": False,   # never interrupt on reads
                "write_file": True,   # always interrupt, default decisions
            }
        ),
    ],
)
```

**Why meta-builder needs this:**

meta-builder's `VisionAgent` has a concept of human interaction via `RequirementsSlots`, but this is a pre-planning interview pattern. There is no mechanism to pause mid-execution when the generated agent calls a tool with significant side effects (e.g., deploying code, deleting data, sending external messages). The `RemediationAgent` and `EvaluatorAgent` can trigger high-impact tool calls without any approval gate.

`HumanInTheLoopMiddleware` provides a standardized, checkpointed interrupt pattern. Critically, it works at the tool-call level — you can selectively interrupt on destructive tools while letting read-only tools pass through unimpeded.

**Where to integrate:**

- `meta_builder/agents/remediation_agent.py` — before `execute_shell_command`, `write_file`, or `apply_patch`
- `meta_builder/agents/vision_agent.py` — optionally before finalizing `WorkflowDesign` for user confirmation
- Any generated agent with `deploy_*` or `delete_*` tools

**Integration plan:**

1. Add `checkpointer` to all Conductor-managed agents (required prerequisite for HITL).
2. Add `HumanInTheLoopMiddleware` to the RemediationAgent with interrupts on file-write and shell-exec tools.
3. Expose a UI endpoint in meta-builder's API to receive human decisions and resume the interrupted thread.

**Impact:** High benefit for production safety, medium effort. Requires the checkpointer infrastructure to be in place first.

---

### 1.3 ModelCallLimit and ToolCallLimit Middleware

**What they do:** Impose hard caps on the number of LLM API calls (ModelCallLimit) or tool executions (ToolCallLimit) per thread or per run invocation.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelCallLimitMiddleware, ToolCallLimitMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[search, database_query, web_scraper],
    checkpointer=InMemorySaver(),
    middleware=[
        # Global model call budget
        ModelCallLimitMiddleware(
            thread_limit=100,   # max 100 model calls across an entire conversation
            run_limit=20,       # max 20 model calls in a single .invoke()
            exit_behavior="end", # terminate gracefully rather than error
        ),
        # Global tool budget
        ToolCallLimitMiddleware(thread_limit=200, run_limit=50),
        # Per-tool budgets for expensive operations
        ToolCallLimitMiddleware(tool_name="web_scraper", run_limit=5, exit_behavior="error"),
        ToolCallLimitMiddleware(tool_name="database_query", thread_limit=50),
    ],
)
```

**Why meta-builder needs this:**

meta-builder has `max_replans` in the Conductor's configuration, but this only caps the replan cycle count — it does not cap individual model calls within a replan cycle, and it does not cap tool calls at all. A generated agent in an infinite tool-use loop, or a Conductor stuck retrying a bad plan, can run up unbounded API costs with no circuit breaker.

**Where to integrate:**

- `meta_builder/orchestration/conductor.py` — `ModelCallLimitMiddleware` with configurable `run_limit` per pipeline run
- `meta_builder/llm/llm_factory.py` — surface `run_limit` as a configurable parameter when constructing agents
- All generated agents' compiled graphs — pass limits through the `AgentConfig` schema

**Impact:** High value for cost control and runaway prevention, low effort. Pure addition.

---

### 1.4 ModelFallbackMiddleware

**What it does:** Automatically tries a list of fallback models when the primary model fails (network error, rate limit, capacity error). Falls through the list in order.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelFallbackMiddleware

agent = create_agent(
    model="anthropic:claude-sonnet-4-6",    # primary
    tools=[],
    middleware=[
        ModelFallbackMiddleware(
            "openai:gpt-4.1",               # first fallback
            "anthropic:claude-3-5-haiku",   # second fallback
            "openai:gpt-4.1-mini",          # emergency fallback
        ),
    ],
)
```

**Why meta-builder needs this:**

meta-builder has an ad-hoc `fallback_llm_client` concept in the LLM Factory, but it is not wired into a standardized retry/fallback pattern. The fallback is a configuration option — it is not automatically invoked on transient model failures. When Claude is unavailable, meta-builder silently fails rather than routing to GPT-4.1.

**Where to integrate:**

- `meta_builder/llm/llm_factory.py` — replace the `fallback_llm_client` pattern with `ModelFallbackMiddleware`
- `meta_builder/orchestration/conductor.py` — apply to all Conductor agents

**Impact:** Medium benefit, low effort. Improves reliability in multi-provider deployments.

---

### 1.5 PIIMiddleware

**What it does:** Detects and handles Personally Identifiable Information (PII) in conversations. Supports built-in types (`email`, `credit_card`, `ip`, `mac_address`, `url`) and custom detectors. Four strategies: `block`, `redact`, `mask`, `hash`. Can apply to input (user messages), output (AI messages), or tool results independently.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware
import re

# Built-in PII types:
agent = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        PIIMiddleware("email", strategy="redact", apply_to_input=True, apply_to_output=True),
        PIIMiddleware("credit_card", strategy="mask", apply_to_input=True),
        PIIMiddleware("ip", strategy="hash", apply_to_tool_results=True),
    ],
)

# Custom detector — regex:
agent2 = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        PIIMiddleware(
            "api_key",
            detector=r"sk-[a-zA-Z0-9]{32}",
            strategy="block",
            apply_to_input=True,
            apply_to_output=True,
        ),
    ],
)

# Custom detector — validation function:
def detect_ssn(content: str) -> list[dict]:
    matches = []
    for match in re.finditer(r"\d{3}-\d{2}-\d{4}", content):
        first_three = int(match.group(0)[:3])
        if first_three not in [0, 666] and not (900 <= first_three <= 999):
            matches.append({"text": match.group(0), "start": match.start(), "end": match.end()})
    return matches

agent3 = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        PIIMiddleware("ssn", detector=detect_ssn, strategy="hash", apply_to_input=True),
    ],
)
```

**Why meta-builder needs this:**

meta-builder's `SafetyGuardrails` middleware uses basic regex patterns to detect a fixed set of PII types. It has no concept of:
- Selective application (input vs output vs tool results)
- Multiple concurrent PII types with different strategies
- The `hash` strategy (useful for logging/analytics while preserving lookup capability)
- The `block` strategy (raising an exception to prevent processing entirely)
- Custom validation beyond regex (e.g., Luhn-check credit cards, validate SSN against known-invalid ranges)

`PIIMiddleware` is substantially more sophisticated, composable, and production-grade. Critically, it operates at the middleware layer rather than being bolted on as a separate pre-processing step — it fires in the correct position relative to model calls.

**Where to integrate:**

- `meta_builder/middleware/safety_guardrails.py` — replace the custom implementation entirely
- All `create_agent()` calls — apply `PIIMiddleware` as a default in the agent factory

**Impact:** High security benefit, low-medium effort. Requires rethinking how SafetyGuardrails is structured, but the replacement is simpler than the current implementation.

---

### 1.6 ToolRetry and ModelRetry Middleware

**What they do:** Automatically retry failed tool calls (ToolRetryMiddleware) or model API calls (ModelRetryMiddleware) with configurable exponential backoff. Support custom exception filtering, custom failure handlers, and jitter.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolRetryMiddleware, ModelRetryMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[external_api_tool, database_tool],
    middleware=[
        # Retry all tools on any exception, up to 3 times:
        ToolRetryMiddleware(
            max_retries=3,
            backoff_factor=2.0,
            initial_delay=1.0,
            max_delay=60.0,
            jitter=True,         # add ±25% random jitter to avoid thundering herd
            on_failure="return_message",  # return error to LLM rather than raising
        ),

        # Only retry specific tools on specific exceptions:
        ToolRetryMiddleware(
            tools=["external_api_tool"],
            max_retries=5,
            retry_on=(ConnectionError, TimeoutError),
            on_failure="raise",  # propagate unrecoverable failures
        ),

        # Retry model calls with rate-limit awareness:
        ModelRetryMiddleware(
            max_retries=4,
            retry_on=lambda e: getattr(e, "status_code", None) in (429, 503),
            backoff_factor=3.0,
            on_failure="continue",  # return error message, let agent decide
        ),
    ],
)
```

**Why meta-builder needs this:**

meta-builder has manual retry logic scattered across the codebase:
- The LLM Factory has some retry wrapping for model calls
- The RemediationAgent re-runs failed steps manually
- Tool implementations in ToolRegistry may or may not have their own retry logic

This is inconsistent. Some tools are retried, some are not. Some retry with linear backoff, some exponentially. None are coordinated with each other. There is no central configuration point.

`ToolRetryMiddleware` and `ModelRetryMiddleware` centralize this concern. They are composable (stack them), configurable per-tool, and operate at the agent loop level where they have full context.

**Where to integrate:**

- `meta_builder/orchestration/conductor.py` — wrap all Conductor agents with both middlewares
- `meta_builder/tools/tool_registry.py` — remove per-tool retry logic, centralize here
- `meta_builder/llm/llm_factory.py` — remove manual retry wrapping around model clients

**Impact:** High reliability benefit, medium effort. Requires auditing and removing scattered retry logic across the codebase.

---

### 1.7 LLMToolSelectorMiddleware

**What it does:** Before each model call, uses a cheaper/smaller LLM to select the most relevant subset of tools from the full registry. Passes only the selected tools in the model's context window, reducing token waste and improving model focus.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolSelectorMiddleware

# Register all tools, but only pass 3-5 at a time:
agent = create_agent(
    model="gpt-4.1",
    tools=[
        search_tool, database_tool, file_tool, email_tool,
        calendar_tool, code_tool, deploy_tool, test_runner,
        # ...30+ more tools
    ],
    middleware=[
        LLMToolSelectorMiddleware(
            model="gpt-4.1-mini",      # cheap model for selection
            max_tools=5,               # pass at most 5 tools per call
            always_include=["search_tool"],  # always available
            system_prompt=(
                "Select the tools most likely needed for the current conversation step. "
                "Prefer specific tools over general ones."
            ),
        ),
    ],
)
```

**Why meta-builder needs this:**

meta-builder's `ToolRegistry` exposes **all registered tools** to every node in the generated pipeline. This means:

1. Every model call includes tool schema definitions for tools that will never be used in the current context — pure token waste.
2. Models with 30+ tool schemas in context are more likely to hallucinate incorrect tool names or arguments.
3. For generated agents with domain-specific tool sets, many tools are context-irrelevant for most steps.

`LLMToolSelectorMiddleware` uses structured output to intelligently pre-filter before each call. It uses a cheap secondary model, so the overhead is minimal relative to the token savings.

**Where to integrate:**

- `meta_builder/tools/tool_registry.py` — expose `get_all_tools()` but also expose `get_tools_for_context()` that can be backed by the selector
- `meta_builder/orchestration/conductor.py` — add `LLMToolSelectorMiddleware` with threshold logic (e.g., only activate when tool count exceeds 10)

**Impact:** High benefit for large tool registries, low effort. Transparent to agent logic.

---

### 1.8 ContextEditingMiddleware

**What it does:** Manages conversation context by selectively clearing older tool call outputs (which can be very large) from the message history, while preserving the most recent N results. Prevents tool result bloat from consuming the context window.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit

agent = create_agent(
    model="gpt-4.1",
    tools=[web_search, file_reader, code_executor],
    middleware=[
        ContextEditingMiddleware(
            edits=[
                ClearToolUsesEdit(
                    trigger=80_000,       # activate when message history >80k tokens
                    keep=3,              # preserve last 3 tool result pairs
                    clear_tool_inputs=False,  # keep the call args for context
                    exclude_tools=["file_reader"],  # never clear file reads
                    placeholder="[output cleared to save context]",
                ),
            ],
            token_count_method="approximate",
        ),
    ],
)
```

**Why meta-builder needs this:**

Generated agents that call search tools, file readers, or code executors accumulate large tool results in their message history. Each web search result might be 5,000+ characters. After 20 search calls, the context is dominated by stale results that are no longer relevant to the current step. meta-builder has no mechanism to trim this growth.

`ContextEditingMiddleware` solves this by operating at the middleware layer — it inspects and prunes the message list before each model call, keeping only the most recent tool outputs while replacing older ones with placeholder text.

**Where to integrate:**

- `meta_builder/orchestration/conductor.py` — add alongside `SummarizationMiddleware`; these are complementary (summarization compresses messages, context editing clears specific tool outputs)
- All generated agents that use search or retrieval tools

**Impact:** Medium-high benefit for agents with large tool outputs, low effort.

---

### 1.9 SubagentMiddleware

**What it does:** Adds a `task` tool to the main agent that spawns isolated subagents. Each subagent runs with its own context window, tools, model, and optional additional middleware. The main agent receives a clean summary result, not the subagent's full intermediate state.

```python
from langchain.tools import tool
from langchain.agents import create_agent
from deepagents.middleware.subagents import SubAgentMiddleware

@tool
def get_weather(city: str) -> str:
    """Get the weather for a city."""
    return f"Sunny in {city}, 72°F."

@tool
def search_news(query: str) -> str:
    """Search news articles."""
    return f"News results for: {query}"

agent = create_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[],                              # main agent has no direct tools
    middleware=[
        SubAgentMiddleware(
            default_model="anthropic:claude-sonnet-4-6",
            default_tools=[],
            subagents=[
                {
                    "name": "weather_specialist",
                    "description": "Gets weather data for cities and regions.",
                    "system_prompt": "Use the get_weather tool to fulfill weather requests.",
                    "tools": [get_weather],
                    "model": "openai:gpt-4.1-mini",  # cheaper model for specialist
                    "middleware": [],
                },
                {
                    "name": "news_researcher",
                    "description": "Researches current news and events.",
                    "system_prompt": "Use search_news to research topics. Return concise summaries.",
                    "tools": [search_news],
                    "model": "anthropic:claude-sonnet-4-6",
                },
            ],
        ),
    ],
)
```

**Why meta-builder needs this:**

meta-builder has custom multi-agent orchestration logic in the Conductor. It builds multi-agent pipelines from `WorkflowDesign` specs, where each node is a separate agent. However, there is no standardized way for one generated agent to dynamically spawn another in response to a runtime need. The `SubagentMiddleware` pattern provides:

1. **Context isolation** — subagents don't pollute the main agent's context window
2. **Specialization** — each subagent has its own tools, model, and system prompt
3. **Clean handoffs** — main agent receives a summary, not the full execution trace
4. **Nested middleware** — subagents can have their own middleware stacks

This directly maps to what meta-builder tries to achieve with its multi-agent graph compilation, but in a composable and standardized form.

**Where to integrate:**

- `meta_builder/orchestration/conductor.py` — for supervisory patterns where one agent delegates to specialists
- `meta_builder/compilation/graph_compiler.py` — generated multi-agent architectures could use `SubagentMiddleware` instead of custom LangGraph node wiring

**Impact:** High architectural benefit, medium-high effort. Fundamental enough to warrant a dedicated refactoring effort.

---

### 1.10 TodoListMiddleware

**What it does:** Automatically equips agents with a `write_todos` tool and corresponding system prompt guidance for structured task planning. The todo list lives in agent state and tracks multi-step task progress.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[read_file, write_file, run_tests, search_code],
    middleware=[
        TodoListMiddleware(
            system_prompt=(
                "Before starting complex tasks, create a todo list with specific, "
                "actionable steps. Update the list as you complete each step. "
                "Mark items as done when complete."
            ),
            tool_description="Update the task todo list with current progress and next steps.",
        ),
    ],
)
```

**Why meta-builder needs this:**

meta-builder's `VisionAgent` uses `RequirementsSlots` for structured requirement gathering. The Conductor uses its replan cycle as an implicit progress tracker. Neither mechanism is an explicit, agent-visible task list. For long, complex pipelines, agents lose track of their high-level goals as message history grows.

`TodoListMiddleware` gives the agent an explicit, stateful todo list in its own state. The agent updates it at runtime. This improves coherence over long pipelines and provides a machine-readable progress signal.

**Where to integrate:**

- `meta_builder/agents/vision_agent.py` — replace or augment `RequirementsSlots` with `TodoListMiddleware`
- Any generated agent that handles multi-step user requests

**Impact:** Medium benefit, very low effort (single middleware addition).

---

### 1.11 FilesystemMiddleware and ShellToolMiddleware

**FilesystemMiddleware** from Deep Agents provides agents with four filesystem tools (`ls`, `read_file`, `write_file`, `edit_file`) backed by either transient agent state or persistent LangGraph Store. This solves the context-offloading problem: instead of keeping large file contents in the message history, agents write to the filesystem and reference paths.

```python
from langchain.agents import create_agent
from deepagents.middleware.filesystem import FilesystemMiddleware
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

agent = create_agent(
    model="claude-sonnet-4-6",
    store=store,
    middleware=[
        FilesystemMiddleware(
            backend=CompositeBackend(
                default=StateBackend(),
                routes={"/memories/": StoreBackend()}  # /memories/ persists across threads
            ),
            system_prompt="Write large outputs to the filesystem rather than returning them inline.",
            custom_tool_descriptions={
                "write_file": "Write content that would otherwise bloat the message context.",
            },
        ),
    ],
)
```

**ShellToolMiddleware** provides a persistent shell session, enabling sequential shell commands within a single agent run:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ShellToolMiddleware, HostExecutionPolicy, DockerExecutionPolicy

# Trusted environment (already containerized):
agent = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        ShellToolMiddleware(
            workspace_root="/workspace",
            startup_commands=["source .env", "pip install -r requirements.txt"],
            execution_policy=HostExecutionPolicy(),
        ),
    ],
)

# Isolated environment (each run gets its own Docker container):
agent_isolated = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        ShellToolMiddleware(
            workspace_root="/workspace",
            execution_policy=DockerExecutionPolicy(
                image="python:3.11-slim",
                command_timeout=120.0,
            ),
        ),
    ],
)
```

**Why meta-builder needs these:**

meta-builder's `RemediationAgent` runs code and shell commands but does so through discrete, stateless `file_io` and `execute_command` tools. Each tool call creates a new context. There is no persistent shell session — environment variables set in one call are not visible in the next.

`ShellToolMiddleware` provides a persistent session with startup commands, environment injection, and optional Docker isolation. Combined with `FilesystemMiddleware`, the RemediationAgent could write intermediate artifacts to the virtual filesystem rather than dumping them into the message context.

**Where to integrate:**

- `meta_builder/agents/remediation_agent.py` — replace manual `file_io` tools with `FilesystemMiddleware` + `ShellToolMiddleware`
- `meta_builder/tools/tool_registry.py` — remove duplicate shell execution tools

**Impact:** High quality-of-life improvement for RemediationAgent, medium effort.

---

## 2. Model Context Protocol (MCP) Integration

### What It Is

The Model Context Protocol (MCP) is an open standard for connecting AI agents to external tool servers. `langchain-mcp-adapters` bridges LangChain agents and MCP servers: it converts MCP tool definitions into LangChain-native tools, enabling any `create_agent()` to use tools from any MCP server without writing custom integrations.

`MultiServerMCPClient` connects to multiple MCP servers simultaneously, across different transport types: `stdio` (local subprocess), `http`/`sse` (remote HTTP), and custom transports.

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

async def main():
    client = MultiServerMCPClient(
        {
            "filesystem": {
                "transport": "stdio",
                "command": "python",
                "args": ["/usr/local/bin/mcp-filesystem-server.py"],
            },
            "github": {
                "transport": "http",
                "url": "https://mcp.github.com/mcp",
                "headers": {"Authorization": f"Bearer {GITHUB_TOKEN}"},
            },
            "database": {
                "transport": "stdio",
                "command": "node",
                "args": ["/usr/local/lib/mcp-postgres-server/index.js"],
            },
            "web_search": {
                "transport": "http",
                "url": "http://localhost:8001/mcp",
            },
        }
    )

    # All MCP tools become LangChain tools transparently:
    tools = await client.get_tools()
    print(f"Loaded {len(tools)} tools from {len(client.adapters)} MCP servers")

    agent = create_agent("anthropic:claude-sonnet-4-6", tools)
    result = await agent.ainvoke({
        "messages": [{"role": "user", "content": "Search GitHub for open issues labeled 'bug' and summarize them."}]
    })

asyncio.run(main())
```

### Custom MCP Server (Exposing meta-builder as an MCP server)

```python
from fastmcp import FastMCP
from meta_builder.orchestration.conductor import Conductor

mcp = FastMCP("meta-builder")

@mcp.tool()
async def design_agent(requirements: str) -> str:
    """Design an AI agent from natural language requirements.
    
    Args:
        requirements: Natural language description of the agent's purpose and capabilities.
    
    Returns:
        JSON representation of the designed agent workflow.
    """
    conductor = Conductor()
    design = await conductor.design_workflow(requirements)
    return design.model_dump_json()

@mcp.tool()
async def compile_agent(workflow_design_json: str) -> str:
    """Compile a workflow design into an executable agent graph."""
    ...

if __name__ == "__main__":
    mcp.run(transport="http", port=8080)
```

### Stateful Sessions and Interceptors

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.tools import load_mcp_tools
from langchain_mcp_adapters.interceptors import MCPToolCallRequest

# Inject user context into every MCP tool call:
async def inject_tenant_context(request: MCPToolCallRequest, handler):
    """Inject tenant ID from LangGraph runtime into MCP tool calls."""
    runtime = request.runtime
    tenant_id = runtime.context.tenant_id if runtime.context else "default"
    modified = request.override(args={**request.args, "tenant_id": tenant_id})
    return await handler(modified)

client = MultiServerMCPClient(
    {"database": {"transport": "http", "url": "http://localhost:9090/mcp"}},
    tool_interceptors=[inject_tenant_context],
)

# Stateful session (for servers that maintain state between calls):
async with client.session("database") as session:
    tools = await load_mcp_tools(session)
    agent = create_agent("anthropic:claude-sonnet-4-6", tools)
```

### Why meta-builder Needs MCP

meta-builder's `ToolRegistry` is entirely local. Every tool that a generated agent can use must be pre-defined in Python and explicitly registered. This means:

1. **Fixed tool universe** — the set of tools a generated agent can use is bounded by what developers have hard-coded.
2. **No external discovery** — there is no way for generated agents to discover and use tools from external services (GitHub, Notion, Slack, databases) without writing a new integration.
3. **No protocol standardization** — each external integration is custom, with no shared authentication, error handling, or schema format.

MCP solves all three problems. With `langchain-mcp-adapters`, meta-builder's `ToolRegistry` could become an MCP proxy — it would:
- Connect to a configurable list of MCP servers at startup
- Register all discovered tools in the ToolRegistry automatically
- Allow generated agents to be configured with tools from any MCP server

Additionally, meta-builder could expose its own capabilities as an MCP server (see custom server example above), making the entire meta-builder design-build-test loop accessible as a tool to any MCP-compatible agent.

**Where to integrate:**

- `meta_builder/tools/tool_registry.py` — add `MCPToolProvider` that wraps `MultiServerMCPClient`
- `meta_builder/config/agent_config.py` — add `mcp_servers: list[MCPServerConfig]` to the configuration schema
- `meta_builder/api/mcp_server.py` (new file) — expose meta-builder capabilities as a FastMCP server

**Integration plan:**

```python
# meta_builder/tools/mcp_tool_provider.py
from langchain_mcp_adapters.client import MultiServerMCPClient
from meta_builder.config.agent_config import MCPServerConfig

class MCPToolProvider:
    def __init__(self, server_configs: list[MCPServerConfig]):
        self._client = MultiServerMCPClient({
            cfg.name: {
                "transport": cfg.transport,
                **cfg.connection_params,
            }
            for cfg in server_configs
        })
    
    async def get_tools(self):
        return await self._client.get_tools()
    
    async def get_tools_for_server(self, server_name: str):
        async with self._client.session(server_name) as session:
            from langchain_mcp_adapters.tools import load_mcp_tools
            return await load_mcp_tools(session)
```

**Impact:** Very high strategic value — fundamentally expands the tool ecosystem for generated agents. Medium implementation effort.

---

## 3. Context Engineering Framework

### What It Is

LangChain has formalized a three-layer context engineering framework that distinguishes between types of context flowing through an agent:

| Context Type | What You Control | Transient or Persistent |
|---|---|---|
| **Model Context** | Instructions, message history, tools, response format (what goes into model calls) | Transient — per-call |
| **Tool Context** | What tools can read/write from state, store, and runtime context | Persistent — survives between calls |
| **Life-cycle Context** | What happens between model and tool calls: summarization, guardrails, logging | Persistent — modifies state |

This is not just terminology — it is a framework for reasoning about **where** context lives and **how** it flows.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from langchain.tools import ToolRuntime, tool
from dataclasses import dataclass
from typing import Callable

# --- MODEL CONTEXT: dynamically inject system instructions based on state ---
@wrap_model_call
def inject_pipeline_context(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    """Modify what goes into model calls based on conversation state."""
    state = request.state
    pipeline_stage = state.get("pipeline_stage", "design")
    message_count = len(state["messages"])
    
    # Inject stage-specific instructions into the model call:
    stage_instructions = {
        "design": "Focus on extracting requirements and designing the workflow graph.",
        "compile": "Focus on generating correct LangGraph code from the design.",
        "test": "Focus on identifying and diagnosing test failures.",
        "remediate": "Focus on applying minimal fixes to pass failing tests.",
    }
    
    # Dynamically select model based on conversation complexity:
    if message_count > 30:
        modified = request.override(
            system=f"{request.system}\n\n[Stage: {pipeline_stage}] {stage_instructions[pipeline_stage]}",
            model=large_context_model,  # switch to larger model for long conversations
        )
    else:
        modified = request.override(
            system=f"{request.system}\n\n[Stage: {pipeline_stage}] {stage_instructions[pipeline_stage]}",
        )
    
    return handler(modified)

# --- TOOL CONTEXT: tools that read/write to persistent store ---
@dataclass
class PipelineContext:
    user_id: str
    org_id: str

@tool
def get_org_preferences(runtime: ToolRuntime[PipelineContext]) -> str:
    """Read organization-level agent design preferences from long-term memory."""
    assert runtime.store is not None
    org_id = runtime.context.org_id
    prefs = runtime.store.get(("orgs", org_id), "preferences")
    return str(prefs.value) if prefs else "{}"

@tool
def save_successful_pattern(pattern_name: str, pattern_json: str, runtime: ToolRuntime[PipelineContext]) -> str:
    """Save a successful agent design pattern to the organization's memory store."""
    assert runtime.store is not None
    org_id = runtime.context.org_id
    runtime.store.put(("orgs", org_id, "patterns"), pattern_name, {"pattern": pattern_json})
    return f"Saved pattern '{pattern_name}'"

# --- LIFE-CYCLE CONTEXT: middleware that runs between model/tool calls ---
from langchain.agents.middleware import SummarizationMiddleware, PIIMiddleware

agent = create_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[get_org_preferences, save_successful_pattern],
    store=production_store,
    context_schema=PipelineContext,
    middleware=[
        inject_pipeline_context,             # Model Context control
        SummarizationMiddleware(...),         # Life-cycle Context
        PIIMiddleware("email", strategy="redact"),  # Life-cycle Context
    ],
)
```

### Why meta-builder Needs This Framework

meta-builder's context management is entirely ad-hoc:
- System prompts are static strings hardcoded in Python classes
- Tool context is limited to whatever is passed as function arguments
- There is no concept of lifecycle context — things like retry logic, summarization, and PII filtering are bolted on in different places with no unifying structure

Adopting the three-layer framework would:

1. Give every developer on the team a shared vocabulary for discussing "where does this context live?"
2. Make it clear whether a change is transient (per model-call) or persistent (stored in state or store)
3. Unlock the middleware system naturally, since middleware *is* the lifecycle context layer

**Where to integrate:**

This is a structural/conceptual change rather than a code change. The immediate practical implications are:

- `meta_builder/orchestration/conductor.py` — audit every context manipulation and classify it as Model/Tool/Lifecycle
- `meta_builder/agents/` — refactor static system prompts to be injected dynamically via `@wrap_model_call` middleware
- `meta_builder/tools/` — use `ToolRuntime` to access store and context from tools rather than global state

**Impact:** Medium-term architectural clarity benefit. The framework itself is a mindset change; the code follows naturally.

---

## 4. Long-Term Memory via LangGraph Store

### What It Is

LangGraph Store is a namespace-based, key-value document store with optional vector similarity search. It is the standard persistence layer for cross-conversation memory in LangChain agents. Two implementations: `InMemoryStore` (development) and `PostgresStore` (production, with pgvector for semantic search).

```python
from langchain.agents import create_agent
from langchain.tools import ToolRuntime, tool
from langgraph.store.memory import InMemoryStore
from langgraph.store.base import IndexConfig
from langgraph.store.postgres import PostgresStore
from dataclasses import dataclass
from typing_extensions import TypedDict
import os

# --- Development: InMemoryStore with vector search ---
def embed_texts(texts):
    # Replace with your embedding model
    from langchain_openai import OpenAIEmbeddings
    embedder = OpenAIEmbeddings()
    return embedder.embed_documents(texts)

store = InMemoryStore(index=IndexConfig(embed=embed_texts, dims=1536))

# --- Production: PostgreSQL with pgvector ---
store = PostgresStore.from_conn_string(
    os.environ["DATABASE_URL"],
    index=IndexConfig(embed="openai:text-embedding-3-small", dims=1536),
)
store.setup()  # runs migrations

# --- Store operations ---
user_id = "user-123"
org_id = "org-456"

# Hierarchical namespaces:
store.put(("users", user_id, "preferences"), "agent_style", {
    "preferred_framework": "react",
    "verbosity": "concise",
    "favorite_models": ["claude-sonnet-4-6", "gpt-4.1"],
})

store.put(("orgs", org_id, "patterns"), "customer_support_bot", {
    "description": "Customer support agent with escalation",
    "tool_count": 5,
    "avg_success_rate": 0.94,
    "design": { ... }
})

# Semantic search across namespace:
similar_patterns = store.search(
    ("orgs", org_id, "patterns"),
    query="e-commerce order tracking agent",  # vector similarity
    filter={"avg_success_rate": {"$gte": 0.9}},  # content filter
    limit=5,
)

# --- Store-backed tools for agents ---
class UserContext(TypedDict):
    user_id: str

@tool
def recall_past_designs(query: str, runtime: ToolRuntime) -> str:
    """Recall past agent designs similar to the current request."""
    user_id = runtime.context.user_id
    results = runtime.store.search(
        ("users", user_id, "designs"),
        query=query,
        limit=3,
    )
    return "\n".join([f"- {r.key}: {r.value['description']}" for r in results])

@tool
def save_design(name: str, description: str, design_json: str, runtime: ToolRuntime) -> str:
    """Save a successful agent design for future reference."""
    user_id = runtime.context.user_id
    runtime.store.put(
        ("users", user_id, "designs"),
        name,
        {"description": description, "design": design_json, "saved_at": "now"},
    )
    return f"Design '{name}' saved successfully."

agent = create_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[recall_past_designs, save_design],
    store=store,
    context_schema=UserContext,
)
```

### Why meta-builder Needs This

meta-builder has `PipelineMemory` for within-session memory, but it is:
- Custom (not LangGraph Store)
- Thread-scoped (doesn't persist across sessions)
- Not semantically searchable

The implications are significant:

1. **No institutional memory** — each time a user asks meta-builder to design an agent, it starts from zero. Successful patterns from previous sessions are lost.
2. **No personalization** — meta-builder cannot remember user preferences (preferred models, verbosity style, domain-specific tool preferences) across sessions.
3. **No pattern library** — there is no way to build up a corpus of "designs that worked well for X type of request" that can inform future designs.

With LangGraph Store, meta-builder could:
- Remember user preferences across sessions
- Store successful `WorkflowDesign` patterns for retrieval
- Maintain an organizational knowledge base of agent patterns indexed by use-case description
- Give generated agents themselves cross-session memory (not just within a pipeline run)

**Where to integrate:**

- `meta_builder/memory/pipeline_memory.py` — replace custom PipelineMemory with LangGraph Store adapter
- `meta_builder/agents/vision_agent.py` — add `recall_past_designs` and `save_design` tools
- `meta_builder/orchestration/conductor.py` — pass `store=store` to `create_agent()` calls
- `meta_builder/config/agent_config.py` — add `store_backend: Literal["memory", "postgres"]` and connection config

**Impact:** High long-term value, medium implementation effort. Requires PostgreSQL infrastructure for production.

---

## 5. Agent Evals Framework (agentevals)

### What It Is

`agentevals` is LangChain's evaluation framework specifically designed for agent trajectories — the full sequence of messages, tool calls, and model responses in a conversation. It provides two main evaluator types:

**Trajectory Match Evaluators:** Deterministically compare actual vs. reference trajectories with configurable matching modes.

```python
from agentevals.trajectory.match import create_trajectory_match_evaluator
from langchain.agents import create_agent
from langchain.tools import tool
from langchain.messages import HumanMessage

@tool
def get_weather(city: str) -> str:
    """Get weather for a city."""
    return f"It's sunny in {city}."

@tool
def get_forecast(city: str, days: int) -> str:
    """Get weather forecast."""
    return f"{days}-day forecast for {city}: sunny."

agent = create_agent("gpt-4.1", tools=[get_weather, get_forecast])

# Define the reference (expected) trajectory:
reference_trajectory = [
    HumanMessage(content="What's the weather in Seattle?"),
    # AI calls get_weather("Seattle")
    # Tool returns result
    # AI responds with summary
]

# Evaluate with different matching modes:

# STRICT: exact message sequence must match:
strict_evaluator = create_trajectory_match_evaluator(trajectory_match_mode="strict")

# UNORDERED: same tool calls, any order:
unordered_evaluator = create_trajectory_match_evaluator(trajectory_match_mode="unordered")

# SUBSET: reference must be a subset of actual (allows extra tool calls):
subset_evaluator = create_trajectory_match_evaluator(trajectory_match_mode="subset")

# SUPERSET: actual must be a subset of reference (agent must not use extra tools):
superset_evaluator = create_trajectory_match_evaluator(trajectory_match_mode="superset")

result = agent.invoke({"messages": [HumanMessage("What's the weather in Seattle?")]})

evaluation = strict_evaluator(
    outputs=result["messages"],
    reference_outputs=reference_trajectory,
)
# Returns: {"key": "trajectory_match", "score": True/False, "comment": "..."}
```

**LLM-as-Judge Evaluators:** Use an LLM to qualitatively assess trajectory quality without requiring a reference.

```python
from agentevals.trajectory.llm import (
    create_trajectory_llm_as_judge,
    create_async_trajectory_llm_as_judge,
    TRAJECTORY_ACCURACY_PROMPT,
    TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE,
)

# Without reference — judge evaluates on general quality:
judge_evaluator = create_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT,
)

result = agent.invoke({"messages": [HumanMessage("What's the weather in NYC and Seattle?")]})

evaluation = judge_evaluator(
    outputs=result["messages"],
)
# Returns: {"key": "trajectory_accuracy", "score": True, "comment": "Agent correctly..."}

# With reference — judge compares against expected trajectory:
judge_with_ref = create_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE,
)

evaluation = judge_with_ref(
    outputs=result["messages"],
    reference_outputs=reference_trajectory,
)

# Async versions for parallel evaluation:
async_judge = create_async_trajectory_llm_as_judge(
    model="openai:o3-mini",
    prompt=TRAJECTORY_ACCURACY_PROMPT,
)
```

**Building a regression test suite:**

```python
import pytest
from agentevals.trajectory.match import create_trajectory_match_evaluator
from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT

# Test fixtures: pairs of (input, expected_trajectory)
EVAL_DATASET = [
    {
        "input": "Design a customer support bot with email escalation",
        "expected_tools_called": ["search_knowledge_base", "design_workflow"],
        "min_quality_score": 0.8,
    },
    {
        "input": "Build a data pipeline agent that reads from S3 and writes to Postgres",
        "expected_tools_called": ["design_workflow", "select_tools"],
        "min_quality_score": 0.85,
    },
]

@pytest.mark.parametrize("case", EVAL_DATASET)
async def test_meta_builder_trajectory(case, meta_builder_agent):
    result = await meta_builder_agent.ainvoke({
        "messages": [HumanMessage(case["input"])]
    })
    
    # Check that expected tools were called:
    tool_names_called = [
        msg.name for msg in result["messages"]
        if hasattr(msg, "name")  # ToolMessages
    ]
    for expected_tool in case["expected_tools_called"]:
        assert expected_tool in tool_names_called, f"Expected {expected_tool} to be called"
    
    # LLM quality check:
    judge = create_trajectory_llm_as_judge(
        model="openai:o3-mini",
        prompt=TRAJECTORY_ACCURACY_PROMPT,
    )
    eval_result = judge(outputs=result["messages"])
    assert eval_result["score"] is True, f"Quality check failed: {eval_result['comment']}"
```

### Why meta-builder Needs This

meta-builder has an `EvaluatorAgent` that tests generated agent code, but it evaluates *the generated agent's behavior* — not *meta-builder's own reasoning trajectory*. There is no regression testing of the design pipeline itself:

- Does meta-builder correctly call `search_knowledge_base` before `design_workflow`?
- Does meta-builder's VisionAgent extract all required requirements before proceeding?
- When meta-builder remediates, does it call `run_tests` after `apply_patch`?

These are trajectory-level questions. The `agentevals` framework makes them testable and machine-verifiable. Combined with LangSmith's dataset management, this enables regression testing of meta-builder's own behavior across versions.

**Where to integrate:**

- `meta_builder/tests/eval/` (new directory) — trajectory-based regression tests for each agent
- `meta_builder/agents/evaluator_agent.py` — augment with `agentevals` trajectory evaluation for generated agent testing
- CI/CD pipeline — run trajectory evaluations on each pull request

**Impact:** High quality assurance value. Medium implementation effort. Particularly important for maintaining reliability as meta-builder evolves.

---

## 6. `create_agent()` Factory

### What It Is

`create_agent()` is LangChain's high-level, batteries-included factory for building production agents. It handles:
- Model initialization (via `init_chat_model()` with `"provider:model"` syntax)
- Tool binding
- Middleware stack composition
- Checkpointer attachment
- Store attachment
- Context schema wiring
- LangGraph graph compilation

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore

agent = create_agent(
    model="anthropic:claude-sonnet-4-6",   # provider:model syntax
    tools=[tool1, tool2, tool3],
    system_prompt="You are a helpful AI agent.",
    checkpointer=InMemorySaver(),           # enables multi-turn memory
    store=InMemoryStore(),                  # enables long-term memory
    middleware=[...],                       # full middleware stack
    name="my-agent",                       # for LangSmith/Studio
)

# Invoke synchronously:
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Hello"}]},
    config={"configurable": {"thread_id": "thread-123"}},
)

# Invoke asynchronously:
result = await agent.ainvoke(...)

# Stream:
async for chunk in agent.astream({"messages": [...]}):
    print(chunk)
```

### Why meta-builder Needs This

meta-builder builds agents manually: it constructs LangGraph `StateGraph` objects, adds nodes and edges, compiles them, and wires tools and models by hand. This is verbose and error-prone. Every new agent type (Conductor, VisionAgent, EvaluatorAgent, RemediationAgent) reimplements the same boilerplate.

`create_agent()` is not a loss of control — it compiles to LangGraph under the hood and is fully inspectable. The key benefit is that it handles the wiring correctly: middleware is applied in the right order, the checkpointer is attached properly, and the `store` is accessible from tools via `ToolRuntime`.

**Where to integrate:**

- `meta_builder/agents/base_agent.py` (new) — replace per-agent manual graph compilation with `create_agent()`
- `meta_builder/compilation/graph_compiler.py` — use `create_agent()` as the compiled artifact format for generated agents
- `meta_builder/orchestration/conductor.py` — replace the Conductor's manual graph with `create_agent()`

**Impact:** Very high developer-experience benefit. Medium refactoring effort. Once done, unlocks all middleware features automatically.

---

## 7. Voice Agent Architecture

### What It Is

LangChain documents the "sandwich architecture" for voice agents: Speech-to-Text (STT) → LangChain Agent → Text-to-Speech (TTS). This pattern uses WebSockets for bidirectional streaming with sub-700ms latency using providers like AssemblyAI (STT) and Cartesia (TTS).

```python
# Server-side orchestration (FastAPI + WebSockets):
import asyncio
from fastapi import FastAPI, WebSocket
from langchain.agents import create_agent
from langchain.tools import tool
from langgraph.checkpoint.memory import InMemorySaver

# Mock STT/TTS adapters (replace with AssemblyAI, Deepgram, Cartesia, etc.)
class AssemblyAISTT:
    async def transcribe_stream(self, audio_chunks) -> str:
        # Send audio to STT provider, receive transcript
        ...

class CartesiaTTS:
    async def synthesize_stream(self, text_tokens):
        # Send text to TTS provider, receive audio chunks
        ...

app = FastAPI()
checkpointer = InMemorySaver()

@tool
def handle_order(item: str, quantity: int) -> str:
    """Process a customer order."""
    return f"Order confirmed: {quantity}x {item}"

agent = create_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[handle_order],
    system_prompt="You are a voice agent for a sandwich shop. Be concise — your responses will be spoken aloud.",
    checkpointer=checkpointer,
)

@app.websocket("/voice")
async def voice_endpoint(ws: WebSocket):
    await ws.accept()
    thread_id = ws.headers.get("X-Thread-ID", "default")
    stt = AssemblyAISTT()
    tts = CartesiaTTS()
    
    async for audio_chunk in ws.iter_bytes():
        # Step 1: STT
        transcript = await stt.transcribe_stream([audio_chunk])
        if not transcript:
            continue
        
        # Step 2: Agent (streaming)
        agent_response = ""
        async for event in agent.astream(
            {"messages": [{"role": "user", "content": transcript}]},
            config={"configurable": {"thread_id": thread_id}},
            stream_mode="messages",
        ):
            if hasattr(event, "content") and event.content:
                agent_response += event.content
        
        # Step 3: TTS
        async for audio_chunk in tts.synthesize_stream(agent_response):
            await ws.send_bytes(audio_chunk)
```

### Why meta-builder Needs This

meta-builder generates agents — the generated agents have zero voice capability. Adding voice to generated agents requires:

1. A standardized voice agent template in the `WorkflowDesign` schema
2. STT/TTS tool wrappers in the `ToolRegistry`
3. A voice-aware system prompt template
4. WebSocket endpoint generation in the agent scaffolding

None of these exist today. The market for voice-capable AI agents is growing rapidly (customer service, accessibility, mobile interfaces). meta-builder should be able to generate voice agents with the same ease as text agents.

**Where to integrate:**

- `meta_builder/templates/voice_agent_template.py` (new) — a `WorkflowDesign` template for voice agents
- `meta_builder/tools/built_in/voice_tools.py` (new) — STT and TTS tool wrappers
- `meta_builder/agents/vision_agent.py` — detect voice requirements from natural language ("I want a voice interface", "spoken responses")
- `meta_builder/compilation/scaffold_generator.py` — generate WebSocket endpoints for voice agents

**Impact:** High strategic value (new capability category), high implementation effort. Recommend as a dedicated feature track.

---

## 8. Agent Chat UI

### What It Is

Agent Chat UI is an open-source Next.js application that provides a turnkey conversational interface for any LangChain agent. It connects directly to LangGraph-compiled agents via the LangGraph API and provides:

- Real-time streaming chat
- Tool call visualization (shows what tools are being called and their outputs)
- **Time-travel debugging** — rewind conversation history to any checkpoint, edit state, replay
- **State forking** — branch the conversation from any point to explore alternatives
- Human-in-the-loop approval UI (works with `HumanInTheLoopMiddleware`)

It is designed to work with agents compiled via `create_agent()` with zero additional configuration.

```bash
# Spin up Agent Chat UI against any LangGraph agent:
npx create-agent-chat-ui@latest

# Configure it to point to your meta-builder-generated agent:
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_AGENT_URL=http://localhost:8000/my-agent
```

### Why meta-builder Needs This

meta-builder has some integration with LangGraph Studio, but the generated agents themselves have no standardized UI. When a user builds an agent with meta-builder:

1. They get executable Python code
2. They can test it via the meta-builder CLI
3. They can view traces in Langfuse

But there is no visual, interactive way to test and debug the generated agent without writing a frontend. Agent Chat UI fills this gap entirely — it provides time-travel debugging (inspect what the agent was "thinking" at each step), tool visualization (see what tools were called and with what arguments), and state forking (explore "what if I had said X instead?").

**Where to integrate:**

- `meta_builder/compilation/scaffold_generator.py` — add `docker-compose.yml` that includes Agent Chat UI alongside the generated agent
- `meta_builder/api/agent_runner.py` — expose the LangGraph API endpoints that Agent Chat UI expects
- Generated agent documentation — include setup instructions for Agent Chat UI

**Impact:** High developer experience value for generated agents, low implementation effort (Agent Chat UI is a drop-in app).

---

## 9. Generative UI Patterns

### What It Is

LangChain's frontend documentation covers patterns for building rich, agent-aware UIs:
- **Generative UI** — agents stream structured UI component specs alongside text
- **Branching chat** — UI exposes the state-forking capability for users to explore alternatives
- **Join/rejoin** — users can rejoin an ongoing agent session from any client
- **Message queues** — decouple agent execution from UI delivery
- **Reasoning token visualization** — surface chain-of-thought tokens in a collapsed/expandable UI

```python
# Streaming structured UI components from the agent:
from langchain.agents import create_agent
from pydantic import BaseModel
from typing import Literal

class ChartComponent(BaseModel):
    component_type: Literal["chart"] = "chart"
    chart_type: str
    data: list[dict]
    title: str

class TableComponent(BaseModel):
    component_type: Literal["table"] = "table"
    headers: list[str]
    rows: list[list[str]]

# Agent can return structured UI specs in addition to text:
@tool
def generate_sales_report(period: str) -> dict:
    """Generate a sales report for the given period."""
    data = fetch_sales_data(period)
    return {
        "text_summary": f"Sales in {period}: $1.2M total",
        "ui_component": ChartComponent(
            chart_type="bar",
            data=data,
            title=f"Sales by Product — {period}",
        ).model_dump(),
    }
```

### Why meta-builder Needs This

meta-builder currently generates **backend-only agents**. The output is Python code for an agent that processes text in and text out. There is no mechanism for generated agents to:

- Return structured data that a frontend can render as charts/tables
- Stream partial results with UI hints
- Support branching/forking at the user interface level
- Expose reasoning tokens to the frontend for transparency

As AI agents become more consumer-facing, the expectation is that they produce rich, interactive outputs — not just text strings. meta-builder should be able to generate agents with UI awareness built in.

**Where to integrate:**

- `meta_builder/compilation/scaffold_generator.py` — add optional Next.js frontend generation
- `meta_builder/tools/built_in/ui_tools.py` (new) — standardized tools for emitting UI components
- `meta_builder/agents/vision_agent.py` — detect UI requirements from natural language descriptions

**Impact:** High product differentiation, high implementation effort. Recommend as a future roadmap item.

---

## 10. Rate Limiters (langchain-core)

### What It Is

`langchain-core` ships a built-in `InMemoryRateLimiter` based on the token bucket algorithm. It is attached directly to a chat model instance and transparently throttles API requests to stay within rate limits.

```python
from langchain_core.rate_limiters import InMemoryRateLimiter
from langchain_anthropic import ChatAnthropic
from langchain_openai import ChatOpenAI
import time

# Limit Claude to 1 request per second:
claude_limiter = InMemoryRateLimiter(
    requests_per_second=1.0,
    check_every_n_seconds=0.05,   # check every 50ms
    max_bucket_size=5,             # allow burst of 5 requests
)

claude = ChatAnthropic(
    model="claude-sonnet-4-6",
    rate_limiter=claude_limiter,
)

# Limit GPT-4.1 more aggressively (e.g., free tier):
gpt_limiter = InMemoryRateLimiter(
    requests_per_second=0.5,       # 1 request per 2 seconds
    check_every_n_seconds=0.1,
    max_bucket_size=2,
)

gpt = ChatOpenAI(model="gpt-4.1", rate_limiter=gpt_limiter)

# In LLM Factory:
from meta_builder.llm.llm_factory import LLMFactory

class RateLimitedLLMFactory(LLMFactory):
    def __init__(self, config: LLMConfig):
        super().__init__(config)
        self._limiters = {
            "anthropic": InMemoryRateLimiter(
                requests_per_second=config.anthropic_rps,
                max_bucket_size=config.anthropic_burst,
            ),
            "openai": InMemoryRateLimiter(
                requests_per_second=config.openai_rps,
                max_bucket_size=config.openai_burst,
            ),
        }
    
    def create_model(self, provider: str, model_name: str):
        base_model = super().create_model(provider, model_name)
        limiter = self._limiters.get(provider)
        if limiter:
            return base_model.with_rate_limiter(limiter)
        return base_model

# The rate limiter works for both sync and async contexts:
async def demo():
    for i in range(10):
        tic = time.time()
        response = await claude.ainvoke("Hello")
        elapsed = time.time() - tic
        print(f"Request {i}: {elapsed:.2f}s")
        # With requests_per_second=1.0, each request takes ~1s even if the API is faster
```

### Why meta-builder Needs This

meta-builder's `LLMFactory` creates model instances but does not apply rate limiting. In scenarios where meta-builder runs parallel agents (multiple nodes executing concurrently in the Conductor), it can burst many API requests simultaneously, triggering rate limit errors from providers.

The current mitigation is `ModelRetryMiddleware`, but retrying after hitting rate limits adds latency. The correct solution is to never hit the rate limit in the first place — which is what `InMemoryRateLimiter` does.

Note that `InMemoryRateLimiter` is process-local. For production deployments with multiple meta-builder instances, a Redis-backed rate limiter would be needed. But for single-instance deployments, the in-memory limiter is sufficient.

**Where to integrate:**

- `meta_builder/llm/llm_factory.py` — attach `InMemoryRateLimiter` to every model instance, configurable per provider
- `meta_builder/config/agent_config.py` — add `rate_limits: dict[str, RateLimitConfig]` to the config schema

**Implementation:**

```python
# meta_builder/config/rate_limit_config.py
from pydantic import BaseModel

class RateLimitConfig(BaseModel):
    requests_per_second: float = 1.0
    max_bucket_size: float = 5.0
    check_every_n_seconds: float = 0.1

# meta_builder/llm/llm_factory.py
from langchain_core.rate_limiters import InMemoryRateLimiter

class LLMFactory:
    def __init__(self, config: LLMConfig):
        self._config = config
        self._rate_limiters = {
            provider: InMemoryRateLimiter(
                requests_per_second=rate_config.requests_per_second,
                max_bucket_size=rate_config.max_bucket_size,
                check_every_n_seconds=rate_config.check_every_n_seconds,
            )
            for provider, rate_config in (config.rate_limits or {}).items()
        }
    
    def get_model(self, provider: str, model_name: str, **kwargs):
        model = self._create_raw_model(provider, model_name, **kwargs)
        if provider in self._rate_limiters:
            model.rate_limiter = self._rate_limiters[provider]
        return model
```

**Impact:** Medium benefit, very low effort. Single-line addition per model instance.

---

## Summary: Priority Matrix

| Feature | Effort | Benefit | Priority |
|---|---|---|---|
| SummarizationMiddleware | Very Low | High | **P0 — do now** |
| ModelRetry / ToolRetry Middleware | Low | High | **P0 — do now** |
| Rate Limiters | Very Low | Medium | **P0 — do now** |
| ModelCallLimit / ToolCallLimit | Low | High | **P1 — next sprint** |
| PIIMiddleware | Low | High | **P1 — next sprint** |
| LLMToolSelectorMiddleware | Low | Medium | **P1 — next sprint** |
| ContextEditingMiddleware | Low | Medium | **P1 — next sprint** |
| `create_agent()` Factory | Medium | Very High | **P1 — next sprint** |
| Long-Term Memory (LangGraph Store) | Medium | High | **P2 — next quarter** |
| MCP Integration | Medium | Very High | **P2 — next quarter** |
| agentevals Framework | Medium | High | **P2 — next quarter** |
| Context Engineering Framework | Low | Medium (structural) | **P2 — next quarter** |
| HumanInTheLoopMiddleware | Medium | High | **P2 — next quarter** |
| SubagentMiddleware | Medium | High | **P2 — next quarter** |
| FilesystemMiddleware | Medium | Medium | **P2 — next quarter** |
| ModelFallbackMiddleware | Low | Medium | **P2 — next quarter** |
| TodoListMiddleware | Very Low | Medium | **P2 — next quarter** |
| ShellToolMiddleware | Medium | Medium | **P3 — future** |
| Agent Chat UI | Low | High | **P3 — future** |
| agentevals CI Integration | Medium | High | **P3 — future** |
| Voice Agent Architecture | High | High | **P3 — future** |
| Generative UI Patterns | Very High | Medium | **P4 — roadmap** |

---

## Sources

- [LangChain Prebuilt Middleware Documentation](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [LangChain Custom Middleware Documentation](https://docs.langchain.com/oss/python/langchain/middleware/custom)
- [LangChain MCP Adapters Documentation](https://docs.langchain.com/oss/python/langchain/mcp)
- [LangChain Context Engineering Documentation](https://docs.langchain.com/oss/python/langchain/context-engineering)
- [LangChain Long-Term Memory Documentation](https://docs.langchain.com/oss/python/langchain/long-term-memory)
- [agentevals Trajectory Evaluation Documentation](https://docs.langchain.com/langsmith/trajectory-evals)
- [SummarizationMiddleware API Reference](https://reference.langchain.com/python/langchain/agents/middleware/summarization/SummarizationMiddleware)
- [InMemoryRateLimiter API Reference](https://reference.langchain.com/python/langchain-core/rate_limiters/InMemoryRateLimiter)
- [LangChain Voice Agent Documentation](https://docs.langchain.com/oss/javascript/langchain/voice-agent)
- [Agent Chat UI Documentation](https://docs.langchain.com/oss/python/langchain/ui)
- [LangChain Blog: How Middleware Lets You Customize Your Agent Harness](https://blog.langchain.com/how-middleware-lets-you-customize-your-agent-harness/)
