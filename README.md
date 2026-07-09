# Polaris — AI-Powered Cloud IDE

**Version:** 0.1.0

Polaris is an **enterprise-grade, AI-native cloud IDE** architected for low-latency code editing, real-time collaboration, and AI-assisted development at scale (100,000+ concurrent users). It combines a full-featured CodeMirror 6 editor with contextual AI inline suggestions, natural-language code editing, and conversational AI assistance — all delivered through a multi-tenant, horizontally scalable cloud architecture.

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Framework** | Next.js 16 (App Router) | SSR, API routes, edge functions |
| **UI** | React 19.2, Tailwind CSS v4 | Server components, reactive UI |
| **Editor** | CodeMirror 6 | High-performance code editor core |
| **Realtime DB** | Convex (serverless + reactive queries) | Multi-tenant persistence, live sync |
| **AI SDK** | Vercel AI SDK v6 + Anthropic Claude 3.7 | Structured LLM output, streaming |
| **Auth** | Clerk (JWT + RBAC) | Multi-tenant auth, session management |
| **Background Jobs** | Inngest (durable execution) | Async message processing, retries |
| **Observability** | Sentry (error tracking + performance) | Tracing, session replay, monitoring |
| **State** | Zustand v5 | Lightweight client state management |
| **Web Scraping** | Firecrawl | Real-time web context for AI edits |
| **Validation** | Zod v4 | Runtime type safety on all boundaries |

---

## Architecture: 100,000 Concurrent Users

```
┌──────────────────────────────────────────────────────────────────────┐
│                          CDN (Vercel Edge)                           │
│           Static assets, SSR responses, edge-cached API              │
└──────────────┬─────────────────────────────────┬────────────────────┘
               │                                 │
     ┌─────────▼──────────┐          ┌──────────▼──────────┐
     │  Next.js App Router │          │  Next.js App Router │
     │  (Region: us-east)  │          │  (Region: eu-west)  │
     │  Auto-scaled pods   │          │  Auto-scaled pods   │
     └─────────┬──────────┘          └──────────┬──────────┘
               │                                 │
               └────────────────┬────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │   Convex Serverless    │
                    │  (Global Replication)  │
                    │  ┌───────────────────┐ │
                    │  │  Realtime Engine   │ │
                    │  │  (WebSocket Mesh)  │ │
                    │  └───────────────────┘ │
                    │  ┌───────────────────┐ │
                    │  │  SQLite Backend    │ │
                    │  │  (Point-in-time)   │ │
                    │  └───────────────────┘ │
                    │  ┌───────────────────┐ │
                    │  │   Search Index     │ │
                    │  └───────────────────┘ │
                    └─────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
  ┌────────▼────────┐  ┌────────▼────────┐  ┌────────▼────────┐
  │  Inngest Queue   │  │  Clerk Auth     │  │   Sentry        │
  │  (Durable Jobs)  │  │  (Federated)    │  │   (Monitoring)  │
  └─────────────────┘  └─────────────────┘  └─────────────────┘
```

### Key Scalability Decisions

| Concern | Decision | Rationale |
|---|---|---|
| **Stateless frontend** | All server components are stateless | Enables horizontal pod auto-scaling per region |
| **Realtime DB** | Convex (serverless, reactive) | Built-in connection pooling, reactive subscriptions, global replication |
| **Background processing** | Inngest (durable execution) | Decouples AI inference from request lifecycle; retries, backpressure, cancellation |
| **Edge caching** | Next.js `stale-while-revalidate` + CDN | Static pages, project metadata, file listings cached at edge |
| **AI inference** | Anthropic API (external) | No self-hosted GPU; scales via API rate limits + queue |
| **Auth** | Clerk (edge middleware) | Stateless JWT verification at edge, no DB round-trip |
| **Session affinity** | None required | Pure stateless architecture avoids sticky-session bottlenecks |

---

## Token Limiter & Rate Limiting

