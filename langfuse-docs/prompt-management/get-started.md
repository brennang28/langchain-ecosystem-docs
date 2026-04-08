---
title: Get Started
sidebarTitle: Get Started
description: Get started with Langfuse Prompt Management.
---

# Get Started with Prompt Management

This guide walks you through creating and using a prompt with Langfuse. If you're looking to understand what prompt management is and why it matters, check out the [Prompt Management Overview](/docs/prompt-management/overview) first. For details on how prompts are structured in Langfuse and how it works in the background, see [Core Concepts](/docs/prompt-management/data-model).


## Agentic installation [#agentic-installation]

Install the [Langfuse AI Skill](https://github.com/langfuse/skills) to let your coding agent access all Langfuse features.

Ask your coding agent to install the skill by pointing to the [GitHub repository](https://github.com/langfuse/skills) and instruct it to migrate your prompts.

```txt
Install the Langfuse AI skill from github.com/langfuse/skills
and use it to migrate the prompts in this codebase to Langfuse.
```


Langfuse has a [Cursor Plugin](https://cursor.com/docs/plugins) that includes the skill automatically.


Then prompt your agent:

```txt
Migrate the prompts in this codebase to Langfuse.
```


Install via npm ([skills CLI](https://www.npmjs.com/package/skills)):

```bash
npx skills add langfuse/skills --skill "langfuse"
```

If you want to target a specific agent directly:

```bash
npx skills add langfuse/skills --skill "langfuse" --agent "<agent-id>"
```

<details>
<summary>Alternatively you can manually clone the skill</summary>

1. Clone repo somewhere stable
```bash
git clone https://github.com/langfuse/skills.git /path/to/langfuse-skills
```

2. Make sure your agent's skills dir exists
```bash
mkdir -p /path/to/<agent-skill-root>/skills
```

3. Symlink the skill folder
```bash
ln -s /path/to/langfuse-skills/skills/langfuse /path/to/<agent-skill-root>/skills/langfuse
```

</details>

Then prompt your agent:

```txt
Migrate the prompts in this codebase to Langfuse.
```


## Manual installation [#manual-installation]
This guide helps you get started with Langfuse Prompt Management manually.


### Get API keys

1.  [Create Langfuse account](https://langfuse.com/cloud) or [self-host Langfuse](/self-hosting).
2.  Create new API credentials in the project settings.

### Create a prompt [#create-update-prompt-diy]


Use the Langfuse UI to create a new prompt or update an existing one. You'll need to select the [prompt type](/docs/prompt-management/data-model#text-vs-chat-prompts), you can't change this afterwards.

```bash
pip install langfuse
```

Add your Langfuse credentials as environment variables so the SDK knows which project to create the prompt in.


```bash
LANGFUSE_SECRET_KEY = "sk-lf-..."
LANGFUSE_PUBLIC_KEY = "pk-lf-..."
LANGFUSE_BASE_URL = "https://cloud.langfuse.com" # 🇪🇺 EU region
# LANGFUSE_BASE_URL = "https://us.cloud.langfuse.com" # 🇺🇸 US region
```


Use the Python SDK to create a new prompt or update an existing one.

```python
# Create a text prompt
langfuse.create_prompt(
    name="movie-critic",
    type="text",
    prompt="As a {{criticlevel}} movie critic, do you like {{movie}}?",
    labels=["production"]  # optionally, directly promote to production
)

# Create a chat prompt
langfuse.create_prompt(
    name="movie-critic-chat",
    type="chat",
    prompt=[
      { "role": "system", "content": "You are an {{criticlevel}} movie critic" },
      { "role": "user", "content": "Do you like {{movie}}?" },
    ],
    labels=["production"]  # optionally, directly promote to production
)
```

If you already have a prompt with the same `name`, the prompt will be added as a new version.


```bash
npm i @langfuse/client
```

Add your Langfuse credentials as environment variables so the SDK knows which project to create the prompt in.


```bash
LANGFUSE_SECRET_KEY = "sk-lf-..."
LANGFUSE_PUBLIC_KEY = "pk-lf-..."
LANGFUSE_BASE_URL = "https://cloud.langfuse.com" # 🇪🇺 EU region
# LANGFUSE_BASE_URL = "https://us.cloud.langfuse.com" # 🇺🇸 US region
```


```ts
const langfuse = new LangfuseClient();
```

Use the JS/TS SDK to create a new prompt or update an existing one.

```ts
// Create a text prompt
await langfuse.prompt.create({
  name: "movie-critic",
  type: "text",
  prompt: "As a {{criticlevel}} critic, do you like {{movie}}?",
  labels: ["production"] // optionally, directly promote to production
});

// Create a chat prompt
await langfuse.prompt.create({
  name: "movie-critic-chat",
  type: "chat",
  prompt: [
    { role: "system", content: "You are an {{criticlevel}} movie critic" },
    { role: "user", content: "Do you like {{movie}}?" },
  ],
  labels: ["production"] // optionally, directly promote to production
});
```

If you already have a prompt with the same `name`, the prompt will be added as a new version.


Use the [Public API](https://api.reference.langfuse.com/#tag/prompts/post/api/public/v2/prompts) to create a new prompt or update an existing one.

```bash
curl -X POST "https://cloud.langfuse.com/api/public/v2/prompts" \
  -u "your-public-key:your-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "chat",
    "name": "movie-critic",
    "prompt": [
      { "role": "system", "content": "You are an {{criticlevel}} movie critic" },
      { "role": "user", "content": "Do you like {{movie}}?" }
    ]
  }'

```


If you have prompts in your existing codebase, you can migrate them to Langfuse programmatically.

**Using the Langfuse Skill**

1. Install the [Langfuse Skill](https://github.com/langfuse/skills):

```bash
# Cursor plugin
/add-plugin langfuse

# skills CLI
npx skills add langfuse/skills --skill "langfuse"

# Manual: clone and symlink
git clone https://github.com/langfuse/skills.git /path/to/langfuse-skills
ln -s /path/to/langfuse-skills/skills/langfuse ~/.skills/langfuse
```

2. Ask the agent to migrate your prompts:

```
Migrate the hardcoded prompts in this codebase to Langfuse prompt management.
```

**Using the API**

You can write a script that reads your existing prompts and creates them in Langfuse using the [Public API](https://api.reference.langfuse.com/#tag/prompts/post/api/public/v2/prompts). This is ideal for bulk migrations or CI/CD integration.


Things to look out for

- Langfuse uses a specific syntax for [variables, prompt references, and message placeholders](/docs/prompt-management/data-model#dynamic-rendering-of-prompts). Make sure to update your prompts to use the correct format, if you want to use Langfuse's dynamic rendering capabilities.


### Use the prompt in your code [#use-prompt-diy]


At runtime, you can fetch the prompt from Langfuse. We recommend using the `production` label to fetch the version intentionally chosen for production. Learn more about control (versions/labels) [here](/docs/prompt-management/features/prompt-version-control).

```python
from langfuse import get_client

# Initialize Langfuse client
langfuse = get_client()
```

Below are code examples for both a text type prompt and a chat type prompt. Learn more about prompt types [here](/docs/prompt-management/data-model#text-vs-chat-prompts).

**Text prompt**

```python
# By default, the production version is fetched.
prompt = langfuse.get_prompt("movie-critic")

# Insert variables into prompt template
compiled_prompt = prompt.compile(criticlevel="expert", movie="Dune 2")
# -> "As an expert movie critic, do you like Dune 2?"
```

**Chat prompt**

```python
# By default, the production version of a chat prompt is fetched.
chat_prompt = langfuse.get_prompt("movie-critic-chat", type="chat") # type arg infers the prompt type (default is 'text')

# Insert variables into chat prompt template
compiled_chat_prompt = chat_prompt.compile(criticlevel="expert", movie="Dune 2")
# -> [{"role": "system", "content": "You are an expert movie critic"}, {"role": "user", "content": "Do you like Dune 2?"}]
```


```ts
// Initialize the Langfuse client
const langfuse = new LangfuseClient();
```

Below are code examples for both a text type prompt and a chat type prompt. Learn more about prompt types [here](/docs/prompt-management/data-model#text-vs-chat-prompts).

**Text prompt**

```ts
// By default, the production version of a text prompt is fetched.
const prompt = await langfuse.prompt.get("movie-critic");

// Insert variables into prompt template
const compiledPrompt = prompt.compile({
  criticlevel: "expert",
  movie: "Dune 2",
});
// -> "As an expert movie critic, do you like Dune 2?"
```

**Chat prompt**

```ts
// By default, the production version of a chat prompt is fetched.
const chatPrompt = await langfuse.prompt.get("movie-critic-chat", {
  type: "chat",
}); // type option infers the prompt type (default is 'text')

// Insert variables into chat prompt template
const compiledChatPrompt = chatPrompt.compile({
  criticlevel: "expert",
  movie: "Dune 2",
});
// -> [{"role": "system", "content": "You are an expert movie critic"}, {"role": "user", "content": "Do you like Dune 2?"}]
```


Use the [Public API](https://api.reference.langfuse.com/#tag/prompts/get/api/public/v2/prompts/{promptName}) to fetch a prompt at runtime. By default, the prompt labeled `production` is returned.

```bash
curl "https://cloud.langfuse.com/api/public/v2/prompts/movie-critic?label=production" \
  -u "your-public-key:your-secret-key"
```

For fetching a specific version instead of a label:

```bash
curl "https://cloud.langfuse.com/api/public/v2/prompts/movie-critic?version=1" \
  -u "your-public-key:your-secret-key"
```


```bash
pip install langfuse openai
```

```python
import openai
from langfuse import get_client

# Initialize Langfuse client
langfuse = get_client()
```

Below are code examples for both a text type prompt and a chat type prompt. Learn more about prompt types [here](/docs/prompt-management/data-model#text-vs-chat-prompts).

**Text prompt**

```python
# By default, the production version of a text prompt is fetched.
prompt = langfuse.get_prompt("movie-critic")

# Compile the prompt with variables
compiled_prompt = prompt.compile(criticlevel="expert", movie="Dune 2")

# Use with OpenAI - prompt is a string
completion = openai.chat.completions.create(
  model="gpt-4o",
  messages=[{"role": "user", "content": compiled_prompt}]
)
```

**Chat prompt**

```python
# By default, the production version of a chat prompt is fetched.
chat_prompt = langfuse.get_prompt("movie-critic-chat", type="chat")

# Compile the prompt with variables - returns a list of message dicts
compiled_chat_prompt = chat_prompt.compile(criticlevel="expert", movie="Dune 2")

# Use with OpenAI - prompt is a list of messages
completion = openai.chat.completions.create(
  model="gpt-4o",
  messages=compiled_chat_prompt
)
```

**Example notebook**


}
  />

```bash
npm install @langfuse/openai openai
```

```typescript
// Initialize Langfuse client
const langfuse = new LangfuseClient();

// Wrap OpenAI client
const openai = observeOpenAI(new OpenAI());
```

Below are code examples for both a text type prompt and a chat type prompt. Learn more about prompt types [here](/docs/prompt-management/data-model#text-vs-chat-prompts).

**Text prompt**

```typescript
// By default, the production version of a text prompt is fetched.
const prompt = await langfuse.prompt.get("movie-critic", {
  type: "text",
});

// Compile the prompt with variables
const compiledPrompt = prompt.compile({
  criticlevel: "expert",
  movie: "Dune 2",
});

// Use with OpenAI - prompt is a string
const completion = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [{ role: "user", content: compiledPrompt }],
});
```

**Chat prompt**

```typescript
// By default, the production version of a chat prompt is fetched.
const chatPrompt = await langfuse.prompt.get("movie-critic-chat", {
  type: "chat",
});

// Compile the prompt with variables - returns an array of messages
const compiledChatPrompt = chatPrompt.compile({
  criticlevel: "expert",
  movie: "Dune 2",
});

// Use with OpenAI - prompt is an array of messages
const completion = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: compiledChatPrompt,
});
```


```python
from langfuse import Langfuse
from langchain_core.prompts import ChatPromptTemplate

# Initialize Langfuse client
langfuse = Langfuse()
```

Below are code examples for both a text type prompt and a chat type prompt. Learn more about prompt types [here](/docs/prompt-management/data-model#text-vs-chat-prompts).

These examples contain [variables](/docs/prompt-management/features/variables). As Langfuse and Langchain process input variables of prompt templates differently (`{}` instead of `{{}}`), we provide the `prompt.get_langchain_prompt()` method to transform the Langfuse prompt into a string that can be used with Langchain's PromptTemplate. You can pass optional keyword arguments to `prompt.get_langchain_prompt(**kwargs)` in order to precompile some variables and handle the others with Langchain's PromptTemplate.


**Text prompt**

```python
# By default, the production version of a text prompt is fetched.
langfuse_prompt = langfuse.get_prompt("movie-critic")

# Example using ChatPromptTemplate
langchain_prompt = ChatPromptTemplate.from_template(langfuse_prompt.get_langchain_prompt())

# Example using ChatPromptTemplate with pre-compiled variables.
langchain_prompt = ChatPromptTemplate.from_template(langfuse_prompt.get_langchain_prompt(strictness='tough'))
```

**Chat prompt**

```python
# By default, the production version of a chat prompt is fetched.
langfuse_prompt = langfuse.get_prompt("movie-critic-chat", type="chat")

# Create a Langchain ChatPromptTemplate from the Langfuse prompt chat messages
langchain_prompt = ChatPromptTemplate.from_messages(langfuse_prompt.get_langchain_prompt())
```

**Example notebook**


}
  />

```ts
const langfuse = new LangfuseClient();
```

Below are code examples for both a text type prompt and a chat type prompt. Learn more about prompt types [here](/docs/prompt-management/data-model#text-vs-chat-prompts).

These examples contain [variables](/docs/prompt-management/features/variables). As Langfuse and Langchain process input variables of prompt templates differently (`{}` instead of `{{}}`), we provide the `prompt.get_langchain_prompt()` method to transform the Langfuse prompt into a string that can be used with Langchain's PromptTemplate. You can pass optional keyword arguments to `prompt.get_langchain_prompt(**kwargs)` in order to precompile some variables and handle the others with Langchain's PromptTemplate.


**Text prompt**

```ts
// Get current `production` version
const langfusePrompt = await langfuse.prompt.get("movie-critic");

// Example using ChatPromptTemplate
const promptTemplate = PromptTemplate.fromTemplate(
  langfusePrompt.getLangchainPrompt()
);
```

**Chat prompt**

```ts
// Get current `production` version of a chat prompt
const langfusePrompt = await langfuse.prompt.get(
  "movie-critic-chat",
  { type: "chat" }
);

// Example using ChatPromptTemplate
const promptTemplate = ChatPromptTemplate.fromMessages(
  langfusePrompt.getLangchainPrompt().map((msg) => [msg.role, msg.content])
);
```

**Example notebook**


}
  />

Use Langfuse Prompt Management with the Vercel AI SDK.

```bash
npm install @langfuse/client ai
```

```typescript
// Initialize Langfuse client
const langfuse = new LangfuseClient();
```

Below are code examples for both a text type prompt and a chat type prompt. Learn more about prompt types [here](/docs/prompt-management/data-model#text-vs-chat-prompts).

**Text prompt**

```typescript
// By default, the production version of a text prompt is fetched.
const prompt = await langfuse.prompt.get("movie-critic", {
  type: "text",
});

// Compile the prompt with variables
const compiledPrompt = prompt.compile({
  criticlevel: "expert",
  movie: "Dune 2",
});

// Use with Vercel AI SDK
const result = await generateText({
  model: openai("gpt-4o"),
  prompt: compiledPrompt,
  experimental_telemetry: {
    isEnabled: true,
  },
});
```

**Chat prompt**

```typescript
// By default, the production version of a chat prompt is fetched.
const chatPrompt = await langfuse.prompt.get("movie-critic-chat", {
  type: "chat",
});

// Compile the prompt with variables - returns an array of messages
const compiledChatPrompt = chatPrompt.compile({
  criticlevel: "expert",
  movie: "Dune 2",
});

// Use with Vercel AI SDK
const result = await generateText({
  model: openai("gpt-4o"),
  messages: compiledChatPrompt,
  experimental_telemetry: {
    isEnabled: true,
  },
});
```


Not seeing your latest version? This might be because of the caching behavior. See [prompt caching](/docs/prompt-management/data-model#prompt-caching) for more details.


## Not seeing what you expected?

## Next steps

Now that you've used your first prompt, there are a couple of things we recommend you do next to make the most of Langfuse Prompt Management:

- [Link prompts to traces](/docs/prompt-management/features/link-to-traces) to analyze performance by prompt version
- [Use version control and labels](/docs/prompt-management/features/prompt-version-control) to manage deployments across environments

Looking for something specific? Take a look under _Features_ for guides on specific topics.
