# Meta-Builder-v3: Langfuse Missed Opportunities — Comprehensive Gap Analysis

> **Document scope:** This file catalogs every Langfuse capability that meta-builder-v3 is not using, explains the impact of each gap, provides concrete code examples for closing each gap, and assigns a priority and effort rating. This is an actionable remediation roadmap, not just a feature checklist.

---

## Table of Contents

1. [Framing: The "Capture Without Consume" Problem](#1-framing-the-capture-without-consume-problem)
2. [Observability Missed Opportunities](#2-observability-missed-opportunities)
   - 2.1 [Agent Graphs — Pipeline Visualization](#21-agent-graphs--pipeline-visualization)
   - 2.2 [Sessions — Multi-Turn Grouping](#22-sessions--multi-turn-grouping)
   - 2.3 [Users — End-User Attribution](#23-users--end-user-attribution)
   - 2.4 [Environments — Dev/Staging/Prod Separation](#24-environments--devstaginprod-separation)
   - 2.5 [Releases and Versioning — Deployment Tracking](#25-releases-and-versioning--deployment-tracking)
   - 2.6 [Log Levels — Severity Classification](#26-log-levels--severity-classification)
   - 2.7 [Multi-Modality — Image and Audio Tracing](#27-multi-modality--image-and-audio-tracing)
   - 2.8 [Masking — Server-Side PII Redaction](#28-masking--server-side-pii-redaction)
   - 2.9 [Sampling — Per-Route Trace Rates](#29-sampling--per-route-trace-rates)
   - 2.10 [User Feedback — End-User Rating Collection](#210-user-feedback--end-user-rating-collection)
   - 2.11 [Comments and Corrections — Collaborative Annotation](#211-comments-and-corrections--collaborative-annotation)
   - 2.12 [MCP Tracing — Tool Server Monitoring](#212-mcp-tracing--tool-server-monitoring)
3. [Prompt Management Missed Opportunities](#3-prompt-management-missed-opportunities)
   - 3.1 [A/B Testing — Traffic Splitting Between Versions](#31-ab-testing--traffic-splitting-between-versions)
   - 3.2 [Config Extraction — Model Parameters in Prompt Versions](#32-config-extraction--model-parameters-in-prompt-versions)
   - 3.3 [Multiple Labels — Staging and Latest Environments](#33-multiple-labels--staging-and-latest-environments)
   - 3.4 [Composability — Prompt Includes and Partials](#34-composability--prompt-includes-and-partials)
   - 3.5 [GitHub Sync — Version Control Integration](#35-github-sync--version-control-integration)
   - 3.6 [Playground — Interactive Prompt Testing](#36-playground--interactive-prompt-testing)
   - 3.7 [Webhooks — Change Notifications](#37-webhooks--change-notifications)
   - 3.8 [Proper Caching — TTL-Based vs lru_cache](#38-proper-caching--ttl-based-vs-lru_cache)
   - 3.9 [Prompt-to-Trace Linkage — Closing the Attribution Loop](#39-prompt-to-trace-linkage--closing-the-attribution-loop)
4. [Evaluation Missed Opportunities (THE BIGGEST GAP)](#4-evaluation-missed-opportunities-the-biggest-gap)
   - 4.1 [Scores via SDK — Reporting EvaluatorAgent Results](#41-scores-via-sdk--reporting-evaluatoragent-results)
   - 4.2 [LLM-as-a-Judge — Langfuse Evaluator Templates](#42-llm-as-a-judge--langfuse-evaluator-templates)
   - 4.3 [Annotation Queues — Human Evaluation Workflows](#43-annotation-queues--human-evaluation-workflows)
   - 4.4 [Datasets — Curated Test Sets](#44-datasets--curated-test-sets)
   - 4.5 [Experiments — Langfuse Experiment Tracking](#45-experiments--langfuse-experiment-tracking)
   - 4.6 [Score Analytics — Quality Dashboards](#46-score-analytics--quality-dashboards)
   - 4.7 [Custom Dashboards — Operational Metrics](#47-custom-dashboards--operational-metrics)
5. [Platform Missed Opportunities](#5-platform-missed-opportunities)
   - 5.1 [Metrics API — Programmatic Analytics](#51-metrics-api--programmatic-analytics)
   - 5.2 [Spend Alerts — Cost Monitoring](#52-spend-alerts--cost-monitoring)
   - 5.3 [Data Export — Analytics Pipeline](#53-data-export--analytics-pipeline)
   - 5.4 [MCP Server — AI Editor Integration](#54-mcp-server--ai-editor-integration)
6. [Priority and Effort Matrix](#6-priority-and-effort-matrix)
7. [Implementation Roadmap](#7-implementation-roadmap)

---

## 1. Framing: The "Capture Without Consume" Problem

Meta-builder-v3's Langfuse integration has achieved Phase 1 of observability maturity: **comprehensive capture**. Every function is wrapped, every LangChain invocation is instrumented, every span has a name. The system reliably writes data to Langfuse.

It has not attempted Phase 2: **consume and act**. The data flows in and sits there. No automated evaluation reads it. No quality scores flow back. No experiment comparisons are run. No dashboards track trends. The system is like a factory that installs sensors everywhere but never wires them to a control room.

The missed opportunities in this document are not theoretical edge cases — they are the core value proposition of Langfuse. The observability primitives are the table stakes. Evaluation, experiments, and datasets are the reasons to pay for and operate a system like Langfuse in the first place.

**Quantitative summary of the gap:**

| Category | Features Available | Features Used | Usage % |
|----------|-------------------|---------------|---------|
| Observability | ~15 features | 4 features | 27% |
| Prompt Management | ~10 features | 3 features | 30% |
| Evaluation | ~7 features | 0 features | 0% |
| Platform/API | ~5 features | 1 feature | 20% |
| **Total** | **~37 features** | **~8 features** | **~22%** |

---

## 2. Observability Missed Opportunities

### 2.1 Agent Graphs — Pipeline Visualization

#### What Langfuse Offers

Langfuse v3 introduced native support for visualizing agent execution as directed graphs. When traces include graph-structured span relationships (using Langfuse's graph tracing API or LangGraph integration), the Langfuse UI renders a visual DAG (directed acyclic graph) showing exactly which nodes executed, their order, branch decisions, and timing.

```python
# Langfuse graph tracing API (v3)
from langfuse import observe
from langfuse.types import PromptClient

# With LangGraph, the graph structure is captured automatically
# via the @observe decorator on StateGraph nodes
@observe(name="conductor_route")
async def route_request(state: ConductorState) -> ConductorState:
    ...

@observe(name="planner_decompose")
async def decompose_task(state: ConductorState) -> ConductorState:
    ...
```

For LangGraph specifically, the Langfuse LangChain `CallbackHandler` captures the graph structure automatically when passed as a callback to a LangGraph `StateGraph.invoke()` call.

#### What Meta-Builder Does Instead

Meta-builder uses `@observe` decorators that create **flat span hierarchies** — each function creates a child span of whatever function called it. This produces a call-tree trace, not a graph trace. The Langfuse UI shows a nested span list, not a DAG visualization.

The Conductor pipeline is fundamentally a graph: request → router → planner → [parallel branches] → evaluator → remediator (conditional) → output. This structure is invisible in the current trace representation.

#### The Gap and Its Impact

- **Developers cannot visually understand pipeline routing decisions** from traces alone
- **Parallel branch execution** (when the Conductor runs multiple agents simultaneously) appears as overlapping spans with no visual distinction from sequential execution
- **Conditional branches** (remediator only runs on failure) look the same as unconditional ones
- **Graph bottlenecks** are not identifiable without manually computing span timings across the tree

#### How to Bridge It

If meta-builder uses LangGraph for its Conductor pipeline, graph visualization comes for free by passing the `CallbackHandler` to the graph invocation:

```python
# src/utils/tracing.py — update get_runnable_config
def get_runnable_config(channel: str, session_id: str = None, user_id: str = None) -> dict:
    handler = CallbackHandler()
    metadata = {"langfuse_tags": [channel]}
    if session_id:
        metadata["langfuse_session_id"] = session_id
    if user_id:
        metadata["langfuse_user_id"] = user_id
    return {
        "callbacks": [handler],
        "metadata": metadata,
        # For LangGraph: enable graph metadata capture
        "run_name": f"conductor_{channel}",
    }

# In graph_factory.py or wherever the graph is compiled:
graph = StateGraph(ConductorState)
graph.add_node("plan", plan_node)
graph.add_node("retrieve", retrieve_node)
graph.add_edge("plan", "retrieve")
# ... etc.
compiled = graph.compile()

# Invocation with tracing:
config = get_runnable_config(channel="web", session_id=session_id)
result = await compiled.ainvoke(initial_state, config=config)
```

If meta-builder uses a custom graph engine (not LangGraph), manual graph span creation:

```python
from langfuse import get_client

async def trace_graph_execution(graph_id: str, nodes_executed: list[str], edges: list[tuple]):
    client = get_client()
    # Use Langfuse's span metadata to encode graph structure
    # This won't produce a visual graph but preserves structure for analysis
    client.update_current_trace(
        metadata={
            "graph_id": graph_id,
            "execution_path": nodes_executed,
            "edges_traversed": [f"{a}->{b}" for a, b in edges],
        }
    )
```

#### Priority: P1 | Effort: Medium (2–4 days if using LangGraph, 1 week if custom)

---

### 2.2 Sessions — Multi-Turn Grouping

#### What Langfuse Offers

Langfuse's `session_id` field groups multiple traces from the same logical user session. In the Langfuse UI, sessions view shows all traces from a session in chronological order, enabling:
- Complete user journey replay
- Session-level quality metrics (success rate per session)
- Session duration and turn count analytics
- Identification of sessions that went wrong

```python
# Setting session_id properly
from langfuse import observe, get_client

@observe()
async def handle_request(request: Request) -> Response:
    session_id = request.headers.get("X-Session-ID") or request.cookies.get("session_id")
    get_client().update_current_trace(
        session_id=session_id,
        user_id=request.user.id,
    )
    ...
```

#### What Meta-Builder Does Instead

`session_id` is defined as an optional parameter in `get_runnable_config()` and `set_trace_attributes()` but is rarely passed. Most traces have no `session_id`, meaning every request appears as an isolated trace in Langfuse. Multi-turn Conductor pipelines — where a user iterates with follow-up requests — are not grouped.

#### The Gap and Its Impact

- **Cannot replay user journeys** — a sequence of follow-up requests is invisible as a sequence
- **Cannot measure session success rate** — did the user eventually get what they needed?
- **Cannot identify "stuck sessions"** — users who sent 5 requests and none worked
- **Per-session quality analysis is impossible** — quality metrics are only per-request
- **Regression debugging is harder** — when a user reports "your AI stopped working," you can't pull their session

#### How to Bridge It

```python
# src/api/routes/base.py (or middleware)
from src.utils.tracing import set_trace_attributes

async def request_lifecycle_middleware(request: Request, call_next):
    # Extract or generate session ID from request context
    session_id = (
        request.headers.get("X-Session-ID")
        or request.cookies.get("mb_session")
        or request.state.session_id  # Set by auth middleware
    )
    user_id = getattr(request.state, "user_id", None)
    
    response = await call_next(request)
    return response

# In every @observe-decorated route handler:
@observe(name="api_run_pipeline")
async def run_pipeline(request: PipelineRequest, req: Request):
    set_trace_attributes(
        session_id=req.state.session_id,  # Always set
        user_id=req.state.user_id,
        tags=["pipeline_rag", req.state.channel],
    )
    ...
```

The key change: `session_id` should be **mandatory**, not optional, for all API routes. Every request in a conversation thread should share the same session ID.

#### Priority: P0 | Effort: Low (1–2 days — infrastructure exists, just needs wiring)

---

### 2.3 Users — End-User Attribution

#### What Langfuse Offers

`user_id` in Langfuse enables:
- Per-user quality metrics
- Per-user cost tracking (which users are expensive to serve?)
- Per-user error rates
- Filtering all traces for a specific user when debugging complaints
- GDPR data deletion by user ID

```python
# Langfuse user-level metrics are automatic once user_id is set
get_client().update_current_trace(
    user_id="user_12345",  # Your system's user identifier
)
# Langfuse then aggregates per user: avg score, total cost, error rate, etc.
```

#### What Meta-Builder Does Instead

`user_id` is defined as optional in tracing utilities but is not systematically passed. The system has no per-user observability.

#### The Gap and Its Impact

- **Cannot answer "Is this user experiencing worse quality than others?"**
- **Cannot track per-user LLM costs** — some users may be dramatically more expensive
- **GDPR trace deletion** — if a user requests data deletion, there's no way to find all their traces in Langfuse
- **Cannot identify "power users" vs casual users** in quality analytics

#### How to Bridge It

This is a one-line change at the request entry point:

```python
# In request_processor.py's @observe-decorated entry function
@observe(name="request_processor_process")
async def process(self, request: ProcessRequest) -> ProcessResult:
    set_trace_attributes(
        user_id=request.user_id,      # Pass from request context
        session_id=request.session_id,
        tags=["pipeline_rag", request.channel],
        metadata={"request_type": request.type, "channel": request.channel},
    )
    ...
```

The main prerequisite is ensuring `user_id` is threaded through `ProcessRequest` from the API layer. If authentication middleware sets `request.state.user_id`, this flows naturally.

#### Priority: P0 | Effort: Low (1 day — requires auth context plumbing)

---

### 2.4 Environments — Dev/Staging/Prod Separation

#### What Langfuse Offers

Langfuse supports an `environment` tag (or conventional tag naming like `env:production`) that enables:
- Filtering traces by environment in the UI
- Separate quality baselines per environment
- Preventing dev/test traces from polluting production analytics

The canonical approach is to set an environment tag at trace initialization:

```python
# Langfuse environment separation via tags
import os

ENVIRONMENT = os.environ.get("APP_ENV", "development")  # "development", "staging", "production"

def set_trace_attributes(name=None, session_id=None, user_id=None, tags=None, metadata=None):
    env_tag = f"env:{ENVIRONMENT}"
    full_tags = [env_tag] + (tags or [])
    get_langfuse_client().update_current_trace(
        name=name,
        session_id=session_id,
        user_id=user_id,
        tags=full_tags,
        metadata=metadata,
    )
```

Langfuse also supports a dedicated `environment` field in SDK v3 for proper environment-level separation:

```python
# SDK v3: explicit environment field
os.environ["LANGFUSE_ENVIRONMENT"] = "production"  # Set at startup
# All traces automatically tagged with this environment
```

#### What Meta-Builder Does Instead

`config.yaml` has no `environment` field. All traces from all environments (local dev, CI, staging, production) appear in the same Langfuse project with no separation. A developer running tests locally will pollute production analytics.

#### The Gap and Its Impact

- **Dev noise in production analytics** — local development traces contaminate quality dashboards
- **Cannot set environment-specific quality baselines** — what's acceptable latency in dev vs. prod differs
- **CI test traces pollute evaluation data** — automated test runs create spurious traces
- **Misleading cost analytics** — dev/test LLM calls inflate the cost numbers

#### How to Bridge It

```python
# src/config.py — update configure_tracing()
def configure_tracing(config: dict = None) -> None:
    tracing = config.get("tracing", {}) if config else {}
    enabled = tracing.get("enabled", False)
    provider = tracing.get("provider", "langfuse")
    environment = tracing.get("environment", os.environ.get("APP_ENV", "development"))
    
    if enabled and provider == "langfuse":
        os.environ["LANGFUSE_TRACING_ENABLED"] = "true"
        os.environ["LANGFUSE_ENVIRONMENT"] = environment
    else:
        os.environ["LANGFUSE_TRACING_ENABLED"] = "false"
```

```yaml
# config.yaml
tracing:
  enabled: true
  provider: langfuse
  environment: production  # or: ${APP_ENV}
```

#### Priority: P1 | Effort: Low (2–4 hours)

---

### 2.5 Releases and Versioning — Deployment Tracking

#### What Langfuse Offers

Langfuse's `release` field (set via `LANGFUSE_RELEASE` env var or SDK) tags all traces with a deployment version identifier. This enables:
- Comparing quality metrics before and after a deployment
- Identifying which deployment introduced a regression
- Rolling average quality by release (Langfuse charts this automatically)

```python
# Set at deployment time — typically from CI/CD pipeline
os.environ["LANGFUSE_RELEASE"] = os.environ.get("GIT_SHA", "unknown")
# Or: "v2.3.1", "deploy-2024-01-15", etc.
```

With releases set, Langfuse's dashboard shows quality trends per release:
- "Release abc1234 had 12% higher error rate than def5678"
- "Latency increased 200ms from v2.2 to v2.3"

#### What Meta-Builder Does Instead

No `LANGFUSE_RELEASE` is set. All traces appear as a flat time series — there is no way to segment "what changed" around a deployment boundary. When quality degrades, there is no automatic correlation with which code version was deployed.

#### The Gap and Its Impact

- **Cannot detect deployment regressions automatically** in Langfuse
- **Quality trend charts are uninterpretable** without knowing when deployments happened
- **Post-deployment validation is manual** — developers must manually check traces after each deploy

#### How to Bridge It

```python
# src/config.py
def configure_tracing(config: dict = None) -> None:
    ...
    if enabled and provider == "langfuse":
        os.environ["LANGFUSE_TRACING_ENABLED"] = "true"
        # Set release from build metadata (injected by CI/CD)
        release = (
            os.environ.get("GIT_SHA")
            or os.environ.get("BUILD_VERSION")
            or os.environ.get("APP_VERSION")
            or "unknown"
        )
        os.environ["LANGFUSE_RELEASE"] = release
```

In CI/CD (e.g., GitHub Actions):
```yaml
# .github/workflows/deploy.yml
- name: Deploy
  env:
    GIT_SHA: ${{ github.sha }}
    APP_VERSION: ${{ github.ref_name }}
  run: ./deploy.sh
```

#### Priority: P1 | Effort: Low (2–4 hours)

---

### 2.6 Log Levels — Severity Classification

#### What Langfuse Offers

Langfuse observations support a `level` field with values `DEBUG`, `DEFAULT`, `WARNING`, and `ERROR`. The `ERROR` level is particularly important — observations marked `ERROR` are highlighted in the trace view and counted separately in quality analytics.

```python
from langfuse.decorators import langfuse_context

@observe(name="retrieval_gate_evaluate")
async def evaluate(self, query: str, candidates: list) -> GateResult:
    result = self._run_gate(query, candidates)
    
    if result.passed:
        langfuse_context.update_current_observation(
            level="DEFAULT",
            output={"gate": "passed", "score": result.score},
        )
    else:
        langfuse_context.update_current_observation(
            level="WARNING",  # Gate failure is worth noting, not an error
            output={"gate": "failed", "reason": result.reason},
            status_message=f"Gate failed: {result.reason}",
        )
    
    return result
```

#### What Meta-Builder Does Instead

No observations have explicit `level` set. All spans default to `DEFAULT`. The debugger endpoint reads observations with `level == "ERROR"` — but these levels are only set by the Langfuse SDK when an unhandled exception propagates through an `@observe`-decorated function. No gate failures, quality warnings, or soft errors are explicitly classified.

#### The Gap and Its Impact

- **Gate failures (GateFailed events) are invisible** in Langfuse quality analytics
- **Re-plan triggers (RePlanTriggered events) are not surfaced** as warnings
- **The debugger's `hard_failure` classification** never feeds back as `ERROR`-level observations
- **Tier fallbacks** (when the system downgrades to a simpler model) are not visible as `WARNING`

#### How to Bridge It

```python
# In every gate agent (retrieval_gate.py, decision_gate.py, grounding_gate.py):
from langfuse.decorators import langfuse_context

@observe(name="retrieval_gate_evaluate")
async def evaluate(self, context: GateContext) -> GateResult:
    result = await self._evaluate(context)
    
    level = "DEFAULT"
    if result.is_failure:
        level = "WARNING"
    if result.is_hard_failure:
        level = "ERROR"
    
    langfuse_context.update_current_observation(
        level=level,
        output=result.dict(),
        status_message=result.reason if level != "DEFAULT" else None,
    )
    return result
```

```python
# In planning_agent.py — mark re-plan triggers
@observe(name="planning_agent_replan")
async def replan(self, context: PlanContext, failure_reason: str) -> Plan:
    langfuse_context.update_current_observation(
        level="WARNING",
        status_message=f"Re-plan triggered: {failure_reason}",
    )
    return await self._create_plan(context)
```

#### Priority: P1 | Effort: Medium (3–5 days across all gate/agent files)

---

### 2.7 Multi-Modality — Image and Audio Tracing

#### What Langfuse Offers

Langfuse can capture image and audio inputs/outputs in traces, rendering them directly in the UI. For vision models, this means you can see exactly what image was sent to the model alongside the model's response — invaluable for debugging visual reasoning failures.

```python
from langfuse.decorators import langfuse_context

@observe(name="vision_agent_analyze")
async def analyze_image(self, image_url: str, prompt: str) -> VisionResult:
    # Langfuse can capture the image URL or base64 data
    langfuse_context.update_current_observation(
        input={
            "type": "vision",
            "image_url": image_url,
            "prompt": prompt,
        }
    )
    result = await self._call_vision_model(image_url, prompt)
    langfuse_context.update_current_observation(
        output=result.dict()
    )
    return result
```

When `image_url` is included in the input, Langfuse renders the image inline in the trace view.

#### What Meta-Builder Does Instead

`vision_agent.py` has 7 `@observe` decorators but captures no image data in observations. Traces from VisionAgent show spans with timing but no visual content. A failing vision analysis trace is opaque — you can see that the function ran and failed, but not what image it was analyzing.

#### The Gap and Its Impact

- **Vision agent debugging is blind** — you cannot see what image caused a failure
- **Image quality issues are invisible** — low-quality images that cause model failures cannot be reviewed
- **No visual regression testing** — cannot build a dataset of image → expected output for VisionAgent

#### How to Bridge It

```python
# vision_agent.py
from langfuse.decorators import langfuse_context

@observe(name="vision_agent_analyze_image")
async def analyze_image(self, image_input: ImageInput) -> VisionResult:
    # Capture image metadata in the observation
    langfuse_context.update_current_observation(
        input={
            "image_url": image_input.url,  # Langfuse renders this inline
            "image_type": image_input.media_type,
            "prompt": image_input.prompt,
            "model": self.config.vision_model,
        }
    )
    
    result = await self._invoke_vision_model(image_input)
    
    langfuse_context.update_current_observation(
        output={
            "analysis": result.analysis,
            "confidence": result.confidence,
            "entities_detected": result.entities,
        }
    )
    return result
```

#### Priority: P2 | Effort: Low (1–2 days for vision_agent.py)

---

### 2.8 Masking — Server-Side PII Redaction

#### What Langfuse Offers

Langfuse v3 provides a masking middleware that intercepts trace data before it is sent to the Langfuse server and applies custom redaction logic. This is server-side (client-side in the Python process but before network transmission), meaning PII never leaves the application.

```python
# Langfuse masking configuration
from langfuse import Langfuse
import re

def mask_pii(data: str) -> str:
    """Redact PII before sending to Langfuse."""
    # Email addresses
    data = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', '[EMAIL]', data)
    # Phone numbers
    data = re.sub(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b', '[PHONE]', data)
    # SSNs
    data = re.sub(r'\b\d{3}-\d{2}-\d{4}\b', '[SSN]', data)
    return data

# SDK v3 masking configuration
os.environ["LANGFUSE_MASK"] = "true"
# Or configure programmatically:
client = Langfuse(
    masking_function=mask_pii,
)
```

#### What Meta-Builder Does Instead

Meta-builder has `SafetyGuardrails` for PII detection, but this is applied to **pipeline inputs and outputs** (to prevent PII from reaching the LLM). There is no masking applied to the tracing data itself. Every trace captures the full text of prompts, completions, and intermediate results — all of which may contain PII if a user asked about personal information.

The result: user PII may be present in Langfuse's database, even if it was blocked from reaching the LLM by guardrails.

#### The Gap and Its Impact

- **GDPR/CCPA compliance risk** — PII in traces stored on Langfuse Cloud (or self-hosted server) may violate data minimization principles
- **Breach blast radius** — a Langfuse database breach could expose user PII from traces
- **Audit exposure** — traces visible to all team members with Langfuse access may contain sensitive user information

#### How to Bridge It

```python
# src/utils/tracing.py — add masking to client initialization
import re
from langfuse import Langfuse

def create_pii_masking_function():
    """Create a masking function consistent with SafetyGuardrails PII patterns."""
    patterns = [
        (r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', '[EMAIL_REDACTED]'),
        (r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b', '[PHONE_REDACTED]'),
        (r'\b\d{3}-\d{2}-\d{4}\b', '[SSN_REDACTED]'),
        (r'\b(?:\d{4}[-\s]?){3}\d{4}\b', '[CARD_REDACTED]'),
    ]
    
    def mask(input_text: str) -> str:
        if not isinstance(input_text, str):
            return input_text
        for pattern, replacement in patterns:
            input_text = re.sub(pattern, replacement, input_text)
        return input_text
    
    return mask

def get_langfuse_client() -> Langfuse:
    global _client
    if _client is None:
        masking_fn = create_pii_masking_function()
        _client = Langfuse(masking_function=masking_fn)
    return _client
```

The masking function should be derived from the same patterns used in `SafetyGuardrails` to ensure consistency — PII that the guardrails catch should also be masked in traces.

#### Priority: P0 | Effort: Low (1–2 days) — **Compliance issue, not just a feature gap**

---

### 2.9 Sampling — Per-Route Trace Rates

#### What Langfuse Offers

Langfuse SDK v3 supports sample rates via the `LANGFUSE_SAMPLE_RATE` environment variable (0.0–1.0). At scale, tracing every request is expensive (both in Langfuse costs and in trace storage). Sampling allows:
- Tracing 100% of errors and 10% of successes
- Higher sampling rates for new/risky routes
- Cost control for high-volume, low-value operations

```python
# Global sampling via environment variable
os.environ["LANGFUSE_SAMPLE_RATE"] = "0.1"  # Trace 10% of requests

# Per-trace sampling via SDK (trace-level decision)
from langfuse.decorators import langfuse_context

@observe()
async def high_volume_operation(self, data: dict):
    import random
    if random.random() > 0.05:  # Only trace 5% of this high-volume operation
        langfuse_context.update_current_observation(
            metadata={"sampled": True}
        )
    ...
```

#### What Meta-Builder Does Instead

Binary tracing: either `LANGFUSE_TRACING_ENABLED=true` (100% of everything) or `false` (nothing). There is no middle ground. In production at scale, this may create:
- High Langfuse Cloud costs (all requests traced)
- Trace volume that makes finding important traces harder (signal-to-noise)
- Performance overhead on every request

#### How to Bridge It

```python
# src/config.py
def configure_tracing(config: dict = None) -> None:
    tracing = config.get("tracing", {}) if config else {}
    enabled = tracing.get("enabled", False)
    sample_rate = tracing.get("sample_rate", 1.0)  # Default: trace everything
    
    if enabled:
        os.environ["LANGFUSE_TRACING_ENABLED"] = "true"
        os.environ["LANGFUSE_SAMPLE_RATE"] = str(sample_rate)

# config.yaml
# tracing:
#   enabled: true
#   sample_rate: 0.2        # Trace 20% of requests in production
#   error_sample_rate: 1.0  # Always trace errors (handled in code)
```

For errors specifically, always force-trace regardless of sample rate:

```python
# In request_processor.py — force trace on failure
@observe(name="request_processor_process")
async def process(self, request: ProcessRequest) -> ProcessResult:
    try:
        result = await self._run(request)
        return result
    except Exception as e:
        # Force this trace to be captured regardless of sample rate
        langfuse_context.update_current_observation(
            level="ERROR",
            status_message=str(e),
            metadata={"force_trace": True},
        )
        raise
```

#### Priority: P2 | Effort: Low (1 day)

---

### 2.10 User Feedback — End-User Rating Collection

#### What Langfuse Offers

Langfuse's Scores API allows collecting end-user feedback and attaching it to traces. Scores can be:
- **Thumbs up/down** (boolean: 1 or 0)
- **Star ratings** (1–5 numeric scale)
- **Freeform comments**

When a user rates an AI response, the score is attached to the trace that generated it, enabling:
- Quality measurement from the user's perspective (not just internal evaluators)
- Identification of traces that users found unhelpful
- Training data selection (highly-rated outputs)
- Direct measurement of user satisfaction trends

```python
# Backend endpoint for collecting user feedback
from langfuse import get_client

@router.post("/api/feedback")
async def submit_feedback(feedback: UserFeedback):
    client = get_client()
    client.score(
        trace_id=feedback.trace_id,   # The trace that generated the response
        name="user_satisfaction",
        value=feedback.rating,          # 1 (thumbs up) or 0 (thumbs down)
        comment=feedback.comment,       # Optional freeform feedback
        data_type="BOOLEAN",
    )
    return {"status": "ok"}
```

On the frontend, you capture the `trace_id` from the API response and attach it to the feedback UI:
```json
// API response includes trace_id
{
  "response": "...",
  "trace_id": "trace_abc123",
  "metadata": {"session_id": "sess_xyz"}
}
```

#### What Meta-Builder Does Instead

No user feedback mechanism exists. Quality is measured entirely by internal agents (EvaluatorAgent, VerificationAgent) without any human signal from end users.

#### The Gap and Its Impact

- **Ground truth mismatch** — EvaluatorAgent may score a response highly that users found unhelpful
- **No user satisfaction trend** — cannot tell if improvements to EvaluatorAgent criteria correlate with real user satisfaction
- **No feedback-driven training signal** — cannot identify which responses users considered high-quality for fine-tuning or prompt improvement

#### How to Bridge It

1. Add `trace_id` to all API responses (so frontend can collect it)
2. Add a feedback endpoint that scores the trace
3. Wire frontend feedback UI to the endpoint

```python
# src/api/routes/pipeline.py — return trace_id in response
@observe(name="api_run_pipeline")
async def run_pipeline(request: PipelineRequest):
    from langfuse.decorators import langfuse_context
    result = await processor.process(request)
    
    # Get the current trace ID to return to the client
    trace_id = langfuse_context.get_current_trace_id()
    
    return {
        "result": result,
        "trace_id": trace_id,  # Frontend uses this for feedback
    }

# src/api/routes/feedback.py — new endpoint
@router.post("/api/feedback")
async def submit_feedback(
    trace_id: str,
    rating: float,  # 0.0 or 1.0 for thumbs, 1-5 for stars
    comment: str = None,
):
    get_langfuse_client().score(
        trace_id=trace_id,
        name="user_satisfaction",
        value=rating,
        comment=comment,
        data_type="BOOLEAN" if rating in (0, 1) else "NUMERIC",
    )
    return {"status": "recorded"}
```

#### Priority: P1 | Effort: Medium (3–5 days including frontend)

---

### 2.11 Comments and Corrections — Collaborative Trace Annotation

#### What Langfuse Offers

Langfuse's Comments and Corrections features allow team members to annotate traces:
- **Comments:** Freeform notes on a trace (e.g., "This retrieval missed the key document")
- **Corrections:** Structured corrections to trace outputs (e.g., correct expected answer for a failed trace)

Both are accessible via API:
```python
# Add a comment to a trace (via Langfuse API)
client.api.comments.create(
    object_id=trace_id,
    object_type="TRACE",
    content="Retrieval failed — the source document was chunked incorrectly",
)

# Query traces with corrections for dataset creation
corrected_traces = client.api.trace.list(
    tags=["has_correction"]
)
```

#### What Meta-Builder Does Instead

No comment or correction workflow. Debugging happens by developers manually reviewing traces in the Langfuse UI without any way to record their findings in Langfuse itself.

#### The Gap and Its Impact

- **Debugging knowledge is ephemeral** — findings from trace review are in Slack/Notion, not the trace
- **No way to flag traces for follow-up** from within Langfuse
- **Corrections cannot feed into dataset creation** (see section 4.4)

#### How to Bridge It

Primarily a UI workflow change, but the debugger API in meta-builder could be extended:

```python
# src/api/routes/debugger.py — add annotation support
@router.post("/api/debugger/annotate")
async def annotate_trace(
    trace_id: str,
    comment: str,
    error_type: str = None,
):
    """Add a developer annotation to a trace in Langfuse."""
    langfuse = get_langfuse_client()
    langfuse.api.comments.create(
        object_id=trace_id,
        object_type="TRACE",
        content=f"[{error_type}] {comment}" if error_type else comment,
    )
    return {"status": "annotated"}
```

#### Priority: P3 | Effort: Low (1 day)

---

### 2.12 MCP Tracing — Tool Server Monitoring

#### What Langfuse Offers

Langfuse has MCP (Model Context Protocol) server support for tracing MCP tool calls. If meta-builder uses MCP servers (e.g., for web search, code execution, or external data access), Langfuse can capture these as typed tool-call observations with input/output logging.

```python
# MCP tool call tracing via Langfuse
from langfuse.decorators import langfuse_context

@observe(name="mcp_tool_call")
async def call_mcp_tool(tool_name: str, arguments: dict) -> dict:
    langfuse_context.update_current_observation(
        input={"tool": tool_name, "arguments": arguments},
        metadata={"mcp_server": self.server_url},
    )
    result = await self.mcp_client.call_tool(tool_name, arguments)
    langfuse_context.update_current_observation(
        output=result,
    )
    return result
```

#### What Meta-Builder Does Instead

`tool_acquisition.py` has 4 `@observe` references, but if it uses MCP protocol, the MCP-specific metadata (tool name, server, arguments) is likely not structured in observations.

#### Priority: P3 | Effort: Low — depends on whether MCP is actively used

---

## 3. Prompt Management Missed Opportunities

### 3.1 A/B Testing — Traffic Splitting Between Versions

#### What Langfuse Offers

Langfuse enables prompt A/B testing via labels. You can have multiple labeled versions of a prompt:
- `label: production` — the current live version
- `label: challenger` — the version being tested
- `label: v2-experiment` — custom labels for named experiments

By randomly assigning a percentage of requests to the challenger prompt and tracking scores per label, you can measure whether the new prompt performs better.

```python
# A/B test implementation using Langfuse labels
import random

class PromptManager:
    def get_template_ab(
        self, 
        name: str, 
        ab_ratio: float = 0.1,  # 10% to challenger
        fallback: str = None,
    ):
        """Fetch production or challenger prompt based on ab_ratio."""
        use_challenger = random.random() < ab_ratio
        label = "challenger" if use_challenger else "production"
        
        prompt = self.client.get_prompt(name, label=label, cache_ttl_seconds=300)
        
        # Record which variant was used in the trace
        from langfuse.decorators import langfuse_context
        langfuse_context.update_current_observation(
            metadata={
                "prompt_name": name,
                "prompt_label": label,
                "ab_variant": "challenger" if use_challenger else "control",
            }
        )
        
        if prompt:
            return self._to_langchain_template(prompt), label
        return PromptTemplate.from_template(fallback), "fallback"
```

Then, in Langfuse, filter traces by `ab_variant` metadata and compare average scores between control and challenger groups.

#### What Meta-Builder Does Instead

`PromptManager` always fetches the `"production"` label. There is no mechanism to test prompt variations in production. New prompts must be fully promoted to production before their real-world performance can be measured.

#### The Gap and Its Impact

- **Zero data-driven prompt iteration** — prompts are updated based on developer judgment, not measured performance
- **No safe way to test prompt changes** — any prompt change immediately affects 100% of production traffic
- **Cannot quantify prompt impact** — "did this new system prompt make responses better?" is unanswerable
- **Regression risk** — every prompt change is a potential regression with no gradual rollout

#### How to Bridge It

The `PromptManager` class needs a new method `get_template_ab()` as shown above. Callers that want A/B testing pass a `challenger_ratio` parameter. The trace metadata records which variant was served. EvaluatorAgent scores are then analyzed per variant in Langfuse (once scoring is implemented — see section 4.1).

#### Priority: P1 | Effort: Medium (2–3 days — requires scoring to be meaningful)

---

### 3.2 Config Extraction — Model Parameters in Prompt Versions

#### What Langfuse Offers

Every Langfuse prompt version has a `config` field — arbitrary JSON attached to that version. The canonical use is storing model parameters that should be used with that prompt:

```json
{
  "model": "gpt-4o",
  "temperature": 0.1,
  "max_tokens": 2000,
  "top_p": 0.95
}
```

When a prompt's config includes model parameters, you can update both the prompt template and its parameters in a single Langfuse deployment, without code changes.

```python
class PromptManager:
    def get_template_with_config(self, name: str, label: str = "production"):
        """Fetch prompt and its associated model config."""
        prompt = self.client.get_prompt(name, label=label, cache_ttl_seconds=300)
        if not prompt:
            return None, {}
        
        template = self._to_langchain_template(prompt)
        model_config = prompt.config or {}
        
        return template, {
            "model": model_config.get("model"),
            "temperature": model_config.get("temperature"),
            "max_tokens": model_config.get("max_tokens"),
        }
```

Then in `llm/factory.py`:
```python
template, config = prompt_manager.get_template_with_config("evaluator_critique")
llm = llm_factory.get_llm(
    model=config.get("model") or DEFAULT_MODEL,
    temperature=config.get("temperature", 0.7),
)
chain = template | llm
```

#### What Meta-Builder Does Instead

`PromptManager.get_template()` calls `lf_prompt.get_langchain_prompt()` and discards `lf_prompt.config`. Model selection happens in `llm/factory.py` using a separate configuration system. Prompt templates and model parameters are managed in different places, making it impossible to co-version them.

#### Impact

- **Prompt and model versions are decoupled** — you can't know which model was used with which prompt version
- **Cannot optimize model parameters per prompt** — a summarization prompt might need `temperature=0.1` while a creative prompt needs `0.8`, but this can't be expressed
- **Config changes require code deploys** — model parameter changes can't be made through Langfuse

#### Priority: P1 | Effort: Low (1–2 days)

---

### 3.3 Multiple Labels — Staging and Latest Environments

#### What Langfuse Offers

Langfuse supports arbitrary labels on prompt versions. Common conventions:
- `production` — live version serving real users
- `staging` — version being validated before production promotion
- `latest` — the most recently created version (may be unstable)
- `v1`, `v2` — version aliases for pinned deployments

```python
# Fetch staging prompt for pre-production testing
staging_template = prompt_manager.get_template("planning_system", label="staging")

# Fetch production prompt for live traffic
prod_template = prompt_manager.get_template("planning_system", label="production")
```

This enables a full promotion workflow: develop in Langfuse UI → test with `staging` label in pre-prod → promote to `production` label when validated.

#### What Meta-Builder Does Instead

`PromptManager` hardcodes `label="production"` as the default. There is no staging prompt pipeline, no pre-production validation, no label-based environment separation for prompts.

#### Impact

Prompt changes bypass the same QA process that code changes go through. A developer can push a new prompt to `production` in Langfuse and it immediately affects live traffic (modulo the `lru_cache` delay). There's no equivalent of a code review or staging deployment for prompts.

#### Priority: P1 | Effort: Low (< 1 day — mostly a process change, minor code change)

---

### 3.4 Composability — Prompt Includes and Partials

#### What Langfuse Offers

Langfuse supports prompt composition via includes: a prompt can reference other prompts as reusable components. This is similar to template inheritance:

```
# System prompt "base_system" in Langfuse:
You are {{agent_name}}, an AI assistant. {{base_safety_rules}}

# Prompt "planning_system" that includes base_system:
{{> base_system agent_name="PlanningAgent"}}

Your specific role is to decompose user requests into actionable steps.
```

Changes to `base_safety_rules` automatically propagate to all prompts that include it.

#### What Meta-Builder Does Instead

Each agent has its own self-contained system prompt. Common elements (safety instructions, output format requirements, persona guidelines) are likely duplicated across prompts. When a shared policy changes, every prompt must be updated individually in Langfuse.

#### Impact

- **Policy updates require touching every prompt** — when a safety instruction changes, it must be manually updated in N prompts
- **Drift between prompts** — over time, agents diverge in their implementation of shared policies
- **No DRY principle for prompts** — the same boilerplate is repeated everywhere

#### Priority: P2 | Effort: Medium (2–3 days to restructure prompts into composable components)

---

### 3.5 GitHub Sync — Version Control Integration

#### What Langfuse Offers

Langfuse Cloud supports bi-directional GitHub sync: prompts can be stored as files in a GitHub repository and synced to Langfuse automatically on push. This means:
- Prompts go through the same PR review process as code
- Prompt history is in Git history
- Prompt changes are correlated with code changes in the same commit

```yaml
# .langfuse/prompts/planning_system.yaml
name: planning_system
type: chat
labels:
  - production
messages:
  - role: system
    content: |
      You are a planning agent responsible for...
config:
  model: gpt-4o
  temperature: 0.1
```

#### What Meta-Builder Does Instead

Prompts live in Langfuse UI with no Git backing. Prompt history is in Langfuse's version history (not Git). Prompt changes are not part of the code review process.

#### Impact

- **Prompt changes are not code-reviewed** — any team member with Langfuse access can change production prompts without review
- **Prompt and code changes are not correlated** — can't do `git blame` on a prompt regression
- **No rollback via Git** — reverting a bad prompt requires using Langfuse UI, not `git revert`

#### Priority: P2 | Effort: Low (primarily a process/tooling setup task)

---

### 3.6 Playground — Interactive Prompt Testing

#### What Langfuse Offers

The Langfuse Playground allows testing prompts interactively against real LLM providers, using traces from production as test inputs. The workflow:
1. Find a failing trace in Langfuse
2. Click "Open in Playground"
3. Edit the prompt
4. Re-run against the same inputs
5. Compare outputs
6. Save the improved prompt as a new version

For this to work, traces must be linked to the prompts that generated them via the `langfuse_prompt` metadata key.

#### What Meta-Builder Does Instead

`get_prompt_metadata()` returns `{"langfuse_prompt": lf_prompt}` but it is unclear whether callers pass this metadata to LangChain invocations consistently. Without proper prompt-to-trace linkage, the "Open in Playground" button in Langfuse doesn't have the prompt context to pre-populate the playground.

#### Priority: P2 | Effort: Low (primarily ensuring get_prompt_metadata() return value is used)

---

### 3.7 Webhooks — Change Notifications

#### What Langfuse Offers

Langfuse can send webhook events when prompts are updated (e.g., when a new version is promoted to `production`). This enables:
- Triggering CI/CD pipeline to run prompt regression tests
- Notifying team members in Slack
- Invalidating prompt caches in running services

```python
# Webhook handler for prompt change notifications
@router.post("/webhooks/langfuse")
async def langfuse_webhook(event: LangfuseWebhookEvent):
    if event.type == "prompt.updated" and event.data.label == "production":
        # Invalidate the prompt cache in all running instances
        await cache_invalidation_service.invalidate(f"prompt:{event.data.name}")
        # Notify team
        await slack_client.send(f"Prompt '{event.data.name}' updated to production")
```

#### What Meta-Builder Does Instead

No webhook integration. Running processes never know when prompts are updated in Langfuse. The `lru_cache` only expires on process restart, so even with a webhook handler, the cache would need explicit invalidation.

#### Impact

Prompt updates have an unknown and potentially very long lag before taking effect in production — until the next service restart. This makes Langfuse prompt management nearly useless as a "deploy without code change" mechanism.

#### Priority: P1 | Effort: Medium (2–3 days — requires cache invalidation infrastructure)

---

### 3.8 Proper Caching — TTL-Based vs lru_cache

#### What Langfuse Offers

The Langfuse Python SDK's `get_prompt()` accepts `cache_ttl_seconds`:

```python
# SDK-native TTL caching
prompt = client.get_prompt("planning_system", label="production", cache_ttl_seconds=300)
# Caches for 5 minutes, then fetches fresh version
```

This is superior to `lru_cache` because:
- It has a TTL — prompts are refreshed periodically without service restart
- It handles cache invalidation correctly when prompts are updated
- It is thread-safe (the SDK manages the cache internally)
- It respects the SDK's own cache implementation, including future improvements

#### What Meta-Builder Does Instead

```python
@lru_cache(maxsize=100)
def get_langfuse_prompt(self, name, label="production", fallback=None):
    return self.client.get_prompt(name, label=label)
```

`lru_cache` with no TTL means prompts are fetched once per process lifetime. This effectively defeats the purpose of Langfuse prompt management.

#### How to Bridge It

```python
class PromptManager:
    def __init__(self, client=None, cache_ttl_seconds: int = 300):
        self.client = client or get_client()
        self.cache_ttl_seconds = cache_ttl_seconds
    
    def get_langfuse_prompt(self, name: str, label: str = "production", fallback: str = None):
        # Remove @lru_cache decorator — use SDK-native caching instead
        try:
            return self.client.get_prompt(
                name, 
                label=label, 
                cache_ttl_seconds=self.cache_ttl_seconds,
            )
        except Exception:
            return None
```

#### Priority: P0 | Effort: Low (< 1 day) — **This is a correctness issue, not just optimization**

---

### 3.9 Prompt-to-Trace Linkage — Closing the Attribution Loop

#### What Langfuse Offers

When `langfuse_prompt` is passed in LangChain metadata, Langfuse automatically links the trace to the specific prompt version that generated it. In the Langfuse UI:
- Each trace shows which prompt version was used
- Each prompt version shows which traces used it
- You can see quality metrics aggregated by prompt version

```python
# Correct usage: pass prompt metadata to the chain invocation
template, metadata = prompt_manager.get_template_with_metadata("evaluator_critique")
config = get_runnable_config(channel=channel)
config["metadata"].update(metadata)  # Includes {"langfuse_prompt": lf_prompt}

result = await chain.ainvoke(inputs, config=config)
```

#### What Meta-Builder Does Instead

`get_prompt_metadata()` is defined but it's unclear whether its return value is systematically merged into the `RunnableConfig` metadata. If callers only use `get_template()` without also getting and passing metadata, prompt versions are not linked to traces.

#### Impact

- **Cannot answer "which prompt version generated this trace?"**
- **Cannot aggregate quality by prompt version**
- **Cannot use "Open in Playground" for failing traces**

#### Priority: P1 | Effort: Low (1–2 days — audit and fix all PromptManager call sites)

---

## 4. Evaluation Missed Opportunities (THE BIGGEST GAP)

The evaluation gap is not merely one missed feature among many — it represents the complete absence of a quality feedback loop. Meta-builder-v3 has sophisticated internal evaluation machinery (EvaluatorAgent, VerificationAgent, CritiqueRefiner) that computes quality signals and then discards them. Langfuse's evaluation platform is purpose-built to store, aggregate, and analyze exactly these signals.

### 4.1 Scores via SDK — Reporting EvaluatorAgent Results

#### What Langfuse Offers

The Langfuse Scores API is the mechanism for attaching quality measurements to traces:

```python
from langfuse import get_client

client = get_client()

# Score a trace
client.score(
    trace_id=trace_id,
    name="overall_quality",       # Score name (defines the metric)
    value=0.87,                   # Numeric score (0.0–1.0 typical)
    comment="Strong factual accuracy, minor verbosity issue",
    data_type="NUMERIC",
)

# Score a specific observation (span) within a trace
client.score(
    trace_id=trace_id,
    observation_id=retrieval_span_id,
    name="retrieval_precision",
    value=0.92,
    data_type="NUMERIC",
)

# Boolean scores
client.score(
    trace_id=trace_id,
    name="factually_correct",
    value=1.0,  # 1.0 = true, 0.0 = false
    data_type="BOOLEAN",
)
```

Once scores are submitted, Langfuse automatically:
- Tracks score distributions over time
- Charts quality trends by date, model, prompt version, environment
- Enables filtering traces by score threshold ("show me all traces with quality < 0.5")
- Computes per-session and per-user quality aggregates

#### What Meta-Builder Does Instead

`evaluator_agent.py` has 3 `@observe` decorators but **zero calls to `client.score()`**. The EvaluatorAgent computes internal quality scores (relevance, factuality, completeness, coherence) and returns them in a result object. These scores are used for local decision-making (trigger re-plan? accept output?) but are never reported to Langfuse.

The result: Langfuse contains thousands of traces with zero score data. The UI quality charts are empty. There is no way to ask "what is our average factuality score this week?" or "did quality improve after we updated the planning prompt?"

#### How to Bridge It

```python
# evaluator_agent.py — wire scores to Langfuse after every evaluation
from langfuse.decorators import langfuse_context
from src.utils.tracing import get_langfuse_client

class EvaluatorAgent:
    @observe(name="evaluator_agent_evaluate")
    async def evaluate(
        self, 
        output: str, 
        criteria: EvalCriteria,
        trace_id: str = None,  # The trace to score
    ) -> EvalResult:
        result = await self._run_evaluation(output, criteria)
        
        # Report scores to Langfuse
        client = get_langfuse_client()
        
        # Get current trace ID if not provided
        if not trace_id:
            trace_id = langfuse_context.get_current_trace_id()
        
        if trace_id:
            # Report each sub-score
            for score_name, score_value in result.scores.items():
                client.score(
                    trace_id=trace_id,
                    name=score_name,
                    value=score_value,
                    data_type="NUMERIC",
                )
            
            # Report overall score
            client.score(
                trace_id=trace_id,
                name="overall_quality",
                value=result.overall_score,
                comment=result.summary,
                data_type="NUMERIC",
            )
        
        return result
```

Similarly for `verification_agent.py`:
```python
class VerificationAgent:
    @observe(name="verification_agent_verify")
    async def verify(self, output: str, trace_id: str = None) -> VerificationResult:
        result = await self._verify(output)
        
        client = get_langfuse_client()
        trace_id = trace_id or langfuse_context.get_current_trace_id()
        
        if trace_id:
            client.score(
                trace_id=trace_id,
                name="verification_passed",
                value=1.0 if result.passed else 0.0,
                comment=result.failure_reason,
                data_type="BOOLEAN",
            )
        
        return result
```

#### Priority: P0 | Effort: Medium (3–5 days) — **This is the highest-impact single change**

---

### 4.2 LLM-as-a-Judge — Langfuse Evaluator Templates

#### What Langfuse Offers

Langfuse has a built-in LLM-as-a-Judge system with reusable evaluator templates. You define an evaluator once (e.g., "factuality checker using GPT-4o"), and Langfuse automatically runs it against new traces as they come in.

```python
# Langfuse evaluator templates are defined in the UI, then auto-run on new traces.
# They can also be triggered programmatically:

from langfuse import get_client

client = get_client()

# Create an LLM evaluator
client.api.evaluators.create(
    name="factuality_checker",
    type="llm",
    model="gpt-4o",
    prompt="Evaluate the factual accuracy of the following AI response...\n\nInput: {{input}}\nOutput: {{output}}\n\nScore from 0 to 1:",
    score_name="factuality",
    enabled=True,
)
```

Benefits over meta-builder's custom `EvaluatorAgent`:
- Runs asynchronously after the trace is created (no latency impact on user requests)
- Reusable across projects and environments
- Managed from the Langfuse UI without code changes
- Results are automatically stored as scores and tracked over time
- Can be applied retroactively to historical traces

#### What Meta-Builder Does Instead

`EvaluatorAgent` is a custom Python class that runs **synchronously** within the pipeline, adding latency to every request. Its scores are discarded after use. If you want to add a new evaluation criterion, you must change the code and redeploy.

#### The Gap and Its Impact

- **Evaluation adds latency** — running EvaluatorAgent synchronously in the request path adds 1–5 seconds to every response
- **Evaluation criteria can't be changed without code deploy** — adding a new criterion requires modifying EvaluatorAgent
- **Historical traces can't be re-evaluated** — if you add a new criterion today, you can't retroactively evaluate last month's traces

#### How to Bridge It

Move evaluation from synchronous (in request path) to asynchronous (post-trace):

```python
# Short term: Keep EvaluatorAgent for real-time gate decisions, 
# but also report scores to Langfuse for analytics.

# Medium term: Configure Langfuse evaluators for non-blocking background evaluation.
# This removes EvaluatorAgent from the hot path.

# Long term: Migrate to Langfuse evaluator templates entirely.
# EvaluatorAgent becomes a thin wrapper that calls Langfuse's evaluator API.
```

#### Priority: P1 | Effort: High (1–2 weeks for full migration)

---

### 4.3 Annotation Queues — Human Evaluation Workflows

#### What Langfuse Offers

Langfuse annotation queues are a structured human labeling workflow:
1. Define a queue (e.g., "evaluator failures requiring human review")
2. Configure queue routing rules (e.g., "enqueue all traces with `overall_quality < 0.5`")
3. Annotators open the queue in Langfuse UI, review traces, and add scores or corrections
4. Annotations are stored as scores (type: `ANNOTATION`) on traces

```python
# Programmatically add a trace to an annotation queue
from langfuse import get_client

client = get_client()

# If quality is low, queue for human review
if result.quality_score < 0.5:
    client.api.annotation_queues.add_to_queue(
        queue_id="low_quality_review",
        trace_id=trace_id,
        comment=f"Quality score: {result.quality_score}. Reason: {result.failure_reason}",
    )
```

#### What Meta-Builder Does Instead

`VerificationAgent` and `EvaluatorAgent` are fully automated. There is no human-in-the-loop quality review. Traces that fail automated evaluation are retried by `RemediationAgent` or returned with a failure — but a human never reviews the output.

#### The Gap and Its Impact

- **No ground truth generation** — human annotations would create ground truth for training EvaluatorAgent
- **No quality floor enforcement** — some failures require human judgment that automated evaluation can't provide
- **No data labeling for fine-tuning** — human-annotated traces are the highest-quality training signal

#### How to Bridge It

1. Define annotation queues in Langfuse UI for different failure categories
2. Add a queue dispatch step after EvaluatorAgent for low-scoring outputs
3. Create a lightweight annotation UI (or use Langfuse's built-in UI directly)

```python
# In the pipeline's post-evaluation step:
@observe(name="pipeline_dispatch_annotation")
async def maybe_queue_for_annotation(result: PipelineResult, trace_id: str):
    if result.quality_score < HUMAN_REVIEW_THRESHOLD:
        client = get_langfuse_client()
        client.api.annotation_queues.add_to_queue(
            queue_id=ANNOTATION_QUEUE_ID,
            trace_id=trace_id,
            comment=f"Automated score: {result.quality_score:.2f}",
        )
```

#### Priority: P2 | Effort: Medium (1 week)

---

### 4.4 Datasets — Curated Test Sets

#### What Langfuse Offers

Langfuse Datasets are curated collections of input/output pairs used for regression testing and benchmarking. A dataset item consists of:
- `input` — the query/context that was given to the system
- `expected_output` — the expected correct response
- `metadata` — arbitrary additional context

Datasets can be populated from:
- Production traces (using `create_dataset_item_from_trace`)
- Manually curated examples
- Human annotations

```python
from langfuse import get_client

client = get_client()

# Create a dataset
dataset = client.create_dataset(
    name="pipeline_rag_regression",
    description="Regression test cases for RAG pipeline quality",
)

# Add an item from a production trace
client.create_dataset_item(
    dataset_name="pipeline_rag_regression",
    input={"query": "What is the capital of France?", "context": "..."},
    expected_output={"answer": "Paris", "sources": ["wikipedia_france"]},
    metadata={"source_trace_id": trace_id, "quality_score": 0.95},
)
```

#### What Meta-Builder Does Instead

`AgentGym` uses `SyntheticDataGenerator` to create test data algorithmically. Synthetic data has limitations:
- May not represent the true distribution of production inputs
- Cannot capture the "long tail" of unusual queries that cause failures
- Has no expected outputs based on human judgment

Langfuse datasets complement synthetic data by capturing real production cases, including failures.

#### The Gap and Its Impact

- **Cannot run regression tests against production input distribution** — synthetic data biases toward expected inputs
- **Cannot build a gold-standard evaluation set** — no mechanism to capture and annotate exemplary traces
- **Cannot measure dataset-level quality** — individual trace scores don't accumulate into benchmark numbers
- **Cannot track progress on hard cases** — specific challenging inputs aren't tracked over iterations

#### How to Bridge It

```python
# src/utils/dataset_management.py — new file
from langfuse import get_client
from src.utils.tracing import get_langfuse_client

class DatasetManager:
    def __init__(self):
        self.client = get_langfuse_client()
    
    def promote_trace_to_dataset(
        self,
        trace_id: str,
        dataset_name: str,
        expected_output: dict = None,
        human_score: float = None,
    ):
        """Promote a production trace to a dataset item for regression testing."""
        trace = self.client.api.trace.get(trace_id)
        
        self.client.create_dataset_item(
            dataset_name=dataset_name,
            input=trace.input,
            expected_output=expected_output or trace.output,
            metadata={
                "source_trace_id": trace_id,
                "human_score": human_score,
                "created_at": datetime.utcnow().isoformat(),
            },
        )
    
    def run_dataset_evaluation(self, dataset_name: str, pipeline_fn) -> DatasetRunResult:
        """Run the current pipeline against all items in a dataset."""
        dataset = self.client.get_dataset(dataset_name)
        results = []
        
        for item in dataset.items:
            with item.observe(run_name=f"eval_{datetime.utcnow().isoformat()}") as root_span:
                output = pipeline_fn(item.input)
                root_span.score(
                    name="correctness",
                    value=self._compare(output, item.expected_output),
                )
                results.append(output)
        
        return DatasetRunResult(results)
```

#### Priority: P1 | Effort: High (1–2 weeks)

---

### 4.5 Experiments — Langfuse Experiment Tracking

#### What Langfuse Offers

Langfuse Experiments allow running your pipeline against a dataset multiple times with different configurations (prompt versions, models, parameters) and comparing the results:

```python
# Run an experiment comparing two prompt versions
from langfuse import get_client

client = get_client()

dataset = client.get_dataset("pipeline_rag_regression")

# Run 1: Current production configuration
for item in dataset.items:
    with item.observe(run_name="run_v1_gpt4o") as trace:
        result = pipeline.run(item.input, model="gpt-4o", prompt_label="production")
        trace.score(name="quality", value=evaluate(result, item.expected_output))

# Run 2: New configuration (challenger prompt + gpt-4o-mini for speed)
for item in dataset.items:
    with item.observe(run_name="run_v2_challenger") as trace:
        result = pipeline.run(item.input, model="gpt-4o-mini", prompt_label="challenger")
        trace.score(name="quality", value=evaluate(result, item.expected_output))

# Langfuse UI shows side-by-side comparison of run_v1_gpt4o vs run_v2_challenger
# with statistical significance, per-item comparisons, and aggregate metrics
```

#### What Meta-Builder Does Instead

`AgentGym` runs evolutionary optimization experiments and tracks them in **SQLite** via `ExperimentTracker`. The optimizer mutates agents, runs them against test cases, and records fitness scores. This is sophisticated and appropriate for the evolutionary optimization use case — but it is entirely disconnected from Langfuse, meaning:

- Experiment results are not correlated with production trace quality
- The same evaluation infrastructure can't be used for both A/B testing and evolutionary optimization
- Historical experiment results can't be analyzed with Langfuse's comparison tools
- Langfuse datasets can't be used as the test set for evolutionary optimization

#### How to Bridge It

The Langfuse experiment framework and the SQLite `ExperimentTracker` serve different but overlapping purposes. The migration path:

1. **Immediate:** Have `ExperimentTracker` also push results to Langfuse scores
2. **Medium term:** Use Langfuse datasets as the test set for AgentGym evaluations
3. **Long term:** Run Langfuse experiment runs from within AgentGym's optimization loop

```python
# gym.py — report experiment results to Langfuse
class AgentGym:
    @observe(name="gym_run_experiment")
    async def run_experiment(self, config: ExperimentConfig) -> ExperimentResult:
        with propagate_attributes(tags=["experiment"]):
            result = await self._evolve(config)
        
        # Report experiment results to Langfuse
        client = get_langfuse_client()
        for trial in result.trials:
            client.score(
                trace_id=trial.trace_id,
                name="experiment_fitness",
                value=trial.fitness_score,
                comment=f"Experiment: {config.name}, Generation: {trial.generation}",
                metadata={
                    "experiment_name": config.name,
                    "generation": trial.generation,
                    "config_hash": trial.config_hash,
                },
            )
        
        return result
```

#### Priority: P1 | Effort: High (1–2 weeks for Langfuse integration; evolutionary optimizer stays in SQLite for now)

---

### 4.6 Score Analytics — Quality Dashboards

#### What Langfuse Offers

Once scores are submitted (section 4.1), Langfuse automatically provides:
- **Time-series charts** of score distributions
- **Score by model** — which LLM produces the highest quality outputs?
- **Score by prompt version** — which prompt version performs best?
- **Score by user** — which users receive lower quality responses?
- **Score by tag** — quality differences between channels/routes
- **Latency vs. quality trade-off** charts

These analytics require zero additional configuration once scores are flowing.

#### What Meta-Builder Does Instead

Empty Langfuse dashboards. No score data means no quality analytics. The only way to answer quality questions is to manually inspect individual traces.

#### Impact

This is downstream of the scoring gap (section 4.1). Fixing scoring automatically enables score analytics.

#### Priority: P0 | Effort: Zero additional effort (automatic once scores flow)

---

### 4.7 Custom Dashboards — Operational Metrics

#### What Langfuse Offers

Langfuse custom dashboards allow creating charts from trace data using SQL-like query builders:
- "Number of traces by tag over time"
- "P95 latency by model"
- "Error rate by route"
- "Cost per request by user"
- "Traces per session distribution"

These dashboards can be shared across the team and embedded in internal tools.

#### What Meta-Builder Does Instead

No custom Langfuse dashboards. Operational metrics are presumably tracked via separate monitoring infrastructure (Prometheus/Grafana or similar) if they exist at all.

#### How to Bridge It

This is a UI-only change in Langfuse. No code changes required. Start with:
1. "Error rate over time by route" — using existing trace data
2. "Latency P50/P95/P99 by model" — using existing trace data
3. "Quality score trend" — requires scoring (section 4.1) first
4. "Cost per session" — using LangChain token usage data in traces

#### Priority: P2 | Effort: Low (2–3 hours in Langfuse UI)

---

## 5. Platform Missed Opportunities

### 5.1 Metrics API — Programmatic Analytics

#### What Langfuse Offers

Langfuse's Metrics API enables querying aggregated metrics programmatically:

```python
# Query average score over a time range
from langfuse import get_client

client = get_client()

metrics = client.api.metrics.daily(
    from_timestamp=datetime(2024, 1, 1),
    to_timestamp=datetime(2024, 1, 31),
    trace_name="request_processor_process",
    tags=["pipeline_rag"],
)

for day in metrics.data:
    print(f"{day.date}: {day.count_traces} traces, avg latency: {day.latency_p50}ms")
```

This enables building internal analytics dashboards, automated quality reports, and alerting systems.

#### What Meta-Builder Does Instead

The debugger endpoint reads individual traces but never queries aggregate metrics. Operational dashboards (if any) don't pull from Langfuse.

#### How to Bridge It

```python
# src/api/routes/analytics.py — new internal analytics endpoint
@router.get("/api/analytics/quality-summary")
async def quality_summary(days: int = 7):
    """Return quality metrics for the last N days."""
    client = get_langfuse_client()
    
    metrics = client.api.metrics.daily(
        from_timestamp=datetime.utcnow() - timedelta(days=days),
        to_timestamp=datetime.utcnow(),
    )
    
    return {
        "period_days": days,
        "total_traces": sum(d.count_traces for d in metrics.data),
        "avg_latency_ms": sum(d.latency_p50 or 0 for d in metrics.data) / len(metrics.data),
        "daily_breakdown": [d.dict() for d in metrics.data],
    }
```

#### Priority: P2 | Effort: Low (1–2 days)

---

### 5.2 Spend Alerts — Cost Monitoring

#### What Langfuse Offers

Langfuse tracks LLM token costs (using model pricing tables) and can alert when costs exceed thresholds. The LangChain `CallbackHandler` automatically captures token usage from OpenAI, Anthropic, and other providers, and Langfuse calculates cost based on the model's pricing.

Alerts can be configured for:
- Daily cost threshold
- Per-user cost threshold
- Per-session cost threshold
- Cost spike detection (e.g., 2× daily average)

#### What Meta-Builder Does Instead

Meta-builder uses multiple LLM providers (OpenAI, Anthropic, etc.) via `llm/factory.py`. Token usage is captured by the LangChain `CallbackHandler` and sent to Langfuse. Langfuse is already computing cost data — but no spend alerts are configured.

#### The Gap and Its Impact

- **No cost visibility in production** — don't know which agents or routes are most expensive
- **No runaway cost protection** — a bug causing excessive retries or very long documents could generate large unexpected bills without alerting
- **Cannot optimize for cost** — without per-route cost data, can't target expensive operations for optimization

#### How to Bridge It

This is primarily a configuration change in the Langfuse UI (Settings → Alerts). No code changes required beyond ensuring token usage is captured (which the CallbackHandler already does).

For programmatic cost querying:
```python
# Monitor daily costs
metrics = client.api.metrics.daily(...)
daily_cost = sum(d.total_cost or 0 for d in metrics.data)
if daily_cost > DAILY_COST_THRESHOLD:
    await alert_service.send(f"LLM spend alert: ${daily_cost:.2f} today")
```

#### Priority: P1 | Effort: Low (half-day configuration)

---

### 5.3 Data Export — Analytics Pipeline

#### What Langfuse Offers

Langfuse supports data export for building external analytics pipelines:
- Bulk trace export as JSONL
- S3/GCS export integration
- Webhook-based streaming export

This enables training fine-tuned models on high-quality production traces, building custom analytics in external BI tools, and long-term data retention outside of Langfuse.

#### What Meta-Builder Does Instead

No data export configuration. Production trace data is inaccessible outside the Langfuse UI.

#### How to Bridge It

```python
# Periodic export of high-quality traces for fine-tuning dataset creation
from langfuse import get_client

def export_high_quality_traces(min_score: float = 0.85):
    client = get_client()
    
    # Query high-scoring traces (requires scoring to be implemented first)
    traces = client.api.trace.list(
        tags=["pipeline_rag"],
        from_timestamp=datetime.utcnow() - timedelta(days=7),
    )
    
    high_quality = [
        t for t in traces.data
        if t.scores and any(s.value >= min_score for s in t.scores if s.name == "overall_quality")
    ]
    
    # Export to S3 or local storage for fine-tuning
    return [{"input": t.input, "output": t.output, "score": ...} for t in high_quality]
```

#### Priority: P3 | Effort: Medium (1 week — depends on fine-tuning/analytics needs)

---

### 5.4 MCP Server — AI Editor Integration

#### What Langfuse Offers

Langfuse provides an official MCP (Model Context Protocol) server that exposes Langfuse data to AI coding assistants (Cursor, Claude Code, etc.). With the Langfuse MCP server running, an AI assistant can:
- Query traces when debugging
- Analyze quality trends when optimizing
- Review prompt versions when editing agent code
- Access dataset items when writing tests

```bash
# Configure Cursor/Claude Code to use Langfuse MCP server
# .cursor/mcp.json
{
  "servers": {
    "langfuse": {
      "command": "npx",
      "args": ["langfuse-mcp-server"],
      "env": {
        "LANGFUSE_PUBLIC_KEY": "...",
        "LANGFUSE_SECRET_KEY": "..."
      }
    }
  }
}
```

#### What Meta-Builder Does Instead

No Langfuse MCP server configured. Developers using AI coding assistants for meta-builder work cannot query Langfuse data from within their editor.

#### Impact

In the context of meta-builder itself (an AI meta-builder), this is particularly ironic: the system that builds AI agents doesn't use AI-assisted development tools for its own observability data.

#### Priority: P3 | Effort: Low (1 hour configuration, no code changes)

---

## 6. Priority and Effort Matrix

| # | Opportunity | Priority | Effort | Impact | Files to Change |
|---|-------------|----------|--------|--------|-----------------|
| 1 | Scores via SDK (EvaluatorAgent) | **P0** | Medium | Very High | `evaluator_agent.py`, `verification_agent.py` |
| 2 | PII Masking | **P0** | Low | High (compliance) | `src/utils/tracing.py` |
| 3 | Proper TTL caching for prompts | **P0** | Low | High | `src/utils/prompts.py` |
| 4 | Sessions — mandatory session_id | **P0** | Low | High | All API routes, `request_processor.py` |
| 5 | Users — mandatory user_id | **P0** | Low | High | All API routes, `request_processor.py` |
| 6 | Environments separation | **P1** | Low | Medium | `src/config.py`, `config.yaml` |
| 7 | Releases and version tagging | **P1** | Low | Medium | `src/config.py`, CI/CD |
| 8 | Log levels on gate failures | **P1** | Medium | Medium | All gate agents |
| 9 | A/B testing for prompts | **P1** | Medium | High | `src/utils/prompts.py` |
| 10 | Prompt config extraction | **P1** | Low | Medium | `src/utils/prompts.py`, `llm/factory.py` |
| 11 | Multiple prompt labels (staging) | **P1** | Low | Medium | `src/utils/prompts.py` |
| 12 | Prompt-to-trace linkage | **P1** | Low | Medium | All PromptManager call sites |
| 13 | Prompt webhooks + cache invalidation | **P1** | Medium | High | `src/utils/prompts.py`, new webhook handler |
| 14 | User feedback collection | **P1** | Medium | High | New feedback endpoint, frontend |
| 15 | Spend alerts (Langfuse UI config) | **P1** | Low | Medium | Langfuse UI only |
| 16 | LLM-as-a-Judge templates | **P1** | High | High | `evaluator_agent.py` (migration) |
| 17 | Datasets from production traces | **P1** | High | Very High | New `dataset_management.py` |
| 18 | Experiment tracking (Langfuse) | **P1** | High | High | `gym.py`, `optimizer.py` |
| 19 | Agent graph visualization | **P1** | Medium | Medium | `graph_factory.py` |
| 20 | Annotation queues | **P2** | Medium | Medium | Post-evaluation pipeline |
| 21 | Multi-modality for VisionAgent | **P2** | Low | Medium | `vision_agent.py` |
| 22 | Sampling per route | **P2** | Low | Low | `src/config.py` |
| 23 | Custom dashboards (Langfuse UI) | **P2** | Low | Medium | Langfuse UI only |
| 24 | Metrics API for analytics | **P2** | Low | Low | New analytics endpoint |
| 25 | GitHub Sync for prompts | **P2** | Low | Medium | Repository setup |
| 26 | Playground linkage | **P2** | Low | Low | PromptManager call sites |
| 27 | Composable prompts | **P2** | Medium | Medium | All prompts restructured |
| 28 | Comments and corrections API | **P3** | Low | Low | Debugger extension |
| 29 | MCP Server for AI editors | **P3** | Low | Low | Configuration only |
| 30 | Data export pipeline | **P3** | Medium | Low | New export utility |
| 31 | MCP tracing | **P3** | Low | Low | `tool_acquisition.py` |

---

## 7. Implementation Roadmap

### Phase 1: Foundations (Week 1–2) — Fix what's broken

These are correctness issues or near-zero-effort wins with high impact:

1. **Fix prompt caching** — remove `@lru_cache`, use `cache_ttl_seconds=300` in `get_prompt()`
2. **Add PII masking** — add `masking_function` to Langfuse client initialization
3. **Set environments** — add `APP_ENV` → `LANGFUSE_ENVIRONMENT` in `configure_tracing()`
4. **Set releases** — add `GIT_SHA` → `LANGFUSE_RELEASE` in `configure_tracing()` and CI/CD
5. **Make session_id mandatory** — require session ID in all API routes, propagate to `set_trace_attributes()`
6. **Make user_id mandatory** — require user ID from auth middleware, propagate to `set_trace_attributes()`
7. **Configure spend alerts** — Langfuse UI configuration, 30 minutes

### Phase 2: Scoring (Week 3–4) — Turn data into intelligence

This is the highest-leverage phase:

8. **Wire EvaluatorAgent scores to Langfuse** — call `client.score()` for each quality dimension
9. **Wire VerificationAgent scores to Langfuse** — call `client.score()` for verification pass/fail
10. **Add log levels to gate agents** — mark gate failures as `WARNING`, hard failures as `ERROR`
11. **Return trace_id in API responses** — prerequisite for user feedback
12. **Add user feedback endpoint** — `/api/feedback` that calls `client.score()` with user rating

### Phase 3: Prompt Management (Week 5–6) — Enable data-driven iteration

13. **Extract prompt.config** — use model parameters from prompt versions in `llm/factory.py`
14. **Add staging label support** — add `PROMPT_LABEL` env var respected by `PromptManager`
15. **Ensure prompt-to-trace linkage** — audit all PromptManager call sites, merge `get_prompt_metadata()`
16. **Implement A/B testing** — add `get_template_ab()` method to `PromptManager`
17. **Add prompt webhook handler** — invalidate prompts when Langfuse fires a change event

### Phase 4: Evaluation Platform (Week 7–10) — Close the quality loop

18. **Build dataset management utilities** — `DatasetManager` class for promoting traces to datasets
19. **Integrate AgentGym with Langfuse experiments** — run evolutionary optimization against Langfuse datasets
20. **Configure LLM-as-a-Judge evaluators** — move asynchronous evaluation to Langfuse evaluator templates
21. **Set up annotation queues** — queue low-scoring traces for human review
22. **Build custom dashboards** — quality trend, cost, error rate, latency dashboards in Langfuse UI

### Phase 5: Platform Completion (Week 11–12)

23. **VisionAgent multi-modality** — capture image URLs in observations
24. **Agent graph visualization** — wire LangGraph (or custom graph) to LangChain callback
25. **Metrics API analytics endpoint** — internal `/api/analytics` endpoint using Langfuse metrics
26. **Configure Langfuse MCP Server** — enable AI editor integration for the dev team
27. **Data export pipeline** — configure S3/GCS export for fine-tuning data
