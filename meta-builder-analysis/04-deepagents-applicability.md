# DeepAgents Applicability Analysis: meta-builder-v3

**Project:** meta-builder-v3 (v4.9.0)  
**Generation Tier:** `deep_agent` (one of three tiers: `simple_chain` | `langgraph` | `deep_agent`)  
**Primary File:** `src/generators/deep_agent.py` (DeepAgentGenerator)

---

## Overview

DeepAgents is LangChain's open-source agent harness for long-running, complex, multi-step tasks. Published under MIT license, it provides a batteries-included framework — built on top of LangGraph — that ships with planning tools, filesystem access, sandbox execution, and subagent spawning. An agent created with `create_deep_agent` is, at its core, a compiled LangGraph `StateGraph`, which means it inherits every LangGraph capability (streaming, checkpointing, Studio visualization, Platform deployment) without additional configuration.

Meta-builder-v3 integrates DeepAgents not as a runtime agent framework — the meta-builder itself does not run _as_ a deep agent — but as a **generation target**: one of the three `generation_tier` values in the `WorkflowDesign` IR. When a workflow is classified as `deep_agent` tier, the `DeepAgentGenerator` class takes over from the standard `GraphFactory` to produce an output package optimized for the DeepAgents runtime model rather than for direct LangGraph graph execution.

This creates a layered relationship:
- Meta-builder uses LangGraph for its own runtime
- Meta-builder's GraphFactory generates LangGraph graphs for the `langgraph` tier
- Meta-builder's DeepAgentGenerator generates DeepAgents-compatible harnesses for the `deep_agent` tier

Understanding DeepAgents is therefore essential for understanding what the `deep_agent` tier _is_, what its output structure means, and why the project treats it as architecturally distinct from the `langgraph` tier.

---

## 1. What DeepAgents Is and Its Relationship to LangGraph

### DeepAgents Architecture

DeepAgents is described by LangChain as an "agent harness" — a framework that implements the core tool-calling loop but with opinionated built-in capabilities layered on top. The central entry point is `create_deep_agent`:

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model=your_llm,
    tools=[custom_tool],
    system_prompt="You are a coding assistant.",
    subagents=[specialized_subagent],
    backend=ModalSandbox(sandbox=modal_sandbox),
)
```

The result is a compiled LangGraph `StateGraph` — `create_deep_agent` returns an object that supports `.invoke()`, `.stream()`, `.ainvoke()`, and all other standard LangGraph `Runnable` interfaces.

Built-in capabilities include:
- **`write_todos` / `read_todos`**: Planning tools that enable the agent to create structured task lists before executing
- **Filesystem tools**: `ls`, `read_file`, `write_file`, `edit_file`, `glob`, `grep` — for reading and writing to a configurable backend
- **`task` tool**: Subagent delegation — the main agent can spawn specialized subagents with isolated context windows
- **Auto-summarization**: Compacts older messages when context grows long, preventing overflow on extended sessions
- **Sandbox execution**: When using a sandbox backend, a shell `execute` tool is added for running commands

### The Relationship to LangGraph

DeepAgents does not replace LangGraph — it builds on it. The [Milvus documentation on DeepAgents](https://milvus.io/blog/how-to-build-productionready-ai-agents-with-deep-agents-and-milvus.md) states directly: "Technically, an agent created with `create_deep_agent` is a compiled LangGraph StateGraph." This means:

- DeepAgents agents support LangGraph streaming (`updates`, `messages`, `custom` modes)
- DeepAgents agents support LangGraph checkpointing and human-in-the-loop
- DeepAgents agents are visible in LangGraph Studio
- DeepAgents agents can be deployed on LangGraph Platform via `langgraph.json`

The [LangChain DeepAgents page](https://www.langchain.com/deep-agents) draws the practical distinction: "Use Deep Agents when you want to build an autonomous agent to handle complex, non-deterministic, and long-running tasks. Choose LangGraph when you want low-level control for building stateful, long-running workflows and agents."

The `generation_tier` field in meta-builder's `WorkflowDesign` IR precisely encodes this tradeoff: `langgraph` tier gives the generated agent maximum architectural control; `deep_agent` tier trades control for batteries-included capability.

---

## 2. How Meta-Builder's `deep_agent` Generation Tier Works

### The Three-Tier Model

Meta-builder-v3 defines three generation tiers in `WorkflowDesign.generation_tier`:

| Tier | Output | Use Case |
|------|--------|----------|
| `simple_chain` | A LangChain LCEL chain | Simple, single-pass LLM workflows |
| `langgraph` | A compiled `StateGraph` | Custom topology, full LangGraph control |
| `deep_agent` | A DeepAgents harness package | Complex, long-running, autonomous tasks |

The key insight is that these tiers map to LangChain's own recommended framework selection logic. The `deep_agent` tier is selected by the PlanningAgent when the task analysis indicates:
- Multi-step complexity requiring planning and decomposition
- Large context requirements (file I/O, web research, multi-document synthesis)
- Work delegation across multiple specialized roles
- Long-running or stateful workflows

### Tier Selection in the IR

The `generation_tier` field is set by the PlanningAgent's `select_architecture` node as part of the planning pipeline. The node reads the decomposed requirements and uses LangGraph's structured output to populate a `TopologySpec` that includes the tier. A task described as "build an autonomous research assistant that plans, searches, writes, and revises iteratively" would receive `generation_tier: "deep_agent"` because it maps to DeepAgents' design center.

The `IRMetadata` object on `WorkflowDesign` includes `optimization_passes`, which are run by the IR compiler pipeline before generation. Some passes (e.g., `LoopBoundInsertion`, `ParallelExtraction`) apply to all tiers; others are tier-specific. The `deep_agent` tier triggers harness-specific validation in the generation pipeline.

---

## 3. The DeepAgentGenerator Class

### Location and Role

**`src/generators/deep_agent.py`** contains the `DeepAgentGenerator` class. Its role in the pipeline is:

```
WorkflowDesign (generation_tier="deep_agent")
    → DeepAgentGenerator.generate(design)
        → GraphFactory.build(design)           # structural validation
        → extract_requirements(design)          # comprehensive requirement extraction
        → generate_subagent_source(sub_design)  # recursive, one file per sub-agent
        → generate_harness(design)              # main.py + agent_config.yaml
        → collect_GenerationResult              # in-memory graph + source_files
