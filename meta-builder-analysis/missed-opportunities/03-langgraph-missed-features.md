# LangGraph Missed Features: A Deep Dive for meta-builder-v3

**Document version:** 2026-04-08  
**Meta-builder version analyzed:** v4.9.0  
**LangGraph versions referenced:** 0.3.x – 0.6.x

---

## Executive Summary

Meta-builder-v3 makes competent use of LangGraph's core graph primitives — `StateGraph`, `ToolNode`, `Send`, conditional edges — but leaves the most transformative capabilities completely untouched. This document catalogues **12 distinct feature gaps**, ordered by operational impact. The largest gap by far is the absence of checkpointing: the Conductor pipeline compiles its graph with zero persistence, meaning every pipeline run is a single, fragile, unrecoverable execution. The second largest is the lack of streaming support, which prevents any real-time feedback loop from the multi-phase build process.

The features below are not nice-to-haves. Checkpointing, interrupts, and durable execution together constitute the foundation of a production-grade agentic system. Meta-builder is generating agents that also lack these foundations, which means it is replicating its own architectural shortcomings into every artifact it produces.

---

## Feature Gap Summary Table

| # | Feature | Current State | Priority | Effort |
|---|---------|---------------|----------|--------|
| 1 | Checkpointing & Persistence | Not implemented — graphs compile without checkpointer | **P0** | Medium |
| 2 | Durable Execution Modes | Not configured | **P0** | Low |
| 3 | Interrupts (Human-in-the-Loop) | Schema fields exist, never wired | **P0** | Medium |
| 4 | Time Travel (Replay & Fork) | Zero support | **P1** | Low (requires P1 first) |
| 5 | Functional API (`@entrypoint` + `@task`) | Graph API only | **P1** | Medium |
| 6 | Streaming System | Synchronous-only execution | **P1** | Medium |
| 7 | Memory Store (Long-term) | Custom `PipelineMemory`, not LangGraph Store | **P1** | Medium |
| 8 | `Command` Type for State Updates | Standard returns only | **P2** | Low |
| 9 | Supervisor & Swarm Packages | Reimplemented from scratch | **P2** | Low |
| 10 | Graph Visualization & Debugging | Minimal Studio integration | **P2** | Low |
| 11 | Pregel Runtime (Direct API) | Implicit use only | **P3** | High |
| 12 | `@cache` Decorator | Custom `StructuredLLMCache` | **P3** | Low |

---

## 1. Checkpointing and Persistence — P0 Critical

### What It Is

LangGraph's persistence layer is implemented via **checkpointers** — pluggable backends that save a snapshot of graph state at the end of every super-step. When a graph is compiled with a checkpointer and invoked with a `thread_id`, every node execution is automatically saved. The system supports three backends:

| Checkpointer | Package | Use Case |
|---|---|---|
| `InMemorySaver` | Built-in (`langgraph`) | Development, testing |
| `SqliteSaver` / `AsyncSqliteSaver` | `langgraph-checkpoint-sqlite` | Local / single-process production |
| `PostgresSaver` / `AsyncPostgresSaver` | `langgraph-checkpoint-postgres` | Distributed production |

A checkpoint (`StateSnapshot`) contains:
- `values` — the full state at that point
- `next` — which nodes execute next
- `tasks` — `PregelTask` objects including error and interrupt info
- `metadata` — step number, source node, write operations
- `created_at` — timestamp
- `parent_config` — link to the prior checkpoint

**Basic Integration**

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.postgres import PostgresSaver

# Development
checkpointer = InMemorySaver()

# Production
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@host:5432/dbname"
)

# Compile WITH persistence
app = workflow.compile(checkpointer=checkpointer)

# Every invocation must carry a thread_id
config = {"configurable": {"thread_id": "pipeline-run-uuid-001"}}
result = app.invoke({"user_request": "build me an agent"}, config=config)
```

**Inspecting State After a Run**

```python
# Get the current (latest) state for a thread
snapshot = app.get_state(config)
print(snapshot.values)       # Full state dict
print(snapshot.next)         # What would run next (empty if done)
print(snapshot.metadata)     # Step count, which node wrote last

# List all checkpoints for a thread (most recent first)
history = list(app.get_state_history(config))
for i, checkpoint in enumerate(history):
    print(f"Step {i}: {checkpoint.metadata['step']} — next={checkpoint.next}")
```

### Why It Matters for Meta-Builder

**The Conductor pipeline is the highest-value workflow in the entire codebase.** It orchestrates `process_request → plan → generate → verify` across potentially dozens of LLM calls. Currently, `GraphFactory` and `PipelineGraph` compile every graph with `workflow.compile()` — no checkpointer. This has four concrete failure modes:

1. **No fault tolerance.** If an LLM provider times out after the `plan` phase, the entire pipeline must restart. Every token spent on planning is lost.

2. **No human approval gates.** The `requires_human_approval` field in `AgentAutonomy` and the `Gate` model in WorkflowDesign describe approval logic, but there is no mechanism to pause the pipeline and wait. Without a checkpointer, `interrupt()` cannot function at all.

3. **No conversation continuity.** The Conductor has no memory of prior runs. A user who iterates on a design with "make it use Claude instead" triggers a full re-run from scratch.

4. **No debugging.** When a generated pipeline fails in the `verify` phase, there is no way to inspect intermediate state, replay steps, or understand what the planner decided.

### Where to Integrate

**Primary integration point: `pipeline_graph.py`**

The `PipelineGraph` class constructs the Conductor graph. The checkpointer should be injected at construction time:

```python
# pipeline_graph.py — BEFORE (current)
class PipelineGraph:
    def build(self) -> CompiledGraph:
        workflow = StateGraph(ConductorState)
        # ... add nodes and edges ...
        return workflow.compile()

# pipeline_graph.py — AFTER (with checkpointing)
from langgraph.checkpoint.postgres import AsyncPostgresSaver
from langgraph.checkpoint.memory import InMemorySaver

class PipelineGraph:
    def __init__(self, checkpointer=None):
        self.checkpointer = checkpointer or InMemorySaver()

    def build(self) -> CompiledGraph:
        workflow = StateGraph(ConductorState)
        # ... add nodes and edges ...
        return workflow.compile(checkpointer=self.checkpointer)
```

**Secondary integration: `GraphFactory` (agent generation)**

GraphFactory generates agent graphs from `WorkflowDesign`. Generated agents should also receive checkpointers — this is what enables the agents meta-builder produces to have memory and recoverability:

```python
# graph_factory.py — generated graph compilation
def _compile_generated_graph(
    self,
    workflow: StateGraph,
    design: WorkflowDesign,
    checkpointer=None,
) -> CompiledGraph:
    """
    Compile a generated agent graph with optional persistence.
    Checkpointer should be injected by the caller (Conductor)
    based on the target deployment environment.
    """
    return workflow.compile(checkpointer=checkpointer)
