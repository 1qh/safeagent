# Task Verification Index

This index maps every execution task to its behavioral test-assertion owner. It is derived from the per-task routing index in [Execution Plan](./execution.md).

Scheduling and dependency authority remains in [Execution Plan](./execution.md). This file defines verification ownership only.

---

## How to Use

1. Find your task ID in this document.
2. Open the listed test owner file.
3. Validate behavioral assertions in that file after completing task acceptance criteria and QA scenarios.
4. Complete required gates from [Quality Gates](./quality-gates.md).

---

## Verification Ownership by Test File

### `agents.md` (11 tasks)

- `AGENT_FACTORY` — batch `AGENT_PIPELINE_BATCH`; task spec in `agents.md`
- `AGENT_ROUTER` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `agents.md`
- `DEPENDENT_INTENT` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `agents.md`
- `GEMINI_GROUNDING` — batch `INTEGRATION_BATCH`; task spec in `agents.md`
- `GENERATIVE_UI` — batch `ENDPOINTS_BARREL_BATCH`; task spec in `agents.md`
- `LOCATION_TOOL` — batch `INTEGRATION_BATCH`; task spec in `agents.md`
- `MCP_CLIENT` — batch `FOUNDATION_B_BATCH`; task spec in `agents.md`
- `ORCHESTRATOR` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `agents.md`
- `PROVIDER_FALLBACK` — batch `FOUNDATION_B_BATCH`; task spec in `agents.md`
- `RESPONSE_CALIBRATION` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `agents.md`
- `SUBAGENT_FACTORY` — batch `SERVER_ROUTES_SUBAGENT_BATCH`; task spec in `agents.md`

### `ai-operations.md` (1 tasks)

- `AI_OPERATIONS` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `ai-operations.md`

### `api-governance.md` (1 tasks)

- `API_GOVERNANCE` — batch `FINAL_AUDIT_BATCH`; task spec in `api-governance.md`

### `coding-standards.md` (1 tasks)

- `CODE_STANDARDS` — batch `SCAFFOLDING_BATCH`; task spec in `coding-standards.md`

### `conversation.md` (12 tasks)

- `ATTRIBUTE_NEGATION` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `conversation.md`
- `CLARIFICATION_MODEL` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `conversation.md`
- `CONVERSATION_INTELLIGENCE` — batch `E2E_DEPLOY_BATCH`; task spec in `conversation.md`
- `EMBED_ROUTER` — batch `INTEGRATION_BATCH`; task spec in `conversation.md`
- `FRUSTRATION_SIGNAL` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `conversation.md`
- `LLM_INTENT` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `conversation.md`
- `NON_ACTIONABLE_DETECT` — batch `SELFTEST_MIDINTEGRATION_BATCH`; task spec in `conversation.md`
- `PREFETCH_COORD` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `conversation.md`
- `QUERY_REPLAY` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `conversation.md`
- `REWRITE_STRATEGIES` — batch `FOUNDATION_B_BATCH`; task spec in `conversation.md`
- `REWRITE_TOOL` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `conversation.md`
- `SOURCE_ROUTER` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `conversation.md`

### `demos.md` (2 tasks)

- `DEMO_MOBILE` — batch `FRONTEND_DEMOS_BATCH`; task spec in `demos.md`
- `DEMO_WEB` — batch `FRONTEND_DEMOS_BATCH`; task spec in `demos.md`

### `developer-experience.md` (1 tasks)

- `DEVELOPER_EXPERIENCE` — batch `FRONTEND_DEMOS_BATCH`; task spec in `developer-experience.md`

### `documentation.md` (2 tasks)

- `DOCS_CONTENT` — batch `FRONTEND_DEMOS_BATCH`; task spec in `documentation.md`
- `DOCS_SITE` — batch `E2E_DEPLOY_BATCH`; task spec in `documentation.md`

### `documents.md` (4 tasks)

