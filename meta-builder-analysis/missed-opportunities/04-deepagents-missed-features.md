# DeepAgents Missed Features

> **Document:** 04 of 4 in the Missed Opportunities series  
> **Scope:** Features and patterns documented in DeepAgents that meta-builder-v3 has not adopted  
> **System version:** meta-builder-v3 (v4.9.0)  
> **Date:** 2026-04-08

---

## Overview

Meta-builder-v3's `DeepAgentGenerator` (generation_tier="deep_agent") produces agents that run on the
DeepAgents SDK harness. The generator delegates graph construction to `GraphFactory`, exports source
files (a `subagents/` directory, `main.py` harness, `agent_config.yaml`, `requirements.txt`), and
returns a `GenerationResult` containing both an in-memory graph (used for structural validation) and
source files for sandbox deployment.

The system's own notes acknowledge a critical architectural fact:

> "For deep_agent tier, `GenerationResult.graph` is a structural validation artifact. The actual
> runtime execution relies on the deepagents SDK harness."

This distinction matters. It means meta-builder produces code *for* the harness, but it does not
take advantage of what that harness actually offers. The generated `main.py` uses a minimal subset
of the DeepAgents API — just enough to wire nodes together and launch. The richer capabilities of
the SDK go unused, both in the generator itself and in the agents it produces.

This document catalogs twelve such gaps in order of operational impact. For each one, it describes:
what DeepAgents provides, what meta-builder currently does, the gap and why it matters, a concrete
bridge strategy, and an impact assessment.

---

## Table of Contents