```

**Tertiary integration: `AgentGym` sandbox execution**

The evolutionary optimizer runs agents in a sandbox. Without checkpointing, any long-running evolution job that crashes must restart from generation zero. Adding a `SqliteSaver` per experiment session gives crash recovery for free.

### Implementation Approach

1. Add `LANGGRAPH_CHECKPOINT_BACKEND` to environment config (`"memory"` | `"sqlite"` | `"postgres"`).
2. Create a `CheckpointerFactory` utility that returns the appropriate backend.
3. Inject the checkpointer into `PipelineGraph.__init__`.
4. Thread `thread_id` through the Conductor invocation — map it to the user's session ID or request UUID.
5. Update `langgraph.json` to configure the checkpointer for LangGraph Platform deployment.

---

## 2. Durable Execution Modes — P0 Critical

### What It Is

As of LangGraph 0.6.0, graphs support a `durability` parameter on any invocation method that controls **when checkpoints are written to the backing store**:

| Mode | When State Is Persisted | Crash Recovery | Performance |
|---|---|---|---|
| `"exit"` | Only when graph finishes (success or error) | None — intermediate state lost | Fastest |
| `"async"` | Asynchronously while the next step begins | Near-complete — tiny window of exposure | Good |
| `"sync"` | Synchronously before each step starts | Full — every checkpoint guaranteed | Slower |

```python
# Pass durability per-invocation — no compile-time lock-in
result = app.invoke(
    {"user_request": "build me an agent"},
    config=config,
    durability="sync",   # Guarantee every checkpoint before continuing
)

# Or stream with durability control
async for chunk in app.astream(
    {"user_request": "build me an agent"},
    config=config,
    durability="async",  # Good balance for streaming
    stream_mode="updates",
):
    yield chunk

# Resume after a crash — invoke with None input on same thread_id
# The graph resumes from the last recorded checkpoint
result = app.invoke(None, config=config)
```

### Why It Matters for Meta-Builder

Meta-builder has **three long-running execution contexts** where durability mode is critical:

1. **Conductor pipeline:** A multi-phase build with expensive LLM calls at every phase. `durability="sync"` is the right choice — the performance overhead is trivial compared to LLM latency, and any crash after `generate` but before `verify` would otherwise mean re-running generation.

2. **Evolution optimizer in AgentGym:** Evolutionary runs can take minutes or hours. Without durable state, a Python OOM, network blip, or SIGKILL wipes the entire population. `durability="async"` is correct here — good durability without the synchronous overhead of single-step fitness evaluations.

3. **Generated agents in production:** GraphFactory-generated agents inherit whatever durability mode is configured. Users whose agents are processing long documents or multi-step tasks need `durability="sync"` by default.

### Where to Integrate

**`pipeline_graph.py`** — pass `durability` when invoking the Conductor:

```python
# conductor.py or wherever Conductor.invoke is called
result = conductor_graph.invoke(
    initial_state,
    config={"configurable": {"thread_id": run_id}},
    durability="sync",  # Non-negotiable for Conductor
)
```

**`agent_gym.py`** — evolution loop:

```python
# agent_gym.py — per-generation invoke in the evolution loop
for generation in range(max_generations):
    fitness_results = optimizer_graph.invoke(
        population_state,
        config={"configurable": {"thread_id": experiment_id}},
        durability="async",  # Balance: good recovery, low overhead
    )
```

**`WorkflowDesign` schema** — expose as a configuration field:

```python
class WorkflowDesign(BaseModel):
    # ... existing fields ...
    durability_mode: Literal["exit", "async", "sync"] = "async"
    """Checkpoint durability mode for the generated graph."""
```

### Implementation Approach

This is the lowest-effort P0 item. Once checkpointing is in place (Feature #1), durability requires only a single parameter addition per invocation site. Total implementation: less than 30 lines of code changes across 3 files.

---

## 3. Interrupts (Dynamic Human-in-the-Loop) — P0 Critical

### What It Is

LangGraph's `interrupt()` function is a dynamic breakpoint that can be called from anywhere inside a node — not just at predefined graph edges. When called, it:

1. **Immediately halts** graph execution at that exact line of code
2. **Saves the current checkpoint** (requires checkpointer)
3. **Surfaces a payload** to the caller as an `Interrupt` object in the stream
4. **Waits indefinitely** — there is no timeout
5. **Resumes** when the caller invokes `Command(resume=<value>)` on the same thread

```python
from langgraph.types import interrupt, Command

# Inside any node function:
def review_gate_node(state: ConductorState) -> dict:
    """Gate that pauses for human approval before code generation."""
    approval_request = {
        "phase": "pre_generation",
        "plan_summary": state["plan"]["summary"],
        "estimated_cost": state["plan"]["token_estimate"],
        "action": "Approve to begin code generation, reject to revise plan",
    }

    # Execution halts here. The dict above is returned to the caller.
    decision = interrupt(approval_request)

    # This line only runs after Command(resume=...) is sent
    if decision == "approved":
        return {"gate_status": "passed", "gate_override": False}
    elif decision == "reject":
        return {"gate_status": "failed", "gate_override": False}
    else:
        # Human provided corrective state as a dict
        return {"gate_status": "passed", "gate_override": True, **decision}
```

**Resuming from the caller side:**

```python
# Caller receives an Interrupt event when streaming
for chunk in conductor_graph.stream(initial_input, config=config):
    if isinstance(chunk, dict) and "__interrupt__" in chunk:
        interrupt_data = chunk["__interrupt__"][0].value
        print(f"Waiting for approval: {interrupt_data}")
        break

# ... Human reviews and approves via UI ...

# Resume with human decision
for chunk in conductor_graph.stream(
    Command(resume="approved"),
    config=config,   # Same thread_id — critical
):
    print(chunk)
```

**Static interrupts** (compile-time breakpoints, simpler but less flexible):

```python
# Compile with static interrupt_before a specific node
app = workflow.compile(
    checkpointer=checkpointer,
    interrupt_before=["code_generation_node"],
    interrupt_after=["verify_node"],
)
```

### Why It Matters for Meta-Builder — The Broken Gate System

The most important current gap is that **meta-builder's gate system is purely cosmetic**. The `Gate` model in `WorkflowDesign`, the `requires_human_approval` field in `AgentAutonomy`, and the `"human"` node type in `WorkflowDesign.nodes` are all schema definitions with no runtime behavior. They appear in generated code, but the generated graphs run straight through them.

The Conductor itself has the same problem: even if it encounters a pipeline that it cannot verify, it has no mechanism to pause and ask the user for guidance. It either retries autonomously or fails.

With `interrupt()`, three concrete improvements become available:

**A. Conductor pre-generation approval gate**

```python
# pipeline_graph.py — add a gate node after planning, before generation
def pre_generation_gate(state: ConductorState) -> dict:
    """
    If the plan cost exceeds threshold or autonomy level requires approval,
    pause and ask the human.
    """
    autonomy = state["workflow_design"].autonomy
    plan = state["plan"]

    needs_approval = (
        autonomy.requires_human_approval
        or plan.estimated_tokens > autonomy.max_tokens_per_run
        or plan.risk_level == "high"
    )

    if needs_approval:
        decision = interrupt({
            "type": "pre_generation_approval",
            "plan": plan.dict(),
            "message": "Review the plan before code generation begins.",
        })
        if decision == "reject":
            return {"phase": "plan", "gate_rejected": True}

    return {"phase": "generate", "gate_rejected": False}
