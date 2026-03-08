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

### Evaluation Thresholds and Baselines

Eval tests compare against defined minimum thresholds. Initial baselines are established during the first evaluation run and tightened as the system matures. Score regression greater than five percentage points from previous baseline triggers test failure regardless of absolute threshold.

| Quality Dimension | Minimum Threshold | Target | Measurement Method |
|---|---|---|---|
| Response relevance | 0.70 | 0.85 | LLM-as-judge scoring against query intent |
| Citation accuracy | 0.80 | 0.95 | Automated verification of cited page content match |
| Guardrail precision | 0.90 | 0.98 | True positive rate on known-safe inputs (no false blocks) |
| Guardrail recall | 0.85 | 0.95 | Detection rate on known-malicious inputs |
| Grounding quality | 0.75 | 0.90 | Factual accuracy of web-grounded responses |
| Memory recall accuracy | 0.70 | 0.85 | Correct fact retrieval for known-user scenarios |
| CTA relevance | 0.65 | 0.80 | Contextual appropriateness of suggested actions |
| Retrieval precision at ten | 0.75 | 0.90 | Relevant pages in top ten hybrid search results |
| Evidence sufficiency | 0.70 | 0.85 | Gate correctly opens for answerable queries and closes for unanswerable |
| Hallucination rate | Below 0.05 | Below 0.03 | Unsupported claims in generated responses per anti-hallucination architecture |
| Extraction accuracy | 0.75 | 0.90 | Correctly extracted facts from known-fact conversations |
| Rewrite fidelity | 0.85 | 0.95 | All original entities preserved in rewritten queries |

Threshold enforcement:

- Scores below minimum threshold produce test failure that blocks release.
- Scores between minimum and target produce test pass with improvement annotation.
- Scores above target produce clean pass.
- Score regression greater than five points from previous baseline produces failure regardless of absolute threshold.

### Golden Dataset Requirements

- Minimum fifty test cases per quality dimension.
- Cases sourced from real production scenarios where possible.
- Cases reviewed and approved by human evaluator before inclusion.
- Dataset version-controlled alongside test configuration.
- New cases added when production failures reveal gaps not covered by existing cases.
- Dataset refresh cadence: monthly review, quarterly expansion, immediate addition for critical failures.

### Scorer Categories

Scorers are organized into four categories with different sampling strategies:

| Category | Sampling in Production | Purpose |
|---|---|---|
| Safety scorers | Higher sampling rate (up to every request) | Guardrail precision, injection detection, output safety |
| Relevance scorers | Moderate sampling rate | Response relevance, retrieval quality, evidence sufficiency |
| Quality scorers | Lower sampling rate | Citation accuracy, grounding quality, rewrite fidelity |
| Behavioral scorers | Moderate sampling rate | Memory recall, CTA relevance, humanlikeness dimensions |

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

### Production-Scale Scenarios

Beyond smoke baselines, load tests scale to production-representative levels for deployment confidence. These scenarios run in staging environments with production-like infrastructure.

| Scenario | Smoke Scale | Production Scale | Metric Focus |
|---|---|---|---|
| Streaming chat | 10 concurrent users | 1000 concurrent users | First-token latency p95, stream completion rate |
| File upload | 5 concurrent uploads | 200 concurrent uploads | Acceptance latency, queue depth, processing throughput |
| Retrieval query | 10 concurrent queries | 500 concurrent queries | Hybrid search latency p95, result quality under load |
| Budget enforcement | 50 rapid requests | 5000 rapid requests per second | Counter accuracy, denial consistency, race condition detection |
| Health baseline | 100 requests per second | 10000 requests per second | p50, p95, p99 latency, error rate |
| Mixed workload | Combined above at ten percent | Combined above at fifty percent | System stability, resource contention, degradation patterns |

### Production-Scale Measurement Targets

Production-scale load tests measure:

- Latency percentiles under sustained load (p50, p95, p99) for each route category.
- Error rate under load with target below 0.1 percent for non-rate-limit errors.
- Resource utilization including CPU, memory, and connection pool usage.
- Degradation curve showing performance change as load increases from ten percent to one hundred percent of target capacity.
- Recovery time measuring how quickly latency returns to baseline after load spike removal.
- Connection pool behavior including exhaustion thresholds, queuing depth, and rejection patterns.
- Cache hit ratio under concurrent access to shared cache keys.
- Budget counter accuracy under concurrent spend with no over-admission beyond one-request tolerance.

### Scalability Breakpoint Detection

- Ramp load linearly from baseline to two times target capacity.
- Identify the inflection point where latency degrades non-linearly.
- Identify the saturation point where error rate exceeds acceptable threshold.
- Document system capacity ceiling for deployment planning.
- Validate horizontal scaling by running the same ramp against two, four, and eight API instances.

### Streaming Load Specifics

Streaming endpoints require specialized load testing because each connection holds server resources for the duration of the response:

- Measure maximum concurrent SSE connections per API instance before resource exhaustion.
- Validate that connection cleanup occurs correctly when clients disconnect mid-stream.
- Test interleaved short and long streams to detect resource contention patterns.
- Verify backpressure behavior when slow consumers accumulate.

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

### Extended Target Properties

Beyond the foundational seven properties, property-based testing covers all modules with domain-specific invariants.

**Conversation Pipeline Properties**:

- Intent classification produces valid intent enum values for all generated inputs.
- Rewrite triggers preserve all original entities (names, codes, dates, identifiers) in rewritten output.
- Source priority weighting formula produces weights between zero and one for all source counts.
- RRF fusion scores are non-negative and bounded for all rank combinations.
- Multi-intent decomposition produces at least one sub-query per detected intent count.
- Temporal resolution produces valid date ranges for all temporal phrase patterns with arbitrary timezone offsets.
- Embedding router cosine similarity scores fall between negative one and one for all vector pairs.

**Memory Properties**:

- Fact extraction from user-only content never produces facts attributed to assistant.
- Supersession always preserves audit trail: superseded record exists after every supersession operation.
- Deduplication threshold boundary: similarity exactly at 0.92 produces consistent duplicate detection across repeated evaluations.
- Recency boost multipliers are applied in correct order: 24-hour multiplier greater than 7-day multiplier greater than default.
- Rolling summary token count never exceeds configured maximum after any number of compaction cycles.
- TTL refresh always extends expiry by full TTL duration regardless of current remaining time.
- Emotional context decay counter decrements exactly once per user turn and never goes negative.
- Cross-thread user short-term loading returns at most the configured limit regardless of thread count.

**Document Processing Properties**:

- PDF splitting produces exactly one output per input page for all valid PDFs.
- Chunk overlap is exactly the configured overlap amount for all text lengths above minimum.
- Quota arithmetic never produces negative used_bytes values regardless of operation ordering.
- File status state machine transitions are always unidirectional: no backward transitions for any event sequence.
- Image size filtering correctly accepts at exactly 100 pixels and rejects at 99 pixels for both dimensions.
- Page progress counter increments monotonically and reaches progress_total exactly once.

**Guardrail Properties**:

- Worst-wins aggregation always returns the highest severity in any verdict set regardless of order or count.
- Parallel guardrail execution produces same aggregate result regardless of individual completion order.
- Sliding window buffer size never exceeds configured maximum for any chunk sequence.
- Zero-leak buffer mode never emits bytes to client before buffer reaches configured size or violation is detected.
- Composite guardrail aggregation produces same result regardless of constituent grouping order.
- External guardrail factory with failMode open never blocks requests on network errors.
- External guardrail factory with failMode closed always blocks requests on network errors.

**Infrastructure Properties**:

- Rate limiter sliding window calculations produce correct Retry-After for arbitrary timestamp sequences within any window size.
- Circuit breaker state transitions follow valid state machine paths: only closed-to-open, open-to-half-open, half-open-to-closed, and half-open-to-open.
- Budget counter operations maintain atomicity: no intermediate state visible between increment and expiry setting.
- Budget pessimistic reservation followed by reconciliation produces correct final counter value for all estimate-vs-actual combinations.
- Cache TTL expiry is consistent regardless of read access pattern: a key with TTL N expires at the same absolute time whether read zero or one hundred times.

**Transport Properties**:

- SSE event serialization produces valid SSE format for all nine event type variants with arbitrary payload content.
- Verbosity filtering never removes events that should be visible at the configured level and never includes events that should be hidden.
- Stream chunk ordering is preserved through all transformation stages for any interleaving of event types.
- Session-meta event is always emitted exactly once as the first event in any stream regardless of agent behavior.

### Approach

Use a Bun-compatible property testing library or lightweight generators for domain-specific values. Each property executes at least one hundred randomized iterations. Shrinking is preferred but optional.

## Chaos and Failure Testing

**Runner**: Bun test runner with chaos scope, plus infrastructure tools for network-level fault injection.
**Secrets**: Full environment plus fault injection infrastructure.
**Scope**: Systematic failure injection for all external dependencies to verify graceful degradation, fallback paths, and recovery behavior.

Chaos testing intentionally injects faults to uncover weaknesses before they cause production failures. AI agent systems have unique failure modes: LLM provider timeouts cascade differently than database outages, and cache failures affect budget enforcement differently than retrieval quality.

### Chaos Testing Architecture

```mermaid
flowchart TD
    subgraph CHAOS_LIFECYCLE["Chaos Experiment Lifecycle"]
        STEADY_STATE["Define Steady State\ntask completion rate error rate latency baselines"]
        HYPOTHESIS["Form Hypothesis\nmetrics remain within bounds under fault"]
        INJECT["Inject Fault\nsingle dependency failure at a time"]
        OBSERVE["Observe Behavior\nmonitor all metrics during fault window"]
        VERIFY["Verify Hypothesis\nmetrics within acceptable degradation bounds"]
        RESTORE["Restore and Verify Recovery\nremove fault and confirm metrics return to baseline"]
    end

    STEADY_STATE --> HYPOTHESIS --> INJECT --> OBSERVE --> VERIFY --> RESTORE
    RESTORE -.->|"next fault target"| HYPOTHESIS

    style STEADY_STATE fill:#22aa44,color:#fff
    style INJECT fill:#ff4444,color:#fff
    style VERIFY fill:#4488cc,color:#fff
    style RESTORE fill:#22aa44,color:#fff
```

### Fault Injection Matrix

Every external dependency has a defined failure mode, expected system behavior, and recovery verification.

**LLM Provider Faults**:

| Fault | Expected Behavior | Recovery Verification |
|---|---|---|
| Primary key unavailable | Rotate to next key in pool | Response succeeds with different key |
| All keys exhausted | Circuit breaker opens, fast-fail without network calls | Circuit half-opens after timeout, probe succeeds |
| Rate limit response from provider | Exponential backoff with jitter, no circuit trigger | Requests resume after backoff window |
| Timeout with no response | Retry with timeout, circuit opens after threshold consecutive failures | Circuit recovers after reset timeout |
| Malformed response body | Parse error caught, retry once, surface error if persistent | Normal responses resume immediately |
| Context length exceeded response | No retry, surface as client-facing error | Next request with valid length succeeds |

**PostgreSQL Faults**:

| Fault | Expected Behavior | Recovery Verification |
|---|---|---|
| Connection refused | Health endpoint returns down status, all requests return service unavailable | Automatic reconnection on availability, health returns ok |
| Query timeout | Individual request fails with timeout, connection pool remains healthy | Subsequent queries succeed normally |
| Connection pool exhaustion | New requests queue then timeout, no process crash | Pool recovers as long-running queries complete |
| Disk full simulation | Write failures surface as typed errors, reads continue | Writes resume after space recovery |

**SurrealDB Faults**:

| Fault | Expected Behavior | Recovery Verification |
|---|---|---|
| WebSocket connection dropped | Long-term memory disabled, chat continues via short-term only | Auto-reconnect on next operation, memory resumes |
| Query timeout | Memory operation times out, extraction skipped, short-term persists anyway | Subsequent operations succeed |
| Complete unavailability | Graceful degradation to no long-term memory with warning log | Full memory restored on reconnection |

**Valkey Faults**:

| Fault | Expected Behavior | Recovery Verification |
|---|---|---|
| Connection refused | In-memory fallback for cache, per-instance rate limiting, budget fail-open | Valkey reconnection restores global state |
| Response timeout | Individual operation times out, fallback path used | Subsequent operations succeed normally |
| Eviction under memory pressure | Cache misses increase, source-of-truth queries increase | Application remains functional with higher latency |
| Connection restored after outage | No cache stampede, gradual warm-up | Cache hit ratio returns to baseline within minutes |

**Object Storage Faults**:

| Fault | Expected Behavior | Recovery Verification |
|---|---|---|
| Upload failure | Quota reservation rolled back, file marked failed | Retry upload succeeds, quota correct |
| Download timeout during retrieval | Page context assembly degrades gracefully without full request failure | Subsequent downloads succeed |
| Signed URL expired during client access | Client receives clear expiry error, can re-request | New signed URL generated successfully |
| Complete unavailability | File upload and file-backed retrieval disabled, chat continues | Full file capability restored on recovery |

**Background Job Faults**:

| Fault | Expected Behavior | Recovery Verification |
|---|---|---|
| Worker crash mid-task | Job re-enqueued by queue adapter, idempotent re-execution | Document reaches ready or enriched state |
| Duplicate delivery | Idempotent handlers produce same result, no duplicate records | Single set of page_index rows exists |
| Queue unavailable | In-process fallback executes same handlers directly | Background jobs resume when queue returns |

**Observability Faults**:

| Fault | Expected Behavior | Recovery Verification |
|---|---|---|
| Langfuse unavailable at startup | Warning logged, fallback prompts loaded, no-op tracing | Full tracing resumes when Langfuse returns |
| Langfuse unavailable at runtime | No-op scoring, stale-while-revalidate for cached prompts | Score writes resume on recovery |
| Trace flush timeout | Pending telemetry buffered, no request blocking | Buffer flushes on next successful connection |

**LibreOffice Faults**:

| Fault | Expected Behavior | Recovery Verification |
|---|---|---|
| Sidecar unavailable | DOCX conversion fails, file marked failed with clear error | Conversion succeeds when sidecar returns |
| Conversion timeout after thirty seconds | File marked failed, timeout error recorded | Subsequent conversions succeed within timeout |
| Corrupt output | PDF validation catches corrupt output, file marked failed | Re-conversion of same file succeeds |

### Chaos Experiment Execution

Chaos experiments follow a strict protocol to prevent uncontrolled failure propagation:

1. Verify steady state before injection: all health checks pass, baseline metrics within normal range.
2. Inject exactly one fault at a time: never combine failures in a single experiment.
3. Observe for defined duration: minimum thirty seconds, maximum five minutes per fault.
4. Verify hypothesis: check that degradation metrics remain within acceptable bounds.
5. Restore and verify recovery: remove fault and confirm return to baseline within defined recovery window.
6. Document results: capture metrics, behavior observations, and any unexpected side effects.

### Chaos Execution Schedule

