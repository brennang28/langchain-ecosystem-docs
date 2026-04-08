---
title: Get Started
description: Get started with LLM observability with Langfuse in minutes before diving into all platform features.
---

# Get Started with Tracing

This guide walks you through ingesting your first trace into Langfuse. If you're looking to understand what tracing is and why it matters, check out the [Observability Overview](/docs/observability/overview) first. For details on how traces are structured in Langfuse and how it works in the background, see [Core Concepts](/docs/observability/data-model).


## Agentic installation [#agentic-installation]

Install the [Langfuse AI Skill](https://github.com/langfuse/skills) to let your coding agent access all Langfuse features.

Ask your coding agent to install the skill by pointing to the [GitHub repository](https://github.com/langfuse/skills) and instruct it to get started with tracing.

```txt
Install the Langfuse AI skill from github.com/langfuse/skills
and use it to add tracing to this application with Langfuse
following best practices.
```


Langfuse has a [Cursor Plugin](https://cursor.com/docs/plugins) that includes the skill automatically.


Then prompt your agent:

```txt
Add tracing to this application with Langfuse following best practices.
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
Add tracing to this application with Langfuse following best practices.
```


## Manual installation [#manual-installation]
This guide helps you get started with Langfuse tracing manually.


### Get API keys

1.  [Create Langfuse account](https://cloud.langfuse.com/auth/sign-up) or [self-host Langfuse](/self-hosting).
2.  Create new API credentials in the project settings.

### Ingest your first trace

Choose your framework or SDK to get started:

Langfuse's OpenAI SDK is a drop-in replacement for the OpenAI client that automatically records your model calls without changing how you write code. If you already use the OpenAI python SDK, you can start using Langfuse with minimal changes to your code.

Start by installing the Langfuse OpenAI SDK. It includes the wrapped OpenAI client and sends traces in the background.


```bash
pip install langfuse
```

Set your Langfuse credentials as environment variables so the SDK knows which project to write to.

```bash
LANGFUSE_SECRET_KEY = "sk-lf-..."
LANGFUSE_PUBLIC_KEY = "pk-lf-..."
LANGFUSE_BASE_URL = "https://cloud.langfuse.com" # 🇪🇺 EU region
# LANGFUSE_BASE_URL = "https://us.cloud.langfuse.com" # 🇺🇸 US region
```


Swap the regular OpenAI import to Langfuse’s OpenAI drop-in. It behaves like the regular OpenAI client while also recording each call for you.

```python
from langfuse.openai import openai
```

Use the OpenAI SDK as you normally would. The wrapper captures the prompt, model and output and forwards everything to Langfuse.

```python
completion = openai.chat.completions.create(
  name="test-chat",
  model="gpt-4o",
  messages=[
      {"role": "system", "content": "You are a very accurate calculator. You output only the result of the calculation."},
      {"role": "user", "content": "1 + 1 = "}],
  metadata={"someMetadataKey": "someValue"},
)
```


}
    title="Full OpenAI SDK documentation"
    href="/integrations/model-providers/openai-py"
    arrow
  />
  <img
          src="/images/integrations/colab_icon.png"
          />
      
}
    title="Notebook example"
    href="https://colab.research.google.com/github/langfuse/langfuse-docs/blob/main/cookbook/integration_openai_sdk.ipynb"
    arrow
  />

Langfuse's JS/TS OpenAI SDK wraps the official client so your model calls are automatically traced and sent to Langfuse. If you already use the OpenAI JavaScript SDK, you can start using Langfuse with minimal changes to your code.

First install the Langfuse OpenAI wrapper. It extends the official client to send traces in the background.


**Install package**
```sh
npm install @langfuse/openai
```

**Add credentials**

Add your Langfuse credentials to your environment variables so the SDK knows which project to write to. 


```bash
LANGFUSE_SECRET_KEY = "sk-lf-..."
LANGFUSE_PUBLIC_KEY = "pk-lf-..."
LANGFUSE_BASE_URL = "https://cloud.langfuse.com" # 🇪🇺 EU region
# LANGFUSE_BASE_URL = "https://us.cloud.langfuse.com" # 🇺🇸 US region
```


**Initialize OpenTelemetry**


Install the OpenTelemetry SDK, which the Langfuse integration uses under the hood to capture the data from each OpenAI call.

```bash
npm install @opentelemetry/sdk-node
```

Next is initializing the Node SDK. You can do that either in a dedicated instrumentation file or directly at the top of your main file.


The inline setup is the simplest way to get started. It works well for projects where your main file is executed first and import order is straightforward.

We can now initialize the `LangfuseSpanProcessor` and start the SDK. The `LangfuseSpanProcessor` is the part that takes that collected data and sends it to your Langfuse project. 

Important: start the SDK before initializing the logic that needs to be traced to avoid losing data.

```ts
const sdk = new NodeSDK({
  spanProcessors: [new LangfuseSpanProcessor()],
});
 
sdk.start();
```


The instrumentation file often preferred when you're using frameworks that have complex startup order (Next.js, serverless, bundlers) or if you want a clean, predictable place where tracing is always initialized first.

Create an `instrumentation.ts` file, which sets up the _collector_ that gathers data about each OpenAI call. The `LangfuseSpanProcessor` is the part that takes that collected data and sends it to your Langfuse project.

```ts /LangfuseSpanProcessor/
const sdk = new NodeSDK({
  spanProcessors: [new LangfuseSpanProcessor()],
});

sdk.start();
```

Import the `instrumentation.ts` file first so all later imports run with tracing enabled.

```ts
// Must be the first import
```


Wrap your normal OpenAI client. From now on, each OpenAI request  is automatically collected and forwarded to Langfuse.

**Wrap OpenAI client**
```ts
const openai = observeOpenAI(new OpenAI());

const res = await openai.chat.completions.create({
    messages: [{ role: "system", content: "Tell me a story about a dog." }],
    model: "gpt-4o",
    max_tokens: 300,
});
```


}
    title="Full OpenAI SDK documentation"
    href="/integrations/model-providers/openai-js"
    arrow
  />
  }
    title="Notebook"
    href="/guides/cookbook/js_integration_openai"
    arrow
  />