```

**B. GraphFactory generating real interrupt() calls for human nodes**

Currently, GraphFactory sees `node_type="human"` in `WorkflowDesign.nodes` and generates placeholder code. It should generate actual `interrupt()` calls:

```python
# graph_factory.py — template for human nodes (current — BROKEN)
def _generate_human_node(self, node: NodeDesign) -> str:
    return f"""
def {node.name}(state: {self.state_type_name}) -> dict:
    # TODO: Human approval needed
    return {{}}  # ← This is a no-op; the human never actually sees anything
"""

# graph_factory.py — template for human nodes (CORRECT)
def _generate_human_node(self, node: NodeDesign) -> str:
    return f"""
def {node.name}(state: {self.state_type_name}) -> dict:
    \"\"\"
    Human-in-the-loop node. Pauses execution and surfaces state to caller.
    Resume by invoking with Command(resume=<your_decision>).
    \"\"\"
    from langgraph.types import interrupt
    decision = interrupt({{
        "node": "{node.name}",
        "prompt": "{node.human_prompt or 'Review and approve to continue.'}",
        "state_snapshot": {{k: v for k, v in state.items() if k != 'messages'}},
    }})
    return {{"human_decision": decision, "last_human_node": "{node.name}"}}
"""
```

**C. VisionAgent interactive diagram refinement**

VisionAgent could interrupt after producing an initial architecture diagram and collect user feedback before generating final code:

```python
def vision_refinement_node(state: VisionState) -> dict:
    initial_diagram = state["diagram"]
    feedback = interrupt({
        "type": "diagram_review",
        "diagram_mermaid": initial_diagram.mermaid_source,
        "message": "Review the architecture. Provide feedback or 'approve'.",
    })
    if feedback != "approve":
        return {"diagram_feedback": feedback, "needs_revision": True}
    return {"needs_revision": False}
```

### Where to Integrate

| File | Change |
|---|---|
| `pipeline_graph.py` | Add `pre_generation_gate` node; compile with `checkpointer` |
| `graph_factory.py` | Update `_generate_human_node()` template |
| `vision_agent.py` | Add `interrupt()` call in diagram review step |
| `conductor.py` | Surface `Interrupt` objects to caller via streaming |
| `langgraph.json` | Ensure interrupt support configured for Studio |

---

## 4. Time Travel (Replay and Fork) — P1 High Value

### What It Is

Once checkpointing is in place, LangGraph exposes **time travel** — the ability to replay from or fork at any prior checkpoint in a thread's history. This is accomplished via three operations:

**Get full state history:**

```python
config = {"configurable": {"thread_id": "pipeline-run-001"}}
history = list(app.get_state_history(config))
# Returns list of StateSnapshot objects, most recent first

for checkpoint in history:
    print(f"Step {checkpoint.metadata['step']}: next={checkpoint.next}")
    print(f"  State keys: {list(checkpoint.values.keys())}")
    print(f"  Checkpoint ID: {checkpoint.config['configurable']['checkpoint_id']}")
```

**Replay from a prior checkpoint (re-execute forward from that point):**

```python
# Get the checkpoint_id from history
target_checkpoint = history[3]  # e.g., the state after 'plan' phase
replay_config = target_checkpoint.config  # Contains checkpoint_id

# Invoke with the checkpoint config — LangGraph replays from that point
result = app.invoke(None, config=replay_config)
```

**Fork with modified state (branch, don't overwrite):**

```python
# Modify state at a prior checkpoint and branch execution
fork_config = app.update_state(
    config=target_checkpoint.config,
    values={"plan": revised_plan},      # Override plan with human-revised version
    as_node="planning_node",            # Pretend planning_node wrote this update
)
# fork_config contains a new checkpoint_id — the original thread is untouched
result = app.invoke(None, config=fork_config)
```

### Why It Matters for Meta-Builder

**Debugging failed pipeline runs:**  
When the Conductor's `verify` phase fails, there is currently no way to understand what the `plan` phase decided, what `generate` produced, or where in the chain things went wrong. With time travel, a developer can list the state history, identify exactly which step introduced the bug, inspect the state at that point, and replay forward with a fix.

**The evolutionary optimizer — the killer use case:**  
The `AgentGym` evolution loop produces generations of agents. Today, the entire population history is lost after a run. With checkpointing and time travel, every generation is a checkpoint. The optimizer can:

1. Fork from a checkpoint representing the best agent in generation N
2. Apply a mutation to the forked state
3. Evaluate the mutated agent
4. Keep or discard the fork

This is **exactly how evolutionary algorithms work**, and LangGraph's checkpoint/fork system maps onto it precisely. The optimizer's current in-memory population management is a reinvention of this capability.

**Generated agents — undo/redo for users:**  
Agents that expose document editing, code generation, or multi-step transformations can offer true undo/redo. The user's thread history is the undo stack. Forking creates a new branch without destroying the prior state.

```python
# Example: User-facing undo in a generated agent
class GeneratedAgentAPI:
    def undo_last_step(self, thread_id: str):
        config = {"configurable": {"thread_id": thread_id}}
        history = list(self.app.get_state_history(config))
        if len(history) < 2:
            raise ValueError("Nothing to undo")
        # Get the state one step back
        prior_checkpoint = history[1]
        # Update state to the prior checkpoint's values
        self.app.update_state(
            config=config,
            values=prior_checkpoint.values,
            as_node=prior_checkpoint.metadata.get("source"),
        )

    def fork_at_step(self, thread_id: str, step_index: int, override: dict):
        config = {"configurable": {"thread_id": thread_id}}
        history = list(self.app.get_state_history(config))
        target = history[step_index]
        fork_config = self.app.update_state(
            config=target.config,
            values=override,
        )
        return fork_config["configurable"]["thread_id"]  # New thread for the fork
```

### Where to Integrate

Time travel is a capability that becomes available immediately after Feature #1 (checkpointing) is implemented — no additional code changes required in the core graph. The integration work is:

1. **Admin/debug endpoints** in the API layer: expose `get_state_history()` and `update_state()` for the Conductor thread
2. **Evolution optimizer refactor:** replace in-memory population with checkpoint-based forking
3. **Generated agent SDK:** include helper methods for undo/redo as optional scaffolding

---

## 5. Functional API (`@entrypoint` + `@task`) — P1

### What It Is

LangGraph has two parallel APIs for building workflows. Meta-builder uses only the Graph API (`StateGraph`, nodes, edges). The Functional API provides a different mental model:

| Dimension | Graph API | Functional API |
|---|---|---|
| Structure | Explicit nodes + edges + reducers | Plain Python functions with decorators |
| State | Shared TypedDict with reducers | Scoped to function — no shared state |
| Checkpointing | After every super-step | After each `@task` result |
| Control flow | Graph edges + conditional routing | `if`, `for`, `while`, function calls |
| Visualization | Full graph diagram | Not supported (dynamic) |
| Best for | Complex multi-agent graphs | Sequential pipelines, IR passes, simple chains |

**`@entrypoint`** marks a workflow entry point. It accepts a checkpointer and produces a `Pregel` instance (same runtime as the Graph API):

```python
from langgraph.func import entrypoint, task
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt

