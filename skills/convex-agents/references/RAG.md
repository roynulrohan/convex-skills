# RAG Component

Retrieval-Augmented Generation with vector search, document embeddings, and hybrid search. Integrates with any AI SDK embedding model.

## Installation

```bash
npm install @convex-dev/rag
```

```typescript
// convex/convex.config.ts
import rag from "@convex-dev/rag/convex.config.js";
app.use(rag);
```

Run `npx convex codegen` if `npx convex dev` isn't already running.

## Setup

```typescript
// convex/rag.ts
import { RAG } from "@convex-dev/rag";
import { components } from "./_generated/api";
import { openai } from "@ai-sdk/openai";

const rag = new RAG(components.rag, {
  textEmbeddingModel: openai.embedding("text-embedding-3-small"),
  embeddingDimension: 1536,
});
```

### With Typed Filters and Metadata

```typescript
type FilterTypes = {
  category: string;
  contentType: string;
  categoryAndType: { category: string; contentType: string };
};

type Metadata = {
  storageId: string;
  uploadedBy: string;
  uploadedAt: number;
};

const rag = new RAG<FilterTypes, Metadata>(components.rag, {
  textEmbeddingModel: openai.embedding("text-embedding-3-small"),
  embeddingDimension: 1536,
  filterNames: ["category", "contentType", "categoryAndType"],
});
```

Generic type parameters:

- `FilterTypes` — keys are filter names, values are filter value types. Provides type safety on `filterValues` and `filters`.
- `Metadata` — shape of the `metadata` field on entries.

## Adding Content

### Synchronous Add (with Auto-Chunking)

```typescript
const { entryId, status, created, replacedEntry, usage } = await rag.add(ctx, {
  namespace: "docs",
  text: "Your document content here...",
  key: "my-doc.txt", // Unique key — reusing replaces the old entry
  title: "My Document", // Used in default prompt formatting
  importance: 0.8, // 0-1 weight multiplied into search score (default: 1)
  contentHash: "abc123", // Skip if same hash exists for same key
  metadata: { storageId: "xyz", uploadedBy: "user_1", uploadedAt: Date.now() },
  filterValues: [
    { name: "category", value: "docs" },
    { name: "contentType", value: "markdown" },
  ],
  onComplete: internal.rag.docComplete,
});
// status: "ready" | "pending" | "replaced"
// usage: { tokens: number }
```

### With Custom Chunks

```typescript
// String array
await rag.add(ctx, {
    namespace: 'docs',
    chunks: content.split('\n\n'),
});

// With pre-computed embeddings
await rag.add(ctx, {
    namespace: 'docs',
    chunks: [
        { text: 'chunk text', embedding: [0.1, 0.2, ...], metadata: { page: 1 } },
    ],
});

// LangChain-style chunks
await rag.add(ctx, {
    namespace: 'docs',
    chunks: [
        { pageContent: 'chunk text', metadata: { source: 'file.pdf' } },
    ],
});
```

### Async Add (for Large Files)

```typescript
await rag.addAsync(ctx, {
  namespace: "all-files",
  chunkerAction: internal.rag.chunkerAction,
  metadata: { storageId },
  onComplete: internal.rag.docComplete,
});
```

## Searching

### Vector Search

```typescript
const { results, text, entries, usage } = await rag.search(ctx, {
  namespace: "docs",
  query: "How do I set up authentication?",
  limit: 10, // Max chunks (default: 10)
  searchType: "vector", // Default
});

// results: SearchResult[] — individual chunk matches with scores
// text: string — formatted string with "..." between chunks, "---" between entries
// entries: SearchEntry[] — entries with combined text
// usage: { tokens: number }
```

### Hybrid Search (Vector + Full-Text)

```typescript
const { results, text } = await rag.search(ctx, {
  namespace: "docs",
  query: "authentication setup",
  searchType: "hybrid",
  textWeight: 1, // Weight for text search (default: 1)
  vectorWeight: 1, // Weight for vector search (default: 1)
});
```

Uses Reciprocal Rank Fusion to combine results.

### Text-Only Search

```typescript
const { results } = await rag.search(ctx, {
  namespace: "docs",
  query: "authentication", // Must be string (no embedding computed)
  searchType: "text",
});
```

### Search Options