```

### Pipeline Steps

**Step 1: Structural Validation via GraphFactory**

`DeepAgentGenerator` first calls `GraphFactory.build(design)` to compile the `WorkflowDesign` into a `CompiledStateGraph`. If this compilation fails, the design is structurally invalid — dead nodes, unbound loops, missing edge targets — and generation is aborted. The `GenerationResult.graph` field holds this compiled graph.

Critically: the project notes that "For `deep_agent` tier, `GenerationResult.graph` is a structural validation artifact. The actual runtime execution relies on the DeepAgents SDK harness." This means the compiled LangGraph graph is proof of correctness (LangGraph's compiler validates the topology) but not the runtime artifact. The runtime artifact is the exported harness package.

**Step 2: Requirement Extraction**

`extract_requirements(design)` performs comprehensive analysis of the `WorkflowDesign` to extract:
- All tools required by all nodes and sub-agents
- Memory channel requirements (`MemoryChannel` enum: `shared_state`, `message_passing`, `blackboard`, `episodic`)
- Model requirements per sub-agent
- Backend requirements (filesystem, sandbox, store)
- System prompt requirements per agent
- Context isolation boundaries (which sub-agents need isolated context)

This extraction feeds into both the harness generation and the `agent_config.yaml` construction.

**Step 3: Sub-Agent Source Generation**

For each `sub_agent` in `design.sub_agents`, `DeepAgentGenerator` recursively generates source files. Each sub-agent gets its own directory under `subagents/`:

```
generated_agent/
├── main.py                # DeepAgents harness entry point
├── agent_config.yaml      # Agent configuration
└── subagents/
    ├── research_agent/
    │   ├── agent.py       # Sub-agent definition
    │   └── tools.py       # Sub-agent's tool implementations
    ├── writer_agent/
    │   ├── agent.py
    │   └── tools.py
    └── ...
```

Each `agent.py` file generates a `create_deep_agent(...)` call with the sub-agent's specific system prompt, tools, model, and nested sub-agents (if the sub-agent itself has sub-agents — recursive nesting is supported).

**Step 4: Harness Generation**

The main `main.py` creates the top-level deep agent with all extracted requirements:

```python
# Generated main.py (structure)
from deepagents import create_deep_agent
from subagents.research_agent.agent import research_agent
from subagents.writer_agent.agent import writer_agent
from .tools import all_tools
from .config import load_config

