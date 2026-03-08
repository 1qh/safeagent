# Plan Review Brief — Comprehensive Cross-Reference Verification

> This brief summarizes all 21 plan files and captures every cross-file consistency point.
> Reviewers: read this brief, then verify claims against files 18, 19, and 20.

---

## File Summary (21 files)

| File | Lines | Purpose |
|------|-------|---------|
| overview.md | 254 | High-level architecture diagram, request lifecycle, ToC, deliverables, scale (10M total users, 1% DAU = 100K daily active) |
| 01-system-architecture.md | 578 | Component boundaries (library vs server), infrastructure topology (Postgres+pgvector, SurrealDB, Valkey, S3/MinIO, Trigger.dev), storage architecture, Docker Compose, network topology, data flows (chat + upload), horizontal scaling, connection management (PgBouncer REQUIRED), budget model (INCRBY pessimistic reservation) |
| 02-configuration.md | 341 | Model config constants, thinking levels, library/server responsibility split, env variable flow, `@t3-oss/env-core` with Zod v4, JWT_SECRET production hard-fail (NODE_ENV=production → refuse startup), memory config |
| 03-research-and-decisions.md | 815 | EXEMPT FROM ALL REVIEW RULES. Research spikes, Metis review findings, framework architecture, dependency map, discussion decisions, library selections |
| 04-types-and-foundation.md | 1050 | Core type system (Agent, Guardrail, Memory, File, RAG types), Zod schemas, configuration system, storage factory, MCP health check, provider resolution, extraction safeguard types, memory types |
| 05-agent-and-orchestration.md | 832+ | Agent factory (`createAgent` wrapping `@openai/agents`), orchestrator pattern (supervisor + parallel sub-agents + handoffs), tool registry, location enrichment tool, agent router, queue-based scaling, provider fallback, context budget. **Humanlikeness behaviors**: implicit reference resolution, response energy matching, conversation resumption, clarification patience model |
| 06-guardrails-and-safety.md | 1028 | Input guardrails, streaming output guardrails (tripwireTriggered), zero-leak buffered mode, guardrail factories (regex, keyword, LLM, external, composite), language guard (eld + two-stage), hate speech guard (obscenity + @2toad/profanity + LDNOOBW + Vietnamese lists), memory deletion guardrail, pipeline orchestrator |
| 07-memory-system.md | 1712+ | Three-layer model: Layer 1 (thread short-term, Postgres, 10 turns), Layer 2 (user short-term, cross-thread, Postgres), Layer 3 (long-term, SurrealDB facts/interactions/media). Fact extraction pipeline with safeguards (attribution, sarcasm, hypothetical, hallucination feedback loop). Memory recall tool. User memory control. Structured result memory. Rolling summary cap. Thread resurrection. Context window budget management. **Humanlikeness memory**: emotional context carry-forward (decaying state), communication style preferences, temporal fact markers (past/present/future), fact supersession (contradiction resolution with audit trail) |
| 08-document-processing.md | 879 | Upload pipeline (multimodal-first), document routing (direct ≤6 pages, indexed >6, RAG for large TXT), DOCX→PDF via LibreOffice, per-page summarization (blocking), raw text enrichment (background via Trigger.dev), file status state machine, S3 layout, page_index table, cleanup with stranded file recovery (TTL detects stale `summarizing`) |
| 09-rag-and-retrieval.md | 758 | page_index system, hybrid search with RRF (vector on summaries + vector on raw text + keyword tsvector), graceful degradation, query tool (searchDocument with server-side userId+threadId filter), page context assembly, structured citations (generateObject), cross-conversation RAG (scope: global/thread), large TXT RAG |
| 10-intent-and-routing.md | 692+ | Two-stage intent detection: Stage 1 embedding router (vector similarity classifier, Valkey cached), Stage 2 LLM intent validator + query rewriter (7 triggers, conversation-context rewriting in same generateObject call). IntentConfig (server-defined). Embedding router cache. Multi-intent + dependent multi-intent. Language validation piggybacked. Temporal expression resolution. Non-actionable detection (pleasantries/gibberish short-circuit). **Humanlikeness signals**: correction detection, frustration escalation detection, topic abandonment detection, proactive clarification (ambiguity signal) |
| 11-query-pipeline.md | 681 | Conditional query rewriting (REWRITE_TOOL, 7-trigger, source-specific strategies: HyDE, entity extraction, dense keywords). Source priority execution (parallel fan-out, weighted merge). RAGFlow integration. Two-stage rewrite model: LLM_INTENT does conversation-context rewriting, REWRITE_TOOL does source-specific strategies. Complementary, not competing. Circuit breaker wraps external API calls (per-call level, NOT pipeline-level) |
| 12-file-intelligence.md | 810 | Evidence bundle gate (sufficiency scoring + configurable thresholds), FileRegistry (temporal/ordinal/named resolution), file edge cases, visual grounding (multimodal LLM for charts/tables/images), per-document search tool (searchDocument), anti-hallucination architecture |
| 13-streaming-and-transport.md | 773 | SSE streaming layer (Runner.run() → SSE event translation), stream format boundary, session metadata delivery (traceId + threadId as first event), CTA streaming (createCTATool, hidden from client, max 3 per response), client SDK (@safeagent/client, offline queue), SSE event type reference |
| 14-server-implementation.md | 1018 | Thin server philosophy, startup sequence (JWT_SECRET production hard-fail), request lifecycle, route map, JWT auth (createAuthMiddleware, dev-bypass when no secret, production hard-fail), middleware stack, agent config, guardrail rules, MCP definitions, endpoints (SSE stream, upload, feedback, file CRUD, admin budget), error mapping, graceful shutdown, health endpoint, input validation, budget INCRBY model, OpenAPI docs |
| 15-tui-app.md | 606 | TUI app (OpenTUI Solid), component tree, command routing, agent integration, file upload flow, app shell, chat display, input component, command system (/help, /model, /clear, /quit, /upload) |
| 16-observability-and-eval.md | 834 | Langfuse self-hosted (ClickHouse + Redis + langfuse-web + langfuse-worker), trace hierarchy, Langfuse module (direct SDK), custom spans (guardrails + RAG), user feedback endpoint, Langfuse prompt management, eval/scoring config, self-test infrastructure (Promptfoo via Bun.spawn — automated, zero human intervention), PII filter via @logtape/redaction |
| 17-infrastructure.md | 1167 | Infrastructure stack (Postgres+pgvector via PgBouncer REQUIRED, SurrealDB, Valkey, S3/MinIO, Trigger.dev, LibreOffice), Docker Compose, API key pool, Valkey cache (fail-open per-instance acknowledged), cost tracking + budget enforcement (INCRBY pessimistic reservation with rollback/reconciliation), Trigger.dev integration, rate limiting (sliding window sorted sets), structured logging (LogTape), TTL cleanup, circuit breaker (wraps async functions, per-external-call level), health checks, graceful shutdown, capacity planning |
| 18-testing-strategy.md | 1087 | Testing pyramid, CI pipeline, test suite separation (unit: no secrets, integration: conditional skip), mock model pattern, test types (unit/integration/E2E/eval/load/adversarial/regression/property-based), QA policy. AUDIT_PLAN: 91 MH / 42 MN verification. AUDIT_SCOPE: 99 implementation tasks. Coverage map (every feature area has ≥1 test type, including 13 humanlikeness rows). JWT production enforcement test scenario |
| 19-execution-plan.md | 1126 | Batch timeline (0→FINAL), critical path, 11 parallel execution batches, dependency graph (Mermaid), batch parallelism visualization, agent dispatch map, FULL dependency matrix (103 tasks = 99 impl + 4 audit), barrel export convention, batch completion rules, new task registry (documents 05-12), 5 new humanlikeness tasks (FRUSTRATION_SIGNAL, STYLE_PREFERENCES, FACT_SUPERSESSION, RESPONSE_CALIBRATION, CLARIFICATION_MODEL) |
| 20-constraints-and-success.md | 635 | Core objective, concrete deliverables (7), definition of done (8 criteria + decision tree), 91 must-have requirements (15 sections incl. Humanlikeness), complete must-have → task ownership mapping (all 91 mapped), 42 must-not-have exclusions (3 sections), runtime clarification (Bun only), conventions (TDD, commit strategy, zero human intervention, integration test isolation), final verification pipeline (4 parallel audits), verification gates |

