> ## Documentation Index
> Fetch the complete documentation index at: https://docs.langchain.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Prebuilt middleware

> Prebuilt middleware for common agent use cases

LangChain and [Deep Agents](/oss/python/deepagents/overview) provide prebuilt middleware for common use cases. Each middleware is production-ready and configurable for your specific needs.

## Provider-agnostic middleware

The following middleware work with any LLM provider:

| Middleware                              | Description                                                                  |
| --------------------------------------- | ---------------------------------------------------------------------------- |
| [Summarization](#summarization)         | Automatically summarize conversation history when approaching token limits.  |
| [Human-in-the-loop](#human-in-the-loop) | Pause execution for human approval of tool calls.                            |
| [Model call limit](#model-call-limit)   | Limit the number of model calls to prevent excessive costs.                  |
| [Tool call limit](#tool-call-limit)     | Control tool execution by limiting call counts.                              |
| [Model fallback](#model-fallback)       | Automatically fallback to alternative models when primary fails.             |
| [PII detection](#pii-detection)         | Detect and handle Personally Identifiable Information (PII).                 |
| [To-do list](#to-do-list)               | Equip agents with task planning and tracking capabilities.                   |
| [LLM tool selector](#llm-tool-selector) | Use an LLM to select relevant tools before calling main model.               |
| [Tool retry](#tool-retry)               | Automatically retry failed tool calls with exponential backoff.              |
| [Model retry](#model-retry)             | Automatically retry failed model calls with exponential backoff.             |
| [LLM tool emulator](#llm-tool-emulator) | Emulate tool execution using an LLM for testing purposes.                    |
| [Context editing](#context-editing)     | Manage conversation context by trimming or clearing tool uses.               |
| [Shell tool](#shell-tool)               | Expose a persistent shell session to agents for command execution.           |
| [File search](#file-search)             | Provide Glob and Grep search tools over filesystem files.                    |
| [Filesystem](#filesystem-middleware)    | Provide agents with a filesystem for storing context and long-term memories. |
| [Subagent](#subagent)                   | Add the ability to spawn subagents.                                          |

### Summarization

Automatically summarize conversation history when approaching token limits, preserving recent messages while compressing older context. Summarization is useful for the following:

* Long-running conversations that exceed context windows.
* Multi-turn dialogues with extensive history.
* Applications where preserving full conversation context matters.

**API reference:** [`SummarizationMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/SummarizationMiddleware)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[your_weather_tool, your_calculator_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-4.1-mini",
            trigger=("tokens", 4000),
            keep=("messages", 20),
        ),
    ],
)
```


<details>
<summary>Configuration options</summary>

> 💡 **Tip**
>
> The `fraction` conditions for `trigger` and `keep` (shown below) rely on a chat model's [profile data](/oss/python/langchain/models#model-profiles) if using `langchain>=1.1`. If data are not available, use another condition or specify manually:
> 
>     ```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
>     from langchain.chat_models import init_chat_model
> 
>     custom_profile = {
>         "max_input_tokens": 100_000,
>         # ...
>     }
>     model = init_chat_model("gpt-4.1", profile=custom_profile)
>     ```


  
- **`param`** (`string | BaseChatModel`): Model for generating summaries. Can be a model identifier string (e.g., `'openai:gpt-4.1-mini'`) or a `BaseChatModel` instance. See [`init_chat_model`](https://reference.langchain.com/python/langchain/chat_models/base/init_chat_model) for more information.


  
- **`param`** (`ContextSize | list[ContextSize] | None`): Condition(s) for triggering summarization. Can be:

    * A single [`ContextSize`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/ContextSize) tuple (specified condition must be met)
    * A list of [`ContextSize`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/ContextSize) tuples (any condition must be met - OR logic)

    Condition should be one of the following:

    * `fraction` (float): Fraction of model's context size (0-1)
    * `tokens` (int): Absolute token count
    * `messages` (int): Message count

    At least one condition must be specified. If not provided, summarization will not trigger automatically.

    See the API reference for [`ContextSize`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/ContextSize) for more information.


  
- **`param`** (`ContextSize`): How much context to preserve after summarization. Specify exactly one of:

    * `fraction` (float): Fraction of model's context size to keep (0-1)
    * `tokens` (int): Absolute token count to keep
    * `messages` (int): Number of recent messages to keep

    See the API reference for [`ContextSize`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/ContextSize) for more information.


  
- **`param`** (`function`): Custom token counting function. Defaults to character-based counting.


  
- **`param`** (`string`): Custom prompt template for summarization. Uses built-in template if not specified. The template should include `{messages}` placeholder where conversation history will be inserted.


  
- **`param`** (`number`): Maximum number of tokens to include when generating the summary. Messages will be trimmed to fit this limit before summarization.


  
- **`param`** (`string`): **Deprecated:** Use `summary_prompt` to provide the full prompt instead.


  
- **`param`** (`number`): **Deprecated:** Use `trigger: ("tokens", value)` instead. Token threshold for triggering summarization.


  
- **`param`** (`number`): **Deprecated:** Use `keep: ("messages", value)` instead. Recent messages to preserve.


</details>


<details>
<summary>Full example</summary>

The summarization middleware monitors message token counts and automatically summarizes older messages when thresholds are reached.

  **Trigger conditions** control when summarization runs:

  * Single condition object (specified must be met)
  * Array of conditions (any condition must be met - OR logic)
  * Each condition can use `fraction` (of model's context size), `tokens` (absolute count), or `messages` (message count)

  **Keep condition** control how much context to preserve (specify exactly one):

  * `fraction` - Fraction of model's context size to keep
  * `tokens` - Absolute token count to keep
  * `messages` - Number of recent messages to keep

  ```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
  from langchain.agents import create_agent
  from langchain.agents.middleware import SummarizationMiddleware


  # Single condition: trigger if tokens >= 4000
  agent = create_agent(
      model="gpt-4.1",
      tools=[your_weather_tool, your_calculator_tool],
      middleware=[
          SummarizationMiddleware(
              model="gpt-4.1-mini",
              trigger=("tokens", 4000),
              keep=("messages", 20),
          ),
      ],
  )

  # Multiple conditions: trigger if number of tokens >= 3000 OR messages >= 6
  agent2 = create_agent(
      model="gpt-4.1",
      tools=[your_weather_tool, your_calculator_tool],
      middleware=[
          SummarizationMiddleware(
              model="gpt-4.1-mini",
              trigger=[
                  ("tokens", 3000),
                  ("messages", 6),
              ],
              keep=("messages", 20),
          ),
      ],
  )

  # Using fractional limits
  agent3 = create_agent(
      model="gpt-4.1",
      tools=[your_weather_tool, your_calculator_tool],
      middleware=[
          SummarizationMiddleware(
              model="gpt-4.1-mini",
              trigger=("fraction", 0.8),
              keep=("fraction", 0.3),
          ),
      ],
  )
  ```

</details>


### Human-in-the-loop

Pause agent execution for human approval, editing, or rejection of tool calls before they execute. [Human-in-the-loop](/oss/python/langchain/human-in-the-loop) is useful for the following:

* High-stakes operations requiring human approval (e.g. database writes, financial transactions).
* Compliance workflows where human oversight is mandatory.
* Long-running conversations where human feedback guides the agent.

**API reference:** [`HumanInTheLoopMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/human_in_the_loop/HumanInTheLoopMiddleware)


> ⚠️ **Warning**
>
> Human-in-the-loop middleware requires a [checkpointer](/oss/python/langgraph/persistence#checkpoints) to maintain state across interruptions.


```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver


def your_read_email_tool(email_id: str) -> str:
    """Mock function to read an email by its ID."""
    return f"Email content for ID: {email_id}"

def your_send_email_tool(recipient: str, subject: str, body: str) -> str:
    """Mock function to send an email."""
    return f"Email sent to {recipient} with subject '{subject}'"

agent = create_agent(
    model="gpt-4.1",
    tools=[your_read_email_tool, your_send_email_tool],
    checkpointer=InMemorySaver(),
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "your_send_email_tool": {
                    "allowed_decisions": ["approve", "edit", "reject"],
                },
                "your_read_email_tool": False,
            }
        ),
    ],
)
```


> 💡 **Tip**
>
> For complete examples, configuration options, and integration patterns, see the [Human-in-the-loop documentation](/oss/python/langchain/human-in-the-loop).


> ℹ️ **Note:**
>
> Watch this [video guide](https://www.youtube.com/watch?v=SpfT6-YAVPk) demonstrating Human-in-the-loop middleware behavior.


### Model call limit

Limit the number of model calls to prevent infinite loops or excessive costs. Model call limit is useful for the following:

* Preventing runaway agents from making too many API calls.
* Enforcing cost controls on production deployments.
* Testing agent behavior within specific call budgets.

**API reference:** [`ModelCallLimitMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/model_call_limit/ModelCallLimitMiddleware)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import ModelCallLimitMiddleware
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="gpt-4.1",
    checkpointer=InMemorySaver(),  # Required for thread limiting
    tools=[],
    middleware=[
        ModelCallLimitMiddleware(
            thread_limit=10,
            run_limit=5,
            exit_behavior="end",
        ),
    ],
)
```


> ℹ️ **Note:**
>
> Watch this [video guide](https://www.youtube.com/watch?v=nJEER0uaNkE) demonstrating Model Call Limit middleware behavior.


<details>
<summary>Configuration options</summary>


- **`param`** (`number`): Maximum model calls across all runs in a thread. Defaults to no limit.


  
- **`param`** (`number`): Maximum model calls per single invocation. Defaults to no limit.


  
- **`param`** (`string`): Behavior when limit is reached. Options: `'end'` (graceful termination) or `'error'` (raise exception)


</details>


### Tool call limit

Control agent execution by limiting the number of tool calls, either globally across all tools or for specific tools. Tool call limits are useful for the following:

* Preventing excessive calls to expensive external APIs.
* Limiting web searches or database queries.
* Enforcing rate limits on specific tool usage.
* Protecting against runaway agent loops.

**API reference:** [`ToolCallLimitMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_call_limit/ToolCallLimitMiddleware)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import ToolCallLimitMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[search_tool, database_tool],
    middleware=[
        # Global limit
        ToolCallLimitMiddleware(thread_limit=20, run_limit=10),
        # Tool-specific limit
        ToolCallLimitMiddleware(
            tool_name="search",
            thread_limit=5,
            run_limit=3,
        ),
    ],
)
```


> ℹ️ **Note:**
>
> Watch this [video guide](https://www.youtube.com/watch?v=6gYlaJJ8t0w) demonstrating Tool Call Limit middleware behavior.


<details>
<summary>Configuration options</summary>


- **`param`** (`string`): Name of specific tool to limit. If not provided, limits apply to **all tools globally**.


  
- **`param`** (`number`): Maximum tool calls across all runs in a thread (conversation). Persists across multiple invocations with the same thread ID. Requires a checkpointer to maintain state. `None` means no thread limit.


  
- **`param`** (`number`): Maximum tool calls per single invocation (one user message → response cycle). Resets with each new user message. `None` means no run limit.

    **Note:** At least one of `thread_limit` or `run_limit` must be specified.


  
- **`param`** (`string`): Behavior when limit is reached:

    * `'continue'` (default) - Block exceeded tool calls with error messages, let other tools and the model continue. The model decides when to end based on the error messages.
    * `'error'` - Raise a `ToolCallLimitExceededError` exception, stopping execution immediately
    * `'end'` - Stop execution immediately with a `ToolMessage` and AI message for the exceeded tool call. Only works when limiting a single tool; raises `NotImplementedError` if other tools have pending calls.


</details>


<details>
<summary>Full example</summary>

Specify limits with:

  * **Thread limit** - Max calls across all runs in a conversation (requires checkpointer)
  * **Run limit** - Max calls per single invocation (resets each turn)

  Exit behaviors:

  * `'continue'` (default) - Block exceeded calls with error messages, agent continues
  * `'error'` - Raise exception immediately
  * `'end'` - Stop with ToolMessage + AI message (single-tool scenarios only)

  ```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
  from langchain.agents import create_agent
  from langchain.agents.middleware import ToolCallLimitMiddleware


  global_limiter = ToolCallLimitMiddleware(thread_limit=20, run_limit=10)
  search_limiter = ToolCallLimitMiddleware(tool_name="search", thread_limit=5, run_limit=3)
  database_limiter = ToolCallLimitMiddleware(tool_name="query_database", thread_limit=10)
  strict_limiter = ToolCallLimitMiddleware(tool_name="scrape_webpage", run_limit=2, exit_behavior="error")

  agent = create_agent(
      model="gpt-4.1",
      tools=[search_tool, database_tool, scraper_tool],
      middleware=[global_limiter, search_limiter, database_limiter, strict_limiter],
  )
  ```

</details>


### Model fallback

Automatically fallback to alternative models when the primary model fails. Model fallback is useful for the following:

* Building resilient agents that handle model outages.
* Cost optimization by falling back to cheaper models.
* Provider redundancy across OpenAI, Anthropic, etc.

**API reference:** [`ModelFallbackMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/model_fallback/ModelFallbackMiddleware)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import ModelFallbackMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        ModelFallbackMiddleware(
            "gpt-4.1-mini",
            "claude-3-5-sonnet-20241022",
        ),
    ],
)
```


> ℹ️ **Note:**
>
> Watch this [video guide](https://www.youtube.com/watch?v=8rCRO0DUeIM) demonstrating Model Fallback middleware behavior.


<details>
<summary>Configuration options</summary>


- **`param`** (`string | BaseChatModel`): First fallback model to try when the primary model fails. Can be a model identifier string (e.g., `'openai:gpt-4.1-mini'`) or a `BaseChatModel` instance.


  
- **`param`** (`string | BaseChatModel`): Additional fallback models to try in order if previous models fail


</details>


### PII detection

Detect and handle Personally Identifiable Information (PII) in conversations using configurable strategies. PII detection is useful for the following:

* Healthcare and financial applications with compliance requirements.
* Customer service agents that need to sanitize logs.
* Any application handling sensitive user data.

**API reference:** [`PIIMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/pii/PIIMiddleware)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        PIIMiddleware("credit_card", strategy="mask", apply_to_input=True),
    ],
)
```

#### Custom PII types

You can create custom PII types by providing a `detector` parameter. This allows you to detect patterns specific to your use case beyond the built-in types.

**Three ways to create custom detectors:**

1. **Regex pattern string** - Simple pattern matching

2. **Custom function** - Complex detection logic with validation

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware
import re


# Method 1: Regex pattern string
agent1 = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        PIIMiddleware(
            "api_key",
            detector=r"sk-[a-zA-Z0-9]{32}",
            strategy="block",
        ),
    ],
)

# Method 2: Compiled regex pattern
agent2 = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        PIIMiddleware(
            "phone_number",
            detector=re.compile(r"\+?\d{1,3}[\s.-]?\d{3,4}[\s.-]?\d{4}"),
            strategy="mask",
        ),
    ],
)

# Method 3: Custom detector function
def detect_ssn(content: str) -> list[dict[str, str | int]]:
    """Detect SSN with validation.

    Returns a list of dictionaries with 'text', 'start', and 'end' keys.
    """
    import re
    matches = []
    pattern = r"\d{3}-\d{2}-\d{4}"
    for match in re.finditer(pattern, content):
        ssn = match.group(0)
        # Validate: first 3 digits shouldn't be 000, 666, or 900-999
        first_three = int(ssn[:3])
        if first_three not in [0, 666] and not (900 <= first_three <= 999):
            matches.append({
                "text": ssn,
                "start": match.start(),
                "end": match.end(),
            })
    return matches

agent3 = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        PIIMiddleware(
            "ssn",
            detector=detect_ssn,
            strategy="hash",
        ),
    ],
)
```

**Custom detector function signature:**

The detector function must accept a string (content) and return matches:

Returns a list of dictionaries with `text`, `start`, and `end` keys:

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
def detector(content: str) -> list[dict[str, str | int]]:
    return [
        {"text": "matched_text", "start": 0, "end": 12},
        # ... more matches
    ]
```


> 💡 **Tip**
>
> For custom detectors:
> 
>   * Use regex strings for simple patterns
>   * Use RegExp objects when you need flags (e.g., case-insensitive matching)
>   * Use custom functions when you need validation logic beyond pattern matching
>   * Custom functions give you full control over detection logic and can implement complex validation rules


<details>
<summary>Configuration options</summary>


- **`param`** (`string`): Type of PII to detect. Can be a built-in type (`email`, `credit_card`, `ip`, `mac_address`, `url`) or a custom type name.


  
- **`param`** (`string`): How to handle detected PII. Options:

    * `'block'` - Raise exception when detected
    * `'redact'` - Replace with `[REDACTED_{PII_TYPE}]`
    * `'mask'` - Partially mask (e.g., `****-****-****-1234`)
    * `'hash'` - Replace with deterministic hash


  
- **`param`** (`function | regex`): Custom detector function or regex pattern. If not provided, uses built-in detector for the PII type.


  
- **`param`** (`boolean`): Check user messages before model call


  
- **`param`** (`boolean`): Check AI messages after model call


  
- **`param`** (`boolean`): Check tool result messages after execution


</details>


### To-do list

Equip agents with task planning and tracking capabilities for complex multi-step tasks. To-do lists are useful for the following:

* Complex multi-step tasks requiring coordination across multiple tools.
* Long-running operations where progress visibility is important.


> ℹ️ **Note**
>
> This middleware automatically provides agents with a `write_todos` tool and system prompts to guide effective task planning.


**API reference:** [`TodoListMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/todo/TodoListMiddleware)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[read_file, write_file, run_tests],
    middleware=[TodoListMiddleware()],
)
```


> ℹ️ **Note:**
>
> Watch this [video guide](https://www.youtube.com/watch?v=yTWocbVKQxw) demonstrating To-do List middleware behavior.


<details>
<summary>Configuration options</summary>


- **`param`** (`string`): Custom system prompt for guiding todo usage. Uses built-in prompt if not specified.


  
- **`param`** (`string`): Custom description for the `write_todos` tool. Uses built-in description if not specified.


</details>


### LLM tool selector

Use an LLM to intelligently select relevant tools before calling the main model. LLM tool selectors are useful for the following:

* Agents with many tools (10+) where most aren't relevant per query.
* Reducing token usage by filtering irrelevant tools.
* Improving model focus and accuracy.

This middleware uses structured output to ask an LLM which tools are most relevant for the current query. The structured output schema defines the available tool names and descriptions. Model providers often add this structured output information to the system prompt behind the scenes.

**API reference:** [`LLMToolSelectorMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_selection/LLMToolSelectorMiddleware)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolSelectorMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[tool1, tool2, tool3, tool4, tool5, ...],
    middleware=[
        LLMToolSelectorMiddleware(
            model="gpt-4.1-mini",
            max_tools=3,
            always_include=["search"],
        ),
    ],
)
```


<details>
<summary>Configuration options</summary>


- **`param`** (`string | BaseChatModel`): Model for tool selection. Can be a model identifier string (e.g., `'openai:gpt-4.1-mini'`) or a `BaseChatModel` instance. See [`init_chat_model`](https://reference.langchain.com/python/langchain/chat_models/base/init_chat_model) for more information.

    Defaults to the agent's main model.


  
- **`param`** (`string`): Instructions for the selection model. Uses built-in prompt if not specified.


  
- **`param`** (`number`): Maximum number of tools to select. If the model selects more, only the first max\_tools will be used. No limit if not specified.


  
- **`param`** (`list[string]`): Tool names to always include regardless of selection. These do not count against the max\_tools limit.


</details>


### Tool retry

Automatically retry failed tool calls with configurable exponential backoff. Tool retry is useful for the following:

* Handling transient failures in external API calls.
* Improving reliability of network-dependent tools.
* Building resilient agents that gracefully handle temporary errors.

**API reference:** [`ToolRetryMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_retry/ToolRetryMiddleware)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import ToolRetryMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[search_tool, database_tool],
    middleware=[
        ToolRetryMiddleware(
            max_retries=3,
            backoff_factor=2.0,
            initial_delay=1.0,
        ),
    ],
)
```


<details>
<summary>Configuration options</summary>


- **`param`** (`number`): Maximum number of retry attempts after the initial call (3 total attempts with default)


  
- **`param`** (`list[BaseTool | str]`): Optional list of tools or tool names to apply retry logic to. If `None`, applies to all tools.


  
- **`param`** (`tuple[type[Exception], ...] | callable`): Either a tuple of exception types to retry on, or a callable that takes an exception and returns `True` if it should be retried.


  
- **`param`** (`string | callable`): Behavior when all retries are exhausted. Options:

    * `'return_message'` - Return a `ToolMessage` with error details (allows LLM to handle failure)
    * `'raise'` - Re-raise the exception (stops agent execution)
    * Custom callable - Function that takes the exception and returns a string for the `ToolMessage` content


  
- **`param`** (`number`): Multiplier for exponential backoff. Each retry waits `initial_delay * (backoff_factor ** retry_number)` seconds. Set to `0.0` for constant delay.


  
- **`param`** (`number`): Initial delay in seconds before first retry


  
- **`param`** (`number`): Maximum delay in seconds between retries (caps exponential backoff growth)


  
- **`param`** (`boolean`): Whether to add random jitter (`±25%`) to delay to avoid thundering herd


</details>


<details>
<summary>Full example</summary>

The middleware automatically retries failed tool calls with exponential backoff.

  **Key configuration:**

  * `max_retries` - Number of retry attempts (default: 2)
  * `backoff_factor` - Multiplier for exponential backoff (default: 2.0)
  * `initial_delay` - Starting delay in seconds (default: 1.0)
  * `max_delay` - Cap on delay growth (default: 60.0)
  * `jitter` - Add random variation (default: True)

  **Failure handling:**

  * `on_failure='return_message'` - Return error message
  * `on_failure='raise'` - Re-raise exception
  * Custom function - Function returning error message

  ```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
  from langchain.agents import create_agent
  from langchain.agents.middleware import ToolRetryMiddleware


  agent = create_agent(
      model="gpt-4.1",
      tools=[search_tool, database_tool, api_tool],
      middleware=[
          ToolRetryMiddleware(
              max_retries=3,
              backoff_factor=2.0,
              initial_delay=1.0,
              max_delay=60.0,
              jitter=True,
              tools=["api_tool"],
              retry_on=(ConnectionError, TimeoutError),
              on_failure="continue",
          ),
      ],
  )
  ```

</details>


### Model retry

Automatically retry failed model calls with configurable exponential backoff. Model retry is useful for the following:

* Handling transient failures in model API calls.
* Improving reliability of network-dependent model requests.
* Building resilient agents that gracefully handle temporary model errors.

**API reference:** [`ModelRetryMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/model_retry/ModelRetryMiddleware)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import ModelRetryMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[search_tool, database_tool],
    middleware=[
        ModelRetryMiddleware(
            max_retries=3,
            backoff_factor=2.0,
            initial_delay=1.0,
        ),
    ],
)
```


<details>
<summary>Configuration options</summary>


- **`param`** (`number`): Maximum number of retry attempts after the initial call (3 total attempts with default)


  
- **`param`** (`tuple[type[Exception], ...] | callable`): Either a tuple of exception types to retry on, or a callable that takes an exception and returns `True` if it should be retried.


  
- **`param`** (`string | callable`): Behavior when all retries are exhausted. Options:

    * `'continue'` (default) - Return an `AIMessage` with error details, allowing the agent to potentially handle the failure gracefully
    * `'error'` - Re-raise the exception (stops agent execution)
    * Custom callable - Function that takes the exception and returns a string for the `AIMessage` content


  
- **`param`** (`number`): Multiplier for exponential backoff. Each retry waits `initial_delay * (backoff_factor ** retry_number)` seconds. Set to `0.0` for constant delay.


  
- **`param`** (`number`): Initial delay in seconds before first retry


  
- **`param`** (`number`): Maximum delay in seconds between retries (caps exponential backoff growth)


  
- **`param`** (`boolean`): Whether to add random jitter (`±25%`) to delay to avoid thundering herd


</details>


<details>
<summary>Full example</summary>

The middleware automatically retries failed model calls with exponential backoff.

  ```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
  from langchain.agents import create_agent
  from langchain.agents.middleware import ModelRetryMiddleware


  # Basic usage with default settings (2 retries, exponential backoff)
  agent = create_agent(
      model="gpt-4.1",
      tools=[search_tool],
      middleware=[ModelRetryMiddleware()],
  )

  # Custom exception filtering
  class TimeoutError(Exception):
      """Custom exception for timeout errors."""
      pass

  class ConnectionError(Exception):
      """Custom exception for connection errors."""
      pass

  # Retry specific exceptions only
  retry = ModelRetryMiddleware(
      max_retries=4,
      retry_on=(TimeoutError, ConnectionError),
      backoff_factor=1.5,
  )


  def should_retry(error: Exception) -> bool:
      # Only retry on rate limit errors
      if isinstance(error, TimeoutError):
          return True
      # Or check for specific HTTP status codes
      if hasattr(error, "status_code"):
          return error.status_code in (429, 503)
      return False

  retry_with_filter = ModelRetryMiddleware(
      max_retries=3,
      retry_on=should_retry,
  )

  # Return error message instead of raising
  retry_continue = ModelRetryMiddleware(
      max_retries=4,
      on_failure="continue",  # Return AIMessage with error instead of raising
  )

  # Custom error message formatting
  def format_error(error: Exception) -> str:
      return f"Model call failed: {error}. Please try again later."

  retry_with_formatter = ModelRetryMiddleware(
      max_retries=4,
      on_failure=format_error,
  )

  # Constant backoff (no exponential growth)
  constant_backoff = ModelRetryMiddleware(
      max_retries=5,
      backoff_factor=0.0,  # No exponential growth
      initial_delay=2.0,  # Always wait 2 seconds
  )

  # Raise exception on failure
  strict_retry = ModelRetryMiddleware(
      max_retries=2,
      on_failure="error",  # Re-raise exception instead of returning message
  )
  ```

</details>


### LLM tool emulator

Emulate tool execution using an LLM for testing purposes, replacing actual tool calls with AI-generated responses. LLM tool emulators are useful for the following:

* Testing agent behavior without executing real tools.
* Developing agents when external tools are unavailable or expensive.
* Prototyping agent workflows before implementing actual tools.

**API reference:** [`LLMToolEmulator`](https://reference.langchain.com/python/langchain/agents/middleware/tool_emulator/LLMToolEmulator)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolEmulator

agent = create_agent(
    model="gpt-4.1",
    tools=[get_weather, search_database, send_email],
    middleware=[
        LLMToolEmulator(),  # Emulate all tools
    ],
)
```