- **Unit-level chaos** (every commit): circuit breaker state transitions, retry logic, fallback path selection — all with mocked dependencies.
- **Integration-level chaos** (nightly): TCP-proxy-based fault injection against real services in Docker Compose.
- **Game day chaos** (monthly): multi-fault scenarios, sustained outages, recovery exercises.

## Contract Testing

**Runner**: Bun test runner.
**Secrets**: None (schema validation only).
**Scope**: Consumer-driven contracts between the library and server repositories verifying interface stability.

Contract testing guarantees that interface changes in the library propagate correctly to the server without requiring full integration test execution. The library defines what it provides. The server defines what it expects. Contract tests verify these promises align.

### Contract Testing Architecture

```mermaid
flowchart LR
    subgraph LIBRARY_SIDE["Library Repository"]
        LIB_EXPORTS["Typed Exports\ninterfaces schemas event types"]
        LIB_PROVIDER_CONTRACT["Provider Contract\nwhat library promises to deliver"]
    end

    subgraph SERVER_SIDE["Server Repository"]
        SERVER_IMPORTS["Consumed Interfaces\nimported types and schemas"]
        SERVER_CONSUMER_CONTRACT["Consumer Contract\nwhat server expects to receive"]
    end

    subgraph CONTRACT_VERIFICATION["Verification Layer"]
        SCHEMA_COMPAT["Schema Compatibility\nZod schema structural alignment"]
        TYPE_COMPAT["Type Surface Stability\npublic export surface unchanged"]
        EVENT_COMPAT["Event Shape Contract\nSSE event types match consumer expectations"]
        ERROR_COMPAT["Error Code Contract\nlibrary codes match server message map"]
    end

    LIB_PROVIDER_CONTRACT --> SCHEMA_COMPAT
    SERVER_CONSUMER_CONTRACT --> SCHEMA_COMPAT
    LIB_PROVIDER_CONTRACT --> TYPE_COMPAT
    SERVER_CONSUMER_CONTRACT --> TYPE_COMPAT
    LIB_PROVIDER_CONTRACT --> EVENT_COMPAT
    SERVER_CONSUMER_CONTRACT --> EVENT_COMPAT
    LIB_PROVIDER_CONTRACT --> ERROR_COMPAT
    SERVER_CONSUMER_CONTRACT --> ERROR_COMPAT

    style LIBRARY_SIDE fill:#e8f5e9,stroke:#2e7d32
    style SERVER_SIDE fill:#e3f2fd,stroke:#1565c0
    style CONTRACT_VERIFICATION fill:#fff3e0,stroke:#ef6c00
```

### Contract Areas

Each contract area represents an interface boundary where library and server must agree:

- **SSE event type shapes**: all nine event types (session-meta, text-delta, trace-step, cta, citation, location, tripwire, done, error) with defined Zod schemas validated on both sides.
- **Agent configuration interface**: the configuration object that server passes to library agent factory, validated against library schema.
- **Guardrail pipeline configuration interface**: guardrail function signatures, severity types, verdict shapes.
- **Memory configuration interface**: memory config objects including all numeric thresholds.
- **Storage factory interface**: storage configuration shape and auto-detection contract.
- **Stream event iterator interface**: the async iterable shape returned by agent execution.
- **Error code union**: library emits typed error codes, server maps every code to a message — contract verifies completeness.
- **Typed result patterns**: neverthrow Result type shapes for all boundary operations.
- **Trace-step event discriminated union**: step-specific payload shapes matched between library emission and client SDK consumption.

### Contract Execution

Contract tests run on every commit in both repositories. When the library publishes a new contract artifact, the server CI validates against it. When the server updates its consumer expectations, the library CI validates it can still satisfy them.

Breaking contract changes are detected before they reach integration testing, saving significant feedback time.

## Mutation Testing

**Runner**: Mutation testing framework compatible with Bun.
**Secrets**: None.
**Scope**: Safety-critical paths where undetected mutations could cause security or quality failures.

Mutation testing systematically introduces small changes (mutations) to code and verifies that existing tests catch them. Surviving mutants reveal gaps in test coverage for critical logic.

### Mutation Testing Process

```mermaid
flowchart TD
    subgraph MUTATION_LIFECYCLE["Mutation Testing Lifecycle"]
        IDENTIFY_PATHS["Identify Safety-Critical Paths\nguardrails budget enforcement rate limiting injection detection"]
        GENERATE_MUTANTS["Generate Mutations\nthreshold shifts logic inversions boundary changes"]
        EXECUTE_TESTS["Execute Full Test Suite\nrun against each mutant"]
        ANALYZE_SURVIVORS["Analyze Surviving Mutants\neach survivor reveals a test gap"]
        STRENGTHEN_TESTS["Add Missing Tests\nwrite tests that kill each survivor"]
    end

    IDENTIFY_PATHS --> GENERATE_MUTANTS --> EXECUTE_TESTS --> ANALYZE_SURVIVORS --> STRENGTHEN_TESTS
    STRENGTHEN_TESTS -.->|"iterate until zero survivors"| GENERATE_MUTANTS

    style IDENTIFY_PATHS fill:#22aa44,color:#fff
    style GENERATE_MUTANTS fill:#ff9900,color:#fff
    style ANALYZE_SURVIVORS fill:#ff4444,color:#fff
    style STRENGTHEN_TESTS fill:#4488cc,color:#fff
```

### Mutation Target Paths

Mutation testing focuses exclusively on paths where a missed mutation could cause security, safety, or correctness failures:

- **Input guardrail pipeline**: mutate severity thresholds, detection pattern logic, worst-wins aggregation comparisons.
- **Output guardrail sliding window**: mutate buffer size checks, verdict evaluation logic, chunk suppression conditions.
- **Injection detection ensemble**: mutate tier decision logic (LLM-alone blocks, two-tier flags), parallel execution coordination, blocking threshold comparisons.
- **Memory extraction safeguards**: mutate attribution filter conditions, certainty filter thresholds, injection classifier integration.
- **Content sanitization**: mutate pattern matching expressions, redaction logic, boundary framing insertion.
- **Budget enforcement**: mutate comparison operators in limit checks, reservation increment logic, rollback conditions.
- **Rate limiting**: mutate window boundary calculations, counter increment operations, rejection threshold comparisons.
- **Evidence gate**: mutate sufficiency scoring formula, weight values, threshold comparison operators.
- **File validation**: mutate magic byte comparisons, size limit checks, quota arithmetic.
- **Trust hierarchy enforcement**: mutate trust level assignments, boundary framing conditions, zero-trust content handling.

### Mutation Categories

- **Threshold mutations**: shift numeric boundaries by plus or minus one, by ten percent, to zero, and to maximum value.
- **Logic inversions**: flip boolean conditions, swap AND with OR, invert comparison operators.
- **Boundary shifts**: off-by-one in window sizes, TTL values, limit checks, array indices.
- **Removal mutations**: remove validation steps, skip aggregation stages, omit safety checks.
- **Reorder mutations**: change execution order in pipelines, swap priority levels in aggregation.

### Mutation Testing Schedule

Mutation testing runs nightly against safety-critical paths. Target mutation kill rate is above ninety-five percent for all guarded paths. Surviving mutants are triaged within one business day.

## Snapshot and Golden-File Testing

**Runner**: Bun test runner.
**Secrets**: None for structural snapshots; keys required for golden dataset evaluation.
**Scope**: Non-deterministic LLM outputs validated through structural and quality-score snapshots rather than exact text matching.

Traditional snapshot testing breaks for AI systems because outputs vary between runs. Snapshot testing for this system captures structure, properties, and quality scores rather than raw text content.