checkpointer = InMemorySaver()

@task
def validate_input(request: dict) -> dict:
    """Each @task result is checkpointed — crash-safe."""
    if not request.get("description"):
        raise ValueError("description required")
    return {"validated": True, "request": request}

@task
def call_llm_for_plan(validated: dict) -> dict:
    """LLM call result saved to checkpoint — won't re-run on resume."""
    response = llm.invoke([HumanMessage(content=validated["request"]["description"])])
    return {"plan": response.content}

@entrypoint(checkpointer=checkpointer)
def simple_pipeline(inputs: dict) -> dict:
    """
    Standard Python control flow — no graph edges needed.
    Each task result is checkpointed automatically.
    """
    validated = validate_input(inputs).result()

    if validated.get("needs_approval"):
        approved = interrupt({"plan": "pending", "action": "approve to continue"})
        if approved != "yes":
            return {"status": "rejected"}

    plan = call_llm_for_plan(validated).result()
    return {"status": "complete", "plan": plan["plan"]}
```

**Task caching with `CachePolicy`:**

```python
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy

@task(cache_policy=CachePolicy(ttl=300))  # Cache for 5 minutes
def expensive_analysis(code: str) -> dict:
    """If called twice with same input within TTL, returns cached result."""
    return static_analyzer.analyze(code)

@entrypoint(checkpointer=checkpointer, cache=InMemoryCache())
def analysis_workflow(inputs: dict) -> dict:
    # Second call with same 'code' value returns immediately from cache
    result = expensive_analysis(inputs["code"]).result()
    return result
```

**Parallel task execution:**

```python
@entrypoint(checkpointer=checkpointer)
def parallel_pipeline(topics: list[str]) -> list[str]:
    """Tasks submitted simultaneously — all run in parallel."""
    futures = [generate_section(topic) for topic in topics]
    results = [f.result() for f in futures]  # Wait for all
    return results
```

### Why It Matters for Meta-Builder

**IR compiler passes** — the Intermediate Representation compiler applies a sequence of transformation passes (parsing → validation → optimization → emission). Each pass is a pure function with no cross-pass state. These are exactly what `@task` was designed for:

```python
# ir_compiler.py — current approach (no crash recovery)
class IRCompiler:
    def compile(self, workflow_design: WorkflowDesign) -> IR:
        parsed = self.parse(workflow_design)        # If crash here...
        validated = self.validate(parsed)           # ...all work lost
        optimized = self.optimize(validated)
        emitted = self.emit(optimized)
        return emitted

# ir_compiler.py — Functional API approach (crash recovery at each pass)
@task
def parse_design(design_dict: dict) -> dict:
    return IRParser().parse(WorkflowDesign(**design_dict)).dict()

@task
def validate_ir(ir_dict: dict) -> dict:
    return IRValidator().validate(IR(**ir_dict)).dict()

@task
def optimize_ir(ir_dict: dict) -> dict:
    return IROptimizer().optimize(IR(**ir_dict)).dict()

@task
def emit_code(ir_dict: dict) -> dict:
    return CodeEmitter().emit(IR(**ir_dict))

@entrypoint(checkpointer=checkpointer)
def compile_workflow(design_dict: dict) -> dict:
    parsed = parse_design(design_dict).result()
    validated = validate_ir(parsed).result()
    optimized = optimize_ir(validated).result()
    code = emit_code(optimized).result()
    return code
```

If the LLM-backed `optimize_ir` step crashes mid-call, the next invocation on the same thread skips `parse_design` and `validate_ir` (results already checkpointed) and retries only `optimize_ir`.

**`generation_tier="simple_chain"` in generated agents** — WorkflowDesign has a `generation_tier` field. Simple linear chains do not benefit from `StateGraph`'s complexity. GraphFactory could generate Functional API workflows for `simple_chain` tier, which are shorter, more readable, and require no state type definition.

**Research pipelines** — any pipeline that sequentially calls tools (web search → summarize → cite → synthesize) is a natural fit for the Functional API.

### Where to Integrate

| File | Change |
|---|---|
| `ir_compiler.py` | Refactor compiler passes to `@task` functions under a `@entrypoint` |
| `graph_factory.py` | Add Functional API code generator for `generation_tier="simple_chain"` |
| `pipeline_graph.py` | Consider Functional API for the Conductor's linear phases |

---

## 6. Streaming System — P1 High Value

### What It Is

LangGraph graphs expose `.stream()` (sync) and `.astream()` (async) methods that yield chunks as execution progresses. Five streaming modes are supported:

| Mode | What You Receive | Best For |
|---|---|---|
| `"values"` | Full state dict after each step | State inspection, debugging |
| `"updates"` | Delta dict per node (only changed keys) | Progress tracking, lightweight |
| `"messages"` | `(token, metadata)` tuples from LLMs | Token-level streaming, UX |
| `"custom"` | Arbitrary data emitted via `get_stream_writer()` | Progress bars, tool results |
| `"debug"` | Maximum internal event detail | Deep debugging |

**Basic streaming:**

```python
# Stream node-level updates as the Conductor runs
async for chunk in conductor_graph.astream(
    initial_state,
    config=config,
    stream_mode="updates",
):
    node_name = list(chunk.keys())[0]
    print(f"[{node_name}] completed: {list(chunk[node_name].keys())}")
```

**Token-level LLM streaming:**

```python
# Stream tokens as the planning LLM generates the plan
async for token, metadata in conductor_graph.astream(
    initial_state,
    config=config,
    stream_mode="messages",
):
    if metadata["langgraph_node"] == "planning_node" and token.content:
        yield token.content  # Stream to frontend in real-time
```

**Custom events from inside nodes:**

```python
from langgraph.config import get_stream_writer

def code_generation_node(state: ConductorState) -> dict:
    writer = get_stream_writer()

    # Emit progress events visible to caller
    writer({"type": "progress", "phase": "generation", "pct": 0})
    for i, component in enumerate(state["plan"]["components"]):
        writer({"type": "component_start", "name": component["name"]})
        code = generate_component(component)
        writer({"type": "component_done", "name": component["name"], "lines": len(code)})
        writer({"type": "progress", "phase": "generation", "pct": (i+1) / len(state["plan"]["components"]) * 100})

    return {"generated_code": all_code}

# Caller receives custom events
async for chunk in conductor_graph.astream(
    initial_state, config=config, stream_mode="custom"
):
    if chunk.get("type") == "progress":
        update_progress_bar(chunk["pct"])
    elif chunk.get("type") == "component_done":
        print(f"Generated {chunk['name']} ({chunk['lines']} lines)")