- `DOC_PIPELINE` — batch `AGENT_PIPELINE_BATCH`; task spec in `documents.md`
- `FILE_STORAGE` — batch `AGENT_PIPELINE_BATCH`; task spec in `documents.md`
- `SPIKE_RAG_DEPS` — batch `RAG_VALIDATION_BATCH`; task spec in `documents.md`
- `UPLOAD_PIPELINE` — batch `SELFTEST_MIDINTEGRATION_BATCH`; task spec in `documents.md`

### `durable-execution.md` (1 tasks)

- `DURABLE_EXECUTION` — batch `INTEGRATION_BATCH`; task spec in `durable-execution.md`

### `extensibility.md` (1 tasks)

- `EXTENSIBILITY_INFRA` — batch `CONFIG_GUARDS_BATCH`; task spec in `extensibility.md`

### `foundation.md` (9 tasks)

- `BARREL_EXPORTS` — batch `ENDPOINTS_BARREL_BATCH`; task spec in `foundation.md`
- `CONFIG_DEFAULTS` — batch `CONFIG_GUARDS_BATCH`; task spec in `foundation.md`
- `CORE_TYPES` — batch `FOUNDATION_A_BATCH`; task spec in `foundation.md`
- `MCP_HEALTH` — batch `FOUNDATION_A_BATCH`; task spec in `foundation.md`
- `PROVIDER_HELPERS` — batch `FOUNDATION_A_BATCH`; task spec in `foundation.md`
- `SCAFFOLD_LIB` — batch `SCAFFOLDING_BATCH`; task spec in `foundation.md`
- `SPIKE_CORE_STACK` — batch `BLOCKING_SPIKE_BATCH`; task spec in `foundation.md`
- `STORAGE_WRAPPER` — batch `FOUNDATION_A_BATCH`; task spec in `foundation.md`
- `ZOD_SCHEMAS` — batch `FOUNDATION_B_BATCH`; task spec in `foundation.md`

### `frontend-sdk.md` (7 tasks)

- `FRONTEND_CLI` — batch `E2E_DEPLOY_BATCH`; task spec in `frontend-sdk.md`
- `REACT_HOOKS` — batch `SERVER_ROUTES_SUBAGENT_BATCH`; task spec in `frontend-sdk.md`
- `RN_COMPONENTS` — batch `ENDPOINTS_BARREL_BATCH`; task spec in `frontend-sdk.md`
- `SCAFFOLD_FRONTEND` — batch `FOUNDATION_A_BATCH`; task spec in `foundation.md`
- `STORYBOOK_FRONTEND` — batch `E2E_DEPLOY_BATCH`; task spec in `frontend-sdk.md`
- `TRACE_UI` — batch `E2E_DEPLOY_BATCH`; task spec in `frontend-sdk.md`
- `WEB_COMPONENTS` — batch `ENDPOINTS_BARREL_BATCH`; task spec in `frontend-sdk.md`

### `guardrails.md` (8 tasks)

- `GUARD_FACTORY` — batch `CONFIG_GUARDS_BATCH`; task spec in `guardrails.md`
- `GUARD_PIPELINE` — batch `INTEGRATION_BATCH`; task spec in `guardrails.md`
- `HATE_SPEECH_GUARD` — batch `INTEGRATION_BATCH`; task spec in `guardrails.md`
- `INPUT_GUARD` — batch `CONFIG_GUARDS_BATCH`; task spec in `guardrails.md`
- `INPUT_VALIDATION` — batch `SELFTEST_MIDINTEGRATION_BATCH`; task spec in `guardrails.md`
- `LANG_GUARD` — batch `INTEGRATION_BATCH`; task spec in `guardrails.md`
- `OUTPUT_GUARD` — batch `CONFIG_GUARDS_BATCH`; task spec in `guardrails.md`
- `ZERO_LEAK_GUARD` — batch `INTEGRATION_BATCH`; task spec in `guardrails.md`