<details>
<summary>Configuration options</summary>


- **`param`** (`list[str | BaseTool]`): List of tool names (str) or BaseTool instances to emulate. If `None` (default), ALL tools will be emulated. If empty list `[]`, no tools will be emulated. If array with tool names/instances, only those tools will be emulated.


  
- **`param`** (`string | BaseChatModel`): Model to use for generating emulated tool responses. Can be a model identifier string (e.g., `'anthropic:claude-sonnet-4-6'`) or a `BaseChatModel` instance. Defaults to the agent's model if not specified. See [`init_chat_model`](https://reference.langchain.com/python/langchain/chat_models/base/init_chat_model) for more information.


</details>


<details>
<summary>Full example</summary>

The middleware uses an LLM to generate plausible responses for tool calls instead of executing the actual tools.

  ```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
  from langchain.agents import create_agent
  from langchain.agents.middleware import LLMToolEmulator
  from langchain.tools import tool


  @tool
  def get_weather(location: str) -> str:
      """Get the current weather for a location."""
      return f"Weather in {location}"

  @tool
  def send_email(to: str, subject: str, body: str) -> str:
      """Send an email."""
      return "Email sent"


  # Emulate all tools (default behavior)
  agent = create_agent(
      model="gpt-4.1",
      tools=[get_weather, send_email],
      middleware=[LLMToolEmulator()],
  )

  # Emulate specific tools only
  agent2 = create_agent(
      model="gpt-4.1",
      tools=[get_weather, send_email],
      middleware=[LLMToolEmulator(tools=["get_weather"])],
  )

  # Use custom model for emulation
  agent4 = create_agent(
      model="gpt-4.1",
      tools=[get_weather, send_email],
      middleware=[LLMToolEmulator(model="claude-sonnet-4-6")],
  )
  ```

