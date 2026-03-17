---
name: convex-agents
description: "Building AI agents with @convex-dev/agent: Agent class, generateText, streamText, createTool with Zod, thread management, RAG, persistent text streaming, and workflow orchestration. Use when building chatbots, AI assistants, tool-calling agents, conversational AI, vector search, RAG, AI text streaming, or any LLM-powered features on Convex. Triggers on: @convex-dev/agent, @convex-dev/rag, @convex-dev/persistent-text-streaming, Agent class, generateText, streamText, createTool, agent threads, AI tools, RAG with Convex, vector search for AI, embedding generation, conversational history."
---

# Convex Agents

Build persistent, stateful AI agents with Convex using the `@convex-dev/agent` component.

## Setup

```bash
npm install @convex-dev/agent ai zod @ai-sdk/openai
```

The agent uses AI SDK, so any provider works — `@ai-sdk/openai`, `@ai-sdk/anthropic`, `@ai-sdk/google`, etc. Install whichever you need.

Register the component:

```typescript
// convex/convex.config.ts
import { defineApp } from "convex/server";
import agent from "@convex-dev/agent/convex.config";

const app = defineApp();
app.use(agent);

export default app;
```

The agent component manages its own internal tables for threads and messages — do NOT define `threads` or `messages` tables in your schema.

## Defining an Agent

Agents use AI SDK providers (not raw OpenAI client). The constructor takes `components.agent` and a config object:

```typescript
// convex/agents/support.ts
import { Agent, createTool } from "@convex-dev/agent";
import { stepCountIs } from "ai";
import { components } from "./_generated/api";
import { openai } from "@ai-sdk/openai";
import { z } from "zod/v3";

export const supportAgent = new Agent(components.agent, {
  name: "Support Agent",
  languageModel: openai.chat("gpt-4o-mini"),
  textEmbeddingModel: openai.embedding("text-embedding-3-small"),
  instructions:
    "You are a helpful customer support agent. Be concise and friendly.",
  tools: {
    searchKnowledgeBase: createTool({
      description: "Search the knowledge base for relevant articles",
      args: z.object({
        query: z.string().describe("The search query"),
      }),
      handler: async (ctx, { query }): Promise<string[]> => {
        const results = await ctx.runQuery(api.knowledge.search, { query });
        return results.map((r) => r.content);
      },
    }),
  },
  // Required if you want tool calls to happen automatically (default is 1)
  stopWhen: stepCountIs(5),
  callSettings: { maxRetries: 3, temperature: 0.7 },
});
```

### Key constructor options

| Option               | Type             | Purpose                                                                                       |
| -------------------- | ---------------- | --------------------------------------------------------------------------------------------- |
| `name`               | string           | Agent identifier                                                                              |
| `languageModel`      | AI SDK model     | e.g., `openai.chat("gpt-4o-mini")`                                                            |
| `textEmbeddingModel` | AI SDK embedding | For RAG vector search                                                                         |
| `instructions`       | string           | System prompt                                                                                 |
| `tools`              | object           | Tool definitions (keyed by name)                                                              |
| `stopWhen`           | function         | e.g., `stepCountIs(5)` from `"ai"` — for multi-step tool use                                  |
| `contextOptions`     | object           | Configure context message fetching (see below)                                                |
| `storageOptions`     | object           | Configure message storage (see below)                                                         |
| `usageHandler`       | async fn         | Track token usage: `(ctx, { usage, model, provider, agentName, threadId, userId }) => {}`     |
| `contextHandler`     | async fn         | Filter/enrich context messages before LLM call                                                |
| `rawResponseHandler` | async fn         | Log every request/response: `(ctx, { request, response, agentName, threadId, userId }) => {}` |
| `callSettings`       | object           | `{ maxRetries, temperature }` etc.                                                            |

## Thread Management

Threads are created via standalone `createThread` or via `agent.createThread`. The component manages thread storage internally.