### `infrastructure.md` (9 tasks)

- `CIRCUIT_BREAKER` — batch `FOUNDATION_B_BATCH`; task spec in `infrastructure.md`
- `COST_TRACKING` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `infrastructure.md`
- `DOCKER_COMPOSE` — batch `FOUNDATION_A_BATCH`; task spec in `infrastructure.md`
- `KEY_POOL` — batch `FOUNDATION_B_BATCH`; task spec in `infrastructure.md`
- `RATE_LIMITING` — batch `INTEGRATION_BATCH`; task spec in `infrastructure.md`
- `STRUCT_LOGGING` — batch `FOUNDATION_A_BATCH`; task spec in `infrastructure.md`
- `TRIGGER_TASKS` — batch `SELFTEST_MIDINTEGRATION_BATCH`; task spec in `infrastructure.md`
- `TTL_CLEANUP` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `infrastructure.md`
- `VALKEY_CACHE` — batch `FOUNDATION_B_BATCH`; task spec in `infrastructure.md`

### `memory.md` (13 tasks)

- `CONTEXT_BUDGET` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `memory.md`
- `EXTRACTION_SAFEGUARDS` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `memory.md`
- `FACT_EXTRACTION` — batch `INTEGRATION_BATCH`; task spec in `memory.md`
- `FACT_SUPERSESSION` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `memory.md`
- `MEMORY_CONTROL` — batch `INTEGRATION_BATCH`; task spec in `memory.md`
- `MEMORY_RECALL` — batch `INTEGRATION_BATCH`; task spec in `memory.md`
- `SHORT_TERM_MEM` — batch `FOUNDATION_B_BATCH`; task spec in `memory.md`
- `STRUCTURED_RESULT_MEM` — batch `AGENT_PIPELINE_BATCH`; task spec in `memory.md`
- `STYLE_PREFERENCES` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `memory.md`
- `SUMMARY_CAP` — batch `CONFIG_GUARDS_BATCH`; task spec in `memory.md`
- `SURREALDB_CLIENT` — batch `FOUNDATION_A_BATCH`; task spec in `memory.md`
- `THREAD_RESURRECTION` — batch `SELFTEST_MIDINTEGRATION_BATCH`; task spec in `memory.md`
- `USER_SHORTTERM_MEM` — batch `FOUNDATION_B_BATCH`; task spec in `memory.md`

### `monitoring.md` (2 tasks)

- `INCIDENT_PROCEDURES` — batch `FRONTEND_DEMOS_BATCH`; task spec in `monitoring.md`
- `MONITORING_INFRA` — batch `E2E_DEPLOY_BATCH`; task spec in `monitoring.md`

### `observability.md` (5 tasks)

- `CUSTOM_SPANS` — batch `SELFTEST_MIDINTEGRATION_BATCH`; task spec in `observability.md`
- `EVAL_CONFIG` — batch `INTEGRATION_BATCH`; task spec in `observability.md`
- `LANGFUSE_MODULE` — batch `FOUNDATION_B_BATCH`; task spec in `observability.md`
- `PROMPT_MGMT` — batch `INTEGRATION_BATCH`; task spec in `observability.md`
- `SELF_TEST` — batch `SELFTEST_MIDINTEGRATION_BATCH`; task spec in `observability.md`

### `release-pipeline.md` (2 tasks)

- `CI_PIPELINE` — batch `FOUNDATION_B_BATCH`; task spec in `release-pipeline.md`
- `RELEASE_PIPELINE` — batch `E2E_DEPLOY_BATCH`; task spec in `release-pipeline.md`

### `retrieval.md` (8 tasks)

