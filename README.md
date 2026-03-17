# Convex Skills

AI agent skills for [Convex](https://convex.dev) development, following the [Agent Skills](https://skills.sh) open format.

Existing Convex skill sets were outdated, incomplete, or scattered. These skills merge official and community sources, fill the gaps, and verify everything against official documentation and source code.

## Install

```bash
# Install all skills
npx skills add roynulrohan/convex-skills --all

# Or pick what you need
npx skills add roynulrohan/convex-skills -s convex
npx skills add roynulrohan/convex-skills -s convex-components
npx skills add roynulrohan/convex-skills -s convex-helpers
npx skills add roynulrohan/convex-skills -s convex-agents
npx skills add roynulrohan/convex-skills -s convex-security
```

### Manual Installation

Clone the repo and copy the skills you need into your project's `.claude/skills/` directory:

```bash
git clone https://github.com/roynulrohan/convex-skills.git
cp -r convex-skills/skills/convex .claude/skills/
```

## Available Skills

| Skill                                          | Description                                                                                                                                 |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| [convex](skills/convex/)                       | Core backend & client — functions, schema, database, auth, scheduling, file storage, search, React client patterns, Next.js, TanStack Query |
| [convex-components](skills/convex-components/) | Official `@convex-dev/*` component ecosystem — rate limiting, aggregates, workflows, payments, presence, collaborative editing, and more    |
| [convex-helpers](skills/convex-helpers/)       | Official utility library — custom function wrappers, relationship helpers, triggers, Zod validation, CRUD generation, sessions              |
| [convex-agents](skills/convex-agents/)         | AI features with `@convex-dev/agent` — tool calling, threads, streaming, structured output, RAG, persistent text streaming                  |
| [convex-security](skills/convex-security/)     | Security patterns — auth checks, RBAC, row-level access, rate limiting, audit trails, argument validation, environment variables            |

## Which Skills Do I Need?

Start with **`convex`** — the other skills build on it.

For most projects, add **`convex-components`**. You'll reach for components sooner than you expect.

For larger apps or teams, add **`convex-helpers`**. Custom function middleware, triggers, relationship loading, and typed validators pay off as your codebase grows.

Those three cover the vast majority of Convex development. The remaining two serve specific needs:

- **`convex-security`** — for implementing or auditing authentication, access control, rate limiting, and other security patterns
- **`convex-agents`** — for AI features: chatbots, knowledge base search, document Q&A, AI-assisted workflows, or any LLM-powered functionality

## Usage

Skills activate automatically once installed. When your agent detects a relevant task, the corresponding skill provides context and patterns.

**Example prompts:**

```
Create a Convex schema with users, posts, and comments
Set up file uploads with image optimization
Add Stripe subscription billing
Build a real-time collaborative editor
Implement rate limiting on my API endpoints
Add RAG-powered document search
Audit my Convex functions for security issues
```