```typescript
// convex/threads.ts
import { createThread, listMessages, listUIMessages } from "@convex-dev/agent";
import { components } from "./_generated/api";
import { mutation, query } from "./_generated/server";
import { paginationOptsValidator } from "convex/server";
import { v } from "convex/values";
import { supportAgent } from "./agents/support";

// Create a thread in a mutation
export const createNewThread = mutation({
  args: { userId: v.string(), title: v.optional(v.string()) },
  handler: async (ctx, { userId, title }) => {
    const threadId = await createThread(ctx, components.agent, {
      userId,
      title: title ?? "New Conversation",
      summary: "Customer support conversation",
    });
    return threadId;
  },
});

// List messages (server-side)
export const getMessages = query({
  args: { threadId: v.string(), paginationOpts: paginationOptsValidator },
  handler: async (ctx, args) => {
    return await listMessages(ctx, components.agent, {
      threadId: args.threadId,
      excludeToolMessages: true,
      paginationOpts: args.paginationOpts,
    });
  },
});

// List messages formatted for UI rendering
export const getUIMessages = query({
  args: { threadId: v.string(), paginationOpts: paginationOptsValidator },
  handler: async (ctx, args) => {
    return await listUIMessages(ctx, components.agent, {
      threadId: args.threadId,
      paginationOpts: args.paginationOpts,
    });
  },
});
```

### Creating threads in actions (with immediate generation)

```typescript
export const startConversation = action({
  args: { userId: v.string(), prompt: v.string() },
  handler: async (ctx, { userId, prompt }) => {
    const { thread, threadId } = await supportAgent.createThread(ctx, {
      userId,
    });
    const result = await thread.generateText({ prompt });
    return { threadId, response: result.text };
  },
});
```

Note: `agent.createThread` returns `{ thread, threadId }` where `thread` has `generateText`, `streamText`, `generateObject`, and `streamObject` methods bound to it.

### Continuing an existing thread

For chat apps where users return to existing conversations, use `continueThread`:

```typescript
export const reply = action({
  args: { threadId: v.string(), prompt: v.string() },
  handler: async (ctx, { threadId, prompt }) => {
    const { thread } = await supportAgent.continueThread(ctx, { threadId });
    const result = await thread.generateText({ prompt });
    return result.text;

    // The thread object also has:
    // thread.streamText({ prompt, ... })
    // thread.generateObject({ prompt, schema })
    // thread.streamObject({ prompt, schema })
    // thread.getMetadata()
    // thread.updateMetadata({ patch: { title, summary, userId } })
  },
});
```

## Generating Responses

The agent provides `generateText`, `streamText`, and `generateObject`. All use a **3-argument signature**: `(ctx, threadOptions, generateOptions)`.

### generateText

```typescript
export const chat = action({
  args: { threadId: v.string(), message: v.string() },
  handler: async (ctx, { threadId, message }) => {
    const result = await supportAgent.generateText(
      ctx,
      { threadId },
      { prompt: message },
    );
    return result.text;
  },
});
```

### streamText

```typescript
export const streamChat = action({
  args: { threadId: v.string(), prompt: v.string() },
  handler: async (ctx, { threadId, prompt }) => {
    await supportAgent.streamText(
      ctx,
      { threadId },
      { prompt },
      { saveStreamDeltas: { chunking: "word", throttleMs: 100 } },
    );
  },
});
```

### Async streaming with optimistic updates

For real-time UI updates, use the mutation → scheduler → action pattern:

```typescript
import { syncStreams, vStreamArgs, listUIMessages } from "@convex-dev/agent";

// 1. Mutation saves user message and schedules streaming
export const initiateStreaming = mutation({
  args: { prompt: v.string(), threadId: v.string() },
  handler: async (ctx, { prompt, threadId }) => {
    const { messageId } = await supportAgent.saveMessage(ctx, {
      threadId,
      prompt,
      skipEmbeddings: true,
    });
    await ctx.scheduler.runAfter(0, internal.chat.streamAsync, {
      threadId,
      promptMessageId: messageId,
    });
    return messageId;
  },
});

// 2. Action does the actual streaming
export const streamAsync = internalAction({
  args: { promptMessageId: v.string(), threadId: v.string() },
  handler: async (ctx, { promptMessageId, threadId }) => {
    const result = await supportAgent.streamText(
      ctx,
      { threadId },
      { promptMessageId },
      { saveStreamDeltas: true },
    );
    await result.consumeStream();
  },
});

// 3. Query for messages with streaming support
export const listThreadMessages = query({
  args: {
    threadId: v.string(),
    paginationOpts: paginationOptsValidator,
    streamArgs: vStreamArgs,
  },
  handler: async (ctx, args) => {
    const streams = await syncStreams(ctx, components.agent, args);
    const paginated = await listUIMessages(ctx, components.agent, args);
    return { ...paginated, streams };
  },
});
```