### Snapshot Strategy

```mermaid
flowchart TD
    subgraph SNAPSHOT_TYPES["Snapshot Testing Strategies"]
        STRUCTURAL["Structural Snapshots\nJSON shape, field presence, type conformance"]
        BEHAVIORAL["Behavioral Snapshots\ntool call sequences, event ordering, state transitions"]
        QUALITY_SCORE["Quality Score Snapshots\neval scores stored as snapshot values"]
        FORMAT_SNAP["Format Compliance Snapshots\nSSE event format, citation schema, error shapes"]
    end

    subgraph GOLDEN_MANAGEMENT["Golden Dataset Management"]
        CURATE["Curate from Production Failures\nreal failures become test cases"]
        HUMAN_REVIEW["Human Review Required\nsnapshot updates never auto-accepted"]
        BASELINE_SCORES["Baseline Quality Scores\nper-case quality measurements"]
        REGRESSION_DETECT["Regression Detection\nscore drops below baseline trigger review"]
    end

    STRUCTURAL --> GOLDEN_MANAGEMENT
    BEHAVIORAL --> GOLDEN_MANAGEMENT
    QUALITY_SCORE --> GOLDEN_MANAGEMENT
    FORMAT_SNAP --> GOLDEN_MANAGEMENT

    style SNAPSHOT_TYPES fill:#f3e5f5,stroke:#7b1fa2
    style GOLDEN_MANAGEMENT fill:#e8f5e9,stroke:#2e7d32
```

### Structural Snapshots

Structural snapshots validate that output shape is correct without asserting exact content:

- Agent response objects contain all required fields with correct types.
- Citation objects contain source, fileId, page, and quote fields.
- SSE events conform to their type-specific schemas.
- Tool call sequences match expected tool names and argument shapes.
- Guardrail verdicts contain severity and conceptId fields.
- Memory extraction results contain factType, confidence, and temporal state fields.
- Error responses contain error code and message fields matching typed error union.

### Behavioral Snapshots

Behavioral snapshots validate execution patterns rather than output content:

- Multi-intent queries produce expected sub-query decomposition structure.
- Source priority execution produces expected source ordering.
- Dependent intent handling produces expected sequential-then-parallel execution pattern.
- Guardrail pipeline execution produces verdicts in expected aggregation order.
- Memory recall produces expected retrieval-then-ranking pipeline behavior.

### Quality Score Snapshots

For eval-dependent validation, quality scores serve as the snapshot rather than raw text:

- Each golden dataset case has a recorded baseline quality score per dimension.
- Re-evaluation produces scores that are compared against baselines.
- Score drops greater than five percentage points from baseline trigger mandatory human review.
- Score improvements are accepted and baselines are updated after review.
- Quality score history is retained for trend analysis.

### Golden Dataset Lifecycle

- **Initial creation**: minimum fifty cases per quality dimension sourced from realistic scenarios.
- **Expansion**: new cases added when production usage reveals failure modes not covered.
- **Maintenance**: monthly review of cases for continued relevance, quarterly score recalibration.
- **Retirement**: cases removed only when the tested behavior is intentionally changed, with documented justification.

## Streaming-Specific Tests

**Runner**: Bun test runner with streaming scope.
**Secrets**: Mock model for unit-level streaming; keys required for integration streaming tests.
**Scope**: Dedicated testing patterns for SSE streaming behavior covering mid-stream failures, backpressure, reconnection, and protocol compliance.

Streaming responses require specialized testing because they hold server resources for extended durations, involve incremental delivery, and have failure modes that do not exist in request-response patterns.

### Streaming Test Architecture

```mermaid
flowchart TD
    subgraph STREAM_TEST_TYPES["Streaming Test Categories"]
        FORMAT_TESTS["Format Compliance\nevent type validation, line format, done marker"]
        MIDSTREAM_TESTS["Mid-Stream Failure\nprovider error, network drop, timeout"]
        BACKPRESSURE_TESTS["Backpressure Handling\nslow consumer, fast producer, buffer limits"]
        RECONNECT_TESTS["Reconnection Behavior\nclient reconnect, state recovery, deduplication"]
        PARTIAL_TESTS["Partial Response\nincomplete chunks, buffer assembly, cleanup"]
        CONCURRENT_TESTS["Concurrent Streams\nisolation, resource limits, cleanup on disconnect"]
    end

    FORMAT_TESTS --> MIDSTREAM_TESTS --> BACKPRESSURE_TESTS
    BACKPRESSURE_TESTS --> RECONNECT_TESTS --> PARTIAL_TESTS --> CONCURRENT_TESTS

    style FORMAT_TESTS fill:#22aa44,color:#fff
    style MIDSTREAM_TESTS fill:#ff4444,color:#fff
    style BACKPRESSURE_TESTS fill:#ff9900,color:#fff
    style RECONNECT_TESTS fill:#4488cc,color:#fff
    style PARTIAL_TESTS fill:#aa44aa,color:#fff
    style CONCURRENT_TESTS fill:#8866cc,color:#fff
```

### Format Compliance Tests

- Each of nine SSE event types emits correct structure when serialized.
- Session-meta event is always the first event in every stream.
- Done event is always the last event in every successful stream.
- Error event terminates stream and no further events follow.
- Text-delta events contain only text content, no metadata leakage.
- Trace-step events appear only when verbosity is set to full.
- CTA events contain valid schema: id, label, action type, and optional URL and icon, with maximum three per response.
- Location events contain coordinate data without exposing internal tool call details.
- Tripwire events contain conceptId and fallback text.

### Mid-Stream Failure Tests

- Provider error during generation: error event emitted, stream closed cleanly, no partial corruption in persisted data.
- Network drop between server and client: server-side cleanup runs, resources released, connection state cleared.
- Guardrail tripwire during stream in production mode: remaining chunks suppressed, fallback message injected, tripwire event emitted.
- Guardrail tripwire during stream in development mode: exception thrown with diagnostic information.
- Provider timeout during generation: stream closed after configured timeout, error event emitted.
- Memory extraction after partial stream: extraction skipped, short-term memory persisted with partial flag.
- Budget accounting after partial stream: actual token count reconciled against estimate, counter corrected.

### Backpressure Tests

- Slow consumer does not crash server or corrupt stream state.
- Server-side buffer limits prevent unbounded memory growth per connection.
- Fast producer with slow consumer: chunks buffered up to limit, then backpressure applied.
- Multiple slow consumers simultaneously: each connection independently managed without cross-contamination.
- Consumer disconnect during backpressure: connection cleanup runs, buffered data released.

### Reconnection Tests

- Client SDK reconnects automatically after connection drop.
- Reconnected client receives events from interruption point without duplicates or gaps.
- Client SDK offline queue persists messages during disconnection and syncs on reconnect.
- Multiple rapid disconnects and reconnects do not create duplicate connections or leak resources.
- Reconnection with expired authentication token: client re-authenticates before resuming.

### Concurrent Stream Tests

- Multiple simultaneous streams for the same user maintain independent state.
- Each stream tracks its own token budget consumption independently.
- Stream cancellation by one client does not affect other active streams.
- Server resource limits: maximum concurrent streams per user enforced without crash.
- Cleanup on disconnect: all per-connection resources released, no memory leaks across stream lifecycle.

### Zero-Leak Buffer Mode Tests

- Buffer fills to configured size before any bytes reach client.
- Violation detected during buffer phase: entire buffer suppressed, zero bytes sent, fallback injected.
- Buffer fills with no violation: buffer flushed to client, streaming mode begins with sliding window.
- Time-to-first-token delayed by exactly buffer fill duration under normal generation speed.
- Multiple violations during buffer phase: first violation triggers suppression, subsequent violations are no-ops.

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