config = load_config("agent_config.yaml")

agent = create_deep_agent(
    model=config["model"],
    tools=all_tools,
    system_prompt=config["system_prompt"],
    subagents=[research_agent, writer_agent],
    backend=config["backend"],
)
```

The `agent_config.yaml` externalizes all configuration:

```yaml
# Generated agent_config.yaml (structure)
model: claude-sonnet-4
system_prompt: |
  You are a research and writing agent. Plan before acting.
  Break complex tasks into steps. Delegate research to research-agent
  and writing to writer-agent.
backend:
  type: filesystem
  root_dir: /workspace
memory_channels:
  - type: episodic
    store: postgres
```

### GenerationResult

The `GenerationResult` object returned by `DeepAgentGenerator.generate()` contains:
- `graph`: The compiled `CompiledStateGraph` (validation artifact)
- `source_files`: Dict mapping file paths to file contents (the harness package)
- `requirements`: Extracted `AgentRequirements` object
- `sandbox_descriptor`: Configuration for the sandbox environment

---

## 4. The Deep Agent Harness Export

### What "Export" Means

`GraphFactory.generate()` does `build()` + `export_source()` + `persist()`. For the `deep_agent` tier, the `export_source` step is handled by `DeepAgentGenerator`, and it produces a significantly richer output than for the `langgraph` tier.

The `langgraph` tier's `export_source` produces Python source code for the `StateGraph` construction. The `deep_agent` tier's export produces a complete deployable package:

| Artifact | Purpose |
|----------|---------|
| `main.py` | Entry point; calls `create_deep_agent(...)` |
| `agent_config.yaml` | Externalizes all configuration |
| `subagents/<name>/agent.py` | Sub-agent definitions |
| `subagents/<name>/tools.py` | Sub-agent tool implementations |
| `requirements.txt` | Python dependencies including `deepagents` |
| `langgraph.json` | LangGraph Platform deployment config |
| `Dockerfile` | Container build for sandbox deployment |

The `Dockerfile` is generated for `deep_agent` tier when the `WorkflowDesign` includes sandbox requirements. It builds a container with the DeepAgents SDK and the agent's dependencies pre-installed, suitable for use as a sandbox backend.

### The Semantics Note

The project's documentation includes a critical note: "For `deep_agent` tier, `GenerationResult.graph` is a structural validation artifact. The actual runtime execution relies on the DeepAgents SDK harness."

This distinction matters for several reasons:

1. **Graph vs. harness**: The compiled LangGraph graph validates that the agent's _topology_ is correct (no unreachable nodes, valid edges). But the DeepAgents runtime adds capabilities (planning tools, filesystem, auto-summarization) that are not represented in the bare LangGraph graph. The actual behavior at runtime will include these built-in capabilities.

2. **Testing**: `AgentGym`'s `DeepAgentRunner` tests the harness (running `main.py` in a sandbox), not the compiled graph directly. The `GraphRunner` tests the compiled graph for structural behavior; the `DeepAgentRunner` tests the full harness behavior.

3. **State schema divergence**: The compiled graph's state schema may differ from the actual runtime state managed by the DeepAgents internals (which include message history, todo list state, file system state). The exported harness's `agent_config.yaml` is the authoritative source for runtime state configuration.

---

## 5. Sub-Agent Isolation and Nested Graph Patterns

### Context Isolation: The Core Value Proposition

The [DeepAgents subagents documentation](https://docs.langchain.com/oss/python/deepagents/subagents) explains why subagents are the core value of the deep agent pattern: "Subagents solve the context bloat problem. When agents use tools with large outputs (web search, file reads, database queries), the context window fills up quickly with intermediate results. Subagents isolate this detailed work — the main agent receives only the final result, not the dozens of tool calls that produced it."

Meta-builder's `WorkflowDesign.sub_agents: list[WorkflowDesign]` field is a direct encoding of this pattern. Each sub-agent in the design represents a context isolation boundary: work delegated to a sub-agent does not pollute the main agent's context.

### CompiledSubAgent Pattern

DeepAgents supports two sub-agent patterns:
1. **Dictionary-based**: A plain Python dict with `name`, `description`, `system_prompt`, `tools`, and optional `model`
2. **CompiledSubAgent**: A wrapper around a pre-built compiled LangGraph graph

`DeepAgentGenerator` generates code that uses both patterns:
- For sub-agents with `generation_tier: "langgraph"` in their own `WorkflowDesign`, it generates a `CompiledSubAgent` wrapping the GraphFactory-built graph
- For sub-agents with `generation_tier: "deep_agent"`, it generates a nested `create_deep_agent()` call (full deep agent as a sub-agent)
- For simple delegation sub-agents, it generates dictionary-based sub-agent specs

This creates a recursive architecture: a `deep_agent` tier output can contain `langgraph` tier sub-agents as `CompiledSubAgent` instances, allowing fine-grained topology control at the sub-agent level while maintaining the harness at the top level.

### MemoryChannel Mapping

The `WorkflowDesign.MemoryChannel` enum specifies how agents share information:

| MemoryChannel | DeepAgents Implementation |
|---|---|
| `shared_state` | Top-level StateGraph state fields shared across agents |
| `message_passing` | `task()` tool invocation with structured return value |
| `blackboard` | LangGraph Store backend (key-value shared across agents) |
| `episodic` | LangGraph Store with episodic memory indexing |

`DeepAgentGenerator` maps each `MemoryChannel` value to the appropriate DeepAgents backend configuration, generating the corresponding backend initialization code in `main.py`.

### Skill Isolation

DeepAgents' subagents support **skill isolation**: a sub-agent's skills (filesystem, tools, specialized capabilities) are fully isolated from the main agent. This maps to meta-builder's `WorkflowNode.available_tools` per-node specification. Each sub-agent in the generated harness receives only the tools specified in its corresponding `WorkflowNode` design, preventing tool leakage across context boundaries.

---

## 6. Sandbox Execution Model

### What DeepAgents Sandboxes Are

The [DeepAgents sandbox documentation](https://docs.langchain.com/oss/python/deepagents/sandboxes) describes sandboxes as "backends that define the environment where the agent operates." A sandbox backend provides:
- Filesystem isolation from the host system
- Shell command execution via the `execute` tool
- All standard filesystem tools (`ls`, `read_file`, `write_file`, etc.)

Supported sandbox providers include Modal, Daytona, E2B, Runloop, and Deno. Each runs agent code in an isolated container or VM, preventing the agent from accessing the host filesystem, environment variables, or other processes.

### Meta-Builder's SandboxFactory and SandboxPool

Meta-builder-v3's AgentGym module contains:

**`SandboxFactory`** — Creates Modal or Docker sandboxes for testing generated agents. For `deep_agent` tier candidates, `SandboxFactory` creates the appropriate sandbox backend (matching the `backend` type in `agent_config.yaml`) and initializes it with the generated agent package.

**`SandboxPool`** — Manages a pool of reusable sandbox instances. Creating a new sandbox (especially a Modal or Docker container) has non-trivial startup latency. The pool pre-warms sandbox instances and assigns them to test runs on demand. When a test run completes, the sandbox is either returned to the pool (if clean) or terminated (if the agent left a dirty state).

**`DeepAgentRunner`** — Uses a sandbox from the pool to execute the generated agent harness. It:
1. Mounts the generated source files into the sandbox
2. Runs `python main.py` with a test prompt from `SyntheticDataGenerator`
3. Captures stdout/stderr and any file system artifacts
4. Measures execution time, token usage (via `ProviderUsageTracker`), and output quality
5. Returns metrics for the fitness function

**`sandbox_descriptor`** in `GenerationResult` — Specifies which sandbox provider and configuration the agent was designed for. The `DeepAgentRunner` uses this to initialize the correct sandbox backend for testing.

### Two Sandbox Patterns

The DeepAgents documentation identifies two deployment patterns:
1. **Agent uses sandbox** (default): The agent runs on the host and uses the sandbox for filesystem/shell operations. API keys stay on the host (secure).
2. **Agent in sandbox**: The entire agent runs inside the sandbox, communicating over network. More isolated but requires API keys inside the sandbox (security risk).

`DeepAgentGenerator` generates code for Pattern 1 by default. The generated `main.py` instantiates the sandbox backend and passes it to `create_deep_agent(backend=...)`, keeping the agent's LLM calls on the host while isolating filesystem and shell operations inside the sandbox.

---

## 7. AgentGym and DeepAgents Testing Patterns

### AgentGym Architecture

AgentGym is meta-builder's evaluation harness for generated agents. It contains:

| Component | Role |
|-----------|------|
| `SyntheticDataGenerator` | Creates test scenarios using LLM-driven adaptive difficulty (Search Self-Play) |
| `GraphRunner` | Tests `langgraph` tier agents in-process |
| `DeepAgentRunner` | Tests `deep_agent` tier agents in sandbox |
| `ModuleRunner` | Runs module-level unit tests |
| `SandboxFactory` | Creates sandboxes on demand |
| `SandboxPool` | Manages reusable sandbox pool |
| `ProviderUsageTracker` | Tracks token/API usage across all providers |

### SyntheticDataGenerator: Search Self-Play

The `SyntheticDataGenerator` uses a Search Self-Play pattern: an LLM generates test scenarios, evaluates the candidate agent's performance on those scenarios, and uses the evaluation results to generate harder scenarios in subsequent iterations. This is an adversarial test generation loop that adapts to the agent's capabilities, probing for weaknesses.

For `deep_agent` tier agents, scenario generation must account for the agent's multi-step nature:
- Scenarios need to be complex enough to trigger the planning mechanism (`write_todos`)
- Scenarios should include tasks that benefit from subagent delegation
- Scenarios may involve long context (large file reads, multi-document synthesis) that tests the auto-summarization capability

### DeepAgentRunner Evaluation Loop

For each test scenario:

1. **Initialize sandbox**: `SandboxPool.acquire()` returns a pre-warmed sandbox
2. **Mount sources**: Inject the generated agent package into the sandbox filesystem
3. **Run agent**: Execute `python main.py` with the scenario prompt
4. **Capture outputs**: Collect terminal output, file artifacts, and LangGraph stream events
5. **Score**: Evaluate against the scenario's expected outcomes (task completion, correctness, efficiency)
6. **Track usage**: `ProviderUsageTracker` records token consumption and API costs
7. **Return sandbox**: Return clean sandbox to pool, or terminate if dirty

The fitness score computed from these evaluations drives the evolutionary optimization loop.

### Testing Deep Agents Specifically

DeepAgents' planning-first behavior creates specific testing challenges. A well-functioning deep agent should:
- Always call `write_todos` before beginning complex tasks (testable: check if first tool call is `write_todos`)
- Delegate appropriately to subagents rather than handling everything in the main context (testable: check `lc_agent_name` metadata in stream)
- Not overflow context on long tasks (testable: check `ProviderUsageTracker` token counts)

`DeepAgentRunner` tests for these behaviors by inspecting the LangGraph stream events from the executed agent, using the `lc_agent_name` metadata field in message events to track which agent (main or which subagent) produced each output.

---

## 8. The Research Department as a Deep Agent Architecture

### Architecture Overview

Meta-builder's Research Department is the best concrete example of a deep agent architecture running within the project itself:

| Component | Deep Agent Role |
|-----------|----------------|
| `ResearchOrchestrator` | Main agent / supervisor |
| `ScoutMonitor` | Sub-agent: monitors arXiv, Papers with Code |
| `VerdictPanel` | Sub-agent: multi-judge evaluation |
| `DecisionGate` | Sub-agent: go/no-go decisions |
| `ExperimentRunner` | Sub-agent: runs experiments with tracking |
| `tool_acquisition` | Sub-agent: discovers and integrates new tools |

This maps directly to the DeepAgents pattern:
- `ResearchOrchestrator` is the top-level agent (equivalent to `create_deep_agent(subagents=[...])`)
- Each sub-component is a specialized sub-agent with its own context window and tools
- The orchestrator delegates via task assignments, receiving only synthesized results

### Context Isolation in Practice

`ScoutMonitor` monitors research sources and may process hundreds of paper abstracts per run. If this context lived in the `ResearchOrchestrator`'s context window, it would overflow rapidly. By isolating `ScoutMonitor` as a sub-agent, the orchestrator receives only "N relevant papers found: [titles and relevance scores]" rather than the full text of all abstracts.

`VerdictPanel`'s multi-judge evaluation involves multiple LLM calls with lengthy inputs. Isolating this in a sub-agent keeps the orchestrator's context clean. The panel's deliberation (multiple judge perspectives, rebuttals, consensus) stays inside `VerdictPanel`'s context; the orchestrator receives only the final verdict.

### Self-Reference: The Meta-Builder Generates Architectures Like Itself

There is a notable self-referential quality to the Research Department: it is essentially a deep agent architecture (supervisor + specialized sub-agents with context isolation), and the meta-builder is designed to generate exactly this kind of architecture via the `deep_agent` tier. The Research Department is simultaneously a user of the architecture pattern and a concrete proof that the pattern works.

When the PlanningAgent analyzes a task description like "build an autonomous research assistant," the architecture it designs (`WorkflowDesign`) will structurally resemble the Research Department itself: a supervisor-style orchestrator with scouts, evaluators, decision makers, and execution runners as sub-agents.

---

## 9. Evolution of Agents as a DeepAgents Concept

### The Evolutionary Optimizer Loop

The evolutionary optimization loop in meta-builder-v3:

```
1. PlanningAgent → WorkflowDesign (initial population)
2. GraphFactory / DeepAgentGenerator → CandidateAgent.graph (compiled)
3. AgentGym → fitness score (tested in sandbox)
4. EvolutionaryOptimizer → mutate / select (next generation)
5. goto 2
```

Each `CandidateAgent` contains:
- `design`: The `WorkflowDesign` (the genome)
- `graph`: The compiled `CompiledStateGraph` (proof of structural validity)
- `fitness`: Evaluation score from AgentGym
- `sandbox_descriptor`: The sandbox environment specification

### Evolution of Deep Agents Specifically

For `deep_agent` tier candidates, mutation operates at the `WorkflowDesign` level (the IR), not on the DeepAgents source code. Mutations include:

- Adding or removing sub-agents from `design.sub_agents`
- Changing sub-agent `system_prompt` descriptions
- Modifying tool assignments per sub-agent
- Changing `MemoryChannel` types
- Adjusting `orchestration_pattern` (supervisor → swarm, etc.)
- Modifying context isolation boundaries

The `DeepAgentGenerator` re-runs on the mutated design to produce a new harness package, which is then tested in `DeepAgentRunner`. This means the evolutionary loop can discover optimal sub-agent decompositions without a human ever designing them.

### Fitness Landscape for Deep Agents

The fitness function for `deep_agent` tier candidates needs to capture properties that are specific to the deep agent architecture:

- **Planning adherence**: Does the agent call `write_todos` appropriately?
- **Delegation quality**: Are sub-agents used for appropriate tasks? (Not delegating trivial tasks; delegating context-heavy tasks)
- **Context efficiency**: Does the main agent's context stay manageable? (Token count in main agent vs. sub-agents)
- **Task completion rate**: Does the agent complete multi-step tasks successfully?
- **Subagent result quality**: Do sub-agents return useful, synthesized results to the main agent?

These metrics are computed by `DeepAgentRunner` through stream inspection and are aggregated into the fitness score that drives selection pressure.

---

## 10. WorkflowDesign IR as Bridge Between Design and Execution

### The IR's Central Role

`WorkflowDesign` is meta-builder's intermediate representation (IR) — the language-agnostic specification that bridges natural language requirements (from VisionAgent) to executable artifacts (from GraphFactory / DeepAgentGenerator). For the `deep_agent` tier, the IR-to-execution bridge is particularly important because the execution model (DeepAgents SDK) is richer and more opinionated than bare LangGraph.

### IR Compiler Passes for Deep Agents

The IR compiler pipeline runs optimization and validation passes before generation. For `deep_agent` tier designs:

**`LoopBoundInsertion`** — Adds iteration caps to any cyclic patterns in the WorkflowDesign. Deep agents naturally have cycles (the planning → execution → evaluation → replan loop), and unbounded cycles risk runaway execution and runaway costs.

**`DeadNodeElimination`** — Removes unreachable nodes from the design. For deep agents, a sub-agent that is never referenced by the main agent's `task()` tool call would be generated but never invoked. Removing it saves generation work and produces cleaner harnesses.

**`StateFieldPruning`** — Removes unused state fields. Deep agent state schemas can become bloated if the PlanningAgent included fields for capabilities later de-scoped. Pruning ensures the generated state class is minimal.

**`ParallelExtraction`** — Identifies sub-agents that can run concurrently (no data dependencies between them). Annotates the topology with parallelism hints that `DeepAgentGenerator` uses to emit async subagent invocation patterns.

**`CostEstimation`** — Annotates nodes with estimated token costs by provider tier. For deep agents, this is particularly important because sub-agent invocations multiply token costs. A design with 5 sub-agents each making 10 LLM calls per invocation has a very different cost profile than a design with 2 sub-agents making 3 calls each. The `CostEstimation` pass annotates this in `IRMetadata` so the evolutionary optimizer can select cost-efficient candidates alongside high-fitness ones.

### WorkflowDesign Fields That Map Directly to DeepAgents Concepts

| WorkflowDesign Field | DeepAgents Concept |
|---|---|
| `generation_tier: "deep_agent"` | Use `create_deep_agent()` as the harness |
| `sub_agents: list[WorkflowDesign]` | `subagents=[...]` parameter in `create_deep_agent()` |
| `orchestration_pattern: "supervisor"` | Main agent delegates via `task()` tool with explicit routing |
| `orchestration_pattern: "swarm"` | Peer-to-peer handoffs via subagent-to-subagent delegation |
| `orchestration_pattern: "hierarchical"` | Nested `create_deep_agent()` calls with their own sub-agents |
| `MemoryChannel.shared_state` | Shared StateGraph state keys |
| `MemoryChannel.episodic` | LangGraph Store with episodic indexing |
| `WorkflowNode.type == "human"` | `interrupt()` call in DeepAgents node |
| `WorkflowEdge.requires_human_approval` | Conditional interrupt at edge crossing |
| `state_fields[*].reducer == "add_messages"` | `messages: Annotated[list, add_messages]` in generated StateGraph |

### Why the IR Is the Right Abstraction Level

The IR operates at a level above both LangGraph primitives and DeepAgents configuration. This is architecturally correct: the PlanningAgent doesn't need to know that `fan_out_parallel` maps to `Send` in LangGraph and to async sub-agent dispatch in DeepAgents. It specifies the semantic intent (parallelize these tasks), and the generator layer (GraphFactory for `langgraph` tier, DeepAgentGenerator for `deep_agent` tier) handles the mapping to the appropriate primitive.

This separation means:
1. The PlanningAgent can design architectures without knowing generator-specific idioms
2. Changing the DeepAgents SDK version requires updating only `DeepAgentGenerator`, not the IR schema
3. The same `WorkflowDesign` could potentially generate both a `langgraph` tier graph and a `deep_agent` tier harness — different execution artifacts from the same design, useful for comparing their behavior in AgentGym

---

## Summary: DeepAgents Integration Map

| Aspect | Meta-Builder Implementation | DeepAgents Concept |
|--------|---------------------------|-------------------|
| Generation target | `generation_tier: "deep_agent"` | `create_deep_agent()` harness |
| Generator class | `src/generators/deep_agent.py` | DeepAgentGenerator |
| Structural validation | `GraphFactory.build()` on design | LangGraph compilation |
| Runtime artifact | Exported harness package | `main.py` + `agent_config.yaml` |
| Sub-agent design | `WorkflowDesign.sub_agents` | `subagents` parameter |
| Context isolation | Per-sub-agent source files | DeepAgents `task()` tool |
| Sandbox testing | `AgentGym.DeepAgentRunner` | Sandbox backends (Modal, Daytona) |
| Memory channels | `MemoryChannel` enum in IR | StateBackend, StoreBackend, CompositeBackend |
| Evolutionary optimization | `EvolutionaryOptimizer` on IR | Mutation of `WorkflowDesign` fields |
| Research Department | ResearchOrchestrator + sub-components | Supervisor + specialized sub-agents |
| IR compiler passes | LoopBoundInsertion, CostEstimation | Pre-generation safety and cost optimization |
| Deployment | `langgraph.json` in generated package | LangGraph Platform deployment |

Meta-builder-v3's `deep_agent` tier is, in essence, a code generator that writes DeepAgents programs from a structured intermediate representation. The `WorkflowDesign` IR captures the semantic intent of a complex autonomous agent; `DeepAgentGenerator` translates that intent into a deployable DeepAgents harness; and `AgentGym` evaluates the resulting agent's behavior in an isolated sandbox environment. The evolutionary optimizer closes the loop by mutating the IR and re-running this pipeline until the fitness criteria are met.