### generateObject (structured output)

```typescript
import { z } from "zod/v3";

const result = await thread.generateObject({
  prompt: "Generate a plan based on the conversation so far",
  schema: z.object({
    steps: z.array(z.string()),
    estimatedTime: z.number(),
  }),
});
// result.object is typed from the schema
```

### streamObject (streaming structured output)

```typescript
const result = await thread.streamObject({
  prompt: "Generate a plan based on the conversation so far",
  schema: z.object({
    steps: z.array(z.string()),
    estimatedTime: z.number(),
  }),
});
// Partial object available as stream progresses
```

## Tools

Tools use `createTool` with **Zod schemas** (not Convex validators). The handler receives a context with `runQuery`, `runMutation`, `userId`, `threadId`, etc.

```typescript
import { createTool } from "@convex-dev/agent";
import { z } from "zod/v3";
import { api } from "../_generated/api";

export const searchProducts = createTool({
  description: "Search for products in the catalog",
  args: z.object({
    query: z.string().describe("Search query for products"),
    category: z.string().optional().describe("Optional category filter"),
    limit: z.number().default(10).describe("Maximum results to return"),
  }),
  handler: async (ctx, { query, category, limit }): Promise<Product[]> => {
    // ctx includes: runQuery, runMutation, userId, threadId, messageId
    return await ctx.runQuery(api.products.search, { query, category, limit });
  },
});

export const createOrder = createTool({
  description: "Create a new order for the customer",
  args: z.object({
    productIds: z.array(z.string()).describe("Product IDs to order"),
    shippingAddress: z.string().describe("Shipping address"),
  }),
  handler: async (ctx, { productIds, shippingAddress }): Promise<string> => {
    const orderId = await ctx.runMutation(api.orders.create, {
      userId: ctx.userId,
      productIds,
      shippingAddress,
    });
    return `Order ${orderId} created successfully`;
  },
});
```

Tools are registered on the Agent constructor via the `tools` object, not passed per-call:

```typescript
const agent = new Agent(components.agent, {
  languageModel: openai.chat("gpt-4o-mini"),
  tools: { searchProducts, createOrder },
  stopWhen: stepCountIs(5),
});
```

You can also use standard AI SDK `tool()` from `"ai"` alongside `createTool`.

### Agents as tools

Agents can call other agents by wrapping them as tools:

```typescript
const researchAgent = new Agent(components.agent, {
  name: "Research Agent",
  languageModel: openai.chat("gpt-4o"),
  instructions: "You are a research specialist.",
});

const askResearcher = createTool({
  description: "Ask the research agent a detailed question",
  args: z.object({
    question: z.string().describe("The research question"),
  }),
  handler: async (ctx, { question }, options): Promise<string> => {
    const { thread } = await researchAgent.createThread(ctx, {
      userId: ctx.userId,
    });
    const result = await thread.generateText(
      { prompt: [...options.messages, { role: "user", content: question }] },
      { storageOptions: { saveMessages: "all" } },
    );
    return result.text;
  },
});

const orchestrator = new Agent(components.agent, {
  name: "Orchestrator",
  languageModel: openai.chat("gpt-4o-mini"),
  instructions: "Coordinate between specialists.",
  tools: { askResearcher },
  stopWhen: stepCountIs(5),
});
```

## RAG (Retrieval Augmented Generation)

RAG uses the agent's `textEmbeddingModel` for vector search. Search context manually and inject into prompts:

```typescript
const context = await rag.search(ctx, {
  namespace: "global",
  query: userPrompt,
  limit: 10,
});

const result = await agent.generateText(
  ctx,
  { threadId },
  {
    prompt: `# Context:\n\n${context.text}\n\n---\n\n# Question:\n\n${userPrompt}`,
  },
);
```

Or configure `contextOptions` on the agent for automatic context injection.

### HTTP action streaming

For HTTP endpoints that stream responses directly to clients:

```typescript
export const streamHTTP = httpAction(async (ctx, req) => {
  const { prompt, threadId } = await req.json();
  const result = await supportAgent.streamText(
    ctx,
    { threadId },
    { prompt },
    { saveStreamDeltas: { returnImmediately: true } },
  );
  return result.toUIMessageStreamResponse();
});
```

The `returnImmediately` option returns before the stream is fully saved, and `toUIMessageStreamResponse()` converts the stream to an HTTP `Response` object.

## Shared Agent Defaults

Use the `Config` type to share settings across multiple agents:

```typescript
import { Agent, createTool, type Config } from "@convex-dev/agent";
import { openai } from "@ai-sdk/openai";

const sharedDefaults = {
  languageModel: openai.chat("gpt-4o-mini"),
  textEmbeddingModel: openai.embedding("text-embedding-3-small"),
  usageHandler: async (ctx, args) => {
    const { usage, model, provider, agentName, threadId, userId } = args;
    await ctx.runMutation(internal.usage.log, { ...usage, agentName, userId });
  },
  callSettings: { maxRetries: 3, temperature: 1.0 },
} satisfies Config;

const supportAgent = new Agent(components.agent, {
  instructions: "You are a support agent.",
  ...sharedDefaults,
});

const salesAgent = new Agent(components.agent, {
  instructions: "You are a sales agent.",
  ...sharedDefaults,
});
```

## Context & Storage Options

These can be set as defaults on the Agent constructor OR overridden per-call as the 4th argument to `generateText`/`streamText`.

### contextOptions

Controls what messages the LLM sees:

```typescript
const result = await agent.generateText(
  ctx,
  { threadId },
  { prompt },
  {
    contextOptions: {
      excludeToolMessages: true, // default: true
      recentMessages: 100, // recent messages to include (after search)
      searchOptions: {
        limit: 10, // max messages from search
        textSearch: false, // use text search
        vectorSearch: false, // use vector search (requires textEmbeddingModel)
        messageRange: { before: 2, after: 1 }, // surrounding messages per search hit
      },
      searchOtherThreads: false, // search across user's other threads
    },
  },
);
```

### contextHandler (advanced)

Override how context is assembled — useful for injecting memories, few-shot examples, or cross-thread context:

```typescript
const result = await agent.generateText(
  ctx,
  { threadId },
  { prompt },
  {
    contextHandler: async (ctx, args) => {
      const memories = await ctx.runQuery(api.memories.getForUser, {
        userId: args.userId,
      });
      const examples = [
        { role: "user" as const, content: "How do I reset my password?" },
        {
          role: "assistant" as const,
          content: "Go to Settings > Security > Reset Password.",
        },
      ];
      return [
        ...memories.map((m) => ({
          role: "system" as const,
          content: m.content,
        })),
        ...examples,
        ...args.search,
        ...args.recent,
        ...args.inputMessages,
        ...args.inputPrompt,
        ...args.existingResponses,
      ];
    },
  },
);
```

The `args` object in `contextHandler` provides: `search`, `recent`, `inputMessages`, `inputPrompt`, `existingResponses`, `userId`, `threadId`, `allMessages`.

### storageOptions

Controls which messages get saved to the database:

```typescript
const result = await thread.generateText(
  { prompt },
  {
    storageOptions: {
      saveMessages: "all" | "none" | "promptAndOutput",
    },
  },
);
```

### saveMessage (pre-save prompts)

Useful for optimistic UI — save the user message before generation starts:

```typescript
const { messageId } = await agent.saveMessage(ctx, {
  threadId,
  prompt: "user's message",
  skipEmbeddings: true,
});
// Then pass promptMessageId to generateText/streamText
const result = await agent.generateText(
  ctx,
  { threadId },
  { promptMessageId: messageId },
);
```

## React Hooks

The `@convex-dev/agent/react` module provides hooks for building chat UIs with streaming support.

### useUIMessages

Paginated message list with real-time streaming:

```typescript
import { useUIMessages, useSmoothText, optimisticallySendMessage } from "@convex-dev/agent/react";
import { useMutation } from "convex/react";
import type { UIMessage } from "@convex-dev/agent/react";

