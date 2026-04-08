# 05 — RAG System Analysis: Meta-Builder-V3 Multi-Route Ensemble Architecture

> **Project:** meta-builder-v3 (v4.9.0) — AI-Native Orchestration Compiler  
> **Focus:** `src/agents/RAG/` — The Quad-Hybrid RAG system upgraded to Multi-Route Ensemble  
> **Date:** April 2026

---

## Table of Contents

1. [Executive Overview](#1-executive-overview)
2. [Architecture: Multi-Route Ensemble Design](#2-architecture-multi-route-ensemble-design)
3. [Component Deep-Dives](#3-component-deep-dives)
   - 3.1 EnsembleRetriever — The Orchestrator
   - 3.2 VectorStore — LanceDB + BM25 Hybrid
   - 3.3 GraphRetriever — Knowledge Graph Traversal
   - 3.4 KnowledgeGraphBuilder — Graph Construction
   - 3.5 GuideRetriever — Hierarchical Tree Navigation
   - 3.6 TreeIndexer — Document Tree Construction
   - 3.7 SemanticRouter — Query Classification and Routing
   - 3.8 RetrievalGate — CRAG Quality Gate
   - 3.9 CitationVerifier — Source Verification Pipeline
   - 3.10 RRF Fusion — Reciprocal Rank Fusion + Reranking
   - 3.11 PwCRetriever — Papers with Code Retrieval
   - 3.12 CodeRetriever — Code-Specific Retrieval
4. [RagTier Enum — Progressive Capability System](#4-ragtier-enum--progressive-capability-system)
5. [LangChain Abstraction Mapping](#5-langchain-abstraction-mapping)
6. [SemanticRouter and Query Routing Patterns](#6-semanticrouter-and-query-routing-patterns)
7. [CRAG Quality Gate and Self-Corrective RAG](#7-crag-quality-gate-and-self-corrective-rag)
8. [RRF Fusion Pipeline and Reranking](#8-rrf-fusion-pipeline-and-reranking)
9. [Citation Verification Pipeline](#9-citation-verification-pipeline)
10. [RAG Scaffolding in Generated Projects](#10-rag-scaffolding-in-generated-projects)
11. [Comparison with Standard LangChain RAG Patterns](#11-comparison-with-standard-langchain-rag-patterns)
12. [Data Flow Walkthrough: End-to-End Query](#12-data-flow-walkthrough-end-to-end-query)
13. [Dependency and Compatibility Analysis](#13-dependency-and-compatibility-analysis)

---

## 1. Executive Overview

Meta-builder-v3's RAG system represents a significant evolution beyond standard LangChain retrieval patterns. Rather than a simple vector-search pipeline, it implements a **Multi-Route Ensemble** architecture that dynamically selects retrieval strategies based on query semantics and complexity, fuses results from multiple heterogeneous retrievers, applies cross-encoder reranking, and enforces a quality gate that can trigger autonomous query rewriting.

The system operates across five capability tiers (`RagTier.NONE` through `RagTier.MAXIMUM`), enabling the same codebase to run from a pure-LLM zero-retrieval mode up to a full ensemble with CLaRa contextual enrichment, Corrective RAG (CRAG) quality gating, and citation verification.

**Key differentiators from baseline LangChain RAG:**

| Dimension | Standard LangChain | Meta-Builder RAG |
|---|---|---|
| Retriever count | 1 (usually) | 4+ parallel (vector, graph, guide, code, PwC) |
| Fusion strategy | None or simple union | Reciprocal Rank Fusion (RRF) → top-20 |
| Reranking | Optional (CohereRerank wrapper) | Built-in Cohere Rerank → top-10 |
| Quality assurance | None | CRAG gate: PASS / REWRITE loop (max 2 iterations) |
| Query routing | None | SemanticRouter: DIRECT / VECTOR / GRAPH / GUIDE routes |
| Citation | None | CitationVerifier: statement-level source verification |
| Knowledge graph | Rare | First-class: NetworkX graph traversal |
| Document trees | None | GuideRetriever with hierarchical GuideSection navigation |
| Contextual encoding | Embeddings only | ClaraEncoder for contextual chunk-level encoding |

The RAG subsystem in `src/agents/RAG/` has approximately **127K lines** of specialized retrieval code, making it one of the largest subsystems in the project.

---

## 2. Architecture: Multi-Route Ensemble Design

### 2.1 High-Level Pipeline

```
User Query
    │
    ▼
SemanticRouter (semantic_router.py)
    │ RouteType: DIRECT | VECTOR | GRAPH | GUIDE | ENSEMBLE
    │ QueryComplexity: SIMPLE | MODERATE | COMPLEX | EXPERT
    │
    ├──► DIRECT route ──────────────────────────────► LLM (no retrieval)
    │
    └──► Active Retriever Selection
              │
              ├── VectorStore retriever (LanceDB + BM25)
              ├── GraphRetriever (NetworkX)     } asyncio.gather()
              ├── GuideRetriever (tree nav)     } parallel execution
              ├── PwCRetriever (papers)         }
              └── CodeRetriever (code)          }
                        │
                        ▼
              RRF Fusion (rrf_fusion.py)
              → Merged ranked list (top-20)
                        │
                        ▼
              Cohere Rerank API
              → top-10 cross-encoder ranked results
                        │
                        ▼
              RetrievalGate / CRAG (retrieval_gate.py)
              GateDecision: PASS | REWRITE | FAIL
                   │
                   ├──► PASS ─────────────────────────► Optional CLaRa enrichment
                   │                                        │
                   └──► REWRITE ──► Query rewrite ──────────┘
                         (max 2 iterations)               │
                                                          ▼
                                              CitationVerifier (verifier.py)
                                              VerifiedStatement models
                                                          │
                                                          ▼
                                              Final verified results
```

### 2.2 Parallel Execution Architecture

The EnsembleRetriever fires all active retrievers simultaneously using `asyncio.gather()`. This is architecturally superior to sequential retrieval:

- **Latency**: Total retrieval time ≈ max(individual retriever times), not the sum
- **Diversity**: Vector, graph, and guide results are fundamentally different document representations
- **Robustness**: If one retriever fails or returns poor results, others compensate

This pattern directly implements what LangChain's documentation describes as "retriever parallelism" but without a native parallel orchestration primitive — meta-builder builds this explicitly with asyncio.

### 2.3 The Ensemble vs. Single-Retriever Tradeoff

The ensemble trades retrieval latency (parallel calls still take time) for dramatically higher recall. The RRF fusion step is the critical enabler: it produces a unified ranking from heterogeneous score spaces (cosine similarity from vectors, graph centrality scores, BM25 TF-IDF scores) without requiring a common scoring scale.

After RRF fusion narrows the candidate set to 20, Cohere's cross-encoder reranker produces a semantically calibrated top-10. This two-stage compression (many → 20 → 10) is a deliberate design choice that mirrors recommendations from the RAG literature on "retrieve many, rerank few."

---

## 3. Component Deep-Dives

### 3.1 EnsembleRetriever — The Orchestrator

**File:** `src/agents/RAG/ensemble_retriever.py` (419 lines)

The EnsembleRetriever is the central coordinator of the entire RAG pipeline. It does not retrieve documents itself — it orchestrates all other retrievers through a defined execution protocol.

**Core responsibilities:**
1. Accept a query and a `RagTier` configuration
2. Invoke `SemanticRouter` to classify the query into a `RouteResult` containing `RouteType` and `QueryComplexity`
3. For DIRECT routes, immediately return an empty retrieval result (LLM answers from parametric memory)
4. For all retrieval routes, determine which retrievers are active based on `RagTier` and `RouteType`
5. Execute active retrievers in parallel via `asyncio.gather()`
6. Pass all ranked lists to `RRF Fusion` → top-20
7. Call Cohere Rerank API → top-10
8. Evaluate quality via `RetrievalGate` (CRAG)
9. If gate returns REWRITE, rewrite the query and loop (max 2 iterations total)
10. If gate returns PASS, optionally enrich with CLaRa
11. Run results through `CitationVerifier` for statement-level verification
12. Return final `VerifiedStatement` list

**Key design patterns:**
- The 419-line size suggests a tightly focused orchestrator without unnecessary abstractions
- Retry logic is bounded: max 2 REWRITE iterations prevents infinite loops
- CLaRa enrichment is optional (`if rag_tier == RagTier.MAXIMUM`) — CLaRa adds latency but improves context coherence

**Comparison to LangChain's `EnsembleRetriever`:**

LangChain's built-in `EnsembleRetriever` (from `langchain.retrievers`) accepts a list of retrievers and weights, applies RRF internally, and returns a fused list. It is synchronous by default and does not include:
- Query routing
- Quality gating
- Cross-encoder reranking
- Citation verification

Meta-builder's `EnsembleRetriever` is a superset that uses the same conceptual name but implements a full multi-stage pipeline. It essentially wraps LangChain's concept with production-grade orchestration.

---

### 3.2 VectorStore — LanceDB + BM25 Hybrid Search

**File:** `src/agents/RAG/vector_store.py` (~18K lines)

The VectorStore component implements hybrid search combining dense vector retrieval (semantic) and sparse keyword retrieval (BM25). This is the foundational retriever that all RagTiers above NONE use.

**LanceDB integration:**
- LanceDB is an embedded vector database optimized for large-scale similarity search
- Supports on-disk storage with zero infrastructure (no separate server process)
- The choice of LanceDB over FAISS or Chroma reflects a preference for production-ready embedded storage
- LanceDB's columnar storage format enables efficient batch operations

**BM25 integration:**
- `rank_bm25>=0.2.0` dependency
- BM25 provides lexical precision that dense vectors often miss for exact terminology (API names, error codes, library versions)
- The hybrid combination addresses the classic recall-precision tradeoff: BM25 handles exact matches, vectors handle semantic similarity

**Contextual embeddings:**
- "Contextual embeddings" suggests the VectorStore doesn't simply embed isolated chunks — it prepends or appends surrounding context before embedding, similar to Anthropic's "contextual retrieval" technique
- This improves chunk-level embedding quality by preserving document-level context

**Multi-modal support:**
- The `~18K` line count suggests substantial complexity beyond basic text retrieval
- Multi-modal support implies the ability to index and retrieve images, code, diagrams, or structured data alongside text

**LangChain mapping:**
The VectorStore is a custom implementation sitting where LangChain's `VectorStore` abstract base class would sit (`langchain_core.vectorstores.VectorStore`). The `as_retriever()` pattern is preserved — the VectorStore exposes a retriever interface compatible with LangChain's `BaseRetriever`.

---

### 3.3 GraphRetriever — Knowledge Graph Traversal

**File:** `src/agents/RAG/graph_retriever.py` (~13K lines)

The GraphRetriever implements multi-hop reasoning over a NetworkX knowledge graph. Where vector similarity finds semantically adjacent content, graph traversal finds relationally connected content.

**GraphExpansion mechanism:**
- Given a query entity or concept, GraphExpansion traverses the knowledge graph outward
- Multi-hop: can follow entity → relation → entity → relation → entity chains
- This is particularly powerful for causal questions ("why does X cause Y?") and compositional questions ("what is the relationship between A and B through C?")

**NetworkX dependency:**
- `networkx>=3.0` is required for STANDARD tier and above
- NetworkX operates entirely in-memory — for large graphs, this creates memory pressure
- The graph is pre-built by `KnowledgeGraphBuilder` and serialized/loaded

**Integration with EnsembleRetriever:**
- The graph retriever returns `Document` objects (LangChain-compatible) with metadata including the traversal path taken (hop sequence)
- This path metadata enables downstream components (especially CitationVerifier) to trace the reasoning chain

**LangChain mapping:**
LangChain has no native knowledge graph retriever in its core library. The closest analogs are:
- `Neo4jGraph` and `GraphCypherQAChain` (requires Neo4j infrastructure)
- `NetworkxEntityGraph` in `langchain_community` (very basic entity extraction)

Meta-builder's `GraphRetriever` is significantly more sophisticated: it builds its own graph from documents via `KnowledgeGraphBuilder` and implements custom multi-hop traversal with `GraphExpansion`.

---

### 3.4 KnowledgeGraphBuilder — Graph Construction

**File:** `src/agents/RAG/graph_builder.py` (~33K lines)

The `KnowledgeGraphBuilder` is the largest component in the RAG subsystem at ~33K lines, reflecting the complexity of converting unstructured documents into a structured knowledge graph.

**Core pipeline:**
1. **Document ingestion**: accepts raw documents (PDFs, markdown, code, etc.)
2. **Entity extraction**: identifies named entities, concepts, technical terms
3. **Relation extraction**: identifies relationships between entities (uses_library, implements_interface, extends_class, etc.)
4. **Graph construction**: builds a directed NetworkX graph with entities as nodes and relations as edges
5. **Edge weighting**: edges receive confidence scores from the extraction process
6. **Serialization**: graph is serialized for persistence and reloading

**The 33K line count suggests:**
- Multiple extraction strategies (rule-based, LLM-based, hybrid)
- Support for multiple document types with specialized parsers
- Graph update and merge operations (adding new documents without rebuilding)
- Graph quality metrics and pruning
- Custom relation types specific to the technical domain (software architecture, AI/ML concepts)

**LangChain mapping:**
- Partially maps to `langchain_community.graphs.NetworkxEntityGraph`
- Also analogous to LangGraph's concept of "state as a graph" but applied to knowledge, not execution
- The knowledge graph extraction pipeline resembles LangChain's information extraction chains

---

### 3.5 GuideRetriever — Hierarchical Tree Navigation

**File:** `src/agents/RAG/guide_retriever.py` (~31K lines)

The `GuideRetriever` implements a fundamentally different retrieval paradigm: instead of similarity search, it navigates a pre-built hierarchical tree of document sections to find the structurally most relevant content.

**GuideSection model:**
- Represents a node in the document tree
- Contains: section title, content, depth level, parent reference, child references, embedding
- The tree mirrors the logical structure of documentation (chapter → section → subsection → paragraph)

**Hierarchical navigation algorithm:**
1. Given a query, embed it and compare against section embeddings at each tree level
2. Select the most relevant section at the top level
3. Descend into that section's children, repeating the comparison
4. Continue until reaching leaf nodes or a configurable depth limit
5. Return the leaf content plus context from ancestor sections

**Why hierarchical retrieval?**

Standard chunking loses document structure. A query about "authentication in the Advanced tier" might find isolated chunks that lack the context that "Advanced tier" refers to a specific configuration section. GuideRetriever preserves this by navigating the table of contents.

**LangChain mapping:**
- Most closely maps to `ParentDocumentRetriever` from `langchain_classic`
- `ParentDocumentRetriever` retrieves small chunks but returns parent documents
- GuideRetriever goes further: it retrieves contextually by navigating the tree rather than simply looking up parents by ID
- Also comparable to LangChain's `MultiVectorRetriever` for structured document navigation

---

### 3.6 TreeIndexer — Document Tree Construction

**File:** `src/agents/RAG/tree_indexer.py` (~17K lines)

The `TreeIndexer` builds the hierarchical document trees consumed by `GuideRetriever`. It is the construction counterpart to GuideRetriever's querying.

**ClaraEncoder:**
- The TreeIndexer uses `ClaraEncoder` for contextual encoding
- `ClaraEncoder` (CLaRa = Contextual Language Representation) encodes each chunk with awareness of its position in the document hierarchy
- This is analogous to Anthropic's "contextual retrieval" approach where each chunk is prefixed with its document-level context before embedding

**Tree construction pipeline:**
1. Parse documents into sections using structural markers (headers, numbering, indentation)
2. Build parent-child relationships based on section hierarchy
3. Encode each section with `ClaraEncoder` (context-aware embeddings)
4. Store in the `GuideSection` tree model
5. Serialize tree for GuideRetriever consumption

**LangChain mapping:**
- Analogous to LangChain's text splitters (especially `MarkdownHeaderTextSplitter`)
- Goes beyond splitting by maintaining the full tree structure rather than a flat list of chunks
- The ClaraEncoder's contextual embedding is more sophisticated than standard `langchain_openai.OpenAIEmbeddings`

---

### 3.7 SemanticRouter — Query Classification and Routing

**File:** `src/agents/RAG/semantic_router.py` (~7K lines)

The `SemanticRouter` is the decision gate that determines which retrieval path a query should follow. It examines query characteristics and produces a `RouteResult` that the `EnsembleRetriever` uses to configure its execution.

**Key types:**

```python
class RouteType(Enum):
    DIRECT    # Skip retrieval entirely; LLM answers from parametric knowledge
    VECTOR    # Standard vector + BM25 similarity search
    GRAPH     # Knowledge graph traversal for relational queries
    GUIDE     # Hierarchical document tree navigation
    ENSEMBLE  # All retrievers active (maximum coverage)

class QueryComplexity(Enum):
    SIMPLE    # Single-fact, direct answer expected
    MODERATE  # Multi-step but well-defined answer
    COMPLEX   # Requires synthesis across multiple sources
    EXPERT    # Requires deep domain knowledge and multi-hop reasoning

class RouteResult:
    route: RouteType
    complexity: QueryComplexity
    confidence: float
    metadata: dict  # additional routing context
```

**Routing logic:**

The SemanticRouter uses a combination of:
1. **Classifier**: a trained or few-shot prompted model that maps query features to RouteType
2. **Complexity estimator**: heuristics (entity count, question type, query length) combined with model scoring
3. **Confidence thresholding**: low-confidence routes may default to ENSEMBLE

**Route selection heuristics:**
- `DIRECT`: greetings, simple definitions, queries about the LLM's own capabilities
- `GRAPH`: queries with explicit relational structure ("how does X relate to Y", "why does A cause B")
- `GUIDE`: "how to" queries, procedural questions, section-lookup patterns
- `VECTOR`: general technical questions, "what is" queries
- `ENSEMBLE`: ambiguous queries, high complexity score, multi-part questions

**LangChain mapping:**

LangChain does not have a native `SemanticRouter` in its core library. The closest analogs:
- **`RunnableBranch`**: routes between different chains based on conditions, but uses simple predicates, not ML classification
- **`RouterChain`** (legacy): routes to named chains based on classifier output
- **`MultiPromptChain`** (legacy): routes queries to prompt-specialized chains

Meta-builder's SemanticRouter is more sophisticated than all of these: it classifies into a richer taxonomy (RouteType × QueryComplexity) and its output configures the entire retrieval pipeline, not just which prompt template to use.

The `semantic-router` PyPI package (from Aurelio AI) provides the nearest equivalent in the open-source ecosystem, but meta-builder's implementation appears to be a custom build rather than a wrapper.

---

### 3.8 RetrievalGate — CRAG Quality Gate

**File:** `src/agents/RAG/retrieval_gate.py` (~9K lines)

The `RetrievalGate` implements Corrective RAG (CRAG), an agentic self-correction loop that evaluates retrieval quality before allowing the system to generate a response.

**Key types:**

```python
class GateDecision(Enum):
    PASS    # Retrieved documents are sufficiently relevant; proceed to generation
    REWRITE # Query should be rewritten and retrieval retried
    FAIL    # Retrieval has failed after maximum iterations

class GateResult:
    decision: GateDecision
    confidence: float
    reason: str
    iteration: int  # current iteration count (1 or 2)
    rewritten_query: Optional[str]  # populated on REWRITE decision
```

**Evaluation criteria:**

The gate evaluates:
1. **Relevance**: are the retrieved documents topically relevant to the query?
2. **Coverage**: do the documents collectively cover the query's information needs?
3. **Coherence**: are the documents internally consistent without contradictions?
4. **Specificity**: are the documents specific enough (not just tangentially related)?

**CRAG loop mechanics:**

```
Iteration 1:
  RetrievalGate evaluates top-10 reranked results
  → PASS: proceed to CLaRa enrichment
  → REWRITE: generate rewritten query, re-run entire pipeline from SemanticRouter
  → FAIL: propagate failure (very rare, usually on malformed queries)

Iteration 2 (if REWRITE was triggered):
  Re-run with rewritten query
  RetrievalGate evaluates new results
  → PASS: proceed
  → REWRITE: still fail gracefully (max 2 iterations enforced in EnsembleRetriever)
  → FAIL: propagate failure
```

**Query rewriting strategy:**

When the gate decides REWRITE, it generates a rewritten query by:
1. Identifying which aspects of the query led to poor retrieval (too vague, wrong terminology, too narrow)
2. Expanding or reformulating the query to improve recall
3. Ensuring the rewritten query preserves the original intent

**LangChain/LangGraph mapping:**

The CRAG gate directly implements the pattern described in the [LangChain CRAG blog post](https://blog.langchain.com/agentic-rag-with-langgraph/) and the [Self-Reflective RAG paper](https://arxiv.org/abs/2310.11511). In a pure LangGraph implementation, this would be a StateGraph with:
- `retrieve_node` → `grade_node` → conditional edge → `generate_node` or `rewrite_node`
- `rewrite_node` → `retrieve_node` (loop back)
- Iteration counter in state to enforce max retries

Meta-builder implements this same logic but as a method call within `EnsembleRetriever` rather than as a separate LangGraph sub-graph. This is a deliberate architectural choice: keeping the CRAG logic internal to the retriever avoids exposing retrieval implementation details to the top-level workflow graph.

---

### 3.9 CitationVerifier — Source Verification Pipeline

**File:** `src/agents/RAG/verifier.py` (~9K lines)

The `CitationVerifier` performs the final step of the RAG pipeline: verifying that every statement in a potential response is actually supported by the retrieved documents.

**VerifiedStatement model:**

```python
class VerifiedStatement:
    statement: str          # The claim or assertion
    supported: bool         # Is it supported by source documents?
    source_ids: List[str]   # IDs of supporting documents
    confidence: float       # Confidence in the verification
    verbatim_evidence: str  # The relevant passage from source
```

**Verification pipeline:**
1. **Statement extraction**: decompose the candidate response into individual verifiable claims
2. **Evidence retrieval**: for each claim, find the most relevant passage in the retrieved documents
3. **Entailment checking**: determine if the passage entails (supports) the claim
4. **Confidence scoring**: score each verification on a continuous scale
5. **Flagging**: low-confidence or unsupported statements are flagged for removal or caveat

**Why statement-level verification?**

Document-level retrieval can produce retrieved documents that are broadly relevant but don't actually support every specific claim in a generated response. The CitationVerifier prevents hallucinations by enforcing evidence requirements at the claim level.

**LangChain mapping:**
- Closest analog is LangChain's `FlashrankRerank` or `CohereRerank` post-processing
- More directly analogous to `langchain_community.document_transformers.EmbeddingsRedundantFilter`
- No native "citation verifier" exists in LangChain core; this is a custom meta-builder contribution
- The pattern resembles RAGAS's `faithfulness` metric (verifying response grounding) but implemented as a generation-time gate rather than an evaluation metric

---

### 3.10 RRF Fusion — Reciprocal Rank Fusion + Reranking

**File:** `src/agents/RAG/rrf_fusion.py` (~7K lines)

The `rrf_fusion.py` module implements Reciprocal Rank Fusion and the reranking wrapper.

**Key types:**

```python
class ScoredResult:
    document: Document
    score: float
    rank: int
    retriever_source: str  # which retriever produced this result
```

**Reciprocal Rank Fusion formula:**

For a document \(d\) across \(k\) ranking lists, the RRF score is:

\[ \text{RRF}(d) = \sum_{i=1}^{k} \frac{1}{c + r_i(d)} \]

where \(r_i(d)\) is the rank of document \(d\) in retrieval list \(i\), and \(c = 60\) is a constant (standard value from the RRF paper).

**Why RRF over score fusion?**

Different retrievers use incompatible scoring scales:
- BM25 scores: unbounded positive values with different scales per corpus
- Cosine similarity: range [-1, 1]
- Graph centrality: PageRank-style probabilities [0, 1]

Score normalization before fusion introduces arbitrary choices. RRF is score-scale-agnostic — it only uses rank positions, which are universally comparable.

**Two-stage compression:**

```
Raw results: N_vector + N_graph + N_guide + ... documents (potentially 50-200 total)
       ↓
RRF Fusion → top-20 (eliminates duplicates, creates unified ranking)
       ↓
Cohere Rerank → top-10 (cross-encoder semantic scoring)
```

The `rerank()` function wraps the Cohere Rerank API call. Cohere's cross-encoder scores each (query, document) pair jointly — unlike bi-encoder embeddings that score query and document independently, cross-encoders see both simultaneously and produce more accurate relevance judgments.

**LangChain mapping:**
- LangChain's built-in `EnsembleRetriever` uses the same RRF formula
- LangChain's `ContextualCompressionRetriever` with `CohereRerank` compressor performs the reranking step
- Meta-builder combines both into a single cohesive module with explicit `ScoredResult` tracking for provenance

---

### 3.11 PwCRetriever — Papers with Code Retrieval

**File:** `src/agents/RAG/pwc_retriever.py` (~6K lines)

The `PwCRetriever` provides specialized retrieval from the Papers with Code dataset and API, enabling the system to retrieve research papers and their associated code repositories.

**Use cases:**
- Retrieving benchmark results for specific tasks or architectures
- Finding state-of-the-art methods for a problem domain
- Linking implementation code to paper claims

**Integration with EnsembleRetriever:**
- Active when route complexity is EXPERT or when the query mentions research domains
- Returns `Document` objects with metadata including: paper title, authors, publication date, task, dataset, metric, code URL

**Connection to Research Department:**
- `ScoutMonitor` in `src/agents/research/` uses similar arXiv/PwC monitoring
- `PwCRetriever` provides the retrieval layer for the research pipeline's context needs

---

### 3.12 CodeRetriever — Code-Specific Retrieval

**File:** `src/agents/RAG/code_retriever.py` (~5K lines)

The `CodeRetriever` implements specialized retrieval for code artifacts. Standard text embeddings perform poorly on code because code has fundamentally different structure than prose.

**Key capabilities:**
- Code-specific embedding models (e.g., CodeBERT, code-oriented OpenAI embeddings)
- AST-aware chunking: split code at function/class boundaries rather than arbitrary token windows
- Syntax-sensitive search: recognizes that `function_name()` and `function_name` are the same query target
- Language-aware: handles Python, JavaScript, TypeScript, and other common languages differently

**When active:**
- For `STANDARD` tier and above when query contains code, function names, or programming concepts
- When `RouteType` indicates a code-lookup pattern ("how do I implement X", "show me the code for Y")

---

## 4. RagTier Enum — Progressive Capability System

**File:** `src/models/rag_tiers.py`

The `RagTier` enum enables a project generated by meta-builder to be configured with exactly the retrieval capability it needs, from zero to maximum.

### 4.1 Tier Definitions

| Tier | Value | Description | Active Components |
|---|---|---|---|
| `NONE` | 0 | Pure LLM, no retrieval | LLM only (parametric memory) |
| `BASIC` | 1 | Vector search | VectorStore (LanceDB + BM25) |
| `STANDARD` | 2 | + Graph RAG | VectorStore + GraphRetriever |
| `ADVANCED` | 3 | + Tree navigation | VectorStore + GraphRetriever + GuideRetriever |
| `MAXIMUM` | 4 | Full ensemble | All + CLaRa + CitationVerifier + CRAG gate |

### 4.2 Tier Requirements

```python
# RagTier.BASIC
requirements = [
    "lancedb>=0.4.0",   # Vector storage
    "rank_bm25>=0.2.0", # Sparse retrieval
]

# RagTier.STANDARD (+ BASIC)
requirements += [
    "networkx>=3.0",    # Graph operations
]

# RagTier.ADVANCED and MAXIMUM
# guide_retriever, tree_indexer, verifier, retrieval_gate
# depend only on langchain_core (stdlib-level additions)
# No additional third-party packages beyond STANDARD
```

### 4.3 Tier Selection Logic

The `PlanningAgent` determines `RagTier` based on:
- **Domain**: research, documentation, code → higher tiers
- **Data type**: unstructured documents → STANDARD+; structured databases → BASIC or NONE
- **Complexity**: simple Q&A → BASIC; multi-hop reasoning → STANDARD+
- **Cost sensitivity**: MAXIMUM adds Cohere API call costs per query

### 4.4 Progressive Enhancement Philosophy

The RagTier system reflects a key meta-builder design principle: **generated projects should be as capable as needed but no more complex than necessary**. A chatbot for a small FAQ base doesn't need NetworkX graph traversal. A research assistant synthesizing arXiv papers needs MAXIMUM.

This is architecturally analogous to LangChain's layered package structure (`langchain-core` < `langchain` < `langchain-community` < integration packages), but applied to runtime capability rather than package dependencies.

---

## 5. LangChain Abstraction Mapping

### 5.1 Core LangChain Concepts and Their Meta-Builder Equivalents

| LangChain Abstraction | Meta-Builder Component | Notes |
|---|---|---|
| `BaseRetriever` | All retriever components | All expose `get_relevant_documents()` |
| `EnsembleRetriever` | `EnsembleRetriever` (custom) | Superset with routing, CRAG, reranking |
| `VectorStore` | `VectorStore` (LanceDB) | Custom implementation over LanceDB |
| `BM25Retriever` | Built into `VectorStore` | Part of hybrid search |
| `ContextualCompressionRetriever` | EnsembleRetriever pipeline | Cohere reranking is the compressor |
| `CohereRerank` | `rrf_fusion.rerank()` | Direct Cohere API integration |
| `Document` | Standard LangChain `Document` | All retrievers return LC-compatible Documents |
| `Embeddings` | `ClaraEncoder`, standard embeddings | Contextual encoding layer |
| `RouterChain` | `SemanticRouter` | More sophisticated, ML-based routing |
| `MultiQueryRetriever` | Implicit in REWRITE loop | Query rewriting serves similar purpose |
| `ParentDocumentRetriever` | `GuideRetriever` | Tree-based vs. ID-based parent lookup |

### 5.2 Document Model Compatibility

All meta-builder retrievers return standard LangChain `Document` objects:

```python
from langchain_core.documents import Document

Document(
    page_content="...",
    metadata={
        "source": "...",
        "retriever": "graph",     # which retriever produced this
        "score": 0.87,            # retriever-specific score
        "rrf_score": 0.034,       # post-fusion RRF score
        "cohere_rank": 3,         # post-reranking position
        "hop_path": ["A", "B"],   # for graph retriever
        "section_path": [...],    # for guide retriever
    }
)
```

This LangChain-compatible document model means that the RAG pipeline output can be dropped into any standard LangChain chain without adaptation.

---

## 6. SemanticRouter and Query Routing Patterns

### 6.1 Route Classification in Practice

The SemanticRouter's classification has direct implications for retrieval quality and cost:

| Query Example | Route | Complexity | Rationale |
|---|---|---|---|
| "What is LangGraph?" | VECTOR | SIMPLE | Standard definition lookup |
| "What is your name?" | DIRECT | SIMPLE | About the LLM itself, no retrieval needed |
| "How does StateGraph relate to Pregel?" | GRAPH | MODERATE | Explicit relational question |
| "How to implement CRAG in LangGraph?" | GUIDE | MODERATE | Procedural how-to |
| "Compare all RAG strategies for production deployments considering cost and latency" | ENSEMBLE | COMPLEX | Multi-faceted synthesis |
| "What is the state-of-the-art accuracy on HumanEval with open weights models?" | ENSEMBLE + PwC | EXPERT | Research domain, needs PwCRetriever |

### 6.2 DIRECT Route — Zero-Retrieval Optimization

The DIRECT route is a performance optimization: queries that can be answered from the LLM's parametric knowledge don't benefit from retrieval and only add latency.

Examples of DIRECT-routed queries:
- Simple factual recall the LLM reliably knows ("What is Python?")
- Conversational responses ("Thanks, that helps!")
- Meta-queries about the system ("What can you help me with?")

The DIRECT route at the EnsembleRetriever level is analogous to LangChain's `RunnableBranch` with a passthrough branch.

### 6.3 Complexity Scaling

`QueryComplexity` influences behavior beyond route selection:
- `SIMPLE`: fewer retrievers active, smaller k values (fewer results per retriever)
- `COMPLEX`/`EXPERT`: all retrievers active, larger k values, more CRAG iterations allowed

---

## 7. CRAG Quality Gate and Self-Corrective RAG

### 7.1 CRAG Theoretical Foundation

Corrective RAG (CRAG) was introduced in the paper ["Corrective Retrieval Augmented Generation"](https://arxiv.org/abs/2401.15884) (Yan et al., 2024). The core insight is that standard RAG systems generate responses even when retrieval quality is poor, leading to hallucinations. CRAG adds a lightweight evaluator that decides whether to:

1. **PASS**: proceed with current retrieval results
2. **REWRITE**: reformulate the query for better retrieval
3. **FAIL**: fall back to web search or acknowledge inability to answer

### 7.2 Meta-Builder's CRAG Implementation

Meta-builder's `RetrievalGate` implements a bounded version of CRAG:
- Maximum 2 REWRITE iterations (EnsembleRetriever enforces the cap)
- FAIL is a terminal state that propagates upstream
- No web search fallback (unlike the original CRAG paper) — the system relies only on its indexed knowledge

This is a deliberate choice for generated applications: external web search is a capability that may not always be available or desirable in deployed applications.

### 7.3 Self-Corrective Loop Analysis

```
Query: "What are the performance benchmarks for the tree_indexer at scale?"

Iteration 1:
  Retrieved docs: [general tree indexer docs, unrelated benchmarks, API reference]
  Gate decision: REWRITE (coverage gap — no performance/scale data found)
  Rewritten query: "tree_indexer throughput performance benchmarks scalability"

Iteration 2:
  Retrieved docs: [performance tests, CRAG results with timing data, related work]
  Gate decision: PASS (coverage improved)
  → Proceed to CLaRa enrichment + CitationVerifier
```

### 7.4 CRAG vs. LangGraph Native CRAG

The LangChain blog demonstrates implementing CRAG as a LangGraph StateGraph ([reference](https://blog.langchain.com/agentic-rag-with-langgraph/)). Meta-builder's approach encapsulates the CRAG loop within the retriever abstraction rather than exposing it as a graph node. Both approaches are valid:

| Approach | Advantage | Disadvantage |
|---|---|---|
| LangGraph StateGraph | Visible in graph, debuggable with LangSmith | Exposing retrieval internals to workflow graph |
| Meta-builder (encapsulated) | Clean abstraction, retriever is a black box | Harder to observe CRAG iterations without custom logging |

---

## 8. RRF Fusion Pipeline and Reranking

### 8.1 RRF Mathematical Details

Given \(k\) retrieval lists and a constant \(c = 60\):

\[ \text{RRF}(d) = \sum_{i=1}^{k} \frac{1}{60 + r_i(d)} \]

If a document does not appear in retrieval list \(i\), it does not contribute to the sum (or equivalently, its rank is treated as infinite).

**Example with 3 retrievers:**

| Document | Vector rank | Graph rank | Guide rank | RRF Score |
|---|---|---|---|---|
| Doc A | 1 | 5 | — | 1/61 + 1/65 = 0.0316 |
| Doc B | 3 | 1 | 2 | 1/63 + 1/61 + 1/62 = 0.0484 |
| Doc C | 2 | — | 1 | 1/62 + 1/61 = 0.0325 |

Doc B ranks highest despite not being #1 in any individual retriever — consensus across multiple retrievers is rewarded.

### 8.2 Why top-20 Before Reranking

Cohere's Rerank API has practical constraints:
- Input limit: typically 1000 documents per call
- Output: returns top-N relevance scores
- Latency: increases with input size

Choosing top-20 from RRF fusion before Cohere Rerank balances:
- Sufficient recall (enough candidates for the reranker to find good results)
- Manageable latency (20 documents is fast to rerank)
- Cost efficiency (Cohere charges per token in reranked documents)

### 8.3 Cross-Encoder Reranking vs. Bi-Encoder Embedding

| Dimension | Bi-encoder (embedding) | Cross-encoder (Cohere Rerank) |
|---|---|---|
| Query-doc relationship | Independent encoding | Joint encoding |
| Accuracy | Lower | Higher |
| Speed | Fast (precomputed) | Slower (runtime per pair) |
| Scalability | All documents | Subset only (hence RRF first) |

The RRF→Rerank pipeline exploits both: fast recall via bi-encoders, high-precision ranking via cross-encoders.

---

## 9. Citation Verification Pipeline

### 9.1 Statement Decomposition

The CitationVerifier first decomposes a candidate response into atomic, verifiable statements. For example:

```
Response: "LangGraph's StateGraph supports time-travel debugging via getStateHistory(),
           which returns a list of StateSnapshot objects."

Statements:
  1. "LangGraph's StateGraph supports time-travel debugging"
  2. "Time-travel debugging uses getStateHistory()"
  3. "getStateHistory() returns StateSnapshot objects"
```

### 9.2 Entailment Verification

For each statement, the verifier:
1. Retrieves the most relevant passage from the retrieved documents
2. Runs entailment classification: does passage P entail statement S?
3. Produces `VerifiedStatement` with confidence score

Statements with confidence below threshold are:
- Removed from the response, or
- Flagged with a caveat ("this claim could not be verified"), or
- Marked for human review

### 9.3 Source Attribution

The `source_ids` in `VerifiedStatement` enable precise citation attribution — every claim in the response can point to its specific source document. This is critical for enterprise applications with auditability requirements.

### 9.4 LangChain Equivalents

There is no direct LangChain equivalent for statement-level citation verification. The nearest patterns are:
- RAGAS `faithfulness` metric: evaluates faithfulness post-generation (evaluation, not runtime gate)
- `ContextualCompressionRetriever`: compresses retrieved documents but doesn't verify generated claims
- LangSmith evaluators: post-hoc evaluation, not inline verification

---

## 10. RAG Scaffolding in Generated Projects

### 10.1 How RagTier Integrates with PlanningAgent

When the `PlanningAgent` designs a workflow, it determines the `RagTier` based on:
1. The user's domain description
2. The complexity of the intended use case
3. Available infrastructure constraints

The `WorkflowDesign.generation_tier` and `WorkflowDesign.memory_channels` interact with `RagTier` to define the complete information architecture.

### 10.2 GraphFactory RAG Integration

When `GraphFactory` compiles a `WorkflowDesign` into a LangGraph `StateGraph`, it:
1. Reads `WorkflowDesign.rag_tier` (implicit from configuration)
2. Instantiates the appropriate components for the tier
3. Wires the `EnsembleRetriever` (or simpler retriever for lower tiers) into the state graph as a retriever node
4. Adds retrieval state fields to the `StateGraph`'s state schema

### 10.3 Generated Project RAG Node Pattern

A generated project at `RagTier.STANDARD` would include a retrieval node like:

```python
# Generated by GraphFactory for RagTier.STANDARD
from src.agents.RAG.ensemble_retriever import EnsembleRetriever
from src.models.rag_tiers import RagTier

retriever = EnsembleRetriever(
    rag_tier=RagTier.STANDARD,
    vector_store=vector_store,
    graph_retriever=graph_retriever,
)

def retrieve_node(state: AgentState) -> dict:
    query = state["messages"][-1].content
    results = await retriever.aretrieve(query)
    return {"retrieved_documents": results}
```

### 10.4 DeepAgent RAG Integration

For `generation_tier = deep_agent`, the `DeepAgentGenerator` exports the RAG system as part of the Deep Agent harness. Sub-agents running in Modal/Docker sandboxes receive a configured retriever that points to the shared vector store, enabling retrieval-augmented reasoning even within isolated execution environments.

---

## 11. Comparison with Standard LangChain RAG Patterns

### 11.1 Naive RAG (LangChain baseline)

```python
# Standard LangChain naive RAG
from langchain.chains import RetrievalQA
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

vectorstore = FAISS.from_documents(docs, OpenAIEmbeddings())
qa = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
)
```

**Limitations of naive RAG that meta-builder addresses:**
- Single retriever → lower recall for complex queries
- No quality gate → generates even with poor retrieval
- No reranking → bi-encoder scoring only
- No citation verification → hallucinations possible
- No routing → always retrieves even when unnecessary

### 11.2 Advanced LangChain RAG (community best practice)

```python
# LangChain advanced RAG
from langchain.retrievers import EnsembleRetriever, ContextualCompressionRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_cohere import CohereRerank

bm25 = BM25Retriever.from_documents(docs)
vector = vectorstore.as_retriever(search_kwargs={"k": 10})
ensemble = EnsembleRetriever(retrievers=[bm25, vector], weights=[0.4, 0.6])
reranker = CohereRerank(model="rerank-english-v3.0", top_n=5)
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=ensemble,
)
```

**Still missing vs. meta-builder:**
- Knowledge graph retrieval
- Hierarchical tree navigation
- Semantic query routing
- CRAG self-correction loop
- Citation verification

### 11.3 Feature Comparison Matrix

| Feature | Naive RAG | LangChain Advanced | Meta-Builder BASIC | Meta-Builder MAXIMUM |
|---|---|---|---|---|
| Dense vector search | ✓ | ✓ | ✓ | ✓ |
| Sparse BM25 search | ✗ | ✓ | ✓ | ✓ |
| RRF fusion | ✗ | ✓ (EnsembleRetriever) | ✓ | ✓ |
| Cross-encoder reranking | ✗ | ✓ (optional) | ✗ | ✓ |
| Knowledge graph retrieval | ✗ | ✗ | ✗ | ✓ (STANDARD+) |
| Hierarchical tree retrieval | ✗ | Partial (ParentDoc) | ✗ | ✓ (ADVANCED+) |
| Query routing | ✗ | ✗ | ✓ | ✓ |
| CRAG quality gate | ✗ | ✗ | ✗ | ✓ |
| Citation verification | ✗ | ✗ | ✗ | ✓ |
| Contextual encoding | ✗ | ✗ | ✗ | ✓ (CLaRa) |
| Papers with Code retrieval | ✗ | ✗ | ✗ | ✓ |
| Code-specific retrieval | ✗ | ✗ | ✗ | ✓ |

---

## 12. Data Flow Walkthrough: End-to-End Query

**Query:** "What are the performance characteristics of GraphRetriever for large knowledge graphs?"

**Tier:** MAXIMUM

```
1. EnsembleRetriever receives query

2. SemanticRouter classification:
   → RouteType: ENSEMBLE (complex technical query, multiple aspects)
   → QueryComplexity: COMPLEX
   → Confidence: 0.82

3. Active retrievers: [VectorStore, GraphRetriever, GuideRetriever, CodeRetriever]
   (PwCRetriever not activated — not EXPERT complexity, not explicitly research domain)

4. asyncio.gather() fires all 4 retrievers simultaneously:
   - VectorStore: returns 10 results via LanceDB dense + BM25 sparse fusion
   - GraphRetriever: traverses knowledge graph from "GraphRetriever" node, 2 hops, returns 8 related docs
   - GuideRetriever: navigates tree to "GraphRetriever" section, returns 6 docs with hierarchy context
   - CodeRetriever: finds graph traversal code examples, returns 5 results
   Total: ~29 candidate documents

5. RRF Fusion:
   - Deduplicates (some docs appear in multiple retrievers)
   - Applies RRF formula across 4 rank lists
   - Returns top-20 unified ranked list

6. Cohere Rerank:
   - Sends (query, top-20) to Cohere Rerank API
   - Returns top-10 cross-encoder ranked results

7. RetrievalGate (CRAG):
   - Evaluates: relevance, coverage, coherence, specificity
   - Decision: REWRITE
   - Reason: "Documents cover graph structure but not performance at scale specifically"
   - Rewritten query: "GraphRetriever NetworkX performance scalability large graph memory benchmarks"

8. Iteration 2:
   - asyncio.gather() re-fires with rewritten query
   - RRF Fusion → top-20
   - Cohere Rerank → top-10
   - RetrievalGate: PASS (performance/benchmarking content found)

9. CLaRa enrichment:
   - Enriches top-10 with contextual framing

10. CitationVerifier:
    - Decomposes potential response into verifiable statements
    - Verifies each statement against retrieved documents
    - Returns VerifiedStatement list with source attribution

11. Final output: 10 verified, citation-backed documents ready for generation
    Total latency: ~800-1200ms (parallel retrieval) + ~200ms (Cohere) + ~300ms (gate) = ~1.3-1.7s
```

---

## 13. Dependency and Compatibility Analysis

### 13.1 Core Dependencies by Tier

```
RagTier.NONE:
  (no RAG dependencies)

RagTier.BASIC:
  lancedb>=0.4.0
  rank_bm25>=0.2.0

RagTier.STANDARD:
  + networkx>=3.0

RagTier.ADVANCED / MAXIMUM:
  + langchain_core (already a project dependency)
  + cohere (for Cohere Rerank in MAXIMUM)
```

### 13.2 Version Compatibility Matrix

| Package | Min Version | Why |
|---|---|---|
| `lancedb` | 0.4.0 | Stable async API introduced |
| `rank_bm25` | 0.2.0 | Bug fixes in BM25Okapi |
| `networkx` | 3.0 | Major API stability, performance improvements |
| `cohere` | Current | Rerank v3 models |
| `langchain_core` | 0.1.0+ | BaseRetriever, Document classes |

### 13.3 Async Compatibility

The entire RAG pipeline is designed for async operation:
- `asyncio.gather()` for parallel retrieval
- Cohere Rerank is called with async client
- `EnsembleRetriever.aretrieve()` is the primary interface

This is consistent with LangChain's direction (v0.2+ emphasizes async-first design) and LangGraph's async execution model.

---

*End of File 1: RAG System Analysis*

*See also: [06-component-mapping-matrix.md](./06-component-mapping-matrix.md) for cross-reference to LangChain/LangGraph concepts, and [07-enhancement-opportunities.md](./07-enhancement-opportunities.md) for RAG improvement opportunities.*