| Option                 | Type                                | Default                   | Description                                    |
| ---------------------- | ----------------------------------- | ------------------------- | ---------------------------------------------- |
| `limit`                | `number`                            | `10`                      | Max chunks returned (before context expansion) |
| `searchType`           | `"vector" \| "text" \| "hybrid"`    | `"vector"`                | Search mode                                    |
| `filters`              | `EntryFilter[]`                     | `[]`                      | OR'd filter values                             |
| `chunkContext`         | `{ before: number, after: number }` | `{ before: 0, after: 0 }` | Surrounding chunks per result                  |
| `vectorScoreThreshold` | `number`                            | none                      | Minimum score cutoff                           |
| `textWeight`           | `number`                            | `1`                       | Weight for text search in hybrid mode          |
| `vectorWeight`         | `number`                            | `1`                       | Weight for vector search in hybrid mode        |

### Filtering

Filters are **OR'd** in search. For AND semantics, use a composite filter value:

```typescript
// OR: matches category=news OR contentType=article
const results = await rag.search(ctx, {
  namespace: "docs",
  query: "latest updates",
  filters: [
    { name: "category", value: "news" },
    { name: "contentType", value: "article" },
  ],
});

// AND: matches category=news AND contentType=article
const results = await rag.search(ctx, {
  namespace: "docs",
  query: "latest updates",
  filters: [
    {
      name: "categoryAndType",
      value: { category: "news", contentType: "article" },
    },
  ],
});
```

### Context Expansion

Include surrounding chunks for better context:

```typescript
const results = await rag.search(ctx, {
  namespace: "docs",
  query: "auth setup",
  chunkContext: { before: 1, after: 2 }, // 1 chunk before, 2 after each match
});
```

Overlapping context ranges are deduplicated automatically.

## Generate Text (Search + LLM)

Search and generate in one call:

```typescript
import { openai } from "@ai-sdk/openai";

const result = await rag.generateText(ctx, {
  search: {
    namespace: "docs",
    query: "How do I set up auth?",
    limit: 5,
  },
  prompt: "How do I set up authentication?",
  model: openai("gpt-4o"),
});

// result.text — LLM response
// result.context.results — the search results used
// result.context.text — formatted context string
// result.context.entries — entry-level results
```

Default system prompt: `"You use the context provided only to produce a response. Do not preface the response with acknowledgement of the context."`

Context formatting adapts per model provider:

- **Anthropic/Google**: XML tags
- **OpenAI**: Markdown
- **Meta/Ollama**: Plain text

You can also pass `messages` for conversation history and all other AI SDK `generateText` options.

## Managing Entries

### List Entries

```typescript
// Paginated
const page = await rag.list(ctx, {
  namespaceId,
  order: "desc",
  status: "ready", // default: "ready"
  paginationOpts: { numItems: 25, cursor: null },
});

// Or with limit
const entries = await rag.list(ctx, {
  namespaceId,
  limit: 50,
});
```

### Get Single Entry

```typescript
const entry = await rag.getEntry(ctx, { entryId });
```

### Find by Content Hash

```typescript
const existing = await rag.findEntryByContentHash(ctx, {
  namespace: "docs",
  key: "my-file.txt",
  contentHash: "abc123",
});
```

### List Chunks

```typescript
const chunks = await rag.listChunks(ctx, {
  entryId,
  order: "asc",
  paginationOpts: { numItems: 50, cursor: null },
});
// Each chunk: { order, state, text, metadata? }
```

### Delete

```typescript
// Synchronous (use in actions)
await rag.delete(ctx, { entryId });

// Async/background (use in mutations)
await rag.deleteAsync(ctx, { entryId });

// By key (synchronous, use in actions)
await rag.deleteByKey(ctx, { namespaceId, key: "my-file.txt" });

// By key (async, use in mutations)
await rag.deleteByKeyAsync(ctx, { namespaceId, key: "my-file.txt" });
```

## Namespaces

```typescript
// Get or create
const { namespaceId, status } = await rag.getOrCreateNamespace(ctx, {
  namespace: "docs",
  status: "ready", // default: "ready"
  onComplete: internal.rag.namespaceReady,
});

// Get existing (returns null if not found)
const ns = await rag.getNamespace(ctx, { namespace: "docs" });
// Returns: { namespaceId, namespace, status, filterNames, dimension, modelId, version, createdAt }
```

## Lifecycle Hooks

### onComplete Handler

React to entry lifecycle events:

```typescript
export const docComplete = rag.defineOnComplete<DataModel>(
  async (ctx, { entry, replacedEntry, namespace, error }) => {
    if (error) {
      // Chunking/embedding failed — clean up
      await rag.deleteAsync(ctx, { entryId: entry.entryId });
      return;
    }
    if (replacedEntry) {
      // Old version replaced — clean up
      await rag.deleteAsync(ctx, { entryId: replacedEntry.entryId });
    }
  },
);
```

**Note:** `onComplete` is not called if `contentHash` matches an existing entry (no work done).

### Custom Chunker

```typescript
export const chunkerAction = rag.defineChunkerAction<DataModel>(
  async (ctx, args) => {
    const storageId = args.entry.metadata!.storageId;
    const file = await ctx.storage.get(storageId);
    const text = new TextDecoder().decode(await file!.arrayBuffer());
    return { chunks: text.split("\n\n") };
  },
);
```

Can also return an `AsyncIterable<InputChunk>` for streaming chunks.

## Utility Functions

### Default Chunker

```typescript
import { defaultChunker } from "@convex-dev/rag";

const chunks = defaultChunker(text, {
  minLines: 1, // default: 1
  minCharsSoftLimit: 100, // default: 100
  maxCharsSoftLimit: 1000, // default: 1000
  maxCharsHardLimit: 10000, // default: 10000
  delimiter: "\n\n", // default: "\n\n"
});
```

### Hybrid Rank (Reciprocal Rank Fusion)

```typescript
import { hybridRank } from "@convex-dev/rag";

const merged = hybridRank(
  [vectorResults, textResults],
  { weights: [1, 0.5], k: 10 }, // k: RRF constant, lower = more top-biased
);
```

### Content Hash

```typescript
import { contentHashFromArrayBuffer } from "@convex-dev/rag";

const hash = await contentHashFromArrayBuffer(await file.arrayBuffer());
```

### MIME Type Detection

```typescript
import {
  guessMimeTypeFromExtension,
  guessMimeTypeFromContents,
} from "@convex-dev/rag";

guessMimeTypeFromExtension("file.pdf"); // "application/pdf"
const mimeType = guessMimeTypeFromContents(await file.arrayBuffer());
```

## InputChunk Types

```typescript
type InputChunk =
  | string
  | {
      text: string;
      metadata?: Record<string, Value>;
      embedding?: number[];
      keywords?: string;
    } // Mastra
  | {
      pageContent: string;
      metadata?: Record<string, Value>;
      embedding?: number[];
      keywords?: string;
    }; // LangChain
```

The `keywords` field overrides what text is used for full-text search (defaults to chunk text).

## Exported Types and Validators

| Export            | Kind         | Description                          |
| ----------------- | ------------ | ------------------------------------ |
| `Entry`           | Type         | Entry metadata object                |
| `EntryId`         | Branded type | Entry identifier                     |
| `NamespaceId`     | Branded type | Namespace identifier                 |
| `SearchResult`    | Type         | Chunk result with score              |
| `SearchEntry`     | Type         | Entry + combined text                |
| `SearchType`      | Type         | `"vector" \| "text" \| "hybrid"`     |
| `Status`          | Type         | `"pending" \| "ready" \| "replaced"` |
| `Namespace`       | Type         | Full namespace object                |
| `Chunk`           | Type         | `{ order, state, text, metadata? }`  |
| `EntryFilter`     | Type         | `{ name, value }`                    |
| `InputChunk`      | Type         | String or Mastra/LangChain chunk     |
| `vEntry`          | Validator    | For args/schema                      |
| `vEntryId`        | Validator    | Branded string validator             |
| `vNamespaceId`    | Validator    | Branded string validator             |
| `vSearchEntry`    | Validator    | Search entry validator               |
| `vSearchResult`   | Validator    | Search result validator              |
| `vSearchType`     | Validator    | Search type validator                |
| `vOnCompleteArgs` | Validator    | onComplete args validator            |

## Key Behaviors

- **Content replacement**: Adding with the same `key` creates a new "pending" entry, swaps embeddings atomically, promotes to "ready", marks old as "replaced". Old content kept for in-flight searches until deleted.
- **Deduplication**: If `contentHash` matches existing entry for same key, the add is skipped (no `onComplete` fired).
- **Filter OR semantics**: Multiple filters are OR'd. Use composite filter values for AND.
- **Context formatting**: `generateText` auto-adapts format per model provider.
- **Chunk context dedup**: Overlapping `chunkContext` ranges don't produce duplicate chunks.

## Integration with Agent Component

Register a RAG search as an agent tool to give your agent access to your knowledge base. See the Agent skill for the tool integration pattern.