Langfuse's Vercel AI SDK integration uses OpenTelemetry to automatically trace your AI calls. If you already use the Vercel AI SDK, you can start using Langfuse with minimal changes to your code.


**Install packages**

Install the Vercel AI SDK, OpenTelemetry, and the Langfuse integration packages.

```bash
npm install ai @ai-sdk/openai @langfuse/tracing @langfuse/otel @opentelemetry/sdk-node
```

**Add credentials**

Set your Langfuse credentials as environment variables so the SDK knows which project to write to.


```bash
LANGFUSE_SECRET_KEY = "sk-lf-..."
LANGFUSE_PUBLIC_KEY = "pk-lf-..."
LANGFUSE_BASE_URL = "https://cloud.langfuse.com" # 🇪🇺 EU region
# LANGFUSE_BASE_URL = "https://us.cloud.langfuse.com" # 🇺🇸 US region
```


**Initialize OpenTelemetry with Langfuse**

Set up the OpenTelemetry SDK with the Langfuse span processor. This captures telemetry data from the Vercel AI SDK and sends it to Langfuse.

```typescript
const sdk = new NodeSDK({
  spanProcessors: [new LangfuseSpanProcessor()],
});

sdk.start();
```

**Enable telemetry in your AI SDK calls**

Pass `experimental_telemetry: { isEnabled: true }` to your AI SDK functions. The AI SDK automatically creates telemetry spans, which the `LangfuseSpanProcessor` captures and sends to Langfuse.

```typescript
const { text } = await generateText({
  model: openai("gpt-4o"),
  prompt: "What is the weather like today?",
  experimental_telemetry: { isEnabled: true },
});
```


}
    title="Full Vercel AI SDK documentation"
    href="/integrations/frameworks/vercel-ai-sdk"
    arrow
  />

Langfuse's LangChain integration uses a callback handler to record and send traces to Langfuse. If you already use LangChain, you can start using Langfuse with minimal changes to your code.

First install the Langfuse SDK and your LangChain SDK.


```bash
pip install langfuse langchain-openai
```

Add your Langfuse credentials as environment variables so the callback handler knows which project to write to.


