# Meta-Builder-v3: Current Langfuse Integration — In-Depth Analysis

> **Document scope:** This file catalogs, explains, and evaluates everything meta-builder-v3 currently does with Langfuse. It is deliberately descriptive before being critical — understand the existing integration fully before reading the gap analysis in `02-missed-opportunities.md`.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Integration Footprint at a Glance](#2-integration-footprint-at-a-glance)
3. [Tracing Infrastructure](#3-tracing-infrastructure)
   - 3.1 [@observe Decorator — The Backbone](#31-observe-decorator--the-backbone)
   - 3.2 [propagate_attributes — Cross-Thread Context](#32-propagate_attributes--cross-thread-context)
   - 3.3 [LangChain CallbackHandler](#33-langchain-callbackhandler)
   - 3.4 [Trace Attribute Management](#34-trace-attribute-management)
   - 3.5 [The DEPRECATED with_tracing_channel Decorator](#35-the-deprecated-with_tracing_channel-decorator)
   - 3.6 [Langfuse Client Singleton](#36-langfuse-client-singleton)
4. [Prompt Management](#4-prompt-management)
   - 4.1 [PromptManager Class Architecture](#41-promptmanager-class-architecture)
   - 4.2 [Cache Strategy](#42-cache-strategy)
   - 4.3 [LangChain Template Bridge](#43-langchain-template-bridge)
   - 4.4 [Fallback Behavior](#44-fallback-behavior)
   - 4.5 [Prompt Metadata Propagation](#45-prompt-metadata-propagation)
5. [Debugger API Integration](#5-debugger-api-integration)
   - 5.1 [Error Trace Queries](#51-error-trace-queries)
   - 5.2 [Error Classification Logic](#52-error-classification-logic)
   - 5.3 [Tracing Report Endpoint](#53-tracing-report-endpoint)
6. [Pipeline Event Bus (Complementary System)](#6-pipeline-event-bus-complementary-system)
7. [Configuration and Environment Setup](#7-configuration-and-environment-setup)
8. [File-by-File Reference Map](#8-file-by-file-reference-map)
9. [Strengths of the Current Integration](#9-strengths-of-the-current-integration)
10. [Patterns and Anti-Patterns](#10-patterns-and-anti-patterns)
11. [Integration Depth Assessment Table](#11-integration-depth-assessment-table)
12. [Conclusion](#12-conclusion)

---

## 1. Executive Summary

Meta-builder-v3 contains **132 Langfuse references across 34 source files**. On the surface, that level of penetration suggests a mature, deeply integrated observability posture. In practice, the integration is broad but shallow: nearly every function is instrumented with `@observe`, which provides comprehensive span coverage, but the system stops there. Evaluation, experiments, datasets, scoring, and advanced prompt management — the capabilities that turn raw traces into actionable intelligence — are entirely absent.

The integration breaks down into three rough tiers:

| Tier | Capability | Current State |
|------|-----------|---------------|
| **Heavy** | Distributed tracing via `@observe` | Production-grade, 90+ usages |
| **Partial** | Prompt management via `PromptManager` | Production label only, no A/B, no config |
| **Minimal** | Error analysis via Debugger API | Narrow read-only usage |
| **Absent** | Scoring, datasets, experiments, annotation | 0 usages |
| **Absent** | Sessions, users, environments, sampling | Infrastructure exists, rarely applied |

The practical effect: meta-builder-v3 uses Langfuse as an expensive, well-instrumented **logging system**. Every span is captured. None of the spans are evaluated, scored, grouped into experiments, or used to drive quality improvement loops. The system generates observability data but does not consume it programmatically.

### By the numbers

- **132** Langfuse import/usage references
- **34** source files with Langfuse code
- **~90+** `@observe` decorated functions
- **7** distinct Langfuse Python SDK features used
- **0** calls to `langfuse.score()`
- **0** dataset objects
- **0** experiment runs in Langfuse
- **0** annotation queue interactions
- **~30%** of Langfuse's feature surface utilized

---

## 2. Integration Footprint at a Glance

```
meta-builder-v3/
├── src/
│   ├── api/
│   │   └── routes/           ← 22 @observe references (highest concentration)
│   │       └── debugger.py   ← Langfuse API queries for error traces
│   ├── config.py             ← configure_tracing() setup
│   ├── llm/
│   │   └── factory.py        ← 2 @observe references
│   ├── orchestrator/
│   │   └── pipeline_events.py ← Custom event bus (complements Langfuse)
│   └── utils/
│       ├── prompts.py        ← PromptManager (full Langfuse prompt integration)
│       └── tracing.py        ← Central tracing utilities (Langfuse client, handlers)
└── agents/
    ├── ensemble_retriever.py   (7)
    ├── guide_retriever.py      (4)
    ├── retrieval_gate.py       (3)
    ├── semantic_router.py      (2)
    ├── critique_refiner.py     (3)
    ├── evaluator_agent.py      (3)
    ├── optimizer.py            (2)
    ├── don_draper_agent.py     (2)
    ├── sal_romano_agent.py     (2)
    ├── graph_factory.py        (2)
    ├── gym.py                  (2)
    ├── curator_agent.py        (5)
    ├── grounding_gate.py       (3)
    ├── librarian.py            (6)
    ├── web_research.py         (3)
    ├── planning_agent.py      (10)  ← second-highest single-file count
    ├── remediation_agent.py    (2)
    ├── request_processor.py    (5)
    ├── decision_gate.py        (4)
    ├── tool_acquisition.py     (4)
    ├── tracking.py             (5)
    ├── runner.py               (2)
    ├── candidate_queue.py      (3)
    ├── monitor.py              (3)
    ├── sources.py              (3)
    ├── verification_agent.py   (5)
    └── vision_agent.py         (7)
```

---

## 3. Tracing Infrastructure

The tracing infrastructure lives primarily in `src/utils/tracing.py` and is consumed across the entire codebase. It provides six distinct facilities.

### 3.1 @observe Decorator — The Backbone

**Import pattern:**
```python
from langfuse import observe
```

**Usage forms observed across the codebase:**

```python
# Form 1: Auto-named span (uses function name)
@observe()
async def run_pipeline(self, request: dict) -> dict:
    ...

# Form 2: Explicitly named span (preferred for clarity)
@observe(name="ensemble_retriever_retrieve")
async def retrieve(self, query: str) -> list[Document]:
    ...

# Form 3: Named span with semantic grouping
@observe(name="planning_agent_plan")
async def plan(self, context: PlanningContext) -> Plan:
    ...
```

`@observe` is the single most-used Langfuse primitive in the codebase. It wraps a function, creates a Langfuse span for its execution, and automatically captures:

- Start and end timestamps
- Exceptions (if raised)
- The span name (either function name or the explicit `name=` argument)
- Parent span from context (enabling trace tree construction)

**Distribution by file (selected):**

| File | Count | Role |
|------|-------|------|
| `src/api/routes/` (all route files) | 22 | HTTP endpoint spans |
| `planning_agent.py` | 10 | Planning sub-steps |
| `ensemble_retriever.py` | 7 | Retrieval chain steps |
| `vision_agent.py` | 7 | Vision processing steps |
| `librarian.py` | 6 | Document management operations |
| `curator_agent.py` | 5 | Content curation steps |
| `request_processor.py` | 5 | Request lifecycle steps |
| `tracking.py` | 5 | State tracking operations |
| `verification_agent.py` | 5 | Verification pipeline steps |

**Design decision — named vs. unnamed spans:**

The codebase uses both forms inconsistently. Where `@observe(name="X")` is used, trace UIs show human-readable span names that survive refactoring. Where `@observe()` is used, the span name is the Python function name, which is brittle if functions are renamed. `planning_agent.py` has the highest count (10) and appears to use explicit naming consistently, suggesting it was the most carefully instrumented file.

**Thread and async compatibility:**

`@observe` works correctly with both `async def` and regular `def` functions. Meta-builder-v3 is a mixed async/sync codebase, and the decorator handles both. The `propagate_attributes` context manager (section 3.2) supplements this where threading breaks the automatic context propagation.

### 3.2 propagate_attributes — Cross-Thread Context

**Import pattern:**
```python
from langfuse import observe, propagate_attributes
```

**Used in:** `optimizer.py`, `gym.py`, `librarian.py`, `tracing.py`

**Purpose:** Langfuse's `@observe` uses Python's `contextvars` to propagate trace context from parent to child spans. This works automatically in async code (since asyncio preserves context) and in simple synchronous call chains. It breaks when:
- A new thread is spawned (threads do not inherit `contextvars` context)
- A thread pool executor is used
- Work is dispatched via `concurrent.futures`

`propagate_attributes` solves this by explicitly capturing the current context and applying it inside a `with` block, which can then be used to seed the child thread's context.

**Usage example from `tracing.py`:**
```python
with propagate_attributes(tags=[channel]):
    # Child spans created inside this block will be associated
    # with the current trace and will carry the channel tag
    result = await some_async_operation()
```

**Usage in `optimizer.py` and `gym.py`:**

These files run evolutionary optimization loops that may spawn multiple evaluation workers. `propagate_attributes` ensures that evaluation spans appear as children of the optimizer's trace root rather than as disconnected orphan traces.

**Usage in `librarian.py`:**

The Librarian manages document storage operations that may involve background I/O threads. `propagate_attributes` ensures retrieval and indexing spans correctly nest under the originating request trace.

**Assessment:** This is a sophisticated and correct use of `propagate_attributes`. Many Langfuse integrations miss this entirely, leading to orphaned spans and broken trace trees in threaded environments. Meta-builder's use of it in the three most thread-heavy components (optimizer, gym, librarian) demonstrates intentional instrumentation design.

### 3.3 LangChain CallbackHandler

**Import pattern:**
```python
from langfuse.langchain import CallbackHandler
```

**Defined in:** `src/utils/tracing.py`

**Full implementation:**
```python
def get_tracing_handler(channel: str = None, session_id: str = None, user_id: str = None) -> CallbackHandler:
    """
    Create a LangChain CallbackHandler for Langfuse tracing.
    The handler auto-instruments all LangChain chain/LLM/tool invocations.
    """
    handler = CallbackHandler()
    return handler

def get_runnable_config(
    channel: str,
    session_id: str = None,
    user_id: str = None
) -> RunnableConfig:
    """
    Build a LangChain RunnableConfig that injects Langfuse tracing.
    """
    handler = get_tracing_handler(channel=channel, session_id=session_id, user_id=user_id)
    metadata = {}
    if channel:
        metadata["langfuse_tags"] = [channel]
    if session_id:
        metadata["langfuse_session_id"] = session_id
    if user_id:
        metadata["langfuse_user_id"] = user_id
    return {"callbacks": [handler], "metadata": metadata}
```

**How it works:** The `CallbackHandler` registers as a LangChain callback, intercepting events fired by LangChain's callback system at every node in a chain: `on_llm_start`, `on_llm_end`, `on_chain_start`, `on_chain_end`, `on_tool_start`, `on_tool_end`, and so on. It translates these into Langfuse observations, automatically creating:

- **LLM observations** for every model call (capturing model name, prompt, completion, token counts, latency)
- **Chain observations** for every LangChain chain execution
- **Tool observations** for every tool invocation

**Metadata injection:** The `langfuse_tags`, `langfuse_session_id`, and `langfuse_user_id` metadata keys are special: the Langfuse CallbackHandler reads them from LangChain's metadata dict and applies them to the trace. This is the official way to attach Langfuse attributes to LangChain-traced runs.

**Critical observation:** The `get_tracing_handler` function accepts `channel`, `session_id`, and `user_id` parameters but does **not pass them to `CallbackHandler()`**. They are only used to build the `metadata` dict in `get_runnable_config`. This means the handler itself has no awareness of these values — they only flow through LangChain's metadata mechanism. If a LangChain chain doesn't properly propagate metadata (which is common with nested chains), these values may be lost.

### 3.4 Trace Attribute Management

**Defined in:** `src/utils/tracing.py`

```python
def set_trace_attributes(
    name: str = None,
    session_id: str = None,
    user_id: str = None,
    tags: list[str] = None,
    metadata: dict = None
) -> None:
    """
    Update the current trace with attributes. Must be called from within
    an @observe-decorated function to have an active trace context.
    """
    get_langfuse_client().update_current_trace(
        name=name,
        session_id=session_id,
        user_id=user_id,
        tags=tags,
        metadata=metadata,
    )
```

**What this does:** `update_current_trace()` enriches the root trace (not just the current span) with metadata. This is how trace-level attributes are set when you're deep in a call stack — you don't need a reference to the original trace object.

**Where this is called:** `tracking.py` (5 references), `request_processor.py` (5 references), and within API routes. The pattern appears to be: enter an `@observe`-decorated function → call `set_trace_attributes` with request context → proceed with the operation. This ensures traces are enriched with contextual information even when the initial entry point (the API route) didn't have all the information needed to set attributes immediately.

**Observed limitation:** `session_id` and `user_id` are optional and appear to be passed inconsistently. This means most traces in Langfuse cannot be grouped by session or attributed to users, limiting the ability to replay user journeys or compute per-user quality metrics.

### 3.5 The DEPRECATED with_tracing_channel Decorator

```python
def with_tracing_channel(channel: str):
    """
    DEPRECATED: Prefer @observe() and set_trace_attributes().
    
    This decorator wraps a function in a propagate_attributes context
    that applies a channel tag to all child spans.
    """
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            with propagate_attributes(tags=[channel]):
                return await func(*args, **kwargs)
        return wrapper
    return decorator
```

**Status:** Marked deprecated in source comments. Superseded by the cleaner pattern of using `@observe()` on the entry point and then calling `set_trace_attributes(tags=[channel])` inside the function body.

**Why it was deprecated:** `with_tracing_channel` required knowing the channel at decoration time (import time), which was inflexible. The current pattern allows dynamic tag assignment at runtime based on request parameters. The deprecated decorator is presumably still present in some older agent files that haven't been migrated.

**Architectural lesson:** The evolution from `with_tracing_channel` → `@observe() + set_trace_attributes()` shows the team iterating toward a cleaner separation of concerns: `@observe` handles span lifecycle, `set_trace_attributes` handles semantic enrichment.

### 3.6 Langfuse Client Singleton

**Defined in:** `src/utils/tracing.py`

```python
from langfuse import Langfuse, get_client, propagate_attributes
from langfuse.langchain import CallbackHandler

_client: Optional[Langfuse] = None

def get_langfuse_client() -> Langfuse:
    """
    Returns the shared Langfuse client instance.
    Uses get_client() for SDK v3 compatibility (returns the global client
    initialized by the SDK's auto-configuration from environment variables).
    Falls back to creating a new instance if get_client() returns None.
    """
    global _client
    if _client is None:
        _client = get_client()
    return _client
```

**Design notes:**

- Uses `get_client()` (SDK v3 pattern) rather than `Langfuse()` constructor directly. `get_client()` returns the global client initialized by the SDK's environment variable auto-configuration (`LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`), which means the SDK's own initialization path (including any SDK-level configuration) is respected.
- The module-level `_client` variable is a lazy-initialized singleton. This avoids the cost of creating a new Langfuse connection on every call while still deferring initialization until first use.
- Thread safety: the pattern is not thread-safe under highly concurrent initialization (two threads could both see `_client is None` and both call `get_client()`). In practice, `get_client()` returns the same global instance, so the worst case is redundant assignment of the same value — not a real bug.

**Used for:**
- `update_current_trace()` calls in `set_trace_attributes`
- Direct Langfuse API calls in `debugger.py` (`langfuse.api.trace.list(...)`, `langfuse.api.observations.get_many(...)`)

---

## 4. Prompt Management

### 4.1 PromptManager Class Architecture

**Defined in:** `src/utils/prompts.py`

```python
from langfuse import get_client
from langchain_core.prompts import ChatPromptTemplate, PromptTemplate

class PromptManager:
    """
    Centralized prompt management using Langfuse as the prompt registry.
    Supports both chat and completion prompt types.
    """
    
    def __init__(self, client=None):
        self.client = client or get_client()
    
    @lru_cache(maxsize=100)
    def get_langfuse_prompt(self, name: str, label: str = "production", fallback: str = None):
        """
        Fetch a prompt from Langfuse by name and label.
        Returns None if the prompt doesn't exist and no fallback is provided.
        """
        try:
            return self.client.get_prompt(name, label=label)
        except Exception:
            return None
    
    def get_template(self, name: str, label: str = "production", fallback: str = None):
        """
        Fetch a prompt and return it as a LangChain PromptTemplate or ChatPromptTemplate.
        """
        lf_prompt = self.get_langfuse_prompt(name, label, fallback)
        if lf_prompt:
            template_str = lf_prompt.get_langchain_prompt()
            if lf_prompt.type == 'chat':
                return ChatPromptTemplate.from_template(template_str)
            return PromptTemplate.from_template(template_str)
        if fallback:
            return PromptTemplate.from_template(fallback)
        return None
    
    def get_prompt_metadata(self, name: str, label: str = "production"):
        """
        Return metadata dict for linking a prompt to its Langfuse trace.
        """
        lf_prompt = self.get_langfuse_prompt(name, label)
        return {"langfuse_prompt": lf_prompt} if lf_prompt else {}
```

**What Langfuse's `get_prompt()` returns:** A `LangfusePrompt` object with:
- `.get_langchain_prompt()` — serializes the prompt to LangChain-compatible format
- `.type` — `"chat"` or `"text"`
- `.config` — arbitrary JSON config attached to this prompt version (model params, temperature, etc.)
- `.version` — integer version number
- `.name` — prompt name
- `.label` — the label used to fetch it

The `PromptManager` only uses `get_langchain_prompt()` and `.type`. The `.config` field (which could carry model parameters like `model`, `temperature`, `max_tokens`) is completely ignored.

### 4.2 Cache Strategy

```python
@lru_cache(maxsize=100)
def get_langfuse_prompt(self, name: str, label: str = "production", fallback: str = None):
```

**How `lru_cache` works here:** Python's `functools.lru_cache` caches by argument values. Since `label` defaults to `"production"` and is almost never varied, effectively all calls with the same `name` hit the cache after the first fetch. The cache persists for the lifetime of the `PromptManager` instance.

**Problems with this approach:**
1. **No TTL.** If a prompt is updated in Langfuse, the cache will never invalidate during a running process. A service restart is required to pick up changes.
2. **Defeats Langfuse's versioning.** The whole point of Langfuse prompt management is that you can iterate on prompts without redeploying. `lru_cache` with no expiry makes the system behave as if prompts were hardcoded.
3. **Langfuse SDK provides `cache_ttl_seconds`.** The `get_prompt()` call accepts a `cache_ttl_seconds` parameter that implements TTL-based caching at the SDK level, including proper invalidation. Using `lru_cache` bypasses this.

The correct approach:
```python
# SDK-native caching with TTL
prompt = self.client.get_prompt(name, label=label, cache_ttl_seconds=300)
```

### 4.3 LangChain Template Bridge

`lf_prompt.get_langchain_prompt()` converts Langfuse's native prompt format to a LangChain-compatible string. Langfuse uses `{{variable}}` (double braces) as its native variable syntax; LangChain uses `{variable}` (single braces). `get_langchain_prompt()` handles this conversion automatically.

```python
# Langfuse stores:  "Analyze {{content}} for {{criteria}}"
# get_langchain_prompt() returns:  "Analyze {content} for {criteria}"
# LangChain then uses this as a PromptTemplate
```

**Type dispatch:**
```python
if lf_prompt.type == 'chat':
    return ChatPromptTemplate.from_template(template_str)
return PromptTemplate.from_template(template_str)
```

This correctly handles the two Langfuse prompt types. `ChatPromptTemplate` expects message-structured prompts (system/human/ai); `PromptTemplate` handles single-string completion prompts.

**Gap:** `ChatPromptTemplate.from_template(template_str)` creates a simple human-message template from a string. This loses the multi-message structure that Langfuse chat prompts can express (system prompt + user prompt as separate messages). The proper conversion for chat prompts uses `ChatPromptTemplate.from_messages(lf_prompt.get_langchain_prompt())` — note the plural form, which accepts a list of message tuples.

### 4.4 Fallback Behavior

```python
def get_template(self, name, label="production", fallback=None):
    lf_prompt = self.get_langfuse_prompt(name, label, fallback)
    if lf_prompt:
        # Use Langfuse version
        ...
    if fallback:
        return PromptTemplate.from_template(fallback)
    return None
```

The fallback mechanism is a safety net for when Langfuse is unreachable or the prompt doesn't exist. This is good defensive design — the system won't crash if Langfuse goes down, it will just use the hardcoded template.

**However:** The fallback path means it's impossible to know from the logs alone whether a given pipeline run used the Langfuse-managed prompt or the fallback. No metric is emitted, no trace attribute is set, and no alert fires when fallbacks are used. In production, silent fallback to hardcoded prompts is an invisible quality regression.

### 4.5 Prompt Metadata Propagation

```python
def get_prompt_metadata(self, name: str, label: str = "production"):
    lf_prompt = self.get_langfuse_prompt(name, label)
    return {"langfuse_prompt": lf_prompt} if lf_prompt else {}
```

**Purpose:** Langfuse's LangChain integration can link a trace to the specific prompt version that generated it, if the prompt object is passed through LangChain's metadata. The `langfuse_prompt` key is recognized by the `CallbackHandler`.

**Implementation gap:** The method returns the metadata dict, but it is unclear from the codebase whether this dict is consistently passed to LangChain invocations. If callers call `get_template()` without also calling `get_prompt_metadata()` and passing the result to the chain invocation, then prompt-to-trace linkage is broken. The prompt version that generated a trace cannot be determined from the Langfuse UI.

---

## 5. Debugger API Integration

### 5.1 Error Trace Queries

**File:** `src/api/routes/debugger.py`

This is the only place in meta-builder-v3 that reads data back from Langfuse (rather than writing to it). It provides an internal API endpoint for surfacing recent pipeline errors.

```python
from src.utils.tracing import get_langfuse_client

# Endpoint: GET /api/debugger/errors?limit=N
async def get_recent_errors(limit: int = 10):
    langfuse = get_langfuse_client()
    
    # Fetch recent traces tagged as pipeline_rag
    traces_response = langfuse.api.trace.list(
        tags=["pipeline_rag"],
        limit=limit * 2  # Fetch extra to account for filtering
    )
    
    errors = []
    for trace in traces_response.data:
        # Fetch all observations for this trace
        obs_response = langfuse.api.observations.get_many(
            trace_id=trace.id,
            limit=100
        )
        # Classify each error observation
        for obs in obs_response.data:
            if obs.level == "ERROR":
                error_type = classify_error(obs)
                errors.append({
                    "trace_id": trace.id,
                    "observation_id": obs.id,
                    "error_type": error_type,
                    "message": obs.status_message,
                    "timestamp": obs.start_time,
                })
    
    return errors[:limit]
```

**What this enables:** Developers can query the debugger endpoint to see categorized recent errors without having to manually navigate the Langfuse UI. This is a genuinely useful capability — it surfaces Langfuse's data through the application's own API surface.

### 5.2 Error Classification Logic

The debugger classifies errors into four categories:

| Category | Description | Example |
|----------|-------------|---------|
| `expected_test` | Known test scenarios, not real errors | Test harness injections |
| `transient_timeout` | Network/API timeouts that self-resolve | LLM provider timeouts |
| `schema_parse` | Pydantic/JSON parsing failures | Malformed LLM output |
| `hard_failure` | Unrecoverable errors requiring investigation | Missing required fields, crashes |

This classification layer is valuable because it reduces alert fatigue — not every `ERROR`-level observation is a real problem. `expected_test` and `transient_timeout` categories are noise in a production system; `hard_failure` is the signal worth acting on.

**Gap:** Classification logic is heuristic (pattern matching on error messages). There is no feedback loop to Langfuse — the classification result is never written back as a score or annotation, so Langfuse's own analytics cannot be used to track error type trends over time.

### 5.3 Tracing Report Endpoint

```python
# Endpoint: POST /api/debugger/parse-report
async def parse_tracing_report(report_content: str):
    """
    Parse a tracing_report.md file and return structured analysis.
    Expected format: the output of meta-builder's internal tracing report generator.
    """
    # Parses markdown-formatted tracing reports into structured dicts
    # Used by the development team for offline analysis
```

This endpoint suggests meta-builder has a separate tracing report generation system (`tracing_report.md`) that exists alongside Langfuse. The fact that this report format needs a dedicated parser implies it's a parallel observability mechanism rather than derived from Langfuse data.

---

## 6. Pipeline Event Bus (Complementary System)

**File:** `src/orchestrator/pipeline_events.py`

The source code comments explicitly describe this as: *"Complements Langfuse tracing with programmatic, in-process event delivery."*

```python
# Event types defined in pipeline_events.py:
class PipelineStarted(Event): ...
class PhaseStarted(Event): ...
class GatePassed(Event): ...
class GateFailed(Event): ...
class StructuralValidationFailed(Event): ...
class RePlanTriggered(Event): ...
class PipelineCompleted(Event): ...
class TierFallback(Event): ...
```

**How it differs from Langfuse:**

| Dimension | Langfuse Tracing | Pipeline Event Bus |
|-----------|-----------------|-------------------|
| Delivery | Async, network, eventual | Synchronous, in-process, immediate |
| Consumers | Langfuse server | Local subscribers |
| Durability | Persistent | Ephemeral |
| Latency | ~100ms+ | <1ms |
| Query | Langfuse UI/API | Application logic |
| Purpose | Observability | Reactive logic |

**Why both exist:** Langfuse traces are inherently asynchronous — they are shipped to the Langfuse server and cannot drive real-time decisions within the pipeline. When the Conductor needs to react to a `GateFailed` event (e.g., triggering re-planning), it needs the event synchronously, in-process. The event bus serves this purpose. Langfuse serves the retrospective analysis purpose.

**The missed opportunity:** These two systems are not wired together. A `GateFailed` event in the event bus should ideally also update the current Langfuse trace (e.g., set `level=WARNING`, add metadata about which gate failed and why). Currently, the gate failure appears in Langfuse only because the gate function is `@observe`-decorated and its exception is automatically captured — but the Langfuse span may not have the same structured metadata that the event bus carries.

---

## 7. Configuration and Environment Setup

**File:** `src/config.py`

```python
def configure_tracing(config: dict = None) -> None:
    """
    Configure Langfuse tracing from the application config.
    Must be called before any @observe-decorated functions execute.
    """
    tracing = config.get("tracing", {}) if config else {}
    enabled = tracing.get("enabled", False)
    provider = tracing.get("provider", "langfuse")
    
    if enabled and provider == "langfuse":
        os.environ["LANGFUSE_TRACING_ENABLED"] = "true"
        # Langfuse SDK reads these from environment:
        # LANGFUSE_PUBLIC_KEY — project public key
        # LANGFUSE_SECRET_KEY — project secret key
        # LANGFUSE_HOST      — Langfuse server URL (default: https://cloud.langfuse.com)
    else:
        os.environ["LANGFUSE_TRACING_ENABLED"] = "false"
```

**Config YAML structure:**
```yaml
tracing:
  enabled: true
  provider: langfuse
```

**What's missing from config:**
- No `environment` field (dev/staging/prod trace separation)
- No `release` field (deployment version tracking)
- No `sample_rate` field (per-environment sampling)
- No `debug` field (SDK debug logging)
- No per-provider config (all Langfuse credentials must be in environment variables, not config file)

The binary `enabled: true/false` approach means there's no concept of "trace some requests but not all" — either everything is traced or nothing is. For a production system handling variable load, this is inflexible.

---

## 8. File-by-File Reference Map

The following table shows every file with Langfuse references and the specific features used:

| File | Ref Count | Features Used |
|------|-----------|---------------|
| `src/api/routes/*.py` | 22 | `@observe`, `set_trace_attributes` |
| `planning_agent.py` | 10 | `@observe(name=...)` |
| `ensemble_retriever.py` | 7 | `@observe(name=...)` |
| `vision_agent.py` | 7 | `@observe(name=...)` |
| `librarian.py` | 6 | `@observe`, `propagate_attributes` |
| `curator_agent.py` | 5 | `@observe(name=...)` |
| `request_processor.py` | 5 | `@observe`, `set_trace_attributes` |
| `tracking.py` | 5 | `@observe`, `set_trace_attributes`, `propagate_attributes` |
| `verification_agent.py` | 5 | `@observe(name=...)` |
| `guide_retriever.py` | 4 | `@observe(name=...)` |
| `decision_gate.py` | 4 | `@observe(name=...)` |
| `tool_acquisition.py` | 4 | `@observe(name=...)` |
| `retrieval_gate.py` | 3 | `@observe(name=...)` |
| `critique_refiner.py` | 3 | `@observe(name=...)` |
| `evaluator_agent.py` | 3 | `@observe(name=...)` |
| `grounding_gate.py` | 3 | `@observe(name=...)` |
| `web_research.py` | 3 | `@observe(name=...)` |
| `candidate_queue.py` | 3 | `@observe(name=...)` |
| `monitor.py` | 3 | `@observe(name=...)` |
| `sources.py` | 3 | `@observe(name=...)` |
| `semantic_router.py` | 2 | `@observe()` |
| `optimizer.py` | 2 | `@observe`, `propagate_attributes` |
| `don_draper_agent.py` | 2 | `@observe()` |
| `sal_romano_agent.py` | 2 | `@observe()` |
| `graph_factory.py` | 2 | `@observe()` |
| `gym.py` | 2 | `@observe`, `propagate_attributes` |
| `remediation_agent.py` | 2 | `@observe()` |
| `runner.py` | 2 | `@observe()` |
| `llm/factory.py` | 2 | `@observe()` |
| `src/utils/tracing.py` | — | All primitives (definition file) |
| `src/utils/prompts.py` | — | `PromptManager`, `get_client` |
| `src/api/routes/debugger.py` | — | `langfuse.api.trace.list`, `.observations.get_many` |
| `src/config.py` | — | `configure_tracing` |
| `src/orchestrator/pipeline_events.py` | — | Complementary system (no Langfuse imports) |

---

## 9. Strengths of the Current Integration

The current integration has genuine strengths that should be preserved as the system evolves.

### 9.1 Pervasive Span Coverage

With `@observe` on virtually every agent, gate, retriever, and route handler, the trace tree for any pipeline execution is comprehensive. A developer can open any trace in the Langfuse UI and see the complete execution path from HTTP request to final response, with timing at every step. This level of instrumentation is rarely achieved in practice — most systems only instrument the top-level entry points.

### 9.2 Correct Thread Context Propagation

The use of `propagate_attributes` in the three most thread-intensive components (optimizer, gym, librarian) shows awareness of the contextvars pitfall. Many Langfuse integrations produce broken trace trees in threaded environments; meta-builder's instrumentation handles this correctly.

### 9.3 Clean Centralization

All Langfuse-specific code is centralized in `src/utils/tracing.py` and `src/utils/prompts.py`. Agents and routes import from these utilities, not directly from Langfuse. This means:
- A single place to update when the Langfuse SDK version changes
- A single place to add features (sampling, environments) without touching 34 files
- The ability to swap Langfuse for a different provider by changing two files

### 9.4 Prompt Management with Fallbacks

The `PromptManager.get_template()` fallback mechanism ensures the system degrades gracefully when Langfuse is unavailable. Production systems that hard-depend on external prompt registries are fragile; the fallback pattern is sound.

### 9.5 LangChain Integration Completeness

By using both `@observe` (for direct Python code) and `CallbackHandler` (for LangChain chains), meta-builder achieves double coverage: the outer Python function is observed by `@observe`, and the inner LangChain invocations are observed by the callback. This avoids gaps in the trace tree where LangChain chains would otherwise appear opaque.

### 9.6 Meaningful Span Naming

The widespread use of `@observe(name="descriptive_name")` rather than relying on Python function names means traces remain readable even after refactoring. `ensemble_retriever_retrieve` is more meaningful in a trace UI than a generic function name.

### 9.7 Debugger as Internal Observability API

The debugger endpoint that queries Langfuse API for error traces is a clever pattern: it makes Langfuse data accessible through the application's own API surface, reducing the need for developers to directly access the Langfuse UI for common debugging tasks.

---

## 10. Patterns and Anti-Patterns

### Patterns (Good)

**Pattern: Centralized tracing utilities**
```python
# Good: All tracing setup in one place
from src.utils.tracing import get_runnable_config, set_trace_attributes

# Agents don't import from langfuse directly (mostly)
```

**Pattern: Named spans for semantic clarity**
```python
@observe(name="planning_agent_decompose_task")
async def decompose_task(self, request: str) -> list[str]:
    ...
```

**Pattern: Defensive prompt fallbacks**
```python
template = prompt_manager.get_template(
    name="evaluator_critique",
    fallback=EVALUATOR_CRITIQUE_FALLBACK_TEMPLATE
)
```

**Pattern: Complementary systems for different concerns**
The event bus handles reactive in-process logic; Langfuse handles retrospective analysis. These are genuinely different concerns that warrant different tools.

### Anti-Patterns (Bad)

**Anti-pattern: lru_cache without TTL on Langfuse prompts**
```python
# Bad: Cache never expires — prompt updates require service restart
@lru_cache(maxsize=100)
def get_langfuse_prompt(self, name, label="production", fallback=None):
    return self.client.get_prompt(name, label=label)

# Good: Use SDK-native TTL caching
def get_langfuse_prompt(self, name, label="production", fallback=None):
    return self.client.get_prompt(name, label=label, cache_ttl_seconds=300)
```

**Anti-pattern: Ignoring prompt.config**
```python
# Bad: Model parameters managed separately from prompt versions
# prompt.config might contain: {"model": "gpt-4o", "temperature": 0.2}
lf_prompt = self.get_langfuse_prompt(name)
template = lf_prompt.get_langchain_prompt()
# prompt.config is never read — model selection happens elsewhere

# Better: Extract model config from prompt version
config = lf_prompt.config or {}
model = config.get("model", DEFAULT_MODEL)
temperature = config.get("temperature", 0.7)
```

**Anti-pattern: Computing scores but not reporting them**
```python
# Bad: EvaluatorAgent computes scores internally but never reports to Langfuse
class EvaluatorAgent:
    @observe(name="evaluator_agent_evaluate")
    async def evaluate(self, output: str, criteria: EvalCriteria) -> EvalResult:
        score = self._compute_score(output, criteria)
        return score  # Score disappears — never sent to Langfuse

# Good: Report scores back
async def evaluate(self, output: str, criteria: EvalCriteria, trace_id: str) -> EvalResult:
    score = self._compute_score(output, criteria)
    get_langfuse_client().score(
        trace_id=trace_id,
        name=criteria.name,
        value=score.value,
        comment=score.reasoning,
    )
    return score
```

**Anti-pattern: Optional session_id that's rarely provided**
```python
# Bad: session_id is optional and callers rarely provide it
def get_runnable_config(channel, session_id=None, user_id=None):
    ...

# Result: Traces cannot be grouped into sessions
# Multi-turn Conductor interactions appear as isolated traces
```

**Anti-pattern: Silent fallback to hardcoded prompts**
```python
# Bad: Fallback is invisible — no metric, no log, no trace attribute
if lf_prompt:
    template = convert(lf_prompt)
if fallback:
    return PromptTemplate.from_template(fallback)
    # Should be: log warning AND set trace attribute AND increment counter
```

**Anti-pattern: Binary tracing (all-or-nothing)**
```python
# Bad: No sampling — either everything is traced or nothing is
if enabled and provider == "langfuse":
    os.environ["LANGFUSE_TRACING_ENABLED"] = "true"

# Good: Support per-route or per-environment sampling
```

---

## 11. Integration Depth Assessment Table

| Langfuse Capability Area | Current State | Depth | Notes |
|--------------------------|--------------|-------|-------|
| **Distributed Tracing** | Active | ████████░░ 80% | Missing: sampling, environments, releases |
| **Span Naming** | Active | ███████░░░ 70% | Mix of named/unnamed spans |
| **Thread Context** | Active | █████████░ 90% | Good use of propagate_attributes |
| **LangChain Integration** | Active | ███████░░░ 70% | CallbackHandler used, metadata gaps |
| **Trace Attributes** | Partial | █████░░░░░ 50% | session_id/user_id rarely passed |
| **Prompt Fetching** | Active | ██████░░░░ 60% | Production label only, no TTL |
| **Prompt Versioning** | Absent | ██░░░░░░░░ 20% | No A/B, no label variety |
| **Prompt Config** | Absent | █░░░░░░░░░ 10% | .config field never read |
| **Prompt-to-Trace Link** | Partial | ███░░░░░░░ 30% | get_prompt_metadata() defined, usage unclear |
| **Scoring** | Absent | ░░░░░░░░░░ 0%  | Zero score() calls |
| **Datasets** | Absent | ░░░░░░░░░░ 0%  | Zero dataset interactions |
| **Experiments** | Absent | ░░░░░░░░░░ 0%  | Zero experiment interactions |
| **Annotation Queues** | Absent | ░░░░░░░░░░ 0%  | Zero annotation interactions |
| **Sessions** | Partial | ██░░░░░░░░ 20% | Field exists, rarely populated |
| **Users** | Partial | ██░░░░░░░░ 20% | Field exists, rarely populated |
| **Environments** | Absent | ░░░░░░░░░░ 0%  | No env separation |
| **Releases** | Absent | ░░░░░░░░░░ 0%  | No version tagging |
| **Log Levels** | Absent | ░░░░░░░░░░ 0%  | No explicit level setting |
| **Masking** | Absent | ░░░░░░░░░░ 0%  | Client-side PII handling only |
| **Sampling** | Absent | ░░░░░░░░░░ 0%  | Binary enable/disable only |
| **Error API** | Active | ██████░░░░ 60% | Debugger reads errors, no write-back |
| **Metrics API** | Absent | ░░░░░░░░░░ 0%  | No programmatic metric queries |
| **User Feedback** | Absent | ░░░░░░░░░░ 0%  | No feedback collection |
| **Spend Monitoring** | Absent | ░░░░░░░░░░ 0%  | No cost alerts |

**Overall integration depth: ~30% of available Langfuse capabilities**

---

## 12. Conclusion

Meta-builder-v3's Langfuse integration is wide but thin. The decision to instrument every function with `@observe` was correct and valuable — it ensures comprehensive trace coverage that most teams would envy. The `PromptManager` provides a clean abstraction for Langfuse-managed prompts with sensible fallbacks.

But the integration stopped at the "capture everything" phase and never advanced to the "act on the data" phase. The system generates rich tracing data and then ignores it. EvaluatorAgent computes quality scores that vanish. AgentGym runs experiments tracked in SQLite that could be tracked in Langfuse. Multi-turn Conductor pipelines are not grouped into sessions. Deployed versions are not tagged as releases.

The result is a system that can answer "what happened?" (via the trace UI) but cannot answer "how is quality trending?", "which prompt version performs better?", or "which users are experiencing failures?" — all questions Langfuse is capable of answering, if the integration went deeper.

The gaps are detailed exhaustively in `02-missed-opportunities.md`.