</details>


### Context editing

Manage conversation context by clearing older tool call outputs when token limits are reached, while preserving recent results. This helps keep context windows manageable in long conversations with many tool calls. Context editing is useful for the following:

* Long conversations with many tool calls that exceed token limits
* Reducing token costs by removing older tool outputs that are no longer relevant
* Maintaining only the most recent N tool results in context

**API reference:** [`ContextEditingMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/context_editing/ContextEditingMiddleware), [`ClearToolUsesEdit`](https://reference.langchain.com/python/langchain/agents/middleware/context_editing/ClearToolUsesEdit)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit

agent = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        ContextEditingMiddleware(
            edits=[
                ClearToolUsesEdit(
                    trigger=100000,
                    keep=3,
                ),
            ],
        ),
    ],
)
```


<details>
<summary>Configuration options</summary>


- **`param`** (`list[ContextEdit]`): List of [`ContextEdit`](https://reference.langchain.com/python/langchain/agents/middleware/context_editing/ContextEdit) strategies to apply


  
- **`param`** (`string`): Token counting method. Options: `'approximate'` or `'model'`


  **[`ClearToolUsesEdit`](https://reference.langchain.com/python/langchain/agents/middleware/context_editing/ClearToolUsesEdit) options:**

  
- **`param`** (`number`): Token count that triggers the edit. When the conversation exceeds this token count, older tool outputs will be cleared.


  
- **`param`** (`number`): Minimum number of tokens to reclaim when the edit runs. If set to 0, clears as much as needed.


  
- **`param`** (`number`): Number of most recent tool results that must be preserved. These will never be cleared.


  
- **`param`** (`boolean`): Whether to clear the originating tool call parameters on the AI message. When `True`, tool call arguments are replaced with empty objects.


  
- **`param`** (`list[string]`): List of tool names to exclude from clearing. These tools will never have their outputs cleared.


  
- **`param`** (`string`): Placeholder text inserted for cleared tool outputs. This replaces the original tool message content.


</details>


<details>
<summary>Full example</summary>

The middleware applies context editing strategies when token limits are reached. The most common strategy is `ClearToolUsesEdit`, which clears older tool results while preserving recent ones.

  **How it works:**

  1. Monitor token count in conversation
  2. When threshold is reached, clear older tool outputs
  3. Keep most recent N tool results
  4. Optionally preserve tool call arguments for context

  ```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
  from langchain.agents import create_agent
  from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit


  agent = create_agent(
      model="gpt-4.1",
      tools=[search_tool, your_calculator_tool, database_tool],
      middleware=[
          ContextEditingMiddleware(
              edits=[
                  ClearToolUsesEdit(
                      trigger=2000,
                      keep=3,
                      clear_tool_inputs=False,
                      exclude_tools=[],
                      placeholder="[cleared]",
                  ),
              ],
          ),
      ],
  )
  ```

</details>


### Shell tool

Expose a persistent shell session to agents for command execution. Shell tool middleware is useful for the following:

* Agents that need to execute system commands
* Development and deployment automation tasks
* Testing and validation workflows
* File system operations and script execution


> ⚠️ **Warning**
>
> **Security consideration**: Use appropriate execution policies (`HostExecutionPolicy`, `DockerExecutionPolicy`, or `CodexSandboxExecutionPolicy`) to match your deployment's security requirements.


> ℹ️ **Note**
>
> **Limitation**: Persistent shell sessions do not currently work with interrupts (human-in-the-loop). We anticipate adding support for this in the future.


**API reference:** [`ShellToolMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/shell_tool/ShellToolMiddleware)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import (
    ShellToolMiddleware,
    HostExecutionPolicy,
)

agent = create_agent(
    model="gpt-4.1",
    tools=[search_tool],
    middleware=[
        ShellToolMiddleware(
            workspace_root="/workspace",
            execution_policy=HostExecutionPolicy(),
        ),
    ],
)
```


<details>
<summary>Configuration options</summary>


- **`param`** (`str | Path | None`): Base directory for the shell session. If omitted, a temporary directory is created when the agent starts and removed when it ends.


  
- **`param`** (`tuple[str, ...] | list[str] | str | None`): Optional commands executed sequentially after the session starts


  
- **`param`** (`tuple[str, ...] | list[str] | str | None`): Optional commands executed before the session shuts down


  
- **`param`** (`BaseExecutionPolicy | None`): Execution policy controlling timeouts, output limits, and resource configuration. Options:

    * `HostExecutionPolicy` - Full host access (default); best for trusted environments where the agent already runs inside a container or VM
    * `DockerExecutionPolicy` - Launches a separate Docker container for each agent run, providing harder isolation
    * `CodexSandboxExecutionPolicy` - Reuses the Codex CLI sandbox for additional syscall/filesystem restrictions


  
- **`param`** (`tuple[RedactionRule, ...] | list[RedactionRule] | None`): Optional redaction rules to sanitize command output before returning it to the model.

    
> ⚠️ **Warning**
>
> Redaction rules are applied post execution and do not prevent exfiltration of secrets or sensitive data when using `HostExecutionPolicy`.


  
- **`param`** (`str | None`): Optional override for the registered shell tool description


  
- **`param`** (`Sequence[str] | str | None`): Optional shell executable (string) or argument sequence used to launch the persistent session. Defaults to `/bin/bash`.


  
- **`param`** (`Mapping[str, Any] | None`): Optional environment variables to supply to the shell session. Values are coerced to strings before command execution.


</details>


<details>
<summary>Full example</summary>

The middleware provides a single persistent shell session that agents can use to execute commands sequentially.

  **Execution policies:**

  * `HostExecutionPolicy` (default) - Native execution with full host access
  * `DockerExecutionPolicy` - Isolated Docker container execution
  * `CodexSandboxExecutionPolicy` - Sandboxed execution via Codex CLI

  ```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
  from langchain.agents import create_agent
  from langchain.agents.middleware import (
      ShellToolMiddleware,
      HostExecutionPolicy,
      DockerExecutionPolicy,
      RedactionRule,
  )


  # Basic shell tool with host execution
  agent = create_agent(
      model="gpt-4.1",
      tools=[search_tool],
      middleware=[
          ShellToolMiddleware(
              workspace_root="/workspace",
              execution_policy=HostExecutionPolicy(),
          ),
      ],
  )

  # Docker isolation with startup commands
  agent_docker = create_agent(
      model="gpt-4.1",
      tools=[],
      middleware=[
          ShellToolMiddleware(
              workspace_root="/workspace",
              startup_commands=["pip install requests", "export PYTHONPATH=/workspace"],
              execution_policy=DockerExecutionPolicy(
                  image="python:3.11-slim",
                  command_timeout=60.0,
              ),
          ),
      ],
  )

  # With output redaction (applied post execution)
  agent_redacted = create_agent(
      model="gpt-4.1",
      tools=[],
      middleware=[
          ShellToolMiddleware(
              workspace_root="/workspace",
              redaction_rules=[
                  RedactionRule(pii_type="api_key", detector=r"sk-[a-zA-Z0-9]{32}"),
              ],
          ),
      ],
  )
  ```

</details>


### File search

Provide Glob and Grep search tools over a filesystem. File search middleware is useful for the following:

* Code exploration and analysis
* Finding files by name patterns
* Searching code content with regex
* Large codebases where file discovery is needed

**API reference:** [`FilesystemFileSearchMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/file_search/FilesystemFileSearchMiddleware)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from langchain.agents.middleware import FilesystemFileSearchMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[],
    middleware=[
        FilesystemFileSearchMiddleware(
            root_path="/workspace",
            use_ripgrep=True,
        ),
    ],
)
```


<details>
<summary>Configuration options</summary>


- **`param`** (`str`): Root directory to search. All file operations are relative to this path.


  
- **`param`** (`bool`): Whether to use ripgrep for search. Falls back to Python regex if ripgrep is unavailable.


  
- **`param`** (`int`): Maximum file size to search in MB. Files larger than this are skipped.


</details>


<details>
<summary>Full example</summary>

The middleware adds two search tools to agents:

  **Glob tool** - Fast file pattern matching:

  * Supports patterns like `**/*.py`, `src/**/*.ts`
  * Returns matching file paths sorted by modification time

  **Grep tool** - Content search with regex:

  * Full regex syntax support
  * Filter by file patterns with `include` parameter
  * Three output modes: `files_with_matches`, `content`, `count`

  ```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
  from langchain.agents import create_agent
  from langchain.agents.middleware import FilesystemFileSearchMiddleware
  from langchain.messages import HumanMessage


  agent = create_agent(
      model="gpt-4.1",
      tools=[],
      middleware=[
          FilesystemFileSearchMiddleware(
              root_path="/workspace",
              use_ripgrep=True,
              max_file_size_mb=10,
          ),
      ],
  )

  # Agent can now use glob_search and grep_search tools
  result = agent.invoke({
      "messages": [HumanMessage("Find all Python files containing 'async def'")]
  })

  # The agent will use:
  # 1. glob_search(pattern="**/*.py") to find Python files
  # 2. grep_search(pattern="async def", include="*.py") to find async functions
  ```

</details>


### Filesystem middleware

Context engineering is a main challenge in building effective agents. This is particularly difficult when using tools that return variable-length results (for example, `web_search` and RAG), as long tool results can quickly fill your context window.

`FilesystemMiddleware` from [Deep Agents](/oss/python/deepagents/overview) provides four tools for interacting with both short-term and long-term memory:

* `ls`: List the files in the filesystem
* `read_file`: Read an entire file or a certain number of lines from a file
* `write_file`: Write a new file to the filesystem
* `edit_file`: Edit an existing file in the filesystem

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from deepagents.middleware.filesystem import FilesystemMiddleware

# FilesystemMiddleware is included by default in create_deep_agent
# You can customize it if building a custom agent
agent = create_agent(
    model="claude-sonnet-4-6",
    middleware=[
        FilesystemMiddleware(
            backend=None,  # Optional: custom backend (defaults to StateBackend)
            system_prompt="Write to the filesystem when...",  # Optional custom addition to the system prompt
            custom_tool_descriptions={
                "ls": "Use the ls tool when...",
                "read_file": "Use the read_file tool to..."
            }  # Optional: Custom descriptions for filesystem tools
        ),
    ],
)
```

#### Short-term vs. long-term filesystem

By default, these tools write to a local "filesystem" in your graph state. To enable persistent storage across threads, configure a `CompositeBackend` that routes specific paths (like `/memories/`) to a `StoreBackend`.

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from deepagents.middleware import FilesystemMiddleware
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
                routes={"/memories/": StoreBackend()}
            ),
            custom_tool_descriptions={
                "ls": "Use the ls tool when...",
                "read_file": "Use the read_file tool to..."
            }  # Optional: Custom descriptions for filesystem tools
        ),
    ],
)
```

When you configure a `CompositeBackend` with a `StoreBackend` for `/memories/`, any files prefixed with **/memories/** are saved to persistent storage and survive across different threads. Files without this prefix remain in ephemeral state storage.

### Subagent

Handing off tasks to subagents isolates context, keeping the main (supervisor) agent's context window clean while still going deep on a task.

The subagents middleware from [Deep Agents](/oss/python/deepagents/overview) allows you to supply subagents through a `task` tool.

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.tools import tool
from langchain.agents import create_agent
from deepagents.middleware.subagents import SubAgentMiddleware


@tool
def get_weather(city: str) -> str:
    """Get the weather in a city."""
    return f"The weather in {city} is sunny."

agent = create_agent(
    model="claude-sonnet-4-6",
    middleware=[
        SubAgentMiddleware(
            default_model="claude-sonnet-4-6",
            default_tools=[],
            subagents=[
                {
                    "name": "weather",
                    "description": "This subagent can get weather in cities.",
                    "system_prompt": "Use the get_weather tool to get the weather in a city.",
                    "tools": [get_weather],
                    "model": "gpt-4.1",
                    "middleware": [],
                }
            ],
        )
    ],
)
```

A subagent is defined with a **name**, **description**, **system prompt**, and **tools**. You can also provide a subagent with a custom **model**, or with additional **middleware**. This can be particularly useful when you want to give the subagent an additional state key to share with the main agent.

For more complex use cases, you can also provide your own prebuilt LangGraph graph as a subagent.

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langchain.agents import create_agent
from deepagents.middleware.subagents import SubAgentMiddleware
from deepagents import CompiledSubAgent
from langgraph.graph import StateGraph

# Create a custom LangGraph graph
def create_weather_graph():
    workflow = StateGraph(...)
    # Build your custom graph
    return workflow.compile()

weather_graph = create_weather_graph()

# Wrap it in a CompiledSubAgent
weather_subagent = CompiledSubAgent(
    name="weather",
    description="This subagent can get weather in cities.",
    runnable=weather_graph
)

agent = create_agent(
    model="claude-sonnet-4-6",
    middleware=[
        SubAgentMiddleware(
            default_model="claude-sonnet-4-6",
            default_tools=[],
            subagents=[weather_subagent],
        )
    ],
)
```

In addition to any user-defined subagents, the main agent has access to a `general-purpose` subagent at all times. This subagent has the same instructions as the main agent and all the tools it has access to. The primary purpose of the `general-purpose` subagent is context isolation—the main agent can delegate a complex task to this subagent and get a concise answer back without bloat from intermediate tool calls.

## Provider-specific middleware

These middleware are optimized for specific LLM providers. See each provider's documentation for full details and examples.


Prompt caching, bash tool, text editor, memory, and file search middleware for Claude models.
  
Prompt caching middleware for Amazon Bedrock models.
  
Content moderation middleware for OpenAI models.
  

***


  
> ℹ️ **Note:**
>
> [Edit this page on GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/middleware/built-in.mdx) or [file an issue](https://github.com/langchain-ai/docs/issues/new/choose).


  
> ℹ️ **Note:**
>
> [Connect these docs](/use-these-docs) to Claude, VSCode, and more via MCP for real-time answers.