```bash
LANGFUSE_SECRET_KEY = "sk-lf-..."
LANGFUSE_PUBLIC_KEY = "pk-lf-..."
LANGFUSE_BASE_URL = "https://cloud.langfuse.com" # 🇪🇺 EU region
# LANGFUSE_BASE_URL = "https://us.cloud.langfuse.com" # 🇺🇸 US region
```


Initialize the Langfuse callback handler. LangChain has its own callback system, and Langfuse listens to those callbacks to record what your chains and LLMs are doing.

```python
from langfuse.langchain import CallbackHandler

langfuse_handler = CallbackHandler()
```

Add the Langfuse callback handler to your chain. The Langfuse callback handler plugs into LangChain’s event system. Every time the chain runs or the LLM is called, LangChain emits events, and the handler turns those into traces and observations in Langfuse.

```python {10}
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
 
llm = ChatOpenAI(model_name="gpt-4o")
prompt = ChatPromptTemplate.from_template("Tell me a joke about {topic}")
chain = prompt | llm
 
response = chain.invoke(
    {"topic": "cats"}, 
    config={"callbacks": [langfuse_handler]})
```


}
    title="Full LangChain SDK documentation"
    href="/integrations/frameworks/langchain"
    arrow
  />
  <img
          src="/images/integrations/colab_icon.png"
          />
      
}
    title="Notebook"
    href="https://colab.research.google.com/github/langfuse/langfuse-docs/blob/main/cookbook/integration_langchain.ipynb"
    arrow
  />

Langfuse's LangChain integration uses a callback handler to record and send traces to Langfuse. If you already use LangChain, you can start using Langfuse with minimal changes to your code.

First install the Langfuse core SDK and the LangChain integration.


```bash
npm install @langfuse/core @langfuse/langchain
```


Add your Langfuse credentials as environment variables so the integration knows which project to send your traces to.


```bash
LANGFUSE_SECRET_KEY = "sk-lf-..."
LANGFUSE_PUBLIC_KEY = "pk-lf-..."
LANGFUSE_BASE_URL = "https://cloud.langfuse.com" # 🇪🇺 EU region
# LANGFUSE_BASE_URL = "https://us.cloud.langfuse.com" # 🇺🇸 US region
```


**Initialize OpenTelemetry**


Install the OpenTelemetry SDK, which the Langfuse integration uses under the hood to capture the data from each OpenAI call.

```bash
npm install @opentelemetry/sdk-node
```

Next is initializing the Node SDK. You can do that either in a dedicated instrumentation file or directly at the top of your main file.


The inline setup is the simplest way to get started. It works well for projects where your main file is executed first and import order is straightforward.

We can now initialize the `LangfuseSpanProcessor` and start the SDK. The `LangfuseSpanProcessor` is the part that takes that collected data and sends it to your Langfuse project. 

Important: start the SDK before initializing the logic that needs to be traced to avoid losing data.

```ts
const sdk = new NodeSDK({
  spanProcessors: [new LangfuseSpanProcessor()],
});
 
sdk.start();
```


The instrumentation file often preferred when you're using frameworks that have complex startup order (Next.js, serverless, bundlers) or if you want a clean, predictable place where tracing is always initialized first.

Create an `instrumentation.ts` file, which sets up the _collector_ that gathers data about each OpenAI call. The `LangfuseSpanProcessor` is the part that takes that collected data and sends it to your Langfuse project.

```ts /LangfuseSpanProcessor/
const sdk = new NodeSDK({
  spanProcessors: [new LangfuseSpanProcessor()],
});

sdk.start();
```

Import the `instrumentation.ts` file first so all later imports run with tracing enabled.

```ts
// Must be the first import
```


Finally, initialize the Langfuse `CallbackHandler` and add it to your chain. The `CallbackHandler` listens to the LangChain agent's actions and prepares that information to be sent to Langfuse.

```typescript
// Initialize the Langfuse CallbackHandler
const langfuseHandler = new CallbackHandler();
```

The line `{ callbacks: [langfuseHandler] }` is what attaches the `CallbackHandler` to the agent.

