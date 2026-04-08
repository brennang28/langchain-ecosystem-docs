# Langfuse Learning Guide
## A Comprehensive Reference for LLM Engineering

**Last Updated:** April 2026  
**Audience:** Tech-savvy professionals who want to deeply understand Langfuse without writing code  
**Source:** [Langfuse Official Documentation](https://langfuse.com/docs) | [GitHub: langfuse/langfuse](https://github.com/langfuse/langfuse)

---

## Table of Contents

1. [Part 1: Platform Overview](#part-1-platform-overview)
   - What Langfuse Is and Why It Exists
   - The Three-Product Architecture
   - Key Concepts
   - Data Model Overview
   - Deployment Options
   - Langfuse in the LLM Development Lifecycle
   - Langfuse vs. Alternatives

2. [Part 2: Observability Deep Dive](#part-2-observability-deep-dive)
   - The Core Problem Observability Solves
   - Data Model: Traces and Observations
   - SDK Integration Overview
   - Feature-by-Feature Walkthrough

3. [Part 3: Prompt Management Deep Dive](#part-3-prompt-management-deep-dive)
   - What Prompt Management Solves
   - Data Model: Prompts, Versions, Labels
   - Feature-by-Feature Walkthrough
   - SDK Usage Patterns

4. [Part 4: Evaluation Deep Dive](#part-4-evaluation-deep-dive)
   - What Evaluation Solves
   - Core Concepts: Scores
   - Evaluation Methods
   - The Experiments System

5. [Part 5: Supporting Systems](#part-5-supporting-systems)
   - Metrics and Dashboards
   - API and Data Platform
   - Security and Guardrails
   - Administration

6. [Part 6: Integration Patterns](#part-6-integration-patterns)
   - LangChain and LangGraph
   - OpenAI SDK Integration
   - Custom Python Integration
   - Production Patterns
   - Multi-Service Tracing
   - Evaluation Pipelines

---

# Part 1: Platform Overview

## What Langfuse Is and Why It Exists

Langfuse is an **open-source LLM engineering platform** that helps teams collaboratively debug, analyze, and iterate on their AI applications. It was first publicly released in May 2023 and has grown to over 17,000 GitHub stars, making it the leading open-source option in the LLM observability space.

The core insight behind Langfuse is that building applications with Large Language Models is fundamentally different from building traditional software. In traditional software, a function either returns the right value or it doesn't — the behavior is deterministic. LLM applications are different in three critical ways:

**1. Non-determinism.** The same input can produce different outputs. A GPT-4 response to "summarize this document" changes every time. You cannot test LLM apps the way you test traditional code — you need statistical, continuous evaluation rather than pass/fail unit tests.

**2. Opacity.** When an LLM application misbehaves, it's hard to know why. Did the user's question confuse the model? Did the retrieval system return irrelevant documents? Did the prompt template have a bug? Did one prompt in a chain of five cause the failure? Without detailed tracing, debugging feels like guesswork.

**3. Rapid iteration.** LLM application development moves fast. Teams change prompts, swap models, adjust retrieval logic, and change parameters constantly. Without a structured way to version these changes and measure their impact, it's easy to accidentally regress and not notice.

Langfuse was built to solve all three of these problems in one integrated platform, giving teams:
- **Visibility** into what's happening inside their LLM applications (Observability)
- **Control** over the prompts their applications use (Prompt Management)
- **Measurement** of whether their applications are actually working well (Evaluation)

---

## The Three-Product Architecture

Langfuse is structured around three tightly integrated core products, plus a supporting platform layer. Each product can be used independently, but they become significantly more powerful when used together.

```
┌─────────────────────────────────────────────────────────────┐
│                         LANGFUSE                            │
│                                                             │
│  ┌─────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │Observability│  │Prompt Management │  │  Evaluation  │  │
│  │             │  │                  │  │              │  │
│  │  Traces &   │◄─┤  Versioned       │  │  Scores &    │  │
│  │  Monitoring │  │  Prompt CMS      │─►│  Experiments │  │
│  │             │  │                  │  │              │  │
│  └──────┬──────┘  └────────┬─────────┘  └──────┬───────┘  │
│         │                  │                    │          │
│  ┌──────▼──────────────────▼────────────────────▼───────┐  │
│  │              Platform: API, Export, Security          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**How the products interconnect:**

- **Observability → Evaluation:** Every trace captured by Observability can receive Evaluation scores. You can score live production traces automatically (online evaluation) or score batched historical traces (offline evaluation).
- **Prompt Management → Observability:** When a prompt from the Prompt Management system is used in an application, that usage is logged in the trace. You can then see exactly which version of a prompt was used for any given interaction.
- **Prompt Management → Evaluation:** You can run experiments that test a specific prompt version against a dataset, comparing quality scores across versions. This closes the loop: edit a prompt, test it, measure the result.
- **All three → Dashboard:** A unified analytics dashboard aggregates signals from all three products — cost, latency, quality scores, prompt versions — into a single operational view.

---

## Key Concepts

Before diving into each product, here are the fundamental building blocks you'll encounter throughout Langfuse.

### Traces

A **trace** is the top-level record of a single interaction with your LLM application. Think of it like a receipt for one complete transaction. If a user sends a question to your AI chatbot, that entire request-response cycle — from the moment the user hits send to the moment they see an answer — is one trace.

A trace captures:
- The overall input (what the user asked)
- The overall output (what the application returned)
- Timing (how long the whole thing took)
- User information, session information, metadata, tags
- All the sub-operations that happened in between (these are "Observations")

### Observations

**Observations** are the building blocks within a trace. Every meaningful operation inside your application can be logged as an observation. There are three types:

- **Generations:** Records of calls to an LLM. A generation captures the exact prompt sent to the model, the model's response, the model name, the number of tokens used, and the cost.
- **Spans:** Records of any other operation — retrieving documents from a database, calling an API, running a function, formatting data. Spans capture start/end times (measuring duration) and inputs/outputs.
- **Events:** Point-in-time markers for things that don't have duration — logging that a user clicked something, that a cache was hit, that a specific code path was reached.

Observations can nest inside each other, creating a hierarchical tree structure that mirrors the actual call structure of your application.

### Scores

**Scores** are evaluation measurements attached to traces or individual observations. A score has a name (like "helpfulness" or "accuracy"), a value (numeric, boolean, or categorical), and optionally a comment.

Scores are the universal currency of evaluation in Langfuse. Whether a human annotator rates a response, an automated LLM-as-a-judge evaluates it, or your application code measures something specific — all of these produce Scores that can be compared and analyzed.

### Prompts

In Langfuse Prompt Management, a **prompt** is a named, versioned template stored in Langfuse's database. You give it a name, write the template text with placeholders for variables, and every time you change it, a new version is created. Your application can fetch the "production" version at runtime without any code changes.

### Datasets

**Datasets** are curated collections of test cases used for evaluation experiments. Each item in a dataset has an input, an optional expected output, and optional metadata. Datasets let you run the same set of real-world or hand-crafted examples against different versions of your application to measure whether things are getting better or worse.

---

## Data Model Overview

Understanding Langfuse's data hierarchy helps everything else make sense:

```
Project
├── Traces
│   ├── Metadata (user, session, tags, release, environment)
│   └── Observations (nested tree)
│       ├── Generations (LLM calls)
│       │   ├── Input prompt
│       │   ├── Output response
│       │   ├── Model name
│       │   ├── Token usage
│       │   └── Cost
│       ├── Spans (any operation)
│       │   ├── Input
│       │   ├── Output
│       │   └── Duration
│       └── Events (point-in-time markers)
│
├── Scores (attached to Traces or Observations)
│   ├── Name, Value, Type (numeric/boolean/categorical)
│   └── Source (API, UI annotation, LLM-as-a-judge)
│
├── Prompts
│   ├── Name
│   ├── Versions (full history)
│   │   ├── Template text with {{variables}}
│   │   └── Config (model, temperature, etc.)
│   └── Labels ("production", "staging", "latest", custom)
│
└── Datasets
    ├── Dataset Items (input, expected_output)
    └── Dataset Runs
        └── Dataset Run Items → Scores
```

---

## Deployment Options

Langfuse offers two deployment paths: the managed cloud service and self-hosting.

### Langfuse Cloud

The managed cloud offering at [cloud.langfuse.com](https://cloud.langfuse.com) is the fastest way to get started. Langfuse handles all infrastructure, updates, and scaling.

**Pricing tiers:**
- **Free tier:** 50,000 observations per month at no cost — sufficient for small-scale development
- **Pro plan:** Starting at $59/month for higher limits and additional features
- **Enterprise plans:** Custom pricing with SLAs, dedicated support, and advanced security

**Best for:** Teams that want to move fast, don't have strict data residency requirements, and prefer not to manage infrastructure.

### Self-Hosted

Langfuse can be deployed entirely on your own infrastructure using Docker or Kubernetes. The self-hosted version uses the exact same codebase as the cloud offering (the core platform is MIT-licensed). Some enterprise features require a commercial license.

**Infrastructure components:**
- A Postgres database (stores traces, prompts, scores, datasets)
- A ClickHouse instance (analytical queries and dashboards)
- The Langfuse application server
- An S3-compatible blob store (optional, for media and exports)

**Best for:** Teams with data compliance requirements (healthcare, finance, government), organizations that want full control over their data, or large-scale deployments where self-hosting is more cost-efficient than cloud pricing.

> **Why This Matters:** Self-hosting isn't just about saving money. For teams handling sensitive data — customer conversations, medical information, proprietary business logic — keeping all LLM inputs and outputs within your own infrastructure is often a legal or regulatory requirement. Langfuse's open-source model means you can inspect the code, verify how data is handled, and modify it if needed.

---

## How Langfuse Fits in the LLM Development Lifecycle

LLM application development typically follows a cycle that Langfuse supports at every stage:

**Stage 1: Build and Prototype**
- Write initial prompts, connect to an LLM, get something working
- Langfuse role: Log traces from the start — even during development — to build a record of how your application behaves
- Use the Prompt Playground to iterate on prompt wording interactively

**Stage 2: Evaluate and Debug**
- Figure out where the application is failing, why, and how to fix it
- Langfuse role: Inspect individual traces to see exactly what went wrong; set up annotation queues for team members to rate outputs; run experiments comparing different prompt versions

**Stage 3: Deploy to Production**
- Release to real users
- Langfuse role: Monitor cost, latency, and quality in real time; use environments (dev/staging/production) to separate traffic; set up automated LLM-as-a-judge evaluation on live traffic

**Stage 4: Iterate and Improve**
- Use production data to inform the next round of improvements
- Langfuse role: Export low-scoring traces to datasets; run experiments testing improvements; promote winning prompt versions to production via a label change; compare results across releases

This cycle repeats continuously. Langfuse is designed to tighten each iteration loop.

---

## Langfuse vs. Alternatives

The LLM observability space has several strong players. Here's how Langfuse compares to the most common alternatives:

### Langfuse vs. LangSmith

**LangSmith** (from the LangChain team) is Langfuse's closest competitor. Both cover tracing, prompt management, and evaluation.

| Dimension | Langfuse | LangSmith |
|-----------|----------|-----------|
| Open source | Yes (MIT license, fully self-hostable) | No (proprietary, self-host requires enterprise license) |
| Framework agnostic | Yes — works with any framework or raw API | LangChain-native; works with others but optimized for LangChain |
| First release | May 2023 | November 2023 |
| GitHub stars | 17,000+ | 670+ |
| Alerting | Via Metrics API and integrations | Native, configurable alerting built in |
| Self-hosting complexity | Moderate (Docker/Kubernetes) | Complex (enterprise closed-source) |
| Best for | Teams wanting open-source flexibility, framework-agnostic stacks, data control | Teams building purely with LangChain/LangGraph who want fast setup |

> **The key tradeoff:** LangSmith is faster to set up if you're using LangChain. Langfuse gives you more control, open-source transparency, and flexibility to switch frameworks without switching observability tools.

### Langfuse vs. Arize / Phoenix

**Arize** and its open-source sibling **Phoenix** come from the ML monitoring world and are particularly strong at:
- Embedding drift detection (tracking when the semantic meaning of inputs shifts over time)
- Integration with traditional ML workflows
- OpenTelemetry-native instrumentation

Langfuse is stronger at:
- Prompt management and versioning
- Experiments and dataset-driven evaluation
- Collaborative annotation workflows

> **When to choose Arize/Phoenix:** If you have a mature ML team with existing monitoring infrastructure and need sophisticated drift detection. If you're focused primarily on LLM app development iteration, Langfuse's integrated product suite is typically the better fit.

### Langfuse vs. Weights & Biases (W&B)

**Weights & Biases** is a general ML experiment tracking platform that has added LLM-specific features. W&B is excellent for:
- Model training and fine-tuning experiments
- Research environments with complex hyperparameter tracking
- Teams already using W&B for traditional ML

Langfuse focuses specifically on LLM *application* development (inference-time observability, prompt management, evaluation). If you're building an LLM application rather than training models, Langfuse is more purpose-built for your needs.

### Langfuse vs. Braintrust

**Braintrust** is a newer commercial platform focused on the evaluation loop. It emphasizes:
- CI/CD integration for automated evaluation
- Exhaustive auto-captured metrics
- A polished playground for product managers

Braintrust's free tier is generous (1M spans) but it is proprietary and not self-hostable. Langfuse's MIT license and self-hosting capability are significant advantages for data-sensitive environments.

---

# Part 2: Observability Deep Dive

## The Core Problem Observability Solves

Imagine you deployed an AI customer support assistant. Users are complaining that it gives wrong answers sometimes. Where do you start debugging?

Without observability, you're in the dark:
- You don't know which specific questions triggered wrong answers
- You don't know what prompt was sent to the LLM
- You don't know if the retrieval system pulled relevant documents
- You don't know how long each step took
- You don't know if the issue is happening 1% of the time or 50% of the time

With Langfuse Observability, every interaction is logged in detail. You can filter traces by date, user, score, error status, or metadata. You can open any individual trace and see a step-by-step timeline: what retrieval returned, what prompt was sent to the LLM, what the LLM responded, how long each step took, how many tokens were used.

This transforms debugging from guesswork into systematic investigation.

---

## Data Model: Traces and Observations

### Traces

A **trace** is the root-level container for a single end-to-end interaction. One user request = one trace (usually). Traces have:

- **ID:** A unique identifier. Can be auto-generated or set by your application (useful for correlating with your own system's request IDs).
- **Name:** A human-readable label, like "customer-support-query" or "document-summarizer".
- **Input:** The overall input to the application (usually the user's message or the data being processed).
- **Output:** The final output produced by the application.
- **Timestamp:** When the trace started.
- **User ID:** Which end user initiated this interaction.
- **Session ID:** Which conversation session this belongs to (for multi-turn applications).
- **Tags:** Free-form labels for filtering and categorization.
- **Metadata:** A key-value store for any custom attributes you want to attach.
- **Release:** Which version of your application code produced this trace.
- **Environment:** Which deployment environment (development, staging, production).
- **Level:** The severity level of the trace (DEBUG, DEFAULT, WARNING, ERROR).

### Observations: Generations

**Generations** are the most important type of observation — they record actual calls to an LLM. A generation captures:

- **Input:** The exact prompt sent to the model. For chat models, this includes the full message array (system prompt, conversation history, user message). For completion models, it's the prompt string.
- **Output:** The model's exact response.
- **Model:** Which model was used (e.g., "gpt-4o", "claude-3-5-sonnet-20241022").
- **Usage:** Token counts — prompt tokens, completion tokens, total tokens.
- **Cost:** Calculated cost of the API call (in USD). Langfuse maintains pricing tables for common models and calculates this automatically.
- **Start/End Time:** The exact timing, enabling precise latency measurement.
- **Completion Start Time:** When the first token of the response arrived (useful for measuring time-to-first-token separately from total latency).
- **Prompt:** A reference to the Langfuse Prompt Management object that was used (linking Observability to Prompt Management).
- **Temperature, Max Tokens:** The generation parameters used.

### Observations: Spans

**Spans** record any non-LLM operation that has meaningful duration. Common uses:
- **Retrieval steps:** Searching a vector database returns a span showing what was searched, what was returned, and how long it took.
- **API calls:** Calling an external service (weather API, database query, web search) can be logged as a span.
- **Processing steps:** Chunking a document, formatting data, running business logic — any step worth measuring.

Spans have:
- **Name:** A descriptive label for the operation.
- **Input/Output:** What went in and what came out.
- **Start/End Time:** Duration measurement.
- **Level:** Severity classification.

### Observations: Events

**Events** are instantaneous markers — they don't have duration. Use them for:
- Logging that a cache was hit
- Recording that a user clicked a specific UI element
- Marking that a certain code branch was taken
- Any other point-in-time signal you want to capture

### Hierarchical Nesting

The power of the observation model is that observations can nest inside each other, creating a tree that mirrors the actual execution structure of your application:

```
Trace: "customer-support-query"
├── Span: "retrieve-context"               (1.2 seconds)
│   └── Event: "cache-miss"
├── Generation: "generate-response"        (2.8 seconds)
│   ├── Input: [system prompt + context + user message]
│   ├── Output: "Here's how you can reset..."
│   └── Usage: 1,847 tokens, $0.014
└── Span: "post-processing"               (0.1 seconds)
    └── Event: "response-filtered"
```

This tree view is displayed in the Langfuse UI as a timeline, making it immediately clear which steps took the most time and where errors occurred.

---

## SDK Integration Overview

Langfuse provides multiple integration paths, ranging from high-level automatic instrumentation to low-level manual control.

### Python SDK: `@observe` Decorator

The most convenient integration for Python applications is the `@observe` decorator. You simply decorate your functions with `@observe`, and Langfuse automatically:
- Creates a trace when the outermost decorated function is called
- Creates nested spans for each inner decorated function
- Captures function inputs and outputs
- Propagates context through the call stack (including across async code)

```python
from langfuse.decorators import observe, langfuse_context

@observe  # This function becomes a Span
def retrieve_documents(query):
    # ... vector search logic ...
    return documents

@observe  # This is the root Trace
def answer_question(user_question):
    docs = retrieve_documents(user_question)
    # LangChain/OpenAI calls are auto-logged as Generations
    response = llm.invoke(prompt.format(context=docs, question=user_question))
    return response
```

When `answer_question()` is called, Langfuse automatically creates a trace called "answer_question" containing a nested span called "retrieve_documents". No manual IDs or context management required.

### LangChain and LangGraph Integration

For applications built with LangChain or LangGraph, Langfuse provides a callback handler that plugs directly into LangChain's execution pipeline:

```python
from langfuse.callback import CallbackHandler

handler = CallbackHandler()

# Use with any LangChain chain or agent
result = my_chain.invoke(
    {"question": user_input},
    config={"callbacks": [handler]}
)
```

This single line of integration causes Langfuse to capture every step in the LangChain execution — chain calls, LLM calls, tool invocations, retrieval steps — as a properly nested trace hierarchy, with no additional instrumentation needed.

For LangGraph agents (which execute as directed graphs of nodes), Langfuse renders the execution as a visual graph showing which nodes were visited, in what order, and how long each took.

### OpenAI SDK Integration

For applications using the OpenAI Python SDK directly (without LangChain), Langfuse provides a simple wrapper:

```python
from langfuse.openai import openai  # Drop-in replacement

# All subsequent OpenAI calls are automatically traced
response = openai.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": user_message}]
)
```

Changing the import line is all that's needed — every OpenAI call is then automatically logged as a generation in Langfuse, with full token usage and cost tracking.

### Low-Level API

For maximum control, Langfuse provides a low-level Python API where you manually create and manage traces and observations:

```python
from langfuse import Langfuse

langfuse = Langfuse()

# Create a trace
trace = langfuse.trace(name="my-application", user_id="user-123")

# Create a generation within the trace
generation = trace.generation(
    name="main-llm-call",
    model="gpt-4o",
    input=[{"role": "user", "content": "Hello"}],
    output="Hi there!",
    usage={"prompt_tokens": 10, "completion_tokens": 5}
)

# Create a span
span = trace.span(name="retrieval", input={"query": "..."}, output=[...])

# Flush all pending events (important before application shutdown)
langfuse.flush()
```

### Other Integrations

Beyond Python, Langfuse integrates with:
- **JavaScript/TypeScript SDK:** Full feature parity with the Python SDK
- **LlamaIndex:** Native integration for RAG pipelines built with LlamaIndex
- **Haystack:** Pipeline instrumentation for Haystack-based applications
- **Anthropic SDK:** Drop-in wrapper similar to the OpenAI integration
- **LiteLLM:** Use LiteLLM as an LLM gateway with Langfuse tracing built in
- **OpenTelemetry:** Since Langfuse is built on OpenTelemetry standards, any system that emits OTEL traces can send data to Langfuse
- **Vercel AI SDK, Semantic Kernel, AutoGen:** Community and official integrations for these frameworks

---

## Feature-by-Feature Walkthrough

### Token and Cost Tracking

Every generation automatically captures token usage from the LLM API response. Langfuse maintains a pricing table for dozens of popular models (GPT-4o, Claude 3.5 Sonnet, Gemini 1.5 Pro, etc.) and calculates the dollar cost of each generation.

You can:
- View cost per individual trace (what did this one user request cost?)
- View cost per user (which users are the most expensive to serve?)
- View cost per model (compare GPT-4 vs. GPT-4o cost at scale)
- View cost trends over time (is cost increasing as usage grows?)
- Set custom pricing for models not in Langfuse's table
- Define custom usage metrics for non-standard pricing models

> **Why This Matters:** LLM costs can grow unexpectedly fast. A feature that costs $0.01 per query becomes $10,000/month at 1M queries/month. Without cost tracking, teams often discover expensive problems only when they see their cloud bill. Langfuse lets you catch cost issues immediately and attribute them to specific features, users, or prompts.

---

### Sessions

A **session** is a group of traces that belong to the same ongoing interaction. For a chatbot, all the messages in a single conversation are one session. You assign traces to a session by setting the `session_id` attribute.

Sessions let you:
- View the full conversation history for any user interaction
- Measure session-level metrics (total tokens per conversation, average turns)
- Filter traces by session to debug multi-turn conversation issues
- See how conversations evolve over multiple exchanges

```python
# All traces with the same session_id are grouped together
trace = langfuse.trace(
    name="chat-message",
    session_id="conversation-abc123",
    user_id="user-456"
)
```

> **Why This Matters:** Many LLM applications are conversational. A single turn of a conversation is not meaningful in isolation — you need to see the whole conversation to understand why the model responded the way it did. Sessions provide that full context.

---

### Users

The `user_id` attribute associates a trace with a specific end user of your application. This enables:
- Per-user cost analysis (which users drive the most API spend?)
- Per-user quality filtering (are certain users getting worse responses?)
- Privacy-compliant tracking (use anonymized user IDs to track behavior without storing PII)
- User-level session grouping

> **Why This Matters:** When a user reports that "the AI gave me a wrong answer," you need to be able to look up all their recent interactions to diagnose the issue. User IDs make this possible.

---

### Tags and Metadata

**Tags** are free-form string labels you can attach to traces or observations. They enable filtering and grouping in the Langfuse UI. Examples: `"rag"`, `"customer-support"`, `"premium-user"`, `"high-latency"`.

**Metadata** is a key-value dictionary for structured attributes. Unlike tags, metadata values can be any data type and can be queried specifically. Example:
```python
metadata={
    "product_version": "2.1.4",
    "customer_tier": "enterprise",
    "query_type": "factual",
    "document_count": 5
}
```

> **Why This Matters:** Tags and metadata are how you slice your observability data. Instead of looking at all traces, you want to look at only the traces from premium enterprise customers who used the RAG feature and had high latency. Tags and metadata make these complex filters possible.

---

### Environments

Langfuse supports multiple **environments** — typically `development`, `staging`, and `production`. Setting the environment on a trace (or configuring it at the SDK level) ensures that:
- Dev and staging traces don't pollute your production dashboards
- You can compare behavior across environments
- You can apply different evaluation rules per environment
- Cost and quality metrics are separated by environment

> **Why This Matters:** Development teams generate lots of test traces. Without environment separation, these test traces skew your production quality and cost metrics, making it hard to know the true state of your live application.

---

### Releases and Versioning

The `release` attribute lets you tag traces with a version identifier matching your application's deployment version. This enables:
- Side-by-side comparison of metrics before and after a deployment
- Rollback analysis (did the new release cause quality degradation?)
- Attribution of issues to specific code versions

> **Why This Matters:** When you deploy a new version of your application, you want to know within hours whether quality or cost changed. Release tagging makes before/after comparisons immediate.

---

### Trace IDs and Distributed Tracing

Langfuse supports **custom trace IDs**, which allows you to correlate Langfuse traces with your application's own request tracking system. If your application logs every API request with a `request_id`, you can use that same ID as the Langfuse trace ID, so you can look up any request in both your application logs and Langfuse simultaneously.

For **distributed tracing** across multiple services (for example, a frontend server calling a backend service that calls an LLM), Langfuse allows you to pass trace context between services. The second service picks up the trace ID and adds its observations to the same trace, creating a unified view across service boundaries.

---

### Log Levels

Each trace and observation can be assigned a **log level**:
- `DEBUG`: Low-level diagnostic information, typically filtered out in production
- `DEFAULT`: Normal operational events
- `WARNING`: Something unexpected happened but the request succeeded
- `ERROR`: The request failed or produced an error

Log levels enable you to filter your trace view to only see errors and warnings — critical for incident investigation — without being drowned in normal traffic.

---

### Agent Graphs

For agentic applications — where an AI agent uses tools, makes decisions, and loops — Langfuse renders the execution as a **directed graph** rather than a simple linear timeline. Each node in the graph represents a step (an LLM call, a tool invocation, a decision point), and edges show the flow between steps.

This is particularly powerful for **LangGraph** applications, where the execution is literally defined as a graph. Langfuse's graph visualization matches the mental model developers use when building agents, making it much easier to trace which path the agent took through its decision tree.

> **Why This Matters:** Debugging agent behavior requires understanding the sequence of decisions an agent made. A flat list of LLM calls doesn't tell you that the agent chose to use Tool A because of what Tool B returned. A graph visualization makes these causal chains obvious.

---

### Multi-Modality

Langfuse supports tracing applications that work with multiple media types, not just text. You can log:
- **Images:** Attach image inputs/outputs to generations (for vision models like GPT-4o Vision)
- **Audio:** Log audio inputs and outputs (for speech-to-text and text-to-speech pipelines)
- **Video:** Log video content in traces

All media is stored and rendered in the Langfuse UI, so you can inspect exactly what a vision model "saw" when it produced a particular response.

---

### Masking (PII Redaction)

The **masking** feature allows you to redact or transform sensitive data before it is sent to Langfuse. You configure a masking function in the SDK that receives trace data and returns a sanitized version.

Use cases:
- Strip credit card numbers, social security numbers, or health information from prompts before logging
- Replace user names with anonymized identifiers
- Redact proprietary business information that shouldn't leave your network

```python
def mask_pii(data):
    # Replace email addresses with a placeholder
    import re
    return re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', 
                  '[EMAIL_REDACTED]', str(data))

# Configure masking at the SDK level
langfuse = Langfuse(mask=mask_pii)
```

> **Why This Matters:** LLM applications frequently process sensitive user data. Logging this data to an observability platform — even a self-hosted one — may violate privacy regulations (GDPR, HIPAA, CCPA). Masking lets you get the observability benefits without the compliance risks.

---

### Sampling

**Sampling** lets you control what percentage of traces are sent to Langfuse. At high traffic volumes, logging every single trace can be expensive and unnecessary. A 10% sample rate may give you sufficient statistical signal at 1/10th the cost.

You can configure sampling at the SDK level (e.g., sample 10% of all traces) or implement custom sampling logic in your application (e.g., always log error traces but sample only 5% of successful ones).

> **Why This Matters:** At scale, observability costs money — both in API quota for Langfuse Cloud and in infrastructure for self-hosted deployments. Intelligent sampling lets you control costs while maintaining statistical visibility into your application's behavior.

---

### Queuing and Batching

Langfuse SDKs send trace data **asynchronously** in the background. When your application logs a trace, the SDK:
1. Adds the event to an in-memory queue
2. Returns control to your application immediately (zero blocking)
3. Periodically flushes the queue by sending batches of events to Langfuse

This means that tracing adds essentially no latency to your application's response time. Users never wait for Langfuse to process data.

> **Why This Matters:** Observability tools that add latency to your application are worse than useless — they hurt the user experience you're trying to improve. Langfuse's async architecture makes it safe to enable full tracing even in latency-sensitive production environments.

---

### Comments

Team members can add **comments** to any trace or observation in the Langfuse UI. Comments support collaborative debugging — a developer can flag a suspicious trace, add a comment explaining the issue, and a colleague can reply with a diagnosis.

This turns Langfuse into a shared workspace for the team, rather than just an individual debugging tool.

---

### Corrections

The **corrections** feature allows annotators to propose corrected outputs for a trace. If an LLM produced an incorrect answer, a human can write the correct answer and attach it to the trace. These corrections can then be used as training data or as part of evaluation workflows.

---

### User Feedback

Langfuse can capture **end-user feedback** from your application's UI. If your chatbot has a thumbs-up/thumbs-down button, you can wire those clicks to Langfuse via the SDK or API, and they will appear as scores attached to the corresponding trace.

```python
# When user clicks thumbs up in your UI
langfuse.score(
    trace_id=trace_id_of_the_response,
    name="user_feedback",
    value=1,  # 1 = thumbs up, 0 = thumbs down
    comment="User marked response as helpful"
)
```

This creates a ground-truth signal from actual users, which is often the most valuable evaluation data available.

> **Why This Matters:** User feedback is real-world quality measurement. An automated evaluator might score a response as "relevant" but users might still find it unhelpful. Capturing the actual thumbs-ups and thumbs-downs tells you what's actually working in production.

---

### URL Linking (Deep Links)

Every trace in Langfuse has a permanent URL. You can share this URL with a colleague, embed it in a bug report, or link to it from your application's admin dashboard. This makes it easy to share specific problematic traces for collaborative investigation.

---

### MCP Tracing

Langfuse supports tracing for **Model Context Protocol (MCP)** tool servers. MCP is an emerging standard for how AI models interact with external tools and data sources. When your application uses an MCP server (for example, to browse the web, query a database, or read files), those tool interactions are logged as observations in the trace.

This extends observability to the full scope of what agentic AI systems do — not just LLM calls, but all the tools they use.

---

# Part 3: Prompt Management Deep Dive

## What Prompt Management Solves

The prompts in an LLM application are essentially the application's logic. A poorly worded system prompt can make an otherwise capable model useless. A well-crafted prompt can make even a smaller, cheaper model perform well.

Teams working on LLM applications change their prompts constantly — tweaking wording, adding examples, changing structure, adjusting tone. Without a systematic approach, this creates serious problems:

**The version control problem.** If prompts live in code files, changing them requires a code deployment. If they live in a shared document or Notion page, there's no history of what changed, when, and why. If they're hardcoded in environment variables, no one can track what's in production.

**The collaboration problem.** Product managers, domain experts, and writers often have the most insight into what a prompt should say — but they can't edit a prompt that lives in a Python file without involving a developer. This creates bottlenecks.

**The deployment problem.** "We need to fix the prompt in production immediately" — if prompts are in code, this requires a deployment pipeline and minutes to hours of lag time. Teams need to be able to update production prompts instantly.

**The experimentation problem.** Which version of the prompt is actually better? Without a structured way to test and compare, teams make decisions based on intuition and small manual tests.

Langfuse Prompt Management solves all four problems by providing a prompt CMS (Content Management System) that serves as the single source of truth for all prompts.

---

## Data Model: Prompts, Versions, Labels

### Prompts

A **prompt** in Langfuse is identified by a unique name (like `"customer-support-system-prompt"` or `"summarization-template"`). Under that name, Langfuse maintains a complete version history — every change creates a new version, and old versions are never deleted.

**Two prompt types:**

1. **Text prompts:** A single string with optional `{{variable}}` placeholders. Used for completion-style models or when you need a simple template.

2. **Chat prompts:** An array of message objects (role + content), matching the message format used by chat LLMs like GPT-4. Each message can contain variables. This is the most common type for modern LLM applications.

### Versions

Every edit creates a new **version** with an incrementing number (v1, v2, v3...). Each version is immutable — once created, it doesn't change. This gives you a complete, auditable history of how a prompt evolved.

Each version stores:
- The full prompt template text or message array
- The config (model settings like temperature, max tokens, model name)
- Who created it and when
- Optional notes describing what changed

### Labels

**Labels** are named pointers to specific versions. Think of them like Git tags. The most important built-in labels are:

- `"production"`: The version currently serving live user traffic
- `"staging"`: A candidate version being tested before promotion to production
- `"latest"`: Always points to the most recently created version

You can also create custom labels for your own workflow (e.g., `"v2-experiment"`, `"customer-a-test"`).

When your application fetches a prompt, it fetches by label — not by version number. This means you can update which version is "production" without changing any application code. You simply relabel in the Langfuse UI, and the next time your application fetches the prompt, it gets the new version.

### Variables

Prompt templates use **`{{variable_name}}`** syntax for dynamic content. At runtime, your application fetches the template and compiles it with actual values:

```
Template: "You are a helpful assistant for {{company_name}}. 
           The user's account tier is {{account_tier}}."

Compiled: "You are a helpful assistant for Acme Corp. 
           The user's account tier is premium."
```

### Config

Each prompt version can have an attached **config** object storing model parameters:
```json
{
  "model": "gpt-4o",
  "temperature": 0.3,
  "max_tokens": 1000,
  "top_p": 0.95
}
```

Your application can read these config values at runtime and use them when calling the LLM, ensuring the prompt and its intended model parameters stay in sync.

---

## Feature-by-Feature Walkthrough

### Version Control

Every time you save a change to a prompt — whether through the Langfuse UI, the SDK, or the API — a new version is created automatically. The full version history is accessible in the UI, showing:
- The diff between any two versions (what exactly changed)
- Who made each change
- When each change was made
- Which label is currently applied to each version

> **Why This Matters:** Version control for prompts is as important as version control for code. When a production prompt starts producing worse results, you need to know exactly what changed and be able to roll back instantly. Without versioning, you're flying blind.

---

### Labels and Promotion

The label system creates a structured workflow for promoting prompts through environments:

1. You write a new prompt version → it automatically gets the `"latest"` label
2. You test it in staging → you apply the `"staging"` label
3. After validation, you promote it to production → you move the `"production"` label

This happens without any code changes. Your application always fetches `label="production"`, and what "production" means is controlled in Langfuse.

---

### Variables and Message Placeholders

Variables (`{{variable_name}}`) handle dynamic content within a prompt. When your application compiles the prompt, it passes a dictionary of variable values:

```python
prompt = langfuse.get_prompt("support-prompt", label="production")
compiled = prompt.compile(
    company_name="Acme Corp",
    user_tier="premium",
    user_question="How do I reset my password?"
)
# compiled is now a ready-to-send message array or string
```

**Message Placeholders** are a more advanced feature for chat prompts. They allow you to insert an entire array of messages into a specific position in the chat template. This is useful for injecting retrieved documents, conversation history, or few-shot examples dynamically:

```
System: "You are a helpful assistant. Use the following context:
{{context_placeholder}}"  ← An array of retrieved document messages
                              gets inserted here at runtime
User: "{{user_question}}"
```

---

### Config: Model Settings

Attaching model configuration to a prompt version creates a single source of truth for both the prompt text and the model settings intended to work with it. If the optimal temperature for a creative writing prompt is 0.9 but for a factual Q&A prompt is 0.1, these settings are stored alongside the prompt text.

Your application reads the config and uses it when making the LLM call, ensuring consistency between what you tested in the playground and what runs in production.

---

### A/B Testing

Langfuse enables **A/B testing of prompts** by supporting multiple active production labels simultaneously. For example:
- Version 5 gets the label `"prod-a"` (control)
- Version 8 gets the label `"prod-b"` (treatment)

Your application randomly assigns users to either `prod-a` or `prod-b` with a configurable weight (e.g., 50/50 or 80/20). Because both versions log to Langfuse with the prompt version recorded, you can then compare:
- Latency between the two versions
- Token usage and cost
- Quality scores (if you have automated evaluation)
- User feedback signals

> **Why This Matters:** Prompt changes that seem obviously better in a small manual test often turn out to have mixed results at scale. A/B testing gives you statistical confidence that a new prompt version actually improves the metrics you care about before you fully commit to it.

---

### Caching

Fetching a prompt from Langfuse's API on every request adds a small network round-trip. Langfuse SDKs provide **client-side caching** that stores fetched prompts in memory for a configurable TTL (time-to-live, defaulting to 60 seconds).

On the first request after startup, the SDK fetches the latest production prompt from Langfuse. For the next 60 seconds, all requests use the cached version. After the TTL expires, the SDK fetches again — getting any updates you've made in the meantime.

This design provides:
- Near-zero latency for prompt fetching in hot paths
- Near-real-time propagation of prompt updates (within one TTL period)
- **Guaranteed availability:** If Langfuse is unreachable, the SDK falls back to the last-fetched version (or a compile-time fallback you configure)

---

### Composability

**Composable prompts** allow you to include one prompt within another using a special syntax. For example, if you have a shared "tone guidelines" prompt that should appear in multiple other prompts, you can reference it by name rather than copy-pasting:

```
System: "{{> tone-guidelines}}"
         ↑ Langfuse resolves this reference and inserts the 
           current production version of "tone-guidelines" here

User: "{{user_message}}"
```

When the parent prompt is fetched, Langfuse automatically resolves these references and returns the fully composed prompt. This enables modular, DRY (Don't Repeat Yourself) prompt architecture where shared components can be updated in one place.

---

### Folders

As a project grows, the number of named prompts can become unwieldy. **Folders** let you organize prompts in a hierarchy:

```
prompts/
├── customer-support/
│   ├── system-prompt
│   ├── escalation-template
│   └── follow-up-email
├── document-processing/
│   ├── summarization
│   └── extraction
└── shared/
    ├── tone-guidelines
    └── safety-rules
```

---

### GitHub Integration

Langfuse offers **bi-directional sync** with GitHub repositories. You can:
- Store your prompts as YAML files in a Git repository
- Have changes in GitHub automatically create new versions in Langfuse
- Export Langfuse prompts back to GitHub for code review workflows

This brings prompts into your standard engineering review process — pull requests, code reviews, CI checks — while still benefiting from Langfuse's runtime management capabilities.

> **Why This Matters:** For teams where all changes must go through code review, treating prompts like code (in a Git repository with PR-based review) maintains compliance with engineering standards. The GitHub integration lets you have both: code-review discipline AND runtime prompt management.

---

### Link to Traces

Every generation that uses a Langfuse prompt records a reference to the specific prompt version that was used. This creates a direct link between Observability and Prompt Management:

- From any trace in the Langfuse UI, you can see which prompt version was used
- From any prompt version in the Prompt Management UI, you can see all traces that used it
- You can compare quality metrics across different prompt versions that have been used in production

---

### Playground

The **Langfuse Playground** is a browser-based interface for testing prompts interactively. You can:
- Load any prompt version
- Fill in variable values
- Select a model and configure parameters
- Run the prompt and see the output
- Edit the prompt and immediately see how the output changes

The playground supports all major model providers (OpenAI, Anthropic, Google, AWS Bedrock) once you configure your API keys. It requires no code — making it accessible to non-developer team members like product managers, writers, and domain experts.

---

### MCP Server

Langfuse exposes prompts via a **Model Context Protocol (MCP) server**, allowing AI code editors (like Cursor, Claude Code, or GitHub Copilot) to directly access and use your organization's prompts when helping developers write code.

---

### Webhooks and Slack Notifications

You can configure **webhooks** to notify external systems when prompts are updated. A popular configuration is to send a Slack message whenever a prompt label changes — for example, notifying the team when a new version is promoted to production.

---

### Guaranteed Availability

Langfuse's SDK design ensures that prompt fetching failures never break your application:
1. **In-memory cache:** If Langfuse is temporarily unreachable, the SDK uses the last-cached version
2. **Compile-time fallbacks:** You can configure a hardcoded fallback prompt that the SDK uses if no cached version is available

This means your application continues to function even during Langfuse outages or network issues.

---

## SDK Usage Patterns

### Basic Fetch and Compile

```python
from langfuse import Langfuse

langfuse = Langfuse()

# Fetch the production version of a prompt
prompt = langfuse.get_prompt("my-support-prompt", label="production")

# Compile with variable values
compiled_messages = prompt.compile(
    company_name="Acme Corp",
    user_tier="premium"
)

# Use with your LLM client
response = openai_client.chat.completions.create(
    model=prompt.config["model"],       # Model from prompt config
    temperature=prompt.config["temperature"],  # Temperature from config
    messages=compiled_messages
)
```

### Fetch for LangChain

```python
from langfuse import Langfuse

langfuse = Langfuse()

# Fetch and convert to LangChain format
prompt = langfuse.get_prompt("my-prompt", label="production")
langchain_template = prompt.get_langchain_prompt()

# Use directly in a LangChain chain
chain = langchain_template | llm | output_parser
result = chain.invoke({"variable": "value"})
```

### Link Prompt to Trace

```python
# When creating a generation, link the prompt object
# This records which prompt version was used
generation = langfuse_context.update_current_observation(
    prompt=prompt  # The Langfuse prompt object
)
```

### Create a New Version via SDK

```python
# Programmatically create a new prompt version (useful in CI/CD pipelines)
langfuse.create_prompt(
    name="my-prompt",
    type="chat",
    prompt=[
        {"role": "system", "content": "You are a helpful assistant for {{company}}."},
        {"role": "user", "content": "{{user_message}}"}
    ],
    labels=["staging"],  # Immediately label it as staging
    config={"model": "gpt-4o", "temperature": 0.5}
)
```

---

# Part 4: Evaluation Deep Dive

## What Evaluation Solves

Building an LLM application is easy. Knowing whether it's actually good is hard.

The challenge is that LLM outputs are complex artifacts that require judgment to assess. Is this response helpful? Is it accurate? Does it follow the brand's tone of voice? Did it hallucinate a fact? These questions can't be answered with simple code checks.

Langfuse Evaluation provides a suite of tools for systematically measuring LLM application quality across multiple dimensions:

- **Manual evaluation:** Human annotators review outputs and assign scores
- **Automated evaluation:** LLMs judge other LLMs' outputs (LLM-as-a-judge)
- **User-driven evaluation:** End users signal quality through thumbs up/down
- **Dataset-based evaluation:** Run test suites against curated inputs and measure results
- **Experiment comparison:** Compare quality across prompt versions, model versions, or code changes

---

## Core Concepts: Scores

### What Is a Score?

A **Score** is the fundamental primitive of evaluation in Langfuse. A score represents one measurement of quality or any other property of a trace or observation.

Every score has:
- **Trace ID or Observation ID:** What is being evaluated
- **Name:** What property is being measured (e.g., `"accuracy"`, `"helpfulness"`, `"toxicity"`)
- **Value:** The measurement
- **Data Type:** Numeric, boolean, or categorical
- **Source:** Where the score came from (API/SDK, UI annotation, or LLM-as-a-judge)
- **Comment (optional):** Explanation or reasoning for the score

### Score Data Types

**Numeric scores** are floating-point numbers on a range you define. Convention is 0.0 to 1.0, where higher is better, but any range works. Examples:
- `accuracy: 0.85`
- `relevance: 0.92`
- `latency_score: 0.4` (low because response was slow)

**Boolean scores** are pass/fail. Examples:
- `contains_hallucination: true`
- `follows_format: false`
- `safe_content: true`

**Categorical scores** are one of a defined set of labels. Examples:
- `quality: "good" | "acceptable" | "poor"`
- `sentiment: "positive" | "neutral" | "negative"`
- `intent_matched: "yes" | "no" | "partial"`

### Score Configs

**Score Configs** define the schema for a type of score — the name, data type, valid range or categories, and description. They ensure consistency when multiple people or automated systems are all scoring the same dimension.

---

## Evaluation Methods

### Method 1: Scores via SDK

The most programmatic evaluation method — your application code directly submits scores to Langfuse:

```python
langfuse.score(
    trace_id="abc123",              # Which trace to score
    name="response_quality",        # What you're measuring
    value=0.85,                     # The score (0-1 numeric)
    comment="Response was accurate but slightly verbose"
)
```

You can also score at the observation level (scoring a specific generation or span within a trace):

```python
langfuse.score(
    trace_id="abc123",
    observation_id="gen-xyz789",    # Score a specific generation
    name="factual_accuracy",
    value=1.0
)
```

Common use cases for SDK-based scoring:
- Measuring a deterministic property (e.g., did the response contain required keywords?)
- Running a local evaluation library (e.g., ROUGE score for summarization quality)
- Capturing the result of a programmatic safety check
- Submitting scores from your own custom evaluation pipeline

> **Why This Matters:** Not all evaluation requires human judgment. If you can write code to measure a property (response length within bounds, required fields present in JSON output, no prohibited words), doing so programmatically is faster and cheaper than human review.

---

### Method 2: Scores via UI (Manual Annotation)

The Langfuse UI allows team members to manually score any trace or observation directly in the browser, without any code. An annotator:
1. Opens a trace in the Langfuse UI
2. Clicks "Add Score" or uses a configured score template
3. Selects a value (e.g., chooses "good" from a categorical selector, or enters a 0-10 number)
4. Optionally adds a comment explaining the rating
5. Saves the score

Manual annotation is particularly valuable for:
- Subjective qualities that are hard to automate (tone, creativity, helpfulness)
- Calibrating LLM-as-a-judge systems (comparing human and automated scores)
- Building training datasets for fine-tuning
- High-stakes use cases where human judgment is required

---

### Method 3: Annotation Queues

**Annotation Queues** provide a structured workflow for coordinating human evaluation across a team. Rather than having individual annotators randomly browse traces, queues:

1. **Define a scoring task:** Which traces to evaluate, which score dimensions to apply, and any instructions for annotators
2. **Populate the queue:** Manually add traces, or automatically add traces that match filters (e.g., all traces from the past 24 hours with user_feedback score < 0.5)
3. **Assign to reviewers:** Queue items can be assigned to specific team members or left for any available reviewer
4. **Track progress:** See how many items have been reviewed, who reviewed them, and what scores were given
5. **Analyze results:** Aggregate scores from the queue feed into Langfuse's analytics

> **Why This Matters:** Without structured annotation queues, human evaluation tends to be inconsistent — different team members evaluate different traces using different criteria. Queues create a reproducible, trackable process that produces comparable data across time.

---

### Method 4: LLM-as-a-Judge

**LLM-as-a-judge** is Langfuse's most powerful automated evaluation method. The idea: use a separate, capable LLM (the "judge") to evaluate the output of your primary LLM application.

#### How It Works

You create an **evaluator** in Langfuse by defining:
1. A judge LLM (e.g., GPT-4o as the judge)
2. A system prompt for the judge (the evaluation criteria and instructions)
3. Variables from the trace to inject into the judge prompt (e.g., `{{input}}`, `{{output}}`, `{{expected_output}}`)
4. The output format (what score the judge should return)
5. When to run the evaluator (on every new trace? On a filtered subset? On-demand?)

When triggered, Langfuse:
1. Fetches the trace's data
2. Populates the judge prompt with actual trace values
3. Calls the judge LLM
4. Parses the response to extract the score
5. Attaches the score to the original trace

#### Built-in Evaluator Templates

Langfuse ships with pre-configured evaluator templates for common evaluation dimensions:
- **Hallucination:** Does the response contain information not supported by the provided context?
- **Relevance:** Does the response address the user's actual question?
- **Toxicity:** Does the response contain harmful, offensive, or inappropriate content?
- **Helpfulness:** Is the response genuinely useful to the user?
- **Completeness:** Does the response fully address all aspects of the question?
- **Correctness:** Is the response factually accurate (when a reference answer is provided)?

These templates have been prompt-engineered to produce reliable, consistent scores and can be deployed with a few clicks.

#### Custom Evaluator Templates

For domain-specific evaluation, you can write your own judge prompts:

```
System: "You are evaluating a customer support response for a SaaS company.
         Score the response on how well it follows our support guidelines:
         - Should acknowledge the issue empathetically
         - Should provide a clear next step
         - Should not promise features that don't exist
         
         User message: {{input}}
         Support agent response: {{output}}
         
         Return a score from 0-10 and a brief explanation."
```

Langfuse parses the judge's response to extract the numeric score and comment.

#### Online vs. Offline Evaluation

- **Online evaluation:** Evaluators run automatically on live production traces as they come in. This gives you a continuous quality signal on production traffic.
- **Offline evaluation:** You run evaluators on a batch of existing traces — for example, re-evaluating all traces from last week after you've improved your evaluation prompt.

> **Why This Matters:** Human evaluation is expensive and slow — you might be able to review 100 traces per day with a small team. LLM-as-a-judge can evaluate thousands of traces per hour at a fraction of the cost of human reviewers. The tradeoff is that LLM judges have their own biases and limitations. Best practice is to use LLM-as-a-judge for high-volume screening and human annotation for ground-truth calibration.

---

### Method 5: User Feedback

As described in the Observability section, user feedback is captured via the SDK and stored as scores. User feedback scores can be:
- **Explicit:** Thumbs up/down, star ratings, satisfaction surveys
- **Implicit:** Whether the user accepted a suggestion, how long they engaged with a response

User feedback provides the most direct signal about real-world quality.

---

### Score Analytics

All scores — regardless of source — appear in Langfuse's analytics dashboards:

- **Score distributions:** What percentage of traces scored above threshold?
- **Score trends over time:** Is quality improving or declining?
- **Score by dimension:** Compare accuracy vs. helpfulness vs. relevance
- **Score by segment:** Compare scores for different user tiers, environments, or prompt versions
- **Score correlation:** Does a high accuracy score correlate with positive user feedback?

---

## The Experiments System

The Experiments system connects Prompt Management and Evaluation, enabling systematic comparison of different versions of your application.

### Datasets

A **dataset** is a curated collection of test cases. Each item in a dataset has:
- **Input:** The query, document, or data to process
- **Expected Output (optional):** The "gold standard" correct answer
- **Metadata:** Any additional context about the test case

Datasets can be created:
- **Manually:** Adding items one by one in the Langfuse UI
- **From traces:** Converting real production traces into test cases (click any trace and save it to a dataset)
- **Via SDK:** Uploading items programmatically

The "create from traces" feature is particularly powerful: it lets you build a representative test suite from actual user interactions rather than synthetic examples.

```python
# Create a dataset
langfuse.create_dataset(name="customer-support-test-cases")

# Add items
langfuse.create_dataset_item(
    dataset_name="customer-support-test-cases",
    input={"question": "How do I cancel my subscription?"},
    expected_output="To cancel your subscription, go to Settings > Billing > Cancel Plan."
)
```

### Dataset Runs (Experiments)

A **dataset run** (experiment) is the result of running a version of your application against all items in a dataset. You associate a dataset run with a specific prompt version, model version, or code configuration.

Running an experiment:
```python
from langfuse import Langfuse

langfuse = Langfuse()
dataset = langfuse.get_dataset("customer-support-test-cases")

# Run your application against each dataset item
for item in dataset.items:
    with item.observe(run_name="gpt-4o-v2-prompt") as trace:
        # Your application code here
        result = my_llm_app(item.input["question"])
        trace.update(output=result)
```

After running, all the outputs are stored in Langfuse linked to the dataset run. You can then:
1. Apply scoring (automated or manual) to rate each output
2. Compare this run against previous runs side by side
3. Identify which dataset items got worse (regressions) or better (improvements)

### Experiment Comparison

The **Experiment Compare view** in the Langfuse UI shows multiple dataset runs side by side. For each dataset item, you can see:
- The input
- The output from each run
- Scores for each output
- Which run performed better overall

**Baseline comparison** lets you designate one run as the baseline (typically your current production version) and compare all other runs against it. Items that regressed are highlighted, making it immediately obvious whether a change made things worse.

> **Why This Matters:** This is the core of the improvement loop. You have a hypothesis — "this new prompt will improve accuracy." Instead of deploying to production and hoping, you run the new prompt against a test dataset, compare scores to the baseline, and only promote to production if the scores actually improve. This is how software engineering discipline applies to LLM development.

### Data Model Summary

```
Dataset
├── Dataset Items (input, expected_output, metadata)
│   └── Dataset Run Items (one per run)
│       └── Trace (linked to the item's run)
│           └── Scores (evaluation results)
└── Dataset Runs (named experiments)
    ├── Run name (e.g., "gpt-4o-prompt-v3")
    ├── Description
    └── All Run Items with aggregated scores
```

---

# Part 5: Supporting Systems

## Metrics and Dashboards

### Built-in Dashboard

The Langfuse main dashboard provides an at-a-glance view of your application's health across three dimensions:

**Quality metrics:**
- Average evaluation scores over time
- Score distributions by dimension
- Percentage of traces flagged as errors

**Cost metrics:**
- Total cost by day/week/month
- Cost breakdown by model
- Cost per user or session
- Cost trends and anomalies

**Performance metrics:**
- Average latency by trace type
- P50/P75/P95/P99 latency percentiles
- Token usage trends
- Error rates

### Custom Dashboards

Beyond the built-in dashboard, Langfuse allows you to create custom dashboards tailored to your specific needs. You can:
- Add custom metric panels with filters
- Track any score dimension
- Segment metrics by tags, users, environments, or releases
- Set up views for different team functions (engineering, product, support)

### Metrics API

The **Metrics API** provides programmatic access to aggregated metrics. You can query for:
- Average score for a specific score name in a time window
- Total cost by model for the past 30 days
- Error rate by environment
- Any other aggregation over your trace data

This enables:
- Exporting metrics to external dashboards (Grafana, Datadog)
- Building custom alerts based on metric thresholds
- Automated reporting pipelines

> **Why This Matters:** Teams often want their LLM quality metrics alongside infrastructure metrics in a single dashboard. The Metrics API allows Langfuse data to feed into your existing monitoring stack rather than requiring everyone to check a separate tool.

---

## API and Data Platform

### REST API

Langfuse exposes a comprehensive REST API covering all platform functionality:
- Create and read traces, generations, spans, events
- Read and write scores
- Manage prompts (create versions, update labels, fetch)
- Manage datasets and run experiments
- Query metrics and aggregations
- Export trace data

The API is documented and follows standard REST conventions, making it accessible from any programming language or tool.

### Data Export

For data-intensive use cases — custom analysis, fine-tuning dataset creation, compliance archiving — Langfuse supports exporting raw data to blob storage:

- Export to AWS S3, Google Cloud Storage, or Azure Blob Storage
- Schedule regular exports (daily, weekly)
- Export in JSON or Parquet format
- Filter exports by time range, environment, tags, or other attributes

> **Why This Matters:** Your LLM application's interaction data is valuable. It can train fine-tuned models, inform product decisions, and serve as evidence for compliance audits. Blob storage exports ensure you own your data and can use it however you need.

### MCP Server

Langfuse exposes an **MCP (Model Context Protocol) server** that allows AI assistants and code editors to interact with your Langfuse data programmatically. An AI coding assistant can:
- Fetch prompts from your Langfuse project
- Query recent traces
- Add scores to traces

This enables AI-assisted workflows where your development tools can directly access and interact with your LLM observability data.

---

## Security and Guardrails

### Input/Output Guardrails

Langfuse's **guardrail system** integrates evaluation scores with application flow control. You can configure rules that:
- Block an LLM output from being shown to the user if it scores below a threshold on a safety check
- Flag a response for human review before delivery
- Route problematic inputs to a fallback handler

Example workflow:
1. Application generates a response
2. A fast safety evaluator runs and produces a `toxicity_score`
3. If `toxicity_score > 0.8`, Langfuse signals the application to block the response
4. Application shows the user a fallback message instead

This happens synchronously in the request path, providing real-time guardrails for production safety.

> **Why This Matters:** LLM outputs can be unpredictable and sometimes harmful. Guardrails provide a safety net that catches problematic outputs before they reach users, reducing reputational and legal risk.

---

## Administration

### RBAC (Role-Based Access Control)

Langfuse supports fine-grained **role-based access control**:
- **Admin:** Full access including billing, user management, and project settings
- **Member:** Can view and edit all project data (traces, prompts, scores)
- **Viewer:** Read-only access to traces and dashboards
- Custom roles (Enterprise): Define exactly what each role can see and do

This allows teams to give product managers read access to dashboards without giving them the ability to delete traces or change prompt production labels.

### SSO (Single Sign-On)

Enterprise deployments support SSO via:
- SAML 2.0 (compatible with Okta, Azure AD, Google Workspace, and most enterprise identity providers)
- OIDC (OpenID Connect)

SSO enables centralized authentication management — users log in with their company credentials, and IT can provision/deprovision access centrally.

### Audit Logs

Every action taken in Langfuse — creating a prompt version, changing a label, deleting a trace, updating scores — is recorded in an immutable **audit log** with:
- Who performed the action
- What action was performed
- When it happened
- What the before/after state was

Audit logs support compliance requirements (SOC 2, HIPAA, GDPR) and enable security investigations.

### Data Retention and Deletion

You can configure:
- **Retention policies:** Automatically delete traces older than N days
- **On-demand deletion:** Delete specific traces, users' data, or date ranges
- **GDPR-compliant deletion:** Remove all data associated with a specific user ID (right to erasure)

### Spend Alerts

Configure alerts when your Langfuse usage (observations count) or tracked LLM spend exceeds thresholds. Alerts can be sent via email or webhook to Slack.

### Ask AI (Natural Language Interface)

**Ask AI** is a feature that lets you query your trace data using natural language. Instead of building complex filters in the UI, you can type questions like:
- "Show me traces from last week where the user seemed frustrated"
- "Which prompt version had the highest accuracy scores in production?"
- "What were the most common failure modes in the customer support bot?"

Ask AI uses an LLM to interpret your question, generate the appropriate query against your trace data, and return results.

---

# Part 6: Integration Patterns

## LangChain and LangGraph Integration

LangChain is one of the most common frameworks for building LLM applications, and Langfuse provides deep, first-class integration.

### Basic LangChain Setup

```python
from langfuse.callback import CallbackHandler
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

# Create the Langfuse callback handler
langfuse_handler = CallbackHandler()

# Build your chain normally
prompt = ChatPromptTemplate.from_template("Answer this question: {question}")
llm = ChatOpenAI(model="gpt-4o")
chain = prompt | llm

# Pass the handler as a callback — all steps are automatically traced
result = chain.invoke(
    {"question": "What is the capital of France?"},
    config={"callbacks": [langfuse_handler]}
)
```

Every step in the chain — the prompt template formatting, the LLM call — becomes a nested observation in a single Langfuse trace. Token usage and cost are captured automatically.

### LangGraph Agents

For LangGraph-based agents, the callback handler traces the entire agent execution graph:

```python
from langfuse.callback import CallbackHandler
from langgraph.graph import StateGraph

langfuse_handler = CallbackHandler()

# Your LangGraph agent setup
app = StateGraph(state_schema).compile()

# Run with Langfuse tracing
result = app.invoke(
    {"messages": [("user", "Research the latest AI papers")]},
    config={"callbacks": [langfuse_handler]}
)
```

Langfuse renders the agent's execution as a directed graph in the UI, showing which nodes were visited and in what sequence, with timing and token usage for each node.

### Adding User Context

To associate LangChain traces with users and sessions:

```python
langfuse_handler = CallbackHandler(
    user_id="user-123",
    session_id="conversation-abc",
    trace_name="customer-support-agent",
    tags=["production", "customer-support"]
)
```

---

## OpenAI SDK Integration

The OpenAI SDK integration is the simplest possible integration — one import change:

```python
# Before: from openai import OpenAI
# After:
from langfuse.openai import OpenAI

client = OpenAI()  # Same API, all calls now traced

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": user_message}
    ]
)
```

Every call to `client.chat.completions.create()` is automatically logged as a generation in Langfuse, complete with the full prompt, response, model name, token usage, and calculated cost.

### Adding Trace Metadata to OpenAI Calls

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[...],
    # Langfuse metadata passed as extra kwargs
    name="answer-generation",       # Name for this generation
    user_id="user-123",             # Associated user
    session_id="conv-abc",          # Session grouping
    tags=["rag", "production"],     # Tags for filtering
    metadata={"query_type": "factual"}
)
```

---

## Custom Python Integration

For applications not using LangChain or OpenAI, use the `@observe` decorator pattern:

```python
from langfuse.decorators import observe, langfuse_context
import anthropic

client = anthropic.Anthropic()

@observe(name="retrieve-context")
def retrieve_documents(query: str) -> list[str]:
    """Fetch relevant documents from vector database."""
    # ... vector search logic ...
    return relevant_docs

@observe(name="generate-answer")
def generate_answer(question: str, context: list[str]) -> str:
    """Call Claude to generate an answer."""
    prompt = build_prompt(question, context)
    
    message = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )
    
    # Manually log the generation details if not using Langfuse's Anthropic wrapper
    langfuse_context.update_current_observation(
        input=prompt,
        output=message.content[0].text,
        model="claude-3-5-sonnet-20241022",
        usage={"input_tokens": message.usage.input_tokens,
               "output_tokens": message.usage.output_tokens}
    )
    
    return message.content[0].text

@observe(name="rag-pipeline")  # Root trace
def answer_user_question(question: str) -> str:
    langfuse_context.update_current_trace(
        user_id=current_user_id,
        session_id=current_session_id,
        tags=["rag"]
    )
    
    context = retrieve_documents(question)
    answer = generate_answer(question, context)
    return answer
```

The `@observe` decorator handles all the tracing hierarchy automatically — `answer_user_question` becomes the root trace, `retrieve_documents` and `generate_answer` become nested spans.

---

## Production Patterns

### Sampling at Scale

At high traffic volumes, log only a fraction of traces:

```python
import random
from langfuse.decorators import observe

@observe(name="api-handler")
def handle_request(request):
    # Only create detailed traces for 10% of requests
    if random.random() > 0.1:
        langfuse_context.update_current_trace(enabled=False)
    
    # Always log errors regardless of sampling
    try:
        return process(request)
    except Exception as e:
        langfuse_context.update_current_trace(
            enabled=True,  # Override sampling — always log errors
            level="ERROR",
            tags=["error"]
        )
        raise
```

### Masking PII in Production

```python
import re
from langfuse import Langfuse

def mask_sensitive_data(data):
    """Remove PII before sending to Langfuse."""
    text = str(data)
    # Mask email addresses
    text = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',
                  '[EMAIL]', text)
    # Mask phone numbers
    text = re.sub(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b', '[PHONE]', text)
    # Mask SSNs
    text = re.sub(r'\b\d{3}-\d{2}-\d{4}\b', '[SSN]', text)
    return text

langfuse = Langfuse(mask=mask_sensitive_data)
```

### Environment-Based Configuration

Use environment variables to configure Langfuse differently per deployment:

```python
import os
from langfuse import Langfuse

langfuse = Langfuse(
    public_key=os.environ["LANGFUSE_PUBLIC_KEY"],
    secret_key=os.environ["LANGFUSE_SECRET_KEY"],
    host=os.environ.get("LANGFUSE_HOST", "https://cloud.langfuse.com")
)

# Set the environment on all traces from this process
import langfuse
langfuse.configure(
    release=os.environ.get("APP_VERSION", "unknown"),
    environment=os.environ.get("APP_ENV", "production")
)
```

---

## Multi-Service Distributed Tracing

When an LLM application spans multiple services (e.g., an API gateway, a backend service, and a model service), you can create a unified trace across all of them by propagating the trace ID.

**Service A (API Gateway):**
```python
# Create the root trace
trace = langfuse.trace(
    name="user-request",
    user_id=user_id,
    input=request_data
)

# Pass the trace ID to Service B
response = requests.post(
    "http://backend-service/process",
    json={"data": request_data},
    headers={"X-Trace-Id": trace.id}  # Pass trace ID in header
)

trace.update(output=response.json())
```

**Service B (Backend Service):**
```python
# Receive the trace ID from Service A
trace_id = request.headers.get("X-Trace-Id")

# Add observations to the SAME trace
span = langfuse.span(
    trace_id=trace_id,        # Same trace as Service A
    name="backend-processing",
    input=processing_input
)

# ... do work ...

span.end(output=result)
```

In the Langfuse UI, both services' operations appear in a single unified trace, making the full request path visible across service boundaries.

---

## Prompt Management in CI/CD Pipelines

Integrating Langfuse Prompt Management into your CI/CD pipeline enables automated quality gates:

### Promote Prompt After Tests Pass

```yaml
# GitHub Actions workflow
- name: Run prompt evaluation
  run: python scripts/evaluate_prompt.py --prompt-name "support-bot" --label "staging"
  
- name: Promote to production if score threshold met
  if: ${{ steps.evaluate.outputs.score >= 0.85 }}
  run: python scripts/promote_prompt.py --prompt-name "support-bot" --from "staging" --to "production"
```

```python
# promote_prompt.py
from langfuse import Langfuse

langfuse = Langfuse()

# Get the current staging prompt version
staging_prompt = langfuse.get_prompt("support-bot", label="staging")

# Promote: assign "production" label to this version
langfuse.update_prompt(
    name="support-bot",
    version=staging_prompt.version,
    labels=["production"]
)
```

### Automated Regression Testing

```python
# evaluate_prompt.py — runs in CI/CD
from langfuse import Langfuse
import statistics

langfuse = Langfuse()

# Get the staging prompt
staging_prompt = langfuse.get_prompt("support-bot", label="staging")

# Get the test dataset
dataset = langfuse.get_dataset("support-bot-regression-tests")

scores = []
for item in dataset.items:
    with item.observe(run_name=f"ci-test-v{staging_prompt.version}") as trace:
        # Run the application with this prompt
        result = run_app_with_prompt(item.input, staging_prompt)
        trace.update(output=result)
        
        # Score with automated evaluator
        score = evaluate_response(result, item.expected_output)
        scores.append(score)
        
        langfuse.score(
            trace_id=trace.id,
            name="ci_accuracy",
            value=score
        )

avg_score = statistics.mean(scores)
print(f"Average accuracy: {avg_score:.3f}")

# Exit with error if quality threshold not met
if avg_score < 0.85:
    print("Quality threshold not met — blocking promotion")
    exit(1)
```

---

## Automated Evaluation Pipelines

For continuous production monitoring, set up automated evaluation that runs on new traces:

### Online LLM-as-a-Judge

Configure in the Langfuse UI:
1. Go to Evaluation → LLM-as-a-judge evaluators
2. Create a new evaluator with your judge prompt
3. Configure it to run automatically on traces matching certain criteria
4. Set the scoring schema

Langfuse handles the rest — new qualifying traces are automatically sent to the judge LLM, and scores are attached.

### Custom Offline Evaluation Pipeline

For custom evaluation logic, run a scheduled job:

```python
# evaluation_pipeline.py — run nightly via cron
from langfuse import Langfuse
from datetime import datetime, timedelta

langfuse = Langfuse()

# Fetch yesterday's traces
traces = langfuse.fetch_traces(
    from_timestamp=datetime.now() - timedelta(days=1),
    tags=["production"],
    limit=1000
)

for trace in traces.data:
    # Run your custom evaluation
    quality_score = my_custom_evaluator(
        input=trace.input,
        output=trace.output
    )
    
    # Submit the score
    langfuse.score(
        trace_id=trace.id,
        name="custom_quality",
        value=quality_score
    )

langfuse.flush()
print(f"Evaluated {len(traces.data)} traces")
```

---

## Putting It All Together: The Full Development Loop

Here's how all six parts of Langfuse work together in a production scenario:

**Step 1 — Initial Development**
- Developer writes a first prompt, stores it in Langfuse Prompt Management as v1
- Implements the application using the `@observe` decorator for tracing
- Runs a few manual tests; traces appear in Langfuse

**Step 2 — Build Evaluation Infrastructure**
- Creates a dataset from production traces that represent common queries
- Sets up an LLM-as-a-judge evaluator for accuracy and helpfulness
- Runs the dataset against the initial prompt; establishes a baseline score

**Step 3 — Iterate on Prompts**
- Product manager identifies issues from user feedback (low thumbs-up rate)
- Makes changes to the prompt in Langfuse UI (no code changes needed)
- New version created; labeled as "staging"
- CI/CD pipeline runs the dataset against the staging prompt
- Score improves; "production" label is moved to the new version

**Step 4 — Monitor Production**
- LLM-as-a-judge runs automatically on all production traces
- Dashboard shows quality, cost, and latency trends
- Environment-separated views show production vs. development traffic clearly

**Step 5 — Catch Regressions**
- A model provider updates a model and behavior changes slightly
- Automated evaluation detects a quality drop; score dips below threshold
- Alert fires; team investigates traces from the affected time window
- Team identifies the issue, adjusts the prompt, and validates with the dataset before promoting

**Step 6 — Scale and Optimize**
- Sampling is configured to log 20% of traces at high volume
- PII masking ensures customer data stays clean
- Custom dashboards track KPIs for different product lines
- Data exports feed the monthly quality report

This complete lifecycle — from initial development through continuous production improvement — is what Langfuse is designed to support end-to-end.

---

## Quick Reference: Key Terms

| Term | Definition |
|------|------------|
| **Trace** | A top-level record of one end-to-end request or interaction |
| **Observation** | A sub-unit within a trace (Generation, Span, or Event) |
| **Generation** | An observation recording a single LLM API call |
| **Span** | An observation recording a non-LLM operation with duration |
| **Event** | An observation marking a point-in-time occurrence |
| **Score** | A measurement of quality or other property, attached to a trace or observation |
| **Prompt** | A named, versioned template stored in Langfuse Prompt Management |
| **Label** | A named pointer to a specific prompt version (e.g., "production") |
| **Dataset** | A curated collection of test inputs and expected outputs |
| **Dataset Run** | The result of running an application against a dataset (an experiment) |
| **Session** | A group of traces belonging to one multi-turn conversation |
| **Evaluator** | A configured LLM-as-a-judge that automatically scores traces |
| **Annotation Queue** | A structured workflow for routing traces to human reviewers |
| **Release** | A tag marking which application version produced a trace |
| **Environment** | A label separating dev/staging/production traces |
| **Masking** | Transforming sensitive data before it's sent to Langfuse |
| **Sampling** | Logging only a fraction of traces to control volume and cost |

---

## Resources and Next Steps

- **Official Documentation:** [langfuse.com/docs](https://langfuse.com/docs)
- **GitHub Repository:** [github.com/langfuse/langfuse](https://github.com/langfuse/langfuse)
- **Python SDK Documentation:** [langfuse.com/docs/sdk/python](https://langfuse.com/docs/sdk/python)
- **Community:** [langfuse.com/discord](https://langfuse.com/discord)
- **Changelog:** [langfuse.com/changelog](https://langfuse.com/changelog)
- **Self-Hosting Guide:** [langfuse.com/docs/deployment/self-host](https://langfuse.com/docs/deployment/self-host)
- **Integrations Reference:** [langfuse.com/docs/integrations](https://langfuse.com/docs/integrations)

---

*This guide covers Langfuse as of April 2026. The platform evolves rapidly — check the official changelog for the most current feature set.*
