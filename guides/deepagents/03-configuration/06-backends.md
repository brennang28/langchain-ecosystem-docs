> ## Documentation Index
> Fetch the complete documentation index at: https://docs.langchain.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Backends

> Choose and configure filesystem backends for Deep Agents. You can specify routes to different backends, implement virtual filesystems, and enforce policies.

Deep Agents expose a filesystem surface to the agent via tools like `ls`, `read_file`, `write_file`, `edit_file`, `glob`, and `grep`. These tools operate through a pluggable backend. The `read_file` tool natively supports image files (`.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`) across all backends, returning them as multimodal content blocks.

Sandboxes and the [`LocalShellBackend`](https://reference.langchain.com/python/deepagents/backends/local_shell/LocalShellBackend) also provide an `execute` tool.

This page explains how to [choose a backend](#specify-a-backend), [route different paths to different backends](#route-to-different-backends), [implement your own virtual filesystem](#use-a-virtual-filesystem) (e.g., S3 or Postgres), [add policy hooks](#add-policy-hooks), [work with binary and multimodal files](#multimodal-and-binary-files), [comply with the backend protocol](#protocol-reference), and [update existing backends to v2](#update-existing-backends-to-v2).

## Quickstart

Here are a few prebuilt filesystem backends that you can quickly use with your deep agent:

| Built-in backend                                                 | Description                                                                                                                                                                                                                                                                                       |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Default](#statebackend-ephemeral)                               | `agent = create_deep_agent()` <br /> Ephemeral in state. The default filesystem backend for an agent is stored in `langgraph` state. Note that this filesystem only persists *for a single thread*.                                                                                               |
| [Local filesystem persistence](#filesystembackend-local-disk)    | `agent = create_deep_agent(backend=FilesystemBackend(root_dir="/Users/nh/Desktop/"))` <br />This gives the deep agent access to your local machine's filesystem. You can specify the root directory that the agent has access to. Note that any provided `root_dir` must be an absolute path.     |
| [Durable store (LangGraph store)](#storebackend-langgraph-store) | `agent = create_deep_agent(backend=StoreBackend())` <br />This gives the agent access to long-term storage that is *persisted across threads*. This is great for storing longer term memories or instructions that are applicable to the agent over multiple executions.                          |
| [Sandbox](/oss/python/deepagents/sandboxes)                      | `agent = create_deep_agent(backend=sandbox)` <br />Execute code in isolated environments. Sandboxes provide filesystem tools plus the `execute` tool for running shell commands. Choose from Modal, Daytona, Deno, or local VFS.                                                                  |
| [Local shell](#localshellbackend-local-shell)                    | `agent = create_deep_agent(backend=LocalShellBackend(root_dir=".", env={"PATH": "/usr/bin:/bin"}))` <br />Filesystem and shell execution directly on the host. No isolation—use only in controlled development environments. See [security considerations](#localshellbackend-local-shell) below. |
| [Composite](#compositebackend-router)                            | Ephemeral by default, `/memories/` persisted. The Composite backend is maximally flexible. You can specify different routes in the filesystem to point towards different backends. See Composite routing below for a ready-to-paste example.                                                      |

```mermaid  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
graph TB
    Tools[Filesystem Tools] --> Backend[Backend]

    Backend --> State[State]
    Backend --> Disk[Filesystem]
    Backend --> Store[Store]
    Backend --> Sandbox[Sandbox]
    Backend --> LocalShell[Local Shell]
    Backend --> Composite[Composite]
    Backend --> Custom[Custom]

    Composite --> Router{Routes}
    Router --> State
    Router --> Disk
    Router --> Store

    Sandbox --> Execute["#43; execute tool"]
    LocalShell --> Execute["#43; execute tool"]

    classDef trigger fill:#DCFCE7,stroke:#16A34A,stroke-width:2px,color:#14532D
    classDef process fill:#DBEAFE,stroke:#2563EB,stroke-width:2px,color:#1E3A8A
    classDef decision fill:#FEF3C7,stroke:#F59E0B,stroke-width:2px,color:#78350F
    classDef output fill:#F3E8FF,stroke:#9333EA,stroke-width:2px,color:#581C87

    class Tools trigger
    class Backend,State,Disk,Store,Sandbox,LocalShell,Composite,Custom process
    class Router decision
    class Execute output
```

## Built-in backends

### StateBackend (ephemeral)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
# By default we provide a StateBackend
agent = create_deep_agent()

# Under the hood, it looks like
from deepagents.backends import StateBackend

agent = create_deep_agent(
    backend=StateBackend()
)
```

**How it works:**

* Stores files in LangGraph agent state for the current thread via [`StateBackend`](https://reference.langchain.com/python/deepagents/backends/state/StateBackend).
* Persists across multiple agent turns on the same thread via checkpoints.

**Best for:**

* A scratch pad for the agent to write intermediate results.
* Automatic eviction of large tool outputs which the agent can then read back in piece by piece.

Note that this backend is shared between the supervisor agent and subagents, and any files a subagent writes will remain in the LangGraph agent state
even after that subagent's execution is complete. Those files will continue to be available to the supervisor agent and other subagents.

### FilesystemBackend (local disk)

[`FilesystemBackend`](https://reference.langchain.com/python/deepagents/backends/filesystem/FilesystemBackend) reads and writes real files under a configurable root directory.

<Warning>
  This backend grants agents direct filesystem read/write access.
  Use with caution and only in appropriate environments.

  **Appropriate use cases:**

  * Local development CLIs (coding assistants, development tools)
  * CI/CD pipelines (see security considerations below)

  **Inappropriate use cases:**

  * Web servers or HTTP APIs - use `StateBackend`, `StoreBackend`, or a [sandbox backend](/oss/python/deepagents/sandboxes) instead

  **Security risks:**

  * Agents can read any accessible file, including secrets (API keys, credentials, `.env` files)
  * Combined with network tools, secrets may be exfiltrated via SSRF attacks
  * File modifications are permanent and irreversible

  **Recommended safeguards:**

  1. Enable [Human-in-the-Loop (HITL) middleware](/oss/python/deepagents/human-in-the-loop) to review sensitive operations.
  2. Exclude secrets from accessible filesystem paths (especially in CI/CD).
  3. Use a [sandbox backend](/oss/python/deepagents/sandboxes) for production environments requiring filesystem interaction.
  4. **Always** use `virtual_mode=True` with `root_dir` to enable path-based access restrictions (blocks `..`, `~`, and absolute paths outside root).
     Note that the default (`virtual_mode=False`) provides no security even with `root_dir` set.
</Warning>

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from deepagents.backends import FilesystemBackend

agent = create_deep_agent(
    backend=FilesystemBackend(root_dir=".", virtual_mode=True)
)
```

**How it works:**

* Reads/writes real files under a configurable `root_dir`.
* You can optionally set `virtual_mode=True` to sandbox and normalize paths under `root_dir`.
* Uses secure path resolution, prevents unsafe symlink traversal when possible, can use ripgrep for fast `grep`.

**Best for:**

* Local projects on your machine
* CI sandboxes
* Mounted persistent volumes

### LocalShellBackend (local shell)

<Warning>
  This backend grants agents direct filesystem read/write access **and** unrestricted shell execution on your host.
  Use with extreme caution and only in appropriate environments.

  **Appropriate use cases:**

  * Local development CLIs (coding assistants, development tools)
  * Personal development environments where you trust the agent's code
  * CI/CD pipelines with proper secret management

  **Inappropriate use cases:**

  * Production environments (such as web servers, APIs, multi-tenant systems)
  * Processing untrusted user input or executing untrusted code

  **Security risks:**

  * Agents can execute **arbitrary shell commands** with your user's permissions
  * Agents can read any accessible file, including secrets (API keys, credentials, `.env` files)
  * Secrets may be exposed
  * File modifications and command execution are **permanent and irreversible**
  * Commands run directly on your host system
  * Commands can consume unlimited CPU, memory, disk

  **Recommended safeguards:**

  1. Enable [Human-in-the-Loop (HITL) middleware](/oss/python/deepagents/human-in-the-loop) to review and approve operations before execution. This is **strongly recommended**.
  2. Run in dedicated development environments only. Never use on shared or production systems.
  3. Use a [sandbox backend](/oss/python/deepagents/sandboxes) for production environments requiring shell execution.

  **Note:** `virtual_mode=True` provides no security with shell access enabled, since commands can access any path on the system.
</Warning>

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from deepagents.backends import LocalShellBackend

agent = create_deep_agent(
    backend=LocalShellBackend(root_dir=".", env={"PATH": "/usr/bin:/bin"})
)
```

**How it works:**

* Extends `FilesystemBackend` with the `execute` tool for running shell commands on the host.
* Commands run directly on your machine using `subprocess.run(shell=True)` with no sandboxing.
* Supports `timeout` (default 120s), `max_output_bytes` (default 100,000), `env`, and `inherit_env` for environment variables.
* Shell commands use `root_dir` as the working directory but can access any path on the system.

**Best for:**

* Local coding assistants and development tools
* Quick iteration during development when you trust the agent

### StoreBackend (LangGraph store)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from langgraph.store.memory import InMemoryStore
from deepagents.backends import StoreBackend

agent = create_deep_agent(
    backend=StoreBackend(
        namespace=lambda ctx: (ctx.runtime.context.user_id,),
    ),
    store=InMemoryStore()  # Good for local dev; omit for LangSmith Deployment
)
```

<Note>
  When deploying to [LangSmith Deployment](/langsmith/deployment), omit the `store` parameter. The platform automatically provisions a store for your agent.
</Note>

<Tip>
  The `namespace` parameter controls data isolation. For multi-user deployments, always set a [namespace factory](/oss/python/deepagents/backends#namespace-factories) to isolate data per user or tenant.
</Tip>

**How it works:**

* [`StoreBackend`](https://reference.langchain.com/python/deepagents/backends/store/StoreBackend) stores files in a LangGraph [`BaseStore`](https://reference.langchain.com/python/langchain-core/stores/BaseStore) provided by the runtime, enabling cross‑thread durable storage.

**Best for:**

* When you already run with a configured LangGraph store (for example, Redis, Postgres, or cloud implementations behind [`BaseStore`](https://reference.langchain.com/python/langchain-core/stores/BaseStore)).
* When you're deploying your agent through [LangSmith Deployment](/langsmith/deployment) (a store is automatically provisioned for your agent).

#### Namespace factories

A namespace factory controls where `StoreBackend` reads and writes data. It receives a `BackendContext` and returns a tuple of strings used as the store namespace. Use namespace factories to isolate data between users, tenants, or assistants.

Pass the namespace factory to the `namespace` parameter when constructing a `StoreBackend`:

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
NamespaceFactory = Callable[[BackendContext], tuple[str, ...]]
```

The `BackendContext` provides:

* `ctx.runtime.context` — User-supplied context passed via LangGraph's [context schema](https://langchain-ai.github.io/langgraph/concepts/runtime/) (for example, `user_id`)

* `ctx.runtime.server_info` — Server-specific metadata when running on LangGraph Server (assistant ID, graph ID, authenticated user)

* `ctx.runtime.execution_info` — Execution identity information (thread ID, run ID, checkpoint ID)

* `ctx.state` — Current agent state

<Note>
  `ctx.runtime.server_info` and `ctx.runtime.execution_info` require `deepagents>=0.5.0`.
</Note>

**Common namespace patterns:**

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from deepagents.backends import StoreBackend

# Per-user: each user gets their own isolated storage
backend = StoreBackend(
    namespace=lambda ctx: (ctx.runtime.server_info.user.identity,),  # [!code highlight]
)

# Per-assistant: all users of the same assistant share storage
backend = StoreBackend(
    namespace=lambda ctx: (
        ctx.runtime.server_info.assistant_id,  # [!code highlight]
    ),
)

# Per-thread: storage scoped to a single conversation
backend = StoreBackend(
    namespace=lambda ctx: (
        ctx.runtime.execution_info.thread_id,  # [!code highlight]
    ),
)
```

You can combine multiple components to create more specific scopes — for example, `(user_id, thread_id)` for per-user per-conversation isolation, or append a suffix like `"filesystem"` to disambiguate when the same scope uses multiple store namespaces.

Namespace components must contain only alphanumeric characters, hyphens, underscores, dots, `@`, `+`, colons, and tildes. Wildcards (`*`, `?`) are rejected to prevent glob injection.

<Warning>
  The `namespace` parameter will be **required** in v0.5.0. Always set it explicitly for new code.
</Warning>

<Note>
  When no namespace factory is provided, the legacy default uses the `assistant_id` from LangGraph config metadata. This means all users of the same [assistant](/langsmith/assistants) share the same storage. For multi-user [going to production](/oss/python/deepagents/going-to-production), always provide a namespace factory.
</Note>

### CompositeBackend (router)

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore

agent = create_deep_agent(
    backend=CompositeBackend(
        default=StateBackend(),
        routes={
            "/memories/": StoreBackend(),
        }
    ),
    store=InMemoryStore()  # Store passed to create_deep_agent, not backend
)
```

**How it works:**

* [`CompositeBackend`](https://reference.langchain.com/python/deepagents/backends/composite/CompositeBackend) routes file operations to different backends based on path prefix.
* Preserves the original path prefixes in listings and search results.

**Best for:**

* When you want to give your agent both ephemeral and cross-thread storage, a `CompositeBackend` allows you provide both a `StateBackend` and `StoreBackend`
* When you have multiple sources of information that you want to provide to your agent as part of a single filesystem.
  * e.g. You have long-term memories stored under `/memories/` in one Store and you also have a custom backend that has documentation accessible at /docs/.

## Specify a backend

* Pass a backend instance to `create_deep_agent(backend=...)`. The filesystem middleware uses it for all tooling.
* The backend must implement `BackendProtocol` (for example, `StateBackend()`, `FilesystemBackend(root_dir=".")`, `StoreBackend()`).
* If omitted, the default is `StateBackend()`.

## Route to different backends

Route parts of the namespace to different backends. Commonly used to persist `/memories/*` and keep everything else ephemeral.

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, FilesystemBackend

agent = create_deep_agent(
    backend=CompositeBackend(
        default=StateBackend(),
        routes={
            "/memories/": FilesystemBackend(root_dir="/deepagents/myagent", virtual_mode=True),
        },
    )
)
```

Behavior:

* `/workspace/plan.md` → `StateBackend` (ephemeral)
* `/memories/agent.md` → `FilesystemBackend` under `/deepagents/myagent`
* `ls`, `glob`, `grep` aggregate results and show original path prefixes.

Notes:

* Longer prefixes win (for example, route `"/memories/projects/"` can override `"/memories/"`).
* For StoreBackend routing, ensure a store is provided via `create_deep_agent(store=...)` or provisioned by the platform.

## Use a virtual filesystem

Build a custom backend to project a remote or database filesystem (e.g., S3 or Postgres) into the tools namespace.

Design guidelines:

* Paths are absolute (`/x/y.txt`). Decide how to map them to your storage keys/rows.

* Implement `ls` and `glob` efficiently (server-side filtering where available, otherwise local filter).

* For external persistence (S3, Postgres, etc.), return `files_update=None` (Python) or omit `filesUpdate` (JS) in write/edit results — only in-memory state backends need to return a files update dict.

* Use `ls` and `glob` as the method names.

* Return structured result types with an `error` field for missing files or invalid patterns (do not raise).

S3-style outline:

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from deepagents.backends.protocol import (
    BackendProtocol, WriteResult, EditResult, LsResult, ReadResult, GrepResult, GlobResult,
)

class S3Backend(BackendProtocol):
    def __init__(self, bucket: str, prefix: str = ""):
        self.bucket = bucket
        self.prefix = prefix.rstrip("/")

    def _key(self, path: str) -> str:
        return f"{self.prefix}{path}"

    def ls(self, path: str) -> LsResult:
        # List objects under _key(path); build FileInfo entries (path, size, modified_at)
        ...

    def read(self, file_path: str, offset: int = 0, limit: int = 2000) -> ReadResult:
        # Fetch object; return ReadResult(file_data=...) or ReadResult(error=...)
        ...

    def grep(self, pattern: str, path: str | None = None, glob: str | None = None) -> GrepResult:
        # Optionally filter server‑side; else list and scan content
        ...

    def glob(self, pattern: str, path: str = "/") -> GlobResult:
        # Apply glob relative to path across keys
        ...

    def write(self, file_path: str, content: str) -> WriteResult:
        # Enforce create‑only semantics; return WriteResult(path=file_path, files_update=None)
        ...

    def edit(self, file_path: str, old_string: str, new_string: str, replace_all: bool = False) -> EditResult:
        # Read → replace (respect uniqueness vs replace_all) → write → return occurrences
        ...
```

Postgres-style outline:

* Table `files(path text primary key, content text, created_at timestamptz, modified_at timestamptz)`
* Map tool operations onto SQL:
  * `ls` uses `WHERE path LIKE $1 || '%'`
  * `glob` filter in SQL or fetch then apply glob in Python
  * `grep` can fetch candidate rows by extension or last modified time, then scan lines

## Add policy hooks

Enforce enterprise rules by subclassing or wrapping a backend.

Block writes/edits under selected prefixes (subclass):

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from deepagents.backends.filesystem import FilesystemBackend
from deepagents.backends.protocol import WriteResult, EditResult

class GuardedBackend(FilesystemBackend):
    def __init__(self, *, deny_prefixes: list[str], **kwargs):
        super().__init__(**kwargs)
        self.deny_prefixes = [p if p.endswith("/") else p + "/" for p in deny_prefixes]

    def write(self, file_path: str, content: str) -> WriteResult:
        if any(file_path.startswith(p) for p in self.deny_prefixes):
            return WriteResult(error=f"Writes are not allowed under {file_path}")
        return super().write(file_path, content)

    def edit(self, file_path: str, old_string: str, new_string: str, replace_all: bool = False) -> EditResult:
        if any(file_path.startswith(p) for p in self.deny_prefixes):
            return EditResult(error=f"Edits are not allowed under {file_path}")
        return super().edit(file_path, old_string, new_string, replace_all)
```

Generic wrapper (works with any backend):

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
from deepagents.backends.protocol import (
    BackendProtocol, WriteResult, EditResult, LsResult, ReadResult, GrepResult, GlobResult,
)

class PolicyWrapper(BackendProtocol):
    def __init__(self, inner: BackendProtocol, deny_prefixes: list[str] | None = None):
        self.inner = inner
        self.deny_prefixes = [p if p.endswith("/") else p + "/" for p in (deny_prefixes or [])]

    def _deny(self, path: str) -> bool:
        return any(path.startswith(p) for p in self.deny_prefixes)

    def ls(self, path: str) -> LsResult:
        return self.inner.ls(path)

    def read(self, file_path: str, offset: int = 0, limit: int = 2000) -> ReadResult:
        return self.inner.read(file_path, offset=offset, limit=limit)
    def grep(self, pattern: str, path: str | None = None, glob: str | None = None) -> GrepResult:
        return self.inner.grep(pattern, path, glob)
    def glob(self, pattern: str, path: str = "/") -> GlobResult:
        return self.inner.glob(pattern, path)
    def write(self, file_path: str, content: str) -> WriteResult:
        if self._deny(file_path):
            return WriteResult(error=f"Writes are not allowed under {file_path}")
        return self.inner.write(file_path, content)
    def edit(self, file_path: str, old_string: str, new_string: str, replace_all: bool = False) -> EditResult:
        if self._deny(file_path):
            return EditResult(error=f"Edits are not allowed under {file_path}")
        return self.inner.edit(file_path, old_string, new_string, replace_all)
```

## Migrate from backend factories

<Warning>
  The backend factory pattern is **deprecated**. Pass pre-constructed backend instances directly instead of factory functions.
</Warning>

Previously, backends like `StateBackend` and `StoreBackend` required a factory function that received a runtime object, because they needed runtime context (state, store) to operate. Backends now resolve this context internally via LangGraph's `get_config()`, `get_store()`, and `get_runtime()` helpers, so you can pass instances directly.

### What changed

| Before (deprecated)                                                  | After                                                   |
| -------------------------------------------------------------------- | ------------------------------------------------------- |
| `backend=lambda rt: StateBackend(rt)`                                | `backend=StateBackend()`                                |
| `backend=lambda rt: StoreBackend(rt)`                                | `backend=StoreBackend()`                                |
| `backend=lambda rt: CompositeBackend(default=StateBackend(rt), ...)` | `backend=CompositeBackend(default=StateBackend(), ...)` |
| `backend: (config) => new StateBackend(config)`                      | `backend: new StateBackend()`                           |
| `backend: (config) => new StoreBackend(config)`                      | `backend: new StoreBackend()`                           |

### Deprecated APIs

| Deprecated                                                | Replacement                                                  |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| Passing a callable to `backend=` in `create_deep_agent`   | Pass a backend instance directly                             |
| `runtime` constructor argument on `StateBackend(runtime)` | `StateBackend()` (no arguments needed)                       |
| `runtime` constructor argument on `StoreBackend(runtime)` | `StoreBackend()` or `StoreBackend(namespace=..., store=...)` |
| `files_update` field on `WriteResult` and `EditResult`    | State writes are now handled internally by the backend       |
| `Command` wrapping in middleware write/edit tools         | Tools return plain strings; no `Command(update=...)` needed  |

<Note>
  The factory pattern still works at runtime and emits a deprecation warning. Update your code to use direct instances before the next major version.
</Note>

### Migration example

```python  theme={"theme":{"light":"catppuccin-latte","dark":"catppuccin-mocha"}}
# Before (deprecated)
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend

agent = create_deep_agent(
    backend=lambda rt: CompositeBackend(
        default=StateBackend(rt),
        routes={"/memories/": StoreBackend(rt, namespace=lambda ctx: (ctx.runtime.context.user_id,))},
    ),
)

# After
agent = create_deep_agent(
    backend=CompositeBackend(
        default=StateBackend(),
        routes={"/memories/": StoreBackend(namespace=lambda ctx: (ctx.runtime.context.user_id,))},
    ),
)
```

## Protocol reference

Backends must implement [`BackendProtocol`](https://reference.langchain.com/python/deepagents/backends/protocol/BackendProtocol).

Required methods:

* `ls(path: str) -> LsResult`
  * Return entries with at least `path`. Include `is_dir`, `size`, `modified_at` when available. Sort by `path` for deterministic output.
* `read(file_path: str, offset: int = 0, limit: int = 2000) -> ReadResult`
  * Return file data on success. On missing file, return `ReadResult(error="Error: File '/x' not found")`.
* `grep(pattern: str, path: Optional[str] = None, glob: Optional[str] = None) -> GrepResult`
  * Return structured matches. On error, return `GrepResult(error="...")` (do not raise).
* `glob(pattern: str, path: str = "/") -> GlobResult`
  * Return matched files as `FileInfo` entries (empty list if none).
* `write(file_path: str, content: str) -> WriteResult`
  * Create-only. On conflict, return `WriteResult(error=...)`. On success, set `path` and for state backends set `files_update={...}`; external backends should use `files_update=None`.
* `edit(file_path: str, old_string: str, new_string: str, replace_all: bool = False) -> EditResult`
  * Enforce uniqueness of `old_string` unless `replace_all=True`. If not found, return error. Include `occurrences` on success.

Supporting types:

* `LsResult(error, entries)` — `entries` is a `list[FileInfo]` on success, `None` on failure.
* `ReadResult(error, file_data)` — `file_data` is a `FileData` dict on success, `None` on failure.
* `GrepResult(error, matches)` — `matches` is a `list[GrepMatch]` on success, `None` on failure.
* `GlobResult(error, matches)` — `matches` is a `list[FileInfo]` on success, `None` on failure.
* `WriteResult(error, path, files_update)`
* `EditResult(error, path, files_update, occurrences)`
* `FileInfo` with fields: `path` (required), optionally `is_dir`, `size`, `modified_at`.
* `GrepMatch` with fields: `path`, `line`, `text`.
* `FileData` with fields: `content` (str), `encoding` (`"utf-8"` or `"base64"`), `created_at`, `modified_at`.
  :::

***

<div className="source-links">
  <Callout icon="edit">
    [Edit this page on GitHub](https://github.com/langchain-ai/docs/edit/main/src/oss/deepagents/backends.mdx) or [file an issue](https://github.com/langchain-ai/docs/issues/new/choose).
  </Callout>

  <Callout icon="terminal-2">
    [Connect these docs](/use-these-docs) to Claude, VSCode, and more via MCP for real-time answers.
  </Callout>
</div>
