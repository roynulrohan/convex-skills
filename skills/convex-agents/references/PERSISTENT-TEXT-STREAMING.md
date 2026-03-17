# Persistent Text Streaming Component

Stream AI-generated text to clients via HTTP while automatically persisting to your Convex database on sentence boundaries.

## Installation

```bash
npm install @convex-dev/persistent-text-streaming
```

```typescript
// convex/convex.config.ts
import persistentTextStreaming from "@convex-dev/persistent-text-streaming/convex.config";
app.use(persistentTextStreaming);
```

## How It Works

```
┌──────────────┐     HTTP stream      ┌──────────────┐
│  Driving      │◄════════════════════│   Convex     │
│  Client       │     (real-time)     │   httpAction │──► AI Provider
└──────────────┘                      └──────────────┘
                                            │
                                   persists on sentence
                                        boundaries
                                            │
┌──────────────┐     reactive query   ┌─────▼────────┐
│  Other        │◄───────────────────│   Convex DB  │
│  Clients      │     (database)      └──────────────┘
└──────────────┘
```

- **Driving client** (the one that initiated the stream): gets real-time HTTP streaming
- **Other clients** (or same client after reload): read from database via reactive query
- Database writes happen on **sentence boundaries**, not per-token — reducing write frequency

## Backend Setup

```typescript
// convex/chat.ts
import { PersistentTextStreaming } from "@convex-dev/persistent-text-streaming";
import type { StreamId } from "@convex-dev/persistent-text-streaming";
import { components } from "./_generated/api";
import { mutation, query, httpAction } from "./_generated/server";

const streaming = new PersistentTextStreaming(
  components.persistentTextStreaming,
);
```

### Create a Stream

```typescript
import { StreamIdValidator } from "@convex-dev/persistent-text-streaming";

export const createStream = mutation({
  handler: async (ctx): Promise<StreamId> => {
    return await streaming.createStream(ctx);
  },
});
```

### Read Stream Body (Database Fallback)

```typescript
export const getStreamBody = query({
  args: { streamId: StreamIdValidator },
  handler: async (ctx, { streamId }) => {
    return await streaming.getStreamBody(ctx, streamId);
    // Returns: { text: string, status: StreamStatus }
  },
});
```

`StreamStatus`: `"pending"` | `"streaming"` | `"done"` | `"error"` | `"timeout"`

### Stream via HTTP Action

```typescript
export const streamChat = httpAction(async (ctx, request) => {
  const { streamId, prompt } = await request.json();

  const response = await streaming.stream(
    ctx,
    request,
    streamId as StreamId,
    async (ctx, request, streamId, appendChunk) => {
      const aiStream = await openai.chat.completions.create({
        model: "gpt-4o",
        messages: [
          { role: "system", content: "You are a helpful assistant." },
          { role: "user", content: prompt },
        ],
        stream: true,
      });

      for await (const chunk of aiStream) {
        const content = chunk.choices[0]?.delta?.content || "";
        await appendChunk(content);
      }
    },
  );

  // CORS headers required
  response.headers.set("Access-Control-Allow-Origin", "*");
  response.headers.set("Vary", "Origin");
  return response;
});
```

The `appendChunk` callback handles both HTTP streaming and database persistence. Works with any AI provider (OpenAI, Anthropic, etc.) — just iterate the stream and call `appendChunk()`.

### HTTP Route Setup

```typescript
// convex/http.ts
import { httpRouter } from "convex/server";
import { httpAction } from "./_generated/server";
import { streamChat } from "./chat";

const http = httpRouter();

http.route({
  path: "/chat-stream",
  method: "POST",
  handler: streamChat,
});

// CORS preflight
http.route({
  path: "/chat-stream",
  method: "OPTIONS",
  handler: httpAction(async (_, request) => {
    return new Response(null, {
      headers: {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Methods": "POST",
        "Access-Control-Allow-Headers": "Content-Type, Authorization",
        "Access-Control-Max-Age": "86400",
      },
    });
  }),
});

export default http;
```

