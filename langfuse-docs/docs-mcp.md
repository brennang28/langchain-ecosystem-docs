---
title: Docs MCP Server
---

# Langfuse Docs MCP Server

The Langfuse Docs MCP server exposes the Langfuse docs to AI agents.

Core use case: Use Cursor (or other AI Coding Agent) to automatically integrate Langfuse Tracing into your codebase, see [get started](/docs/get-started) for detailed instructions and an example prompt.


> ℹ️ **Note:** This is the public MCP server for the Langfuse documentation. There is also an authenticated MCP server to integrate with the rest of the Langfuse data platform ([docs](/docs/api-and-data-platform/features/mcp-server)).


## Install


Add Langfuse Docs MCP to Cursor via the one-click install:


<details>
<summary>Manual configuration</summary>

Add the following to your `mcp.json`:

```json
{
  "mcpServers": {
    "langfuse-docs": {
      "url": "https://langfuse.com/api/mcp"
    }
  }
}
```

</details>


Add Langfuse Docs MCP to Copilot in VSCode via the one-click install:


<details>
<summary>Manual configuration</summary>

Add Langfuse Docs MCP to Copilot in VSCode via the following steps:

1. Open Command Palette (⌘+Shift+P)
2. Open "MCP: Add Server..."
3. Select `HTTP`
4. Paste `https://langfuse.com/api/mcp`
5. Select name (e.g. `langfuse-docs`) and whether to save in user or workspace settings
6. You're all set! The MCP server is now available in Agent mode

</details>


Add Langfuse Docs MCP to Claude Code via the CLI:

```bash
claude mcp add \
  --transport http \
  langfuse-docs \
  https://langfuse.com/api/mcp \
  --scope user
```

<details>
<summary>Manual configuration</summary>

Alternatively, add the following to your settings file:

- **User scope**: `~/.claude/settings.json`
- **Project scope**: `your-repo/.claude/settings.json`
- **Local scope**: `your-repo/.claude/settings.local.json`

```json
{
  "mcpServers": {
    "langfuse-docs": {
      "transportType": "http",
      "url": "https://langfuse.com/api/mcp",
      "verifySsl": true
    }
  }
}
```

**One-liner JSON import**

```bash
claude mcp add-json langfuse-docs \
  '{"type":"http","url":"https://langfuse.com/api/mcp"}'
```

Once added, start a Claude Code session (`claude`) and type `/mcp` to confirm the connection.

</details>


Add Langfuse Docs MCP to Windsurf via the following steps:

1. Open Command Palette (⌘+Shift+P)
2. Open "MCP Configuration Panel"
3. Select `Add custom server`
4. Add the following configuration:

   ```json
   {
     "mcpServers": {
       "langfuse-docs": {
         "command": "npx",
         "args": ["mcp-remote", "https://langfuse.com/api/mcp"]
       }
     }
   }
   ```


Langfuse uses the `streamableHttp` protocol to communicate with the MCP server. This is supported by most clients.

```json
{
  "mcpServers": {
    "langfuse-docs": {
      "url": "https://langfuse.com/api/mcp"
    }
  }
}
```

If you use a client that does not support `streamableHttp` (e.g. Windsurf), you can use the `mcp-remote` command as a local proxy.

```json
{
  "mcpServers": {
    "langfuse-docs": {
      "command": "npx",
      "args": ["mcp-remote", "https://langfuse.com/api/mcp"]
    }
  }
}
```


## About

- Endpoint: `https://langfuse.com/api/mcp`
- Transport: `streamableHttp`
- Authentication: None
- Tools:
  - `searchLangfuseDocs`: Semantic search (RAG) over the Langfuse documentation. Returns a concise answer synthesized from relevant docs. Use for broader questions; prefer getLangfuseDocsPage for specific pages. Powered by [Inkeep RAG API](https://docs.inkeep.com/ai-api/rag-mode/http-request).
  - `getLangfuseDocsPage`: Fetch the raw Markdown for a specific Langfuse docs page. Accepts a docs path (e.g., `/docs/observability/overview`) or a full `https://langfuse.com` URL. Use for specific pages, integrations, or code samples.
  - `getLangfuseOverview`: Get a high-level index by fetching [llms.txt](https://langfuse.com/llms.txt). Use at the start of a session to discover key docs endpoints. Avoid repeated calls.

## References

- Implementation of the MCP server: [route.ts](https://github.com/langfuse/langfuse-docs/blob/main/app/api/mcp/route.ts)
- [Agentic Onboarding](/docs/get-started) powered by the MCP server
- [Ask AI](/docs/ask-ai): RAG chat with the Langfuse docs to get answers to your questions
- [langfuse.com/llms.txt](https://langfuse.com/llms.txt): concise overview with page titles and links to detailed sub-files ([llms-docs.txt](https://langfuse.com/llms-docs.txt), [llms-integrations.txt](https://langfuse.com/llms-integrations.txt), [llms-self-hosting.txt](https://langfuse.com/llms-self-hosting.txt))

## REST Endpoint

The underlying docs search (`searchLangfuseDocs` tool) is also available independently as a REST API at `https://langfuse.com/api/search-docs`.

```bash
curl "https://langfuse.com/api/search-docs?query=Langfuse+Docs+MCP+Server"
```

Use this endpoint directly when you need lightweight semantic search outside of MCP.
