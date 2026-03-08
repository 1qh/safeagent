# 03 — System Architecture

> **safeagent** is a multi-tenant AI agent platform built for 10 million users. Every piece of state lives outside the API server. The server itself is stateless, horizontally scalable, and replaceable at any time without data loss.

---

## Table of Contents

- [System Component Boundaries](#system-component-boundaries)
- [Infrastructure Topology](#infrastructure-topology)
- [Storage Architecture](#storage-architecture)
- [Docker Compose Service Map](#docker-compose-service-map)
- [Network Topology](#network-topology)
- [Data Flow: Chat Request](#data-flow-chat-request)
- [Data Flow: File Upload and Processing](#data-flow-file-upload-and-processing)
- [Horizontal Scaling Model](#horizontal-scaling-model)
- [Connection Management](#connection-management)
- [Storage Decision Table](#storage-decision-table)
- [External References](#external-references)

---

## System Component Boundaries

The system is organized into two independently managed codebases that are deployed together: one for reusable agent capabilities and one for HTTP serving.

The shared library codebase contains core agent intelligence: agent definitions built on `@openai/agents` (with Gemini via `aisdk()` bridge), memory adapters, file processing pipelines, budget enforcement, and tool implementations. It also includes a terminal application and a client SDK.

The API-serving codebase remains intentionally thin: it imports the shared library, applies deployment-specific configuration, and exposes capabilities over HTTP via Elysia. This boundary keeps core logic reusable and testable in isolation while keeping transport concerns separate.

```mermaid
graph LR
    subgraph safeagent["safeagent workspace"]
        core["Core Library\nAgent logic · Memory · Tools · File processing"]
        tui["TUI App\nTerminal UI"]
        client["Client SDK\nHTTP SDK"]
    end

    subgraph server_repo["server repo"]
        elysia["Elysia API Server\nRoutes · SSE · Auth lifecycle hooks"]
    end

    core --> elysia
    core --> tui
    client -.->|"HTTP"| elysia
```

---

## Infrastructure Topology

Every service the system depends on is shown below. Arrows indicate the direction of connection initiation (client → server).

```mermaid
graph TB
    subgraph clients["Clients"]
        browser["Browser / Web App"]
        sdk["SDK Consumer"]
    end

    subgraph direct_clients["Direct Library Consumers"]
        tui_client["TUI\n(imports safeagent directly,\nno HTTP)"]
    end

    subgraph api_layer["API Layer (stateless, N instances)"]
        api1["API Server :3000"]
        api2["API Server :3000"]
        apiN["API Server :3000 ..."]
    end

    subgraph storage["Persistent Storage"]
        pg[("PostgreSQL + pgvector\n:5432")]
        surreal[("SurrealDB\n:8000")]
        minio[("MinIO / S3\n:9000")]
    end

    subgraph cache["Cache & Counters"]
        valkey[("Valkey\n:6379")]
    end

    subgraph jobs["Background Jobs"]
        trigger_web["Trigger.dev Webapp\n:3040"]
        trigger_worker["Trigger.dev Worker"]
    end

    subgraph observability["Observability (opt-in)"]
        langfuse_web["Langfuse Web\n:3100"]
        langfuse_worker["Langfuse Worker"]
        clickhouse[("ClickHouse\n:8123")]
        langfuse_redis[("Redis (Langfuse)\n:6380")]
    end

    subgraph conversion["Document Conversion"]
        libreoffice["LibreOffice Headless\n:2002"]
    end

    browser & sdk --> api1 & api2 & apiN
    tui_client -.->|"direct import\n(same workspace)"| pg & surreal & minio & valkey

    api1 & api2 & apiN --> pg
    api1 & api2 & apiN --> surreal
    api1 & api2 & apiN --> minio
    api1 & api2 & apiN --> valkey
    api1 & api2 & apiN --> trigger_web
    api1 & api2 & apiN --> libreoffice

    trigger_web --> pg
    trigger_web --> minio
    trigger_web --> valkey

    api1 & api2 & apiN -.->|"traces (opt-in)"| langfuse_web
    langfuse_worker --> clickhouse
    langfuse_worker --> langfuse_redis
    langfuse_worker --> minio
```

---

## Storage Architecture

Each storage system has a specific, non-overlapping responsibility. Nothing is stored in two places unless one is a cache of the other (and the cache is always the secondary source of truth).

```mermaid
graph LR
    subgraph postgres["PostgreSQL + pgvector"]
        pg_conv["Conversation store\nShort-term memory\n(Drizzle ORM + bun-sql)"]
        pg_pgvector["PgVector\nLarge TXT RAG\n(custom pgvector via Drizzle)"]
        pg_page["page_index table\nHNSW + tsvector GIN\n(Drizzle ORM)"]
        pg_files["file_uploads\nuser_storage_quotas\n(Drizzle ORM)"]
        pg_usage["usage_events\nuser_budget_limits\n(Drizzle ORM)"]
        pg_langfuse[("langfuse DB\n(separate database)")]
    end

    subgraph surreal["SurrealDB"]
        surreal_graph["Graph memory\nRELATE statements"]
        surreal_vec["Vector memory\nCosine similarity search"]
    end

    subgraph minio["MinIO (S3-compatible)"]
        minio_files["safeagent bucket\nOriginal files\nPer-page PDFs\nExtracted images"]
        minio_langfuse["langfuse-media bucket\nLangfuse media blobs"]
    end

    subgraph valkey["Valkey"]
        valkey_budget["Budget counters\nAtomic INCR"]
        valkey_rate["Rate limiting\nSorted sets"]
        valkey_session["Session cache"]
        valkey_router["Embedding router cache\nTopic embeddings"]
        valkey_intent["Intent LLM cache"]
        valkey_registry["FileRegistry cache\nTemporal/ordinal resolution"]
    end

    agent["Agent Request"] --> pg_conv
    agent --> surreal_graph & surreal_vec
    agent --> pg_page & pg_pgvector
    agent --> valkey_budget & valkey_rate & valkey_session
    agent --> valkey_router & valkey_intent & valkey_registry

    file_upload["File Upload"] --> minio_files
    file_upload --> pg_files
    file_upload --> pg_page

    background["Background Jobs\n(Trigger.dev)"] --> pg_usage
    background --> minio_files
    background --> valkey_budget

    langfuse_svc["Langfuse Services"] --> pg_langfuse
    langfuse_svc --> minio_langfuse
```

### Storage Responsibilities in Detail

**PostgreSQL + pgvector** carries the most diverse workload. It hosts four logically separate concerns under one connection pool:

- **Conversation store** (via Drizzle ORM + bun-sql): stores short-term conversational memory as serialized thread state. Schema ownership is application-managed.
- **Custom pgvector** (via Drizzle): stores dense vector embeddings for large plain-text documents, queried with approximate nearest-neighbor search.
- **`page_index` table** (via Drizzle ORM): stores per-page document chunks with both an HNSW vector index and a tsvector GIN index, enabling hybrid retrieval with Reciprocal Rank Fusion.
- **File metadata tables** (`file_uploads`, `user_storage_quotas`): typed relational records managed by Drizzle ORM.
- **Usage and budget tables** (`usage_events`, `user_budget_limits`): append-only event log plus per-user limit configuration.
- **Langfuse database**: a completely separate Postgres database (same server, different `PGDATABASE`) used exclusively by Langfuse services.

**SurrealDB** runs in server mode (Docker container, WebSocket transport). It stores long-term memory as a graph: entities are nodes, relationships are edges created with `RELATE`. Vector similarity search runs as a sequential scan over stored embeddings — no MTREE index is needed at this scale because the long-term memory corpus per user is bounded.

**MinIO** stores all binary objects. The main `safeagent` bucket holds original uploaded files, per-page PDF splits, and images extracted during document processing. A separate `langfuse-media` bucket holds Langfuse's media attachments. Presigned URLs with a 7-day TTL serve images directly to clients without proxying through the API server.

**Valkey** handles everything that needs sub-millisecond latency and atomic operations. Budget counters use `INCR` for lock-free increment. Rate limiting uses sorted sets with sliding windows. The embedding router caches topic embeddings so intent classification doesn't re-embed on every request. FileRegistry caches temporal and ordinal file references (e.g., "the file I uploaded yesterday") with Postgres as the authoritative source of truth. Location enrichment caches geocoding and optional image lookups so repeated place mentions do not re-hit external providers.

---

## Docker Compose Service Map

Services are grouped into runtime profiles. The default profile starts the recommended local development stack. Only Postgres is strictly required to boot — all other services degrade gracefully when absent (see [15 — Infrastructure & Operations](./15-infrastructure.md) for the degradation model). Optional profiles add observability and background job infrastructure.

```mermaid
graph TB
    subgraph default["Default Profile (always on)"]
        pg_svc["postgres\npgvector/pgvector\n:5432"]
        surreal_svc["surrealdb\n:8000"]
        minio_svc["minio\n:9000 (API)\n:9001 (Console)"]
        valkey_svc["valkey\n:6379"]
        libre_svc["libreoffice\nheadless\n:2002"]
    end

    subgraph trigger_profile["Trigger profile"]
        trigger_web_svc["trigger-webapp\n:3040"]
        trigger_super_svc["trigger-supervisor"]
        trigger_docker_svc["trigger-docker-proxy"]
        trigger_electric_svc["trigger-electric"]
        trigger_registry_svc["trigger-registry\n:5000"]
    end

    subgraph langfuse_profile["Langfuse profile"]
        langfuse_web_svc["langfuse-web\n:3100"]
        langfuse_worker_svc["langfuse-worker"]
        clickhouse_svc["clickhouse\n:8123"]
        langfuse_redis_svc["redis (langfuse)\n:6380"]
    end

    pg_svc -.->|"shared server\nseparate DB"| langfuse_web_svc
    minio_svc -.->|"shared server\nseparate bucket"| langfuse_worker_svc

    trigger_super_svc --> pg_svc
    trigger_super_svc --> minio_svc
    trigger_super_svc --> valkey_svc

    langfuse_worker_svc --> clickhouse_svc
    langfuse_worker_svc --> langfuse_redis_svc
```

### Profile Details

**Default profile** starts five services:

| Service | Image | Purpose |
|---------|-------|---------|
| `postgres` | `pgvector/pgvector` | All relational data + vector indexes |
| `surrealdb` | Official SurrealDB | Long-term graph + vector memory |
| `minio` | Official MinIO | Object storage for files and media |
| `valkey` | Official Valkey | Cache, counters, rate limiting |
| `libreoffice` | Headless LibreOffice | DOCX-to-PDF conversion |

### Core Constants

All model, provider, and environment constants are defined in the single source of truth: [04 — Foundation](./04-foundation.md). No constants are duplicated here — refer to file 04 for the authoritative table.

**Trigger profile** adds Trigger.dev's self-hosted stack (five services). The webapp provides the dashboard and HTTP API. The supervisor orchestrates task execution via the docker proxy. Electric handles real-time event streaming from Postgres. A local registry hosts task container images. All services connect to the default-profile Postgres and Valkey instances. See [15 — Infrastructure & Operations](./15-infrastructure.md) for the complete service list.

**Langfuse profile** adds six services for full agent observability. Langfuse Web and Worker share the Postgres server (using a separate `langfuse` database) and the MinIO server (using a separate `langfuse-media` bucket). ClickHouse stores trace event data. A dedicated Redis instance (on port 6380 to avoid collision with Valkey on 6379) handles Langfuse's internal queue.

---

## Network Topology

All services communicate over a single Docker bridge network (`safeagent-net`). External access is limited to the API server port and the MinIO console.

```mermaid
graph TB
    subgraph external["External (host network)"]
        client_ext["Clients\n(browser, TUI, SDK)"]
        admin["Admin\n(MinIO console, Langfuse UI,\nTrigger.dev dashboard)"]
    end

    subgraph docker_net["safeagent-net (Docker bridge)"]
        subgraph api_group["API Servers"]
            api_svc["elysia-api\n:3000 → host:3000"]
        end

        subgraph data_group["Data Services"]
            pg_net["postgres\n:5432 (internal only)"]
            surreal_net["surrealdb\n:8000 (internal only)"]
            valkey_net["valkey\n:6379 (internal only)"]
        end

        subgraph object_group["Object Storage"]
            minio_api_net["minio API\n:9000 (internal only)"]
            minio_console_net["minio console\n:9001 → host:9001"]
        end

        subgraph job_group["Jobs (profile: trigger)"]
            trigger_net["trigger-webapp\n:3040 → host:3040"]
        end

        subgraph obs_group["Observability (profile: langfuse)"]
            langfuse_net["langfuse-web\n:3100 → host:3100"]
            clickhouse_net["clickhouse\n:8123 (internal only)"]
            langfuse_redis_net["redis\n:6380 (internal only)"]
        end

        subgraph conv_group["Conversion"]
            libre_net["libreoffice\n:2002 (internal only)"]
        end
    end

    client_ext -->|"HTTP/SSE :3000"| api_svc
    admin -->|"HTTP :9001"| minio_console_net
    admin -->|"HTTP :3040"| trigger_net
    admin -->|"HTTP :3100"| langfuse_net

    api_svc --> pg_net & surreal_net & valkey_net & minio_api_net & libre_net
    api_svc -.->|"traces"| langfuse_net
    api_svc -->|"HTTP trigger"| trigger_net
```

### Port Reference

| Port | Service | Exposure | Protocol |
|------|---------|---------|---------|
| 3000 | Elysia API | Public | HTTP / SSE |
| 5432 | PostgreSQL | Internal | TCP (Postgres wire) |
| 8000 | SurrealDB | Internal | WebSocket |
| 9000 | MinIO API | Internal | HTTP (S3) |
| 9001 | MinIO Console | Admin | HTTP |
| 6379 | Valkey | Internal | RESP (Redis protocol) |
| 2002 | LibreOffice | Internal | HTTP |
| 3040 | Trigger.dev | Admin | HTTP |
| 3100 | Langfuse Web | Admin | HTTP |
| 8123 | ClickHouse | Internal | HTTP |
| 6380 | Redis (Langfuse) | Internal | RESP |

---

## Data Flow: Chat Request

A typical chat message travels through several layers before the agent responds. SSE keeps the connection open for streaming tokens.

```mermaid
sequenceDiagram
    participant C as Client
    participant API as Elysia API Server
    participant VK as Valkey
    participant PG as PostgreSQL
    participant SR as SurrealDB
    participant LLM as LLM Provider
    participant LF as Langfuse (opt-in)

    C->>API: Chat streaming request {message, threadId}
    API->>VK: Check rate limit (sorted set)
    VK-->>API: OK / rate limited

    API->>VK: Reserve estimated token budget (INCRBY estimated → check total vs limit)
    VK-->>API: OK (or 429 if total exceeds limit → DECRBY to rollback)

    API->>PG: Load thread memory (Conversation store)
    PG-->>API: Conversation history

    Note over API: Long-term memory is agent-initiated:<br/>agent calls memoryRecall tool if needed<br/>(not auto-injected on every request)

    API->>VK: Check intent cache (embedding router)
    VK-->>API: Cached intent OR miss

    Note over API: On cache miss: embed query,<br/>classify intent, cache result

    API->>LLM: Stream completion (with memory context)
    LLM-->>API: Token stream (SSE)

    API-->>C: SSE token stream

    Note over API: After stream completes:

    API->>PG: Persist updated memory (Conversation store)
    API->>VK: Reconcile budget (INCRBY/DECRBY difference: actual − estimated)
    API->>PG: Append usage_event
    API->>LF: Flush trace (async, opt-in)
```

The critical path (rate check → budget check → memory load → LLM stream) is kept as short as possible. Memory persistence and usage accounting happen after the stream completes so they don't add latency to the user-facing response.

---

## Data Flow: File Upload and Processing

File processing is split into a synchronous upload phase (fast, user-facing) and an asynchronous enrichment phase (background, Trigger.dev).

```mermaid
sequenceDiagram
    participant C as Client
    participant API as Elysia API Server
    participant VK as Valkey
    participant PG as PostgreSQL
    participant S3 as MinIO (S3)
    participant LO as LibreOffice
    participant TR as Trigger.dev
    participant LLM as LLM Provider

    C->>API: Upload file {file}

    API->>VK: Check storage quota (counter)
    VK-->>API: Quota OK / exceeded

    API->>PG: Check user_storage_quotas (Drizzle)
    PG-->>API: Quota details

    API->>S3: PUT original file → safeagent bucket
    S3-->>API: Object key + ETag

    API->>PG: INSERT file_uploads record (Drizzle)
    PG-->>API: fileId

    API->>VK: Update FileRegistry cache
    API-->>C: {fileId, status: "uploading"}

    Note over API: Blocking stage (in-process, synchronous)

    API->>PG: UPDATE file_uploads (status: "summarizing")

    alt DOCX file
        API->>LO: Convert DOCX → PDF
        LO-->>API: PDF bytes
        API->>S3: PUT converted PDF
    end

    API->>API: Split PDF into per-page PDFs
    API->>S3: PUT per-page PDFs (batch)
    API->>API: Extract images from pages
    API->>S3: PUT extracted images
    API->>LLM: Gemini per-page summarization (parallel)
    LLM-->>API: Page summaries
    API->>LLM: Embed summaries (batch)
    LLM-->>API: Dense vectors
    API->>PG: INSERT page_index rows (summary + vector)
    API->>PG: UPDATE file_uploads (status: "ready")
    API->>VK: Invalidate FileRegistry cache entry

    Note over API,TR: Background stage via Trigger.dev

    API->>TR: Trigger "background-enrichment" job {fileId}
    TR->>PG: Load page_index rows for file
    TR->>S3: Fetch per-page PDFs
    TR->>TR: Extract raw text per page (unpdf)
    TR->>LLM: Embed raw text (batch)
    LLM-->>TR: Dense vectors
    TR->>PG: UPDATE file_uploads (status: "enriching")
    TR->>PG: UPDATE page_index rows (raw_text + raw_embedding + tsvector)
    TR->>PG: UPDATE file_uploads (status: "enriched")
```

The synchronous path (upload → S3 → metadata record) completes in under a second. The enrichment pipeline (conversion → splitting → embedding → indexing) runs in the background and can take seconds to minutes depending on file size. The client polls the file status endpoint when processing completes.

---

## Horizontal Scaling Model

The API server is stateless. Every instance connects to the same external services. A standard round-robin load balancer distributes traffic without sticky sessions.

```mermaid
graph TB
    subgraph lb["Load Balancer (round-robin)"]
        lb_node["nginx / cloud LB\n:443 → :3000"]
    end

    subgraph api_fleet["API Server Fleet"]
        api_a["API Instance A\n:3000"]
        api_b["API Instance B\n:3000"]
        api_c["API Instance C\n:3000"]
        api_n["API Instance N\n:3000"]
    end

    subgraph shared_state["Shared External State"]
        pg_shared[("PostgreSQL\n(primary + read replicas)")]
        surreal_shared[("SurrealDB")]
        minio_shared[("MinIO\n(distributed mode)")]
        valkey_shared[("Valkey\n(cluster mode)")]
    end

    subgraph bg_fleet["Background Worker Fleet"]
        tr_worker_a["Trigger.dev Worker A"]
        tr_worker_b["Trigger.dev Worker B"]
    end

    lb_node --> api_a & api_b & api_c & api_n

    api_a & api_b & api_c & api_n --> pg_shared
    api_a & api_b & api_c & api_n --> surreal_shared
    api_a & api_b & api_c & api_n --> minio_shared
    api_a & api_b & api_c & api_n --> valkey_shared

    tr_worker_a & tr_worker_b --> pg_shared
    tr_worker_a & tr_worker_b --> minio_shared
    tr_worker_a & tr_worker_b --> valkey_shared
```

### Scaling Properties

**API servers** scale horizontally with no coordination. Because SSE streams are request-scoped (not persistent WebSockets), any instance can handle any request. If an instance dies mid-stream, the client reconnects and the next instance picks up from the last persisted memory state.

**PostgreSQL** scales reads with read replicas. Write traffic (memory persistence, usage events) goes to the primary. The connection pool budget (200 max connections) is divided across all API instances — adding instances requires either reducing per-instance pool size or adding a connection pooler like PgBouncer in front. At 10M-user scale, high-write tables (`page_index`, `file_uploads`, `usage_events`) should be range-partitioned by `user_id` hash to distribute write I/O and keep individual partition indexes manageable. Drizzle ORM supports partitioned tables via raw DDL migrations — the application query layer remains unchanged because all queries already include `user_id` in their WHERE clauses, which enables partition pruning automatically.

**SurrealDB** scales vertically in the initial deployment. At 10M users, the long-term memory corpus is bounded per user (hundreds to low thousands of facts), so total storage grows linearly (~1KB per fact × ~500 average facts × 10M users = ~5TB). SurrealDB's TiKV-backed distributed mode handles this range, but the deployment topology (replica count, region placement, failover) is environment-specific. Sequential scan for vector similarity remains viable because queries are always scoped to a single userId partition.

**MinIO** runs in distributed mode across multiple nodes for production, providing both redundancy and throughput scaling. Object keys use a userId-prefixed hierarchy, which can create hot prefixes if a small number of users dominate upload volume. MinIO's erasure-coding distributes data across drives regardless of key prefix, so this is a monitoring concern (watch per-drive I/O balance) rather than an architectural one. If prefix hotspotting is observed, a hash-prefix scheme (first N characters of a userId hash prepended to the key) distributes requests across the internal keyspace more evenly.

**Valkey** runs in cluster mode for production, sharding keys across nodes. Budget counters and rate limiting use hash tags to ensure related keys land on the same shard. At 10M users, baseline memory is approximately ~600 bytes per active user (rate-limit sorted set + budget counters), totaling ~6GB for the full user base if all users are concurrently active. In practice, concurrent active users are a fraction of total users — a 1% daily active rate means ~60MB of hot state. The embedding router cache (~300KB for 75 topic embeddings), FileRegistry cache, and geocoding cache add workload-dependent memory that should be monitored and budgeted separately. Valkey's `maxmemory-policy allkeys-lru` ensures graceful eviction of cold entries under memory pressure.

**Trigger.dev workers** scale independently of API servers. More workers means more parallel background jobs without affecting API latency.

---

## Connection Management

All database clients share a single Postgres server with a hard limit of 200 connections. This budget must be divided carefully across all consumers.

```mermaid
graph TB
    subgraph pg_server["PostgreSQL Server (max_connections=200)"]
        pg_budget["200 connections total"]
    end

    subgraph api_instance["Per API Instance"]
        conv_pool["Conversation store pool\n(Drizzle ORM + bun-sql)\n~5 connections"]
        pgvector_pool["Custom pgvector pool\n(Drizzle)\n~5 connections"]
        drizzle_pool["Drizzle ORM pool\n(page_index, files, usage)\n~10 connections"]
    end

    subgraph trigger_instance["Per Trigger.dev Worker"]
        trigger_pool["Worker pool\n~10 connections"]
    end

    subgraph langfuse_instance["Langfuse Services"]
        langfuse_pool["Langfuse pool\n~10 connections"]
    end

    pg_budget --> conv_pool & pgvector_pool & drizzle_pool
    pg_budget --> trigger_pool
    pg_budget --> langfuse_pool
```

### Connection Budget Strategy

Each API instance uses three separate connection pools targeting the same Postgres server:

- **Conversation store pool**: application-managed memory storage. Short-lived queries, low concurrency.
- **Custom pgvector pool**: RAG embedding queries. Moderate concurrency during file processing.
- **Drizzle ORM pool**: All schema-managed tables. Highest concurrency — file metadata, usage events, page index queries all go here.

The total per-instance connection count must stay low enough that `N instances × connections_per_instance + worker_connections + langfuse_connections < max_connections`. At 10 API instances with 20 connections each, that's 200 connections consumed by API alone — zero headroom for workers or Langfuse. **PgBouncer (or equivalent connection pooler) is required in any deployment beyond single-instance development** (see [15 — Infrastructure & Operations § Postgres at Scale](./15-infrastructure.md#postgres-at-scale)). PgBouncer multiplexes hundreds of application connections onto a smaller pool of actual Postgres connections, eliminating the per-instance pool sizing constraint entirely. The `max_connections=200` value in Docker Compose is a local development default — production Postgres behind PgBouncer can serve thousands of application-side connections.

**SurrealDB** uses a persistent WebSocket connection per API instance. The TypeScript SDK manages reconnection automatically.

**Valkey** uses `ioredis` with a connection URL in `redis://` scheme (not `valkey://` — the Redis protocol is identical, but the URL scheme must match what `ioredis` expects). A single client instance per API process handles all Valkey operations.

**MinIO** connections are stateless HTTP requests via `Bun.S3Client`. No persistent connection pool needed — each request opens and closes independently.

---

## Storage Decision Table

| Concern | Storage | Reason |
|---------|---------|--------|
| Short-term memory | Postgres (Conversation store) | Application-managed via Drizzle, thread-scoped |
| Long-term memory | SurrealDB | Graph relationships + vector similarity in one store |
| Per-page doc search | Postgres (Drizzle `page_index` + pgvector + tsvector) | Hybrid RRF combining dense and sparse retrieval |
| Large TXT RAG | Custom pgvector (Drizzle) | Custom text chunking pipeline, application-managed |
| Original files | S3 via `Bun.S3Client` | Scalable object storage, presigned URL delivery |
| File metadata + quotas | Postgres (Drizzle ORM) | Type-safe relational schema, quota enforcement |
| Background jobs | Trigger.dev (self-hosted) | Containerized tasks, retries, dashboard, HTTP trigger |
| Real-time cache + counters | Valkey | Sub-millisecond budget counters, atomic INCR, rate limits |
| Embedding router cache | Valkey | Cached topic embeddings for fast intent classification |
| FileRegistry cache | Valkey (+ Postgres source of truth) | Fast temporal/ordinal file reference resolution |
| Observability traces | Langfuse (ClickHouse + Postgres + Redis) | Full agent tracing with structured span data |

---

## External References

- [OpenAI Agents SDK documentation](https://openai.github.io/openai-agents-js/)
- [AI SDK documentation (model layer)](https://sdk.vercel.ai/docs)
- [Elysia documentation](https://elysiajs.com)
- [SurrealDB documentation](https://surrealdb.com/docs)
- [Trigger.dev documentation](https://trigger.dev/docs)
- [Langfuse documentation](https://langfuse.com/docs)
- [Valkey documentation](https://valkey.io/docs)
- [MinIO documentation](https://min.io/docs)
- [Drizzle ORM documentation](https://orm.drizzle.team/docs)
- [Bun documentation](https://bun.sh/docs)

---

*Previous: [02 — Research & Decisions](./02-research.md) | Next: [04 — Foundation](./04-foundation.md)*
