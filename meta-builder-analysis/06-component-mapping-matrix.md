# 06 — Component Mapping Matrix: LangChain, LangGraph, and DeepAgents → Meta-Builder-V3

> **Project:** meta-builder-v3 (v4.9.0) — AI-Native Orchestration Compiler  
> **Purpose:** Exhaustive mapping of every ecosystem concept to specific meta-builder components  
> **Date:** April 2026

---

## Overview

This document provides a comprehensive cross-reference between the LangChain ecosystem (LangChain Core, LangChain Modules, LangGraph, DeepAgents) and their counterparts in meta-builder-v3. Each table entry identifies:

- **Ecosystem Concept**: The canonical LangChain/LangGraph/DeepAgents term
- **Meta-Builder Component**: The specific class, function, or module in meta-builder-v3
- **File Location**: Path within the project (`src/...`)
- **Integration Depth**: How deeply the concept is adopted (Native / Extended / Custom / Unused)
- **Notes**: Implementation details, divergences, and insights

**Integration Depth Legend:**

| Level | Meaning |
|---|---|
| **Native** | Uses the LangChain class/interface directly with minimal modification |
| **Extended** | Subclasses or wraps a LangChain base class with significant additions |
| **Custom** | Implements the conceptual equivalent but without inheriting from LangChain classes |
| **Scaffolded** | Generated into output projects but not used in meta-builder-v3 internals |
| **Planned** | Documented as a future enhancement; not yet implemented |

---

## Table of Contents

