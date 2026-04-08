---
title: MCP Server
sidebarTitle: MCP Server
description: Native Model Context Protocol (MCP) server for Langfuse, enabling AI assistants to interact with your Langfuse data programmatically.
---

# Langfuse MCP Server

Langfuse includes a native [Model Context Protocol](https://modelcontextprotocol.io) (MCP) server that enables AI assistants and agents to interact with your Langfuse data programmatically.

Currently, the MCP server is available for [Prompt Management](/docs/prompt-management/overview) and will be extended to the rest of the Langfuse data platform in the future. If you have feedback or ideas for new tools, please [share them on GitHub](https://github.com/orgs/langfuse/discussions/10605).


> ℹ️ **Note:** This is the authenticated MCP server for the Langfuse data platform. There is also a public MCP server for the Langfuse documentation ([docs](/docs/docs-mcp)).


> ℹ️ **Note:** Alternatively you can also work with the [Langfuse Agent Skill](/docs/api-and-data-platform/features/agent-skill). Many use cases that aren't supported by the MCP server can be achieved using the skill.


## Configuration

The Langfuse MCP server uses a stateless architecture where each API key is scoped to a specific project. Use the following configuration to connect to the MCP server:

- Endpoint: `https://cloud.langfuse.com/api/public/mcp`
- Transport: `streamableHttp`
- Authentication: Basic Auth via authorization header


- Endpoint: `https://us.cloud.langfuse.com/api/public/mcp`
- Transport: `streamableHttp`
- Authentication: Basic Auth via authorization header


- Endpoint: `https://hipaa.cloud.langfuse.com/api/public/mcp`
- Transport: `streamableHttp`
- Authentication: Basic Auth via authorization header


- Endpoint: `https://your-domain.com/api/public/mcp`
- Transport: `streamableHttp`
- Authentication: Basic Auth via authorization header


## Available Tools

The MCP server provides five tools for comprehensive prompt management.


> ⚠️ **Note:** **Both read and write tools are available by default.** If you only want to
>   use read-only tools, configure your MCP client with an allowlist to restrict
>   access to write operations (`createTextPrompt`, `createChatPrompt`,
>   `updatePromptLabels`).


### Read Operations

- **`getPrompt`** - Fetch a specific prompt by name with optional label or version

  - Supports filtering by production/staging labels
  - Returns compiled prompt with metadata
  - Read-only operation (auto-approved by clients)

- **`listPrompts`** - List all prompts in the project
  - Optional filtering by name, tag, or label
  - Cursor-based pagination support
  - Returns prompt metadata and available versions

### Write Operations

- **`createTextPrompt`** - Create a new text prompt version

  - Supports template variables with `{{variable}}` syntax
  - Optional labels, config, tags, and commit message
  - Automatic version incrementing

- **`createChatPrompt`** - Create a new chat prompt version

  - OpenAI-style message format (role + content)
  - Supports system, user, and assistant roles
  - Template variables in message content

- **`updatePromptLabels`** - Manage labels across prompt versions
  - Add or move labels between versions
  - Labels are unique (auto-removed from other versions)
  - Cannot modify the auto-managed `latest` label

## Set up


### Get Authentication Header

1. Navigate to your project settings and create or copy a **project-scoped API key**:
   - Public Key: `pk-lf-...`
   - Secret Key: `sk-lf-...`
2. Encode the credentials to base64 format:
   ```bash
   echo -n "pk-lf-your-public-key:sk-lf-your-secret-key" | base64
   ```

### Client Setup

1. Register the Langfuse MCP server with a single command, replace `{your-base64-token}` with your encoded credentials:

   ```bash /{your-base64-token}/
   # Langfuse Cloud (EU)
   claude mcp add --transport http langfuse https://cloud.langfuse.com/api/public/mcp \
       --header "Authorization: Basic {your-base64-token}"

   # Langfuse Cloud (US)
   claude mcp add --transport http langfuse https://us.cloud.langfuse.com/api/public/mcp \
       --header "Authorization: Basic {your-base64-token}"

   # Langfuse Cloud (HIPAA)
   claude mcp add --transport http langfuse https://hipaa.cloud.langfuse.com/api/public/mcp \
       --header "Authorization: Basic {your-base64-token}"

   # Self-Hosted (HTTPS required)
   claude mcp add --transport http langfuse https://your-domain.com/api/public/mcp \
       --header "Authorization: Basic {your-base64-token}"

   # Local Development
   claude mcp add --transport http langfuse http://localhost:3000/api/public/mcp \
       --header "Authorization: Basic {your-base64-token}"
   ```

2. Verify the connection by asking Claude Code to `list all prompts in the project`. Claude Code should use the `listPrompts` tool to return the list of prompts.


1. Open Cursor Settings (`Cmd/Ctrl + Shift + J`)
2. Navigate to **Tools & Integrations** tab
3. Click **"Add Custom MCP"**
4. Add your Langfuse MCP server configuration, replace `{your-base64-token}` with your encoded credentials:

```json /{your-base64-token}/
{
  "mcp": {
    "servers": {
      "langfuse": {
        "url": "https://cloud.langfuse.com/api/public/mcp",
        "headers": {
          "Authorization": "Basic {your-base64-token}"
        }
      }
    }
  }
}
```


```json /{your-base64-token}/
{
  "mcp": {
    "servers": {
      "langfuse": {
        "url": "https://us.cloud.langfuse.com/api/public/mcp",
        "headers": {
          "Authorization": "Basic {your-base64-token}"
        }
      }
    }
  }
}
```


```json /{your-base64-token}/
{
  "mcp": {
    "servers": {
      "langfuse": {
        "url": "https://hipaa.cloud.langfuse.com/api/public/mcp",
        "headers": {
          "Authorization": "Basic {your-base64-token}"
        }
      }
    }
  }
}
```


```json /{your-base64-token}/
{
  "mcp": {
    "servers": {
      "langfuse": {
        "url": "https://your-domain.com/api/public/mcp",
        "headers": {
          "Authorization": "Basic {your-base64-token}"
        }
      }
    }
  }
}
```


5. Save the file and restart Cursor
6. The server should appear in the MCP settings with a green dot indicating it's active


- Endpoint: `/api/public/mcp`
  - EU: `https://cloud.langfuse.com/api/public/mcp`
  - US: `https://us.cloud.langfuse.com/api/public/mcp`
  - HIPAA: `https://hipaa.cloud.langfuse.com/api/public/mcp`
  - Self-Hosted: `https://your-domain.com/api/public/mcp`
- Transport: `streamableHttp`
- Authentication: Basic Auth via authorization header
  - `Authorization: Basic {your-base64-token}`


## Use Cases

The MCP server enables powerful workflows for AI-assisted prompt management:

- **Prompt Creation**: "Create a new chat prompt for customer support with system instructions and example messages"
- **Version Management**: "Update the staging label to point to version 3 of the email-generation prompt"
- **Prompt Discovery**: "List all prompts tagged with 'production' and show their latest versions"
- **Iterative Development**: "Create a new version of the code-review prompt with improved instructions"

## Feedback

We'd love to hear about your experience with the Langfuse MCP server. Share your feedback, ideas, and use cases in our [GitHub Discussion](https://github.com/orgs/langfuse/discussions/10605).

## Related Documentation

- [Prompt Management with MCP](/docs/prompt-management/features/mcp-server) - Prompt-specific workflows and examples
- [Prompt Management Overview](/docs/prompt-management/overview) - Learn about Langfuse prompt management
- [Public API](/docs/api-and-data-platform/features/public-api) - REST API for programmatic access
