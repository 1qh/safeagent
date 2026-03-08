# 16 — Testing

> **Scope**: Complete testing methodology covering eight testing types, suite separation for CI behavior, mock-model patterns, QA policy, coverage linkage to requirement IDs, and task specifications for E2E verification, publish preparation, smoke validation, load benchmarking, and final audits.
>
> **Tasks**: E2E_TESTS, PKG_PUBLISH, SMOKE_TESTS, LOAD_TESTS, AUDIT_PLAN, AUDIT_CODE, AUDIT_QA, AUDIT_SCOPE.

## Table of Contents

- [Testing Philosophy](#testing-philosophy)
- [Testing Pyramid](#testing-pyramid)
- [Test Execution Flow](#test-execution-flow)
- [CI Pipeline](#ci-pipeline)
- [Test Suite Separation](#test-suite-separation)
- [Mock Model Pattern](#mock-model-pattern)
- [Unit Tests](#unit-tests)
- [Integration Tests](#integration-tests)
- [End-to-End Tests (E2E_TESTS)](#end-to-end-tests-e2e_tests)
- [Eval Tests](#eval-tests)
- [Load Tests (LOAD_TESTS)](#load-tests-load_tests)
- [Adversarial Tests](#adversarial-tests)
- [Regression Tests](#regression-tests)
- [Property-Based Tests](#property-based-tests)
- [Chaos and Failure Testing](#chaos-and-failure-testing)
- [Contract Testing](#contract-testing)
- [Mutation Testing](#mutation-testing)
- [Snapshot and Golden-File Testing](#snapshot-and-golden-file-testing)
- [Streaming-Specific Tests](#streaming-specific-tests)
- [Per-Module Test Specifications](#per-module-test-specifications)
- [Development Seed Data](#development-seed-data)
- [QA Policy](#qa-policy)
- [Task Specifications](#task-specifications)
- [Coverage Map](#coverage-map)
- [Extended Coverage Map](#extended-coverage-map)
- [Requirement-Level Coverage (MH_*, MN_*)](#requirement-level-coverage-mh_-mn_)
- [Verification Strategy Summary](#verification-strategy-summary)
- [Cross-Plan References](#cross-plan-references)
- [External References](#external-references)

## Testing Philosophy

> **Core Principle**: We sacrifice speed of development to ensure nothing breaks when we move forward. Every logic path, every feature boundary, and every failure mode has comprehensive automated verification before code is considered complete.

This philosophy governs all testing decisions across both repositories. It rests on seven pillars.

### Pillar 1 — Incremental Confidence

Every implementation task delivers tests and implementation together. TDD discipline (RED → GREEN → REFACTOR) is non-negotiable. When a module ships, its test coverage ships simultaneously. The system grows in lockstep: no feature exists without corresponding verification. Moving forward means every step lands on proven ground.

### Pillar 2 — Layered Defense

No single testing layer catches all failure modes. Unit tests catch logic errors. Integration tests catch wiring errors. End-to-end tests catch behavioral regressions. Eval tests catch quality degradation. Adversarial tests catch safety bypasses. Property-based tests catch invariant violations. Chaos tests catch resilience gaps. Every critical path is covered by at least three independent layers.

```mermaid
flowchart LR
    subgraph DEFENSE_LAYERS["Layered Defense Model"]
        UNIT_LAYER["Unit\nlogic errors"]
        INTEGRATION_LAYER["Integration\nwiring errors"]
        END_TO_END_LAYER["End-to-End\nbehavioral regression"]
        EVAL_LAYER["Eval\nquality degradation"]
        ADVERSARIAL_LAYER["Adversarial\nsafety bypass"]
        PROPERTY_LAYER["Property-Based\ninvariant violation"]
        CHAOS_LAYER["Chaos\nresilience gaps"]
    end

    UNIT_LAYER --> INTEGRATION_LAYER --> END_TO_END_LAYER --> EVAL_LAYER
    EVAL_LAYER --> ADVERSARIAL_LAYER --> PROPERTY_LAYER --> CHAOS_LAYER

    style UNIT_LAYER fill:#22aa44,color:#fff
    style INTEGRATION_LAYER fill:#33bb55,color:#fff
    style END_TO_END_LAYER fill:#4488cc,color:#fff
    style EVAL_LAYER fill:#5599dd,color:#fff
    style ADVERSARIAL_LAYER fill:#ff4444,color:#fff
    style PROPERTY_LAYER fill:#8866cc,color:#fff
    style CHAOS_LAYER fill:#ff9900,color:#fff
```

### Pillar 3 — Deterministic Core, Stochastic Edge

The system isolates deterministic orchestration (routing, retries, guardrails, state machines, configuration merging, budget arithmetic, rate limiting) into testable code layers where assertions are exact. LLM outputs are inherently non-deterministic and require eval-based validation rather than assertion-based testing. These two paradigms coexist but never conflate. Deterministic logic uses strict equality assertions. Stochastic output uses threshold-based quality scoring.

### Pillar 4 — Eval as First-Class Citizen

AI agent systems fail differently than traditional software. Output quality degradation, reasoning drift, and emergent misbehavior do not manifest as exceptions. They manifest as subtle changes in response quality that only evaluation can detect. Evaluation datasets are treated as code: version-controlled, reviewed, and maintained. Eval results are compared against defined thresholds, and regressions are failures. Golden datasets grow from production failures, not synthetic scenarios.

### Pillar 5 — Test at the Right Speed

Run cheap tests on every commit. Run expensive tests nightly. Run chaos experiments in staging. Match test execution cost to failure impact. Unit tests gate every push. Integration tests gate release readiness. Eval tests gate quality certification. Load tests gate deployment confidence. This stratification preserves developer velocity while maintaining comprehensive coverage.

```mermaid
flowchart TD
    subgraph TEST_SPEED["Test Speed Stratification"]
        EVERY_COMMIT["Every Commit\nunit tests and property tests\nseconds"]
        EVERY_PR["Every Pull Request\nintegration tests and contract tests\nminutes"]
        NIGHTLY["Nightly\neval tests and mutation tests\ntens of minutes"]
        STAGING["Staging\nchaos experiments and load tests\nhours"]
        GAME_DAY["Game Day\nfull chaos and production-scale load\nscheduled"]
    end

    EVERY_COMMIT --> EVERY_PR --> NIGHTLY --> STAGING --> GAME_DAY

    style EVERY_COMMIT fill:#22aa44,color:#fff
    style EVERY_PR fill:#33bb55,color:#fff
    style NIGHTLY fill:#4488cc,color:#fff
    style STAGING fill:#ff9900,color:#fff
    style GAME_DAY fill:#ff4444,color:#fff
```

### Pillar 6 — Contract as Boundary

The library and server exist in separate repositories with a clean interface boundary. Consumer-driven contract tests verify that interface changes propagate correctly without requiring full integration test execution. Schema validation catches mismatches at the unit test level. The library defines what it provides. The server defines what it expects. Contract tests verify these promises align.

### Pillar 7 — Embrace Non-Determinism

Rather than fighting LLM variability, testing strategies account for it. Snapshot structure, not content. Evaluate quality, not exact correctness. Use golden datasets with property assertions rather than string matching. Accept that streaming output varies and test format compliance and behavioral invariants instead. For every non-deterministic output, define what properties must hold regardless of the specific text generated.

### The Testing Tax

Every test incurs maintenance cost that compounds over time. The testing philosophy manages this tax deliberately:

- **Unit tests**: near-zero maintenance cost, highest value per line. Write liberally.
- **Integration tests**: moderate maintenance cost, essential for external boundary validation. Write selectively for real integration points.
- **End-to-end tests**: highest maintenance cost, highest confidence per test. Write for critical user flows only.
- **Eval tests**: ongoing curation cost as production reveals new failure modes. Budget for eval maintenance as a continuous activity.
- **Load tests**: minimal maintenance, high infrastructure cost. Run on demand, not continuously.
- **Adversarial tests**: growing corpus, low individual cost. Expand continuously as new attack vectors emerge.

The goal is not maximum test count but maximum confidence per maintenance dollar. Intelligent test selection and layered coverage outperform naive coverage expansion.

## Testing Pyramid

The project uses fourteen testing types organized into three tiers by execution frequency and cost. Every module has unit coverage; critical paths are covered by at least three layers.

```mermaid
graph TB
    subgraph FAST_TIER["Fast Tier — Every Commit"]
        direction TB
        UNIT_TESTS["Unit Tests\nfastest, cheapest, most numerous"]
        PROPERTY_TESTS["Property-Based Tests\nfuzzed inputs and invariants"]
        CONTRACT_TESTS["Contract Tests\nlibrary-server interface"]
        SNAPSHOT_TESTS["Snapshot Tests\nstructural and quality-score"]
        REGRESSION_TESTS["Regression Tests\ncaptured failures do not recur"]
    end

    subgraph MEDIUM_TIER["Medium Tier — Every PR or Nightly"]
        direction TB
        INTEGRATION_TESTS["Integration Tests\nreal provider calls, conditional skip"]
        E2E_TESTS["End-to-End Tests\nfull flow across repositories"]
        EVAL_TESTS["Eval Tests\nPromptfoo plus custom evaluation scorers"]
        ADVERSARIAL_TESTS["Adversarial Tests\nguardrail bypass and injection"]
        MUTATION_TESTS["Mutation Tests\nsafety-critical path validation"]
        STREAMING_TESTS["Streaming Tests\nmid-stream failure and backpressure"]
    end

    subgraph EXPENSIVE_TIER["Expensive Tier — Staging and Game Day"]
        direction TB
        LOAD_TESTS_LAYER["Load Tests\nstreaming, concurrency, latency"]
        CHAOS_TESTS["Chaos Tests\nsystematic failure injection"]
    end

    FAST_TIER --> MEDIUM_TIER --> EXPENSIVE_TIER

    style UNIT_TESTS fill:#22aa44,color:#fff
    style PROPERTY_TESTS fill:#22aa44,color:#fff
    style CONTRACT_TESTS fill:#22aa44,color:#fff
    style SNAPSHOT_TESTS fill:#22aa44,color:#fff
    style REGRESSION_TESTS fill:#22aa44,color:#fff
    style INTEGRATION_TESTS fill:#4488cc,color:#fff
    style E2E_TESTS fill:#4488cc,color:#fff
    style EVAL_TESTS fill:#5599dd,color:#fff
    style ADVERSARIAL_TESTS fill:#ff4444,color:#fff
    style MUTATION_TESTS fill:#aa44aa,color:#fff
    style STREAMING_TESTS fill:#4488cc,color:#fff
    style LOAD_TESTS_LAYER fill:#ff9900,color:#fff
    style CHAOS_TESTS fill:#ff9900,color:#fff
```

## Test Execution Flow

All suites run through the Bun test runner flow. Selection, discovery, execution, reporting, and evidence capture follow one shared lifecycle.

```mermaid
flowchart LR
    RUN_REQUEST["Run tests"]
    SCOPE_FILTER["Filter by scope or pattern"]
    TEST_DISCOVERY["Discover test files"]
    TEST_EXECUTION["Execute describe and it blocks"]
    RESULT_REPORT["Report pass, fail, skip\nand coverage when enabled"]
    EVIDENCE_CAPTURE["Save evidence artifacts"]

    RUN_REQUEST --> SCOPE_FILTER --> TEST_DISCOVERY --> TEST_EXECUTION --> RESULT_REPORT --> EVIDENCE_CAPTURE

    style RUN_REQUEST fill:#333,color:#fff
    style RESULT_REPORT fill:#22aa44,color:#fff
```

## CI Pipeline

CI follows strict order. Unit tests run first without secrets. Integration tests run only when keys exist. End-to-end and eval stages run after functional gates are green.

```mermaid
flowchart TD
    subgraph UNIT_STAGE["Unit Tests"]
        UNIT_RUN["Run unit tests\nno secrets required"]
        UNIT_CHECK{"All pass?"}
        UNIT_RUN --> UNIT_CHECK
    end

    subgraph INTEGRATION_STAGE["Integration Tests"]
        KEY_GATE{"Required keys\navailable?"}
        INTEGRATION_SKIP["Conditional skip\nclean and intentional"]
        INTEGRATION_RUN["Run integration tests\nreal provider calls"]
        INTEGRATION_CHECK{"All pass?"}
        KEY_GATE -->|No| INTEGRATION_SKIP
        KEY_GATE -->|Yes| INTEGRATION_RUN --> INTEGRATION_CHECK
    end

    subgraph END_TO_END_STAGE["End-to-End Tests"]
        END_TO_END_RUN_STAGE["Run full-flow end-to-end tests"]
        END_TO_END_CHECK_STAGE{"All pass?"}
        END_TO_END_RUN_STAGE --> END_TO_END_CHECK_STAGE
    end

    subgraph EVAL_STAGE["Eval Tests"]
        EVAL_RUN_STAGE["Promptfoo external tool\nplus custom evaluation scorers"]
        EVAL_CHECK_STAGE{"Thresholds met?"}
        EVAL_RUN_STAGE --> EVAL_CHECK_STAGE
    end

    UNIT_STAGE -->|Pass| INTEGRATION_STAGE
    INTEGRATION_STAGE -->|Pass or Skip| END_TO_END_STAGE
    END_TO_END_STAGE -->|Pass| EVAL_STAGE

    UNIT_CHECK -->|Fail| ABORT_UNIT["Abort"]
    INTEGRATION_CHECK -->|Fail| ABORT_INTEGRATION["Abort"]
    END_TO_END_CHECK_STAGE -->|Fail| ABORT_END_TO_END["Abort"]
    EVAL_CHECK_STAGE -->|Fail| ABORT_EVAL["Abort"]

    style ABORT_UNIT fill:#ff4444,color:#fff
    style ABORT_INTEGRATION fill:#ff4444,color:#fff
    style ABORT_END_TO_END fill:#ff4444,color:#fff
    style ABORT_EVAL fill:#ff4444,color:#fff
    style UNIT_RUN fill:#22aa44,color:#fff
    style INTEGRATION_RUN fill:#33bb55,color:#fff
    style END_TO_END_RUN_STAGE fill:#4488cc,color:#fff
    style EVAL_RUN_STAGE fill:#5599dd,color:#fff
```

## Test Suite Separation

This is the core testing infrastructure rule: if a unit test fails because of missing secrets, that is a test design bug.

### Boundaries

```mermaid
flowchart TD
    subgraph UNIT_BOUNDARY["Unit Tests - Zero Secrets"]
        UNIT_MODULE_LOGIC["Module logic"]
        UNIT_TYPE_CONVERSION["Type conversions"]
        UNIT_CONFIG_MERGE["Config merging"]
        UNIT_GUARD_DETECTION["Guardrail detection logic"]
        UNIT_STREAM_TRANSFORMS["Stream transformers"]
        UNIT_MOCK_RESPONSES["Mock model responses"]
        UNIT_PURE_FUNCTIONS["Pure functions"]
    end

    subgraph INTEGRATION_BOUNDARY["Integration Tests - Secrets Required"]
        INTEGRATION_REAL_LLM["Real model calls"]
        INTEGRATION_REAL_EMBEDDINGS["Real embedding generation"]
        INTEGRATION_DB_ROUND_TRIPS["Database round-trips"]
        INTEGRATION_STORAGE_ROUND_TRIPS["Object storage upload and download"]
        INTEGRATION_EXTERNAL_GUARDS["External API guardrails"]
        INTEGRATION_TRACING["Observability trace creation"]
    end

    subgraph END_TO_END_BOUNDARY["End-to-End Tests - Full Environment"]
        END_TO_END_AGENT_FLOW["Agent create to stream to consume"]
        END_TO_END_SERVER_FLOW["Server start to stream to auth"]
        END_TO_END_DOCUMENT_FLOW["Upload to process to query to cite"]
        END_TO_END_MEMORY_FLOW["Extract to store to recall"]
        END_TO_END_GUARDRAIL_FLOW["Trigger to tripwire to fallback"]
    end

    UNIT_BOUNDARY -.->|"never crosses"| INTEGRATION_BOUNDARY
    INTEGRATION_BOUNDARY -.->|"never crosses"| END_TO_END_BOUNDARY

    style UNIT_BOUNDARY fill:#e8f5e9,stroke:#2e7d32
    style INTEGRATION_BOUNDARY fill:#fff3e0,stroke:#ef6c00
    style END_TO_END_BOUNDARY fill:#e3f2fd,stroke:#1565c0
```

### Rules

**Unit tests** run with zero secrets and no external dependencies. Model behavior is mocked with the AI SDK test mock model. External systems are replaced with stubs or fakes.

**Integration tests** require real keys. Primary provider key is mandatory, secondary keys optional. Every integration test file uses conditional skip at the outermost describe block. A plain outer describe in integration scope is a test infrastructure defect.

**End-to-end tests** require complete environment setup: data services, object storage, cache, and at least one provider key. End-to-end suites run in dedicated scope and are not part of the default invocation.

**Spike validation** is integration by definition and may require keys. After the spike, all ongoing implementation must keep unit tests passing with zero keys.

## Mock Model Pattern

All unit tests use the AI SDK test mock model from AI SDK test utilities. The mock supports text streaming, tool calls, and structured outputs without external traffic.

```mermaid
flowchart TD
    subgraph MOCK_SETUP["Mock model setup"]
        MOCK_IMPORT["Import MockLanguageModel"]
        MOCK_CONFIGURE["Configure doStream or doGenerate"]
        MOCK_STREAM_DEFINE["Define ReadableStream\nwith text deltas"]
        MOCK_TOOL_DEFINE["Define tool-call chunks"]
    end

    subgraph MOCK_USAGE["Usage in tests"]
        MOCK_AGENT_CREATE["agent factory with mock model"]
        MOCK_RUNNER_CALL["Agent execution for stream or non-stream"]
        MOCK_ASSERTIONS["Assert chunks, tool calls, text output"]
    end

    subgraph MOCK_PATTERNS["Mock patterns"]
        MOCK_TEXT_PATTERN["Text response pattern\ntext deltas then close"]
        MOCK_TOOL_PATTERN["Tool call pattern\ndelta to call to result"]
        MOCK_GUARD_PATTERN["Guardrail trigger pattern\nflagged text trips guardrail"]
        MOCK_MULTI_TURN_PATTERN["Multi-turn pattern\nsequential stream variants"]
    end

    MOCK_IMPORT --> MOCK_CONFIGURE --> MOCK_STREAM_DEFINE
    MOCK_CONFIGURE --> MOCK_TOOL_DEFINE
    MOCK_AGENT_CREATE --> MOCK_RUNNER_CALL --> MOCK_ASSERTIONS
    MOCK_SETUP --> MOCK_USAGE

    MOCK_TEXT_PATTERN -.-> MOCK_CONFIGURE
    MOCK_TOOL_PATTERN -.-> MOCK_CONFIGURE
    MOCK_GUARD_PATTERN -.-> MOCK_CONFIGURE
    MOCK_MULTI_TURN_PATTERN -.-> MOCK_CONFIGURE

    style MOCK_SETUP fill:#f3e5f5,stroke:#7b1fa2
    style MOCK_USAGE fill:#e8f5e9,stroke:#2e7d32
    style MOCK_PATTERNS fill:#fff8e1,stroke:#f57f17
```

### How Streaming Is Mocked

The mock stream handler returns an object containing a readable stream. Chunks are typed deltas (`text-delta`, `tool-call-delta`, `tool-call`). Stream close marks completion. The raw provider-call payload can be stubbed for metadata assertions.

### How Tool Calls Are Mocked

Tool-call chunks include tool name, call ID, and arguments. The tool executor runs as if a real model requested it. Multi-turn test flows can feed tool results into subsequent mock turns.

### How Guardrail Triggers Are Mocked

Mock output emits known violating text. Streaming guardrails consume sliding windows and either throw tripwire in development or suppress remainder and inject fallback in production. Assertions target resulting stream behavior.

## Unit Tests

**Runner**: Bun test runner.
**Secrets**: None.
**Scope**: Every module, every export boundary, every typed interface edge.

Unit tests are the foundation: fast, deterministic, and infrastructure-independent. Every implementation task delivers tests and implementation together.

### What Gets Unit Tested

- Agent creation defaults and overrides.
- Configuration merge and validation behavior.
- Guardrail detection logic across regex, keyword, and model-assisted paths.
- Severity aggregation with worst-wins behavior.
- Guardrail pipeline orchestration.
- Stream transformation and chunk processing.
- Memory message formatting and turn truncation.
- Document chunking settings.
- Hybrid retrieval query composition.
- Citation schema extraction.
- Rate limiter window arithmetic.
- Circuit breaker state transitions.
- Logger context propagation.
- Token-budget arithmetic.
- CTA catalog validation.
- File validation by type signature and size limits.
- Public export surface stability.

### Mock Strategy

Every external boundary is mocked:

- **Model calls**: controlled AI SDK test mock model behaviors.
- **Database**: in-memory adapters or query-shape assertions.
- **Object storage**: deterministic buffer-return stubs.
- **Cache**: in-memory map-like adapter.
- **MCP servers**: stub transport and tool definitions.
- **Observability export**: no-op sink that captures spans for assertions.

## Integration Tests

**Runner**: Bun test runner with integration scope.
**Secrets**: Primary key required; optional secondary keys supported.
**Scope**: Real provider calls, real database operations, real external integrations.

Integration tests validate real external behavior. They are slower, cost-bearing, and less deterministic than unit suites. They do not gate basic CI green, but they are required before release readiness.

### Conditional Skip Pattern

Every integration file gates its top-level suite with a deterministic conditional skip helper. Missing keys cause clean skips, not failures. Multi-service tests compose key checks with logical conjunctions.

### What Gets Integration Tested

- Streaming with real model responses.
- Guardrail detection on real model outputs.
- Real embedding generation.
- Database round-trips including insert and retrieval.
- Object storage upload, download, and signed-link generation.
- Observability trace creation and score attachment.
- Memory extraction quality under real model behavior.
- Office-document conversion pipeline behavior.
- Rate limiter behavior against a real cache service.

## End-to-End Tests (E2E_TESTS)

**Runner**: Bun test runner with end-to-end scope.
**Secrets**: Full environment.
**Scope**: Complete user-level flows across library and server concerns.

E2E tests validate complete behavior as a user experiences it: create agent, stream response, verify output and side effects. They span agent runtime, transport, memory, guardrails, retrieval, and feedback chain.

### Three-Layer Memory QA Scenarios

- Thread short-term memory: 12 turns in one thread results in the most recent 10 turns plus rolling summary coverage for earlier turns.
- User short-term memory: activity in one thread appears in a new thread for the initial turns.
- User short-term fade-out: after enough turns in the new thread, cross-thread injection stops.
- Rolling summary continuity: long threads with topic switches keep all major topics represented.
- Interaction extraction: search intent becomes interaction memory.
- Media fact extraction: visual message content yields media facts and entity capture.
- Preference correction: corrected preference supersedes prior contradictory preference.
- Structured result memory: ordinal follow-up in new thread resolves to the correct prior result.
- Temporal recall: temporal references trigger time-scoped memory retrieval.
- Recency boost: newer relevant facts outrank older equally similar facts.
- Memory inspect: self-knowledge query returns categorized memory summary.
- Memory delete: requested topic deletion removes matching records and invalidates cache.
- Dependent intents: feedback intent applies before follow-up search intent.
- Cross-thread intent detection: vague new-thread input classifies correctly with cross-thread context.
- Auto-trigger recall: first message in a new thread can trigger automatic recall injection.

### Extraction Safeguard Scenarios

1. Third-party attribution stores interaction signal, not user preference.
2. Sarcasm handling stores correct negative sentiment.
3. Hypothetical statements do not become stored personal facts.
4. Hallucinated prior preference in model output is not extracted as fact.
5. Attribute negation in query constraints excludes conflicting results.
6. Query replay rewrite preserves intent while updating location or facet.
7. Pleasantry short-circuit skips heavy retrieval pipeline.
8. Gibberish short-circuit avoids full pipeline and responds safely.
9. Thread resurrection after long gap triggers recall and staleness context.
10. Rolling summary cap remains within summary-token limit.
11. Context-budget enforcement truncates lower-priority context first.
12. Oversized input is rejected with client-facing validation error.
13. Mid-stream failure skips extraction and avoids partial-fact writes.

**Security scenarios**:

- User isolation: one user never receives another user context.
- Memory deletion purges both long-term records and cache entries.
- Ordinal resolution remains scoped to requesting user.

### Test Matrix

```mermaid
flowchart TD
    subgraph LIBRARY_END_TO_END["Library end-to-end tests"]
        LIB_STREAM_FLOW["Full streaming flow\ncreate to stream to consume"]
        LIB_GUARDRAIL_FLOW["Guardrail violation\ntrigger to tripwire to fallback"]
        LIB_GROUNDING_FLOW["Grounding agent\ngrounding metadata present"]
        LIB_MCP_FLOW["MCP tools\nregistered and callable"]
        LIB_SHORT_MEMORY_FLOW["Short-term memory\npersisted across thread calls"]
        LIB_LONG_MEMORY_FLOW["Long-term memory round-trip\nextract to store to recall"]
        LIB_CTA_FLOW["CTA streaming\ntool-call hidden, cta event emitted"]
        LIB_BUDGET_FLOW["Token budget enforcement\nlimit reached yields denial"]
        LIB_LOCATION_FLOW["Location enrichment\ninternal tool hidden, location events emitted"]
    end

    subgraph SERVER_END_TO_END["Server end-to-end tests"]
        SERVER_STREAM_FLOW["Streaming route\nrequest yields stream"]
        SERVER_AUTH_FLOW["Auth required\nunauthenticated denied"]
        SERVER_GUARDRAIL_STREAM["Guardrail event in stream\ntripwire visible"]
        SERVER_TRACE_FLOW["Observability trace\ntrace visible after stream"]
        SERVER_FEEDBACK_FLOW["Feedback chain\ntrace linkage preserved"]
        SERVER_IMAGE_FLOW["Image pipeline\nupload to query to citation image access"]
    end

    style LIBRARY_END_TO_END fill:#e3f2fd,stroke:#1565c0
    style SERVER_END_TO_END fill:#fce4ec,stroke:#c62828
```

### Long-Term Memory Round-Trip (Critical)

1. Create an agent with fact extraction and recall capabilities.
2. Send preference-bearing user content with user context.
3. Wait for asynchronous extraction completion.
4. Verify stored facts exist for the right user and categories.
5. Open a different thread and ask a recall-triggering query.
6. Assert recall is invoked and prior facts appear in the response context.

This validates extraction to storage to recall across conversation boundaries with correct user scoping.

### Image Pipeline E2E

1. Upload a document that contains raster images.
2. Poll until processing reaches completion.
3. Ask a question requiring visual understanding.
4. Verify citations include image links.
5. Verify inline image references are present in response formatting.
6. Fetch one signed link and verify returned bytes match image signatures.
7. Verify fallback image route behavior redirects correctly.

## Eval Tests

**Runner**: Promptfoo as external development tool plus custom evaluation scorers.
**Secrets**: Keys required for judge-style evaluation.
**Scope**: Output quality, safety quality, retrieval quality.

Eval tests measure response quality rather than only functional correctness. Results are compared to defined thresholds; regressions are failures.

### What Gets Evaluated

- Response relevance.
- Citation accuracy.
- Guardrail precision.
- Guardrail recall.
- Grounding quality.
- Memory recall accuracy.
- CTA relevance.

### Promptfoo Integration

The library provides a Promptfoo provider factory to expose compatible evaluation interfaces. A self-test runner starts an ephemeral service layer, emits evaluation configuration, and evaluation runs externally. Result artifacts are collected for threshold comparison. Custom scorer helpers integrate specialized metrics into the same evaluation pipeline.

## Load Tests (LOAD_TESTS)

**Runner**: k6 as standalone binary.
**Secrets**: Full authenticated environment.
**Scope**: Concurrency behavior, latency baselines, and enforcement behavior under pressure.

Load tests measure system behavior under concurrent demand. They are run on demand and are not CI gates. Initial smoke runs produce baseline measurements that guide future large-scale exercises.

### Architecture

```mermaid
flowchart TD
    subgraph LOAD_GENERATOR["k6 load generator"]
        FIRST_VIRTUAL_USER["First virtual user"]
        SECOND_VIRTUAL_USER["Second virtual user"]
        NTH_VIRTUAL_USER["Nth virtual user"]
    end

    subgraph SERVER_UNDER_TEST["Server under test"]
        AUTH_MIDDLEWARE_LAYER["Auth middleware\nJWT verification"]
        ROUTE_LAYER["Route handlers\nstreaming upload query"]
        DEPENDENCY_LAYER["Dependencies\ndatabase cache object storage"]
    end

    subgraph METRICS_CAPTURE["Metrics collection"]
        STDOUT_SUMMARY["k6 stdout summary"]
        JSON_SUMMARY["k6 JSON metrics"]
        EVIDENCE_OUTPUT["Evidence artifacts"]
    end

    FIRST_VIRTUAL_USER & SECOND_VIRTUAL_USER & NTH_VIRTUAL_USER -->|"HTTP and streaming\nauthenticated requests\ncorrelation id"| AUTH_MIDDLEWARE_LAYER
    AUTH_MIDDLEWARE_LAYER --> ROUTE_LAYER --> DEPENDENCY_LAYER

    LOAD_GENERATOR --> STDOUT_SUMMARY --> EVIDENCE_OUTPUT
    LOAD_GENERATOR --> JSON_SUMMARY --> EVIDENCE_OUTPUT

    style LOAD_GENERATOR fill:#ff9900,color:#fff
    style SERVER_UNDER_TEST fill:#4488cc,color:#fff
    style METRICS_CAPTURE fill:#22aa44,color:#fff
```

### Script Catalog

Five load scripts cover critical routes:

1. Streaming: concurrent streamed chat, measuring first-token delay and full-stream duration.
2. File upload: concurrent uploads within size and batch constraints, measuring acceptance and processing start.
3. Retrieval query: concurrent retrieval-augmented queries, measuring hybrid retrieval latency.
4. Budget enforcement: rapid same-user requests validating denial after limit and counter consistency.
5. Health baseline: high-rate baseline for p50, p95, and p99 latency.

All scripts share base configuration for endpoint, auth token, and thresholds. Each script includes assertions for status and content behavior.

### Smoke Baselines

Initial smoke runs provide informational baselines:

- Health endpoint p99 target under one hundred milliseconds at low concurrency.
- Streaming route establishes connections and emits measurable first-token timing.
- Upload route accepts requests and initiates processing.
- Budget enforcement denies post-limit requests without concurrency leaks.

## Adversarial Tests

**Runner**: Bun test runner.
**Secrets**: None for pure detection paths; keys only when adversarial integration is required.
**Scope**: Safety bypass attempts, injection, obfuscation, and escalation.

Adversarial tests model attacker behavior. Each guardrail has bypass-attempt tests that prove resistance against common and evolving techniques.

### Attack Categories

- Prompt injection.
- Jailbreak pattern families.
- Unicode obfuscation.
- Token splitting.
- Encoding attacks.
- Multi-turn escalation.
- Context confusion.
- Output manipulation attempts.

### Testing Approach

Unit-level adversarial cases run payloads against individual guardrail functions. Integration-level adversarial cases run payloads through full input-to-output pipelines. Newly discovered bypasses are converted into permanent regression tests.

### Injection-Specific Test Categories

Beyond the general attack categories listed above, prompt injection testing requires dedicated suites.

**Direct injection tests**:
- Role-override attempts.
- System prompt extraction attempts.
- Instruction override prompts that try to supersede prior constraints.
- Delimiter injection that mimics higher-priority message formats.
- Encoding evasion, including base64 payloads, unicode homoglyph substitution, and token-split instructions.
- Multi-language injection attempts in non-primary languages.

**Indirect injection tests via retrieval**:
- Documents with embedded instructions hidden inside otherwise legitimate content.
- Exfiltration payloads in document content, especially URL query strings intended to leak conversation data through image rendering.
- Citation manipulation payloads that attempt to induce false or fabricated citations.
- Gradual context poisoning distributed across multiple retrieved chunks.

**Indirect injection tests via memory**:
- Crafted conversational turns designed to plant instruction-like content as user facts.
- Recall-triggered injection where poisoned memory activates on new-thread auto-recall.
- Memory supersession attacks that attempt to overwrite legitimate memory with injected directives.

**Multi-turn escalation tests**:
- Gradual persona drift over multi-turn windows.
- Privilege escalation sequences that attempt to activate elevated behavior.
- Context window exhaustion followed by late-turn injection payloads.
- Benign setup turns followed by a trigger turn that activates planted context.

**Output-side injection tests**:
- System prompt leakage attempts under multiple extraction prompt styles.
- Attention collapse detection where responses prioritize injected content over the actual user query.
- Exfiltration through generated tool arguments intended to leak sensitive data.

Each test category runs at both unit level for guardrail behavior and integration level for the full input-to-output pipeline. The regression suite expands continuously as new bypass techniques are discovered.

### Security and Concurrency Scenarios

- JWT algorithm confusion attempts are rejected.
- Signed-link leakage prevention prevents unauthorized object access beyond scope and expiry.
- Concurrent quota and budget checks reject over-limit race attempts.
- Per-source outage chaos verifies one failing source does not silently corrupt merged outputs.

## Regression Tests

**Runner**: Bun test runner.
**Secrets**: Depend on original failure context.
**Scope**: Previously observed failures and production-like edge cases.

Every bug fix adds a regression test that fails before the fix and passes after. Regression tests are permanent and remain through refactors.

### Structure

Each regression test documents:

- Original failure behavior.
- Minimal reproduction input.
- Expected post-fix behavior.
- Owning fix task.

Regression tests are co-located with protected modules and marked in test descriptions as regression-focused.

## Property-Based Tests

**Runner**: Bun test runner.
**Secrets**: None.
**Scope**: Invariant validation through randomized input generation.

Property testing validates truths that must hold for all valid inputs, not only hand-picked cases.

### Target Properties

- Configuration merge always retains required fields.
- Severity aggregation always returns highest severity regardless of order.
- Token counting remains non-negative and monotonic with added text.
- Rate-limiter windows align correctly for arbitrary timestamps.
- Citation extraction never yields page values outside document bounds.
- File signature validation avoids false negatives for supported formats.
- Memory truncation always preserves system context and most recent turns.

### Approach

Use a Bun-compatible property testing library or lightweight generators for domain-specific values. Each property executes at least one hundred randomized iterations. Shrinking is preferred but optional.

## Development Seed Data

Both repositories provide idempotent seed workflows for realistic local test datasets.

**Library seed** populates long-term memory data with user facts, graph relations, and vectors to exercise recall and deduplication behavior.

**Server seed** populates relational metadata, budget data, and representative document assets to exercise retrieval, budget enforcement, and file-management flows.

**Idempotency** is mandatory. Re-running seeds must yield stable state without duplicates or failures.

**Test user scope** uses a dedicated seeded user identity and valid development auth context so testing exercises real auth boundaries.

**Execution rule** requires supporting services to be available; failed dependencies produce explicit seed failure.

## QA Policy

Every implementation task from initial stack setup through final leak prevention includes agent-executed QA scenarios. No manual verification is considered sufficient.

### Evidence Collection

Every QA scenario writes a structured evidence artifact with task and scenario identifiers. Artifacts capture raw outputs proving pass behavior.

### Tool Categories

| Tool Category | Use Case | Description |
|---|---|---|
| Test runner execution | Module assertions | Runs scoped suites through Bun test runner |
| Eval execution mode | Quick export validation | Verifies minimal runtime behavior in isolation |
| Interactive terminal automation | TUI behavior validation | Automates keystrokes and output assertions in terminal sessions |
| HTTP request execution | API and streaming validation | Sends authenticated requests and validates status and payload behavior |

### TDD Discipline

1. RED: write failing behavior test.
2. GREEN: implement minimum behavior to pass.
3. REFACTOR: clean implementation while keeping tests green.

Tests and implementation are inseparable deliverables per task.

### E2E Cleanup

End-to-end suites rely on a cleanup helper that removes thread-associated conversation data, file metadata, retrieval chunks, and long-term memory records. This guarantees isolation without full database reset between runs.

## Task Specifications

### Task E2E_TESTS

**What to do**:

- Create dedicated end-to-end suites for library and server concerns.
- Validate full streaming flow from agent creation to final output consumption.
- Validate guardrail violation behavior across development and production modes.
- Validate grounding mode metadata.
- Validate MCP tool registration and invocation.
- Validate short-term memory persistence across thread turns.
- Validate long-term memory round-trip: extraction, persistence, cross-thread recall.
- Validate server streaming route behavior and auth enforcement.
- Validate guardrail signal visibility in stream events.
- Use Bun runner and mock model where practical to control cost.
- Validate CTA streaming behavior: hidden internal call, visible `cta` event payload.
- Validate observability trace visibility after stream completion.
- Validate feedback linkage from trace identifier to stored score.
- Validate token-budget denial structure at limit.
- Validate image pipeline from upload through query and image citation delivery.

**Must NOT do**:

- Do not require keys for unit tests.
- Do not include TUI rendering assertions in end-to-end suites.

**Depends on**: TUI_AGENT, SELF_TEST, SERVER_AGENT_CFG, SERVER_ROUTES, SERVER_MCP, SERVER_GUARDRAILS, UPLOAD_ENDPOINT, TUI_UPLOAD, DOCKER_COMPOSE, FEEDBACK_ENDPOINT, CLIENT_SDK, COST_TRACKING, AGENT_ROUTER, VALKEY_CACHE, TRIGGER_TASKS, RATE_LIMITING, FILE_CRUD, TTL_CLEANUP, JWT_AUTH, CROSS_CONV_RAG, ADMIN_API.

**Acceptance Criteria**:

- End-to-end suites pass for library scope.
- End-to-end suites pass for server scope.
- Full streaming path validates successfully.
- Guardrail handling is correct in both modes.
- MCP tools are functional.
- Long-term memory round-trip is proven.
- CTA stream cleanliness is proven.
- Trace visibility and feedback chain are proven.
- Token-budget denial response is structured and correct.
- Image lifecycle from upload to citation access is proven.

**QA Scenarios**:

- Full streaming suite validates stream initiation, chunk handling, final output, and evidence capture.
- Server streaming suite validates authenticated stream behavior and evidence capture.
- Image pipeline suite validates visual retrieval, image citation links, byte-signature checks, and fallback behavior.
- Long-term memory suite validates extraction, persistence, and cross-thread recall with evidence capture.

### Task PKG_PUBLISH

**What to do**:

- Prepare concise package documentation with quick start, feature highlights, API overview, config examples, and constraints.
- Ensure root licensing artifacts are complete.
- Ensure package metadata is complete and publish-ready.
- Ensure declaration output is generated for editor tooling.
- Prepare matching metadata and docs for the client SDK package.
- Validate publish dry-run behavior and package contents.
- Validate local install behavior and primary export usability.

**Must NOT do**:

- Do not include unresolved dependency specifiers.
- Do not include test assets in publish payload.
- Do not expand documentation into tutorial-length content.

**Depends on**: BARREL_EXPORTS.

**Acceptance Criteria**:

- Documentation and license artifacts are present and correct.
- Dry-run publish checks succeed for both packages.
- Local install verification succeeds.
- Declarations are generated.

**QA Scenarios**:

- Publishability validation confirms required docs, metadata correctness, dry-run success, local-install behavior, and evidence capture.

### Task SMOKE_TESTS

**What to do**:

- Build smoke workflow covering startup, health behavior, stream behavior, auth denial behavior, and graceful shutdown.
- Define container build behavior for both pre-publish and post-publish modes.
- Add server service integration into compose stack with dependency health ordering.
- Extend environment template with required server variables.
- Enforce auth-secret behavior: development allows explicit bypass mode with warning; production fails closed without secret.
- Provide startup and shutdown automation.

**Must NOT do**:

- Do not add orchestration for out-of-scope platforms.
- Do not add CI pipeline orchestration here.

**Depends on**: SERVER_AGENT_CFG, SERVER_ROUTES, SERVER_MCP, SERVER_GUARDRAILS, UPLOAD_ENDPOINT, DOCKER_COMPOSE, FILE_CRUD, ADMIN_API.

**Acceptance Criteria**:

- Smoke workflow passes health, stream, and auth checks.
- Container build succeeds and runtime health passes.
- Development mode without auth secret starts with clear warning.
- Production mode without auth secret fails startup with clear error.

**QA Scenarios**:

- Smoke validation captures startup, health, stream, and auth evidence.
- Container validation captures build and runtime health evidence.
- Development auth-bypass validation captures warning evidence.
- Production fail-closed validation captures startup rejection evidence.

### Task LOAD_TESTS

**What to do**:

- Create load suite with five scenarios: streaming, upload, retrieval, budget enforcement, health baseline.
- Use pre-generated authenticated context and request-correlation identifiers.
- Create shared threshold configuration.
- Run smoke-scale load passes and capture baseline metrics.
- Save structured output artifacts and summarize findings.
- Treat these scripts as foundation for future larger-scale execution.

**Must NOT do**:

- Do not run production-scale load in this task.
- Do not add k6 as a package dependency.
- Do not block correctness delivery on performance target attainment.
- Do not wire load tests into CI gates.

**Depends on**: SERVER_ROUTES, UPLOAD_ENDPOINT, DOCKER_COMPOSE.

**Acceptance Criteria**:

- Load suite exists with five scenarios and shared config.
- Each scenario runs successfully at smoke scale.
- Health baseline remains within sanity latency target at smoke load.
- Streaming scenario yields stable stream behavior and measurable latency.
- Upload scenario yields acceptance and processing initiation.
- Budget scenario yields deterministic post-limit denials.
- Structured output artifacts are captured per scenario.

**QA Scenarios**:

- Streaming smoke validates pass-rate, check-rate, and stream data continuity.
- Budget smoke validates pre-limit success and post-limit denial correctness under rapid requests.

### Task AUDIT_PLAN

**Depends on**: PKG_PUBLISH.

**Agent**: oracle.

**What to do**:

- Review the plan and implementation end to end.
- Verify every Must Have item has implementation and verification coverage.
- Verify every Must NOT item has no violations.
- Verify evidence artifacts exist for completed tasks.
- Verify deliverables align with plan commitments.

**Acceptance Criteria**:

- Must Have coverage is complete.
- Must NOT violations are zero.
- Task evidence is complete.
- Deliverable alignment is complete.

**Output format**: `Must Have [N/ALL] | Must NOT Have [N/ALL] | Tasks [N/N] | VERDICT: APPROVE/REJECT`.

**QA Scenarios**:

- Complete compliance yields APPROVE.
- Any Must NOT violation yields REJECT with file and line evidence.
- Missing evidence yields REJECT.
- Deliverable mismatch yields REJECT.

### Task AUDIT_CODE

**Depends on**: PKG_PUBLISH.

**Agent**: executor.

**What to do**:

- Execute build, lint, and tests.
- Review changed files for unsafe typing escapes, production logging misuse, empty catches, commented dead blocks, and unused imports.
- Review naming and abstraction quality to detect low-value generated patterns.
- Verify core dependencies are present and correctly wired.
- Verify forbidden storage and formatting patterns are absent.

**Acceptance Criteria**:

- Build has zero errors.
- Lint has zero errors.
- Tests pass.
- Unsafe typing escapes are justified or absent.
- Production logging uses approved logging paths.
- Commented-out dead blocks are absent.
- Low-quality generated-pattern issues are absent.

**Output format**: `Build [PASS/FAIL] | Lint [PASS/FAIL] | Tests [PASS/FAIL] | Files [CLEAN/ISSUES] | VERDICT`.

**QA Scenarios**:

- Clean build, lint, tests, and code quality yields APPROVE.
- Unsafe typing escapes without justification yield REJECT with evidence.
- Production logging misuse yields REJECT with evidence.
- Commented dead blocks yield REJECT with evidence.
- Forbidden storage usage yields REJECT with evidence.

### Task AUDIT_QA

**Depends on**: PKG_PUBLISH.

**Agent**: executor, optionally with browser-automation support if needed.

**What to do**:

- Start from clean environment state with dependencies and service stack.
- Execute every QA scenario from every implementation task.
- Execute cross-task integration checks:
  - Server imports library and starts.
  - Streaming works end to end.
  - Guardrails trigger correctly.
  - MCP tools are available.
  - TUI launches and streams.
  - Eval execution runs.
  - Upload to process to retrieval to citation works.
  - TUI upload command works.
  - Thread isolation is preserved.
  - Cleanup removes all related data.
  - Quota enforcement works.
  - Observability traces appear after stream.
  - Feedback submission links to trace scores.
  - Guardrail scores appear in observability backend.
  - CTA streaming keeps internal calls hidden and emits clean CTA events.
- Save complete evidence set.

**Acceptance Criteria**:

- All per-task QA scenarios pass.
- All fifteen integration checks pass.
- Edge cases are validated and documented.
- Evidence exists for every scenario.

**QA Scenarios**:

- Known-good run produces full pass.
- Intentionally broken guardrail path is detected as failure.
- Upload-to-citation integration path is validated end to end.
- Evidence structure contains one artifact per scenario.

**Output format**: `Scenarios [N/N pass] | Integration [N/15] | Edge Cases [N tested] | VERDICT`.

### Task AUDIT_SCOPE

**Depends on**: PKG_PUBLISH.

**Agent**: deep.

**What to do**:

- For each task from initial stack setup through final visual-grounding scope, compare specification intent and implementation outcome.
- Verify one-to-one scope fidelity: nothing missing, nothing extra.
- Verify Must NOT constraints exhaustively.
- Detect cross-task contamination and unexplained changes.
- Verify absence of forbidden patterns:

| Forbidden Pattern | Why |
|---|---|
| Production persistent storage via unsupported local database path | MN_BUN_SQLITE_PROD |
| Mixed MCP namespacing strategy | MN_MCP_NAMESPACE_COLLISION |
| Model-controlled access filter in retrieval tools | MN_LLM_CONTROLLED_ACCESS |
| Unsupported DOCX parser usage | MN_MAMMOTH |
| Unsupported image library usage | MN_SHARP |
| Legacy image-resize API usage | MN_JIMP_LEGACY |
| Unsupported observability adapter usage | MN_LANGFUSE_OTEL |
| Cloud observability configuration | MN_LANGFUSE_CLOUD |
| Raw SQL for metadata tables where ORM is required | MH_DRIZZLE_ORM |
| Hardcoded CTA catalog in core library | MH_CTA_SERVER_CATALOG |
| Internal CTA tool-call leakage to client stream | MH_CTA_HIDDEN |

**Acceptance Criteria**:

- Every task has one-to-one spec alignment.
- Scope creep is zero.
- Cross-task contamination is zero.
- Forbidden patterns are zero.
- Unaccounted changes are all explained.

**QA Scenarios**:

- Correctly scoped implementation yields full compliance.
- Introduced extra surface area is flagged as creep.
- Missing acceptance implementation is flagged.
- Inserted forbidden pattern is detected and reported.

**Output format**: `Tasks [N/N compliant] | Contamination [CLEAN/ISSUES] | Unaccounted [CLEAN/ISSUES] | VERDICT`.

## Coverage Map

Every Must Have feature area maps to one or more testing layers.

| Feature Area | Unit | Integration | End-to-End | Eval | Load | Adversarial |
|---|---|---|---|---|---|---|
| Agent creation and defaults | ✓ |  | ✓ |  |  |  |
| Input guardrails | ✓ | ✓ | ✓ | ✓ |  | ✓ |
| Streaming output guardrails | ✓ | ✓ | ✓ | ✓ |  | ✓ |
| Zero-leak buffered mode | ✓ |  | ✓ |  |  | ✓ |
| Guardrail factories | ✓ |  |  |  |  |  |
| MCP client wrapper | ✓ | ✓ | ✓ |  |  |  |
| Streaming transport | ✓ |  | ✓ |  | ✓ |  |
| Grounding search mode | ✓ | ✓ | ✓ | ✓ |  |  |
| Short-term memory | ✓ | ✓ | ✓ |  |  |  |
| Long-term memory | ✓ | ✓ | ✓ | ✓ |  |  |
| Memory recall tool | ✓ |  | ✓ |  |  |  |
| Auth middleware | ✓ |  | ✓ |  | ✓ |  |
| Trace and thread delivery | ✓ |  | ✓ |  |  |  |
| Provider-agnostic configuration | ✓ |  |  |  |  |  |
| TUI streaming display | ✓ |  | ✓ |  |  |  |
| Promptfoo provider helper | ✓ |  |  | ✓ |  |  |
| Custom evaluation scorers | ✓ |  |  | ✓ |  |  |
| File upload pipeline | ✓ | ✓ | ✓ |  | ✓ |  |
| Multimodal document processing | ✓ | ✓ | ✓ | ✓ |  |  |
| Hybrid retrieval | ✓ | ✓ | ✓ | ✓ | ✓ |  |
| Structured citations | ✓ | ✓ | ✓ | ✓ |  |  |
| Object storage | ✓ | ✓ | ✓ |  | ✓ |  |
| File signature validation | ✓ |  |  |  |  |  |
| CTA streaming | ✓ |  | ✓ | ✓ |  |  |
| Location enrichment | ✓ |  | ✓ |  |  |  |
| Rate limiting | ✓ | ✓ |  |  | ✓ |  |
| Token budget | ✓ |  | ✓ |  | ✓ |  |
| Circuit breaker | ✓ |  |  |  |  |  |
| Structured logging | ✓ |  |  |  |  |  |
| Observability tracing | ✓ | ✓ | ✓ |  |  |  |
| User feedback linkage | ✓ |  | ✓ |  |  |  |
| ORM and migrations | ✓ | ✓ |  |  |  |  |
| Cross-conversation retrieval | ✓ | ✓ | ✓ |  |  |  |
| Admin API | ✓ |  |  |  |  |  |
| Prompt management | ✓ | ✓ |  |  |  |  |
| Client offline queue | ✓ |  |  |  |  |  |
| Correction handling | ✓ |  | ✓ | ✓ |  |  |
| Emotional context carry-forward | ✓ |  | ✓ | ✓ |  |  |
| Frustration escalation detection | ✓ |  | ✓ | ✓ |  | ✓ |
| Repeated question differentiation | ✓ |  | ✓ | ✓ |  |  |
| Communication style memory | ✓ | ✓ | ✓ |  |  |  |
| Temporal fact markers | ✓ | ✓ |  |  |  |  |
| Fact supersession | ✓ | ✓ | ✓ |  |  |  |
| Implicit reference resolution | ✓ |  | ✓ | ✓ |  |  |
| Response energy matching | ✓ |  | ✓ | ✓ |  |  |
| Conversation resumption | ✓ |  | ✓ |  |  |  |
| Clarification patience model | ✓ |  | ✓ | ✓ |  |  |
| Topic abandonment | ✓ |  | ✓ | ✓ |  |  |
| Proactive clarification | ✓ |  | ✓ | ✓ |  |  |

## Requirement-Level Coverage (MH_*, MN_*)

Coverage is additionally mapped directly to requirement IDs in `01 — Requirements & Constraints`.

### Must Have (MH_*) Coverage Groups

| Requirement Group | Representative IDs | Primary Test Layers |
|---|---|---|
| Agent core | MH_AGENT_DEFAULTS, MH_INPUT_GUARDRAILS, MH_STREAMING_GUARDRAILS, MH_GUARDRAIL_FACTORIES, MH_MCP_CLIENT, MH_SSE_STREAMING, MH_GEMINI_GROUNDING, MH_PROVIDER_AGNOSTIC, MH_CONFIG_OVERRIDABLE, MH_NO_HARDCODED_PROMPTS | Unit, Integration, End-to-End, Eval, Adversarial |
| Language and content safety | MH_LANG_GUARD, MH_HATE_SPEECH_GUARD, MH_LANG_OUTPUT_SCAN, MH_LDNOOBW_ATTRIBUTION, MH_VI_OFFENSIVE_ATTRIBUTION | Unit, Integration, End-to-End, Eval, Adversarial |
| Memory | MH_SHORT_TERM_MEMORY, MH_LONG_TERM_MEMORY, MH_MEMORY_RECALL_TOOL, MH_USERID_SCOPING, MH_NO_WRITES_WITHOUT_USER | Unit, Integration, End-to-End, Eval |
| Auth and transport | MH_JWT_AUTH_REQUIRED, MH_TRACE_THREAD_DELIVERY, MH_ESM_ONLY | Unit, End-to-End, Load |
| TUI | MH_TUI_APP | Unit, End-to-End |
| Evaluation | MH_PROMPTFOO_SELFTEST, MH_EVAL_SCORERS | Unit, Eval |
| Document processing and upload | MH_FILE_UPLOAD, MH_MULTIMODAL_FIRST, MH_PAGE_SUMMARIZATION, MH_PROGRESSIVE_RETRIEVAL, MH_HYBRID_SEARCH_RRF, MH_BACKGROUND_ENRICHMENT, MH_IMAGE_PROCESSING, MH_PAGE_INDEX_TABLE, MH_CUSTOM_QUERY_TOOL, MH_STRUCTURED_CITATIONS | Unit, Integration, End-to-End, Eval, Load |
| Storage and files | MH_S3_STORAGE, MH_FILE_METADATA, MH_MAGIC_BYTES, MH_ASYNC_FILE_PROCESSING, MH_PARTIAL_BATCH_FAILURE, MH_CLEANUP_THREAD, MH_DOCKER_PGVECTOR | Unit, Integration, End-to-End, Load |
| Observability | MH_LANGFUSE_OBSERVABILITY, MH_CUSTOM_SPANS, MH_PII_FILTER, MH_USER_FEEDBACK, MH_LANGFUSE_SELFHOSTED | Unit, Integration, End-to-End |
| Data layer | MH_DRIZZLE_ORM | Unit, Integration |
| CTA streaming | MH_CTA_STREAMING, MH_CTA_SCHEMA, MH_CTA_HIDDEN, MH_CTA_SERVER_CATALOG | Unit, End-to-End, Eval |
| Location enrichment | MH_LOCATION_TOOL, MH_GEOCODE_PLUGGABLE, MH_GEOCODE_CACHED, MH_LOCATION_SUPPRESSED | Unit, End-to-End |
| Infrastructure | MH_RATE_LIMITING, MH_STRUCTURED_LOGGING, MH_FILE_CRUD, MH_TTL_CLEANUP, MH_CIRCUIT_BREAKER, MH_AUTH_MIDDLEWARE | Unit, Integration, End-to-End, Load |
| Humanlikeness | MH_CORRECTION_HANDLING, MH_EMOTIONAL_CONTEXT, MH_FRUSTRATION_DETECTION, MH_REPEATED_QUESTION_DIFF, MH_STYLE_MEMORY, MH_TEMPORAL_FACTS, MH_FACT_SUPERSESSION, MH_IMPLICIT_REFS, MH_RESPONSE_ENERGY, MH_CONVERSATION_RESUMPTION, MH_CLARIFICATION_PATIENCE, MH_TOPIC_ABANDONMENT, MH_PROACTIVE_CLARIFICATION | Unit, Integration, End-to-End, Eval, Adversarial |

### Must Not (MN_*) Guardrail Coverage Groups

| Exclusion Group | Representative IDs | Detection Layers |
|---|---|---|
| Runtime and compatibility exclusions | MN_BUN_SQLITE_PROD, MN_PROMPTFOO_DIRECT_IMPORT, MN_SHARP, MN_JIMP_LEGACY, MN_AWS_SDK_REDUNDANT, MN_LANGFUSE_OTEL, MN_VERCEL_OTEL | Unit static checks, audit scans, regression |
| Architecture exclusions | MN_LLM_CONTROLLED_ACCESS, MN_MAMMOTH, MN_SUMMARIES_ONLY, MN_CHUNKS_WITHOUT_SUMMARIES, MN_LOCAL_FILESYSTEM, MN_CROSS_USER_SHARING, MN_MULTI_PROVIDER_ROUTING, MN_HEAVY_ORM, MN_LANGFUSE_CLOUD, MN_LANGFUSE_DATASETS, MN_LANG_GUARD_DEFAULT_ON, MN_SSE_COMPRESSION, MN_CTA_SERVER_RENDER, MN_LOCATION_DEFAULT_IMAGES | Unit, Integration, End-to-End, audit scans |
| Scope exclusions | MN_WORKFLOW_ENGINE, MN_REDIS_STORAGE, MN_UNSUPPORTED_MEDIA, MN_VOICE_AGENT, MN_WEB_DASHBOARD, MN_COMPLEX_MULTI_AGENT, MN_WEBSOCKET, MN_EDGE_RUNTIME, MN_CUSTOM_EMBEDDINGS, MN_WEBHOOKS, MN_AUTO_PROMPT_OPT, MN_CTA_AB_TESTING, MN_COMPLEX_CTA, MN_TUI_IDE_FEATURES, MN_BUN_HOT, MN_RAW_SURREALQL, MN_DIRECT_ENV_ACCESS | Plan audit, scope audit, code audit |

## Verification Strategy Summary

All verification is automated and agent-executed.

- Infrastructure baseline: Bun test runner.
- Delivery discipline: RED to GREEN to REFACTOR per task.
- Unit gate: always green with zero secrets.
- Integration gate: conditional skip for missing-key environments.
- Evidence gate: every QA scenario emits evidence artifact.
- Mocking standard: the AI SDK test mock model from AI SDK test utilities.
- Separation: distinct unit, integration, and end-to-end scopes.
- Regression policy: every bug fix adds permanent regression coverage.
- Adversarial policy: every guardrail has bypass-attempt coverage.
- Load policy: k6 scripts exist for critical routes, run manually, capture baselines.
- Eval policy: Promptfoo plus custom scorers validate quality dimensions.

## Cross-Plan References

Testing strategy aligns with and validates these plan documents in the new 17-file structure:

- Requirements source of truth: [01 — Requirements & Constraints](./01-requirements.md)
- System architecture: [03 — System Architecture](./03-architecture.md)
- Foundation and shared conventions: [04 — Foundation](./04-foundation.md)
- Conversation behavior: [05 — Conversation Pipeline](./05-conversation.md)
- Agents: [06 — Agents & Orchestration](./06-agents.md)
- Memory design and responsibilities: [07 — Memory & Intelligence](./07-memory.md)
- Document processing: [08 — Document Processing](./08-documents.md)
- Retrieval and RAG behavior: [09 — Retrieval & Evidence](./09-retrieval.md)
- Guardrails and policy enforcement: [10 — Guardrails & Safety](./10-guardrails.md)
- Transport and event semantics: [11 — Streaming & Transport](./11-transport.md)
- Server behavior and auth enforcement details: [12 — Server Implementation](./12-server.md)
- Terminal interface expectations: [13 — TUI App](./13-tui.md)
- Observability behavior: [14 — Observability](./14-observability.md)
- Infrastructure operations and capacity planning: [15 — Infrastructure](./15-infrastructure.md)

## External References

- Bun test runner: https://bun.sh/docs/cli/test
- k6 load testing: https://grafana.com/docs/k6/latest/
- k6 streaming and realtime APIs: https://grafana.com/docs/k6/latest/javascript-api/k6-experimental/websockets/
- Promptfoo: https://promptfoo.dev/docs
- AI SDK testing utilities: https://sdk.vercel.ai/docs/ai-sdk-core/testing

*Previous: [15 — Infrastructure](./15-infrastructure.md) | Next: [17 — Execution Plan](./17-execution.md)*