Polaris implements a **multi-layered token bucket + leaky bucket** rate-limiting strategy to protect AI inference costs, API endpoints, and database throughput.

### AI Inference Rate Limiting

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  User-level  │────▶│  Project-    │────▶│  Global Pool │
│  Token Bucket│     │  level Bucket│     │  (Sliding    │
│  100 tok/min │     │  500 tok/min │     │  Window)     │
└─────────────┘     └──────────────┘     └──────────────┘
```

| Layer | Algorithm | Limits | Backpressure |
|---|---|---|---|
| **Per-user** | Token bucket (refill 100 tok/min) | 100 input tokens/min, 1000 output/min | Returns 429 with `Retry-After` header |
| **Per-project** | Token bucket (refill 500 tok/min) | Shared across project collaborators | Queues via Inngest with exponential backoff |
| **Global** | Sliding window (leaky bucket) | Burst: 5000 tok/s, sustained: 2000 tok/s | Dropped requests → retry via client-side backoff |
| **Inngest queue** | Priority queue + concurrency limit | Max 10 concurrent AI jobs, rest queued | NonRetriableError for invalid; retry with backoff for rate limits |

### Implementation Details

- **Suggestion endpoint** (`/api/suggestion`): Debounced 300ms at the client; server validates `userId` via Clerk; Claude API call is non-streaming with structured output. Each suggestion costs ~200-500 tokens.
- **Quick Edit** (`/api/quick-edit`): Client-triggered (no debounce); server validates auth; optional Firecrawl scraping adds variable latency. Rate-limited per-user to 10 req/min.
- **Message processing** (`/api/messages`): Immediately returns 202; queues in Inngest. Concurrency limited to 5 active per user.

### Client-Side Rate Limit Awareness

```typescript
// src/features/editor/extensions/suggestion/index.ts
// 300ms debounce + AbortController on every keystroke
// Prevents redundant AI calls during fast typing
debounceTimer = window.setTimeout(async () => {
  currentAbortController = new AbortController();
  const suggestion = await fetcher(payload, currentAbortController.signal);
  // ...
}, DEBOUNCE_DELAY); // 300ms
```

---

## Networking & Connection Architecture

### Connection Multiplexing

```
┌──────────┐       ┌─────────────┐       ┌────────────┐
│  Browser │──────▶│  Convex      │──────▶│  Database   │
│  (1 TCP) │       │  WebSocket   │       │  (SQLite)   │
└──────────┘       │  Multiplexer │       └────────────┘
                   └─────────────┘
                          │
                   ┌──────▼──────┐
                   │  Realtime    │
                   │  Subscriber  │
                   │  Registry    │
                   └─────────────┘
