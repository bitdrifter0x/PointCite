# PointCite

A multi-tenant **document search and question-answering platform**. You upload a stack of PDFs, Word files, and notes. You ask a question in plain English. PointCite finds the most relevant passages, hands them to a language model, and returns an answer with numbered citations, so you can check every claim against the page it came from.

Uploads are processed in the background by a separate worker: text is extracted, split into chunks, embedded, and stored in Postgres with pgvector. Search combines keyword matching and vector similarity. Every query is scoped to one workspace, so one team's documents never show up in another team's results.

It runs end to end with **no paid API keys**. If no language model is reachable, the app falls back to a deterministic mock, so the UI always renders something usable.

---

## Table of Contents

1. [What this project proves](#what-this-project-proves)
2. [How to run it](#how-to-run-it)
3. [The demo flow (step by step)](#the-demo-flow-step-by-step)
4. [Architecture](#architecture)
5. [The hard problems and how they're solved](#the-hard-problems-and-how-theyre-solved)
6. [API reference](#api-reference)
7. [Tests](#tests)
8. [What's NOT production-ready](#whats-not-production-ready)
9. [Honest limitations](#honest-limitations)
10. [Troubleshooting](#troubleshooting)

---

## What this project proves

The system shows three things working end to end:

1. **Tenants are isolated by construction.** Every API route resolves the current workspace, and every SQL query, including the raw vector and full-text ones, filters on `workspaceId`. There is no path by which a chunk from another workspace can appear in a result.
2. **Hybrid search beats either method alone.** Keyword search (Postgres full-text) and vector search (pgvector cosine distance) each return their top 25, and the two lists are merged with reciprocal rank fusion.
3. **Every answer can be verified.** Citations render directly under the answer with a score, a filename, and a chunk number. Retrieved text is also scanned for prompt-injection patterns before it reaches the model.

You'll see all three in the app.

---

## How to run it

### Prerequisites

- **Docker Desktop** (or any Docker with Compose v2). Check with `docker compose version`.
- That's it for the default setup. No Node, no pnpm, no Postgres, and no Redis on your machine. They all run in containers.
- **Ollama is optional.** With it, you get real embeddings and a real chat model. Without it, the app uses the deterministic mock.

### Start the stack

Mac/Linux:

```bash
cp .env.example .env
docker compose up -d --build
```

Windows (PowerShell):

```powershell
Copy-Item .env.example .env
docker compose up -d --build
```

The first build takes a few minutes. Later starts take seconds.

Compose starts five containers:

| Container | What it does |
|---|---|
| `postgres` | PostgreSQL 16 with pgvector |
| `redis` | Redis 7, the BullMQ queue |
| `app-migrate` | One-time job: creates tables, indexes, and demo data, then exits |
| `app` | The Next.js app on port 3000 |
| `worker` | The BullMQ ingestion worker |

`app-migrate` showing `Exited (0)` is **normal**. It's a setup job, not a crash. Check everything with:

```bash
docker compose ps
```

### Open the app

Once `app` reports `healthy`, open:

**http://localhost:3000**

**Demo login:** `demo@pointcite.dev` / `pointcite-demo`

### Stop the stack

```bash
docker compose down
```

This keeps your data. Add `-v` only if you want to **delete** the database and uploads.

### Optional: local models and S3-compatible storage

```bash
# local LLM + embeddings
docker compose --profile ollama up -d ollama
docker exec -it $(docker ps -qf name=ollama) ollama pull nomic-embed-text
docker exec -it $(docker ps -qf name=ollama) ollama pull gemma3:4b

# S3-compatible storage for uploads
docker compose --profile s3 up -d minio minio-create-bucket
```

**Pointing the containers at Ollama.** Inside a container, `localhost` is the container itself, so set `OLLAMA_HOST` in `.env` to match where Ollama runs:

| Where Ollama runs | `OLLAMA_HOST` |
|---|---|
| On your machine (Docker Desktop on Windows or Mac) | `http://host.docker.internal:11434` |
| In Compose, using the `ollama` profile | `http://ollama:11434` |

Then restart with `docker compose up -d`. On Linux, `host.docker.internal` is not defined by default, so use the Compose profile instead. The `localhost` default is correct only when you run the app on the host (see "Local development").

### Local development

For faster iteration, run the dependencies in Compose and the app on the host:

```bash
docker compose up -d postgres redis
pnpm install
pnpm db:setup       # db push + pgvector index + seed demo data
pnpm dev
pnpm worker
```

`db:setup` creates the tables, applies the pgvector and full-text indexes, and seeds the demo data. The app and the worker are two separate processes. They share the Postgres database and the Redis queue.

### Environment variables

See `.env.example` for the full list. The ones that matter most:

| Var | Purpose |
|---|---|
| `DATABASE_URL` | Postgres connection. Must have `pgvector` available. |
| `REDIS_URL` | Redis for BullMQ. |
| `BETTER_AUTH_SECRET` | Server secret for cookie and session signing. |
| `STORAGE_BACKEND` | `local` by default. Set `s3` for S3/R2/MinIO-compatible storage. |
| `STORAGE_DIR` | Local upload root when using `local`. |
| `S3_*` | Endpoint, region, bucket, and credentials for the S3 backend. |
| `INTERNAL_ADMIN_EMAILS` | Comma-separated emails allowed into internal operator tools. |
| `ANTHROPIC_API_KEY` | Optional. Enables Claude Agent SDK chat. |
| `RERANKER` | `none` by default. `cohere`, `voyage`, or `ollama` reranks Ask retrieval. |
| `RAG_INJECTION_MODE` | `warn` (default), `strip`, or `block`. See the prompt-injection guard below. |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Enables trace export when set, e.g. `http://localhost:4317`. |
| `OLLAMA_HOST` | Defaults to `http://localhost:11434`. See the note above for use inside Docker. |
| `OLLAMA_CHAT_MODEL` | Defaults to `gemma3:4b`. |
| `EMBEDDING_DIM` | Must match the `vector(N)` column in `schema.prisma`. |
| `MAX_UPLOAD_BYTES` | Per-file upload size limit. |

---

## The demo flow (step by step)

The app walks through the full pipeline in five steps. The intended order:

### Step 1: Log in

Open **http://localhost:3000** and sign in with `demo@pointcite.dev` / `pointcite-demo`.

**What happened before you got here:** the `app-migrate` container created the tables and indexes, then seeded a demo user, a workspace, and three sample documents (`pgvector-overview.md`, `bullmq-ingestion.md`, `rag-citations.md`). They are already chunked, embedded, and searchable.

### Step 2: Upload a document

Go to the **Documents** page and drop in a PDF, a DOCX, or a TXT/MD file.

**What actually happens:**
- The file is size-checked (`MAX_UPLOAD_BYTES`) and type-checked against an allowlist.
- It's saved to storage, under a workspace-prefixed path.
- A `Document` row is created with `status: uploaded`.
- A job is added to the BullMQ `ingest` queue.

The request returns immediately. The heavy work happens in the worker.

### Step 3: Watch it get indexed

Open the **Jobs** page. The worker picks up the job and runs:

1. Extract the text (`pdf-parse`, `mammoth`, or plain UTF-8).
2. Split it into chunks of about 2,000 characters, on paragraph boundaries, with 200 characters of overlap.
3. Embed each chunk (Ollama `nomic-embed-text`, or the deterministic mock).
4. Insert the chunks and their vectors in a single transaction.

The document flips to `indexed`. If something fails, BullMQ retries three times with exponential backoff. After the last failure the document becomes `failed` and the error message is stored. The Documents and Jobs pages have one-click **Retry** buttons, plus **Retry all failed** on the Jobs page.

### Step 4: Search

Open the **Search** page and run the same query in three modes:

- **keyword**: exact-term matching through Postgres full-text search.
- **vector**: meaning-based matching through pgvector.
- **hybrid** (default): both, merged.

Try a query where the document uses different words than you do. With Ollama embeddings, vector and hybrid will find it and keyword won't. (The mock embeddings are hash-based, so vector search is not meaningful without Ollama.)

### Step 5: Ask a question

Open the **Ask** page and try **"What is PointCite?"**.

PointCite retrieves the best chunks, builds a prompt with a numbered context block, calls the chat provider, and returns the answer with **citations** directly underneath: score, filename, and chunk number. Any sentence can be checked against its source.

If no language model is reachable, you'll get a `[DEMO MODE]` answer that lists the top retrieved chunks. That is the mock provider doing its job, not a bug.

### Operator tools

**Bull Board** is mounted at `/internal/bull`. It shows the raw `ingest` queue, including delayed, stalled, and failed jobs. It's open to workspace owners and to emails listed in `INTERNAL_ADMIN_EMAILS`.

---

## Architecture

```
                 ┌──────────────────────┐
                 │     User Browser     │
                 └──────────┬───────────┘
                            │ HTTP
                            ▼
┌───────────────────────────────────────────────────────────┐
│               NEXT.JS APP (App Router, :3000)             │
│                                                           │
│  better-auth session ─► resolve user + current workspace  │
│                                                           │
│  POST /api/documents ──► save file ──► enqueue job        │
│  POST /api/search    ──► keyword + vector ──► fuse (RRF)  │
│  POST /api/ask       ──► retrieve ──► rerank? ──► chat    │
└─────────┬──────────────────────┬──────────────────┬───────┘
          │ Prisma / raw SQL     │ BullMQ enqueue   │ chat
          ▼                      ▼                  ▼
┌──────────────────┐     ┌─────────────┐     ┌───────────────────┐
│   PostgreSQL 16  │     │  Redis 7    │     │  PROVIDER CHAIN   │
│   + pgvector     │     │  (queue)    │     │  1. Claude SDK    │
│                  │     └──────┬──────┘     │  2. Ollama gemma3 │
│  documents       │            │ consume    │  3. mock          │
│  chunks+vectors  │            ▼            └───────────────────┘
│  questions       │     ┌─────────────────────────────────────┐
│  auth tables     │ ◄── │      INGESTION WORKER               │
└──────────────────┘     │  extract ─► chunk ─► embed ─► write │
                         └─────────────────────────────────────┘
```

There are two long-running application processes: the Next.js app (HTTP) and a separate BullMQ worker (`pnpm worker`). They share the Postgres database and the Redis queue, and nothing else.

### The ingestion path

```
upload → POST /api/documents
       → saveUpload (local FS or S3-compatible storage)
       → Document row {status: uploaded}
       → enqueueIngest(documentId)
worker ← BullMQ "ingest" queue
       → extractText (pdf-parse / mammoth / utf-8)
       → chunkText  (~2000 chars, paragraph-aware, 200-char overlap)
       → embedTexts (Ollama nomic-embed-text → mock)
       → INSERT chunks + vectors (single transaction, raw SQL for pgvector)
       → Document {status: indexed, ingestDurationMs}
       → IngestJob {state: completed}
```

### The query path

```
/api/ask → hybrid search, top-6 chunks
         → (optional) rerank the fused top-20
         → build prompt: system rules + numbered <source> blocks
         → provider chain: Claude → Ollama → mock
         → save question, answer, citations, and timings
         → return answer + citations
```

### Tech stack

| Layer | Choice |
|---|---|
| Framework | Next.js 16 (App Router), React 19, TypeScript |
| Styling | Tailwind CSS v4, hand-rolled UI primitives |
| Database | PostgreSQL 16 + `pgvector` (Prisma ORM, `vector(768)`) |
| Queue | Redis 7 + BullMQ (ingestion worker, retries, retention) |
| Auth | [better-auth](https://better-auth.com), email + password |
| LLM | Claude Agent SDK, then Ollama `gemma3:4b`, then mock |
| Embeddings | Ollama `nomic-embed-text` (768-dim), then deterministic mock |
| File parsing | `pdf-parse`, `mammoth` (DOCX), native UTF-8 (TXT/MD) |
| Validation | Zod |
| Tracing | OpenTelemetry (off unless `OTEL_EXPORTER_OTLP_ENDPOINT` is set) |

---

## The hard problems and how they're solved

### Problem 1: One team's documents must never appear in another team's results

In a multi-tenant system, a single missing filter is a data leak. The risk is highest in raw SQL, because the ORM's safety rails aren't there.

**Solution: resolve the workspace once, and filter on it in every query.**

- Every API route identifies the caller through better-auth and the current workspace through `requireCurrentWorkspace()`.
- Every SQL statement, including the raw vector and full-text queries, filters on `workspaceId`. An attacker who tampers with a chunk id still can't fetch foreign data, because the id is checked against the workspace too.
- Storage paths are prefixed with the workspace id, and file names are sanitised before being written.

**Why put the filter inside the SQL itself?** The vector and full-text queries are hand-written because pgvector isn't a native Prisma type. That is exactly where a forgotten filter would hurt most, so those queries carry the `workspaceId` condition explicitly.

### Problem 2: Keyword search and vector search each miss things

Keyword search finds exact terms but misses synonyms. Vector search finds meaning but can miss a rare exact term, like a product code. Neither is enough alone.

**Solution: run both, then merge with reciprocal rank fusion (RRF).**

- **keyword**: `to_tsvector('english', text) @@ websearch_to_tsquery(...)`, ranked by `ts_rank`, served by the `chunk_text_fts` GIN index.
- **vector**: pgvector cosine distance (`embedding <=> query_vector`), served by an HNSW index.
- **hybrid** (the default): take the top 25 from each, then fuse with RRF (k = 60).

**Why RRF and not a weighted sum of scores?** `ts_rank` scores and cosine distances live on different scales, so adding them means picking weights that only work for one data set. RRF looks only at *rank positions*: a chunk near the top of both lists wins. No tuning is needed.

### Problem 3: Retrieved documents can attack the model

Your documents are untrusted input. A file can contain text like "ignore your previous instructions", and once that text is pasted into a prompt, the model may obey it.

**Solution: inspect every retrieved chunk before it reaches the model.**

`src/lib/rag/sanitise.ts` flags six patterns: `ignore_previous`, `role_directive`, `tag_break`, `boundary_marker`, `long_base64`, and `hidden_unicode`. `RAG_INJECTION_MODE` then decides what happens:

| Mode | Effect |
|---|---|
| `warn` (default) | Flag the chunk and pass it through. Zero-width characters are still removed. |
| `strip` | Replace matches of the redactable patterns with `[REDACTED]`. |
| `block` | Drop the chunk entirely. |

The value is case-insensitive, and an unrecognised value falls back to `warn`.

Each chunk is wrapped in a `<source warnings="...">` block, so the model gets explicit metadata about untrusted input. The system prompt also tells the model to treat every `<source>` block as data, never as instructions. The Ask page shows the flags as badges next to each citation.

### Problem 4: The app must work when no AI service is available

A demo that needs a paid API key, or a running model server, breaks the moment someone clones it.

**Solution: ordered fallback chains.** The first provider that succeeds wins. If one is missing or fails, the next is tried.

| | 1st | 2nd | 3rd |
|---|---|---|---|
| **Chat** | Claude Agent SDK (when `ANTHROPIC_API_KEY` is set) | Ollama `gemma3:4b` | Mock |
| **Embeddings** | Ollama `nomic-embed-text` (768-dim) | Deterministic mock | none |

The mock chat provider returns the top retrieved chunks as a labelled `[DEMO MODE]` answer. The mock embedding provider derives a vector from a SHA-256 hash, so it's always available and always gives the same output for the same text.

**One catch:** every provider must produce vectors of the size in `EMBEDDING_DIM`, which must match the `vector(768)` column in `schema.prisma`. Changing the embedding model means re-ingesting your documents.

---

## API reference

All routes require a signed-in session, except `/api/health`, the auth routes, and the public invitation lookup. Every route that touches data resolves the caller's current workspace first.

| Route | What it does |
|---|---|
| `/api/health` | Health check. Docker polls this to mark the app `healthy`. |
| `/api/auth/*` | better-auth: sign-up, sign-in, sessions, password reset, email verification |
| `/api/documents` | Upload a document |
| `/api/documents/[id]` | A single document |
| `/api/documents/[id]/retry` | Re-queue a failed ingestion |
| `/api/jobs` | Ingestion jobs and their state |
| `/api/jobs/retry-all` | Retry every failed job |
| `/api/search` | Keyword, vector, or hybrid search |
| `/api/ask` | Retrieval-augmented answer with citations |
| `/api/stats` | Dashboard statistics |
| `/api/workspaces` | List and create workspaces |
| `/api/workspaces/[id]` | A single workspace |
| `/api/workspaces/[id]/switch` | Make this the current workspace |
| `/api/workspaces/[id]/members` | Workspace members |
| `/api/workspaces/[id]/members/[userId]` | One member |
| `/api/workspaces/[id]/invitations` | Invitations for a workspace |
| `/api/workspaces/[id]/invitations/[invitationId]` | One invitation |
| `/api/invitations/[token]` | Look up an invitation (public) |
| `/api/invitations/[token]/accept` | Accept an invitation |
| `/internal/bull` | Bull Board queue dashboard (owners and `INTERNAL_ADMIN_EMAILS`) |

### Rate limits

Upload, search, ask, and retry use per-user token buckets. Sign-in, sign-up, password reset, verification resend, and public invitation lookup use per-IP limits. A limited request gets a `429` with `Retry-After` and `X-RateLimit-*` headers. All limits are configurable through the `RATE_LIMIT_*` variables.

Set `TRUST_PROXY=1` only when you run behind a proxy you control. It turns on `X-Forwarded-For` parsing. Without it, the limiter uses a single shared bucket, which refuses spoofed identities.

---

## Tests

```bash
pnpm test            # Vitest unit suite (about 60 tests, sub-second)
pnpm test:coverage   # with v8 coverage
pnpm test:e2e        # Playwright smoke: signup → upload → ingest → search → ask
```

The unit tests cover the chunker, RRF fusion, provider-chain demotion (including streaming), the rate-limit Lua semantics, the RBAC rank helper, both Zod schemas, the workspace auth helpers, and the deterministic mock embeddings.

The Playwright smoke test is **local-only**. It starts the dev server and the BullMQ worker through Playwright's `webServer` array, and tears both down when it finishes.

**CI** (`.github/workflows/ci.yml`) runs on every push and pull request to `main`. On a clean machine, with Postgres and Redis as service containers, it runs lint, the type check, the unit tests, the schema push with its indexes, and a production build. A separate **Trivy** workflow scans dependencies, secrets, and config for HIGH and CRITICAL issues.

---

## What's NOT production-ready

This is a portfolio project. To run it for real, you'd change these things:

| In this repo | In production |
|---|---|
| Demo user with a published password, seeded by `db:setup` | No seeding. Real sign-ups, with email verification turned on. |
| `prisma db push --accept-data-loss` on startup | Versioned migrations (`pnpm db:migrate`) |
| Postgres credentials set to `pointcite`, Redis with no password, both ports published to the host | Strong secrets, private network, no published ports |
| `BETTER_AUTH_SECRET` read from `.env` | A generated secret in a secret manager |
| Local disk storage by default | The S3 backend, which is already supported |
| `console` email provider (prints emails to the log) | Resend or SMTP, both already supported |
| Plain HTTP on `localhost:3000` | TLS in front, with `TRUST_PROXY=1` |
| One worker container | Several workers, scaled to the upload load |
| Tracing off | `OTEL_EXPORTER_OTLP_ENDPOINT` pointing at a collector |

The retrieval, isolation, and injection-guard code is production-shaped. The infrastructure around it is what changes.

---

## Honest limitations

I want this README to be useful to someone reviewing the project, so here is what the design does **not** solve.

1. **Answers are only as good as the retrieved chunks.** The Ask flow sends the top 6 chunks to the model. A question that needs facts spread across many documents can miss some of them. Citations let you check an answer, but they don't prove it's correct.
2. **The injection guard is pattern-based.** It catches the six patterns it knows about. A cleverly worded attack can slip past, which is why the prompt also tells the model to treat sources as data.
3. **DOCX parsing is lossy.** `mammoth` flattens Word files to plain text, so tables and images are dropped.
4. **Uploads pass through the Next.js server** before they reach disk or S3. Browser-to-storage presigned uploads would remove that hop.
5. **Changing the embedding model means re-ingesting everything.** The vectors in the database come from one model and one size (`vector(768)`), so a different model can't be mixed in.
6. **Small local models answer less well than hosted ones.** With only Ollama configured, answer quality depends on the model you pull.
7. **Mock mode isn't an answer.** `[DEMO MODE]` lists the retrieved chunks. It proves the pipeline works, not that the app can reason.

---

## Troubleshooting

**`app-migrate` shows `Exited (0)`.** That's normal. It's the one-time setup job.

**`pull access denied for pointcite`** during the first build. Harmless. Compose tries to pull the image before building it, and the build then continues.

**The build says "transferring context" and shows gigabytes.** Something large is in the project folder and isn't excluded. Check the folder for big files, and add them to `.dockerignore`.

**`database "pointcite" does not exist`, or `password authentication failed`.** Postgres reads its user, password, and database name only on the **first** start, when the data volume is empty. If you renamed them later, run `docker compose down -v` to delete the volume, then `docker compose up -d --build`. This deletes your data.

**`[DEMO MODE]` answers.** No language model was reachable. Set `ANTHROPIC_API_KEY`, or run Ollama and check `OLLAMA_HOST` (see "Pointing the containers at Ollama" above).

**An upload stays on "processing".** The worker handles ingestion. Check it with `docker compose logs worker`.

**`/internal/bull` refuses access.** Add your email to `INTERNAL_ADMIN_EMAILS` in `.env`, then restart the app.

**Port 3000, 5432, or 6379 is already in use.** Change the published port in `docker-compose.yml`, for example `"3001:3000"`.