```typescript /{ callbacks: [langfuseHandler] }/
const getWeather = tool(
  (input) => `It's always sunny in ${input.city}!`,
  {
    name: "get_weather",
    description: "Get the weather for a given city",
    schema: z.object({
      city: z.string().describe("The city to get the weather for"),
    }),
  }
);

const agent = createAgent({
  model: "openai:gpt-5-mini",
  tools: [getWeather],
});

console.log(
    await agent.invoke(
        { messages: [{ role: "user", content: "What's the weather in San Francisco?" }] }, 
        { callbacks: [langfuseHandler] }
    )
);
```


}
    title="Full Langchain SDK documentation"
    href="/integrations/frameworks/langchain"
    arrow
  />
  }
    title="Notebook"
    href="/guides/cookbook/js_integration_langchain"
    arrow
  />

The Langfuse Python SDK gives you full control over how you instrument your application and can be used with any other framework.


**1. Install package:**

```bash
pip install langfuse
```

**2. Add credentials:**


```bash
LANGFUSE_SECRET_KEY = "sk-lf-..."
LANGFUSE_PUBLIC_KEY = "pk-lf-..."
LANGFUSE_BASE_URL = "https://cloud.langfuse.com" # 🇪🇺 EU region
# LANGFUSE_BASE_URL = "https://us.cloud.langfuse.com" # 🇺🇸 US region
```


**3. Instrument your application:**

Instrumentation means adding code that records what’s happening in your application so it can be sent to Langfuse. There are three main ways of instrumenting your code with the Python SDK.

In this example we will use the [context manager](/docs/observability/sdk/instrumentation#context-manager). You can also use the [decorator](/docs/observability/sdk/instrumentation#observe-wrapper) or create [manual observations](/docs/observability/sdk/instrumentation#manual-observations).

```python
from langfuse import get_client

langfuse = get_client()

# Create a span using a context manager
with langfuse.start_as_current_observation(as_type="span", name="process-request") as span:
    # Your processing logic here
    span.update(output="Processing complete")

    # Create a nested generation for an LLM call
    with langfuse.start_as_current_observation(as_type="generation", name="llm-response", model="gpt-3.5-turbo") as generation:
        # Your LLM call logic here
        generation.update(output="Generated response")

# All spans are automatically closed when exiting their context blocks


# Flush events in short-lived applications
langfuse.flush()
```
_[When should I call `langfuse.flush()`?](/docs/observability/data-model#background-processing)_

**4. Run your application and see the trace in Langfuse:**


![First trace in Langfuse](/images/docs/observability/first-trace-python.png)

See the [trace in Langfuse](https://cloud.langfuse.com/project/cloramnkj0002jz088vzn1ja4/traces/b8789d62464dc7627016d9748a48ad0d?observation=5c7c133ec919ded7&timestamp=2025-12-03T14:56:19.285Z).


}
    title="Full Python SDK documentation"
    href="/docs/sdk/python/sdk-v3"
    arrow
  />

Use the Langfuse JS/TS SDK to wrap any LLM or Agent


**Install packages**

Install the Langfuse tracing SDK, the Langfuse OpenTelemetry integration, and the OpenTelemetry Node SDK.

```sh
npm install @langfuse/tracing @langfuse/otel @opentelemetry/sdk-node
```

**Add credentials**


Add your Langfuse credentials to your environment variables so the tracing SDK knows which Langfuse project it should send your recorded data to.


```bash
LANGFUSE_SECRET_KEY = "sk-lf-..."
LANGFUSE_PUBLIC_KEY = "pk-lf-..."
LANGFUSE_BASE_URL = "https://cloud.langfuse.com" # 🇪🇺 EU region
# LANGFUSE_BASE_URL = "https://us.cloud.langfuse.com" # 🇺🇸 US region
```


**Initialize OpenTelemetry**


Install the OpenTelemetry SDK, which the Langfuse integration uses under the hood to capture the data from each OpenAI call.

```bash
npm install @opentelemetry/sdk-node
```

Next is initializing the Node SDK. You can do that either in a dedicated instrumentation file or directly at the top of your main file.


The inline setup is the simplest way to get started. It works well for projects where your main file is executed first and import order is straightforward.

We can now initialize the `LangfuseSpanProcessor` and start the SDK. The `LangfuseSpanProcessor` is the part that takes that collected data and sends it to your Langfuse project. 

Important: start the SDK before initializing the logic that needs to be traced to avoid losing data.

```ts
const sdk = new NodeSDK({
  spanProcessors: [new LangfuseSpanProcessor()],
});
 