- `CROSS_CONV_RAG` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `retrieval.md`
- `DOC_SEARCH` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `retrieval.md`
- `EVIDENCE_GATE` — batch `SELFTEST_MIDINTEGRATION_BATCH`; task spec in `retrieval.md`
- `FILE_REGISTRY` — batch `CONFIG_GUARDS_BATCH`; task spec in `retrieval.md`
- `RAGFLOW_CLIENT` — batch `INTEGRATION_BATCH`; task spec in `retrieval.md`
- `RAG_FEEDBACK_LOOP` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `retrieval.md`
- `RAG_INFRA` — batch `INTEGRATION_BATCH`; task spec in `retrieval.md`
- `VISUAL_GROUNDING` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `retrieval.md`

### `security-compliance.md` (2 tasks)

- `MULTI_TENANT_CONFIG` — batch `CONFIG_GUARDS_BATCH`; task spec in `security-compliance.md`
- `SECURITY_COMPLIANCE` — batch `E2E_DEPLOY_BATCH`; task spec in `security-compliance.md`

### `server.md` (10 tasks)

- `ADMIN_API` — batch `ENDPOINTS_BARREL_BATCH`; task spec in `server.md`
- `FEEDBACK_ENDPOINT` — batch `ENDPOINTS_BARREL_BATCH`; task spec in `server.md`
- `FILE_CRUD` — batch `ENDPOINTS_BARREL_BATCH`; task spec in `server.md`
- `JWT_AUTH` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `server.md`
- `SCAFFOLD_SERVER` — batch `SCAFFOLDING_BATCH`; task spec in `server.md`
- `SERVER_AGENT_CFG` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `server.md`
- `SERVER_GUARDRAILS` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `server.md`
- `SERVER_MCP` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `server.md`
- `SERVER_ROUTES` — batch `SERVER_ROUTES_SUBAGENT_BATCH`; task spec in `server.md`
- `UPLOAD_ENDPOINT` — batch `ENDPOINTS_BARREL_BATCH`; task spec in `server.md`

### `testing.md` (8 tasks)

- `AUDIT_CODE` — batch `FINAL_AUDIT_BATCH`; task spec in `testing.md`
- `AUDIT_PLAN` — batch `FINAL_AUDIT_BATCH`; task spec in `testing.md`
- `AUDIT_QA` — batch `FINAL_AUDIT_BATCH`; task spec in `testing.md`
- `AUDIT_SCOPE` — batch `FINAL_AUDIT_BATCH`; task spec in `testing.md`
- `E2E_TESTS` — batch `E2E_DEPLOY_BATCH`; task spec in `testing.md`
- `LOAD_TESTS` — batch `E2E_DEPLOY_BATCH`; task spec in `testing.md`
- `PKG_PUBLISH` — batch `E2E_DEPLOY_BATCH`; task spec in `testing.md`
- `SMOKE_TESTS` — batch `E2E_DEPLOY_BATCH`; task spec in `testing.md`

### `transport.md` (3 tasks)

- `CLIENT_SDK` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `transport.md`
- `CTA_STREAMING` — batch `INTEGRATION_BATCH`; task spec in `transport.md`
- `SSE_STREAMING` — batch `INTEGRATION_BATCH`; task spec in `transport.md`

### `tui.md` (6 tasks)

- `TUI_AGENT` — batch `SERVER_TUI_PIPELINE_BATCH`; task spec in `tui.md`
- `TUI_CHAT` — batch `FOUNDATION_B_BATCH`; task spec in `tui.md`
- `TUI_COMMANDS` — batch `FOUNDATION_B_BATCH`; task spec in `tui.md`
- `TUI_INPUT` — batch `FOUNDATION_B_BATCH`; task spec in `tui.md`
- `TUI_SHELL` — batch `FOUNDATION_A_BATCH`; task spec in `tui.md`
- `TUI_UPLOAD` — batch `EXTENDED_INTEGRATION_BATCH`; task spec in `tui.md`

---

## Coverage Check

- Total tasks covered in this index: 129
- Duplicate task IDs in this index: 0
- Expected source: [Execution Plan](./execution.md) routing index