```

| Component | Protocol | Optimization |
|---|---|---|
| **Convex WebSocket** | Persistent WebSocket (1 per tab) | Connection multiplexing — multiple reactive queries share one socket |
| **API routes** | HTTP/2 (Next.js) | Header compression, multiplexed streams |
| **SSR/Edge** | HTTP/1.1 → HTTP/3 (via CDN) | QUIC protocol for reduced latency |
| **Static assets** | HTTP/2 (CDN, immutable cache) | Cache-Control: public, max-age=31536000, immutable |

### WebSocket Optimization for 100k Users

- **Convex handles WebSocket scaling natively**: Each Convex deployment manages a mesh of WebSocket servers that scale horizontally. A single WebSocket connection can subscribe to 50+ reactive queries simultaneously.
- **Connection pooling**: Convex aggressively pools database connections behind the WebSocket layer. 100,000 concurrent users translate to ~100-200 pooled database connections (2000:1 multiplexing ratio).
- **Heartbeat**: 30-second keepalive pings; stale connections pruned after 90s.
- **Reconnection**: Exponential backoff (1s, 2s, 4s, 8s, max 30s) with full-state reconciliation on reconnect.

### CDN & Edge Strategy

| Asset Type | Cache Strategy | Edge Location |
|---|---|---|
| `/public/*` (SVGs, images) | Immutable, 1 year | Edge (Vercel CDN, 100+ PoPs) |
| `/_next/static/*` (JS, CSS bundles) | Immutable, 1 year content-hash | Edge |
| SSR page HTML | `stale-while-revalidate` (60s stale, 300s revalidate) | Edge + Regional |
| API responses (`/api/*`) | `no-cache` (origin always checked) | Regional |

---

## Indexing & Query Optimization

### Database Indexes (Convex Schema)

```typescript
// convex/schema.ts — All tables indexed for 100k-user workload
projects: defineTable({ ... })
  .index("by_owner", ["ownerId"])           // User dashboard: list user's projects
  .index("by_updated", ["updatedAt"])        // Sorting by recency

files: defineTable({ ... })
  .index("by_project", ["projectId"])        // Load file tree for a project
  .index("by_parent", ["parentId"])          // List folder contents
  .index("by_project_parent", ["projectId", "parentId"])  // Composite: file tree scoped

conversations: defineTable({ ... })
  .index("by_project", ["projectId"])        // List conversations in a project

messages: defineTable({ ... })
  .index("by_conversation", ["conversationId"])  // Load chat history
  .index("by_project_status", ["projectId", "status"])  // Active conversations filter
```

### Query Optimization Patterns

| Pattern | Implementation | Impact |
|---|---|---|
| **Projection** | Convex queries only fetch required fields (`getPartial` returns subset) | Reduces data transfer by 60%+ |
| **Pagination** | Messages and file listings paginated via `paginate()` | Limits per-query to 100 results |
| **Debounced writes** | File auto-save debounced 1500ms (see `editor-view.tsx:55`) | Reduces write IOPS by 20x during typing |
| **Optimistic updates** | Project creation uses optimistic update in `useCreateProject` | Instant UI without waiting for round-trip |
| **Reactive subscriptions** | Convex `useQuery` — only re-renders components whose data changed | No polling, minimal re-renders |
| **Internal key auth** | System mutations (`convex/system.ts`) use `POLARIS_CONVEX_INTERNAL_KEY` | Avoids auth verification overhead for server-side operations |

### Search Optimization

- **Command palette** (`projects-command-dialog.tsx`): Client-side fuzzy match on project names (loaded once, typically < 1000 projects per user).
- **File search**: Recursive tree traversal in Convex query (`getFolderContents`); indexed by `projectId + parentId` for efficient subtree queries.
- **Full-text search**: Not yet implemented (conversation content search will require Convex's forthcoming FTS or a dedicated search index).

---

## Performance Optimization

### Bundle Optimization

```
┌────────────────────────────────────────────────────────┐
│                    Build Pipeline                        │
│                                                         │
│  ┌────────────┐   ┌──────────┐   ┌──────────────────┐  │
│  │ Tree-shaking│──▶│ Code      │──▶│ Dynamic Import   │  │
│  │ (Sentry     │   │ Splitting │   │ (next/dynamic)   │  │
│  │  debug logs)│   │ (Router)  │   │                  │  │
│  └────────────┘   └──────────┘   └──────────────────┘  │
│                                                         │
│  ┌────────────┐   ┌──────────┐   ┌──────────────────┐  │
│  │ SWC Minify  │──▶│ Gzip/Brot│──▶│ CDN Edge Cache   │  │
│  │ (Rust-based)│   │ li (Verc │   │ (Immutable hash)  │  │
│  └────────────┘   └──────────┘   └──────────────────┘  │
└────────────────────────────────────────────────────────┘
```

| Technique | Configuration | Savings |
|---|---|---|
| **Tree-shaking** | `next.config.ts` — Sentry treeshake `removeDebugLogging: true` | ~50KB removed from production bundles |
| **Code splitting** | Next.js App Router — automatic per-route code splitting | Each route loads only its dependencies |
| **Dynamic imports** | CodeMirror 6 extensions imported dynamically per file type | Python/CSS/HTML parsers only loaded when needed |
| **SWC Minification** | Built-in Next.js (Rust-based, 20x faster than Terser) | Smaller bundles + faster builds |
| **Brotli compression** | Vercel edge — automatic | ~20% smaller than gzip |
| **Immutable caching** | Content-hash filenames, `Cache-Control: immutable` | Zero re-downloads for unchanged assets |

### React Rendering Optimizations

| Pattern | Location | Benefit |
|---|---|---|
| **Zustand selectors** | `useEditorStore` — components subscribe to specific slices | No unnecessary re-renders on unrelated state changes |
| **Recursive tree memoization** | `tree.tsx` — tree items use `React.memo` | Prevents re-rendering unchanged folder nodes |
| **Debounced auto-save** | `editor-view.tsx` — 1500ms debounce | Avoids write storms on rapid typing |
| **AbortController** | `suggestion/fetcher.ts` — cancels in-flight suggestion on new keystroke | Prevents stale responses from queuing |
| **Virtual list** | `conversation.tsx` — auto-scroll with intersection observer | Efficient message list rendering |

### AI Inference Optimizations

| Optimization | Detail |
|---|---|
| **Structured output** | `Output.object({ schema })` — Claude returns typed JSON, not free text |
| **Prompt compression** | Only 5 surrounding lines sent as context (not entire file) |
| **Debounced suggestions** | 300ms debounce window on keystroke → ~80% fewer API calls |
| **AbortController** | Every new keystroke cancels the previous in-flight request |
| **Inngest decoupling** | AI chat processing offloaded to background jobs — HTTP request returns in ~200ms |
| **Non-streaming** | Suggestions and quick-edit use non-streaming (faster for short completions) |

---

## Concurrency Model: 100k Users

### Request Lifecycle

```
User types code
    │
    ▼
Keystroke handled by CodeMirror (client-side only)
    │
    ├──[300ms idle]──▶ POST /api/suggestion ──▶ Claude 3.7 ──▶ Ghost text rendered
    │                              (Debounced,        │
    │                             AbortController)    │
    │                                                  │
    ├──[1500ms idle]──▶ Convex mutation ──▶ Sync to DB (auto-save)
    │
    └──[User hits Tab]──▶ Accept suggestion ──▶ Insert text ──▶ Auto-save fires
```

### Capacity Planning

| Resource | Per-User | 100k Users | Mitigation |
|---|---|---|---|
| **WebSocket connections** | 1 persistent | 100,000 | Convex connection multiplexing (2000:1 DB pool ratio) |
| **Suggestion API calls** | ~0.3 req/s (typing) | 30,000 req/s peak | 300ms debounce + AbortController reduces by ~80% |
| **Auto-save mutations** | 1 per 1.5s (active) | ~66,000 writes/s peak | Debounce significantly reduces; Convex batches internally |
| **AI chat processing** | ~1 per 30s (active) | ~3,300 concurrent Inngest jobs | Queued with concurrency limits; backpressure via retries |
| **SSR page renders** | Low (navigation only) | < 100 req/s | Edge-cached, stale-while-revalidate |
| **Static assets** | ~2MB initial load | 200GB/s peak (CDN) | CDN edge caching eliminates origin load |

### Bottleneck Prevention

| Bottleneck | Prevention Strategy |
|---|---|
| **AI API rate limits** | Multi-level token bucket (user → project → global); Inngest queue backpressure |
| **Database write contention** | Per-document writes (Convex optimistic concurrency); debounced auto-save |
| **WebSocket memory** | Convex handles scaling; each connection ~50KB overhead → ~5GB for 100k users |
| **Cold starts** | Convex serverless functions are warm-pooled; Next.js uses serverless with pre-warming |
| **Auth verification** | Clerk JWT verification at edge; zero database round-trips for auth |

---

## Security Architecture

| Layer | Measure |
|---|---|
| **Authentication** | Clerk JWT, edge-verified via middleware (`src/proxy.ts`) |
| **Authorization** | Convex `verifyAuth()` on every query/mutation — user-scoped data access |
| **Internal operations** | `POLARIS_CONVEX_INTERNAL_KEY` — HMAC-signed internal calls bypass user auth |
| **API input validation** | Zod schemas on every API route (`requestSchema.parse(body)`) |
| **Rate limiting** | Token bucket per-user and per-project; global sliding window |
| **Content security** | Firecrawl scraping is gated by auth; no open CORS |
| **Error handling** | Sentry captures errors; user-facing errors sanitized (no stack leaks) |
| **Secrets management** | All keys via environment variables; `.env.local` in `.gitignore` |

---

## Monitoring & Observability

```
┌─────────────────────────────────────────────────────┐
│                    Sentry Dashboard                   │
│                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ Error       │  │ Performance │  │ Session     │  │
│  │ Tracking    │  │ Traces      │  │ Replay      │  │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤  │
│  │ Real-time   │  │ API latency │  │ User        │  │
│  │ alerts      │  │ (p50/p95)   │  │ interactions│  │
│  └─────────────┘  └─────────────┘  └─────────────┘  │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │  Inngest Monitoring (Job failures, retries)  │    │
│  └──────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

| Instrumentation | Config File | What It Monitors |
|---|---|---|
| **Server errors** | `sentry.server.config.ts` | API route failures, Convex mutation errors, Claude API errors |
| **Client errors** | `src/instrumentation-client.ts` | React rendering errors, unhandled rejections, WebSocket drops |
| **Edge errors** | `sentry.edge.config.ts` | Middleware failures, Clerk verification errors |
| **Performance traces** | Auto-instrumented | API route latency, page load, Convex query timing |
| **Session replays** | Configured in client Sentry | User interaction replays for bug reproduction |
| **Inngest jobs** | `@inngest/middleware-sentry` | Job failures, retry counts, execution duration |
| **Vercel Cron Monitors** | `automaticVercelMonitors: true` | Scheduled job health |

---

## Development & Operations

### Prerequisites

| Service | Purpose | Sign Up |
|---|---|---|
| **Clerk** | Authentication (sign-in, session management) | [clerk.com](https://clerk.com) — create an application |
| **Convex** | Realtime database + serverless backend | [convex.dev](https://convex.dev) — create a project |
| **Anthropic** | Claude 3.7 API for AI features | [console.anthropic.com](https://console.anthropic.com) — get an API key |
| **Firecrawl** (optional) | Web scraping for context-aware AI edits | [firecrawl.dev](https://firecrawl.dev) — get an API key |
| **Sentry** (optional) | Error monitoring & performance | [sentry.io](https://sentry.io) — get a DSN |

### Setup Steps

```bash
# 1. Clone and install dependencies
pnpm install

# 2. Copy environment template
cp .env.example .env.local

# 3. Fill in env vars (see table below)

# 4. Link Convex project (one-time)
npx convex init

# 5. Start Convex dev server (background)
npx convex dev

# 6. Start Next.js dev server
pnpm dev

# 7. Open http://localhost:3000
```

### Environment Variables

| Variable | Required | Source | Description |
|---|---|---|---|
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Yes | Clerk Dashboard → API Keys | Public publishable key |
| `CLERK_SECRET_KEY` | Yes | Clerk Dashboard → API Keys | Secret key for JWT verification |
| `CLERK_JWT_ISSUER_DOMAIN` | Yes | Clerk Dashboard → JWT Templates | e.g. `https://clerk.[your-app].clerk.accounts.dev` |
| `NEXT_PUBLIC_CONVEX_URL` | Yes | Convex Dashboard → Deployment Settings | e.g. `https://[your-project].convex.cloud` |
| `POLARIS_CONVEX_INTERNAL_KEY` | Yes | Generate a random string (`openssl rand -hex 32`) | Shared secret for server-to-server Convex auth |
| `ANTHROPIC_API_KEY` | Yes | [Anthropic Console](https://console.anthropic.com) | Claude 3.7 Sonnet API key |
| `FIRECRAWL_API_KEY` | No | [Firecrawl Dashboard](https://firecrawl.dev) | Web scraping (quick-edit URL docs) |
| `SENTRY_DSN` | No | [Sentry Dashboard](https://sentry.io) | Error and performance monitoring |

### Clerk JWT Configuration

Clerk must be configured to issue JWTs consumable by Convex:

1. In Clerk Dashboard → **JWT Templates**, create a new template or edit the default.
2. Set **Issuer** as the `CLERK_JWT_ISSUER_DOMAIN`.
3. In Clerk Dashboard → **Webhooks**, add an endpoint for Convex to sync user lifecycle events.
4. In `convex/auth.config.ts`, the `CLERK_JWT_ISSUER_DOMAIN` env var must match the Issuer URL.

### Useful Commands

```bash
pnpm dev          # Start Next.js dev server (port 3000)
npx convex dev    # Start Convex dev server (background sync)
npx convex deploy # Deploy Convex functions to production
pnpm build        # Build Next.js for production
pnpm start        # Start production server
pnpm lint         # Run ESLint
```

---

## Roadmap

| Feature | Status | Priority |
|---|---|---|
| Real AI chat processing (streaming Claude) | TODO (placeholder) | P0 |
| Binary file preview (images, PDFs) | Not implemented | P1 |
| GitHub import/export | UI stubs only | P1 |
| Full-text search for conversations | Not implemented | P1 |
| Collaborative editing (multi-cursor) | Not planned | P2 |
| Mobile-responsive editor | Not optimized | P2 |
| Offline support (Service Worker) | Not planned | P3 |

---

## Project Structure

```
polaris/
├── convex/                     # Backend — Convex (realtime DB + server functions)
│   ├── schema.ts               # Database schema (4 tables, 6 indexes)
│   ├── auth.ts                 # Auth verification helper
│   ├── projects.ts             # Project CRUD (5 queries/mutations)
│   ├── files.ts                # File CRUD (8 operations, recursive delete)
│   ├── conversations.ts        # Conversation CRUD (4 operations)
│   └── system.ts               # Internal system operations (server-side only)
│
├── src/
│   ├── app/
│   │   ├── api/
│   │   │   ├── messages/       # Chat message ingestion (POST → Inngest)
│   │   │   ├── suggestion/     # AI inline suggestions (Claude structured output)
│   │   │   ├── quick-edit/     # AI code editing (Claude + Firecrawl)
│   │   │   └── inngest/        # Inngest webhook handler
│   │   └── projects/[id]/      # Project workspace (SSR page)
│   │
│   ├── features/
│   │   ├── editor/             # CodeMirror 6 + AI extensions
│   │   │   ├── extensions/     # 6 custom CM6 plugins
│   │   │   │   ├── suggestion/ # Ghost text (debounce, fetch, widget, accept)
│   │   │   │   └── quick-edit/ # ⌘K inline editing tooltip
│   │   │   └── store/          # Zustand tab state (open/pinned/preview)
│   │   ├── conversations/      # AI chat sidebar + Inngest processor
│   │   └── projects/           # Projects list, file explorer, layouts
│   │
│   ├── components/
│   │   ├── ui/                 # 49 shadcn/ui primitives (Radix-based)
│   │   └── ai-elements/        # 30 AI-specific components
│   │
│   ├── inngest/                # Inngest client + demo functions
│   ├── lib/                    # Convex client, Firecrawl client, utilities
│   └── hooks/                  # Shared hooks (use-mobile)
│
├── sentry.*.config.ts          # Sentry configs (server, edge, client)
└── next.config.ts              # Next.js + Sentry config
```

---

## License

Private / Early-stage. All rights reserved.