## Per-Module Test Specifications

This section maps every plan document to specific test specifications, ensuring no feature area lacks verification. Each module lists its testable behaviors organized by test layer. Together with the Coverage Map, this guarantees exhaustive coverage across the entire system.

### Module: Requirements and Constraints (01)

All MH_* constraints are verified through their implementing module tests. All MN_* constraints are verified through audit scans and negative tests.

**Verification approach**:

- Every MH_* identifier has at least one test proving the behavior exists and functions correctly.
- Every MN_* identifier has at least one negative test proving the excluded behavior is absent.
- Definition of Done verification gates are exercised through task acceptance criteria in end-to-end suites.
- Requirement traceability: each test description references the MH or MN identifiers it covers.
- Constraint cross-referencing: tests for compound constraints (such as injection detection requiring ensemble, not single layer) verify the combined behavior.

### Module: System Architecture (03)

Architecture invariants are verified through integration and end-to-end tests rather than dedicated unit tests. These tests validate structural properties of the running system.

**Testable invariants**:

- Layer boundary enforcement: library imports do not pull server-specific modules and vice versa.
- Storage responsibility isolation: each storage system handles only its designated data categories per the storage decision table.
- Docker Compose profiles: default profile starts five services, Langfuse profile adds observability, Trigger profile adds background jobs.
- Graceful degradation topology: each optional service unavailability triggers the correct fallback per the degradation model.
- Horizontal scaling: stateless API design verified by running concurrent requests across multiple API instances sharing the same external services.
- Connection management: pool limits respected under load, PgBouncer integration functional.
- Network topology: all services communicate over single Docker bridge network with correct port assignments.
- Data flow invariants: chat request follows rate-check then budget-check then memory-load then LLM-stream sequence, with persistence and accounting after stream completion.
- File upload flow: synchronous upload phase completes under one second, asynchronous enrichment runs in background.

### Module: Foundation (04)

Foundation tests are primarily unit tests validating configuration, types, schemas, and shared utilities.

**Configuration system**:

- Deep merge behavior: skip undefined values, override null values, replace arrays, override functions, recurse objects.
- Configuration validation: invalid configs return descriptive validation errors through Zod schema reporting.
- Configuration builder: merges per-agent overrides over library defaults correctly.
- Frozen runtime config: config object is immutable after validation.
- No hardcoded deployment prompts: library defaults contain no business-specific prompt content.
- No built-in concept registry: library ships empty concept registry.

**Zod v4 schema validation**:

- All domain schemas accept valid input matching their type contracts.
- All domain schemas reject invalid input with descriptive error paths.
- Discriminated unions (TraceStepType, SSEEvent types) correctly discriminate on their tag field.
- VerbosityLevel schema accepts exactly "standard" and "full".
- GuardMode resolution: pipeline override wins over agent override wins over development default.

**Storage factory**:

- Explicit config always wins over auto-detection.
- Postgres branch returns Drizzle-backed store.
- Memory branch returns in-memory development store.
- Custom branch returns user-supplied implementation.
- Auto-detection uses database URL presence.
- Selection path is logged for debugging.
- SurrealDB storage uses surqlize typed APIs with no raw query strings.

**MCP health check**:

- Tool key parsing extracts server names using underscore-prefix ownership with greedy matching.
- Per-server status: has_matching_tools yields connected, zero tools with allowEmptyTools yields empty, missing tools yields failed.
- onFailure callback invoked with warn or throw behavior.
- Periodic health check scheduling functions correctly.
- No duplicate client creation when wrapper augments existing client.

**Provider resolution**:

- String identifiers resolve to correct provider path.
- Direct provider model instances pass through unchanged.
- Factory functions pass through unchanged.
- Fallback model wraps primary with ordered fallback chain.
- onFallback callback invoked when fallback is used.
- Both stream and generate paths use fallback middleware.

**Environment validation**:

- GOOGLE_API_KEY: comma-separated pool parsed into individual keys.
- JWT_SECRET: missing in production refuses startup, missing in development enables bypass.
- DATABASE_URL: hard-required, missing causes startup failure.
- SURREALDB_URL: missing disables long-term memory only.
- VALKEY_URL: missing falls back to in-memory cache.
- S3_ENDPOINT: missing disables upload path.
- TRIGGER_DEV_API_URL: missing runs jobs in-process.
- LANGFUSE variables: missing disables observability.

**Numeric configuration constants**:

- All thresholds (USER_SHORTTERM_LIMIT, USER_SHORTTERM_FADEOUT, ROLLING_SUMMARY_MAX_TOKENS, CONTEXT_WINDOW_BUDGET, MAX_RECALL_TOKENS, MAX_INPUT_MESSAGE_LENGTH, GIBBERISH_CONFIDENCE_THRESHOLD, RECENCY_BOOST values, TTL values) applied correctly at boundary values.

### Module: Conversation Pipeline (05)

Conversation pipeline tests span unit tests for individual phases and end-to-end tests for the complete six-phase flow.

**Phase 0 — input validation and language gate**:

- Fast language detection for clear allow and block paths.
- Ambiguous or mixed-language content proceeds to intent classification.
- Clear unsupported language triggers p0 tripwire block.
- Post-intent gate checks intendedOutputLanguage support.
- Translation edge cases: input in one language with intent to translate to another.

**Phase 1 — non-actionable detection**:

- Pleasantries, acknowledgments, and single-emoji messages short-circuit.
- Gibberish via entropy and language reliability signals short-circuits.
- Mixed actionable turns do not short-circuit.
- False-positive avoidance for short valid tokens, abbreviations, and non-Latin scripts.
- Short-circuit path skips embedding, LLM intent, rewriting, source routing, and fact extraction.

**Phase 2 — context assembly**:

- Three-layer context loading: thread short-term, user short-term, long-term recall.
- Trust hierarchy enforcement: system instruction highest, user messages medium, retrieved content zero trust.
- Retrieved content wrapped with reinforcement boundaries (sandwich framing, not delimiters alone).
- Zero-trust content containing instruction-like patterns: redacted or preserved with data-only framing.
- Unmarked zero-trust content never inserted into instruction-adjacent positions.

**Phase 3 — intent classification**:

- Embedding router: semantic hint generation with confidence scoring and Valkey cache interaction.
- LLM intent validator: structured output with all fields (intent, topics, rewritten query, clarification, language, temporal, dependent intents, negations, replay structure, behavioral signals).
- Speculative pre-fetching: starts on embedding guess, cancels on LLM disagreement.
- Correction detection: triggers re-interpretation of previous response.
- Frustration detection: signals de-escalation behavior.
- Topic abandonment: flushes stale topic context.
- Multi-intent decomposition: sub-queries with parallel or sequential processing.
- Dependent intents: dependency ordering with constraint passing between intents.
- Attribute negation: property-level exclusion distinct from entity exclusion.
- Query replay: parameter substitution with thread context recovery.
- Temporal resolution: phrases resolved to concrete ranges using user timezone with UTC fallback.

**Phase 4 — rewrite and source routing**:

- Seven rewrite triggers: pronoun referent, short query expansion, multi-intent separation, specific identifier preservation, jargon alignment, ordinal resolution, query replay reconstruction.
- Entity preservation guardrail: all original entities appear verbatim in rewrite, guardrail failure discards rewrite.
- Source priority: parallel fan-out across all configured sources with priority weighting at merge.
- Result merging: weight formula application, priority as scaling factor not absolute gate.
- Attribute negation application: per-source negation strategies (minus-prefix, contextual clause, negation-aware recall, negative preference framing).
- Empty result handling: normal versus suspicious empties with logging.