1. [LangChain Core — Messages, LLMs, Prompts, Tools, Chains, Callbacks, Runnables](#1-langchain-core)
2. [LangChain Modules — Document Loaders, Text Splitters, Embeddings, Vector Stores, Retrievers, Memory](#2-langchain-modules)
3. [LangGraph — StateGraph, State, Reducers, Nodes, Edges, ToolNode, Send, Checkpointing, Sub-graphs, Human-in-Loop](#3-langgraph)
4. [DeepAgents — Agent Harness, Sub-Agent Isolation, Sandbox Execution, Deep Agent Tier](#4-deepagents)
5. [Ecosystem Infrastructure — Tracing, Studio, Deployment](#5-ecosystem-infrastructure)
6. [Inverse Mapping — Meta-Builder Components Without LangChain Equivalents](#6-inverse-mapping)

---

## 1. LangChain Core

### 1.1 Message Types

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `HumanMessage` | Used directly in all agent pipelines | `src/agents/*/` | **Native** | Standard LC message type; no modification needed |
| `AIMessage` | Used in agent response handling | `src/agents/*/` | **Native** | Tool calls embedded via AIMessage.tool_calls |
| `SystemMessage` | Used in PlanningAgent, EvaluatorAgent prompts | `src/agents/planning/`, `src/agents/evaluator/` | **Native** | System prompts constructed as SystemMessage |
| `ToolMessage` | Used in tool execution result handling | `src/tools/registry.py` | **Native** | Tool results returned as ToolMessage with tool_call_id |
| `FunctionMessage` | Legacy; superseded by ToolMessage in project | — | **Unused** | Project targets modern LC API only |
| `ChatMessage` | Not explicitly used | — | **Unused** | Project uses role-specific message types |
| `MessageLikeRepresentation` | Implicitly supported via LC polymorphism | `src/agents/*/` | **Native** | All agents accept MessageLike inputs |
| `MessagesAnnotation` | Used in LangGraph state for generated projects | Scaffolded in GraphFactory output | **Scaffolded** | Generated `StateGraph` nodes use MessagesAnnotation |
| `trim_messages()` | Applied in agents with long conversation history | `src/agents/planning/` | **Native** | Prevents context window overflow in planning loops |
| `filter_messages()` | Used in memory channel processing | `src/agents/memory/` | **Native** | Filters by type when preparing LLM context |
| `merge_message_runs()` | Used in EvaluatorAgent multi-turn context | `src/agents/evaluator/` | **Native** | Merges consecutive AI messages for coherent evaluation |

### 1.2 LLM / Chat Model Interface

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `BaseChatModel` | `LLMFactory` output implements this interface | `src/llm/factory.py` | **Extended** | Factory returns `BaseChatModel`-compatible instances |
| `ChatOpenAI` | `LLMFactory` — OpenAI provider | `src/llm/factory.py` | **Native** | Wraps `langchain_openai.ChatOpenAI` |
| `ChatAnthropic` | `LLMFactory` — Anthropic provider | `src/llm/factory.py` | **Native** | Wraps `langchain_anthropic.ChatAnthropic` |
| `ChatGoogleGenerativeAI` | `LLMFactory` — Google/Gemini provider | `src/llm/factory.py` | **Native** | Wraps `langchain_google_genai` |
| `ChatOllama` | `LLMFactory` — Ollama provider (local + cloud) | `src/llm/factory.py` | **Extended** | Custom routing between local and Ollama cloud; `normalize_model_for_provider()` handles legacy names |
| `AirLLM` | `LLMFactory` — AirLLM provider | `src/llm/factory.py` | **Extended** | For memory-constrained local inference; not a standard LC integration |
| `MockLLM` | `LLMFactory` — Mock provider | `src/llm/factory.py` | **Custom** | Deterministic mock for testing/AgentGym evaluation |
| `BaseLLM.invoke()` | Universally used across all agents | All `src/agents/*/` | **Native** | Primary invocation pattern |
| `BaseLLM.ainvoke()` | Used in async retrieval and generation | `src/agents/RAG/`, generation pipeline | **Native** | All parallel operations use `ainvoke` |
| `BaseLLM.stream()` | Partially supported in VisionAgent | `src/agents/vision/` | **Native** | VisionAgent streams conversational responses |
| `BaseLLM.with_structured_output()` | Heavily used in PlanningAgent, EvaluatorAgent | `src/agents/planning/`, `src/agents/evaluator/` | **Native** | Forces JSON schema adherence for WorkflowDesign output |
| `BaseLLM.bind_tools()` | Used in ToolNode scaffolding | GraphFactory output | **Scaffolded** | Generated agent nodes bind registered tools |
| Call-site model routing | `LLMFactory.resolve()` per-operation routing | `src/llm/factory.py` | **Custom** | Different models for planning vs. evaluation vs. generation (no direct LC equivalent) |
| `normalize_model_for_provider()` | Utility in LLMFactory | `src/llm/factory.py` | **Custom** | Handles legacy model names (e.g., `gpt-4` → `gpt-4o`), Ollama cloud vs local disambiguation |

### 1.3 Prompt Templates

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `ChatPromptTemplate` | Used in all planning and evaluation prompts | `src/agents/planning/`, `src/agents/evaluator/` | **Native** | Standard LC prompt construction |
| `PromptTemplate` | Used for non-chat LLM invocations | Legacy paths in `src/llm/` | **Native** | Less common; most prompts use ChatPromptTemplate |
| `HumanMessagePromptTemplate` | Used in multi-turn prompt construction | `src/agents/*/` | **Native** | Within ChatPromptTemplate chains |
| `SystemMessagePromptTemplate` | Used in all agent system prompts | `src/agents/*/` | **Native** | Defines agent personas and instructions |
| `MessagesPlaceholder` | Used in all agents with conversation history | `src/agents/*/` | **Native** | Injects `{messages}` into prompt |
| `FewShotChatMessagePromptTemplate` | Used in EvaluatorAgent for grading examples | `src/agents/evaluator/` | **Native** | Few-shot examples for consistent scoring |
| `StructuredPrompt` | Used with `with_structured_output()` chains | `src/agents/planning/` | **Native** | WorkflowDesign JSON schema prompting |
| Dynamic prompt assembly | `CritiqueRefiner` assembles prompts from critique | `src/agents/critique/` | **Custom** | No direct LC equivalent; critiques modify prompt templates at runtime |

### 1.4 Tools and Tool Calling

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `@tool` decorator | Used on all tool functions | `src/tools/*/` | **Native** | Standard LC tool definition pattern |
| `BaseTool` | `ToolEntry.source_code` wraps `BaseTool` | `src/tools/registry.py` | **Native** | All tools conform to BaseTool interface |
| `StructuredTool` | Complex tools with Pydantic input schemas | `src/tools/*/` | **Native** | Multi-parameter tools use StructuredTool |
| `ToolRegistry` | `ToolEntry`: name, category, source_code, imports, pip_packages | `src/tools/registry.py` | **Custom** | Extends LC concept; tracks source + packages for code generation |
| `TOOL_IMPORTS` metadata | Decorator annotation for package declarations | `src/tools/registry.py` | **Custom** | No LC equivalent; enables scaffolding of `requirements.txt` |
| `resolve_tools()` | Returns `List[BaseTool]` at runtime | `src/tools/registry.py` | **Extended** | Standard LC `BaseTool` instances but assembled from registry |
| Tool categories: file_io, documents, web, http, data | `ToolEntry.category` enum | `src/tools/registry.py` | **Custom** | Categorical organization for tool selection by PlanningAgent |
| `ToolNode` (LG) | Scaffolded into generated LangGraph projects | `src/generation/graph_factory.py` | **Scaffolded** | `GraphFactory` generates `ToolNode` wired to registered tools |
| Source code scanning (`@tool`) | `ToolRegistry.scan()` extracts source + imports | `src/tools/registry.py` | **Custom** | Introspects source code for code generation; no LC equivalent |

### 1.5 Chains and LCEL (LangChain Expression Language)

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `RunnableSequence` (`\|` operator) | Used in all agent chains | `src/agents/*/` | **Native** | Standard LCEL pipe composition |
| `RunnableParallel` | Used in EnsembleRetriever for parallel retrieval | `src/agents/RAG/ensemble_retriever.py` | **Extended** | Augmented with asyncio.gather() for true parallelism |
| `RunnablePassthrough` | Used in RAG chains to pass query through | `src/agents/RAG/` | **Native** | Standard passthrough for context injection |
| `RunnableLambda` | Used for custom transformation steps | `src/agents/*/` | **Native** | Wraps arbitrary Python functions as Runnables |
| `RunnableBranch` | Used in SemanticRouter dispatch | `src/agents/RAG/semantic_router.py` | **Extended** | SemanticRouter implements more sophisticated routing |
| `RunnableWithFallbacks` | Used in LLMFactory for provider failover | `src/llm/factory.py` | **Native** | `.with_fallbacks()` on primary LLM |
| `RunnableConfig` | Passed through all agent invocations | `src/agents/*/` | **Native** | Carries thread_id, callbacks, run_id |
| `SimpleChain` generator | Generates linear LCEL chains | `src/generation/simple_chain.py` | **Scaffolded** | Emits LCEL pipe expressions for `simple_chain` tier |
| `create_retrieval_chain()` | Used in scaffolded RAG projects | GraphFactory RAG output | **Scaffolded** | Standard LC helper for generated projects |
| `create_history_aware_retriever()` | Scaffolded into generated conversational RAG | GraphFactory output | **Scaffolded** | History-aware retrieval in generated projects |

### 1.6 Callbacks

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `BaseCallbackHandler` | Custom handlers for Langfuse integration | `src/observability/` | **Extended** | Wraps Langfuse callbacks |
| `CallbackManager` | Attached to all LLM invocations via RunnableConfig | All agents | **Native** | Standard LC callback propagation |
| `LangSmithCallbackHandler` | Conditionally active when LANGSMITH_API_KEY set | `src/llm/factory.py` | **Native** | Environment-variable triggered |
| Streaming callbacks | Partially implemented | `src/agents/vision/` | **Partial** | Only VisionAgent streams; other agents buffer |
| `StdOutCallbackHandler` | Used in development/debug mode | `src/llm/factory.py` | **Native** | Debug logging |
| Custom cost callback | Tracks token usage for fitness evaluation | `src/agents/evolution/optimizer.py` | **Custom** | EvaluatorAgent's CostEstimation uses token counts |

---

## 2. LangChain Modules

### 2.1 Document Loaders

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `BaseLoader` | `KnowledgeGraphBuilder` ingestion | `src/agents/RAG/graph_builder.py` | **Native** | Uses LC loaders for document ingestion |
| `TextLoader` | Used in TreeIndexer for plain text | `src/agents/RAG/tree_indexer.py` | **Native** | Standard text file loading |
| `PDFLoader` / `PyPDFLoader` | Used in KnowledgeGraphBuilder | `src/agents/RAG/graph_builder.py` | **Native** | PDF document ingestion for graph building |
| `MarkdownLoader` | Used in GuideRetriever tree construction | `src/agents/RAG/guide_retriever.py` | **Native** | Markdown structure maps to GuideSection tree |
| `WebBaseLoader` | Used in PwCRetriever for paper fetching | `src/agents/RAG/pwc_retriever.py` | **Native** | Fetches Papers with Code web content |
| `GitLoader` | Used in CodeRetriever for repository loading | `src/agents/RAG/code_retriever.py` | **Native** | Loads code repositories for CodeRetriever |
| `DirectoryLoader` | Used in KnowledgeGraphBuilder batch ingestion | `src/agents/RAG/graph_builder.py` | **Native** | Batch document loading |
| Custom loader (multi-modal) | VectorStore multi-modal ingestion | `src/agents/RAG/vector_store.py` | **Custom** | Custom loaders for images, structured data alongside text |

### 2.2 Text Splitters

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `RecursiveCharacterTextSplitter` | Used in VectorStore chunking | `src/agents/RAG/vector_store.py` | **Native** | Standard recursive splitting for vector indexing |
| `MarkdownHeaderTextSplitter` | Used in TreeIndexer for section detection | `src/agents/RAG/tree_indexer.py` | **Extended** | Extended to preserve full parent-child hierarchy |
| `TokenTextSplitter` | Used for token-precise chunking | `src/agents/RAG/vector_store.py` | **Native** | Ensures chunks fit within embedding model limits |
| `PythonCodeTextSplitter` | Used in CodeRetriever | `src/agents/RAG/code_retriever.py` | **Native** | AST-aware Python code splitting |
| `Language` splitter (JS, TS, etc.) | Used in CodeRetriever for other languages | `src/agents/RAG/code_retriever.py` | **Native** | Language-specific code splitting |
| Custom chunk overlap strategy | VectorStore contextual chunking | `src/agents/RAG/vector_store.py` | **Custom** | Contextual embeddings prepend surrounding context before embedding |

### 2.3 Embeddings

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `Embeddings` (base class) | All embedding usage conforms to this | `src/agents/RAG/` | **Native** | Standard LC Embeddings interface |
| `OpenAIEmbeddings` | Default embeddings for VectorStore | `src/agents/RAG/vector_store.py` | **Native** | Used when provider=openai |
| `GoogleGenerativeAIEmbeddings` | Used with Gemini provider | `src/agents/RAG/vector_store.py` | **Native** | Gemini embedding models |
| `CohereEmbeddings` | Optional; used with Cohere Rerank pipeline | `src/agents/RAG/vector_store.py` | **Native** | Consistent embedding space with reranker |
| `OllamaEmbeddings` | Used for local inference scenarios | `src/agents/RAG/vector_store.py` | **Native** | Zero-cost local embeddings |
| `ClaraEncoder` | Contextual encoding layer | `src/agents/RAG/tree_indexer.py` | **Custom** | Prepends document context to chunks before embedding; no LC equivalent |
| Code-specific embeddings | CodeRetriever embedding model | `src/agents/RAG/code_retriever.py` | **Extended** | Code-optimized embedding selection |

### 2.4 Vector Stores

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `VectorStore` (base class) | `VectorStore` (custom, LanceDB) | `src/agents/RAG/vector_store.py` | **Extended** | Implements VectorStore interface over LanceDB; ~18K lines |
| `LanceDB` integration | Core vector storage | `src/agents/RAG/vector_store.py` | **Extended** | Custom wrapper beyond `langchain_community` LanceDB |
| `as_retriever()` | Exposed on VectorStore | `src/agents/RAG/vector_store.py` | **Native** | Returns BaseRetriever-compatible interface |
| `similarity_search()` | Used internally | `src/agents/RAG/vector_store.py` | **Native** | Dense similarity search |
| `similarity_search_with_score()` | Returns scores for RRF | `src/agents/RAG/vector_store.py` | **Native** | Scores passed to `ScoredResult` for RRF fusion |
| `max_marginal_relevance_search()` | Used for diversity-aware retrieval | `src/agents/RAG/vector_store.py` | **Native** | MMR for diverse result sets |
| FAISS / Chroma / Qdrant | Not used (LanceDB preferred) | — | **Unused** | LanceDB chosen for embedded, production-ready storage |

### 2.5 Retrievers

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `BaseRetriever` | All RAG components implement this | `src/agents/RAG/` | **Native** | All return `List[Document]` from `get_relevant_documents()` |
| `EnsembleRetriever` (LC) | Custom `EnsembleRetriever` | `src/agents/RAG/ensemble_retriever.py` | **Extended** | Superset: adds routing, CRAG, Cohere reranking, citation verification |
| `BM25Retriever` | Built into `VectorStore` | `src/agents/RAG/vector_store.py` | **Extended** | `rank_bm25` used directly; not the LC community wrapper |
| `ContextualCompressionRetriever` | Implicit in reranking step | `src/agents/RAG/rrf_fusion.py` | **Custom** | Reranking implemented directly vs. using LC wrapper |
| `CohereRerank` compressor | `rrf_fusion.rerank()` | `src/agents/RAG/rrf_fusion.py` | **Custom** | Direct Cohere API call; same effect as LC's CohereRerank |
| `MultiQueryRetriever` | Implicit in CRAG REWRITE loop | `src/agents/RAG/retrieval_gate.py` | **Custom** | Query rewriting achieves same goal as MultiQueryRetriever |
| `ParentDocumentRetriever` | `GuideRetriever` (conceptual analog) | `src/agents/RAG/guide_retriever.py` | **Custom** | Tree navigation vs. parent ID lookup |
| `SelfQueryRetriever` | Not implemented | — | **Planned** | Metadata-filtered retrieval is a future enhancement |
| `TimeWeightedVectorStoreRetriever` | Not implemented | — | **Unused** | Time-decay not relevant to current use cases |
| `GraphRetriever` (custom) | `GraphRetriever` — NetworkX | `src/agents/RAG/graph_retriever.py` | **Custom** | No LC equivalent; ~13K lines custom implementation |
| `GuideRetriever` (custom) | Hierarchical tree navigation | `src/agents/RAG/guide_retriever.py` | **Custom** | No LC equivalent; ~31K lines |
| `PwCRetriever` (custom) | Papers with Code retrieval | `src/agents/RAG/pwc_retriever.py` | **Custom** | Research literature retrieval; no LC equivalent |
| `CodeRetriever` (custom) | Code-specific retrieval | `src/agents/RAG/code_retriever.py` | **Custom** | Code-optimized retrieval; no LC equivalent |
| `CitationVerifier` (custom) | Statement-level source verification | `src/agents/RAG/verifier.py` | **Custom** | No LC equivalent; production hallucination prevention |
| `RetrievalGate` / CRAG (custom) | Quality gate with rewrite loop | `src/agents/RAG/retrieval_gate.py` | **Custom** | Implements CRAG pattern; no native LC equivalent |
| `SemanticRouter` (custom) | Query routing and classification | `src/agents/RAG/semantic_router.py` | **Custom** | Richer than LC's `RouterChain` |
| `RRFusion` (custom) | Reciprocal Rank Fusion | `src/agents/RAG/rrf_fusion.py` | **Custom** | LC's EnsembleRetriever uses similar math but meta-builder extends it |

### 2.6 Memory

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `MemoryChannel` (custom) | WorkflowDesign memory spec | `src/models/workflow_design.py` | **Custom** | 4 channel types: shared_state, message_passing, blackboard, episodic |
| `shared_state` channel | LangGraph state dict shared across all nodes | GraphFactory output | **Scaffolded** | Maps to LangGraph's state architecture |
| `message_passing` channel | Inter-node message exchange | GraphFactory output | **Scaffolded** | Analogous to LangGraph's `Send` API for dynamic edge routing |
| `blackboard` channel | Global write/read state (agent-agnostic) | GraphFactory output | **Custom** | Blackboard architecture pattern; no direct LC equivalent |
| `episodic` channel | Episodic memory (long-term context storage) | GraphFactory output | **Custom** | Vector-backed memory; analogous to `VectorStoreRetrieverMemory` (legacy) |
| `ConversationBufferMemory` (LC legacy) | Not used directly | — | **Unused** | Superseded by LangGraph's built-in state management |
| `ConversationSummaryMemory` (LC legacy) | Influence on EvaluatorAgent history | `src/agents/evaluator/` | **Custom** | Summary-style memory implemented without legacy class |
| `VectorStoreRetrieverMemory` | Conceptually in episodic channel | GraphFactory output | **Extended** | Episodic memory uses VectorStore retrieval but as custom impl |
| `BaseCheckpointSaver` | Used in LangGraph compiled graphs | `src/generation/graph_factory.py` | **Native** | `MemorySaver` for dev, PostgreSQL saver scaffolded for prod |

---

## 3. LangGraph

### 3.1 StateGraph and Core Graph Primitives

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `StateGraph` | Core of all `langgraph` tier outputs | `src/generation/graph_factory.py` | **Native** | `GraphFactory` produces compiled `StateGraph` instances |
| `StateGraph.compile()` | Always called with checkpointer | `src/generation/graph_factory.py` | **Native** | Returns `CompiledGraph` for execution |
| `StateGraph.add_node()` | Every `WorkflowNode` becomes a graph node | `src/generation/graph_factory.py` | **Native** | Node types: llm, tool, router, human, agent, file_io |
| `StateGraph.add_edge()` | `WorkflowEdge` (control/data) becomes graph edge | `src/generation/graph_factory.py` | **Native** | Standard edges for sequential flow |
| `StateGraph.add_conditional_edges()` | `WorkflowEdge` (conditional_routing) | `src/generation/graph_factory.py` | **Native** | Router nodes generate conditional edges |
| `StateGraph.set_entry_point()` | First node in `WorkflowDesign.nodes` | `src/generation/graph_factory.py` | **Native** | Derived from topology |
| `START` / `END` constants | Used in all generated graphs | `src/generation/graph_factory.py` | **Native** | Standard LG START/END |
| `MessageGraph` | Used for pure message-passing agents | GraphFactory (simple path) | **Native** | Simpler than StateGraph for conversational agents |
| `CompiledGraph.invoke()` | EvaluatorAgent uses to test generated graphs | `src/agents/evolution/optimizer.py` | **Native** | AgentGym runs compiled graphs for fitness evaluation |
| `CompiledGraph.ainvoke()` | Async graph execution in AgentGym | `src/agents/evolution/optimizer.py` | **Native** | Async execution for parallel evaluation |
| `CompiledGraph.stream()` | VisionAgent uses for streaming responses | `src/agents/vision/` | **Native** | Real-time streaming in conversational mode |

### 3.2 State and Annotations

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `TypedDict` state schema | `StateFieldSpec` → generated `TypedDict` | `src/models/workflow_design.py`, `src/generation/graph_factory.py` | **Scaffolded** | PlanningAgent specifies fields; GraphFactory generates TypedDict |
| `StateFieldSpec` | `WorkflowDesign.state_field_specs` | `src/models/workflow_design.py` | **Custom** | Specifies field type (str, list, dict, int, float, bool) and reducer |
| `Annotated[T, reducer]` | Generated from `StateFieldSpec` | `src/generation/graph_factory.py` | **Scaffolded** | `reducer` type in StateFieldSpec maps to `add_messages`, `operator.add`, or custom |
| `MessagesAnnotation` | Used in generated conversational agents | GraphFactory output | **Scaffolded** | Default for `llm` and `agent` node types |
| `Annotation.Root()` | Used in complex generated states | GraphFactory output | **Scaffolded** | For states with multiple annotated fields |
| `add_messages` reducer | Default for `messages` field | `src/generation/graph_factory.py` | **Native** | Applied to all message list state fields |
| Custom reducers | `StateFieldSpec.reducer` field | `src/models/workflow_design.py` | **Extended** | PlanningAgent can specify custom reducer logic |
| `IR/StateFieldPruning` pass | Removes unused state fields | `src/ir/passes/state_field_pruning.py` | **Custom** | Optimizer pass; no LG equivalent — optimizes generated state schemas |

### 3.3 Nodes

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| Node function (sync) | Generated for `sequential` topology nodes | GraphFactory output | **Scaffolded** | `def node_name(state: State) -> dict:` |
| Node function (async) | Generated for parallel/complex nodes | GraphFactory output | **Scaffolded** | `async def node_name(state: State) -> dict:` |
| `ToolNode` | Generated for `tool` WorkflowNode types | `src/generation/graph_factory.py` | **Scaffolded** | Wired to `ToolRegistry.resolve_tools()` output |
| `tools_condition` | Generated for `tool` exit edges | GraphFactory output | **Scaffolded** | Standard LG conditional for tool-calling agents |
| `llm` node type | `WorkflowNode(type="llm")` | `src/models/workflow_design.py` | **Scaffolded** | Generates LLM invocation node |
| `router` node type | `WorkflowNode(type="router")` | `src/models/workflow_design.py` | **Scaffolded** | Generates `add_conditional_edges` with routing function |
| `human` node type | `WorkflowNode(type="human")` | `src/models/workflow_design.py` | **Scaffolded** | Generates `interrupt()` node (see Human-in-Loop below) |
| `agent` node type | `WorkflowNode(type="agent")` | `src/models/workflow_design.py` | **Scaffolded** | Generates ReAct-style agent node (llm + tool routing) |
| `file_io` node type | `WorkflowNode(type="file_io")` | `src/models/workflow_design.py` | **Scaffolded** | File read/write operations using ToolRegistry file_io category |
| `IR/DeadNodeElimination` pass | Removes unreachable nodes pre-compilation | `src/ir/passes/dead_node_elimination.py` | **Custom** | IR pass; no LG equivalent |
| `IR/ParallelExtraction` pass | Annotates `fan_out_parallel` topology | `src/ir/passes/parallel_extraction.py` | **Custom** | Detects parallelizable nodes; generates `Send` API usage |
| `IR/CostEstimation` pass | Annotates nodes with token cost estimates | `src/ir/passes/cost_estimation.py` | **Custom** | google:2K, openai:3K, anthropic:4K, ollama:500 per node |

### 3.4 Edges

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| Normal edges | `WorkflowEdge` (control flow, no condition) | `src/models/workflow_design.py` | **Scaffolded** | Generates `add_edge(A, B)` |
| Conditional edges | `WorkflowEdge` (conditional_routing) | `src/models/workflow_design.py` | **Scaffolded** | Generates `add_conditional_edges(A, route_fn, {...})` |
| Data flow edges | `WorkflowEdge` (data flow) | `src/models/workflow_design.py` | **Custom** | Distinguishes control vs. data flow in IR; both become LG edges |
| `Send` API | Generated for `fan_out_parallel` topology | `src/generation/graph_factory.py` | **Scaffolded** | `ParallelExtraction` IR pass identifies candidates; `Send` enables dynamic parallel branches |
| `IR/LoopBoundInsertion` pass | Adds iteration caps to cyclic edges | `src/ir/passes/loop_bound_insertion.py` | **Custom** | Prevents infinite loops in cyclic_agentic topology |

### 3.5 Checkpointing

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `MemorySaver` | Development/testing checkpointer | `src/generation/graph_factory.py` | **Native** | Used in AgentGym testing |
| `SqliteSaver` | Scaffolded for lightweight production | GraphFactory output (conditional) | **Scaffolded** | Generated for non-distributed deployments |
| `PostgresSaver` | Scaffolded for distributed production | GraphFactory output (conditional) | **Scaffolded** | Generated when `orchestration_pattern = hierarchical/supervisor` |
| `BaseCheckpointSaver` | Interface for custom savers | `src/generation/graph_factory.py` | **Native** | Used to define saver injection |
| Thread-based execution | `thread_id` in `RunnableConfig` | All generated graphs | **Scaffolded** | Every conversation gets a unique `thread_id` |
| Checkpoint state history | `getStateHistory()` / `get_state_history()` | Not yet surfaced in meta-builder UI | **Planned** | Time-travel debugging not yet exposed |
| State persistence across runs | Via checkpointer in compiled graphs | All generated graphs | **Scaffolded** | Standard LG checkpointing |

### 3.6 Sub-Graphs and Hierarchical Agents

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| Sub-graph (nested `StateGraph`) | `hierarchical` orchestration pattern | `src/generation/graph_factory.py` | **Scaffolded** | Generated for `orchestration_pattern = hierarchical` |
| `supervisor` pattern | `orchestration_pattern = supervisor` | `src/models/workflow_design.py` | **Scaffolded** | Generates supervisor + worker sub-graph architecture |
| `swarm` pattern | `orchestration_pattern = swarm` | `src/models/workflow_design.py` | **Scaffolded** | Peer-to-peer agent swarm; generated via `Send` API |
| Parent-child state sharing | State schema inheritance in sub-graphs | GraphFactory hierarchical output | **Scaffolded** | Child graphs share subset of parent state |
| `invoke_subgraph()` | Generated for hierarchical pattern | GraphFactory output | **Scaffolded** | Sub-graph invocation within parent graph nodes |

### 3.7 Human-in-the-Loop

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `interrupt()` | Generated for `human` WorkflowNode type | `src/generation/graph_factory.py` | **Scaffolded** | Standard LG interrupt for human approval gates |
| `Command(resume=...)` | Scaffolded in human node handling | GraphFactory output | **Scaffolded** | Resume pattern generated with interrupt |
| `breakpoints` (compile-time) | Generated for debug builds | GraphFactory debug output | **Scaffolded** | `interrupt_before=[...]` on `compile()` |
| `update_state()` | Not currently generated | — | **Planned** | State editing for human correction not yet scaffolded |
| Time-travel (forking) | Not implemented | — | **Planned** | `get_state_history()` + `update_state()` fork pattern |
| Resume maps (interrupt IDs) | Not yet generated | — | **Planned** | Complex interrupt management not yet in GraphFactory |

### 3.8 Topology Mapping

| WorkflowDesign Topology | LangGraph Pattern | GraphFactory Implementation |
|---|---|---|
| `sequential` | Linear `add_edge` chain | `A → B → C → END` |
| `fan_out_parallel` | `Send` API with parallel branches | IR ParallelExtraction → `Send(node, state)` for each branch |
| `map_reduce` | Map: `Send` to parallel workers; Reduce: aggregator node | Two-phase: scatter then gather |
| `conditional_routing` | `add_conditional_edges` with router function | Router node → conditional dispatch |
| `cyclic_agentic` | Cycles allowed + loop bound from IR | `LoopBoundInsertion` adds iteration counter; cycle back allowed |

---

## 4. DeepAgents

### 4.1 Agent Harness

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| Deep Agent harness | `DeepAgentGenerator` | `src/generation/deep_agent_generator.py` | **Custom** | Generates full Deep Agent export including sandbox descriptors |
| `generation_tier = deep_agent` | Triggers `DeepAgentGenerator` instead of `GraphFactory` | `src/models/workflow_design.py` | **Custom** | Higher generation tier for complex multi-agent systems |
| `execution_descriptor` | `CandidateAgent.execution_descriptor` | `src/agents/evolution/optimizer.py` | **Custom** | Describes how to execute the candidate in sandbox |
| `sandbox_descriptor` | `CandidateAgent.sandbox_descriptor` | `src/agents/evolution/optimizer.py` | **Custom** | Describes sandbox requirements (memory, CPU, packages) |
| Agent harness export | Full project export with entry points | `DeepAgentGenerator` output | **Custom** | Produces runnable project, not just a graph object |

### 4.2 Sub-Agent Isolation

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| Sub-agent isolation | `SandboxPool` management | `src/agents/evolution/optimizer.py` | **Custom** | Each candidate runs in isolated sandbox |
| Agent boundaries | `CandidateAgent.design` (WorkflowDesign) | `src/agents/evolution/optimizer.py` | **Custom** | Each candidate has its own complete WorkflowDesign |
| State isolation | Separate state schemas per candidate | `src/agents/evolution/optimizer.py` | **Custom** | No shared state between candidate agents |
| Error isolation | RemediationAgent handles per-agent errors | `src/agents/remediation/` | **Custom** | Errors in one candidate don't affect others |
| `hierarchical` orchestration | Parent DeepAgent controls sub-agents | `src/models/workflow_design.py` | **Scaffolded** | DeepAgent harness can spawn LangGraph sub-graphs |

### 4.3 Sandbox Execution

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| Modal sandbox | `SandboxPool` — Modal provider | `src/agents/evolution/optimizer.py` | **Custom** | Modal.com for cloud GPU sandbox execution |
| Docker sandbox | `SandboxPool` — Docker provider | `src/agents/evolution/optimizer.py` | **Custom** | Local Docker for isolated execution |
| `AgentGym` | Test harness for fitness evaluation | `src/agents/evolution/optimizer.py` | **Custom** | Runs candidates against evaluation rubrics |
| `SandboxPool` | Pool of pre-warmed sandboxes | `src/agents/evolution/optimizer.py` | **Custom** | Reduces sandbox cold-start latency for evolution |
| Sandbox ephemeral execution | Each fitness evaluation is stateless | `src/agents/evolution/optimizer.py` | **Custom** | Sandbox torn down after evaluation |
| `execution_descriptor` → runtime | Maps descriptor to Modal/Docker invocation | `src/agents/evolution/optimizer.py` | **Custom** | Translates abstract descriptor to concrete run command |

### 4.4 Deep Agent Tier Evolution

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| `CandidateAgent` | Complete agent candidate with genome | `src/agents/evolution/optimizer.py` | **Custom** | Fields: design, graph, fitness, generation, rubric_dims, execution_descriptor, sandbox_descriptor |
| Genetic algorithm | `EvolutionaryOptimizer` core loop | `src/agents/evolution/optimizer.py` | **Custom** | Plan → build → test → evaluate → mutate → select |
| `GraphFactory` in evolution | Candidate `graph` field built by GraphFactory | `src/agents/evolution/optimizer.py` | **Native** | Evolution loop calls GraphFactory.build() per candidate |
| `AgentGym` testing | Evaluates candidate on test scenarios | `src/agents/evolution/optimizer.py` | **Custom** | Runs CompiledGraph.invoke() with test inputs |
| Fitness function | Quality(0.7) + Cost(0.2) + Latency(0.1) | `src/agents/evolution/optimizer.py` | **Custom** | Weighted multi-objective fitness |
| `rubric_dims` | Per-candidate evaluation dimensions | `src/agents/evolution/optimizer.py` | **Custom** | Domain-specific quality criteria |
| Mutation operators | Design mutations: add/remove/modify nodes | `src/agents/evolution/optimizer.py` | **Custom** | Mutates `WorkflowDesign` objects |
| Crossover operators | Combine two `WorkflowDesign` structures | `src/agents/evolution/optimizer.py` | **Custom** | Sub-graph-level crossover |
| MAP-Elites QD Archive | Quality-diversity archive of elite candidates | `src/agents/evolution/optimizer.py` | **Custom** | Discretized feature space; preserves diversity alongside quality |
| Fitness sharing | Penalizes overcrowded fitness regions | `src/agents/evolution/optimizer.py` | **Custom** | Standard QD diversity mechanism |
| Population persistence | Candidates persisted across sessions | `src/agents/evolution/optimizer.py` | **Custom** | Enables resume of evolution runs |
| Lineage tracking | Genealogy of each candidate | `src/agents/evolution/optimizer.py` | **Custom** | Tracks parent/child relationships across generations |

---

## 5. Ecosystem Infrastructure

### 5.1 Observability — Langfuse / LangSmith

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| LangSmith tracing | `LANGSMITH_API_KEY` env → auto-enabled | `src/llm/factory.py` | **Native** | Standard LC tracing via callback |
| `LangSmithCallbackHandler` | Attached to LLMFactory output | `src/llm/factory.py` | **Native** | Traces all LLM calls |
| Langfuse tracing | Custom callback handler | `src/observability/` | **Extended** | Parallel to LangSmith; custom Langfuse integration |
| Run naming (`run_name`) | All major agent calls include run_name | `src/agents/*/` | **Native** | Identifies agents in trace UI |
| Project-level tracing | `LANGCHAIN_PROJECT` env variable | `src/llm/factory.py` | **Native** | Groups traces by project |
| Trace metadata | `tags`, `metadata` in RunnableConfig | `src/agents/*/` | **Native** | Annotates traces with agent/tier metadata |
| EvaluatorAgent → LangSmith eval | Results can be pushed as evals | `src/agents/evaluator/` | **Planned** | LM-as-Judge scores not yet pushed to LangSmith |
| Cost tracking via callbacks | Token count extraction | `src/agents/evolution/optimizer.py` | **Extended** | Token counts drive fitness cost dimension |

### 5.2 LangGraph Studio

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| LangGraph Studio compatible graphs | All GraphFactory output | `src/generation/graph_factory.py` | **Scaffolded** | Generated graphs include `langgraph.json` configuration |
| `langgraph.json` | Generated project configuration | GraphFactory output | **Scaffolded** | Defines graph entrypoint for Studio |
| Visual graph inspection | Enabled via compiled graph | GraphFactory output | **Scaffolded** | Studio renders generated graphs automatically |
| Breakpoints in Studio | Scaffolded for `human` nodes | GraphFactory debug output | **Partial** | `interrupt_before` breakpoints generated |
| State inspection panel | Works automatically | — | **Native** | No extra work needed; Studio reads compiled graph state |
| Thread management | Via checkpointer config | GraphFactory output | **Scaffolded** | `thread_id` pattern used consistently |

### 5.3 Deployment

| Ecosystem Concept | Meta-Builder Component | File Location | Integration Depth | Notes |
|---|---|---|---|---|
| LangGraph Cloud / Platform | Not yet targeted | — | **Planned** | Future: deploy generated graphs to LangGraph Cloud |
| `langgraph-cli` | Not yet integrated | — | **Planned** | CLI for building and deploying LangGraph apps |
| FastAPI wrapper | Not generated currently | — | **Planned** | Generated projects don't yet include API server scaffolding |
| Docker export | `SandboxPool` Docker provider | `src/agents/evolution/optimizer.py` | **Custom** | Docker used for evolution sandboxes; not yet for deployment |
| Modal export | `DeepAgentGenerator` → Modal | `src/generation/deep_agent_generator.py` | **Custom** | Modal used for sandbox; extending to deployment is feasible |
| Streaming API | VisionAgent implements | `src/agents/vision/` | **Partial** | Not yet generalized to all generated graph APIs |

### 5.4 IR Compiler Passes — No Direct LangChain Equivalent

| IR Pass | What It Does | LangGraph Analog |
|---|---|---|
| `LoopBoundInsertion` | Adds iteration caps to cyclic edges | Manually added iteration counters in state |
| `DeadNodeElimination` | Removes unreachable nodes | N/A (LG has no static analysis) |
| `StateFieldPruning` | Removes unused state fields | N/A |
| `ParallelExtraction` | Identifies parallelizable nodes → `Send` | N/A (Send API used manually) |
| `CostEstimation` | Annotates nodes with token cost estimates | N/A |

The IR compiler is a unique meta-builder contribution with no LangChain/LangGraph equivalent. It is an intermediate representation layer between `WorkflowDesign` (the intent) and `StateGraph` (the execution artifact). The IR passes optimize the design before compilation.

---

## 6. Inverse Mapping — Meta-Builder Components Without LangChain Equivalents

These are meta-builder-v3 components that have no direct equivalent in the LangChain ecosystem. They represent original contributions.

| Meta-Builder Component | Description | Why No LC Equivalent |
|---|---|---|
| `PlanningAgent` (WorkflowDesign output) | Generates complete workflow specifications | LC doesn't generate other LC graphs |
| `IR Compiler Passes` (5 passes) | Static analysis and optimization of WorkflowDesign | No LC meta-programming layer |
| `EvolutionaryOptimizer` | Genetic algorithm for agent design optimization | LC doesn't self-optimize agent architectures |
| `MAP-Elites QD Archive` | Quality-diversity archive for evolved agents | Novel application of QD to agent evolution |
| `AgentGym` | Test harness for automated agent evaluation | LC evaluation is post-hoc; AgentGym is inline |
| `SandboxPool` | Pre-warmed Modal/Docker execution pools | LC doesn't manage execution sandboxes |
| `ClaraEncoder` | Contextual chunk encoding | Beyond standard embedding; custom innovation |
| `GuideRetriever` (hierarchical tree) | Tree-navigation retrieval | No LC equivalent for full tree traversal |
| `KnowledgeGraphBuilder` (33K lines) | Document → NetworkX knowledge graph | LC GraphCypherQAChain requires external Neo4j |
| `CitationVerifier` | Statement-level source verification | No LC runtime hallucination gate |
| `RagTier` enum | Progressive capability configuration | LC doesn't tier retrieval capabilities |
| `VisionAgent` | Conversational brainstorming partner | Specialized conversational meta-agent |
| `CritiqueRefiner` | Natural language critique → design updates | No LC agent self-modification pattern |
| `RemediationAgent` | Error-based code fixes | Specialized code repair agent |
| `ResearchOrchestrator` | Research pipeline (arXiv/PwC monitoring) | Domain-specific research automation |
| `Blueprint Validator` | Pre-build WorkflowDesign validation | No LC schema validation for graph designs |
| `TOOL_IMPORTS` metadata | Tracks package deps per tool for scaffolding | LC tools don't track their own pip packages |
| Call-site model resolver | Different models for different operations | LC doesn't natively route per-operation |

---

*End of File 2: Component Mapping Matrix*

*See also: [05-rag-system-analysis.md](./05-rag-system-analysis.md) for deep RAG coverage, and [07-enhancement-opportunities.md](./07-enhancement-opportunities.md) for improvement analysis.*