```

**Multiple modes simultaneously:**

```python
async for mode, chunk in conductor_graph.astream(
    initial_state,
    config=config,
    stream_mode=["updates", "messages", "custom"],
):
    if mode == "updates":
        log_phase_completion(chunk)
    elif mode == "messages":
        stream_token_to_ui(chunk[0].content)  # chunk = (token, metadata)
    elif mode == "custom":
        handle_progress_event(chunk)
```

### Why It Matters for Meta-Builder

The Conductor pipeline currently runs as a completely opaque, synchronous black box. A user who requests "build me a research agent" gets nothing until the entire pipeline completes — potentially minutes later. There is no progress feedback, no way to know which phase is running, no token-by-token visibility into what the planner decided.

This is the second most impactful missing feature after checkpointing. Streaming enables:

1. **Real-time build progress UI** — users see `[planning] → [generating] → [verifying]` as it happens
2. **LLM token streaming** — the plan and code appear progressively rather than all at once
3. **Custom progress events** — `"Generated authentication module (142 lines)"` in real-time
4. **Interrupt surfacing** — interrupts appear as stream events, enabling the UI to show approval dialogs at the right moment

**Generated agents also need streaming.** GraphFactory-generated agents run synchronously. An agent that processes a 50-page document has no way to surface progress. Every generated agent should be exposed via an `astream()` interface.

### Where to Integrate

**`conductor.py` / `pipeline_graph.py`** — switch all Conductor invocations to `astream()`:

```python
# conductor.py — current (no streaming)
async def run_pipeline(self, request: PipelineRequest) -> PipelineResult:
    result = await self.graph.ainvoke(initial_state, config=config)
    return PipelineResult(**result)

# conductor.py — with streaming
async def stream_pipeline(
    self, request: PipelineRequest
) -> AsyncIterator[dict]:
    """Yields streaming chunks as the pipeline executes."""
    async for mode, chunk in self.graph.astream(
        initial_state,
        config=config,
        stream_mode=["updates", "messages", "custom"],
        durability="sync",
    ):
        yield {"mode": mode, "data": chunk}
```

**`graph_factory.py`** — generated agents should expose `astream()` in their scaffolding:

```python
# Template for generated agent class
AGENT_CLASS_TEMPLATE = """
class {agent_name}Agent:
    def __init__(self, checkpointer=None):
        self.graph = build_{agent_name_lower}_graph(checkpointer=checkpointer)

    def invoke(self, input: dict, thread_id: str = None) -> dict:
        config = {{"configurable": {{"thread_id": thread_id or str(uuid4())}}}}
        return self.graph.invoke(input, config=config)

    async def astream(self, input: dict, thread_id: str = None, stream_mode="updates"):
        config = {{"configurable": {{"thread_id": thread_id or str(uuid4())}}}}
        async for chunk in self.graph.astream(input, config=config, stream_mode=stream_mode):
            yield chunk
"""
```

---

## 7. Memory Store (Long-term Memory) — P1

### What It Is

LangGraph's `Store` system is **separate from checkpointing**. While checkpointers handle within-thread state (short-term memory), the Store provides **cross-thread, cross-session persistence** — true long-term memory.

| Capability | Checkpointer | Store |
|---|---|---|
| Scope | Single thread | Any namespace |
| Purpose | Within-run state | Cross-run knowledge |
| Access pattern | Automatic (per superstep) | Explicit get/put/search |
| Semantic search | No | Yes |
| Backends | Postgres, SQLite, Memory | Postgres, Memory |

**Setup and basic operations:**

```python
from langgraph.store.memory import InMemoryStore

def embed(texts: list[str]) -> list[list[float]]:
    return embedding_model.embed_documents(texts)  # LangChain embeddings

store = InMemoryStore(index={"embed": embed, "dims": 1536})

# Namespace: (user_id, category)
namespace = ("user_abc123", "project_preferences")

# Write a memory
store.put(
    namespace,
    "preferred_patterns",
    {
        "orchestration": "supervisor",
        "state_type": "messages",
        "tools": ["web_search", "code_interpreter"],
        "llm_preference": "claude-3-5-sonnet",
    },
)

# Read by key
preferences = store.get(namespace, "preferred_patterns")
print(preferences.value)  # The stored dict

# Semantic search across all items in namespace
results = store.search(
    namespace,
    query="what LLM does this user prefer for coding tasks",
    filter={"orchestration": "supervisor"},  # Metadata filter
    limit=5,
)
```

**Injecting the store into graph nodes:**

```python
from langgraph.store.base import BaseStore

# Node signature — store is injected automatically when graph is compiled with store=
def planning_node(state: ConductorState, store: BaseStore) -> dict:
    """Planning node with access to long-term user memory."""
    user_ns = ("users", state["user_id"], "preferences")

    # Recall prior preferences
    prior_preferences = store.search(
        user_ns,
        query=state["user_request"],
        limit=3,
    )

    # Build context-aware prompt with memory
    memory_context = "\n".join([
        f"- {item.value.get('preference')}" for item in prior_preferences
    ])
    plan = llm.invoke([
        SystemMessage(content=f"User preferences:\n{memory_context}"),
        HumanMessage(content=state["user_request"]),
    ])

    # Save new preference discovered during this run
    store.put(
        user_ns,
        f"session_{state['run_id']}",
        {"preference": "prefers async patterns", "observed_at": "planning"},
    )

    return {"plan": plan.content}

# Compile with store
app = workflow.compile(checkpointer=checkpointer, store=store)
```

### Why It Matters for Meta-Builder

**The current `PipelineMemory` model is a dead end.** It stores metadata about past pipelines in a custom schema, but it is not integrated with LangGraph's Store, meaning:

1. It cannot be searched semantically ("what patterns did this user prefer last month?")
2. It is not namespaced — all pipelines share a flat memory space
3. It is not injectable into graph nodes — each node that needs memory must reach outside the LangGraph execution context
4. It cannot survive across deployment restarts without a custom serialization layer

**The `MemoryChannel` model in `WorkflowDesign`** describes memory channels for generated agents, but GraphFactory never wires them to anything real. Every generated agent starts with zero memory of prior interactions.

**What LangGraph Store unlocks for meta-builder:**

```python
# User's multi-session design continuity
user_ns = ("users", user_id, "design_history")

# After each successful pipeline run, save the design
store.put(user_ns, f"design_{run_id}", {
    "orchestration_pattern": design.orchestration_pattern,
    "nodes": [n.dict() for n in design.nodes],
    "success_metrics": final_metrics,
    "user_feedback": feedback,
})

# At the start of a new request, recall relevant designs
similar_designs = store.search(
    user_ns,
    query=new_request.description,
    limit=5,
)