sdk.start();
```


The instrumentation file often preferred when you're using frameworks that have complex startup order (Next.js, serverless, bundlers) or if you want a clean, predictable place where tracing is always initialized first.

Create an `instrumentation.ts` file, which sets up the _collector_ that gathers data about each OpenAI call. The `LangfuseSpanProcessor` is the part that takes that collected data and sends it to your Langfuse project.

```ts /LangfuseSpanProcessor/
const sdk = new NodeSDK({
  spanProcessors: [new LangfuseSpanProcessor()],
});

sdk.start();
```

Import the `instrumentation.ts` file first so all later imports run with tracing enabled.

```ts
// Must be the first import
```


**Instrument application**

Instrumentation means adding code that records what’s happening in your application so it can be sent to Langfuse. Here, OpenTelemetry acts as the system that collects those recordings.

```ts
// startActiveObservation creates a trace for this block of work.
// Everything inside automatically becomes part of that trace.
await startActiveObservation("user-request", async (span) => {
  span.update({
    input: { query: "What is the capital of France?" },
  });

  // This generation will automatically be a child of "user-request" because of the startObservation function.
  const generation = startObservation(
    "llm-call",
    {
      model: "gpt-4",
      input: [{ role: "user", content: "What is the capital of France?" }],
    },
    { asType: "generation" },
  );

  // ... your real LLM call would happen here ...

  generation
    .update({
      output: { content: "The capital of France is Paris." }, // update the output of the generation
    })
    .end(); // mark this nested observation as complete

  // Add final information about the overall request
  span.update({ output: "Successfully answered." });
});
```


}
    title="Full JS/TS SDK documentation"
    href="/docs/sdk/typescript/guide"
    arrow
  />
  }
    title="Notebook"
    href="/docs/sdk/typescript/example-notebook"
    arrow
  />

Explore all integrations and frameworks that Langfuse supports.


<img
          src="/images/integrations/vercel_ai_sdk_icon.png"
          />
      
}
    title="Vercel AI SDK"
    href="/integrations/frameworks/vercel-ai-sdk"
    arrow
  />
  <img
          src="/images/integrations/llamaindex_icon.png"
          />
      
}
    title="Llamaindex"
    href="/integrations/frameworks/llamaindex"
    arrow
  />
  <img
          src="/images/integrations/crewai_icon.svg"
          />
      
}
    title="CrewAI"
    href="/integrations/frameworks/crewai"
    arrow
  />
  <img
          src="/images/integrations/ollama_icon.svg"
          />
      
}
    title="Ollama"
    href="/integrations/model-providers/ollama"
    arrow
  />
  <img
          src="/images/integrations/litellm_icon.png"
          />
      
}
    title="LiteLLM"
    href="/integrations/gateways/litellm"
    arrow
  />
  <img
          src="/images/integrations/autogen_icon.svg"
          />
      
}
    title="AutoGen"
    href="/integrations/frameworks/autogen"
    arrow
  />
  <img
          src="/images/integrations/google_adk_icon.png"
          />
      
}
    title="Google ADK"
    href="/integrations/frameworks/google-adk"
    arrow
  />
  ### See your trace in Langfuse

After running your application, visit the Langfuse interface to view the trace you just created. _[(Example LangGraph trace in Langfuse)](https://cloud.langfuse.com/project/cloramnkj0002jz088vzn1ja4/traces/7d5f970573b8214d1ca891251e42282c)_

_[What does a good trace look like?](/faq/all/what-does-a-good-trace-look-like)_


## Not seeing what you expected?

## Next steps

Now that you've ingested your first trace, you can start adding on more functionality to your traces. We recommend starting with the following:

- [Group traces into sessions for multi-turn applications](/docs/observability/features/sessions)
- [Split traces into environments for different stages of your application](/docs/observability/features/environments)
- [Add attributes to your traces so you can filter them in the future](/docs/observability/features/tags)

Already know what you want? Take a look under _Features_ for guides on specific topics.