1. [create_deep_agent() Factory](#1-create_deep_agent-factory)
2. [Async Subagents (Non-blocking)](#2-async-subagents-non-blocking)
3. [Skills System (Progressive Disclosure)](#3-skills-system-progressive-disclosure)
4. [Virtual Filesystem with Pluggable Backends](#4-virtual-filesystem-with-pluggable-backends)
5. [Memory System (Agent-scoped + User-scoped)](#5-memory-system-agent-scoped--user-scoped)
6. [Context Engineering Framework](#6-context-engineering-framework)
7. [Agent Client Protocol (ACP)](#7-agent-client-protocol-acp)
8. [Code Execution in Sandboxes](#8-code-execution-in-sandboxes)
9. [Streaming and Frontend Integration](#9-streaming-and-frontend-integration)
10. [Production Deployment Patterns](#10-production-deployment-patterns)
11. [Dynamic System Prompts (@dynamic_prompt)](#11-dynamic-system-prompts-dynamic_prompt)
12. [Full Harness Capabilities](#12-full-harness-capabilities)

---

## 1. create_deep_agent() Factory

### What DeepAgents Provides

DeepAgents exposes a single, opinionated factory function — `create_deep_agent()` — that assembles
a fully capable agent in one call. It accepts a unified configuration object covering every major
concern:

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    # Model configuration
    model="claude-opus-4-5",
    model_kwargs={"temperature": 0.2},

    # Tool configuration
    tools=[search_tool, write_file_tool, execute_tool],

    # Middleware stack (lifecycle hooks)
    middleware=[
        AuthMiddleware(),
        LoggingMiddleware(log_level="info"),
        RateLimitMiddleware(rpm=60),
    ],

    # Subagents (synchronous and asynchronous)
    subagents=[
        ResearchSubagent,
        CodeReviewSubagent,
    ],
    async_subagents=[
        BackgroundIndexerSubagent,
    ],

    # Memory configuration
    memory=UserScopedMemory(backend="langraph_store"),
    agent_memory=AgentScopedMemory(path="AGENTS.md"),

    # Skills (progressive disclosure)
    skills_path="./skills/",

    # Backend / deployment
    backend="langsmith",
    checkpointer=SqliteSaver(conn),

    # Prompt configuration
    system_prompt=base_system_prompt,
    dynamic_prompts=[permission_prompt, memory_prompt],
)
```

The factory handles:
- Wiring all components together in the correct order
- Injecting middleware into the message lifecycle
- Mounting the virtual filesystem
- Registering skill loaders
- Configuring checkpointing
- Setting up subagent communication channels

### What Meta-builder Currently Does

`DeepAgentGenerator` exports a handwritten `main.py` that manually wires nodes together:

```python
# Generated main.py (simplified)
from deepagents.sdk import DeepAgentHarness

harness = DeepAgentHarness(
    config=load_yaml("agent_config.yaml"),
)

for subagent_file in glob("subagents/*.py"):
    harness.register_subagent(import_module(subagent_file))

harness.run()
```

The `agent_config.yaml` carries model, tool lists, and subagent references. There is no middleware,
no memory backend configuration, no skills path, no checkpointer reference in the generated output.

The generator's `_build_harness_code()` method produces this harness from a string template with
placeholder substitution. It does not call `create_deep_agent()` internally — it generates code
that bypasses the factory entirely.

### The Gap and Why It Matters

The gap is structural: meta-builder treats the harness as a thin runner rather than a rich platform.
By bypassing `create_deep_agent()`:

- **Middleware is silently omitted.** There is no lifecycle management in generated agents. If a
  user requests rate limiting, auth validation, or structured logging, the generator has no path to
  add it.
- **Memory is not wired.** The generated agent has no memory backend. Every conversation starts
  from scratch.
- **Checkpointing is absent.** The generated agent cannot resume from interruption.
- **Skills are not mounted.** Any external knowledge files the user provides are not accessible to
  the agent at runtime.
- **Subagent channels are not initialized.** Async subagents declared in `agent_config.yaml` may
  not receive properly initialized communication primitives.

Each of these is a silent failure: the generated agent runs without error but lacks the capabilities
the user expected.

### How to Bridge the Gap

**Step 1:** Refactor `DeepAgentGenerator._build_harness_code()` to emit `create_deep_agent()` calls.

```python
# In DeepAgentGenerator._build_harness_code()
def _build_harness_code(self, spec: AgentSpec) -> str:
    lines = ["from deepagents import create_deep_agent\n"]
    lines.append("agent = create_deep_agent(")
    lines.append(f"    model={spec.model!r},")
    if spec.tools:
        lines.append(f"    tools=[{', '.join(spec.tools)}],")
    if spec.middleware:
        lines.append(f"    middleware=[{', '.join(spec.middleware)}],")
    if spec.memory_backend:
        lines.append(f"    memory={spec.memory_backend},")
    if spec.checkpointer:
        lines.append(f"    checkpointer={spec.checkpointer},")
    if spec.skills_path:
        lines.append(f"    skills_path={spec.skills_path!r},")
    lines.append(")")
    return "\n".join(lines)
```

**Step 2:** Extend `AgentSpec` (or `AgentConfig`) to carry the additional fields that
`create_deep_agent()` accepts. These fields should be settable through `VisionAgent` during
conversational brainstorming.

**Step 3:** Update `GraphFactory` structural validation to verify that the generated
`create_deep_agent()` call would be valid (all referenced tools are registered, subagents exist,
etc.) without actually executing it.

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | High — generated agents gain memory, checkpointing, middleware |
| Implementation effort | Medium — template refactor, not architectural change |
| Risk | Low — factory is additive, not breaking |
| Dependency | Blocks items 2, 3, 4, 5, 6, 11 (all benefit from factory adoption) |

---

## 2. Async Subagents (Non-blocking)

### What DeepAgents Provides

DeepAgents has a first-class `AsyncSubAgent` primitive for work that should not block the
supervisor's main interaction loop:

```python
from deepagents import AsyncSubAgent, update_async_task, cancel_async_task

class BackgroundResearcher(AsyncSubAgent):
    name = "background_researcher"
    description = "Researches topics in the background while conversation continues"

    async def run(self, task: str, task_id: str) -> AsyncIterator[str]:
        # Long-running work
        results = []
        async for chunk in self.search_and_synthesize(task):
            results.append(chunk)
            # Push mid-task update to supervisor
            await update_async_task(
                task_id=task_id,
                status="in_progress",
                partial_result=chunk,
            )
        yield "\n".join(results)

# Supervisor can cancel if user changes direction
await cancel_async_task(task_id="research-123")
```

Key properties of `AsyncSubAgent`:
- **Non-blocking:** The supervisor continues processing user messages while the subagent runs.
- **Mid-task updates:** `update_async_task` lets the subagent push incremental results.
- **Cancellation:** `cancel_async_task` stops work cleanly if the user changes direction.
- **Stateful:** The subagent maintains its own state across update cycles.
- **Agent Protocol compatible:** Async tasks are expressed as ACP-compliant background jobs.

### What Meta-builder Currently Does

Meta-builder's multi-agent architecture is synchronous at every layer:

- **Conductor** is a LangGraph state machine that blocks at each phase until all subagents in that
  phase complete. The `_run_phase()` method uses `asyncio.gather()` but the gather itself is
  awaited before the Conductor can proceed.
- **Evolution jobs** block on each generation: the `EvolutionRunner` iterates generations
  sequentially, waiting for each `GenerationResult` before moving to the next.
- **Research pipeline** (ScoutMonitor → VerdictPanel → ExperimentRunner) blocks at each step.
  `ScoutMonitor` must complete before `VerdictPanel` runs; `VerdictPanel` must complete before
  `ExperimentRunner` can act.
- **Generated agents** produced by `DeepAgentGenerator` use synchronous subagent calls. The
  generated `subagents/` directory contains agents that return results synchronously; the harness
  waits for each before proceeding.

The pattern throughout is: serialize, wait, serialize, wait.

### The Gap and Why It Matters

Synchronous execution means users wait. For a coding agent asked to:
1. Answer a quick question
2. Meanwhile run a 30-second test suite in the background
3. Report results when ready

...meta-builder's architecture forces the user to wait 30 seconds for step 2 before they get the
answer to step 1. Async subagents invert this: the agent answers immediately, delegates the test
run to an async subagent, and surfaces results when available.

The impact is highest in:
- **Research pipelines** — literature search could run in background while analysis begins
- **Evolution** — multi-population evolution could run generations concurrently
- **Generated coding agents** — test execution, linting, and type-checking could parallelize

### How to Bridge the Gap

**Phase 1 — Internal (meta-builder itself):**

Refactor `EvolutionRunner` to use async generations:

```python
# Current
for gen in range(self.config.max_generations):
    result = await self.run_generation(gen)
    if result.converged:
        break

# Target
tasks = [
    asyncio.create_task(self.run_generation_async(gen))
    for gen in range(self.config.max_generations)
    if not self._would_exceed_budget(gen)
]
async for completed in asyncio.as_completed(tasks):
    result = await completed
    self._process_result(result)
    if self._should_stop():
        await self._cancel_remaining(tasks)
        break
```

**Phase 2 — Generated agents:**

Add `AsyncSubAgent` templates to `DeepAgentGenerator`. When a spec includes background tasks
(e.g., `task_type: "research_with_background_synthesis"`), emit async subagent classes instead of
synchronous ones.

**Phase 3 — Conductor:**

Introduce a `PhaseOrchestrator` that tracks async subagent tasks across phases and delivers
mid-task updates to `ConductorState` without blocking the main loop.

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | High — eliminates blocking waits in interactive use |
| Implementation effort | Medium — significant Conductor refactor required |
| Risk | Medium — async state management introduces new failure modes |
| Quick win | Generated agents can adopt AsyncSubAgent with low risk |

---

## 3. Skills System (Progressive Disclosure)

### What DeepAgents Provides

DeepAgents formalizes "skills" as a first-class primitive: structured markdown files with YAML
frontmatter that an agent can discover and load on demand.

A skill file (`skills/database-analysis.md`) looks like:

```markdown
---
name: database-analysis
description: Analyze database schemas, write optimized SQL, and interpret query plans
version: 1.2.0
tools_required: [execute_sql, read_schema]
tags: [sql, databases, performance]
---

# Database Analysis Skill

## When to use this skill
Use this skill when the user asks about database performance, query optimization,
schema design, or data modeling.

## Core techniques
...500 lines of detailed instructions, templates, and examples...
```

At startup, the agent reads only the frontmatter block (name, description, tags) — not the full
content. When the agent determines a skill is needed, it calls `load_skill(name)` to pull the full
content into context. This is **progressive disclosure**: the agent knows what it *can* do without
paying the context cost of knowing *how* to do everything upfront.

Skills can include:
- Detailed instructions and heuristics
- Code templates and examples
- References to external scripts or documentation files
- Sub-skills (nested progressive disclosure)

### What Meta-builder Currently Does

Meta-builder has no skills system. Agents receive their complete instructions in a static system
prompt at construction time. The `ToolRegistry` manages tools (functions) but has no concept of
"skills" (knowledge modules). There is no mechanism for an agent to discover that a skill exists
without already having its content in context.

The `VisionAgent` uses a skills-adjacent pattern: it has a system prompt that describes available
brainstorming modes. But these modes are always fully loaded — there is no lazy disclosure.

For generated agents, `DeepAgentGenerator` concatenates all relevant instructions into the
`system_prompt` field of `agent_config.yaml`. A coding agent that might need to debug Python,
write SQL, analyze performance profiles, and review infrastructure-as-code receives instructions
for all four domains upfront, consuming context budget even when only one domain is relevant.

### The Gap and Why It Matters

The context cost compounds with agent complexity. A well-rounded generated agent serving a
development team might need skills in:
- Python, TypeScript, Go (3 language skills)
- Postgres, Redis, Kafka (3 database skills)
- GitHub Actions, Terraform, Docker (3 infrastructure skills)
- Incident response, postmortem writing, on-call procedures (3 ops skills)

Loading all 12 skills at startup costs roughly 20,000–40,000 tokens per conversation, leaves less
budget for actual work, and degrades model performance (long contexts dilute attention). With
progressive disclosure, only the 1–2 relevant skills are loaded per task.

Beyond context cost, the static approach means skills cannot be updated without regenerating the
agent. The progressive approach lets skills evolve independently.

### How to Bridge the Gap

**Step 1:** Add a `SkillsRegistry` module to meta-builder:

```python
@dataclass
class Skill:
    name: str
    description: str
    version: str
    tools_required: list[str]
    tags: list[str]
    path: Path  # Full content loaded lazily

class SkillsRegistry:
    def __init__(self, skills_dir: Path):
        self._index = self._build_index(skills_dir)

    def list_skills(self) -> list[SkillSummary]:
        """Return name + description only (no full content)."""
        return [SkillSummary(s.name, s.description) for s in self._index.values()]

    def load_skill(self, name: str) -> str:
        """Return full skill content on demand."""
        return self._index[name].path.read_text()
```

**Step 2:** Update `DeepAgentGenerator` to emit `skills_path` in generated code rather than
inlining instructions into `system_prompt`. Write each instruction block as a separate skill file.

**Step 3:** Add a `load_skill` tool to generated agents so they can self-select skills at runtime.

**Step 4:** Integrate with `create_deep_agent()` (see item 1) via the `skills_path` parameter.

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | High — dramatically reduces context waste in complex agents |
| Implementation effort | Low — file conventions, no architectural change |
| Risk | Low — additive, backward compatible |
| Quick win | Meta-builder's own VisionAgent could adopt skills immediately |

---

## 4. Virtual Filesystem with Pluggable Backends

### What DeepAgents Provides

DeepAgents ships a virtual filesystem that abstracts over multiple storage backends:

```python
from deepagents.fs import VirtualFilesystem, StateBackend, StoreBackend, CompositeBackend

# Route different paths to different backends
fs = VirtualFilesystem(
    backend=CompositeBackend({
        "/tmp/": StateBackend(state_key="scratch"),        # In-graph, ephemeral
        "/memory/": StoreBackend(namespace="agent_files"), # LangGraph Store, persistent
        "/workspace/": StoreBackend(namespace="workspace"), # Namespaced to user
    })
)

# Agents get unified tools regardless of backend
tools = fs.get_tools()
# → ls, read_file, write_file, edit_file, glob, grep, execute
```

Backend options:
- **StateBackend**: files live in LangGraph state (in-memory, scoped to conversation)
- **StoreBackend**: files live in LangGraph Store (persistent, cross-conversation, namespace-scoped)
- **CompositeBackend**: routes paths to different backends by prefix

The filesystem tools expose a POSIX-like interface: path-based addressing, directory listing,
pattern matching, in-place editing. Because the backend is pluggable, the agent's code doesn't
change when the storage tier changes.

### What Meta-builder Currently Does

Meta-builder's generated agents receive basic file I/O tools from `ToolRegistry`:

```python
@tool
def read_file(path: str) -> str:
    """Read a file from disk."""
    return Path(path).read_text()

@tool
def write_file(path: str, content: str) -> None:
    """Write content to a file."""
    Path(path).write_text(content)

@tool
def append_file(path: str, content: str) -> None:
    """Append content to a file."""
    with open(path, "a") as f:
        f.write(content)

@tool
def list_directory(path: str) -> list[str]:
    """List directory contents."""
    return os.listdir(path)
```

These are simple `@tool` functions that wrap `pathlib`/`os` calls against the real filesystem.
There is no backend abstraction, no namespace isolation, no persistence layer, no `grep` or `glob`
tool, and no concept of in-graph vs. cross-conversation storage.

`AgentGym` provides a sandboxed filesystem for testing generated agents, but this is for the
testing environment — not something generated agents carry into production.

### The Gap and Why It Matters

The gap is most visible in three scenarios:

1. **Multi-turn file work:** A generated coding agent writes a file in turn 1, then needs to read
   it in turn 5. With `StateBackend`, the file evaporates at conversation end. With `StoreBackend`,
   it persists. Meta-builder's flat `write_file` has no concept of persistence tier — it writes to
   disk, but the sandbox is ephemeral, so the file disappears anyway.

2. **Multi-user agents:** A generated agent serving multiple users should isolate each user's files.
   The `StoreBackend` with `namespace=user_id` provides this automatically. Meta-builder's
   `write_file` writes to a shared path.

3. **Cross-agent file sharing:** When a generated orchestrator delegates to a subagent, both need
   to read and write shared files. The `CompositeBackend` makes this possible by routing `/shared/`
   to a common namespace. Meta-builder has no equivalent.

Beyond these scenarios, the lack of `grep` and `glob` tools means generated agents cannot
efficiently navigate large codebases — a significant weakness for coding agents.

### How to Bridge the Gap

**Short term:** Add `grep` and `glob` tools to `ToolRegistry`. These are two missing tools that
require no architectural change:

```python
@tool
def glob_files(pattern: str, base_path: str = ".") -> list[str]:
    """Find files matching a glob pattern."""
    return [str(p) for p in Path(base_path).glob(pattern)]

@tool
def grep_files(pattern: str, path: str, context_lines: int = 0) -> str:
    """Search file contents with regex."""
    import subprocess
    args = ["grep", "-rn", f"-A{context_lines}", f"-B{context_lines}", pattern, path]
    result = subprocess.run(args, capture_output=True, text=True)
    return result.stdout
```

**Medium term:** Build a `FilesystemBackend` abstraction in meta-builder and route `ToolRegistry`
file tools through it. Adopt `StateBackend` for ephemeral files and `StoreBackend` for persistent
ones.

**Long term:** Adopt DeepAgents' `VirtualFilesystem` directly and integrate it into
`DeepAgentGenerator`'s harness output. Pass `fs.get_tools()` into `create_deep_agent(tools=...)`.

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | Medium-High — generated coding agents need grep/glob immediately |
| Implementation effort | Low (grep/glob) → High (full pluggable backend) |
| Risk | Low for tool additions; Medium for backend abstraction |
| Quick win | Add grep and glob to ToolRegistry this sprint |

---

## 5. Memory System (Agent-scoped + User-scoped)

### What DeepAgents Provides

DeepAgents distinguishes two orthogonal memory dimensions:

**Agent-scoped memory** — shared across all users, builds the agent's persona over time:

```python
from deepagents.memory import AgentMemory

# AGENTS.md is a special convention file the agent reads and updates
agent_mem = AgentMemory(
    path="AGENTS.md",
    writable=True,
    consolidation_policy="background",  # Update between conversations
)
```

**User-scoped memory** — isolated per user, stores preferences and history:

```python
from deepagents.memory import UserMemory

user_mem = UserMemory(
    backend=StoreBackend(namespace_template="user:{user_id}:memory"),
    read_only=False,
    consolidation_policy="end_of_conversation",
)
```

Additional memory features:
- **AGENTS.md convention:** The agent maintains a structured memory file. It reads this at startup,
  updates it at conversation end. Over time, the agent accumulates domain knowledge.
- **Background consolidation:** Memory consolidation runs as an async process, so it doesn't add
  latency to conversation turns.
- **Read-only vs. writable distinction:** Immutable reference memories (like an org's coding
  standards) are read-only; personal preferences are writable.
- **Organization-level memory:** A shared memory namespace for all agents in an org, enabling
  knowledge propagation across agents.

### What Meta-builder Currently Does

Meta-builder has `PipelineMemory`, which stores the results of past pipeline runs:

```python
@dataclass
class PipelineMemory:
    results: list[PipelineResult]  # Last N pipeline executions
    max_entries: int = 50

    def store(self, result: PipelineResult) -> None:
        self.results.append(result)
        if len(self.results) > self.max_entries:
            self.results.pop(0)

    def get_recent(self, n: int = 5) -> list[PipelineResult]:
        return self.results[-n:]
```

This is a run log, not a memory system. It stores what happened, not what was learned. It has:
- No user scoping
- No agent persona
- No consolidation
- No read/write distinction
- No persistence across process restarts (it's in-memory)
- No organization-level sharing

Generated agents inherit this deficiency. A generated assistant agent serving a team has no way to
remember that User A prefers verbose explanations, User B wants concise bullet points, and the
organization's standard is to use US English.

### The Gap and Why It Matters

Memory is the difference between an agent that improves over time and one that resets every
conversation. Without user-scoped memory:
- Generated agents cannot personalize responses
- Users re-explain preferences on every session
- Agents cannot learn from mistakes

Without agent-scoped memory:
- Generated agents cannot develop domain expertise
- Each conversation starts from the same generic baseline
- The AGENTS.md convention (which informs how agents behave in the repository) is not used

The absence of organization-level memory means that insights discovered by one generated agent
cannot propagate to others. Meta-builder generates many agents; none of them share learned
knowledge.

### How to Bridge the Gap

**Step 1:** Add `AGENTS.md` support to `DeepAgentGenerator`. Generate an `AGENTS.md` template
in the output package and wire it to `agent_memory` in `create_deep_agent()`.

**Step 2:** Adopt `StoreBackend` for user memory. When LangGraph Store is available (it is,
given LangGraph is already a dependency), use it as the memory backend:

```python
from langgraph.store.memory import InMemoryStore
# or
from langgraph.store.postgres import AsyncPostgresStore

store = AsyncPostgresStore(conn_string)
user_mem = UserMemory(
    backend=StoreBackend(store, namespace_template="user:{user_id}:memory")
)
```

**Step 3:** Replace `PipelineMemory` with a structured `PipelineStore` backed by LangGraph Store
(see also LangGraph missed feature: Long-term Memory Store).

**Step 4:** Add a `MemoryConsolidator` that runs at conversation end as an async task (see item 2).

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | High — agents that remember users are qualitatively better |
| Implementation effort | Medium — requires LangGraph Store integration |
| Risk | Low — additive feature, doesn't break existing behavior |
| Dependency | Benefits from items 1 and 2 (factory + async consolidation) |

---

## 6. Context Engineering Framework

### What DeepAgents Provides

DeepAgents formalizes context as a structured engineering concern, not an afterthought:

**Input context layers (assembled before each turn):**

```
┌─────────────────────────────────────────────┐
│ 1. System prompt (static instructions)       │
│ 2. Memory (agent-scoped + user-scoped)       │
│ 3. Skills (loaded on demand)                 │
│ 4. Tool prompts (injected by middleware)     │
│ 5. Conversation history (compressed)         │
└─────────────────────────────────────────────┘
```

**Runtime context** — per-invocation configuration propagated to subagents:

```python
from deepagents import RuntimeContext

ctx = RuntimeContext(
    user_id="user-abc",
    permissions=["read", "write"],
    preferences={"verbosity": "concise", "language": "en-US"},
    budget_tokens=8000,
)

# Context automatically propagates to all subagents
result = await agent.ainvoke(message, context=ctx)
```

**Context compression** — automatic offloading when context approaches limits:

```python
from deepagents.context import ContextCompressor

compressor = ContextCompressor(
    strategy="summarize_oldest",   # or "drop_oldest", "semantic_similarity"
    trigger_at=0.85,               # 85% of context window
    target=0.60,                   # Compress down to 60%
    model="claude-haiku-3-5",      # Cheaper model for summarization
)
```

**Context isolation via subagents** — heavy work is delegated to subagents that return compressed
summaries, keeping the supervisor's context clean.

### What Meta-builder Currently Does

Meta-builder has no context management. The evidence is in `ConductorState`:

```python
@dataclass
class ConductorState:
    messages: list[BaseMessage]  # Grows without bound
    phase_results: dict[str, Any]  # All phase results, no eviction
    tool_calls: list[ToolCall]   # All tool calls, no truncation
    ...
```

There is no eviction policy, no compression, no budget tracking. The state grows with every turn
until it hits the model's context limit, at which point the pipeline fails or silently drops
earlier messages.

The only mitigation is a manual hack in the planning prompt:

```python
# In ConductorAgent._build_planning_prompt()
context_snippet = itertools.islice(
    str(self.state.phase_results).split(), 8000
)
```

This `itertools.islice` truncation is not compression — it's truncation by word count, which can
cut off mid-sentence, drop critical information, and produce malformed context.

There is no runtime context propagation. When the Conductor delegates to a subagent, it passes
explicitly constructed arguments — there is no `RuntimeContext` object that carries user identity,
permissions, or preferences automatically.

### The Gap and Why It Matters

Context engineering failures are silent and hard to debug:
- A pipeline that worked on a 5-step task fails mysteriously on a 20-step task — not because the
  logic is wrong, but because the context window was exceeded and earlier state was dropped.
- Subagents make decisions inconsistent with the supervisor because they received different context
  (the supervisor forgot to pass a preference explicitly).
- The `itertools.islice` truncation drops the most recent results if they happen to be long,
  which is exactly backwards — recent context is usually more relevant.

For generated deep agents, which are intended to handle complex, long-running tasks, this is
especially damaging.

### How to Bridge the Gap

**Short term:** Replace `itertools.islice` with a proper summarization call:

```python
# Replace the hack
async def _compress_planning_context(self, state: ConductorState) -> str:
    if len(str(state.phase_results)) < 6000:
        return str(state.phase_results)
    # Summarize with a fast/cheap model
    summary = await self.summary_llm.ainvoke(
        f"Summarize the following pipeline results concisely:\n{state.phase_results}"
    )
    return summary.content
```

**Medium term:** Introduce `RuntimeContext` as a first-class object in `ConductorState` and
propagate it to all subagent calls.

**Long term:** Adopt DeepAgents' `ContextCompressor` in `create_deep_agent()` configuration. Add
context budget tracking to `ConductorState` and trigger compression before approaching limits.

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | High — directly affects pipeline reliability at scale |
| Implementation effort | Low (truncation fix) → Medium (full framework) |
| Risk | Low — compression is conservative by design |
| Quick win | Fix the `itertools.islice` hack immediately |

---

## 7. Agent Client Protocol (ACP)

### What DeepAgents Provides

DeepAgents ships `deepagents-acp`, a library that wraps any DeepAgents agent as an ACP (Agent
Client Protocol) server. ACP enables agents to integrate with development environments:

```python
from deepagents_acp import ACPServer

server = ACPServer(
    agent=my_agent,
    transport="stdio",   # For editor communication
    name="my-coding-agent",
    capabilities=["code_completion", "refactoring", "test_generation"],
)

server.serve()  # Expose agent over stdio
```

ACP-compatible editors and IDEs include:
- **Zed** — native ACP support for inline agent assistance
- **JetBrains IDEs** — ACP plugin for IntelliJ, PyCharm, etc.
- **VS Code** — ACP extension
- **Neovim** — ACP plugin

The protocol supports:
- Message streaming (real-time typing-style responses)
- Context injection (current file, selection, cursor position)
- Multi-turn sessions with editor context

### What Meta-builder Currently Does

Generated agents have no IDE integration. The `DeepAgentGenerator` produces a `main.py` that runs
as a standalone server but does not expose an ACP interface. Coding agents — arguably the most
common use case for generated deep agents — cannot be used inline in an editor.

There is no mention of ACP in `agent_config.yaml` schema, `DeepAgentGenerator`, or any part of
the generation pipeline.

### The Gap and Why It Matters

A generated coding agent that cannot be used inside an editor has significantly reduced utility.
Developers work in editors, not in separate terminal windows. Without ACP, users must context-
switch to a chat interface, copy-paste code, and manually apply suggestions — breaking their flow.

With ACP, a generated coding agent becomes a first-class editor citizen: it can see the current
file, understand the cursor context, apply refactoring inline, and run tests without the user
leaving the editor.

### How to Bridge the Gap

**Step 1:** Add an `acp_server` option to `agent_config.yaml` schema:

```yaml
# agent_config.yaml
name: my-coding-agent
model: claude-opus-4-5
tools: [read_file, write_file, execute, grep_files]
acp_server:
  enabled: true
  transport: stdio
  capabilities: [code_completion, refactoring, test_generation]
```

**Step 2:** When `acp_server.enabled` is true, `DeepAgentGenerator` emits an additional
`server.py` file alongside `main.py`:

```python
# Generated server.py
from deepagents_acp import ACPServer
from main import agent

server = ACPServer(agent=agent, transport="stdio", name="{{ agent_name }}")
server.serve()
```

**Step 3:** Include ACP setup instructions in the generated `README.md` that every generated agent
package receives.

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | High for developer-focused agents |
| Implementation effort | Medium — ACP wrapper is straightforward; schema changes needed |
| Risk | Low — opt-in via config flag |
| Addressable with | Items 1 and 10 (factory + deployment patterns) |

---

## 8. Code Execution in Sandboxes

### What DeepAgents Provides

DeepAgents gives agents a structured `execute()` tool backed by a sandbox environment:

```python
from deepagents.tools import get_code_execution_tools

# Returns: execute, start_shell, run_in_shell, stop_shell
code_tools = get_code_execution_tools(
    sandbox_backend="modal",   # or "docker", "e2b"
    persistent_shell=True,     # Maintain shell state across tool calls
    timeout=30,
    allowed_languages=["python", "bash", "javascript"],
)
```

With `persistent_shell=True`, the agent can:
1. Start a shell session
2. Run `pip install some-library` (modifies shell state)
3. Run subsequent code that uses the installed library
4. All within a single conversation, with state preserved between calls

The sandbox is isolated: the agent cannot access host filesystem paths outside the workspace, and
execution cannot escape the sandbox boundary.

### What Meta-builder Currently Does

Meta-builder's `AgentGym` has sandboxes (Modal and Docker), but these are used to **test generated
agents**, not to give agents code execution capability:

```
AgentGym
├── SandboxFactory       ← Provisions Modal/Docker sandbox for testing
├── SandboxPool          ← Manages pool of test sandboxes
└── DeepAgentRunner      ← Runs generated agent in sandbox for validation
```

The sandboxes are infrastructure for meta-builder's QA pipeline, not tools available to the agents
themselves. Generated agents produced by `DeepAgentGenerator` do not receive `execute()` tools
unless the user explicitly lists an execute tool in their spec.

Even when a user does request code execution, the `ToolRegistry` provides only a basic shell
wrapper — not a persistent session with `start_shell`/`run_in_shell`/`stop_shell` lifecycle.

### The Gap and Why It Matters

Code execution is a table-stakes capability for development-focused agents. Without it, a generated
coding agent that suggests a fix cannot verify the fix compiles. Without persistent shell sessions,
an agent that needs to install a dependency, run setup, and then execute code must restart the
environment each time, losing state.

The meta-builder infrastructure already has Modal and Docker integration. The sandboxes already
exist. The gap is that the sandbox capability is pointed inward (for testing) rather than outward
(for agent use).

### How to Bridge the Gap

**Leverage existing infrastructure:** The `SandboxFactory` in `AgentGym` already knows how to
provision Modal and Docker sandboxes. Extract this capability into a tool:

```python
# In ToolRegistry
from meta_builder.gym.sandbox import SandboxFactory

def create_code_execution_tools(backend: str = "modal") -> list[BaseTool]:
    factory = SandboxFactory(backend=backend)
    sandbox = factory.provision()
    return [
        ExecuteTool(sandbox),
        StartShellTool(sandbox),
        RunInShellTool(sandbox),
        StopShellTool(sandbox),
    ]
```

Generated agents with `tools: [code_execution]` in their spec would receive these tools via
`create_deep_agent(tools=create_code_execution_tools())`.

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | High — coding agents gain the ability to verify their own output |
| Implementation effort | Medium — existing sandbox infra needs to be exposed as tools |
| Risk | Medium — sandbox isolation must be enforced carefully |
| Infrastructure reuse | High — uses existing Modal/Docker integration |

---

## 9. Streaming and Frontend Integration

### What DeepAgents Provides

DeepAgents has frontend integration patterns as part of the SDK:

**Sandbox UI:** A pre-built frontend component that renders:
- File tree (backed by VirtualFilesystem)
- Terminal (for execute tool output)
- Preview pane (for HTML/rendered output)

**Subagent streaming:** Real-time updates from background async tasks stream to the frontend
without polling.

**Todo list rendering:** The `write_todos` tool produces structured output that renders as a
progress tracker in the frontend.

**Streaming patterns:**

```python
# Agents stream tokens and structured events
async for event in agent.astream_events(message):
    if event["event"] == "on_chat_model_stream":
        yield event["data"]["chunk"].content
    elif event["event"] == "on_tool_start":
        yield render_tool_call(event["data"])
    elif event["event"] == "on_custom_event" and event["name"] == "todo_update":
        yield render_todo_list(event["data"])
```

### What Meta-builder Currently Does

Meta-builder generates backend-only agents. The `DeepAgentGenerator` produces:
- `main.py` — agent harness
- `subagents/*.py` — subagent modules
- `agent_config.yaml` — configuration
- `requirements.txt` — dependencies

There is no `frontend/` directory, no streaming configuration, no todo rendering setup. The
`AgentGym` has a `DeepAgentRunner` that runs agents in a sandbox for testing, but this is a
headless test runner, not a frontend.

If a user wants a UI for their generated agent, they must build it themselves from scratch.

### The Gap and Why It Matters

The UX of a generated agent depends entirely on how it's surfaced. A backend-only agent is
invisible to end users unless someone builds a frontend. By not scaffolding frontend patterns,
meta-builder leaves agents that technically work but are practically unusable without additional
development.

Blocking I/O compounds this: because the agents are synchronous (see item 2), even a simple chat
frontend would appear to freeze during long operations. Streaming + async execution together would
make generated agents feel responsive.

### How to Bridge the Gap

**Step 1:** Add a `frontend/` directory option to `DeepAgentGenerator`. When
`spec.include_frontend=True`, generate a minimal frontend scaffold:

```
generated-agent/
├── main.py
├── subagents/
├── agent_config.yaml
├── requirements.txt
└── frontend/           ← New
    ├── index.html
    ├── stream.js       ← SSE stream handler
    └── components/
        ├── chat.js
        ├── file-tree.js
        └── todo-list.js
```

**Step 2:** Configure `main.py` to emit streaming events in the SSE format the frontend expects.

**Step 3:** Integrate todo list rendering by ensuring `write_todos` is in the default tool set
(see item 12 on full harness capabilities).

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | Medium — agents become deployable products, not just backends |
| Implementation effort | High — frontend scaffolding requires design work |
| Risk | Low — opt-in via spec flag |
| Dependency | Strongly benefits from item 2 (async) and item 12 (full harness) |

---

## 10. Production Deployment Patterns

### What DeepAgents Provides

DeepAgents documents production deployment patterns for agents:

**LangSmith deployment with automatic scaling:**

```python
from langsmith import Client

client = Client()
deployment = client.create_deployment(
    agent=my_agent,
    name="production-coding-agent",
    scaling={"min_replicas": 1, "max_replicas": 10},
    health_check="/health",
)
```

**Agent Protocol compatibility:** Agents expose a standard REST API:
- `POST /runs` — start a new agent run
- `GET /runs/{run_id}` — check run status
- `GET /runs/{run_id}/stream` — stream run output
- `DELETE /runs/{run_id}` — cancel a run

**Background task management:** Long-running tasks are managed as durable background jobs with
status tracking, progress reporting, and cancellation.

**Health checks and monitoring:**
```python
# Agents expose health endpoints automatically
GET /health → {"status": "ok", "version": "1.2.0", "uptime": 3600}
GET /metrics → Prometheus-format metrics
```

### What Meta-builder Currently Does

Meta-builder's deployment path is `langgraph.json`:

```json
{
  "graphs": {
    "conductor": "meta_builder.conductor:graph",
    "vision_agent": "meta_builder.vision_agent:graph"
  }
}
```

This is the deployment descriptor for meta-builder itself, not for generated agents. Generated
agents have no deployment configuration beyond `requirements.txt` and a bare `main.py`.

Generated agents do not include:
- `langgraph.json` (for LangGraph Cloud deployment)
- Docker configuration
- Health check endpoints
- Metrics endpoints
- Scaling configuration
- Agent Protocol REST API wiring

### How to Bridge the Gap

**Step 1:** Add a `deployment/` directory to the generated package:

```
generated-agent/
├── main.py
├── subagents/
├── agent_config.yaml
├── requirements.txt
└── deployment/           ← New
    ├── langgraph.json
    ├── Dockerfile
    ├── docker-compose.yml
    └── README.md
```

**Step 2:** The `langgraph.json` should reference the main agent graph:

```json
{
  "graphs": {
    "agent": "./main.py:agent"
  },
  "env": ".env",
  "python_version": "3.12"
}
```

**Step 3:** Add health check wiring to the generated `main.py` when
`spec.deployment_target="langsmith"`.

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | High — generated agents become deployable without additional work |
| Implementation effort | Low — mostly template additions |
| Risk | Low — additive files, no behavioral change |
| Quick win | Add langgraph.json and Dockerfile templates now |

---

## 11. Dynamic System Prompts (@dynamic_prompt)

### What DeepAgents Provides

DeepAgents supports middleware-based dynamic prompts that change system prompt content at runtime:

```python
from deepagents.middleware import dynamic_prompt

@dynamic_prompt
async def permission_prompt(context: RuntimeContext) -> str:
    """Inject permission-aware instructions."""
    if "admin" in context.permissions:
        return "You have admin access. You may modify any file."
    elif "read_only" in context.permissions:
        return "You have read-only access. Do not attempt to modify files."
    return ""

@dynamic_prompt
async def memory_prompt(context: RuntimeContext) -> str:
    """Inject personalization from user memory."""
    prefs = await context.user_memory.load()
    if prefs.verbosity == "concise":
        return "The user prefers concise responses. Use bullet points."
    return ""

agent = create_deep_agent(
    system_prompt="You are a helpful coding assistant.",
    dynamic_prompts=[permission_prompt, memory_prompt],
    ...
)
```

At each invocation, the dynamic prompts are evaluated with the current `RuntimeContext`. The
results are appended to the system prompt. The base prompt is static; the dynamic sections adapt.

### What Meta-builder Currently Does

All agents in meta-builder use static system prompts constructed at initialization time. The
`ConductorAgent`, `VisionAgent`, and all generated agents have prompts that are fixed strings (or
f-strings evaluated once at construction).

There is no mechanism to adapt the system prompt based on:
- User permissions
- User preferences from memory
- Current context (e.g., which files are open, what phase the pipeline is in)
- Organization policy

The Conductor's planning prompt does inject `phase_results` at call time, but this is passed as a
user message, not as a system prompt injection — a significant difference in how models weight it.

### How to Bridge the Gap

**Short term:** Introduce a `PromptBuilder` that can compose prompts from static + dynamic parts:

```python
class PromptBuilder:
    def __init__(self, base: str):
        self._base = base
        self._dynamic: list[Callable] = []

    def add_dynamic(self, fn: Callable[[RuntimeContext], str]) -> "PromptBuilder":
        self._dynamic.append(fn)
        return self

    async def build(self, context: RuntimeContext) -> str:
        parts = [self._base]
        for fn in self._dynamic:
            part = await fn(context)
            if part:
                parts.append(part)
        return "\n\n".join(parts)
```

**Long term:** Adopt `@dynamic_prompt` decorator from DeepAgents middleware and integrate with
`create_deep_agent(dynamic_prompts=...)` (see item 1).

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | Medium — enables personalization and permission-aware behavior |
| Implementation effort | Low (PromptBuilder) → Medium (full middleware integration) |
| Risk | Low — purely additive |
| Dependency | Requires item 5 (memory) and item 1 (factory) for full effect |

---

## 12. Full Harness Capabilities

### What DeepAgents Provides

The DeepAgents harness is an integrated bundle of capabilities that every agent gets by default.
These are not optional extras — they are the standard capabilities of any `create_deep_agent()`
call:

**Planning (write_todos):**

```python
# Built-in tool, always available
await agent.invoke("Build a REST API")
# → Agent automatically creates and maintains a todo list
# → Renders in frontend as a progress tracker
# → Persists in checkpointer state
```

**Virtual filesystem** (covered in item 4)

**Task delegation (ephemeral subagents):**

```python
# The harness provides a delegate() mechanism for one-off subagent tasks
# The supervisor can spin up an ephemeral subagent, get results, and discard it
# No need to pre-declare all subagents at construction time
result = await agent.delegate(
    task="Review this PR for security issues",
    context=pr_diff,
    timeout=60,
)
```

**Context management** (covered in item 6)

**Code execution** (covered in item 8)

**Human-in-the-loop:**

```python
# Harness supports interrupt → human review → resume
# Compatible with LangGraph's interrupt() primitive
result = await agent.invoke(
    "Deploy to production",
    interrupt_before=["deploy_to_production"],
)
# → Agent pauses at deploy step
# → Human reviews and approves
# → Agent resumes
```

### What Meta-builder Currently Does

Meta-builder's generated `main.py` harness includes:
- Model initialization
- Tool registration (from `agent_config.yaml`)
- Subagent registration (from `subagents/` directory)
- Basic conversation loop

It does **not** include:
- `write_todos` tool (or any planning primitive)
- Virtual filesystem
- Ephemeral subagent delegation
- Context management / compression
- Code execution (unless explicitly specified)
- Human-in-the-loop via `interrupt()`

The generated harness is a thin runner, not a capability platform. Each of the missing capabilities
represents a separate feature that users must implement themselves.

### The Gap and Why It Matters

The harness is the most visible artifact that `DeepAgentGenerator` produces. Users evaluate the
quality of meta-builder by the quality of the agents it generates. A generated agent that lacks
planning, memory, code execution, and human-in-the-loop is a prototype, not a production tool.

The cumulative effect of all twelve missed features compounds here: items 1 through 11 are all
things that a proper harness should include. The harness is where they converge.

### How to Bridge the Gap

This item is the integration target for all preceding items. The bridge strategy is:

1. **Adopt `create_deep_agent()`** (item 1) — factory handles the wiring
2. **Add `write_todos` to default tool set** — one-line addition to tool list
3. **Wire VirtualFilesystem** (item 4) — pass `fs.get_tools()` to factory
4. **Add ephemeral delegation** — add a `delegate()` helper to the generated harness
5. **Wire ContextCompressor** (item 6) — add to `create_deep_agent()` call
6. **Wire code execution tools** (item 8) — add to tool list when spec requests it
7. **Add `interrupt_before` support** (via LangGraph missed features) — wire to checkpointer

The generated harness should grow from ~50 lines to ~150 lines, and those 100 additional lines
represent the difference between a prototype and a production-grade agent.

### Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| User-facing impact | Critical — the harness is the product |
| Implementation effort | High — integration point for all preceding items |
| Risk | Medium — changes the shape of generated output; existing generated agents need migration |
| Priority | This is the capstone; implement after items 1, 4, 6, 8 |

---

## Summary: Gap Analysis at a Glance

| # | Feature | Current State | Gap Severity | Effort | Priority |
|---|---------|--------------|--------------|--------|---------|
| 1 | create_deep_agent() factory | Custom template harness | High | Medium | P1 |
| 2 | Async subagents | All synchronous | High | Medium | P1 |
| 3 | Skills system | Static system prompts | High | Low | P1 |
| 4 | Virtual filesystem | Basic @tool functions | Medium | Low→High | P2 |
| 5 | Memory system | Run log only | High | Medium | P1 |
| 6 | Context engineering | No management, hack truncation | High | Low→Med | P0 |
| 7 | ACP integration | None | Low | Medium | P3 |
| 8 | Code execution in sandbox | Test-only sandboxes | Medium | Medium | P2 |
| 9 | Streaming + frontend | Backend only | Medium | High | P2 |
| 10 | Production deployment | No deployment artifacts | Medium | Low | P1 |
| 11 | Dynamic system prompts | Static only | Medium | Low | P2 |
| 12 | Full harness capabilities | Thin runner | Critical | High | P0 |

---

## Recommended Implementation Order

Given the dependency graph among these items:

### Wave 1 — Foundation (Items 1, 3, 6)
These items unblock the most other items and have the best effort-to-impact ratio:
- **Item 6 (Context engineering):** Fix the `itertools.islice` hack immediately. Add
  `RuntimeContext` propagation.
- **Item 3 (Skills system):** Build `SkillsRegistry` and convert existing static prompts.
- **Item 1 (create_deep_agent factory):** Refactor harness code generation to use the factory.

### Wave 2 — Capabilities (Items 4, 5, 8, 10)
With the factory in place, add capabilities:
- **Item 10 (Deployment artifacts):** Low effort, high utility — add templates.
- **Item 4 (Virtual filesystem):** Add grep/glob first; full backend abstraction later.
- **Item 8 (Code execution):** Expose existing sandbox infra as agent tools.
- **Item 5 (Memory system):** Wire LangGraph Store as memory backend.

### Wave 3 — Advanced (Items 2, 9, 11, 12)
- **Item 2 (Async subagents):** Refactor Conductor and generated subagents.
- **Item 11 (Dynamic prompts):** Build PromptBuilder, integrate with factory.
- **Item 12 (Full harness):** Integration capstone — harness all preceding items.
- **Item 9 (Streaming + frontend):** Add frontend scaffolding to generator.

### Wave 4 — Ecosystem (Item 7)
- **Item 7 (ACP):** Add after Wave 3; requires stable harness.

---

## Relationship to Other Documents

This document is part four of the Missed Opportunities series:

- **01-langchain-missed-features.md** — LangChain features not adopted (middleware, MCP, context
  engineering, memory, evals)
- **02-langchain-integration-gaps.md** — Systematic integration gaps (callbacks, serialization,
  document loaders, text splitters)
- **03-langgraph-missed-features.md** — LangGraph features not adopted (checkpointing, durability,
  interrupts, time travel, streaming, functional API)
- **04-deepagents-missed-features.md** (this document) — DeepAgents features not adopted

Items 6 (Context engineering) and 12 (Full harness) in this document overlap with LangGraph's
checkpointing and interrupt gaps described in document 03. Those LangGraph primitives are the
underlying mechanism; DeepAgents' harness is the pattern that integrates them. Both documents
should be consulted when planning the harness refactor.

Items 1 and 5 (factory and memory) overlap with LangGraph's Long-term Memory Store gap in
document 03. LangGraph Store is the storage backend; `create_deep_agent()` is the integration
point.

---

*End of document 04: DeepAgents Missed Features*