**RAGFlow integration**:

- Read-only retrieval: query mapping, chunk-to-citation mapping, dataset isolation.
- Topic-level dataset overrides take precedence over global defaults.
- Single request must not mix incompatible dataset embeddings.

### Module: Agents and Orchestration (06)

**Agent factory**:

- Creation with sensible defaults and full customizability.
- System prompt wiring through Gemini's dedicated system instruction field.
- Provider bridge wiring and model resolution.
- No hardcoded business prompts in library.

**Orchestrator**:

- Multi-intent supervision via handoffs between sub-agents.
- Sub-agent lifecycle management: creation, execution, result collection, cleanup.
- Live synthesis: streaming merge of sub-agent results into unified response.
- Dependent intent handling: sequential processing with constraint passing.
- Independent intent handling: parallel processing with result merging.

**Agent behaviors**:

- Auto-trigger memory recall on first message in new thread.
- Context budgeting: token allocation across layers with priority-based truncation.
- Response energy matching: input characteristics determine output calibration hint.
- Clarification patience model: after threshold turns synthesize best-effort answer.
- Location enrichment tool: pluggable geocoding, cached results, suppressed from stream.
- Tool registration: dynamic scoping per agent, namespace isolation, no duplicate instances.
- Grounding mode: separate agent mode for web-grounded responses with grounding metadata.
- Provider-agnostic configuration: no model-specific branching in core agent logic.

### Module: Memory and Intelligence (07)

Memory tests span unit tests for individual operations and end-to-end tests for the complete three-layer lifecycle.

**Thread short-term memory**:

- Load last ten turns from conversation store scoped by userId and threadId.
- Persist new turn after stream completes.
- Drop turns outside sliding window from context but keep in storage.
- Roll dropped turns into mandatory rolling summary.
- Edge cases: first turn returns zero turns, exactly ten turns fit in window, eleventh turn triggers summary.

**User short-term memory**:

- Cross-thread loading: load messages from other threads scoped by userId.
- Fade-out: injection stops after configured turn threshold.
- Framing: injected context strictly for ambiguity resolution, not proactive mention.
- Edge cases: no prior threads returns empty, user with many threads returns only configured limit.

**Rolling summaries**:

- Trigger on window overflow with incremental merge.
- Token budget enforcement and compaction when exceeded.
- Compaction priority: remove resolved items, collapse old detail, merge topics.
- Summary preserves topic trajectory, decisions, entities, and open loops.
- Summary excludes verbatim transcript and obsolete preferences.

**Thread resurrection**:

- Gap detection exceeding configured threshold.
- Entity extraction from rolling summary.
- Memory recall triggered with extracted entities.
- Staleness note injected.
- Edge cases: just under gap threshold yields no resurrection, just over triggers it.

**Fact extraction pipeline**:

- User-only content: never extracts facts from assistant-originated content.
- Attribution filter: self yields user fact, third_party yields interaction signal, general discarded.
- Certainty filter: stated yields store, hypothetical or asked yields discard.
- Sarcasm detection: inverts polarity from conversational context.
- Temporal classification: PAST, PRESENT, FUTURE states.
- Contradiction detection: attribute match plus semantic check triggers supersession.
- Deduplication: cosine similarity at 0.92 threshold with exact boundary behavior.
- Emotional context: extraction with decay counter, turn-by-turn decrement.
- Interaction signals: search queries, discussed entities, user actions, temporal markers.
- Media facts: vision description plus entity extraction from shared images.
- Injection defense: classify candidate facts, reject instruction-like patterns.
- Fire-and-forget: extraction failure logs error, skips extraction, persists short-term anyway.

**Fact supersession and deduplication**:

- Similarity search returns top five existing facts.
- Below 0.7: add new fact independently. Between 0.7 and 0.92: check predicate match. At or above 0.92: skip as duplicate.
- Predicate match with contradiction: supersede old fact with pointer and edge.
- Predicate match without contradiction: coexist.
- Superseded records kept for audit.
- Recall excludes superseded facts by default.

**Memory recall tool**:

- Auto-trigger on first message in new thread.
- Agent-controlled recall on subsequent turns.
- Semantic search on facts, interactions, and media facts.
- Graph traversal limited to two hops.
- Temporal filter when hint provided.
- Recency weighting: 24-hour 1.5x, 7-day 1.2x, beyond 7 days 1.0x.
- TTL refresh on recalled records.

**Memory control tools**:

- Inspect: loads active facts grouped by type, excludes superseded.
- Delete: search, show candidates, require confirmation, remove records and edges, purge cache.
- Edge cases: zero matches returns explicit message, user decline cancels deletion.

**Context window budget**:

- Track usage across all layers.
- Priority-based truncation: thread history highest, then user short-term, then long-term, then sources.
- Budget enforcement prevents context overflow.

**Memory poisoning defense**:

- Injection classifier at extraction time rejects instruction-like facts.
- Recalled facts framed as zero-trust data-only content.
- Periodic cleanup scans for injection patterns in stored facts.

### Module: Document Processing (08)

**Upload validation**:

- Magic bytes check (not extension-based) against known signatures.
- Size limit: 5MB per file exactly at boundary.
- Turn limit: 5 files per turn exactly at boundary.
- Quota check: reject if result exceeds user maximum.
- Supported types: PDF, DOCX, TXT, PNG, JPG, JPEG, WEBP.
- .doc legacy binary rejected with clear error message.
- Quota reservation rollback on processing failure.

**Document routing**:

- Images: direct mode. TXT up to 8K tokens: direct mode. TXT above 8K: RAG mode.
- PDF up to 6 pages: direct mode. PDF above 6: indexed mode.
- DOCX routing matches PDF after conversion.
- Exact boundary values: 6 pages is direct, 7 pages is indexed.

**DOCX conversion**:

- LibreOffice sidecar invocation over Docker network.
- Thirty-second timeout handling.
- Converted PDF stored in S3 with updated metadata.
- Temp file cleanup after conversion.
- Failed conversion marks file as failed.

**Per-page summarization**:

- PDF splitting into single-page outputs.
- Image extraction with 100px minimum dimension filter.
- Gemini structured output: summary 150-400 words, image descriptions, vector-chart flag.
- Embedding generation per page.
- Vector-chart fallback: PNG render when flag is true and no raster images.
- Concurrency control with configurable limit and round-robin key distribution.
- Retry failed pages up to three times with exponential backoff.
- Single failed page does not abort entire document.

**Background enrichment**:

- Raw text extraction and truncation to 1800 tokens.
- Embedding and tsvector generation.
- Failure reverts to ready state, file remains queryable from summaries.
- Dev mode fallback: in-process when Trigger.dev environment absent.

**File status state machine**:

- All transitions validated: uploading to summarizing, summarizing to ready, ready to enriching, enriching to enriched, enriching to ready on failure.
- Failure transitions: uploading to failed, summarizing to failed.
- Deletion transitions: ready or enriched to deleted.
- No backward transitions for any event sequence.

**Stranded file recovery**:

- TTL cleanup detects files in summarizing state beyond ten minutes.
- Reset and re-enqueue with idempotent re-execution.
- After three retries, transition to failed.

**Cleanup**:

- Per-file: requires both fileId and userId, removes S3 objects, page_index rows, vector chunks, marks deleted, releases quota.
- Per-thread: iterates all files for thread plus user.
- TTL: scheduled daily, finds expired non-deleted files.
- Idempotent: safe to run twice with same input.