---

## Cross-Reference Chain Verification

### 1. JWT_SECRET Production Fail-Closed (Files 02, 14, 17, 18, 20)

| File | Location | Statement |
|------|----------|-----------|
| 02 | Env Variables Reference | Production (`NODE_ENV=production`): missing JWT_SECRET → hard startup refusal |
| 14 | Server Startup Sequence + JWT Auth section | Production: missing JWT_SECRET → refuse startup (hard fail, security boundary) |
| 17 | Infrastructure Stack | JWT_SECRET production hard-failure documented |
| 18 | QA Scenarios | JWT production enforcement test scenario included |
| 20 | MH_JWT_AUTH_REQUIRED | "in production (`NODE_ENV=production`) the server must refuse startup (hard fail — security boundary, no fallback)" |

**Status**: ✅ Consistent across all 5 files.

### 2. Rewrite Triggers (Files 10, 11, 19)

| File | Location | Count |
|------|----------|-------|
| 10 | LLM Intent Validator | 7 triggers for conversation-context rewriting |
| 11 | REWRITE_TOOL | 7-trigger conditional rewriting (source-specific strategies) |
| 19 | New Task Registry, REWRITE_TOOL entry | "7-trigger conditional rewriting" |

**Status**: ✅ Consistently 7 triggers across all 3 files.

