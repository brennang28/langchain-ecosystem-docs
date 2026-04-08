# Meta-Builder-v3 Architecture: Comprehensive Technical Overview

**Version:** 4.9.0  
**Classification:** AI-Native Orchestration Compiler  
**Date:** April 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Conceptual Foundation](#2-conceptual-foundation)
3. [High-Level System Architecture](#3-high-level-system-architecture)
4. [The Conductor: Hub-and-Spoke Orchestration](#4-the-conductor-hub-and-spoke-orchestration)
5. [ConductorState: The Central Data Contract](#5-conductorstate-the-central-data-contract)
6. [Core Agent Roster](#6-core-agent-roster)
7. [The WorkflowDesign Intermediate Representation](#7-the-workflowdesign-intermediate-representation)
8. [RAG System: Quad-Hybrid Ensemble Retrieval](#8-rag-system-quad-hybrid-ensemble-retrieval)
9. [Evolutionary Optimization System](#9-evolutionary-optimization-system)
10. [IR Compiler Passes](#10-ir-compiler-passes)
11. [Generation Tiers](#11-generation-tiers)
12. [LLM Factory: Multi-Provider Abstraction](#12-llm-factory-multi-provider-abstraction)
13. [Tool Registry](#13-tool-registry)
14. [Safety and Telemetry Middleware](#14-safety-and-telemetry-middleware)
15. [Research Department](#15-research-department)
16. [Agent Gym and Sandbox Execution](#16-agent-gym-and-sandbox-execution)
17. [Studio: Visualization and Inspection](#17-studio-visualization-and-inspection)
18. [End-to-End Data Flow Diagram](#18-end-to-end-data-flow-diagram)
19. [Design Principles and Architectural Tradeoffs](#19-design-principles-and-architectural-tradeoffs)

---

## 1. Executive Summary

Meta-builder-v3 is an **AI-native orchestration compiler**: a system that accepts a plain-English description of the AI agent you want to build and then — without manual coding — designs, builds, tests, and refines that agent for you. Think of it as a compiler where the source language is natural language and the output is a working, deployable AI agent with full LangGraph state machine code, proper tool integrations, and an evaluated fitness score.

The system sits at the intersection of several cutting-edge ideas in applied AI:

- **Compiler theory**: meta-builder-v3 models agent construction as a compilation pipeline with IR (Intermediate Representation) passes, optimization stages, and code generation backends — exactly analogous to how LLVM takes high-level C++ and produces optimized machine code.
- **Multi-agent orchestration**: The system is itself a multi-agent system. A deterministic hub (the Conductor) dispatches work to specialized agent spokes, each of which may itself be a LangGraph sub-agent.
- **Evolutionary optimization**: A genetic algorithm can evolve agents across generations, selecting for quality, cost-efficiency, and speed.
- **Retrieval-augmented design**: A sophisticated quad-hybrid RAG system grounds all design decisions in curated knowledge bases about agent patterns, best practices, and architecture guides.

The end product is not just a description or a plan — it is executable Python code that can run immediately in Modal or Docker sandboxes, along with evaluation scores, design rationales, and provenance citations.

**Key metric:** The Conductor itself makes **zero** LLM calls. Every LLM call is delegated to a spoke agent. The hub is a deterministic, inspectable state machine — making the system's control flow auditable and reliable even as individual agents use probabilistic language models.

---

## 2. Conceptual Foundation

### The "Compiler" Metaphor

A traditional compiler works like this:

```
Source Code → Lexer/Parser → AST → Semantic Analysis → IR → Optimizer → Code Generator → Binary
```

Meta-builder-v3 maps this structure onto AI agent construction:

```
Natural Language Request
    → RequestProcessor (parsing / intent extraction)
    → PlanningAgent (semantic analysis / WorkflowDesign IR)
    → IR Compiler Passes (optimization)
    → GraphFactory / DeepAgentGenerator (code generation)
    → VerificationAgent (correctness checking)
    → EvaluatorAgent (quality scoring)
    → Evolutionary Optimizer (iterative refinement)
    → Deployable Agent (executable output)
```

Each stage has a well-defined input and output type, stages are composable and replaceable, and the whole pipeline can be inspected, interrupted, or replayed at any point.

### The "Hub-and-Spoke" Pattern

The Conductor implements the **hub-and-spoke** pattern from distributed systems. In this pattern:

- A **hub** (the Conductor) holds all shared state and makes all routing decisions.
- **Spokes** (agents) perform specialized work in isolation and return results to the hub.
- Spokes have no direct communication with each other — they only see state they need.

This contrasts with peer-to-peer multi-agent topologies where agents pass messages to each other, which can lead to emergent, hard-to-debug behaviors. The hub-and-spoke pattern trades some flexibility for predictability: you can always read the ConductorState to know exactly where the system is and why.

### Deterministic vs. Probabilistic Zones

Meta-builder-v3 deliberately separates deterministic control flow from probabilistic LLM reasoning:

| Zone | Examples | Characteristics |
|------|----------|-----------------|
| Deterministic | Conductor routing, gate evaluation, score comparison | Reproducible, auditable, fast |
| Probabilistic | Agent LLM calls, RAG retrieval, EvaluatorAgent scoring | Creative, flexible, requires validation |

The system enforces quality at every probabilistic boundary by running the deterministic gate logic after each LLM call — if an agent's output doesn't pass the gate, the Conductor resets state and replans.

---

## 3. High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          META-BUILDER-V3 SYSTEM                         │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     CONDUCTOR (StateGraph)                        │   │
│  │                    Hub-and-Spoke Orchestrator                     │   │
│  │                     Makes ZERO LLM calls                         │   │
│  └──────────────┬───────────────────────────────────┬───────────────┘   │
│                 │                                   │                   │
│        Dispatch │                         Return    │                   │
│                 ▼                                   │                   │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │                      AGENT SPOKES                            │       │
│  │                                                              │       │
│  │  RequestProcessor  PlanningAgent   GraphFactory              │       │
│  │  EvaluatorAgent    VerificationAgent  CritiqueRefiner        │       │
│  │  RemediationAgent  VisionAgent     ResearchOrchestrator      │       │
│  └──────────────────────────┬───────────────────────────────────┘       │
│                             │                                           │
│                    Use      │                                           │
│                             ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    INFRASTRUCTURE LAYER                          │    │
│  │                                                                  │    │
│  │  LLM Factory     RAG System (Quad-Hybrid)   Tool Registry       │    │
│  │  Safety MW       Telemetry (Langfuse)        AgentGym           │    │
│  │  IR Compiler     Evolutionary Optimizer      LanceDB/BM25        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

The system receives a natural-language agent description at the top, runs it through the Conductor's phase pipeline, and emits a fully-tested agent as output. Every component in the infrastructure layer is accessed by agents through well-defined interfaces — no agent reaches directly into another agent's internals.

---

## 4. The Conductor: Hub-and-Spoke Orchestration

### What the Conductor Is

The Conductor is the central orchestrator — a LangGraph `StateGraph` that implements the compilation pipeline as a deterministic state machine. It is the only component that knows the complete system state at all times. Every other component (agents, gates, generators) is a node in this graph or a callable dispatched from one of its nodes.

Crucially, the Conductor **does not reason**. It cannot decide "hmm, maybe we should skip verification this time." It follows a fixed graph topology, routing based on numeric scores and boolean gate flags produced by agents, not by its own LLM calls.

### Phase Pipeline

The Conductor progresses through these phases in order:

```
initialize
    │
    ▼
process_request
    │
    ▼
plan_workflow
    │
    ▼
generate_code
    │
    ▼
verify_output
    │
    ▼
done (END)
```

Each phase is a node in the StateGraph. Edges between phases can be:
- **Direct** (unconditional): always proceed to next phase
- **Conditional**: route to next phase OR to a replan entry point based on gate scores
- **Reset/Replan**: gate failure at `verify_output` can reset state and return to `plan_workflow`

### Replan Feedback Loop

The replan loop is the system's self-correction mechanism:

```
plan_workflow → generate_code → verify_output
       ▲                              │
       │      score < threshold?      │
       └──────── REPLAN GATE ─────────┘
```

When `VerificationAgent` or `EvaluatorAgent` returns a score below the quality threshold:

1. The Conductor's gate logic fires.
2. Downstream mutable state fields are reset (clearing the failed plan and generated code).
3. The feedback from the evaluation is written into a designated feedback field in `ConductorState`.
4. The phase pointer is reset to `plan_workflow`.
5. `PlanningAgent` receives the feedback on its next invocation and produces an improved design.

This loop can iterate multiple times without human intervention, automatically improving output quality.

### LangGraph Primitives Used

```python
from langgraph.graph import StateGraph, END
from langchain_core.runnables import RunnableConfig

# Conductor graph construction pattern
conductor_graph = StateGraph(ConductorState)
conductor_graph.add_node("initialize", initialize_node)
conductor_graph.add_node("process_request", process_request_node)
conductor_graph.add_node("plan_workflow", plan_workflow_node)
conductor_graph.add_node("generate_code", generate_code_node)
conductor_graph.add_node("verify_output", verify_output_node)

# Conditional edge for replan
conductor_graph.add_conditional_edges(
    "verify_output",
    route_after_verify,  # deterministic routing function — no LLM
    {
        "done": END,
        "replan": "plan_workflow",
    }
)
```

The routing function `route_after_verify` is pure Python — it reads `state["verification_score"]` and returns a string key. No language model is involved in this decision.

---

## 5. ConductorState: The Central Data Contract

### Design Philosophy

`ConductorState` is a Python `TypedDict` that defines the complete shared memory of the compilation pipeline. Every piece of data that flows between phases lives here. Agents read from it (their relevant slice), do their work, and write back to it (their output fields only).

The state is explicitly divided into two zones with different mutation rules:

### Immutable Zone

Fields written once at initialization, never changed:

```python
class ConductorState(TypedDict):
    # === IMMUTABLE ZONE ===
    # Set during initialize phase, never overwritten
    raw_request: str                    # Original user description
    domain_docs: Optional[str]          # Optional domain context provided by user
    session_id: str                     # Unique ID for this compilation run
    config: BuilderConfig               # System configuration snapshot
    created_at: datetime                # Timestamp
```

The immutable zone provides a stable anchor. When a replan loop fires, agents can always look back at `raw_request` to re-derive intent — they don't lose the original goal even after multiple refinement cycles.

### Mutable Zone

Fields updated as the pipeline progresses:

```python
    # === MUTABLE ZONE ===
    # Updated by phases, may be reset during replan

    # Phase tracking
    current_phase: str                  # Which phase is active
    phase_history: List[str]            # Audit trail of all phase transitions
    replan_count: int                   # Number of replan loops so far

    # Request processing output (set by RequestProcessor)
    processed_request: Optional[ProcessedRequest]  # Structured intent

    # Planning output (set by PlanningAgent) — RESET on replan
    workflow_design: Optional[WorkflowDesign]      # The IR

    # Generation output (set by GraphFactory/DeepAgentGenerator) — RESET on replan
    generated_code: Optional[str]                  # Python source
    generated_graph: Optional[Any]                 # In-memory LangGraph object

    # Verification output (set by VerificationAgent)
    verification_result: Optional[CritiqueResult]
    verification_score: Optional[float]            # 0.0–1.0

    # Evaluation output (set by EvaluatorAgent)
    evaluation_result: Optional[EvaluationResult]
    evaluation_score: Optional[float]              # 0.0–1.0

    # Feedback for replan (written by gate logic, read by PlanningAgent)
    replan_feedback: Optional[str]

    # Final output
    output_artifact: Optional[BuildArtifact]       # Everything the user gets back
```

### Immutability Enforcement

The immutable zone isn't enforced by a language-level mechanism — it's a convention enforced by code review and agent contracts. Each agent receives only the state slice it needs and returns only the fields it's authorized to update. The Conductor merges agent returns back into the full state using shallow update semantics.

This is analogous to how Redux reducers work: each action updates a slice of state, the central store owns the canonical copy, and past state can always be replayed.

---

## 6. Core Agent Roster

### 6.1 RequestProcessor

**Role:** Translate raw user text into structured machine-readable intent.

**Position in pipeline:** First spoke called — `process_request` phase.

**Input:** `raw_request: str`, `domain_docs: Optional[str]`

**Output:** `ProcessedRequest` containing:

```python
class ProcessedRequest(BaseModel):
    intention: IntentionObject          # What the user wants the agent to do
    integrations: IntegrationManifest   # External APIs, tools, DBs identified
    topology: TopologySpec              # Recommended architecture pattern
    complexity_estimate: str            # simple / moderate / complex
    rag_tier_recommendation: RagTier    # How much retrieval to use in planning
```

**How it works:**

The RequestProcessor uses `llm_client.with_structured_output(ProcessedRequest)` — binding a Pydantic schema to the LLM so that it produces a validated, typed object rather than raw text. The LLM is prompted to act as a requirements analyst: it identifies the intent, detects any mentioned external services (GitHub, Slack, databases), estimates complexity, and recommends which topology pattern (sequential, parallel, map-reduce, etc.) fits the use case.

If `domain_docs` are provided, they are included in the context — allowing the user to guide the system with domain-specific knowledge (API documentation, business rules, etc.).

**Why this matters:** Every downstream agent works with the structured `ProcessedRequest` rather than the original freeform text. This normalizes the input early and reduces ambiguity propagation.

---

### 6.2 PlanningAgent

**Role:** Convert structured intent into a detailed `WorkflowDesign` — the system's central intermediate representation.

**Position in pipeline:** `plan_workflow` phase.

**Input:** `ProcessedRequest`, `replan_feedback` (if replanning)

**Output:** `WorkflowDesign` object

**Internal Architecture:**

The PlanningAgent is itself a LangGraph sub-graph with five internal nodes:

```
retrieve_guides
      │
      ▼
decompose
      │
      ▼
select_architecture
      │
      ▼
detail_design
      │
      ▼
validate
```

Each step is purpose-built:

| Node | Responsibility |
|------|---------------|
| `retrieve_guides` | Fetches relevant architecture guides from the RAG system based on the request's complexity and topology |
| `decompose` | Breaks the intent into discrete functional requirements and agent responsibilities |
| `select_architecture` | Chooses the appropriate topology pattern and generation tier |
| `detail_design` | Populates all `WorkflowNode` and `WorkflowEdge` objects with full specifics |
| `validate` | Internal consistency check — ensures edges reference valid nodes, required fields are present |

**Strategies:**

Two detail strategy classes handle different topologies:

- `SingleAgentDetailStrategy`: Used for sequential or simple conditional workflows. Generates a linear set of nodes with straightforward edges.
- `MultiAgentDetailStrategy`: Used for fan-out parallel, map-reduce, or cyclic agentic topologies. Generates sub-graph structures with parallel branches, aggregation nodes, and cycle management.

**RAG Integration:**

The PlanningAgent has a `guide_retriever` — a GuideRetriever instance pre-configured with the correct `RagTier`. During `retrieve_guides`, it queries the knowledge base for architecture patterns relevant to the current request. Retrieved guides arrive with citations, which are preserved in the `WorkflowDesign` metadata.

**Citation Mode:**

When `citation_mode=True`, the PlanningAgent's `detail_design` step annotates each design decision with which retrieved guide influenced it. This gives full provenance from user request → retrieved knowledge → design choice.

**Fallback LLM:**

The PlanningAgent accepts an optional `fallback_llm` — a cheaper or faster model used if the primary LLM fails or if the request is simple enough not to warrant the full model.

---

### 6.3 GraphFactory

**Role:** Materialize a `WorkflowDesign` IR object into a runnable LangGraph `StateGraph` (in-memory) and/or exportable Python source code.

**Position in pipeline:** `generate_code` phase (for `langgraph` and `deep_agent` tiers).

**Input:** `WorkflowDesign`

**Output:** `generated_graph` (StateGraph instance), `generated_code` (Python source string)

**Core Capabilities:**

1. **Dynamic TypedDict Construction**  
   GraphFactory reads `WorkflowDesign.state_fields` and `WorkflowDesign.state_field_specs` to dynamically construct a `TypedDict` class at runtime. This TypedDict becomes the state type for the generated StateGraph.

2. **Per-Node ToolNode Creation**  
   For each `WorkflowNode` with `type="tool"` or with tools listed, GraphFactory instantiates a `ToolNode` using LangGraph's prebuilt `ToolNode`. It resolves tool names through the `ToolRegistry`.

3. **Conditional Edge Wiring**  
   `WorkflowEdge` objects with `condition`, `condition_field`, or `condition_map` attributes are translated into LangGraph conditional edges. The `condition_map` becomes the routing dictionary.

4. **Topology Pattern Handlers:**

| Pattern | Implementation |
|---------|----------------|
| `sequential` | Linear edges between nodes |
| `fan_out_parallel` | Uses `langgraph.types.Send` for parallel dispatch |
| `map_reduce` | Fan-out with Send + aggregator node |
| `conditional_routing` | Conditional edges with routing functions |
| `cyclic_agentic` | Cycle edges with maximum iteration guards |

5. **Multi-Agent Sub-Graph Isolation**  
   When `WorkflowDesign.sub_agents` is populated, GraphFactory creates isolated sub-graphs for each sub-agent and compiles them as nested graphs within the parent graph. This enforces state scope isolation between sub-agents.

6. **Source Code Export**  
   In addition to the in-memory graph, GraphFactory can emit clean, readable Python source code that recreates the graph — suitable for the user to inspect, modify, and deploy independently.

---

### 6.4 EvaluatorAgent

**Role:** LLM-as-Judge for quality assessment — evaluates both agent runtime outputs against test scenarios and workflow design quality.

**Position in pipeline:** Can be called after `plan_workflow` (design evaluation) and after `verify_output` (execution evaluation). Also used extensively in the Evolutionary Optimizer.

**Output type:**

```python
class EvaluationResult(BaseModel):
    score: float                        # 0.0–1.0 overall
    reasoning: str                      # Explanation of score
    success: bool                       # Passed quality threshold?
    feedback: str                       # Actionable suggestions for improvement
    rubric_scores: Dict[str, float]     # Per-dimension scores
```

**Rubric Dimensions (for agent fitness):**

- Task completion accuracy
- Reasoning quality
- Tool use appropriateness
- Response coherence
- Safety compliance

**Design Evaluation Mode:**

When evaluating a `WorkflowDesign` object, the EvaluatorAgent uses a different rubric:

- Architectural soundness
- Node granularity (not too coarse, not too fine)
- Edge logic correctness
- Tool selection appropriateness
- State field completeness

**Why LLM-as-Judge?**

The EvaluatorAgent uses an LLM because correctness in agent design is often subjective and context-dependent — there's no single ground truth. An LLM judge can evaluate "is this the right tool for this task?" in a way that a rule-based system cannot. The tradeoff is that LLM judges can themselves be biased or inconsistent; the system mitigates this by having the `VerdictPanel` (Research Department) use multiple judges for critical decisions.

---

### 6.5 VerificationAgent

**Role:** Validate generated code and graphs for correctness — syntactic, structural, and behavioral.

**Position in pipeline:** `verify_output` phase.

**Output type:**

```python
class CritiqueResult(BaseModel):
    overall_score: float                # 0.0–1.0
    passes: List[str]                   # Things that are correct
    issues: List[str]                   # Problems found
    strengths: List[str]                # Notable good qualities
```

**Verification Layers:**

1. **Syntax Check**: `ast.parse()` on generated Python code — catches syntax errors before any execution.
2. **Import Check**: Verifies that all imported modules exist and are available in the runtime environment.
3. **Structure Check**: Confirms the generated graph has valid nodes, edges, an entry point, and reachable terminal states.
4. **Execution Test**: Actually instantiates and invokes the generated graph with a minimal test input. Catches runtime errors that static analysis misses.
5. **LLM Critique**: Uses an LLM to review the code for logical problems, anti-patterns, and missing error handling — returning the `CritiqueResult`.

Each layer that fails contributes to the `issues` list and reduces `overall_score`. The Conductor's replan gate compares `overall_score` against a configured threshold.

---

### 6.6 CritiqueRefiner

**Role:** Apply natural-language critique feedback to improve an existing `WorkflowDesign`.

**When used:** After the replan gate fires, or when a human provides explicit feedback on a design.

**How it works:**

The CritiqueRefiner receives the current `WorkflowDesign` object and the feedback string (from either `EvaluatorAgent.feedback` or `VerificationAgent.issues`). It prompts an LLM to produce a JSON patch describing specific changes to the design. It then parses this JSON and applies the changes programmatically to the `WorkflowDesign` object and any already-generated source files.

This is notably more targeted than regenerating the entire design from scratch — it modifies only the flagged elements, preserving the parts that were already correct.

---

### 6.7 RemediationAgent

**Role:** Fix code-level errors discovered during execution or verification.

**Scope:** Handles both Python agent code and React/JSX frontend code (for agents with UI components).

**How it works:**

Given an error message (traceback, syntax error, type error) and the relevant source code, the RemediationAgent prompts an LLM to produce a corrected version. It uses a structured prompting approach: the error is clearly labeled, the relevant code snippet is isolated, and the LLM is asked to explain the fix before producing it.

The RemediationAgent can be called multiple times (up to a configured maximum) on the same error, attempting progressively different strategies if the initial fix doesn't work.

---

### 6.8 VisionAgent

**Role:** Conversational brainstorming partner for exploratory, pre-specification conversations with users who aren't sure exactly what they want.

**Architecture:** A LangGraph StateGraph with three nodes:

```
planner → brainstorm → synthesize
```

| Node | Role |
|------|------|
| `planner` | Determines what information is still needed from the user |
| `brainstorm` | Generates options, possibilities, and examples relevant to what the user has said |
| `synthesize` | Combines user responses and brainstormed ideas into a coherent agent specification |

**RequirementsSlots:**

The VisionAgent tracks which aspects of the agent specification have been filled:

```python
class RequirementsSlots(BaseModel):
    purpose: Optional[str]          # What the agent does
    data_sources: List[str]         # What data it works with
    tools_needed: List[str]         # What tools it needs
    output_format: Optional[str]    # What it produces
    user_interaction: bool          # Does it need a human-in-the-loop?
    performance_requirements: Optional[str]
```

The planner node checks which slots are unfilled and generates targeted questions to complete them. Once all slots are filled, the VisionAgent synthesizes a specification that can be handed directly to the Conductor pipeline.

---

## 7. The WorkflowDesign Intermediate Representation

The `WorkflowDesign` is the pivotal data structure of the entire system — the "AST" of the compilation pipeline. Everything before it works to produce it; everything after it consumes it.

### Top-Level Model

```python
class WorkflowDesign(BaseModel):
    # Identity
    name: str                               # Human-readable agent name
    description: str                        # What the agent does

    # State definition
    state_fields: List[str]                 # Simple field names (legacy)
    state_field_specs: List[StateFieldSpec] # Rich field definitions (current)
    memory_channels: List[MemoryChannel]    # LangGraph memory configuration

    # Graph structure
    nodes: List[WorkflowNode]               # Agent nodes
    edges: List[WorkflowEdge]               # Connections between nodes
    entry_point: str                        # Name of the first node

    # Requirements
    tools_required: List[str]               # Tool names from ToolRegistry
    sub_agents: List[str]                   # Sub-agent names for nesting

    # Architecture decisions
    orchestration_pattern: str              # How agents coordinate
    topology: TopologySpec                  # Structural pattern
    rag_tier: RagTier                       # How much RAG to use at runtime
    generation_tier: str                    # simple_chain | langgraph | deep_agent

    # Compiler metadata
    metadata: IRMetadata                    # Version, passes applied
```

### WorkflowNode

```python
class WorkflowNode(BaseModel):
    name: str                               # Unique identifier in the graph
    type: Literal[                          # Node category
        "llm",          # LLM reasoning node
        "tool",         # Tool execution node
        "router",       # Conditional routing node
        "human",        # Human-in-the-loop node
        "agent",        # Sub-agent invocation node
        "file_io"       # File read/write node
    ]
    system_prompt: Optional[str]            # System prompt for LLM nodes
    inputs: List[str]                       # State fields this node reads
    outputs: List[str]                      # State fields this node writes
    model: Optional[str]                    # "provider/model" e.g. "openai/gpt-4o"
    tools: List[str]                        # Tool names for tool/llm nodes
    domain: Optional[str]                   # Domain specialization hint
    autonomy: Optional[AgentAutonomy]       # Autonomy configuration
    file_config: Optional[FileIOConfig]     # For file_io nodes
```

### AgentAutonomy

```python
class AgentAutonomy(BaseModel):
    delegation_scope: str       # What decisions the agent can make independently
    max_tool_calls: int         # Maximum tool invocations per turn
    requires_approval: bool     # Whether human approval gates this node
    approval_criteria: str      # What triggers an approval request
```

### WorkflowEdge

```python
class WorkflowEdge(BaseModel):
    source: str                             # Source node name
    target: str                             # Target node name (or END sentinel)
    condition: Optional[str]                # Python expression for routing
    condition_field: Optional[str]          # State field to evaluate
    condition_map: Optional[Dict[str, str]] # Value → target node mapping
    default_route: Optional[str]            # Fallback if condition doesn't match
    flow_type: Literal["control", "data"]   # Control flow vs data dependency
```

### TopologySpec

```python
class TopologySpec(BaseModel):
    pattern: Literal[
        "sequential",           # A → B → C → D
        "fan_out_parallel",     # A → [B, C, D] → E (parallel)
        "map_reduce",           # A → map([B, B, B]) → reduce(C)
        "conditional_routing",  # A → route → B or C
        "cyclic_agentic"        # A → B → ... → A (with termination)
    ]
```

### IRMetadata

```python
class IRMetadata(BaseModel):
    ir_version: str                     # Schema version for compatibility
    optimization_passes: List[str]      # Which passes have been applied
    created_at: datetime
    planning_agent_version: str
    citations: List[str]                # RAG source citations used in design
```

### StateFieldSpec

More expressive than a simple field name:

```python
class StateFieldSpec(BaseModel):
    name: str
    type: str                   # Python type annotation as string
    description: str            # What this field holds
    default: Optional[Any]      # Default value
    reducer: Optional[str]      # LangGraph reducer function (e.g. "add_messages")
    is_channel: bool            # Whether this is a message channel
```

### MemoryChannel

```python
class MemoryChannel(BaseModel):
    name: str                   # Channel identifier
    channel_type: str           # "message" | "value" | "binop"
    reducer: Optional[str]      # For message channels: add_messages
    scope: str                  # "thread" | "global" | "session"
```

---

## 8. RAG System: Quad-Hybrid Ensemble Retrieval

### Overview

The RAG system backs the PlanningAgent's `retrieve_guides` step and any other agent that needs grounded knowledge. It implements a **quad-hybrid** retrieval strategy — four different retrieval mechanisms running in parallel and then fused — because no single retrieval technique dominates across all query types:

| Retriever Type | Best At |
|---------------|---------|
| Dense (semantic) vector search | Conceptual similarity, paraphrased queries |
| Sparse (BM25) keyword search | Exact term matching, technical identifiers |
| Graph-based traversal | Relational knowledge, "what connects to what" |
| Guide tree navigation | Structured document hierarchies, section lookup |

### Architecture Diagram

```
Query (from PlanningAgent)
     │
     ▼
SemanticRouter ─────────────────────────────────────────────────┐
(routes by topic + estimates complexity)                        │
     │                                                          │
     ├─────────────────────────────────────────────────────┐   │
     │                                                     │   │
     ▼                                                     ▼   │
VectorStore Retriever              GraphRetriever               │
(LanceDB + BM25 hybrid)            (NetworkX knowledge graph)   │
     │                                                     │   │
     └──────────────────────┬──────────────────────────────┘   │
                            │                                   │
                            ▼                                   │
                     RRF Fusion                                 │
                (Reciprocal Rank Fusion                         │
                 combines ranked lists)                         │
                            │                                   │
                            ▼                                   │
                    Cohere Rerank                               │
                (Cross-encoder reranking                        │
                 for relevance precision)                        │
                            │                                   │
                            ▼                                   │
                    CRAG Quality Gate                           │
                (RetrievalGate — checks if                      │
                 results actually answer query)                  │
                    /             \                              │
               Pass               Fail → web search fallback    │
                │                                               │
                ▼                                               │
          CLaRa Enrichment                                      │
        (Contextual Late-stage                                   │
         Retrieval Augmentation)                                 │
                │                                               │
                ▼                                               │
       CitationVerifier                                         │
        (Ensures cited sources                                   │
         actually contain the claim)                            │
                │                                               │
                ▼                                               │
      Final Ranked Results ◄──────────────────────────────────┘
        with Citations                  GuideRetriever
                                   (tree navigation for
                                    structured guide docs)
```

### Component Details

**SemanticRouter**  
Routes queries to the appropriate retriever set and estimates query complexity (which affects how much compute to spend on retrieval). A query about "sequential LLM chains" routes primarily to VectorStore; a query about "how tool X relates to pattern Y" routes to GraphRetriever.

**VectorStore (LanceDB + BM25)**  
LanceDB is the vector database backend, storing dense embeddings of all guide documents. BM25 provides sparse keyword scoring as a complementary signal. The retriever produces a hybrid score blending both signals.

**GraphRetriever (NetworkX)**  
Maintains a knowledge graph where nodes are concepts (agent patterns, tools, LangChain primitives) and edges represent relationships (uses, extends, requires, conflicts-with). Traversal from query concepts finds related concepts that might not appear semantically similar in embedding space.

**GuideRetriever**  
Navigates a tree-structured index of curated guide documents. This is useful when the query maps to a specific section of a known guide — it can fetch the relevant subtree directly rather than relying on embedding similarity.

**RRF Fusion**  
Reciprocal Rank Fusion is a simple but effective technique for combining ranked lists from multiple retrievers. Each document's fusion score is the sum of 1/(rank + k) across all lists it appears in. This rewards documents that rank well in multiple retrievers without requiring score normalization.

**Cohere Rerank**  
After fusion, Cohere's cross-encoder reranking model re-scores each candidate against the query. Cross-encoders are much more accurate than bi-encoders for relevance scoring but too slow to apply to the full corpus — hence they're applied only to the already-filtered fusion output.

**CRAG Quality Gate (RetrievalGate)**  
The Corrective-RAG quality gate evaluates whether the retrieved results actually answer the query. If the gate determines the results are irrelevant or insufficient, it falls back to a web search to supplement local retrieval. This prevents the planning agent from being grounded in irrelevant retrieved content.

**CLaRa Enrichment**  
Contextual Late-stage Retrieval Augmentation adds surrounding context to retrieved chunks — fetching adjacent sections of the source document to ensure the retrieved passage is properly framed.

**CitationVerifier**  
Verifies that each citation in the retrieved results actually supports the claim being made. This guards against hallucinated citations.

### RagTier Enum

The system configures retrieval depth dynamically based on request complexity:

```python
class RagTier(IntEnum):
    NONE     = 0   # No retrieval — use only LLM parametric knowledge
    BASIC    = 1   # VectorStore only, no reranking
    STANDARD = 2   # VectorStore + BM25, with reranking
    ADVANCED = 3   # Full quad-hybrid, with CRAG gate
    MAXIMUM  = 4   # Full pipeline + CLaRa + CitationVerifier + PwCRetriever
```

The `MAXIMUM` tier also activates `PwCRetriever` — a specialized retriever that pulls from Papers with Code to ground design decisions in empirical research.

---

## 9. Evolutionary Optimization System

### Motivation

The Evolutionary Optimizer addresses a fundamental question: for a given agent specification, is the first design the PlanningAgent produces the best possible design? Almost certainly not — the design space is large and the PlanningAgent explores it greedily (one plan per run). The Evolutionary Optimizer treats agent design as an optimization problem and uses genetic algorithms to systematically explore the space.

### CandidateAgent

```python
class CandidateAgent(BaseModel):
    design: WorkflowDesign              # The agent's "genome"
    fitness: float                      # Combined fitness score
    generation: int                     # Which generation produced this candidate
    rubric_dims: Dict[str, float]       # Per-dimension fitness scores
    execution_descriptor: str           # How to run this agent
    sandbox_descriptor: str             # Where to run it (Modal vs Docker)
```

### Fitness Function

```python
fitness = (
    0.7 * quality_score     # EvaluatorAgent score across test scenarios
  + 0.2 * cost_score        # Inverse of estimated token cost
  + 0.1 * latency_score     # Inverse of estimated/measured latency
)
```

The strong weight on quality (0.7) reflects that a fast, cheap agent that doesn't work well is useless. Cost (0.2) is significant — agents that use unnecessarily large models or make excessive LLM calls are penalized. Latency (0.1) is a lighter signal, as latency varies with infrastructure.

### Genetic Algorithm Flow

```
Generation 0: Initial Population
  ├── PlanningAgent produces N diverse designs (varied via temperature, strategy)
  └── Each design built by GraphFactory, tested in AgentGym

  ▼

Fitness Evaluation
  ├── EvaluatorAgent scores each CandidateAgent against test scenarios
  └── Scores stored in CandidateAgent.rubric_dims

  ▼

Selection
  ├── Tournament selection: pick best of K random candidates
  └── Elite preservation: top E candidates survive unchanged

  ▼

Crossover
  └── Combine WorkflowDesign objects: take nodes from parent A, edges from parent B
      (respecting topology constraints)

  ▼

Mutation
  ├── Swap a node's model
  ├── Add/remove a tool
  ├── Change a node's system prompt
  ├── Alter an edge condition
  └── CritiqueRefiner: apply LLM-driven mutation based on fitness feedback

  ▼

Generation N+1: New Population
  └── Repeat until convergence or max_generations reached
```

### MAP-Elites Quality-Diversity Archive

Beyond simple fitness maximization, the system maintains a **MAP-Elites** archive — a grid of cells indexed by behavioral dimensions (e.g., number of nodes × RAG tier). Each cell holds the highest-fitness candidate with that behavioral profile. This ensures the optimizer discovers a *diverse* set of high-quality designs rather than collapsing to a single local optimum.

### Fitness Sharing

In a standard genetic algorithm, if many candidates have similar designs, they all score similarly and crowd out diverse candidates. Fitness sharing reduces the effective fitness of candidates that are too similar to other high-scoring candidates, explicitly rewarding novelty.

### Population Persistence

The population is persisted to disk between runs. If the optimizer is stopped and restarted, it resumes from the last saved generation rather than starting over. This makes long optimization runs (dozens of generations) practical even with interruptions.

---

## 10. IR Compiler Passes

Inspired by compiler optimization pipelines, meta-builder-v3 applies a sequence of transformation passes to the `WorkflowDesign` IR before code generation. Each pass makes the design more correct, efficient, or cost-effective.

### Pass: LoopBoundInsertion

**Problem:** Cyclic agentic topologies (agents that loop until a goal is met) can run forever if the termination condition is never satisfied.

**Fix:** This pass scans all cycles in the `WorkflowEdge` list and ensures each cycle has a maximum iteration guard in the state. If `max_iterations` is not set, the pass inserts a default bound and adds a `StateFieldSpec` to track the iteration counter.

**Before:**
```
Node A → Node B → Node A  (no termination)
```

**After:**
```
Node A → Node B → Router (iteration_count < max_iter?) → Node A | END
StateField: iteration_count: int = 0
```

### Pass: DeadNodeElimination

**Problem:** The PlanningAgent may generate nodes that are never reached — either because no edge points to them or because the entry point is on a different branch.

**Fix:** Graph reachability analysis from the `entry_point`. Any node not reachable from the entry point is removed. Any `StateFieldSpec` only referenced by dead nodes is also removed.

### Pass: StateFieldPruning

**Problem:** The PlanningAgent may generate `state_fields` that are declared but never read or written by any node.

**Fix:** Cross-reference each field against all node `inputs` and `outputs`. Remove fields that appear in neither. This reduces StateGraph memory footprint and makes the state cleaner.

### Pass: ParallelExtraction

**Problem:** The PlanningAgent may generate sequential nodes that could safely run in parallel (no data dependency between them).

**Fix:** Dependency analysis — build a directed acyclic graph of data dependencies between nodes. Identify independent node groups. Replace sequential edges with `Send`-based parallel dispatch edges where safe.

### Pass: CostEstimation

**Problem:** Without knowing the approximate cost of running the generated agent, users can't make informed decisions about which design to use.

**Fix:** This pass annotates each `WorkflowNode` with an estimated per-call token cost based on its model, estimated prompt size, and expected output length. Aggregates to a total estimated cost per agent invocation. Stored in `IRMetadata` for display and for the fitness function's cost component.

### Pass Execution Order

```python
class IRCompiler:
    PASSES = [
        DeadNodeElimination,    # Remove unreachable nodes first
        StateFieldPruning,      # Clean state after dead node removal
        LoopBoundInsertion,     # Add safety guards
        ParallelExtraction,     # Expose parallelism
        CostEstimation,         # Annotate with cost estimates (last — uses final graph)
    ]

    def compile(self, design: WorkflowDesign) -> WorkflowDesign:
        for pass_class in self.PASSES:
            pass_instance = pass_class()
            design = pass_instance.apply(design)
            design.metadata.optimization_passes.append(pass_class.__name__)
        return design
```

---

## 11. Generation Tiers

The final step before a `WorkflowDesign` becomes runnable code is selecting the generation backend. Three tiers are available, each appropriate for different complexity levels:

### Tier 1: simple_chain

**When used:** Low-complexity, purely sequential workflows. No branching, no tool calls, no state needed beyond conversation messages.

**What it generates:** An LCEL (LangChain Expression Language) chain using the pipe operator:

```python
# Example generated simple_chain
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

chain = (
    ChatPromptTemplate.from_messages([
        ("system", "{system_prompt}"),
        ("human", "{input}")
    ])
    | llm
    | StrOutputParser()
)
```

**Advantages:** Very lightweight, easy to understand, low latency. Can be deployed as a simple API endpoint.

**Limitations:** No branching, no tool use, no persistent state.

---

### Tier 2: langgraph

**When used:** Medium complexity — needs state, conditional routing, tool calls, or simple cycles.

**What it generates:** A full LangGraph `StateGraph` with typed state, node functions, and properly wired edges. The `GraphFactory` handles construction.

```python
# Simplified example of generated langgraph code
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode, tools_condition
from typing import TypedDict, List
from langchain_core.messages import BaseMessage

class AgentState(TypedDict):
    messages: List[BaseMessage]
    tool_results: List[str]

def reasoning_node(state: AgentState) -> AgentState:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": state["messages"] + [response]}

graph = StateGraph(AgentState)
graph.add_node("reason", reasoning_node)
graph.add_node("tools", ToolNode(tools))
graph.add_conditional_edges("reason", tools_condition)
graph.add_edge("tools", "reason")
graph.set_entry_point("reason")
compiled = graph.compile()
```

---

### Tier 3: deep_agent

**When used:** High complexity — multi-agent systems, sub-graph isolation, sandbox execution, evolutionary optimization targets.

**What it generates:** Both an in-memory graph (for immediate testing) AND a complete exportable Python module structure:

```
output/
├── agent.py          # Main agent module with all node implementations
├── state.py          # State TypedDict definitions
├── tools.py          # Tool implementations
├── subagents/        # Sub-agent modules if multi-agent
│   ├── retriever_agent.py
│   └── synthesis_agent.py
├── requirements.txt  # Python dependencies
└── run.py            # Entry point for standalone execution
```

This tier uses `DeepAgentGenerator` rather than `GraphFactory` directly. `DeepAgentGenerator` adds:
- Proper module structure with imports
- Entry point scripts for Modal and Docker
- Dockerfile / Modal stub generation
- Environment variable handling
- Logging configuration

---

## 12. LLM Factory: Multi-Provider Abstraction

### Purpose

Different compilation tasks benefit from different models — the RequestProcessor might use a fast, cheap model for straightforward intent extraction, while the PlanningAgent's `detail_design` step benefits from a larger, more capable model. The LLM Factory provides a unified interface for all agent LLM calls, with per-operation model routing.

### Supported Providers

| Provider | Notes |
|----------|-------|
| Google (Gemini) | Strong reasoning, long context |
| OpenAI (GPT-4o, etc.) | General purpose, structured output reliability |
| Anthropic (Claude) | Long context, careful reasoning |
| Ollama | Local deployment, privacy-preserving |
| AirLLM | Ultra-large model inference on consumer hardware |
| Mock | Deterministic test doubles for CI/CD |

### Call-Site Resolver

The Factory implements a **call-site resolver** pattern: each call site (e.g., "PlanningAgent.detail_design") has a named configuration entry specifying which provider+model to use, temperature, max tokens, and fallback behavior. Agents don't hardcode model names — they request an LLM by role name:

```python
# Agent code
llm = llm_factory.get_llm("planning_agent.detail_design")

# Configuration (resolved by factory at runtime)
LLM_ROUTING = {
    "planning_agent.detail_design": {
        "provider": "anthropic",
        "model": "claude-3-5-sonnet",
        "temperature": 0.7,
        "fallback": "openai/gpt-4o"
    },
    "request_processor.intent_extraction": {
        "provider": "openai",
        "model": "gpt-4o-mini",
        "temperature": 0.1,
    }
}
```

This routing table makes it trivial to swap models without touching agent code — just update the configuration.

### Structured Output Binding

All agents that need structured JSON output use `with_structured_output(PydanticModel)`. The Factory wraps this binding:

```python
llm = llm_factory.get_llm("planning_agent.design")
structured_llm = llm.with_structured_output(WorkflowDesign)
result: WorkflowDesign = await structured_llm.ainvoke(messages)
```

The Factory handles the provider-specific implementation differences (some providers use function calling; others use JSON mode; Mock uses predefined fixtures).

---

## 13. Tool Registry

### What It Is

The `ToolRegistry` is a runtime catalog of all available LangChain-compatible tools that generated agents can use. Rather than hardcoding tool lists, `GraphFactory` resolves tool names through the registry at graph construction time.

### Discovery Mechanism

The registry scans the `src/tools/` directory for functions decorated with LangChain's `@tool` decorator:

```python
# Example tool definition (discovered automatically)
from langchain_core.tools import tool

@tool
def search_arxiv(query: str, max_results: int = 5) -> str:
    """Search arXiv for academic papers matching the query."""
    # ... implementation
```

At startup, the ToolRegistry uses Python introspection to find all `@tool`-decorated functions, extract their schemas, and register them by name.

### Runtime Resolution

When `GraphFactory` encounters a `WorkflowNode` with `tools: ["search_arxiv", "web_search"]`, it calls `ToolRegistry.resolve(["search_arxiv", "web_search"])` which returns the actual `BaseTool` objects. These are passed to `ToolNode` or bound to an LLM via `llm.bind_tools(tools)`.

### Available Tool Categories

Based on the integration list:
- **Research**: arXiv retrieval, Papers with Code, Google Scholar
- **Web**: Playwright-based browsing, general web search
- **Code**: GitHub code fetcher, code execution
- **Data**: Google Stax, spreadsheet operations
- **AI Services**: NotebookLM, Jules Agent
- **Utility**: File I/O, datetime, string manipulation

---

## 14. Safety and Telemetry Middleware

### Safety Guardrails Middleware

All LLM inputs and outputs pass through the Safety Guardrails Middleware — a pipeline of filters applied transparently to every LLM interaction:

**PII Detection:**  
Scans prompts and responses for personally identifiable information (names, email addresses, phone numbers, SSNs, credit card numbers). When detected, PII is redacted from logs and optionally from the LLM context itself.

**Prompt Injection Detection:**  
Scans for common prompt injection patterns — attempts by malicious content in retrieved documents or user inputs to hijack the LLM's behavior (e.g., "ignore previous instructions and..."). Detected injections are blocked and logged.

**Output Filtering:**  
Post-processes LLM outputs to remove harmful content, verify format compliance, and enforce output length limits.

The middleware is applied as a wrapper around LLM calls — agents interact with `SafeLLMClient` which transparently enforces all policies.

### Telemetry Middleware (Langfuse)

All LLM calls throughout the system are traced using Langfuse:

```python
from langfuse import observe, propagate_attributes

@observe(name="PlanningAgent.detail_design")
async def detail_design(self, state: PlannerState) -> PlannerState:
    # All LLM calls within this function are automatically traced
    result = await self.llm.ainvoke(messages)
    return {**state, "design": result}
```

The `@observe` decorator captures:
- Input messages and their token counts
- Output content and token counts
- Latency
- Model used
- Trace ID for correlation across the full compilation run

`propagate_attributes` ensures that the top-level session context (session_id, user_id, etc.) flows through all nested spans — so a single compilation run appears as a coherent trace tree in the Langfuse dashboard.

---

## 15. Research Department

The Research Department is a semi-autonomous subsystem for systematic AI research — discovering new agent patterns, evaluating emerging models, and acquiring new tools.

### Components

**ResearchOrchestrator**  
Coordinates the research subsystem. Manages a queue of research questions, dispatches to specialized components, and synthesizes findings into actionable recommendations for the main system.

**ScoutMonitor**  
Continuously monitors the candidate agent population (during evolutionary optimization) for interesting patterns — agents that perform unusually well on specific dimensions, novel topologies that emerged from crossover, etc. Flags these for deeper investigation.

**VerdictPanel**  
A multi-judge evaluation system for high-stakes decisions. Rather than relying on a single EvaluatorAgent, the VerdictPanel runs multiple independent evaluations (different models, different prompting strategies) and aggregates scores using a voting or averaging scheme. Used when a decision is too important to trust to a single judge.

**DecisionGate**  
A structured decision point — given a set of options and evaluation scores from VerdictPanel, DecisionGate applies decision criteria (configured rules, thresholds, business constraints) to select the recommended option.

**ExperimentRunner**  
Runs controlled A/B experiments comparing different agent designs, retrieval strategies, or model configurations. Tracks metrics across multiple runs to produce statistically meaningful comparisons.

**Tool Acquisition**  
A research pipeline that can discover and integrate new tools:
1. Identifies capability gaps from user requests the system struggled with
2. Searches for relevant APIs, libraries, or pre-built tools
3. Generates `@tool`-decorated wrapper code
4. Tests the new tool in AgentGym
5. Registers in ToolRegistry if tests pass

### External Research Integrations

| Integration | Purpose |
|-------------|---------|
| arXiv | Retrieve academic papers on agent architectures, RAG methods, etc. |
| Playwright | Browser automation for web-based research |
| GitHub Code Fetcher | Retrieve and analyze reference implementations |
| Google Stax | Data retrieval and analysis |
| Jules Agent | AI-powered research assistant |
| NotebookLM | Document analysis and synthesis |
| Papers with Code | Find papers with associated code implementations |

---

## 16. Agent Gym and Sandbox Execution

### Purpose

The AgentGym is the testing and evaluation environment for generated agents. Before an agent is accepted as a successful build or counted as a valid candidate in evolutionary optimization, it must pass through the Gym.

### Components

**SyntheticDataGenerator**  
Creates test scenarios for agents — simulated user inputs, tool call sequences, and expected outputs. Uses **adaptive difficulty**: the scenario generator adjusts the complexity of test cases based on the agent's current performance level. A struggling agent gets simpler scenarios first; a high-performing agent gets progressively harder edge cases.

**GraphRunner**  
Executes generated LangGraph StateGraphs in the local process. Used for fast, lightweight testing during development and in the initial generations of evolutionary optimization.

**DeepAgentRunner**  
Executes deep_agent tier outputs — the full module structure rather than just the in-memory graph. Handles module import, entry point invocation, and result collection.

**ModuleRunner**  
A more general runner for arbitrary Python module structures, used when testing components in isolation.

### Sandbox Backends

For safe execution of untrusted generated code:

**Modal Backend**  
Executes agents in Modal serverless functions — isolated, ephemeral compute environments with configurable resources and timeouts. Modal provides cryptographic isolation and billing per execution.

**Docker Backend**  
Executes agents in Docker containers. Used when Modal is not available or when the agent requires specific system dependencies. Containers are built from a base image and destroyed after execution.

Both backends capture stdout, stderr, and execution telemetry. Execution results are returned as structured objects with timing, resource usage, and any errors.

---

## 17. Studio: Visualization and Inspection

### pipeline_graph.py

The Studio component provides LangGraph Studio integration — a visual debugger for LangGraph state machines. `pipeline_graph.py` exports the Conductor's StateGraph in a format compatible with LangGraph Studio, enabling:

- Visual inspection of the phase graph topology
- Real-time state observation during a compilation run
- Breakpoints and step-through debugging
- State history replay

This is primarily a developer tool but becomes essential when debugging a replan loop that keeps cycling — you can watch the state evolve across iterations and identify why the quality gate keeps firing.

---

## 18. End-to-End Data Flow Diagram

```
USER INPUT
"Build me a research agent that reads arXiv papers and 
produces structured summaries with citations"
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│  CONDUCTOR: initialize phase                                      │
│  • Create session_id                                              │
│  • Snapshot config                                                │
│  • Write raw_request to ConductorState immutable zone             │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  CONDUCTOR: process_request phase                                 │
│  → Dispatch to RequestProcessor                                   │
│                                                                   │
│  RequestProcessor:                                                │
│  • LLM call with raw_request                                      │
│  • with_structured_output(ProcessedRequest)                       │
│  • Returns: intention="research_summarization",                   │
│             integrations=["arxiv_api"],                           │
│             topology=TopologySpec(pattern="sequential"),           │
│             rag_tier=STANDARD                                     │
│                                                                   │
│  Conductor writes ProcessedRequest → ConductorState               │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  CONDUCTOR: plan_workflow phase                                    │
│  → Dispatch to PlanningAgent                                      │
│                                                                   │
│  PlanningAgent internal graph:                                    │
│  1. retrieve_guides: RAG query → fetches "sequential RAG agent"  │
│     guide with citations                                          │
│  2. decompose: breaks into [fetch_papers, read_papers,            │
│     extract_key_points, format_citations, write_summary]          │
│  3. select_architecture: sequential + deep_agent tier             │
│  4. detail_design: populates WorkflowNode objects with            │
│     system prompts, tool assignments, model specs                 │
│  5. validate: confirms all edges valid, entry point exists        │
│                                                                   │
│  Returns: WorkflowDesign IR                                       │
│  Conductor writes WorkflowDesign → ConductorState                 │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  IR COMPILER PASSES (applied to WorkflowDesign)                   │
│  1. DeadNodeElimination: no dead nodes found                      │
│  2. StateFieldPruning: removes unused "debug_log" field           │
│  3. LoopBoundInsertion: no cycles → no-op                         │
│  4. ParallelExtraction: no independent nodes → no-op              │
│  5. CostEstimation: estimates $0.003/run                          │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  CONDUCTOR: generate_code phase                                   │
│  → Dispatch to DeepAgentGenerator (deep_agent tier)               │
│                                                                   │
│  DeepAgentGenerator:                                              │
│  • Calls GraphFactory to build in-memory LangGraph StateGraph     │
│  • Generates agent.py, state.py, tools.py, run.py                 │
│  • Writes Modal stub for deployment                               │
│                                                                   │
│  Returns: generated_graph (StateGraph), generated_code (str)     │
│  Conductor writes both → ConductorState                           │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  CONDUCTOR: verify_output phase                                   │
│  → Dispatch to VerificationAgent                                  │
│                                                                   │
│  VerificationAgent:                                               │
│  1. Syntax check: PASS                                            │
│  2. Import check: PASS                                            │
│  3. Structure check: PASS                                         │
│  4. Execution test: Runs with mock arXiv response → PASS          │
│  5. LLM critique: "Well-structured; consider adding retry logic   │
│     for API failures" — score: 0.87                               │
│                                                                   │
│  CritiqueResult.overall_score = 0.87 > threshold (0.75)          │
│  → PASS GATE → proceed to done                                    │
│                                                                   │
│  [If score < 0.75: REPLAN GATE fires, resets WorkflowDesign,     │
│   sends critique feedback to PlanningAgent, loops back]           │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│  CONDUCTOR: done phase                                            │
│  • Package BuildArtifact with:                                    │
│    - WorkflowDesign IR                                            │
│    - Generated source code                                        │
│    - CritiqueResult + EvaluationResult                            │
│    - Cost estimate                                                │
│    - Design citations                                             │
│  • Write to ConductorState.output_artifact                        │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
OUTPUT: Deployable Research Agent
  ├── agent.py (runnable LangGraph StateGraph)
  ├── state.py (ArxivResearchState TypedDict)
  ├── tools.py (search_arxiv, format_citation)
  ├── run.py (entry point)
  ├── WorkflowDesign.json (machine-readable IR)
  └── evaluation_report.json (scores + feedback)
```

---

## 19. Design Principles and Architectural Tradeoffs

### Principle 1: Deterministic Hub, Probabilistic Spokes

**Decision:** The Conductor makes zero LLM calls. All LLM work is delegated.

**Rationale:** This makes the control flow perfectly auditable. You can always explain why a replan fired, why a phase was skipped, or why the system terminated — because these decisions are pure Python conditional logic, not LLM outputs.

**Tradeoff:** The Conductor can't adapt its strategy dynamically based on what it "sees" in the outputs. Adaptation happens within spoke agents or through the replan loop, not through the hub's routing logic.

### Principle 2: Type-Safe Intermediate Representations

**Decision:** `WorkflowDesign` and all related models are Pydantic models. Every interface is typed.

**Rationale:** In a system where LLMs produce the data that flows between components, type safety is the primary defense against data corruption. Pydantic validation catches malformed LLM outputs at the boundary, before they can propagate through the pipeline.

**Tradeoff:** Strict schemas limit LLM creativity — the LLM can't invent new node types or edge attributes that aren't in the schema. This is intentional; unbounded creativity in IR structure would make the GraphFactory impossible to implement reliably.

### Principle 3: Layered Retrieval Beats Single-Method Retrieval

**Decision:** Four-method ensemble retrieval with fusion and reranking.

**Rationale:** Different query types have different characteristics. Semantic search fails on exact technical terms; BM25 fails on paraphrased concepts; graph traversal fails on novel combinations. The ensemble covers the failure modes of each individual approach.

**Tradeoff:** Significantly higher complexity and latency vs. single-method retrieval. Justified at ADVANCED and MAXIMUM tiers; NONE and BASIC tiers sacrifice coverage for speed.

### Principle 4: IR Compiler Passes Enable Optimization Without Brittleness

**Decision:** Apply transformations as composable passes over the IR, not as monolithic code generation logic.

**Rationale:** Each pass has a single responsibility and can be tested in isolation. Adding a new optimization (e.g., "merge consecutive LLM nodes with the same model") requires only adding a new pass — no changes to existing passes or the code generator.

**Tradeoff:** More code than a monolithic generator. IR passes can interfere with each other if not carefully ordered (hence the fixed order).

### Principle 5: Evolutionary Search Over Greedy Planning

**Decision:** The Evolutionary Optimizer treats agent design as a search problem, not a planning problem.

**Rationale:** The PlanningAgent produces a good first design but can't guarantee it's optimal. Genetic search explores the design space systematically and empirically, discovering improvements that no amount of prompt engineering would find.

**Tradeoff:** Computationally expensive — each generation requires building and testing N agent candidates. Only justified for production agents where quality matters enough to invest in optimization.

---

*This document covers meta-builder-v3 v4.9.0. The system is under active development; specific implementation details may evolve. The architectural principles described here represent the stable core design.*