# Planner includes prior successful designs in context:
# "This user has built 3 supervisor-pattern agents. Their preferred
# tool set is [web_search, code_interpreter]. Last agent scored 0.87."
```

### Where to Integrate

| File | Change |
|---|---|
| `pipeline_graph.py` | Compile with `store=store`; pass store to planning and verification nodes |
| `graph_factory.py` | Generate `store: BaseStore` parameter in nodes where `MemoryChannel` is defined |
| `pipeline_memory.py` | Refactor to be a wrapper over LangGraph Store instead of custom persistence |
| `conductor.py` | Pass `user_id` in config and inject store |

---

## 8. `Command` Type for State Updates — P2

### What It Is

LangGraph's `Command` type allows a **node to simultaneously update state AND navigate to a specific next node** — combining what normally requires a node function plus a conditional edge:

```python
from langgraph.types import Command
from typing import Literal

# Standard approach: node updates state, edge routes based on state
def verify_node(state: ConductorState) -> dict:
    result = verifier.check(state["generated_code"])
    return {"verification_result": result}
# Then a conditional edge reads verification_result and routes

# Command approach: node updates state AND routes — no conditional edge needed
def verify_node(state: ConductorState) -> Command[Literal["fix_node", "publish_node"]]:
    result = verifier.check(state["generated_code"])
    if result.passed:
        return Command(
            update={"verification_result": result, "phase": "publish"},
            goto="publish_node",
        )
    else:
        return Command(
            update={"verification_result": result, "phase": "fix", "errors": result.errors},
            goto="fix_node",
        )
```

**Tool-level state updates** — tools can return `Command` to update graph state:

```python
@tool
def web_search(query: str) -> Command:
    results = search_client.search(query)
    return Command(
        update={
            "search_results": results,
            "sources_used": [r.url for r in results],
        }
    )
    # No goto= needed — ToolNode handles routing back to agent
```

**Multi-agent handoffs with data passing:**

```python
def supervisor_node(state: SupervisorState) -> Command[Literal["coder_agent", "researcher_agent"]]:
    decision = llm.with_structured_output(NextAgent).invoke(state["messages"])
    return Command(
        update={"active_agent": decision.agent, "task": decision.task},
        goto=decision.agent,
    )
```

**Subgraph to parent navigation:**

```python
def subgraph_node(state: SubState) -> Command[Literal["parent_result_node"]]:
    result = do_work(state)
    return Command(
        update={"subgraph_output": result},
        goto="parent_result_node",
        graph=Command.PARENT,   # Navigate to node in parent graph
    )
```

### Why It Matters for Meta-Builder

GraphFactory currently generates `condition_map` + `condition_field` routing — a pattern where conditional edges read a state field to choose the next node. The `Command` type makes this more expressive for cases where the routing decision and the state update happen in the same operation.

The most valuable application is **multi-agent handoffs in generated supervisor graphs**. Currently, `orchestration_pattern="supervisor"` generates a supervisor node that updates `next_agent` and relies on a conditional edge that reads it. With `Command`, the supervisor node directly navigates:

```python
# graph_factory.py — supervisor pattern with Command (cleaner)
def _generate_supervisor_node(self, agents: list[AgentDesign]) -> str:
    agent_names = [a.name for a in agents]
    literal_type = " | ".join([f'"{n}"' for n in agent_names] + ['"END"'])
    return f"""
def supervisor_node(state: {self.state_type_name}) -> Command[Literal[{literal_type}]]:
    decision = supervisor_llm.with_structured_output(SupervisorDecision).invoke(
        state["messages"]
    )
    if decision.next == "FINISH":
        return Command(goto=END)
    return Command(
        update={{"active_agent": decision.next, "task": decision.task}},
        goto=decision.next,
    )
"""
```

This eliminates one conditional edge per supervisor pattern and makes the routing logic self-documenting via return type annotations.

---

## 9. Supervisor and Swarm Packages — P2

### What It Is

LangGraph publishes two first-party packages for common multi-agent topologies:

**`langgraph-supervisor`** (`pip install langgraph-supervisor`):

```python
from langgraph_supervisor import create_supervisor, create_handoff_tool
from langchain_openai import ChatOpenAI

# Two specialized sub-agents
research_agent = create_react_agent(model, tools=[web_search], name="researcher")
code_agent = create_react_agent(model, tools=[python_repl], name="coder")

# Supervisor in 4 lines
workflow = create_supervisor(
    [research_agent, code_agent],
    model=ChatOpenAI(model="gpt-4o"),
    prompt="Route to researcher for information gathering, coder for implementation.",
    output_mode="last_message",  # or "full_history"
)
app = workflow.compile(checkpointer=checkpointer, store=store)
result = app.invoke({"messages": [{"role": "user", "content": "Build me a web scraper"}]})
```

**`langgraph-swarm`** (`pip install langgraph-swarm`):

```python
from langgraph_swarm import create_swarm, create_handoff_tool

# Agents with explicit handoff tools — decentralized routing
alice = create_agent(
    model,
    tools=[add, create_handoff_tool(agent_name="Bob", description="Transfer math to Bob")],
    system_prompt="You are Alice, a researcher.",
    name="Alice",
)
bob = create_agent(
    model,
    tools=[python_repl, create_handoff_tool(agent_name="Alice", description="Transfer research to Alice")],
    system_prompt="You are Bob, a programmer.",
    name="Bob",
)

workflow = create_swarm([alice, bob], default_active_agent="Alice")
app = workflow.compile(checkpointer=checkpointer)  # Checkpointer required for swarm state
```

**Supervisor vs. Swarm:**

| Dimension | Supervisor | Swarm |
|---|---|---|
| Routing authority | Central supervisor LLM | Each agent decides handoff |
| Active agent tracking | Supervisor decides each turn | `active_agent` persisted in state |
| Context passing | Full history or last message | Full history by default |
| Best for | Clear task delegation, audit trails | Dynamic, conversational multi-agent |

### Why It Matters for Meta-Builder

Meta-builder's `GraphFactory` has custom implementations for `orchestration_pattern="supervisor"` and `orchestration_pattern="swarm"`. These are non-trivial — they include handoff logic, state reducers, and routing conditions. The `langgraph-supervisor` and `langgraph-swarm` packages provide tested, maintained implementations of these exact patterns.

The specific opportunity: when GraphFactory generates a supervisor or swarm pattern, it should generate code that calls `create_supervisor()` or `create_swarm()` from these packages rather than building the pattern from scratch with raw `StateGraph` primitives. This:

1. Reduces generated code length by ~40% for these patterns
2. Inherits upstream bugfixes and improvements automatically
3. Ensures `create_handoff_tool` is used consistently (the current custom implementation may miss edge cases)
4. Makes the output_mode (`full_history` vs. `last_message`) configurable without custom reducer logic

```python
# graph_factory.py — generated supervisor pattern (BEFORE — from scratch)
def _generate_supervisor_graph(self, design: WorkflowDesign) -> str:
    # 80+ lines of StateGraph setup, handoff tools, conditional edges...