### 3. Circuit Breaker Scope (Files 11, 17, overview)

| File | Statement |
|------|-----------|
| 11 | Circuit breaker wraps external API calls (per-call level, NOT pipeline-level) |
| 17 | `createCircuitBreaker` wraps async functions with configurable failure threshold + reset timeout (per-external-call) |
| overview | Circuit breaker listed in infrastructure layer |

**Status**: ✅ Consistent — per-external-call level, not pipeline-level.

### 4. Budget Model (Files 01, 14, 17)

| File | Statement |
|------|-----------|
| 01 | INCRBY-based pessimistic reservation. Counter accumulates spend; reservation increments optimistically, checks total vs limit, rolls back if exceeded |
| 14 | Budget INCRBY model — pessimistic reservation with rollback |
| 17 | Cost tracking: INCRBY pessimistic reservation with rollback/reconciliation |

**Status**: ✅ Consistent INCRBY model across all 3 files.

### 5. Two-Stage Rewrite Model (Files 10, 11)

| File | Stage | Role |
|------|-------|------|
| 10 | LLM_INTENT | Checks 7 triggers, does conversation-context rewriting in same generateObject call |
| 11 | REWRITE_TOOL | Applies source-specific strategies (HyDE, entity extraction, dense keywords) |

**Status**: ✅ Complementary, not competing. LLM_INTENT = conversation context, REWRITE_TOOL = source-specific.

---

## Counts Verification

| Item | Count | Source of Truth |
|------|-------|-----------------|
| Must-Have (MH) items | 91 | File 20: 15 sections (incl. Humanlikeness with 13 items), all counted |
| Must-Not-Have (MN) items | 42 | File 20: 3 sections (11 Bun/Library + 14 Architecture + 17 Out of Scope) |
| Implementation tasks | 99 | File 19: batch tables |
| Audit tasks | 4 | File 19: AUDIT_PLAN, AUDIT_CODE, AUDIT_QA, AUDIT_SCOPE |
| Total tasks | 103 | File 19: 99 + 4 |
| AUDIT_PLAN MH reference | 91 | File 18: "all 91 Must Have items" |
| AUDIT_PLAN MN reference | 42 | File 18: "all 42 Must NOT Have items" |
| AUDIT_SCOPE task reference | 99 | File 18: "all 99 implementation tasks" |
| Definition of Done criteria | 8 | File 20: LIB_TESTS through EVAL_DEMO |
| Concrete deliverables | 7 | File 20: safeagent, @safeagent/client, TUI, server, self-test, test suite, doc Q&A |
| Execution batches | 11 | File 19: 0, 0.5, 1, 2, 3, 4, 5, 6, 7, 8a, 8b, 9a, 9b, 10, FINAL |
| Rewrite triggers | 7 | Files 10, 11, 19 |

---

## Must-Have → Task Ownership (File 20 — Complete)

File 20 contains a complete mapping of all 91 must-haves to their owning implementation tasks, organized in 15 section tables matching the must-have sections:

- Agent Core (10 items) → AGENT_FACTORY, INPUT_GUARD, OUTPUT_GUARD, GUARD_FACTORY, MCP_CLIENT, SSE_STREAMING, GEMINI_GROUNDING, PROVIDER_HELPERS, CONFIG_DEFAULTS
- Language and Content Safety (5 items) → LANG_GUARD, HATE_SPEECH_GUARD, PKG_PUBLISH
- Memory (5 items) → SHORT_TERM_MEM, SURREALDB_CLIENT, FACT_EXTRACTION, MEMORY_RECALL, CORE_TYPES
- Auth & Transport (3 items) → JWT_AUTH, SSE_STREAMING, SCAFFOLD_LIB
- TUI (1 item) → TUI_SHELL + TUI_CHAT + TUI_INPUT + TUI_COMMANDS + TUI_UPLOAD + TUI_AGENT
- Evaluation (2 items) → SELF_TEST, EVAL_CONFIG
- Document Processing (10 items) → DOC_PIPELINE, RAG_INFRA, TRIGGER_TASKS, UPLOAD_PIPELINE, DOC_SEARCH
- Storage & Files (7 items) → FILE_STORAGE, UPLOAD_PIPELINE, TTL_CLEANUP, DOCKER_COMPOSE
- Observability (5 items) → LANGFUSE_MODULE, CUSTOM_SPANS, FEEDBACK_ENDPOINT, DOCKER_COMPOSE
- Data Layer (1 item) → STORAGE_WRAPPER
- CTA Streaming (4 items) → CTA_STREAMING, SERVER_AGENT_CFG
- Location Enrichment (4 items) → LOCATION_TOOL
- Infrastructure (6 items) → RATE_LIMITING, STRUCT_LOGGING, FILE_CRUD, TTL_CLEANUP, CIRCUIT_BREAKER, JWT_AUTH
- Cross-Cutting (15 items) → Various (CROSS_CONV_RAG, ADMIN_API, PROMPT_MGMT, ZERO_LEAK_GUARD, CLIENT_SDK, SURREALDB_CLIENT, CONFIG_DEFAULTS, CORE_TYPES, SCAFFOLD_LIB, SERVER_ROUTES, SMOKE_TESTS, PKG_PUBLISH)
- Humanlikeness (13 items) → LLM_INTENT, FACT_EXTRACTION, FRUSTRATION_SIGNAL, QUERY_REPLAY, STYLE_PREFERENCES, FACT_SUPERSESSION, MEMORY_RECALL, CONTEXT_BUDGET, RESPONSE_CALIBRATION, THREAD_RESURRECTION, CLARIFICATION_MODEL

---

## Dependency Matrix Completeness (File 19)

The full dependency matrix contains 103 rows (99 implementation + 4 audit tasks). All tasks from batch tables appear in the matrix, including:
- USER_SHORTTERM_MEM (batch 3)
- STRUCTURED_RESULT_MEM (batch 5)
- MEMORY_CONTROL (batch 6)
- DEPENDENT_INTENT (batch 8b)
- STYLE_PREFERENCES (batch 8a)
- FACT_SUPERSESSION (batch 8a)
- RESPONSE_CALIBRATION (batch 8a)
- FRUSTRATION_SIGNAL (batch 8b)
- CLARIFICATION_MODEL (batch 8b)

---

## Key Architectural Decisions

1. **Framework**: `@openai/agents` with `aisdk()` bridge for AI SDK model compatibility
2. **Memory**: Three-layer model (thread → user → long-term SurrealDB)
3. **Storage**: Postgres+pgvector (PgBouncer REQUIRED), SurrealDB via surqlize, Valkey cache, S3/MinIO
4. **Transport**: SSE only (no WebSocket for client API; internal deps may use WS)
5. **Document processing**: Multimodal-first (PDF pages → Gemini directly, summaries are search indexes only)
6. **Query pipeline**: Embedding router → LLM intent → source router → parallel fan-out → weighted merge
7. **Guardrails**: Input (server-defined) + streaming output (tripwireTriggered) + zero-leak buffered mode
8. **Auth**: JWT with production fail-closed, dev-bypass in non-production
9. **Budget**: INCRBY pessimistic reservation (accumulate spend, check vs limit, rollback if exceeded)
10. **Eval**: Promptfoo via Bun.spawn (automated, zero human intervention)
11. **Observability**: Langfuse self-hosted (direct SDK, not OTel wrapper)
12. **ORM**: Drizzle (drizzle-orm/bun-sql) for Postgres, surqlize for SurrealDB
13. **Scale**: 10M total users (1% DAU = 100K daily active, burst headroom)
14. **Humanlikeness**: 13 engine-level behaviors for natural conversation — correction handling, emotional context, frustration detection, repeated question differentiation, style memory, temporal facts, fact supersession, implicit references, response energy matching, conversation resumption, clarification patience, topic abandonment, proactive clarification

---

## Coverage Map (File 18)

Every feature area has at least one test type marked (✓). No unmarked rows.

---

## Conventions (File 20)

- TDD: RED-GREEN-REFACTOR for every task
- Commit after every completed task (atomic commits)
- Zero human intervention for all verification
- Integration tests use conditional skip (skipIf)
- Unit tests require zero secrets
- Subpath barrel export convention (each task's responsibility)
- UPPER_SNAKE task IDs (no numeric IDs)
- ESM-only, Bun-only, no version pinning

---

*This brief was generated from direct tool verification of all 21 plan files.*