### Module: Retrieval and Evidence (09)

**Hybrid search with RRF**:

- Three-arm fusion: summary vector, raw vector, keyword tsvector.
- Over-fetch 40 candidates per arm, RRF with k=50 smoothing.
- Group by file_id and page_number before score aggregation.
- Return top 10 pages.
- Cross-arm agreement dominates single-arm dominance.
- All arms execute through Drizzle-composed queries.

**Graceful degradation**:

- Summary-only mode: arms two and three return zero rows, arm one provides results.
- Enriched mode: all three arms contribute.
- Same user flow regardless of enrichment status.

**Content sanitization**:

- Injection pattern classifier evaluates each retrieved chunk.
- Strip external image embeds, dynamic URL patterns, hidden markup.
- Wrap chunks with explicit data-only boundaries.
- Latency cost approximately 50-100ms per retrieval.

**Structured citations**:

- Attribute-first generation: plan citations before prose.
- Citation fields: source, fileId, page, quote, scope, images.
- Presigned URLs with seven-day TTL.
- Quotes verbatim from raw text when available, summary-derived when summary-only.

**Evidence bundle gate**:

- Sufficiency scoring: coverage 0.4, confidence 0.4, completeness 0.2 weights.
- Gate open: plan citations and generate prose.
- Gate closed: hard refusal, soft caveat, or clarification per config.
- Thresholds configurable per topic.

**FileRegistry**:

- Temporal reference resolution: yesterday, last week, recently.
- Ordinal reference resolution: third document, first PDF.
- Named reference: fuzzy match against filenames.
- Timezone handling: user timezone from request header, UTC fallback.
- Ambiguity: ask user to clarify, no arbitrary guess.
- Cache invalidation on deletion or status changes.

**Cross-conversation RAG**:

- Thread scope versus global scope.
- Scope boost: thread results at full score, global at 0.85x multiplier.
- Global scope never crosses user boundaries.

**Anti-hallucination architecture**:

- Four-layer verification: FileRegistry resolution, evidence gate, attribute-first citations, transparent refusal.
- Phantom file reference caught at layer one.
- Weak evidence caught at layer two.
- Unsupported claims caught at layer three.
- Partial evidence explicitly noted at layer four.

**File edge cases**: all 28 documented scenarios across six categories (content Q&A, cross-reference, visual, ambiguity, conversational, format-specific) each have corresponding test coverage.

### Module: Guardrails and Safety (10)

**Input guardrail pipeline**:

- All guardrails run in parallel, all verdicts collected before aggregation.
- Worst-wins aggregation: p0 > p1 > p2.
- p0 triggers abort and tripwire, p1 triggers onFlag with message pass-through, p2 passes silently.
- Empty guardrail array means message always passes.
- No short-circuit on first p0 (all verdicts collected).
- Error in individual guardrail propagates, stream terminates.

**Injection detection ensemble**:

- Tier one heuristic: regex-driven detection of role-override, delimiter injection, encoding obfuscation, imperative patterns.
- Tier two ML classification: trained injection classifier.
- Tier three LLM-as-judge: lightweight judge model with structured output.
- All tiers run in parallel.
- Decision logic: LLM-alone blocks, two-tier triggers flag, single non-LLM passes.
- Edge cases: only tier one triggers yields pass, tier one plus tier two yields flag, tier three alone yields block.

**Output guardrail sliding window**:

- Configurable buffer size, default fifty tokens.
- Per-chunk evaluation against all output guardrails.
- Production mode: p0 suppresses all remaining chunks, injects fallback.
- Development mode: p0 throws tripwire exception.
- Multi-token patterns spanning chunk boundaries caught by sliding window.

**Output-side injection detection**:

- System prompt leakage: fingerprint-based matching catches verbatim and paraphrased leakage.
- Attention collapse: flag responses over-indexing on single chunk.

**Zero-leak buffered mode**:

- Buffer phase holds configured token count server-side.
- Violation during buffer: suppress entire buffer, zero bytes sent, inject fallback.
- Buffer full with no violation: flush and switch to streaming with sliding window.

**Guardrail factories**:

- Regex: patterns plus concept plus severity yields guardrail function.
- Keyword: keywords plus concept plus severity yields guardrail function, case-insensitive matching.
- LLM: classifier callback plus concept yields guardrail function.
- External moderation: endpoint plus concept plus failMode yields guardrail function.
- External failMode open: network errors return p2 with warning.
- External failMode closed: network errors return p0.
- Composite: constituents run in parallel, worst-wins aggregation.
- All factories return identical function signature.

**Language guard**:

- Fast detect: eld detection with confidence threshold.
- Post-intent gate: read intendedOutputLanguage from intent result.
- Output scanner: catch unsupported-language drift mid-stream with dominance thresholding.
- Edge cases: translation intent, mixed language with place names, short text below minimum length.

**Hate speech guard**:

- English engine: obscenity library with leet-speak, Unicode confusables, repetition handling.
- Multilingual engine: profanity library with Unicode word boundaries across supported languages.
- LDNOOBW supplement for thinner language coverage.
- Both engines run in parallel, any match returns p0.

**Escalation detection**:

- Sliding window anomaly scoring across turns.
- Behavioral signals: persona drift, privilege escalation sequences, context exhaustion.

### Module: Streaming and Transport (11)

**SSE event types**:

- All nine event types (session-meta, text-delta, trace-step, cta, citation, location, tripwire, done, error) conform to defined schemas.
- Each event type serializes to valid SSE format.

**Verbosity filtering**:

- Standard mode excludes trace-step events.
- Full mode includes all event types.
- Verbosity does not affect Langfuse tracing (always full).

**CTA streaming**:

- Tool-call events suppressed from stream.
- Clean CTA event emitted with schema validation: id, label, action type, optional URL and icon.
- Maximum three CTAs per response enforced.

**Location streaming**:

- Location-search tool-call and tool-result events suppressed from output.
- Clean location event emitted with coordinate data.

**Client SDK transport**:

- SSE parsing with correct line buffering for incomplete chunks.
- Event type discrimination and typed event object construction.
- Reconnection with automatic resume.
- Offline queue: persist messages locally, sync on reconnect.

### Module: Server Implementation (12)

**Middleware chain**:

- Ordering enforced: CORS before RequestID before LogContext before JWT before RateLimit before Budget before Handler.
- CORS runs first for preflight success without authentication.
- Request ID generated or forwarded from client header.
- JWT verification: signature, expiry, optional issuer and audience.
- Rate limiting per user and endpoint group with sliding window.
- Budget check reads counters from cache before agent execution.

**JWT authentication edge cases**:

- Missing authorization header returns 401 missing_token.
- Malformed bearer format returns 401 invalid_token.
- Invalid signature returns 401 invalid_token.
- Expired token returns 401 token_expired.
- Valid token with wrong role returns 403 insufficient_role.
- Missing JWT_SECRET in production refuses startup.
- Missing JWT_SECRET in development enables dev-bypass with warning.

**Startup validation**:

- Postgres connectivity validated before accepting traffic.
- Error map completeness validated against all typed error codes.
- Missing required environment values cause startup failure.
- Optional service unavailability logs warning with degraded mode.

**Route handlers**:

- Chat streaming: SSE response, agent resolution, verbosity control, usage accounting after stream.
- File upload: multipart handling, validation, blocking preparation, background enrichment enqueue.
- File management: list, detail, delete, status, page image redirect with signed URLs.
- Feedback: trace ownership validation, binary score, rate limiting.
- Admin budget: detail, update with optional reset, list with over-budget filter.
- Health: unauthenticated, parallel probes, five-second timeout, ok/degraded/down status.

