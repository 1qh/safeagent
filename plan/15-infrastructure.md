# 15 — Infrastructure
> All infrastructure is containerized and declarative. The API server is stateless, and every piece of durable state lives in a purpose-built external service. Background jobs run in Trigger.dev, real-time counters live in Valkey, and every service exposes a health check. When Valkey is unavailable, the system degrades gracefully with in-memory fallbacks. When Trigger.dev is absent, jobs execute in-process. Nothing is mandatory except Postgres.
---
## Table of Contents
- [Infrastructure Stack Overview](#infrastructure-stack-overview)
- [Docker Compose Service Map](#docker-compose-service-map)
- [Deployment Strategy](#deployment-strategy)
- [API Key Pool](#api-key-pool)
- [Valkey Cache](#valkey-cache)
- [Cost Tracking and Budget Enforcement](#cost-tracking-and-budget-enforcement)
- [Trigger.dev Integration](#triggerdev-integration)
- [Rate Limiting](#rate-limiting)
- [Structured Logging](#structured-logging)
- [TTL Cleanup](#ttl-cleanup)
- [Circuit Breaker](#circuit-breaker)
- [Health Checks](#health-checks)
- [Graceful Shutdown](#graceful-shutdown)
- [Database Migrations (Drizzle)](#database-migrations-drizzle)
- [Task Specifications](#task-specifications)
- [Capacity Planning](#capacity-planning)
- [External References](#external-references)
---
## Infrastructure Stack Overview
The complete infrastructure stack spans three container-orchestration profiles. The core profile (five services) is the default startup set. The Langfuse profile (four services) and Trigger.dev profile (five services) are opt-in for observability and production-grade background jobs respectively.
```mermaid
graph TB
    subgraph CORE_SERVICES["Core Services (always running)"]
        POSTGRES_DB[("PostgreSQL + pgvector")]
        SURREAL_DB[("SurrealDB")]
        MINIO_STORAGE[("MinIO S3")]
        VALKEY_CACHE[("Valkey")]
        LIBREOFFICE_SIDECAR["LibreOffice Sidecar\n(separate Compose service)"]
    end
    subgraph TRIGGER_PROFILE["Trigger.dev Profile (opt-in)"]
        TRIGGER_WEBAPP["trigger-webapp"]
        TRIGGER_SUPERVISOR["trigger-supervisor"]
        TRIGGER_DOCKER_PROXY["trigger-docker-proxy"]
        TRIGGER_ELECTRIC["trigger-electric"]
        TRIGGER_REGISTRY["trigger-registry"]
    end
    subgraph LANGFUSE_PROFILE["Langfuse Profile (opt-in)"]
        CLICKHOUSE_DB[("ClickHouse")]
        LANGFUSE_REDIS[("Redis")]
        LANGFUSE_WEB["langfuse-web"]
        LANGFUSE_WORKER["langfuse-worker"]
    end
    subgraph API_LAYER["API Layer (stateless, N instances)"]
        ELYSIA_API["Elysia API Server"]
    end
    ELYSIA_API -->|"Drizzle ORM"| POSTGRES_DB
    ELYSIA_API -->|"WebSocket"| SURREAL_DB
    ELYSIA_API -->|"S3 protocol"| MINIO_STORAGE
    ELYSIA_API -->|"Redis client library"| VALKEY_CACHE
    ELYSIA_API -->|"HTTP API"| TRIGGER_WEBAPP
    ELYSIA_API -->|"langfuse SDK"| LANGFUSE_WEB
    TRIGGER_WEBAPP --> POSTGRES_DB
    TRIGGER_WEBAPP --> VALKEY_CACHE
    TRIGGER_SUPERVISOR --> TRIGGER_WEBAPP
    TRIGGER_SUPERVISOR --> TRIGGER_DOCKER_PROXY
    TRIGGER_ELECTRIC --> POSTGRES_DB
    LANGFUSE_WEB --> POSTGRES_DB
    LANGFUSE_WEB --> CLICKHOUSE_DB
    LANGFUSE_WEB --> LANGFUSE_REDIS
    LANGFUSE_WEB --> MINIO_STORAGE
    LANGFUSE_WORKER --> POSTGRES_DB
    LANGFUSE_WORKER --> CLICKHOUSE_DB
    LANGFUSE_WORKER --> LANGFUSE_REDIS
```
**Design principles**:
- **Stateless API** — every API server instance is replaceable. All durable state lives in Postgres, SurrealDB, MinIO, or Valkey. Horizontal scaling is adding more API instances behind a load balancer.
- **Graceful degradation** — Valkey down means in-memory fallback. Trigger.dev absent means in-process execution. Langfuse missing means silent no-op tracing. Only Postgres is truly required at the infrastructure service level, and the server will not start without a working Postgres connection (see the degradation model below for the full criticality matrix).
- **Profile isolation** — default startup runs only the core development services. Langfuse and Trigger.dev are activated through optional profile selectors.
### Degradation Model (Canonical)
- **Postgres unavailable**: hard failure. Core persistence is unavailable, so the instance is `down` and should return HTTP 503 for health checks.
- **JWT secret absent in production** (`NODE_ENV=production`): hard failure. Authentication is a security boundary, and auth fail-open would allow unauthorized data access. The server refuses to start. This is the only non-Postgres hard failure (see [12 — Server Implementation](./12-server.md)).
- **SurrealDB unavailable**: degrade gracefully. Long-term memory is disabled, but chat and short-term memory continue via Postgres.
- **MinIO or S3 unavailable**: degrade gracefully. File upload and file-backed document retrieval are disabled, but chat and non-file agent flows continue.
- **Valkey unavailable**: degrade gracefully. Rate limiting falls back to per-instance in-memory behavior where each API instance tracks its own counters independently. Global rate enforcement is lost, but per-instance protection remains. Budget checks fail-open so users are never blocked due to infrastructure failure, because budget enforcement is soft limits and not a hard security boundary. This is an intentional availability trade-off: temporary over-admission during a Valkey outage is preferable to blocking all users. Valkey availability should be treated as operationally critical and monitored accordingly, because sustained Valkey downtime means rate limits and budgets are effectively per-instance only.
- **Trigger.dev unavailable**: degrade gracefully. Background jobs execute in-process via the fallback queue adapter.
**Relationship to must-have requirements**: [01 — Requirements & Constraints](./01-requirements.md) lists capabilities like S3 storage, SurrealDB memory, and Valkey rate limiting as must-have deliverables. Must-have means the implementation must be shipped and tested. The degradation model above governs runtime behavior during transient outages, crash recovery, and rolling deployments. These are complementary rather than contradictory: code exists and works when infrastructure is present, while the system remains available when infrastructure is temporarily absent. A production deployment missing a must-have service indefinitely is an operational misconfiguration, not a supported configuration. The health endpoint reports `degraded` status and monitoring should alert on sustained degradation.
---
## Docker Compose Service Map
Every service, its purpose, health check, and persistence volume are shown below. Services are organized by profile group.
```mermaid
graph LR
    subgraph PERSISTENT_VOLUMES["Persistent Volumes"]
        VOLUME_PG["pg_data"]
        VOLUME_SURREAL["surrealdb_data"]
        VOLUME_MINIO["minio_data"]
        VOLUME_VALKEY["valkey_data"]
        VOLUME_CLICKHOUSE["clickhouse_data"]
    end
    subgraph CORE_PROFILE["Core Profile"]
        CORE_POSTGRES["PostgreSQL\npgvector/pgvector\n(PgBouncer required at scale —\nsee Capacity Planning)"]
        CORE_SURREAL["SurrealDB\npersistent local data store"]
        CORE_MINIO["MinIO\nS3 + Console"]
        CORE_MINIO_INIT["minio-init\nBucket creation"]
        CORE_VALKEY["Valkey\nCache + counters"]
    end
    subgraph TRIGGER_STACK["Trigger Profile"]
        TRIGGER_WEB_SERVICE["trigger-webapp"]
        TRIGGER_SUPERVISOR_SERVICE["trigger-supervisor"]
        TRIGGER_DOCKER_SERVICE["trigger-docker-proxy"]
        TRIGGER_ELECTRIC_SERVICE["trigger-electric"]
        TRIGGER_REGISTRY_SERVICE["trigger-registry"]
    end
    subgraph LANGFUSE_STACK["Langfuse Profile"]
        LANGFUSE_CLICKHOUSE["ClickHouse\nOLAP"]
        LANGFUSE_REDIS_QUEUE["Redis\nLangfuse queue"]
        LANGFUSE_WEB_SERVICE["langfuse-web"]
        LANGFUSE_WORKER_SERVICE["langfuse-worker"]
    end
    CORE_POSTGRES --- VOLUME_PG
    CORE_SURREAL --- VOLUME_SURREAL
    CORE_MINIO --- VOLUME_MINIO
    CORE_VALKEY --- VOLUME_VALKEY
    LANGFUSE_CLICKHOUSE --- VOLUME_CLICKHOUSE
    CORE_MINIO_INIT -->|"creates buckets"| CORE_MINIO
```
### Service Details
| Service | Image | Port(s) | Purpose | Health Check | Volume |
|---------|-------|---------|---------|-------------|--------|
| **db** | pgvector/pgvector | 5432 | Short-term memory, Drizzle ORM tables, PgVector chunks, file metadata, Langfuse DB, Trigger DB | Native database readiness probe | pg_data |
| **surrealdb** | surrealdb/surrealdb | 8000 | Long-term memory (graph + vector) | Service health endpoint probe | surrealdb_data |
| **minio** | minio/minio | 9000, 9001 | S3-compatible file storage, Langfuse media | Object storage liveness endpoint probe | minio_data |
| **minio-init** | minio/mc | — | Auto-creates application and media buckets | — | — |
| **valkey** | valkey/valkey | 6379 | Cache, budget counters, rate limiting sorted sets | Cache service ping probe | valkey_data |
| **trigger-webapp** | triggerdotdev/trigger.dev | 3040 | Background job dashboard and API | — | — |
| **trigger-supervisor** | triggerdotdev/supervisor | — | Manages containerized worker execution | — | — |
| **trigger-docker-proxy** | triggerdotdev/docker-provider | — | Docker socket proxy for worker containers | — | — |
| **trigger-electric** | electricsql/electric | — | Real-time sync for Trigger.dev | — | — |
| **trigger-registry** | registry | 5000 | Local registry for task images | — | — |
| **clickhouse** | clickhouse/clickhouse-server | 8123, 9100 | Langfuse OLAP traces and observations | Native analytics service ping probe | clickhouse_data |
| **redis** | redis | 6380 | Langfuse queue and cache (separate from Valkey) | Service ping probe | — |
| **langfuse-web** | langfuse/langfuse | 3100 | Langfuse UI and API | — | — |
| **langfuse-worker** | langfuse/langfuse | — | Async trace processor | — | — |
### Database Initialization
Postgres requires a database initialization script mounted in the standard initialization directory that creates the `langfuse` and `trigger` databases on first startup. These databases are isolated from the main application database.
### Port Conflict Resolution
- Valkey runs on default **6379**, and Langfuse Redis is remapped to **6380**.
- MinIO native port 9000 conflicts with ClickHouse native, so ClickHouse native is remapped to **9100**.
- Langfuse web is remapped to **3100** to avoid conflict with the API server on 3000.
- Trigger webapp is remapped to **3040**.
### LibreOffice
LibreOffice is modeled as a Docker Compose sidecar service, not installed into the API server image. The API server calls the sidecar over the internal Docker network on a dedicated service port for DOCX-to-PDF conversion. For local development without Docker, developers can still install LibreOffice on the host machine.
---
## Deployment Strategy
Deployment follows profile-based topology evolution:
- **Development baseline** runs core services only, with optional in-memory fallbacks if Valkey is unavailable and in-process background execution if Trigger.dev is absent.
- **Staging rollout** enables Trigger.dev and Langfuse profiles to validate full background orchestration, tracing, and reconciliation behavior under realistic traffic.
- **Production baseline** uses stateless API replicas behind load balancing, with rolling replacement and health-gated traffic admission.
- **Failure posture** keeps Postgres as the sole hard dependency and treats all other infrastructure as degradable with explicit health reporting.
- **Operational guardrails** require alerting on prolonged degraded mode, especially Valkey unavailability that weakens global rate and budget enforcement.
This strategy aligns with [03 — System Architecture](./03-architecture.md), [04 — Foundation](./04-foundation.md), [12 — Server Implementation](./12-server.md), and [14 — Observability](./14-observability.md).
---
## API Key Pool
The key pool distributes Gemini API calls across N API keys using round-robin. A single key adds zero overhead. N keys provide N× throughput by spreading requests across independent rate limit quotas.
```mermaid
flowchart TB
    KEY_ENV["GOOGLE_API_KEY\n(read via key pool environment variable)\nkey1,key2,key3"]
    POOL_FACTORY["key pool factory"]
    KEY_POOL_INSTANCE["KeyPool Instance"]
    KEY_ENV --> POOL_FACTORY
    POOL_FACTORY --> KEY_POOL_INSTANCE
    subgraph POOL_INTERNALS["Pool Internals"]
        direction LR
        PROVIDER_COUNTER["Provider Counter\n(atomic round-robin)"]
        EMBEDDER_COUNTER["Embedder Counter\n(independent round-robin)"]
    end
    KEY_POOL_INSTANCE --> POOL_INTERNALS
    subgraph ROUND_ROBIN_DISTRIBUTION["Round-Robin Distribution"]
        direction TB
        REQUEST_ALPHA["Request A"] --> KEY_ONE_PROVIDER_ALPHA["Key 1 → Provider"]
        REQUEST_BETA["Request B"] --> KEY_TWO_PROVIDER_BETA["Key 2 → Provider"]
        REQUEST_GAMMA["Request C"] --> KEY_THREE_PROVIDER_GAMMA["Key 3 → Provider"]
        REQUEST_DELTA["Request D"] --> KEY_ONE_PROVIDER_DELTA["Key 1 → Provider"]
    end
    POOL_INTERNALS --> ROUND_ROBIN_DISTRIBUTION
```
```mermaid
flowchart LR
    subgraph KEY_HEALTH_TRACKING["Per-Key Health Tracking"]
        direction TB
        KEY_HEALTHY["HEALTHY\n(in rotation)"]
        KEY_UNHEALTHY["UNHEALTHY\n(skipped)"]
        KEY_HEALTHY -->|"3 consecutive\nAPI failures"| KEY_UNHEALTHY
        KEY_UNHEALTHY -->|"60s re-probe\nsucceeds"| KEY_HEALTHY
        KEY_UNHEALTHY -->|"ALL keys unhealthy\ndegraded mode"| KEY_HEALTHY
    end
    subgraph POOL_API_SURFACE["KeyPool API"]
        NEXT_PROVIDER["getNextProvider()\nReturns LanguageModel"]
        NEXT_EMBEDDER["getNextEmbedder()\nReturns EmbeddingModel"]
        CONCURRENCY_LIMIT["getConcurrencyLimit()\nReturns N × perKey"]
        POOL_SIZE["size\nNumber of active keys"]
        HEALTH_STATUS["getHealthStatus()\nPer-key status array"]
    end
```
### Key Concepts
- **Comma-separated env var** — the key pool environment variable resolves to the `GOOGLE_API_KEY` environment variable (see [04 — Foundation](./04-foundation.md)). That variable contains API keys separated by commas. Whitespace is trimmed. A single key with no comma means no pool is created, and callers use the provider directly.
- **Independent counters** — provider and embedder calls cycle through keys independently. Summarization may call the provider first and the embedder later, and separate counters distribute load evenly across both paths.
- **Per-key concurrency** — default is 5 concurrent requests per key. With 3 keys, the system supports 15 concurrent API calls. This is configurable through `perKeyConcurrency`.
- **Health checking** — each key tracks consecutive failures. After 3 failures, the key is marked unhealthy and skipped. A background probe re-tests unhealthy keys every 60 seconds. If all keys are unhealthy, the pool falls back to round-robin across all keys in degraded mode.
- **Factory from env** — the key pool env helper reads the env var, returns `undefined` for missing or single-key scenarios, and returns a key pool for two or more keys.
---
## Valkey Cache
Valkey provides sub-millisecond read and write for budget counters, rate limiting sorted sets, and general-purpose caching. The connection uses a Redis connection URL. When Valkey is unavailable, an in-memory fallback satisfies the same interface for development and testing.
```mermaid
flowchart TB
    subgraph CACHE_FACTORY["cache factory"]
        direction TB
        VALKEY_URL_CHECK{"VALKEY_URL\nset?"}
        VALKEY_IMPL["Valkey-backed cache\nRedis client library"]
        MEMORY_IMPL["In-memory cache\nMap-backed fallback"]
        VALKEY_URL_CHECK -->|Yes| VALKEY_IMPL
        VALKEY_URL_CHECK -->|No| MEMORY_IMPL
    end
    subgraph CACHE_INTERFACE["Cache Interface"]
        direction LR
        CACHE_GET["get(key)"]
        CACHE_SET["set(key, val, ttl?)"]
        CACHE_DEL["del(key)"]
        CACHE_INCR["incr(key)"]
        CACHE_INCR_BY["incrBy(key, n)"]
        CACHE_DECR_BY["decrBy(key, n)"]
        CACHE_EXPIRE["expire(key, ttl)"]
        CACHE_CLOSE["close()"]
        CACHE_HEALTH["isHealthy()"]
        CACHE_CLIENT["raw client accessor"]
    end
    CACHE_FACTORY --> CACHE_INTERFACE
    subgraph CACHE_CONSUMERS["Cache Consumers"]
        BUDGET_COUNTERS["Budget Counters (COST_TRACKING)\natomic INCR/INCRBY"]
        RATE_LIMIT_CONSUMER["Rate Limiting (RATE_LIMITING)\nsorted sets via raw client accessor"]
        LIMIT_CACHE["Budget Limit Cache\nper-user overrides, 5-min TTL"]
    end
    CACHE_INTERFACE --> CACHE_CONSUMERS
```
```mermaid
flowchart LR
    subgraph BUDGET_KEY_HELPERS["Budget Key Helpers"]
        direction TB
        DAILY_KEY_HELPER["dailyKey(userId)\nbudget:{userId}:daily:{YYYY-MM-DD}"]
        MONTHLY_KEY_HELPER["monthlyKey(userId)\nbudget:{userId}:monthly:{YYYY-MM}"]
        MIDNIGHT_HELPER["secondsUntilMidnightUTC()\nTTL for daily keys"]
        MONTH_END_HELPER["secondsUntilMonthEndUTC()\nTTL for monthly keys"]
    end
    subgraph VALKEY_REALTIME_OPS["Valkey Real-Time Operations"]
        direction TB
        BUDGET_CHECK_OP["Budget Check\ncache.get(dailyKey) + cache.get(monthlyKey)\nsub-ms latency"]
        BUDGET_RECORD_OP["Budget Record\nMULTI: INCRBY + EXPIRE (atomic)\nfire-and-forget"]
        RATE_LIMIT_OP["Rate Limit\nEVAL Lua script\nZREMRANGEBYSCORE + ZADD + ZCARD"]
    end
    BUDGET_KEY_HELPERS --> VALKEY_REALTIME_OPS
```
### In-Memory Fallback
The memory cache implements the identical `Cache` interface using a `Map`. TTL is checked on read. A periodic sweep every 60 seconds prevents unbounded growth in long-running development servers. The raw client accessor returns `null` in memory mode, and consumers that need raw Redis client library operations such as sorted sets or transactional units degrade to no-op or sequential fallback.
### Connection URL
The `VALKEY_URL` environment variable must use a Redis connection URL scheme. Using `valkey://` causes a connection error.
---
## Cost Tracking and Budget Enforcement
Budget enforcement follows an event-sourced design with two layers: a hot path through Valkey for real-time decisions and a cold path through Postgres for audit and reconciliation.
```mermaid
flowchart TB
    subgraph HOT_PATH["Hot Path (request-time)"]
        direction TB
        INCOMING_REQUEST["Incoming Request"]
        BUDGET_CHECK_STEP["checkTokenBudget(cache, userId)\nReads Valkey counters\nsub-ms"]
        BUDGET_GATE{"Budget\nexceeded?"}
        STREAM_EXECUTION["Runner.run(agent, input, { stream: true })\nLLM call proceeds"]
        TOKEN_RECORD_STEP["recordTokenUsage(cache, db, userId, tokens)\nValkey INCR (atomic)\nPostgres INSERT (fire-and-forget)"]
        BUDGET_REJECT["HTTP 429\nBudgetCheckResult JSON\nRetry-After header"]
        INCOMING_REQUEST --> BUDGET_CHECK_STEP
        BUDGET_CHECK_STEP --> BUDGET_GATE
        BUDGET_GATE -->|Allowed| STREAM_EXECUTION
        BUDGET_GATE -->|Exceeded| BUDGET_REJECT
        STREAM_EXECUTION -->|onUsage callback| TOKEN_RECORD_STEP
    end
    subgraph COLD_PATH["Cold Path (every 5 minutes)"]
        direction TB
        SCHEDULED_AGG_TASK["Trigger.dev Scheduled Task\nbudget-aggregation"]
        AGGREGATION_QUERY["SELECT user_id, SUM(tokens_used)\nFROM usage_events\nGROUP BY user_id, DATE(created_at)"]
        DRIFT_CHECK{"Drift > 1%\nor key missing?"}
        RECONCILE_STEP["cache.set(dailyKey, postgresTotal, ttl)"]
        SCHEDULED_AGG_TASK --> AGGREGATION_QUERY
        AGGREGATION_QUERY --> DRIFT_CHECK
        DRIFT_CHECK -->|Yes| RECONCILE_STEP
    end
    subgraph PERSISTENT_STORAGE["Persistent Storage"]
        VALKEY_COUNTER_STORE[("Valkey\nbudget:{userId}:daily:{date}\nbudget:{userId}:monthly:{month}\nauto-expire via TTL")]
        USAGE_EVENTS_STORE[("Postgres\nusage_events\nappend-only audit trail")]
        USER_LIMITS_STORE[("Postgres\nuser_budget_limits\nper-user overrides")]
    end
    BUDGET_CHECK_STEP --> VALKEY_COUNTER_STORE
    TOKEN_RECORD_STEP --> VALKEY_COUNTER_STORE
    TOKEN_RECORD_STEP --> USAGE_EVENTS_STORE
    AGGREGATION_QUERY --> USAGE_EVENTS_STORE
    BUDGET_CHECK_STEP -.->|"cache miss\n(5-min TTL)"| USER_LIMITS_STORE
    RECONCILE_STEP --> VALKEY_COUNTER_STORE
```
### Budget Check Flow
1. Read daily and monthly counters from Valkey with zero Postgres queries on the hot path.
2. Load per-user budget limits from Valkey cache with a five-minute TTL. On cache miss, query `user_budget_limits`, cache both daily and monthly caps, and fall back to config defaults if no override exists.
3. Compare counters against limits and return `BudgetCheckResult` with `allowed`, `daily`, `monthly`, and `resetsAt`.
4. Daily keys auto-expire at midnight UTC, and monthly keys expire at month end. Fresh counters start at zero for each new period.
### Token Recording
Token recording uses Redis client library transactional atomicity where increment and expiration are committed as one unit. This prevents orphaned keys that never expire if one operation succeeds and the other fails. In memory mode, sequential operations are acceptable for development. The Postgres insert into `usage_events` is fire-and-forget and never blocks the response.
### Budget INCRBY Pessimistic Reservation Model
Budget enforcement uses a pessimistic reservation pattern on accumulating spend counters:
- Before starting an LLM call, the system atomically increments the period spend counter by estimated token count using `INCRBY`.
- The returned value is the post-reservation total. If this exceeds the user limit, the reservation is immediately reversed by `DECRBY` of the same estimate, and the request returns HTTP 429.
- After completion, actual usage is reconciled against estimate. If actual exceeds estimate, an additional `INCRBY` is applied. If actual is lower, `DECRBY` is applied for the difference.
- The same period counters are used by both pre-reservation and usage recording, keeping semantics consistent: counters always represent accumulated spend, starting at zero and increasing toward the cap.
- This increment-check-rollback model avoids check-then-spend race conditions without requiring Lua for budget admission.
### Fail-Open Policy
If Valkey is unavailable during budget checks, the system returns `{ allowed: true }` and emits a warning log. Budget enforcement is a soft operational control, not a hard security boundary.
### Budget Admin API
Beyond `checkTokenBudget` and `recordTokenUsage`, the module exposes admin functions: `getUserBudget` reads per-user limits and current spend from Postgres with Valkey cache and returns `BudgetRecord`; `setUserBudget` updates `user_budget_limits` and invalidates cache; `listUserBudgets` returns paginated budget records with optional `overBudget` filtering.
---
## Trigger.dev Integration
Trigger.dev handles all background job execution. In production, tasks run in isolated containers with retries and dashboard visibility. In development, a transparent in-process fallback executes the same handlers directly.
```mermaid
flowchart TB
    subgraph TASK_DISPATCH["Task Dispatch"]
        API_CALLER["API Server\nqueue.trigger(taskId, payload)"]
        ADAPTER_FACTORY["queue adapter factory"]
        TRIGGER_ENV_CHECK{"TRIGGER_DEV_API_URL\nset?"}
        API_CALLER --> ADAPTER_FACTORY
        ADAPTER_FACTORY --> TRIGGER_ENV_CHECK
    end
    subgraph PRODUCTION_PATH["Production Path"]
        TRIGGER_ADAPTER["queue adapter factory (remote dispatcher)"]
        TRIGGER_HTTP_POST["Task dispatch request\nTrigger.dev task endpoint\nBearer auth + JSON payload"]
        TRIGGER_WEBAPP_SERVICE["Trigger.dev Webapp"]
        TRIGGER_SUPERVISOR_PROC["Supervisor"]
        TASK_CONTAINER["Container\nIsolated task execution"]
        TRIGGER_DASHBOARD["Dashboard\nRetries, logs, metrics"]
        TRIGGER_ENV_CHECK -->|Yes| TRIGGER_ADAPTER
        TRIGGER_ADAPTER --> TRIGGER_HTTP_POST
        TRIGGER_HTTP_POST --> TRIGGER_WEBAPP_SERVICE
        TRIGGER_WEBAPP_SERVICE --> TRIGGER_SUPERVISOR_PROC
        TRIGGER_SUPERVISOR_PROC --> TASK_CONTAINER
        TRIGGER_WEBAPP_SERVICE --> TRIGGER_DASHBOARD
    end
    subgraph DEVELOPMENT_PATH["Development Path"]
        IN_PROCESS_ADAPTER["queue adapter factory (in-process fallback)"]
        HANDLER_EXECUTION["Handler executes in-process\nFire-and-forget\nTracked in runningTasks Set"]
        TRIGGER_ENV_CHECK -->|No| IN_PROCESS_ADAPTER
        IN_PROCESS_ADAPTER --> HANDLER_EXECUTION
    end
```
```mermaid
flowchart LR
    subgraph REGISTERED_TASKS["Registered Tasks"]
        TASK_BACKGROUND["background-enrichment\nPDF text extraction + embedding\nRetries: 3, concurrency: 10"]
        TASK_BUDGET_AGGREGATION["budget-aggregation\nReconcile Valkey ↔ Postgres\nSchedule: */5 * * * *"]
        TASK_CLEANUP["cleanup\nExpired file removal\nRetries: 3, concurrency: 5"]
        TASK_TTL_CLEANUP["expired-file-cleanup\nScheduled TTL sweep\nCron: configurable"]
    end
    subgraph TASK_PAYLOADS["Task Payloads"]
        PAYLOAD_BACKGROUND["BackgroundStagePayload\nfileId, threadId, userId, s3Key"]
        PAYLOAD_BUDGET["BudgetAggregationPayload\n(empty — reads from DB)"]
        PAYLOAD_CLEANUP["CleanupPayload\nthreadId, userId"]
    end
    TASK_BACKGROUND --- PAYLOAD_BACKGROUND
    TASK_BUDGET_AGGREGATION --- PAYLOAD_BUDGET
    TASK_CLEANUP --- PAYLOAD_CLEANUP
```
### Task Handlers
Each registered task maps to a shared handler function. The document enrichment handler covers object retrieval, text extraction, embedding, and upsert; the budget aggregation handler covers Postgres-to-Valkey reconciliation with distributed locking; the cleanup handler covers asynchronous file and data cleanup; and the expired file cleanup function performs scheduled TTL sweeps for expired file records.
### QueueAdapter Interface
The queue adapter interface exposes immediate dispatch. The adapter is created once at server startup and injected into route handlers and pipeline functions.
### In-Process Adapter Details
The in-process adapter tracks running tasks in a `Set<Promise<void>>`. Handler failures are caught and logged and never propagate to callers. `getRunningCount` exposes in-flight tasks for health monitoring. During graceful shutdown, the server awaits `Promise.allSettled` to drain running jobs.
### Handler Idempotency
The background enrichment handler uses `UPSERT` keyed on `file_id + page_number`. Since `page_index` has exactly one row per physical page with nullable enrichment columns populated during processing, retries after partial failure update existing rows without creating duplicates.
---
## Rate Limiting
The rate limiter factory produces per-route rate limiting using Valkey sorted sets and a sliding window algorithm. Each request adds a timestamped entry, expired entries are pruned on every check, and all operations execute in a single Lua script for atomicity at scale.
```mermaid
flowchart TB
    subgraph REQUEST_FLOW["Request Flow"]
        RATE_REQUEST["Incoming Request"]
        USER_EXTRACT["Extract userId\nfrom JWT auth context"]
        KEY_BUILD["Build key\nper-user rate limit key"]
        LUA_EXECUTION["Execute Lua Script\n(single roundtrip)"]
        LIMIT_CHECK{"count >\nmaxRequests?"}
        RATE_ALLOW["→ next()"]
        RATE_REJECT["HTTP 429\nRetry-After header"]
        RATE_REQUEST --> USER_EXTRACT
        USER_EXTRACT --> KEY_BUILD
        KEY_BUILD --> LUA_EXECUTION
        LUA_EXECUTION --> LIMIT_CHECK
        LIMIT_CHECK -->|No| RATE_ALLOW
        LIMIT_CHECK -->|Yes| RATE_REJECT
    end
    subgraph LUA_ATOMIC_SCRIPT["Lua Script (atomic)"]
        direction TB
        LUA_PRUNE["ZREMRANGEBYSCORE key -inf (now - windowMs)\nPrune expired entries"]
        LUA_ADD["ZADD key now member\nAdd current request"]
        LUA_COUNT["ZCARD key\nCount in-window entries"]
        LUA_OLDEST["ZRANGE key 0 0 WITHSCORES\nGet oldest entry for Retry-After"]
        LUA_EXPIRE["EXPIRE key ceil(windowMs / 1000)\nSet TTL on key"]
        LUA_PRUNE --> LUA_ADD --> LUA_COUNT --> LUA_OLDEST --> LUA_EXPIRE
    end
```
```mermaid
flowchart LR
    subgraph SLIDING_WINDOW_SET["Sliding Window Sorted Set"]
        direction TB
        SAMPLE_EXPIRED_ALPHA["t=1000 score=1000\n(expired, pruned)"]
        SAMPLE_EXPIRED_BETA["t=2000 score=2000\n(expired, pruned)"]
        SAMPLE_ACTIVE_ALPHA["t=58000 score=58000\nin window"]
        SAMPLE_ACTIVE_BETA["t=59000 score=59000\nin window"]
        SAMPLE_CURRENT["t=60000 score=60000\ncurrent request"]
        SAMPLE_EXPIRED_ALPHA -.->|"pruned by\nZREMRANGEBYSCORE"| SAMPLE_ACTIVE_ALPHA
    end
    subgraph RATE_CONFIG["Configuration"]
        WINDOW_CONFIG["windowMs: 60,000\n(1 minute default)"]
        MAX_REQUESTS_CONFIG["maxRequests: 60\n(per window default)"]
        KEY_PREFIX_CONFIG["keyPrefix: rl"]
    end
```
### Key Design Decisions
- **Lua EVAL over transactional batching** — sorted set rate limiting needs conditional read-then-write behavior and consistent retry timing in one roundtrip.
- **Member uniqueness** — each entry uses `crypto.randomUUID` to avoid collisions when multiple requests arrive within the same millisecond.
- **Per-user keying** — default extraction uses `userId` from auth context, and custom key extraction is supported for non-standard routes.
- **No-op in development fallback** — when the cache raw client accessor is `null`, global rate limiting is disabled to avoid blocking local workflows without Valkey.
- **Retry-After calculation** — derived from the oldest in-window entry using `ceil`, clamped to a minimum of one second.
---
## Structured Logging
All logging uses LogTape via `@logtape/logtape`. The library calls `getLogger` with hierarchical categories, while the consuming server calls `configure` at startup. Request context (`requestId`, `userId`, `threadId`, `agentId`, `traceId`) propagates through AsyncLocalStorage and enriches every log line automatically.
```mermaid
flowchart TB
    subgraph LOGGER_CREATION["Logger Creation"]
        LOGGER_FACTORY["getLogger(category)\nLogTape logger"]
        LOGGER_CONFIG["LoggerConfig\nlevel, name, redact, base"]
        LOG_LEVEL_ENV["LOG_LEVEL env var\nfallback: 'info'"]
        LOGGER_CONFIG --> LOGGER_FACTORY
        LOG_LEVEL_ENV --> LOGGER_FACTORY
    end
    subgraph CONTEXT_PROPAGATION["AsyncLocalStorage Context Propagation"]
        REQUEST_MIDDLEWARE["Elysia Lifecycle Hook\nreads x-request-id\nextracts userId, threadId"]
        ALS_RUNNER["AsyncLocalStorage\nrunWithLogContext(ctx, fn)"]
        LOG_CONTEXT_NODE["LogContext\nrequestId, userId,\nthreadId, agentId, traceId"]
        REQUEST_MIDDLEWARE --> ALS_RUNNER
        ALS_RUNNER --> LOG_CONTEXT_NODE
    end
    subgraph CATEGORY_HIERARCHY["Hierarchical Category Pattern"]
        ROOT_CATEGORY["Root Category\n['safeagent']"]
        GUARDRAIL_LOGGER["getLogger(['safeagent', 'guardrails'])"]
        RAG_LOGGER["getLogger(['safeagent', 'rag'])"]
        BUDGET_LOGGER["getLogger(['safeagent', 'budget'])"]
        UPLOAD_LOGGER["getLogger(['safeagent', 'upload'])"]
        ROOT_CATEGORY --> GUARDRAIL_LOGGER
        ROOT_CATEGORY --> RAG_LOGGER
        ROOT_CATEGORY --> BUDGET_LOGGER
        ROOT_CATEGORY --> UPLOAD_LOGGER
    end
    subgraph JSON_OUTPUT["JSON Log Output"]
        EXAMPLE_LOG_LINE["{ category: ['safeagent', 'guardrails'],\n  severity: 'info',\n  requestId: 'r-abc123',\n  userId: 'u-456',\n  threadId: 't-789',\n  traceId: 'tr-def012',\n  message: 'output blocked',\n  conceptId: 'violence' }"]
    end
    LOG_CONTEXT_NODE -->|"getLogContext()\nenriches category logger"| CATEGORY_HIERARCHY
    CATEGORY_HIERARCHY --> JSON_OUTPUT
```
```mermaid
flowchart LR
    subgraph REDACTION_DEFAULTS["Default Redaction Paths"]
        REDACT_AUTH_HEADER["Authorization header"]
        REDACT_API_KEY["API key fields"]
        REDACT_JWT_SECRET["JWT secret fields"]
    end
    subgraph LOG_LEVELS["Log Levels"]
        LOG_TRACE["trace — verbose debug"]
        LOG_DEBUG["debug — development detail"]
        LOG_INFO["info — normal operations"]
        LOG_WARN["warn — degraded state"]
        LOG_ERROR["error — failures"]
        LOG_FATAL["fatal — unrecoverable"]
    end
```
### Context API
- **`runWithLogContext`** wraps an async function with context that persists through awaited operations.
- **`getLogContext`** retrieves active context from anywhere in the async call stack.
- **`getLogger`** returns a category-scoped logger that includes active AsyncLocalStorage fields.
### Elysia Lifecycle Hooks
The logger lifecycle hook reads or generates `requestId` from the `x-request-id` header, extracts `userId` and `threadId` from Elysia context when available, and wraps the request handler in `runWithLogContext`. Every log line emitted during request processing includes these fields automatically.
LogTape configuration is applied only in the server entry point or test setup, never inside the library. Sensitive field scrubbing uses `@logtape/redaction`, and OpenTelemetry or Langfuse correlation uses `@logtape/otel`.
### Development Watch Mode
Hot-reload mode has known issues with native modules and debugger attachment. The development workflow uses full process restart on file change:
- Library development runs the test suite in watch mode.
- Server development runs the entry point in watch mode with full restart.
- Debug sessions run watch mode with debugger attachment and avoid hot reload.
---
## TTL Cleanup
Expired files are cleaned up by a scheduled Trigger.dev task. The process finds files past expiration, removes associated storage and search indexes, releases storage quota, and marks metadata as deleted.
```mermaid
flowchart TB
    subgraph TRIGGER_SCHEDULE["Trigger.dev Schedule"]
        CLEANUP_CRON["Cron Schedule\n(daily or hourly, configurable)"]
        CLEANUP_TASK["expired-file-cleanup task"]
        CLEANUP_CRON --> CLEANUP_TASK
    end
    subgraph EXPIRED_QUERY["Find Expired Files"]
        EXPIRED_PREDICATE["Query expired file records\nnon-deleted, past expiration\nbatch-limited"]
    end
    CLEANUP_TASK --> EXPIRED_PREDICATE
    subgraph PER_FILE_CLEANUP["Per-File Cleanup (isolated try/catch)"]
        direction TB
        FILE_META_READ["Read fileSize, userId\nfrom candidate row"]
        OBJECT_STORAGE_DELETE["Delete S3 objects\noriginal + page PDFs + images"]
        PAGE_INDEX_DELETE["Delete page_index rows\nsummary + raw_text entries"]
        VECTOR_CHUNK_DELETE["Delete TXT vector chunks\nPgVector storage"]
        QUOTA_RELEASE["Release storage quota\ndecrBy(userId, fileSize)"]
        MARK_FILE_DELETED["Mark file as deleted\nwith deletion timestamp"]
        FILE_META_READ --> OBJECT_STORAGE_DELETE
        OBJECT_STORAGE_DELETE --> PAGE_INDEX_DELETE
        PAGE_INDEX_DELETE --> VECTOR_CHUNK_DELETE
        VECTOR_CHUNK_DELETE --> QUOTA_RELEASE
        QUOTA_RELEASE --> MARK_FILE_DELETED
    end
    EXPIRED_PREDICATE --> PER_FILE_CLEANUP
    subgraph CLEANUP_RESULT["Cleanup Summary"]
        CLEANUP_SUMMARY["Scanned, deleted,\nand failed counts"]
    end
    PER_FILE_CLEANUP --> CLEANUP_RESULT
```
### Idempotency and Failure Isolation
- Each file is cleaned inside its own try/catch block, so one file failing does not stop the batch.
- Deleting already-missing objects is a no-op. Re-running cleanup after partial failure is safe.
- Failed files retain non-deleted status and error message so they are visible for retry on the next schedule.
- Storage quota release is best effort, where undercount is preferable to permanent over-reservation under partial failure.
### Configurable TTL
Each file type can have a different default TTL at upload time. The cleanup task remains policy-agnostic and only queries records where `expires_at < NOW()` and `status != 'deleted'`.
---
## Circuit Breaker
The circuit breaker wraps asynchronous external calls (Gemini API, RAGFlow API, Langfuse, MCP servers) to prevent cascading failures. When a dependency fails repeatedly, the breaker opens and rejects calls immediately to allow recovery.
```mermaid
stateDiagram-v2
    [*] --> CLOSED_STATE
    CLOSED_STATE --> CLOSED_STATE : Success\n(reset failure counter)
    CLOSED_STATE --> OPEN_STATE : Failure count\n≥ failureThreshold (5)
    OPEN_STATE --> OPEN_STATE : execute() called\n→ circuit-open error\n(fast reject, no call made)
    OPEN_STATE --> HALF_OPEN_STATE : resetTimeoutMs (30s)\nelapsed
    HALF_OPEN_STATE --> CLOSED_STATE : halfOpenMaxAttempts (3)\nsuccesses
    HALF_OPEN_STATE --> OPEN_STATE : Any failure\n→ reopen, reset timer
```
```mermaid
flowchart TB
    subgraph BREAKER_API["CircuitBreaker API"]
        BREAKER_EXECUTE["execute(fn)\nWraps async function"]
        BREAKER_STATE["getState()\nclosed | open | half-open"]
        BREAKER_METRICS["getMetrics()\nfailures, successes, openedAt"]
        BREAKER_FORCE_CLOSE["forceClose()\nManual override"]
    end
    subgraph BREAKER_REGISTRY["Circuit Breaker Registry"]
        direction LR
        BREAKER_GEMINI["get('gemini')"]
        BREAKER_LANGFUSE["get('langfuse')"]
        BREAKER_MCP["get('mcp-server-x')"]
        BREAKER_SNAPSHOT["snapshot()\nAll breakers state summary"]
    end
    subgraph BREAKER_DEFAULTS["Default Configuration"]
        BREAKER_THRESHOLD["failureThreshold: 5"]
        BREAKER_RESET["resetTimeoutMs: 30,000"]
        BREAKER_HALF_OPEN["halfOpenMaxAttempts: 3"]
    end
```
### Design Decisions
- **Per-external-call breaker scope** — each external dependency call path has its own breaker, and circuit state is not shared across unrelated integrations.
- **In-process state** — breaker state remains in process memory with no cross-instance synchronization.
- **Typed error** — circuit-open error carries a `retryAt` timestamp.
- **Registry pattern** — the circuit breaker registry factory provides named breakers with isolated state and health snapshot support.
- **Non-swallowing behavior** — wrapped errors propagate and are not hidden.
- **Injectable clock** — `now` can be injected for deterministic transition testing.
---
## Health Checks
The system exposes an aggregated health endpoint that checks critical and non-critical services independently. Overall status is `ok` when all services are up, `degraded` when only non-critical services are down, and `down` when the critical dependency Postgres is down.
```mermaid
flowchart TB
    subgraph HEALTH_ENDPOINT["Health Check Endpoint"]
        HEALTH_AGGREGATOR["Health Aggregator"]
    end
    subgraph SERVICE_CHECKS["Individual Service Checks"]
        CHECK_POSTGRES["Postgres\nconnectivity check"]
        CHECK_SURREAL["SurrealDB\nservice health probe"]
        CHECK_OBJECT_STORAGE["MinIO\nHEAD bucket"]
        CHECK_VALKEY["Valkey\ncache.isHealthy()"]
        CHECK_TRIGGER["Trigger.dev\nAPI ping (optional)"]
        CHECK_LANGFUSE["Langfuse\npublic health probe\n(optional)"]
        CHECK_MCP["MCP Servers\nper-server ping\n(circuit breaker state)"]
    end
    HEALTH_AGGREGATOR --> CHECK_POSTGRES
    HEALTH_AGGREGATOR --> CHECK_SURREAL
    HEALTH_AGGREGATOR --> CHECK_OBJECT_STORAGE
    HEALTH_AGGREGATOR --> CHECK_VALKEY
    HEALTH_AGGREGATOR --> CHECK_TRIGGER
    HEALTH_AGGREGATOR --> CHECK_LANGFUSE
    HEALTH_AGGREGATOR --> CHECK_MCP
    subgraph HEALTH_RESPONSE["Health Response"]
        direction TB
        RESPONSE_OK["HTTP 200\n{ status: 'ok', uptime, build,\n  checks: { postgres: { status, latencyMs }, ... },\n  mcp: { serverId: { status, latencyMs } } }"]
        RESPONSE_DEGRADED["HTTP 200\n{ status: 'degraded', uptime, build,\n  checks: { postgres: { status, latencyMs },\n  valkey: { status: 'down', latencyMs } },\n  mcp: { ... } }"]
        RESPONSE_DOWN["HTTP 503\n{ status: 'down', uptime, build,\n  checks: { postgres: { status: 'down' } },\n  mcp: { ... } }"]
    end
    HEALTH_AGGREGATOR -->|"All services UP"| RESPONSE_OK
    HEALTH_AGGREGATOR -->|"Any non-critical DOWN,\nall critical UP"| RESPONSE_DEGRADED
    HEALTH_AGGREGATOR -->|"Postgres DOWN"| RESPONSE_DOWN
```
### Service Criticality
| Service | Critical | Failure Impact |
|---------|----------|---------------|
| Postgres | Yes | All persistence unavailable |
| SurrealDB | No | Long-term memory unavailable; short-term memory and chat continue via Postgres |
| MinIO or S3 | No | File upload and file-backed retrieval unavailable; chat continues |
| Valkey | No | Rate limiting falls back to in-memory; budget enforcement soft-fails |
| Trigger.dev | No | Background jobs run in-process via fallback adapter |
| Langfuse | No | Tracing disabled silently |
| MCP Servers | No | Individual tools unavailable |
The health endpoint returns HTTP 200 with `status: "ok"` when all services are reachable, HTTP 200 with `status: "degraded"` when one or more non-critical services are down, and HTTP 503 with `status: "down"` only when Postgres is down.
---
## Graceful Shutdown
On termination signals, the server initiates an ordered shutdown sequence that drains in-flight work before closing connections.
```mermaid
sequenceDiagram
    participant OPERATING_SYSTEM as Operating System
    participant API_SERVER as API Server
    participant IN_FLIGHT_REQUESTS as In-Flight Requests
    participant IN_PROCESS_QUEUE as In-Process Queue
    participant VALKEY_SERVICE as Valkey
    participant POSTGRES_SERVICE as Postgres
    participant SURREAL_SERVICE as SurrealDB
    OPERATING_SYSTEM->>API_SERVER: SIGTERM
    API_SERVER->>API_SERVER: Stop accepting new connections
    API_SERVER->>IN_FLIGHT_REQUESTS: Wait for in-flight requests\n(timeout: 30s)
    IN_FLIGHT_REQUESTS-->>API_SERVER: All requests complete
    API_SERVER->>IN_PROCESS_QUEUE: Promise.allSettled([...runningTasks])\nDrain in-process background jobs
    IN_PROCESS_QUEUE-->>API_SERVER: All tasks settled
    API_SERVER->>VALKEY_SERVICE: cache.close()\nDisconnect Redis client library
    API_SERVER->>POSTGRES_SERVICE: db.close()\nRelease connection pool
    API_SERVER->>SURREAL_SERVICE: client.close()\nDisconnect WebSocket
    API_SERVER->>API_SERVER: process.exit(0)
```
### Shutdown Order
1. **Stop accepting connections** — HTTP server stops listening.
2. **Drain in-flight requests** — wait up to 30 seconds for active requests.
3. **Drain background tasks** — `Promise.allSettled` waits for in-process jobs.
4. **Close external connections** — Valkey, Postgres, and SurrealDB disconnect cleanly.
5. **Exit** — process exits with code 0.
This ordering avoids data loss by allowing counters to flush, usage events to persist, and enrichment jobs to complete or remain safely retryable.
---
## Database Migrations (Drizzle)
Infrastructure-facing schema evolution is managed by Drizzle migrations with explicit operational boundaries:
- **Core budget schema** includes append-only `usage_events` for audit and `user_budget_limits` for per-user overrides.
- **File lifecycle schema** includes expiration and deletion metadata needed by TTL cleanup workflows.
- **Initialization separation** keeps app database migrations distinct from infrastructure bootstrapping of auxiliary databases for Trigger.dev and Langfuse.
- **Scale evolution** allows partition-oriented migration strategy for high-write tables once data volume justifies partitioning.
- **Runtime behavior** remains migration-safe because request-time admission paths rely on Valkey counters and cached limits, while Postgres is the source of reconciliation truth.
---
## Task Specifications
### Task DOCKER_COMPOSE: Docker Compose Infrastructure
**What to do**:
- Define the container orchestration infrastructure with all core services: Postgres (pgvector/pgvector), SurrealDB, MinIO, and Valkey.
- Include storage initialization for automatic bucket provisioning (application bucket, media bucket).
- Add Trigger.dev stack under profile `trigger`: webapp, supervisor, docker-proxy, electric, and local registry.
- Add Langfuse stack under profile `langfuse`: ClickHouse, Redis (port 6380), langfuse-web (port 3100), and langfuse-worker.
- Add a Postgres database initialization script for creating `langfuse` and `trigger` databases.
- Configure database connection pooling with an increased connection pool.
- Add a LibreOffice sidecar container in Docker Compose as a separate service and configure API communication over the Compose network.
- Declare persistent volumes: pg_data, surrealdb_data, minio_data, valkey_data, clickhouse_data.
- Add health checks on every service with appropriate check probes.
- Update environment variable templates with all S3, SurrealDB, Valkey, Trigger.dev, Langfuse, and LibreOffice variables.
**Depends on**: SCAFFOLD_SERVER (Server Scaffolding)
**Acceptance Criteria**:
- Starting the core stack brings all core services to healthy state.
- Postgres has pgvector extension available.
- MinIO responds to health checks and both buckets are auto-created.
- SurrealDB health probe reports success.
- Valkey ping probe returns healthy response.
- Starting trigger profile launches Trigger.dev webapp on port 3040.
- Starting langfuse profile launches Langfuse UI on port 3100 and ClickHouse on port 8123.
- LibreOffice sidecar starts healthy and is reachable from the API server on internal network.
- Postgres `langfuse` and `trigger` databases are auto-created by initialization script.
- Environment variable templates include all required variables.
**QA Scenarios**:
- Container services start healthy and report running state.
- MinIO initialization completes and required buckets exist.
- Langfuse profile starts and analytics probes report success.
- Valkey responds to ping and round-trip cache checks.
- Trigger profile starts and webapp is reachable.
- LibreOffice sidecar is reachable from API context.
---
### Task COST_TRACKING: Cost Tracking and Per-User Token Budgets
**What to do**:
- Create budget module with two-layer architecture: Valkey hot path for real-time decisions and Postgres cold path for audit.
- Add Drizzle schema tables: `usage_events` (append-only) and `user_budget_limits` (per-user overrides).
- Implement budget admission checks that read only from Valkey and return budget decision details with allowance state, period usage, and reset timing.
- Implement token usage recording with atomic counter updates and fire-and-forget Postgres persistence.
- Implement an admin budget read capability that returns per-user limits and current spend from Postgres with Valkey cache.
- Implement an admin budget update capability that persists per-user limits, invalidates cache entries, and optionally resets counters.
- Implement paginated budget administration listing with optional over-budget filtering.
- Daily keys auto-expire at midnight UTC and monthly keys at month end.
- Per-user overrides are cached in Valkey with five-minute TTL and fall back to config defaults (1M daily, 20M monthly).
- Budget exceeded responses return HTTP 429 with full `BudgetCheckResult` including Retry-After.
- Fail open when Valkey is unavailable and log warning.
- Middleware checks budget after `userId` extraction and before agent stream.
- Post-stream recording uses `onUsage` callback in stream handler with fire-and-forget semantics.
- Scheduled aggregation task reconciles Valkey counters with Postgres every five minutes.
**Depends on**: FILE_STORAGE (Drizzle schema), SSE_STREAMING (stream handler), VALKEY_CACHE (Cache module)
**Acceptance Criteria**:
- Hot-path budget checks read from Valkey only.
- Daily and monthly counters auto-expire via TTL.
- Recording updates counters atomically and writes append-only usage events.
- Budget exceeded returns descriptive 429 response.
- Token recording never blocks response.
- `usage_events` receives append-only inserts for every recording.
- Keys follow `budget:{userId}:daily:{YYYY-MM-DD}` and `budget:{userId}:monthly:{YYYY-MM}`.
- Per-user overrides are cached and respected.
- Admin budget read returns limits, current spend, and period metadata.
- Admin budget updates persist limits and invalidate cache.
- Budget administration listing supports pagination and optional over-budget filtering.
**QA Scenarios**:
- Over-limit user is blocked with `allowed: false`, remaining clamped to 0, and usage event persistence.
- Daily counters start fresh with TTL set to midnight.
- Hot path performs zero Postgres queries.
- Admin get returns expected custom limits and spend values.
- Admin set takes effect immediately via cache invalidation.
- Admin list with over-budget filter returns only over-limit users.
---
### Task KEY_POOL: API Key Pool
**What to do**:
- Build the key pool capability with a key pool factory and configuration object.
- Read the key pool environment variable (`GOOGLE_API_KEY`), parse comma-separated keys, and trim whitespace.
- Create one provider factory per key using AI SDK Google adapter.
- Implement round-robin distribution with separate provider and embedder counters.
- `getNextProvider` returns LanguageModel and `getNextEmbedder` returns EmbeddingModel.
- `getConcurrencyLimit` returns key count multiplied by `perKeyConcurrency` (default 5).
- The key pool env helper returns `undefined` for missing or single key values and returns a pool for two or more keys.
- Add per-key health tracking where three consecutive failures mark a key unhealthy and 60-second re-probe allows recovery.
- If all keys are unhealthy, enter degraded mode with full-key round-robin instead of hard stop.
**Depends on**: CORE_TYPES (types), SCAFFOLD_LIB (scaffolding)
**Acceptance Criteria**:
- Pool size reflects parsed key count.
- Provider rotation cycles in round-robin order.
- Embedder rotation remains independent from provider sequence.
- Concurrency limit calculation is correct.
- Empty or blank key lists fail fast.
- Env helper returns `undefined` for missing or single key values.
- Whitespace trimming yields clean key arrays.
**QA Scenarios**:
- Round-robin provider sequence cycles deterministically.
- Provider and embedder counters are independent.
- Env parsing with multiple keys yields expected size and concurrency.
- Single-key env value bypasses pool creation.
---
### Task VALKEY_CACHE: Valkey Cache Module
**What to do**:
- Build the cache capability with a cache factory.
- Valkey implementation uses a Redis client library with a Redis connection URL from `VALKEY_URL`.
- Cache interface includes `get`, `set`, `del`, `incr`, `incrBy`, `decrBy`, `expire`, `close`, `isHealthy`, and a raw client accessor.
- The raw client accessor exposes the underlying Redis client library for sorted sets and transactional operations.
- Add in-memory fallback with map-backed storage, read-time TTL checks, and periodic 60-second sweep.
- In-memory raw client accessor returns `null` so consumers can degrade to no-op or sequential fallback.
- Include budget key helpers: `dailyKey`, `monthlyKey`, `secondsUntilMidnightUTC`, `secondsUntilMonthEndUTC`.
- `get` returns `string | null`, where `null` means no key. Numeric consumers parse values, and general cache consumers store serialized JSON.
**Depends on**: CORE_TYPES (types), SCAFFOLD_LIB (scaffolding)
**Acceptance Criteria**:
- Valkey implementation connects and handles get/set/increment operations.
- In-memory fallback conforms to identical interface.
- Increment operations are atomic and return new value.
- Set supports TTL expiration.
- Budget helper key formats are correct.
- Health status reflects connectivity.
- Missing key returns null.
- Close disconnects without hanging.
**QA Scenarios**:
- Valkey operation sequence round-trips values and counter progression.
- In-memory fallback behavior mirrors Valkey semantics.
- Budget key helpers emit correct date and month key patterns.
---
### Task TRIGGER_TASKS: Trigger.dev Task Definitions and QueueAdapter
**What to do**:
- Create trigger module with QueueAdapter implementations.
- The remote queue adapter dispatches to Trigger.dev through its task endpoint.
- The in-process queue adapter executes handlers in-process with fire-and-forget semantics and running-task tracking.
- The queue adapter factory auto-selects an adapter based on Trigger.dev environment variables.
- Define tasks: background-enrichment (retries 3, concurrency 10), budget-aggregation (every five minutes), cleanup (retries 3, concurrency 5).
- Shared handlers include `processBackgroundStageJob`, `runBudgetAggregation`, and `runCleanup`.
- Ensure handler idempotency through upsert keyed on `file_id + page_number`.
**Depends on**: CORE_TYPES (types), RAG_INFRA (RAG functions), FILE_STORAGE (FileStorage), VALKEY_CACHE (Cache)
**Acceptance Criteria**:
- In-process adapter executes registered handlers.
- Unregistered task IDs fail with explicit error.
- Handler failures do not propagate to caller.
- `getRunningCount` reflects active tasks.
- `runningTasks` empties after completion.
- Trigger adapter sends authenticated dispatch payloads to correct endpoint.
- Queue adapter auto-selection behaves correctly by environment.
- Background enrichment remains idempotent under retries.
**QA Scenarios**:
- Fire-and-forget trigger returns quickly while handler continues.
- Dispatch integration emits correct authenticated request.
- Adapter selection changes with env presence.
- Enrichment handler performs expected retrieval, extraction, upsert, and status updates.
---
### Task RATE_LIMITING: Rate Limiting Middleware
**What to do**:
- Build rate limiting middleware with a rate limiter factory.
- Implement sliding window algorithm using Valkey sorted sets.
- Use single Lua script for prune, add, count, oldest-entry lookup, and expiration.
- Default config: `windowMs` 60,000, `maxRequests` 60, key prefix `rl`.
- Build a per-user rate limit key using JWT auth context.
- Ensure member uniqueness through `crypto.randomUUID`.
- Return HTTP 429 with Retry-After and JSON body when limit exceeded.
- Compute Retry-After from oldest in-window entry using ceiling and minimum one second.
- Support custom `keyExtractor` for non-default route patterns.
- No-op pass-through when the raw client accessor returns null in memory mode.
- Provide deterministic tests with fake clock for boundary conditions.
**Depends on**: CORE_TYPES (types), VALKEY_CACHE (Cache or Valkey)
**Acceptance Criteria**:
- Defaults apply when config omitted.
- Middleware reads auth context and builds expected key.
- Sliding window prunes expired entries and counts in-window requests only.
- Over-limit response includes 429, Retry-After, and JSON body.
- Retry-After is positive integer seconds based on oldest event.
- Concurrent requests stay within configured max due to atomic behavior.
- Custom key extractor overrides default behavior.
- Rate limiter exports are available through barrel.
**QA Scenarios**:
- With maxRequests set to 3, first three requests pass and fourth is rejected with Retry-After.
- After advancing time beyond window, request is allowed and effective counter resets.
---
### Task STRUCT_LOGGING: Structured Logging
**What to do**:
- Create logger module using LogTape where library code uses category loggers and server configures sinks at startup.
- Produce JSON output with configurable level from `LOG_LEVEL` and default `info`.
- Default redaction covers `req.headers.authorization`, `*.apiKey`, and `*.jwtSecret`.
- Implement AsyncLocalStorage request context propagation.
- Context shape includes `requestId`, `userId`, `threadId`, and optional `agentId`.
- API includes `runWithLogContext`, `getLogContext`, and `getLogger`.
- `getLogger` returns category-scoped logger enriched with active async context.
- Hierarchical sub-categories inherit parent sink configuration.
- Elysia lifecycle helper reads or generates request ID from `x-request-id` and wraps request execution in context.
**Depends on**: SCAFFOLD_LIB (scaffolding)
**Acceptance Criteria**:
- `getLogger` yields logger scoped to requested category array.
- `runWithLogContext` and `getLogContext` persist context across async boundaries.
- Emitted logs include active request and user context.
- Sub-category loggers inherit parent configuration and context.
- Redaction covers configured sensitive fields by default.
- Lifecycle helper binds request context and ID generation or echo.
- Module exports are available through barrel.
**QA Scenarios**:
- Async context persists across awaits and appears in emitted logs.
- Sensitive fields are redacted in structured output.
---
### Task TTL_CLEANUP: TTL-Based Automatic Cleanup
**What to do**:
- Build cleanup capability with an expired file cleanup function.
- Query expired rows where `expires_at < NOW()` and `status != 'deleted'`, with optional batch limit.
- Perform per-file cleanup with isolated try/catch: read metadata, delete object storage artifacts, delete `page_index` rows, delete vector chunks, release quota, and mark deleted with timestamp.
- Ensure idempotent retries when objects or rows are already absent.
- Keep batch progress even when one file fails, and persist failure details for retry.
- Register scheduled task in Trigger task registry.
- Return summary object with `scanned`, `deleted`, and `failed` counts.
- No extra migration is needed because expiration and deletion columns are already in schema.
**Depends on**: FILE_STORAGE (file storage), TRIGGER_TASKS (task registry), RAG_INFRA (chunk deletion)
**Acceptance Criteria**:
- Expired query targets only current time threshold and non-deleted status.
- Cleanup removes storage assets, page-index rows, and vector chunks.
- Metadata is updated to deleted status with timestamp after successful cleanup.
- Per-file failures are recorded while batch continues.
- Re-running cleanup is idempotent.
- Scheduled task registration invokes cleanup handler.
- Storage quota is released for each deleted file.
**QA Scenarios**:
- Expired file seed is fully cleaned with deleted status update.
- Partial failure still allows subsequent files to be deleted and summary to reflect mixed outcomes.
---
### Task CIRCUIT_BREAKER: Circuit Breaker for External Calls
**What to do**:
- Build the circuit breaker capability with a circuit breaker factory.
- Implement state machine: closed pass-through, open fast-reject with circuit-open error, and half-open limited probing.
- Default config includes threshold 5, reset timeout 30,000, and half-open max attempts 3.
- `execute` wraps async functions and drives transitions from outcomes.
- Circuit-open error includes `retryAt` timestamp.
- The circuit breaker registry factory provides named per-service breakers with isolated state.
- Registry `snapshot` returns all breaker states for health endpoint.
- Breakers are in-memory per process with no Valkey synchronization.
- Clock function is injectable for deterministic tests.
- Wrapped errors propagate without swallowing.
**Depends on**: CORE_TYPES (types)
**Acceptance Criteria**:
- Defaults match threshold 5, reset timeout 30,000, and half-open attempts 3.
- Breaker transitions to open after threshold failures.
- Open state rejects immediately without invoking wrapped call.
- After reset timeout, half-open admits limited trial calls.
- Successful half-open trials close breaker and reset counters.
- Failed half-open trial reopens breaker and resets timer.
- Registry provides isolated per-service instances.
- `forceClose` resets breaker to closed state.
**QA Scenarios**:
- With threshold 2, two failures open breaker and third call fast-rejects without function invocation.
- After timeout elapses, successful probe closes breaker and normal calls resume.
---
## Capacity Planning
### Postgres at Scale
A direct increased connection-pool posture is insufficient for a ten-million-user system with bursty traffic and background workers. PgBouncer or equivalent connection pooling is required in front of Postgres at scale. Multiplexing large numbers of application connections onto a smaller database pool eliminates per-instance pool sizing pressure.
High-write tables (`page_index`, `file_uploads`, `usage_events`) should be range partitioned by `user_id` hash once row counts exceed tens of millions. Partitioning distributes write I/O, keeps B-tree and vector indexes smaller for better insert and query performance, and enables partition-level maintenance without full-table locking. Since application queries already include `user_id` filters, partition pruning applies without query-layer changes. Drizzle supports this through migration-level DDL while query builder behavior remains unchanged.
### Valkey Memory Budget
Baseline per-user memory is roughly 500 bytes for sliding-window rate-limit sets plus roughly 100 bytes for budget counters, or around 600 bytes per active user. At ten million users with one percent daily activity, hot state is around 60MB. At ten percent concurrent activity under burst, hot state is around 600MB. Full-population concurrency would be roughly 6GB.
Additional cache consumers include embedding router topic cache, file registry cache, and geocoding cache. These are workload-dependent and should be monitored through memory telemetry. Configure `maxmemory` with `allkeys-lru` eviction so degradation remains graceful under pressure: evicted rate-limit entries are regenerated on subsequent requests, and evicted cache entries fall through to Postgres or upstream providers.
### SurrealDB Sizing
Long-term memory remains bounded per user, generally hundreds to low thousands of facts over a lifetime. At approximately 1KB per fact and 500 average facts per user, total storage is approximately 5TB at ten million users. SurrealDB distributed mode handles this range. Similarity search is scoped by `userId`, so per-query scanned set remains bounded even as global volume grows.
Deployment topology details like replica counts, shard counts, and region placement are environment-specific. The application layer remains topology-agnostic by connecting to a single SurrealDB endpoint.
### S3 or MinIO Key Distribution
Object keys use user-prefixed hierarchy for ownership cleanup. Concentrated upload traffic among few users can create hot prefixes, but storage-layer distribution mitigates much of this effect. If hotspotting appears at scale, prepend a short hash prefix derived from user identity to spread traffic across keyspace. This change remains isolated to key-generation utilities and does not require broader application changes.
---
## External References
- Trigger.dev: [https://trigger.dev/docs](https://trigger.dev/docs)
- Valkey: [https://valkey.io/docs](https://valkey.io/docs)
- MinIO: [https://min.io/docs](https://min.io/docs)
- LogTape: [https://logtape.org/](https://logtape.org/)
- ioredis: [https://redis.github.io/ioredis/](https://redis.github.io/ioredis/)
- Circuit Breaker pattern: failure threshold plus cooldown window to stop repeated calls to unhealthy dependencies, followed by controlled probes before returning to closed state
- Sliding window rate limiting: [https://redis.io/docs/latest/develop/data-types/sorted-sets/](https://redis.io/docs/latest/develop/data-types/sorted-sets/)
- AsyncLocalStorage: [https://bun.sh/docs/runtime/web-apis](https://bun.sh/docs/runtime/web-apis)
---
*Previous: [14 — Observability](./14-observability.md) | Next: [16 — Testing](./16-testing.md)*