# graph_factory.py — generated supervisor pattern (AFTER — using package)
def _generate_supervisor_graph(self, design: WorkflowDesign) -> str:
    agents_str = ", ".join([f"{a.name}_agent" for a in design.agents])
    return f"""
from langgraph_supervisor import create_supervisor

workflow = create_supervisor(
    [{agents_str}],
    model=get_llm("{design.llm_model}"),
    prompt=\"\"\"{design.supervisor_prompt}\"\"\",
    output_mode="{design.output_mode or 'last_message'}",
)
app = workflow.compile(checkpointer=checkpointer, store=store)
"""
```

**Caveat:** The `langgraph-supervisor` README explicitly notes it is maintained primarily for LangChain 1.0 compatibility and recommends manual implementation for most production use cases. Evaluate against the specific generated use case.

---

## 10. Graph Visualization and Debugging — P2

### What It Is

Every compiled LangGraph graph can generate a visual representation of its structure:

```python
# Mermaid diagram (text — paste into mermaid.live or embed in docs)
print(conductor_graph.get_graph().draw_mermaid())

# PNG via Mermaid.ink API (no local dependencies)
from IPython.display import Image, display
display(Image(conductor_graph.get_graph().draw_mermaid_png()))

# PNG with custom styling
from langgraph.graph.graph import CurveStyle, MermaidDrawMethod, NodeStyles
png_bytes = conductor_graph.get_graph().draw_mermaid_png(
    draw_method=MermaidDrawMethod.API,
    curve_style=CurveStyle.LINEAR,
    node_colors=NodeStyles(
        first="#e8f4fd",
        last="#d4edda",
        default="#fff3cd",
    ),
)

# Subgraph visualization
display(Image(conductor_graph.get_graph(xray=True).draw_mermaid_png()))
```

**Runtime state inspection:**

```python
# Inspect current state mid-execution (from outside the graph)
config = {"configurable": {"thread_id": "pipeline-001"}}
snapshot = conductor_graph.get_state(config)
print(f"Current phase: {snapshot.values.get('phase')}")
print(f"Next nodes: {snapshot.next}")

# Manually override state (for debugging or human correction)
conductor_graph.update_state(
    config=config,
    values={"phase": "verify", "plan": corrected_plan},
    as_node="planning_node",
)
```

### Why It Matters for Meta-Builder

**`pipeline_graph.py` already exists** but is described as "basic" Studio support. The specific gaps:

1. **Generated agents have no visualization.** GraphFactory never emits `draw_mermaid()` calls. Every generated agent is a black box — the user cannot see the graph structure of what was built for them.

2. **The Conductor's own graph is never rendered.** A developer debugging the Conductor has no quick way to see the full node-edge structure.

3. **No `get_state()` / `update_state()` in the debug API.** There is no endpoint to inspect or override Conductor state during a run.

**Quick wins:**

```python
# graph_factory.py — add visualization method to every generated agent
AGENT_CLASS_TEMPLATE += """
    def visualize(self) -> str:
        \"\"\"Returns Mermaid diagram source for this agent's graph.\"\"\"
        return self.graph.get_graph().draw_mermaid()

    def visualize_png(self, output_path: str = None) -> bytes:
        \"\"\"Returns PNG bytes of this agent's graph. Optionally saves to file.\"\"\"
        png = self.graph.get_graph().draw_mermaid_png()
        if output_path:
            with open(output_path, "wb") as f:
                f.write(png)
        return png
"""

# pipeline_graph.py — expose Conductor's own diagram
class PipelineGraph:
    def get_diagram(self) -> str:
        return self.compiled_graph.get_graph(xray=True).draw_mermaid()
```

**LangGraph Studio integration** — `langgraph.json` already exists in meta-builder. To unlock full Studio support (breakpoints, state inspection panel, step-by-step replay):

```json
{
  "graphs": {
    "conductor": "./pipeline_graph.py:build_conductor_graph",
    "planning_agent": "./planning_agent.py:build_planning_agent"
  },
  "env": ".env",
  "python_version": "3.11",
  "dependencies": ["."]
}
```

With Studio connected, developers can set breakpoints at any node, inspect state at each step, and manually resume interrupted pipelines — all without code changes.

---

## 11. Pregel Runtime (Direct API) — P3

### What It Is

`StateGraph.compile()` produces a `Pregel` instance — the core LangGraph runtime engine. Pregel implements a **Bulk Synchronous Parallel** execution model:

1. **Plan:** Determine which actors (nodes) have pending input in their subscribed channels
2. **Execute:** Run all selected actors in parallel until all complete, one fails, or timeout
3. **Update:** Apply all actor outputs to channels via reducers
4. Repeat until no actors are selected or `max_steps` is reached

Every super-step ends with a checkpoint. Channels are the communication mechanism:

```python
from langgraph.channels import EphemeralValue, LastValue, Topic
from langgraph.pregel import Pregel, NodeBuilder

# Direct Pregel construction — bypasses StateGraph entirely
node_a = (
    NodeBuilder()
    .subscribe_only("input_channel")      # Read from this channel
    .do(lambda x: x.upper())              # Transform
    .write_to("output_channel")           # Write to this channel
)

app = Pregel(
    nodes={"node_a": node_a},
    channels={
        "input_channel": EphemeralValue(str),   # Single-step value, no persistence
        "output_channel": EphemeralValue(str),
    },
    input_channels=["input_channel"],
    output_channels=["output_channel"],
    checkpointer=checkpointer,
    step_timeout=30_000,  # 30s per super-step
)

result = app.invoke({"input_channel": "hello world"})
# → {"output_channel": "HELLO WORLD"}
```

**Channel types:**

| Channel | Behavior | Use Case |
|---|---|---|
| `EphemeralValue` | Cleared after each step | Single-pass data flow |
| `LastValue` | Retains last written value | Standard state fields |
| `Topic` | Accumulates all writes (list) | Message aggregation, `add_messages` |

### Why It Matters for Meta-Builder

Meta-builder uses `StateGraph` which wraps Pregel and provides `TypedDict` state, reducers, and conditional edges. This is correct for complex agents.

The direct Pregel API is relevant in one specific scenario: **the Conductor's parallel fan-out via `Send`**. When multiple components are generated in parallel, LangGraph routes each through a separate invocation. Understanding Pregel's super-step model clarifies why all parallel `Send` operations happen in a single super-step and why their results are visible to subsequent nodes only after that step completes.

**Practical optimization:** The `step_timeout` parameter on `Pregel` (accessible via `workflow.compile(...).step_timeout = 60_000`) can prevent stuck parallel tasks from blocking the entire pipeline indefinitely. Currently, no timeout is configured on the Conductor graph, meaning a single failing parallel sub-agent can block the pipeline forever.

```python
# pipeline_graph.py — add step timeout
compiled = workflow.compile(checkpointer=checkpointer)
compiled.step_timeout = 120_000  # 2 minutes per super-step
```

This is a P3 because it requires no new LangGraph features — just awareness of the underlying runtime configuration.

---

## 12. Task Cache Decorator (`CachePolicy`) — P3

### What It Is

In the Functional API, `@task` supports a `cache_policy` parameter that caches task results for a configurable TTL. The graph must be compiled (via `@entrypoint`) with a cache store:

```python
from langgraph.cache.memory import InMemoryCache
from langgraph.func import entrypoint, task
from langgraph.types import CachePolicy