## React Client

```typescript
import { useStream } from '@convex-dev/persistent-text-streaming/react';
import { api } from '../convex/_generated/api';

function ChatMessage({ streamId, isDriven }: { streamId: StreamId; isDriven: boolean }) {
    const convexSiteUrl = import.meta.env.VITE_CONVEX_SITE_URL;

    const { text, status } = useStream(
        api.chat.getStreamBody,                              // Database fallback query
        new URL(`${convexSiteUrl}/chat-stream`),             // HTTP streaming endpoint
        isDriven,                                            // true = this client initiated the stream
        streamId,                                            // Stream identifier
        { authToken: 'optional-auth-token' }                 // Optional config
    );

    return (
        <div>
            <p>{text}</p>
            {status === 'streaming' && <span className="cursor-blink" />}
            {status === 'error' && <span>Error occurred</span>}
        </div>
    );
}
```

### `useStream` Parameters

| Param          | Type            | Description                                                |
| -------------- | --------------- | ---------------------------------------------------------- |
| `queryRef`     | Query reference | Query that calls `getStreamBody` — database fallback       |
| `httpEndpoint` | `URL`           | Streaming HTTP endpoint URL                                |
| `isDriven`     | `boolean`       | `true` = use HTTP streaming, `false` = use database query  |
| `streamId`     | `StreamId`      | The stream identifier                                      |
| `options?`     | `object`        | `{ authToken?: string; headers?: Record<string, string> }` |

### `useStream` Return Value

| Field    | Type           | Description                                                          |
| -------- | -------------- | -------------------------------------------------------------------- |
| `text`   | `string`       | Current accumulated text                                             |
| `status` | `StreamStatus` | `"pending"` \| `"streaming"` \| `"done"` \| `"error"` \| `"timeout"` |

### Tracking Driven State

The key concept is `isDriven` — track which streams this browser session created:

```typescript
function Chat() {
    const [drivenStreams, setDrivenStreams] = useState<Set<string>>(new Set());

    const sendMessage = async () => {
        const streamId = await createStream();
        setDrivenStreams((prev) => new Set(prev).add(streamId));

        // Trigger the HTTP stream
        await fetch(`${convexSiteUrl}/chat-stream`, {
            method: 'POST',
            body: JSON.stringify({ streamId, prompt: userInput }),
        });
    };

    return (
        <div>
            {messages.map((msg) => (
                <ChatMessage
                    key={msg.streamId}
                    streamId={msg.streamId}
                    isDriven={drivenStreams.has(msg.streamId)}
                />
            ))}
        </div>
    );
}
```

## API Reference

### `PersistentTextStreaming` Methods

| Method                                       | Context    | Description                             |
| -------------------------------------------- | ---------- | --------------------------------------- |
| `createStream(ctx)`                          | Mutation   | Create a new stream, returns `StreamId` |
| `getStreamBody(ctx, streamId)`               | Query      | Read stream text + status from database |
| `stream(ctx, request, streamId, generateFn)` | httpAction | Stream text via HTTP + persist to DB    |

### Exports

| Export                    | From                                          | Description                 |
| ------------------------- | --------------------------------------------- | --------------------------- |
| `PersistentTextStreaming` | `@convex-dev/persistent-text-streaming`       | Main component class        |
| `StreamId`                | `@convex-dev/persistent-text-streaming`       | Branded string type         |
| `StreamIdValidator`       | `@convex-dev/persistent-text-streaming`       | Convex validator for `args` |
| `useStream`               | `@convex-dev/persistent-text-streaming/react` | React hook                  |

## Design Notes

- HTTP stream sends tokens as they arrive (low latency for the driving client)
- Database updates happen on sentence boundaries (not per-token), reducing write frequency
- On disconnect, the database has content up to the last sentence boundary
- Non-driving clients read from the reactive database query which updates as new content is persisted
- Multiple concurrent streams are supported via unique `StreamId` identifiers