function ChatMessages({ threadId }: { threadId: string }) {
  const { results, status, loadMore } = useUIMessages(
    api.chat.streaming.listThreadMessages,
    { threadId },
    { initialNumItems: 20, stream: true },
  );

  return (
    <div>
      {results.map((message) => (
        <Message key={message.key} message={message} />
      ))}
      {status === "CanLoadMore" && (
        <button onClick={() => loadMore(10)}>Load more</button>
      )}
    </div>
  );
}
```

### useSmoothText

Smooth character-by-character rendering for streaming messages:

```typescript
import { useSmoothText } from "@convex-dev/agent/react";

function Message({ message }: { message: UIMessage }) {
  const [visibleText] = useSmoothText(message.text, {
    startStreaming: message.status === "streaming",
  });

  return (
    <div className={message.role === "user" ? "user-message" : "assistant-message"}>
      <span>{message.agentName ?? message.role}</span>
      <p>{visibleText}</p>
      {message.status === "streaming" && <span>Typing...</span>}
    </div>
  );
}
```

### Optimistic updates for sending messages

```typescript
function ChatInput({ threadId }: { threadId: string }) {
  const sendMessage = useMutation(api.chat.streaming.initiateStreaming)
    .withOptimisticUpdate(
      optimisticallySendMessage(api.chat.streaming.listThreadMessages),
    );

  const handleSubmit = async (prompt: string) => {
    await sendMessage({ threadId, prompt });
  };

  return <form onSubmit={(e) => { e.preventDefault(); handleSubmit(input); }}>...</form>;
}
```

## Critical Rules

- Use AI SDK providers (`@ai-sdk/openai`, `@ai-sdk/anthropic`, etc.), NOT the raw `openai` package
- Tools use **Zod schemas** (`z.object`), NOT Convex validators (`v.object`)
- Import `zod` as `"zod/v3"` (not `"zod"`)
- Import `stepCountIs` from `"ai"`, NOT from `"@convex-dev/agent"`
- The agent component manages its own tables — do NOT define `threads`/`messages` in your schema
- `generateText`/`streamText` on the agent use 3+ args: `(ctx, threadOptions, generateOptions, overrides?)`
- On a `thread` object (from `agent.createThread` or `agent.continueThread`), they use 1-2 args: `(generateOptions, overrides?)`
- Thread IDs are strings, not `v.id("threads")`
- Set `stopWhen: stepCountIs(N)` if you want multi-step tool calling (default is 1 step)
- Annotate tool handler return types to avoid TypeScript type cycles

## Best Practices

- Use streaming for better UX with long responses
- Use `saveStreamDeltas` for multi-client real-time sync
- Store conversation history via the component (it handles this automatically)
- Rate limit agent interactions to control costs
- Use `excludeToolMessages: true` when displaying messages in UI
- Log tool usage via `usageHandler` for cost tracking

## Related AI Components

These components are commonly used alongside agents:

- **[RAG](references/RAG.md)** — Retrieval-augmented generation with vector search, document embeddings, and hybrid search (`@convex-dev/rag`)
- **[Persistent Text Streaming](references/PERSISTENT-TEXT-STREAMING.md)** — Stream AI text via HTTP while persisting to database on sentence boundaries (`@convex-dev/persistent-text-streaming`)

## References

- Convex AI docs: https://docs.convex.dev/ai
- Agent Component: https://www.npmjs.com/package/@convex-dev/agent
- AI SDK: https://sdk.vercel.ai/docs