**Error handling**:

- Every typed error code mapped to user-facing message.
- Missing error code mapping causes startup failure.
- Mid-stream errors emit typed error event and close stream.

**Graceful shutdown**:

- Stop accepting new connections on termination signal.
- Active streams signaled to drain with thirty-second timeout.
- Tracing backend flushed with ten-second timeout.
- Database and cache connections closed.
- New requests during shutdown receive 503 with Retry-After header.

### Module: TUI App (13)

**Setup and rendering**:

- Correct jsxImportSource for OpenTUI Solid compilation.
- Preload configuration for reactive state updates.
- Full-screen layout with four persistent panels.
- Terminal resize reflow.

**Keyboard handling**:

- Enter submits, Shift+Enter inserts newline.
- Ctrl+C cancels stream or quits with guarded behavior.
- Ctrl+L clears state and pending attachments.
- Up and Down arrow for input history recall.
- Tab for slash command autocomplete.
- Escape dismisses overlays.

**Command routing**:

- Slash commands intercepted before agent.
- Known commands dispatched to handlers.
- Unknown commands show inline error message.
- Tab completion works for all supported command names.

**Chat display**:

- Streaming markdown rendering with syntax-highlighted code blocks.
- Auto-scroll follows new content unless user scrolled up.
- Auto-scroll resumes when user returns to bottom.
- Only currently streaming message re-renders per chunk.

**Input component**:

- Multiline growth up to eight lines, then internal scrolling.
- Empty and whitespace-only submissions rejected.
- Input disabled during streaming, re-enabled when streaming ends.
- Local in-memory input history separate from chat history.

**Agent integration**:

- Direct mode: import library runtime and call execution directly.
- Client SDK mode: remote execution with equivalent stream semantics.
- Event normalization: same downstream handling regardless of mode.
- TripWire handling: exception caught, error banner displayed, fallback text shown.
- Stream cancellation: Ctrl+C cancels stream, partial response preserved.

**File upload flow**:

- File picker with type filtering, arrow navigation, multi-select.
- Magic bytes validation for type checking.
- Progress tracking through file status state machine.
- Pending attachment management through status bar.
- Cleanup on /clear command.
- Edge cases: oversize file, excess files, quota exceeded, unsupported type.

### Module: Observability (14)

**Langfuse integration**:

- Tracing exporter translates framework trace and span events into Langfuse API calls.
- Automatic span creation for agent runs, LLM generations, tool calls, guardrails, handoffs.
- Custom domain spans added for input guardrail, output guardrail, RAG pipeline, file processing, memory operations.
- No-op behavior when Langfuse not configured: all consumers work without null checks.

**Trace lifecycle**:

- Every agent interaction produces trace.
- traceId shared between SSE events and Langfuse traces.
- Verbosity controls SSE emission only, Langfuse always receives full trace.

**PII redaction**:

- Two-layer pipeline: deterministic entity detection plus structured-output LLM review.
- Default-on in production.
- Confidence failures route to quarantine.
- Redaction counters and reason tags visible for audit.

**User feedback**:

- Trace ownership validation prevents cross-user score submission.
- Binary score (zero or one) attached to trace.
- Rate limiting: ten requests per user per minute.
- Graceful handling when Langfuse absent: 200 response with traced false.

**Prompt management**:

- Startup preload from Langfuse with cache hydration.
- Variable interpolation with strict missing-variable checks.
- Stale-while-revalidate semantics on cache expiry.
- Circuit breaker prevents repeated latency spikes.
- Langfuse outage at startup loads fallback prompts.
- Langfuse outage at runtime falls back to local prompt.

**Self-test infrastructure**:

- Promptfoo subprocess spawning with ephemeral server.
- Config generation and result parsing.
- Clean dependency boundary: no Promptfoo library import.

### Module: Infrastructure (15)

**Docker Compose**:

- Core profile five services start correctly with health checks.
- Langfuse profile adds four services on non-conflicting ports.
- Trigger profile adds five services.
- Port conflict resolution: Valkey on 6379, Langfuse Redis on 6380, ClickHouse native on 9100, Langfuse web on 3100.

**Degradation model**:

- Postgres unavailable: hard failure, HTTP 503.
- JWT secret absent in production: hard failure, startup refused.
- Each optional service (SurrealDB, MinIO, Valkey, Trigger.dev, Langfuse) degrades gracefully per documented behavior.
- Valkey unavailable: budget fail-open, per-instance rate limiting, in-memory cache.

**Budget enforcement**:

- Pessimistic reservation: increment by estimate, check limit, rollback on over-limit.
- Reconciliation: adjust counter based on actual versus estimated usage.
- Daily and monthly counter auto-expiry.
- Atomic increment and expiration committed as one unit.
- Valkey unavailable returns allowed true with warning.
- Admin capabilities: detail, update with cache invalidation, paginated list.

**Rate limiting**:

- Sliding window sorted sets with Lua script atomicity.
- Member uniqueness using random UUID for same-millisecond handling.
- Retry-After derived from oldest in-window entry, clamped to minimum one second.
- Per-user keying with custom key extraction support.
- Development no-op fallback when Valkey unavailable.

**Circuit breaker**:

- Three-state machine: closed, open, half-open.
- Closed to open: after configured consecutive failures.
- Open to half-open: after reset timeout elapsed.
- Half-open to closed: probe succeeds.
- Half-open to open: probe fails.

**Trigger.dev integration**:

- Production dispatch to Trigger.dev task endpoint.
- Development in-process fallback executes same handlers.
- Registered tasks: background-enrichment, budget-aggregation, cleanup, expired-file-cleanup.

**Structured logging**:

- Hierarchical categories with async context propagation.
- Request correlation through requestId.

### Module: Execution Plan (17)

- Task dependency graph: all dependencies correctly specified with no circular dependencies.
- Task ordering: batches contain only tasks whose dependencies are satisfied by prior batches.
- Acceptance criteria: each task has verifiable criteria that can be mechanically checked.

### Module: Frontend SDK (18)

**Transport adapter**:

- SSE parsing with correct line buffering.
- Event type discrimination producing typed event objects.
- Reconnection with automatic resume and deduplication.

**React hooks**:

- Six hooks with correct behavior for chat, streaming state, feedback submission, trace display, file upload, and offline mode.
- Hooks integrate with AI SDK ChatTransport interface.

**Web components**:

- 48 ai-elements based components render correctly with proper ARIA attributes.
- Eight custom components (trace timeline, verbosity toggle, server switch, and others) function correctly.
- Component installation CLI provides individual component installation.

**Type safety**:

- SSE event types flow through client SDK to React hooks to UI components without type loss.
- Runtime event parsing validates against Zod schemas.

**Accessibility**:

- WCAG 2.1 AA compliance for all components.
- Keyboard navigation for all interactive elements.
- Screen reader compatibility with appropriate ARIA attributes.

**Offline queue**:

- Messages queued when server unreachable.
- Local persistence across app restarts.
- Auto-sync on reconnect with no duplicate delivery.

**Storybook**:

- Usage examples for all web components.
- Usage examples for React Native components.

### Module: Demos (19)

**Next.js demo**:

- All web components render and function correctly.
- Server switching between multiple safeagent instances.
- Verbosity toggle switches between standard and full modes.
- File upload through the complete pipeline.

**Expo demo**:

- Offline-first behavior: queue messages, persist locally, sync on reconnect.
- All React Native components render correctly.
- Server switching functions correctly.

**Eden Treaty integration**:

- Type-safe API calls through Elysia Eden Treaty client.

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