@task(cache_policy=CachePolicy(ttl=300))  # 5-minute TTL
def analyze_code_structure(code: str) -> dict:
    """Expensive static analysis — cache when called with same code."""
    return static_analyzer.run(code)

@entrypoint(checkpointer=checkpointer, cache=InMemoryCache())
def verification_pipeline(inputs: dict) -> dict:
    # First call: runs analysis (1–2 seconds)
    result1 = analyze_code_structure(inputs["code"]).result()

    # Second call with same input: returns from cache immediately
    # Stream output includes '__metadata__': {'cached': True}
    result2 = analyze_code_structure(inputs["code"]).result()

    return {"analysis": result1}

# Stream output shows caching in action
for chunk in verification_pipeline.stream({"code": "def foo(): pass"}, stream_mode="updates"):
    print(chunk)
# → {'analyze_code_structure': {...}}
# → {'analyze_code_structure': {...}, '__metadata__': {'cached': True}}
# → {'verification_pipeline': {'analysis': {...}}}
```

### Why It Matters for Meta-Builder

Meta-builder has a `StructuredLLMCache` that caches LLM responses in a custom layer. This is reasonable, but it duplicates infrastructure that LangGraph now provides natively for Functional API workflows.

The specific opportunity: if the IR compiler passes are refactored to use `@task` (Feature #5), those tasks automatically become cacheable via `CachePolicy`. The `validate_ir` and `parse_design` passes are deterministic given the same input — their results can be cached across pipeline runs. This is particularly valuable in the evolutionary optimizer, where many agents share similar base designs and re-validation of unchanged components is redundant.

**Integration note:** `CachePolicy` is only available in the Functional API (via `@task`). It does not apply to `StateGraph` nodes. This is another reason to consider migrating sequential, deterministic pipeline stages to the Functional API.

```python
@task(cache_policy=CachePolicy(ttl=600))  # 10-minute cache for deterministic passes
def validate_ir(ir_dict: dict) -> dict:
    return IRValidator().validate(IR(**ir_dict)).dict()

# Cache is keyed by function name + serialized input
# Same IR validated twice → second call returns instantly
```

---

## Implementation Roadmap

### Phase 1 — Foundation (Sprint 1–2, ~2 weeks)
*Enables everything downstream*

| Task | Files | Effort |
|---|---|---|
| Add `CheckpointerFactory` utility | `utils/checkpointer.py` (new) | 0.5 day |
| Inject checkpointer into `PipelineGraph` | `pipeline_graph.py` | 0.5 day |
| Add `durability="sync"` to Conductor invocation | `conductor.py` | 0.5 day |
| Thread `thread_id` through request context | `conductor.py`, `api/routes.py` | 1 day |
| Configure `LANGGRAPH_CHECKPOINT_BACKEND` env var | `config.py` | 0.5 day |
| Add PostgresSaver for production deployment | `langgraph.json`, `config.py` | 1 day |

**Outcome:** Conductor gains fault tolerance, conversation continuity, and enables all downstream features.

### Phase 2 — Human-in-the-Loop (Sprint 3, ~1 week)
*Requires Phase 1*

| Task | Files | Effort |
|---|---|---|
| Add `pre_generation_gate` node to Conductor | `pipeline_graph.py` | 1 day |
| Wire `interrupt()` to gate node | `pipeline_graph.py` | 0.5 day |
| Update `_generate_human_node()` in GraphFactory | `graph_factory.py` | 1 day |
| Expose `Interrupt` events via streaming API | `conductor.py`, `api/routes.py` | 1 day |
| Add `Command(resume=...)` endpoint | `api/routes.py` | 0.5 day |

**Outcome:** `requires_human_approval` and `Gate` models become operational. Approval gates actually pause the pipeline.

### Phase 3 — Streaming (Sprint 4, ~1 week)

| Task | Files | Effort |
|---|---|---|
| Switch Conductor invocation to `astream()` | `conductor.py` | 1 day |
| Add `get_stream_writer()` calls in generation node | `pipeline_graph.py` | 1 day |
| Add streaming scaffolding to generated agent template | `graph_factory.py` | 1 day |
| Expose SSE or WebSocket streaming endpoint | `api/routes.py` | 1 day |

**Outcome:** Users see real-time build progress. Token-level streaming from planning LLM.

### Phase 4 — Long-term Memory (Sprint 5, ~1 week)

| Task | Files | Effort |
|---|---|---|
| Refactor `PipelineMemory` as LangGraph Store wrapper | `pipeline_memory.py` | 2 days |
| Inject `store` into Conductor nodes | `pipeline_graph.py` | 0.5 day |
| Generate `store: BaseStore` injection in agent templates | `graph_factory.py` | 1 day |
| Wire `MemoryChannel` definitions to Store namespaces | `graph_factory.py` | 1 day |

**Outcome:** User preferences and design history persist across sessions. Generated agents have cross-thread memory.

### Phase 5 — Functional API & Caching (Sprint 6–7, ~2 weeks)

| Task | Files | Effort |
|---|---|---|
| Refactor IR compiler passes to `@task` | `ir_compiler.py` | 2 days |
| Add Functional API generator for `simple_chain` tier | `graph_factory.py` | 2 days |
| Add `CachePolicy` to deterministic IR tasks | `ir_compiler.py` | 0.5 day |
| Add `Command`-based supervisor pattern in GraphFactory | `graph_factory.py` | 1 day |
| Evaluate `langgraph-supervisor` for generated patterns | `graph_factory.py` | 1 day |

**Outcome:** IR compilation gains crash recovery. Simple chain agents are shorter and more readable. Supervisor patterns use `Command` for cleaner routing.

### Phase 6 — Visualization & Polish (Sprint 8, ~1 week)

| Task | Files | Effort |
|---|---|---|
| Add `visualize()` / `visualize_png()` to generated agent template | `graph_factory.py` | 0.5 day |
| Add `get_diagram()` to `PipelineGraph` | `pipeline_graph.py` | 0.5 day |
| Add `step_timeout` to Conductor | `pipeline_graph.py` | 0.5 day |
| Expand `langgraph.json` with all agent graphs for Studio | `langgraph.json` | 0.5 day |
| Add debug endpoint for `get_state()` / `update_state()` | `api/routes.py` | 1 day |

---

## Summary: The Three Changes That Matter Most

If only three things are implemented, they should be, in order:

1. **Checkpointing on the Conductor** — one constructor parameter change in `pipeline_graph.py` and a `thread_id` in the config. Unlocks fault tolerance, human-in-the-loop, time travel, and memory. Without this, every other feature in this document is impossible.

2. **`interrupt()` wired to gate nodes** — connects the existing `requires_human_approval` schema to actual LangGraph behavior. Turns a dormant schema field into a functional human oversight mechanism.

3. **Streaming from the Conductor** — switches `ainvoke()` to `astream()` in the Conductor's execution path. Eliminates the opaque black-box UX and provides real-time build visibility.

Everything else in this document builds on these three. The total effort for items 1–3 is approximately **6–8 developer-days** — a small investment for a step-change in reliability and usability.
