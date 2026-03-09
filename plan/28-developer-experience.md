# 28 — Developer Experience & Onboarding

> **Scope**: Developer Experience and onboarding governance for `safeagent` from first encounter to expert usage.
>
> **Tasks**: DEVELOPER_ENTRY_EXPERIENCE, PROGRESSIVE_API_EXPERIENCE, ERROR_EXPERIENCE_GOVERNANCE, LOCAL_EXPERIENCE_ENABLEMENT, STUDIO_EXPERIENCE, TESTING_EXPERIENCE, TEMPLATE_EXPERIENCE, AI_ASSISTANT_EXPERIENCE, TYPESCRIPT_PERFORMANCE_GOVERNANCE, OBSERVABILITY_EXPERIENCE, TOOL_CREATION_EXPERIENCE

---

## Table of Contents
- [Strategic Context](#strategic-context)
- [Project Scaffolding](#project-scaffolding)
- [Progressive API Design](#progressive-api-design)
- [Error Taxonomy and Diagnostics](#error-taxonomy-and-diagnostics)
- [Local Development Environment](#local-development-environment)
- [Interactive Development Studio](#interactive-development-studio)
- [Testing Utilities](#testing-utilities)
- [Template and Starter Ecosystem](#template-and-starter-ecosystem)
- [AI Coding Agent Integration](#ai-coding-agent-integration)
- [TypeScript Performance Budget](#typescript-performance-budget)
- [OpenTelemetry and Observability Integration](#opentelemetry-and-observability-integration)
- [Tool Development Workflow](#tool-development-workflow)
- [Delivery Checklist](#delivery-checklist)
- [Mermaid Diagrams](#mermaid-diagrams)
- [Reference URLs](#reference-urls)
- [Navigation](#navigation)

## Strategic Context
- Developer Experience is a first-order adoption risk, not a documentation afterthought.
- Onboarding quality directly determines community trust, paid adoption velocity, and long-term ecosystem depth.
- `safeagent` governance in this file defines how beginner productivity and expert control coexist without conflict.
- The framework commitment is Bun-only runtime and toolchain behavior with one npm package: `safeagent`.
- Package fragmentation is explicitly out of scope; scoped package families are disallowed for this initiative.
- SurrealDB integration is constrained to surqlize for consistent behavior and durable query governance.
- PostgreSQL integration is constrained to Drizzle for schema consistency and migration discipline.
- The onboarding model must remain local-first, network-optional, and practical for individual developers before team rollout.
- This plan extends and harmonizes testing, observability, TUI, demos, and extensibility directives from adjacent files.
- Every experience decision in this file must improve one of: time-to-first-output, confidence-to-ship, or mean-time-to-diagnose.

### Experience Principles
- Zero-friction entry is mandatory for first-time developers.
- Progressive disclosure is mandatory for intermediate and expert growth.
- Runtime safety and type safety must reinforce each other.
- Diagnostics must be actionable without support escalation.
- Local development must work without cloud dependency.
- Instrumentation must be default-visible and low overhead.
- Template quality must be production-credible, not tutorial-only.
- Testing must support deterministic local confidence and high-signal CI gates.
- Observability export must stay framework-agnostic and standards-based.
- Tool creation must prioritize isolation, replayability, and publication portability.

### Outcome Commitments
- New developers can reach visible output quickly and repeat that success without hidden setup complexity.
- Teams can adopt advanced controls in stages instead of facing one monolithic interface.
- Error handling can be triaged by category and domain without fragile pattern matching.
- Local environments can run full development loops with no internet reliance for core capabilities.
- AI coding assistants can load lightweight context by default and expand guidance on demand.
- Type-checking and editor responsiveness remain measurable quality gates under high tool counts.
- Tracing and telemetry can be exported to all compatible platforms through OpenTelemetry.

## Project Scaffolding

### Intent
- Scaffolding must reduce first-contact uncertainty and produce confidence immediately.
- The generator concept is interactive and user-guided rather than static copy-only templates.
- Initial setup must prioritize local execution and visible outcome instead of credential setup.

### Mandatory Requirements
- The project generator is interactive and designed around guided choices.
- Template selection includes provider choice, use case orientation, and framework integration posture.
- First run must produce visible output with no hidden prerequisite beyond documented local setup.
- An environment template is required and must default to local-first provider behavior.
- Local model provider defaults must be preferred over cloud-first assumptions.
- Scaffolding must preserve Bun-only workflows and `safeagent` single-package constraints.
- Generated outputs must avoid hidden dependency on network-only infrastructure.

### Governance Rules
- Scaffolding prompts must use clear language that explains implications of each choice.
- Template metadata must identify intended audience: beginner, intermediate, expert.
- Generated projects must include onboarding notes focused on outcome validation, not internal mechanics.
- Generated projects must include explicit indicators for optional cloud augmentation, not baseline dependency.
- Generator evolution must be reviewed every release iteration with onboarding telemetry.

### Acceptance Signals
- Median time from scaffold start to first visible output remains below an agreed threshold.
- Setup failure reports trend downward across each release iteration.
- First-time users can recover from setup issues using provided diagnostics guidance without external help.
- New developer satisfaction improves in surveys tied to initial workflow completion.

### Non-Goals
- Scaffolding does not attempt to replace architecture governance from foundational files.
- Scaffolding does not expose expert-only controls during beginner flow unless explicitly requested.
- Scaffolding does not force cloud account setup during initial flow.

## Progressive API Design

### Intent
- The API surface must scale with developer maturity while preserving a stable happy path.
- Beginners must achieve useful outcomes without facing complexity intended for advanced use.

### Three-Tier API Surface
- Tier one provides a quick one-liner experience for immediate productivity.
- Tier two provides standard configuration for common production adjustments.
- Tier three provides expert builder composition with middleware extensibility.
- Tier transitions must be incremental and understandable, not disruptive rewrites.
- Each tier must have clear decision criteria that describe when to advance.

### Mandatory Requirements
- Beginners are never forced into expert controls to complete core tasks.
- Happy-path usage is zero-config for common scenarios.
- Schema-as-single-source-of-truth is mandatory using Zod v4.
- Zod v4 schema definitions must drive runtime validation and TypeScript typing consistently.
- Tool results must use discriminated unions for safe narrowing.
- Public API contracts must avoid untyped escape hatches as outward-facing return shapes.
- Progressive surface area must remain Bun-compatible without alternate runtime assumptions.

### Design Governance
- Tier one language emphasizes outcomes and defaults.
- Tier two language emphasizes explicit control and predictable override behavior.
- Tier three language emphasizes composition, policy layering, and lifecycle controls.
- Cross-tier semantics must remain aligned to avoid conceptual drift.
- Migration across tiers must preserve behavior clarity and user confidence.

### Safety and Type Integrity
- Discriminated union outcomes must include stable identifiers and structured payload variants.
- Runtime validation failures must map to documented error taxonomy categories.
- Type narrowing patterns must remain robust across package boundary usage.
- Public contracts must preserve backward-compatible shape expectations through release changes.

### Adoption Metrics
- Percentage of users shipping successfully on tier one without escalation.
- Percentage of users moving from tier one to tier two for sustained production needs.
- Percentage of expert adopters using middleware composition successfully.
- Reduction in support requests tied to unclear API surface choices.

## Error Taxonomy and Diagnostics

### Intent
- Error behavior must enable fast, confident resolution instead of generic failure states.
- Developers must understand what failed, why it failed, and what to do next.

### Named Error Hierarchy
The system SHALL provide a minimum of fifteen distinct named error types covering: configuration validation, schema validation, agent execution, authentication and authorization, rate limiting, context window exhaustion, maximum step limits, tool resolution failure, invalid tool results, memory operations, guardrail violations, provider communication, transport failures, budget enforcement, and timeout conditions.

### Error Domain Classification
The error taxonomy SHALL classify every error into one of at least eight semantic domains: tool operations, agent lifecycle, memory operations, retrieval operations, provider communication, transport layer, guardrail enforcement, and configuration.

### Error Category Classification
The error taxonomy SHALL classify every error into one of three fault-origin categories:
- Developer-originated faults from mistakes and invalid setup assumptions.
- Framework-originated faults from internal defects and contract failures.
- External faults from provider, network, or dependency failures.

### Message Quality Contract
- Every error message must answer: what happened.
- Every error message must answer: why it happened.
- Every error message must answer: what should happen next.
- Every error message must include context fields aligned to domain and category.
- Diagnostic payloads must remain concise enough for terminal readability.
- Human-readable guidance must remain present even when machine-readable metadata exists.

### Isolation and Resilience
- Step-level error isolation is required so one tool failure does not terminate entire agent execution.
- Partial success outcomes must remain visible with structured failure records.
- Recovery policies must support retry, skip, and fallback decisions by error category.
- Error boundaries must prevent cascading failures across independent steps.

### Type-Safe Identification Pattern
- Error identification must use stable discriminants and structured metadata.
- Cross-boundary recognition must not rely on fragile inheritance-based identification.
- Identification patterns must be deterministic across transpilation or bundling differences.
- Consumer logic must be able to branch on domain and category safely.

### Diagnostic Maturity Targets
- Time-to-understand primary failure cause declines with each release iteration.
- Support escalations caused by ambiguous error messaging decline across onboarding cohorts.
- Reproducibility quality improves through enriched context and trace linkage.

## Local Development Environment

### Intent
- Local development is the default operating mode, not a reduced or secondary path.
- Internet access and paid credentials must not block baseline framework learning and iteration.

### Mandatory Requirements
- Ollama support is first-class and documented as primary local provider path.
- Mock provider mode must require zero API keys for local work.
- Core architecture must be offline-capable for memory, tools, and tracing workflows.
- Environment validation concept must check setup readiness and return actionable guidance.
- Provider abstraction must support switching through a single configuration decision point.
- Local defaults must avoid accidental cloud coupling.

### Local-First Governance
- Local provider posture must remain parity-tested against cloud-backed providers for core flows.
- Offline behavior must degrade gracefully and report precise missing capabilities.
- Development feedback loops must remain fast under local-only execution.
- Local traces must remain accessible for root-cause analysis with no external service dependency.

### SurrealDB and PostgreSQL Constraints
- SurrealDB usage is REQUIRED through surqlize for consistency and developer trust.
- PostgreSQL usage is REQUIRED through Drizzle for relational schema governance.
- Storage guidance must preserve one conceptual model across local and team environments.

### Adoption Criteria
- New developer can complete local setup and first run without cloud keys.
- Mock mode enables deterministic learning and testing without billing exposure.
- Offline work remains productive for core design, testing, and diagnostics loops.

### External Reference
- Official Ollama documentation: https://ollama.com/

## Interactive Development Studio

### Intent
- Developers need a visual development workspace that exposes runtime behavior and iteration quality.
- The studio concept complements terminal workflows with trace-aware feedback.

### Mandatory Requirements
- Local browser-based development UI concept is required.
- Agent chat interface is required for real-time behavioral testing.
- Step trace viewer with timing is required.
- Tool isolation testing is required before agent attachment.
- Token usage visibility is required.
- Cost visibility is required where applicable.
- Middleware-based tracing activation is required with zero-config behavior.
- Traces are stored locally and automatically excluded from source control workflows.

### UX Governance
- Studio information hierarchy must prioritize current run status, step timeline, and outcome clarity.
- Visual states must communicate success, warning, retry, and failure distinctly.
- Timing data must be readable at both step granularity and end-to-end granularity.
- Token and cost views must differentiate estimate and observed values clearly.

### Operational Governance
- Studio must remain optional for teams that prefer terminal-only workflows.
- Studio telemetry must align with privacy and compliance controls from security and observability plans.
- Studio data retention defaults must favor short-lived local inspection.

### External References
- Official Vercel AI SDK documentation: https://ai-sdk.dev/docs
- Official Mastra Studio documentation: https://mastra.ai/docs/getting-started/studio

## Testing Utilities

### Intent
- Testing must give developers confidence at three horizons: immediate local correctness, CI quality gates, and production realism.

### Mandatory Utility Set
- Mock provider must support fixture-based responses.
- Mock provider must support streaming simulation.
- Mock provider must support controlled error injection.
- Agent run simulation helper must return complete step trace for assertions.
- Fixture recording mode must capture real responses for deterministic replay.
- Custom test matchers must support expressive agent assertions.
- Deterministic mode must include fixed temperature behavior.
- Deterministic mode must include seeded identifiers.
- Deterministic mode must include retry suppression.

### Testing Pyramid Requirements
- Unit layer uses mock providers and prioritizes speed and zero external cost.
- Eval layer uses model-judged quality checks and acts as CI gating signal.
- Integration layer uses real provider behavior on scheduled cadence.
- Layer boundaries must remain clear so signal quality is interpretable.
- Promotion of checks between layers must be deliberate and justified.

### Fixture Governance
- Fixtures must preserve semantic coverage across happy path and failure path scenarios.
- Recording cadence must prevent drift from live provider behavior.
- Replay determinism must be validated on every release iteration.
- Sensitive data capture in fixtures must be prohibited by default.

### CI Governance
- Unit checks run on every change.
- Eval checks run as merge gating criteria according to reliability policy.
- Integration checks run on scheduled cadence with trend tracking.
- Failure triage must classify whether issue is test artifact, framework behavior, or provider drift.

### Quality Signals
- Mean unit runtime remains fast enough for frequent local execution.
- Eval gate stability improves across release iterations.
- Integration findings produce actionable follow-up rather than intermittent noise.

### External Reference
- Official Vitest documentation: https://vitest.dev/

## Template and Starter Ecosystem

### Intent
- Templates accelerate adoption by giving trustworthy starting points for real workloads.

### Mandatory Template Set
- Basic agent template.
- Tool-enabled agent template.
- Multi-agent collaboration template.
- Retrieval-augmented generation template.
- Human-in-the-loop template.
- Local-model-first template.

### Quality Bar
- Every template is standalone and runnable immediately.
- Every template demonstrates visible output on first run.
- Every template includes clear adaptation notes for real deployment contexts.
- Every template must align with Bun-only and single-package package policy.
- Every template must align with local-first defaults and offline-friendly posture.

### Framework Integration Guidance
- Integration guides must cover Elysia and Hono.
- Integration guidance must preserve framework-native patterns while maintaining safeagent conventions.
- Guidance must explain how to adopt progressively from minimal use to advanced composition.

### Example Ecosystem
- A top-level examples directory SHALL contain standalone runnable examples for focused concepts.
- Each example SHALL be independently executable with its own declared dependencies.
- Example entries must state intended audience and required baseline knowledge.
- Example curation must prioritize realistic patterns over toy-only demonstrations.
- Example refresh cadence must track major release iterations and deprecate stale patterns.
- Example directory governance must define ownership, minimum quality standards, and review requirements.

### Ecosystem Governance
- Template review board must approve additions based on quality and maintenance ownership.
- Community contributions must meet reliability and onboarding criteria before inclusion.
- Template lifecycle states must be explicit: active, maintenance, or archived.

## AI Coding Agent Integration

### Intent
- AI-assisted development should accelerate adoption while preserving framework correctness.

### Mandatory Requirements
- Provide an agent skill artifact for major coding assistants.
- Support assistant ecosystems including Claude Code, Cursor, and Gemini CLI.
- Progressive disclosure model is required.
- Lightweight metadata loads by default around a small token budget.
- Full instruction set loads on demand when deeper guidance is needed.

### Governance Requirements
- Metadata layer must include core principles, safety constraints, and key terminology.
- Expanded instruction layer must include deeper design guidance and troubleshooting heuristics.
- Assistant guidance must remain aligned with current release behavior and migration notes.
- Drift detection must identify outdated assistant guidance before release publication.

### Adoption Outcomes
- Developers receive context-aware suggestions that match framework intent.
- Assistant-guided outputs reduce onboarding mistakes and policy violations.
- Expert teams can use assistant support without sacrificing architectural control.

## TypeScript Performance Budget

### Intent
- Type system quality must remain a productivity multiplier, not a latency tax.

### Mandatory Requirements
- CI must enforce a budget ensuring fast type-check behavior with 30 or more registered tools.
- Complex Zod schema workloads must be included in performance checks.
- IDE responsiveness must be treated as measurable quality.
- Regression thresholds must trigger intervention before release promotion.

### Measurement Model
- Track end-to-end type-check duration under representative project scale.
- Track incremental edit feedback latency in common development flows.
- Track memory pressure under high-schema and high-tool workloads.
- Track diagnostic latency for complex discriminated union outcomes.

### Governance Policy
- Performance budgets must be visible and reviewed each release iteration.
- Budget exceptions require explicit risk acceptance and remediation plan.
- Improvements should focus on contract design and type simplification where safe.
- New API additions must include expected type-performance impact analysis.

### Quality Signals
- Type-check duration remains stable as tool count grows.
- Editor responsiveness remains predictable for advanced schema usage.
- Developer-reported sluggishness decreases after optimization initiatives.

## OpenTelemetry and Observability Integration

### Intent
- Observability must connect onboarding, reliability, and governance through one standards-based trace model.

### Mandatory Requirements
- Framework-agnostic trace export through OpenTelemetry is required.
- Compatibility with Langfuse is required.
- Compatibility with Braintrust is required.
- Compatibility with Jaeger is required.
- This plan must extend observability direction defined in file 14.

### Observability Governance
- Trace schema must capture step lifecycle, tool invocations, guardrail decisions, and error category metadata.
- Trace correlation must support end-to-end diagnosis from onboarding failures to production incidents.
- Sampling policy must balance cost control and diagnostic completeness.
- Local trace persistence must support short-loop debugging with easy cleanup.

### Integration Expectations
- Export behavior must avoid coupling to a single vendor data model.
- Instrumentation defaults must be safe, informative, and low overhead.
- Teams must be able to route telemetry to existing operations stacks without custom rewrites.

### External Reference
- Official OpenTelemetry documentation: https://opentelemetry.io/docs/

## Tool Development Workflow

### Intent
- Tool creation must be safe, testable, and publishable without hidden integration risk.

### Mandatory Requirements
- Tool sandbox is required for isolated validation before agent attachment.
- Trace replay is required to reproduce specific executions.
- MCP-first publishing model is required so each tool can be published as an MCP server.
- A structured tool metadata format is required for discovery and governance.

### Workflow Governance
- Tool contracts must define input and output schema expectations clearly.
- Tool sandbox runs must include success path and failure path validation.
- Replay workflows must preserve timing and context metadata sufficient for diagnosis.
- Publishing readiness must include security, reliability, and documentation checks.

### Discovery Governance
- Registry metadata must include ownership, maturity, trust level, and support policy.
- Discovery experience must prioritize safe defaults and clear capability descriptions.
- Deprecated tools must remain discoverable with migration guidance until retirement window closes.

### Operational Outcomes
- Tool integration defects are detected earlier through sandbox isolation.
- Reproducibility improves through trace replay practices.
- Ecosystem growth accelerates through standardized publishing and discovery.

## Delivery Checklist

### This File Delivers
- Governance model for end-to-end Developer Experience from first contact through expert operations.
- Project scaffolding strategy with local-first defaults and visible first-run outcome requirements.
- Progressive API model with three-tier disclosure and type-safe schema governance via Zod v4.
- Comprehensive error taxonomy, domain and category classification, and actionable diagnostics contract.
- Local development architecture requirements including Ollama-first and offline-capable workflows.
- Interactive development studio requirements for chat testing, trace viewing, and cost transparency.
- Testing utility strategy with deterministic replay, fixture recording, and layered quality gates.
- Template ecosystem strategy with six production-grade starter categories and framework guidance.
- AI coding assistant integration strategy using progressive disclosure and bounded metadata loading.
- TypeScript performance governance with measurable CI and IDE responsiveness budgets.
- OpenTelemetry integration requirements aligned with broader observability strategy.
- Tool development lifecycle governance including sandboxing, replay, and MCP-first publishing.

### Cross-Reference Table
| Related File | Relationship to This File | Dependency Direction | Coordination Requirement |
|---|---|---|---|
| [04 — Foundation](./04-foundation.md) | Establishes baseline architecture and runtime policy | Foundation constrains DX design choices | Keep Bun-only and single-package constraints aligned |
| [13 — TUI](./13-tui.md) | Defines terminal interaction patterns that complement studio workflows | TUI informs local developer interaction ergonomics | Align diagnostics language and trace affordances |
| [14 — Observability](./14-observability.md) | Defines trace and telemetry semantics extended here | Observability underpins diagnostics and studio visibility | Keep OpenTelemetry payload semantics consistent |
| [16 — Testing](./16-testing.md) | Defines core testing strategy expanded here for DX | Testing enables onboarding confidence and CI policy | Harmonize deterministic test posture and quality gates |
| [19 — Demos](./19-demos.md) | Provides adoption-facing demonstrations and runnable narratives | Demos accelerate onboarding and template validation | Keep examples synchronized with template ecosystem |
| [20 — Documentation](./20-documentation.md) | Defines docs architecture and publishing standards | Documentation carries onboarding and migration guidance | Ensure progressive disclosure language remains consistent |
| [23 — Coding Standards](./23-coding-standards.md) | Sets implementation discipline and quality norms | Standards affect template quality and assistant guidance | Ensure diagnostics and type safety constraints stay aligned |
| [24 — Extensibility](./24-extensibility.md) | Governs extension boundaries and trust controls | Extensibility enables MCP-first tool publishing | Align registry policy, sandboxing, and trust metadata |

### Delivery Acceptance Gates
- All required sections in this file are present and populated with governance language.
- Mermaid diagrams are present with UPPER_SNAKE task identifiers.
- Diagram labels remain conceptual and avoid implementation-specific naming.
- Bun-only, single-package, surqlize, and Drizzle constraints are explicit.
- Error taxonomy includes at least fifteen named error types and required domain and category classifications.
- Progressive API tiering includes zero-config happy path and expert disclosure boundaries.
- Testing pyramid and deterministic tooling model are explicitly defined.
- OpenTelemetry compatibility requirements and file 14 extension note are explicit.

## Operational Rollout Governance

### Rollout Phases
- Phase one focuses on onboarding reliability and first-run visibility outcomes.
- Phase two focuses on progressive API confidence and diagnostics maturity.
- Phase three focuses on studio adoption, telemetry export quality, and template ecosystem depth.
- Phase four focuses on expert workflows, MCP-first tool publishing maturity, and partner enablement.

### Entry Criteria Per Phase
- Phase one entry requires stable scaffold outcomes and local-first defaults.
- Phase two entry requires tiered API clarity and discriminated union consistency.
- Phase three entry requires trace visibility, token insight, and local retention policy readiness.
- Phase four entry requires reproducible tool workflows and registry governance maturity.

### Exit Criteria Per Phase
- Phase one exit requires low setup failure trend and positive first-use sentiment.
- Phase two exit requires declining API confusion incidents and improved type-safety confidence.
- Phase three exit requires sustained trace usefulness and reduced diagnosis latency.
- Phase four exit requires ecosystem contribution quality and predictable publication outcomes.

### Risk Register Focus Areas
- Onboarding fragility risk from hidden assumptions in scaffold defaults.
- API confusion risk from unclear tier boundaries and naming consistency.
- Diagnostic ambiguity risk from insufficient context in failure payloads.
- Local parity risk where offline behavior diverges from connected behavior.
- Template staleness risk from missing maintenance ownership.
- Assistant guidance drift risk from outdated skills metadata.
- Performance regression risk from complex schema growth and tool scale.

### Mitigation Posture
- Maintain mandatory readiness checks before rollout phase transitions.
- Run periodic onboarding simulations with mixed experience-level cohorts.
- Tie diagnostics quality to measurable support outcomes.
- Enforce deterministic testing gates before broad rollout expansion.
- Require observability checks for feature areas affecting runtime trust.

### Governance Cadence
- Weekly triage for onboarding blockers and diagnostics quality findings.
- Biweekly review for template quality, example health, and framework guide accuracy.
- Monthly review for type-performance budgets and editor responsiveness trend.
- Monthly review for local development parity against connected provider behavior.
- Quarterly review for assistant guidance drift and ecosystem impact metrics.

### Ownership Model
- Product ownership defines adoption goals and release prioritization.
- Engineering ownership defines technical quality bars and gating checks.
- Developer relations ownership defines ecosystem feedback loops and adoption narratives.
- Reliability ownership defines observability fitness and diagnostics standards.
- Security ownership verifies policy alignment for local traces and publishing workflows.

## Adoption Metrics and Quality Signals

### Onboarding Metrics
- Time from first encounter to first visible output.
- Percentage of first-time users reaching successful first run.
- Recovery success rate after initial setup errors.
- Repeat usage rate within the first adoption window.

### API Experience Metrics
- Distribution of usage across quick, standard, and expert tiers.
- Upgrade rate from quick tier to standard tier over time.
- Upgrade rate from standard tier to expert tier for advanced projects.
- Error density linked to misapplied tier assumptions.

### Diagnostics Metrics
- Mean-time-to-diagnose for common failure categories.
- Percentage of errors resolved without external support.
- Percentage of errors with complete context fields.
- Rate of repeated failures caused by unclear remediation guidance.

### Local Development Metrics
- Local-first setup completion rate without cloud credentials.
- Offline workflow success rate for core development loops.
- Mock-provider usage rate during onboarding and testing.
- Provider-switch success rate through single configuration decision point.

### Studio Metrics
- Studio adoption rate across active developer cohort.
- Trace-view engagement rate during debugging sessions.
- Tool isolation usage before agent attachment.
- Token and cost panel usage during optimization work.

### Testing Metrics
- Unit suite median runtime and stability trend.
- Eval gate pass rate and false-positive rate.
- Integration trend quality across scheduled execution windows.
- Deterministic replay success rate across recorded fixtures.

### Ecosystem Metrics
- Template activation rate by category.
- Example completion rate for standalone scenarios.
- Community contribution acceptance rate for templates.
- Ecosystem maintenance backlog age distribution.

### Performance Metrics
- Type-check latency under high tool counts.
- Editor diagnostic response latency under complex schemas.
- Trend of performance exceptions accepted versus remediated.
- Impact of schema complexity on developer workflow throughput.

### Observability Metrics
- Trace completeness ratio across execution paths.
- Correlation success rate between diagnostics and traces.
- Export success rate to compatible observability backends.
- Sampling strategy effectiveness for balancing cost and insight.

### Reporting Expectations
- Metrics must be reviewable by engineering, product, and developer relations stakeholders.
- Trends must be tracked by release iteration with root-cause annotations.
- Metric regressions must trigger documented response plans.
- Metrics that no longer inform decisions must be retired deliberately.

## Mermaid Diagrams

### Developer Journey Flow
```mermaid
flowchart LR
    FIRST_ENCOUNTER[FIRST_ENCOUNTER]
    SCAFFOLD_SELECTION[SCAFFOLD_SELECTION]
    FIRST_VISIBLE_OUTPUT[FIRST_VISIBLE_OUTPUT]
    ITERATIVE_DEVELOPMENT[ITERATIVE_DEVELOPMENT]
    QUALITY_VALIDATION[QUALITY_VALIDATION]
    RELEASE_READINESS[RELEASE_READINESS]
    EXPERT_PRACTICE[EXPERT_PRACTICE]

    FIRST_ENCOUNTER --> SCAFFOLD_SELECTION
    SCAFFOLD_SELECTION --> FIRST_VISIBLE_OUTPUT
    FIRST_VISIBLE_OUTPUT --> ITERATIVE_DEVELOPMENT
    ITERATIVE_DEVELOPMENT --> QUALITY_VALIDATION
    QUALITY_VALIDATION --> RELEASE_READINESS
    RELEASE_READINESS --> EXPERT_PRACTICE

    ITERATIVE_DEVELOPMENT -->|Trace Insight| ITERATIVE_DEVELOPMENT
    QUALITY_VALIDATION -->|Deterministic Replay| QUALITY_VALIDATION
```

### Progressive API Disclosure
```mermaid
flowchart TB
    QUICK_ONE_LINER[QUICK_ONE_LINER]
    STANDARD_CONFIGURATION[STANDARD_CONFIGURATION]
    EXPERT_BUILDER[EXPERT_BUILDER]
    MIDDLEWARE_COMPOSITION[MIDDLEWARE_COMPOSITION]
    SCHEMA_SINGLE_SOURCE[SCHEMA_SINGLE_SOURCE]
    DISCRIMINATED_RESULTS[DISCRIMINATED_RESULTS]

    QUICK_ONE_LINER --> STANDARD_CONFIGURATION
    STANDARD_CONFIGURATION --> EXPERT_BUILDER
    EXPERT_BUILDER --> MIDDLEWARE_COMPOSITION

    QUICK_ONE_LINER --> SCHEMA_SINGLE_SOURCE
    STANDARD_CONFIGURATION --> SCHEMA_SINGLE_SOURCE
    EXPERT_BUILDER --> DISCRIMINATED_RESULTS
    SCHEMA_SINGLE_SOURCE --> DISCRIMINATED_RESULTS
```

### Testing Pyramid
```mermaid
flowchart TB
    UNIT_LAYER[UNIT_LAYER_FAST_FREE]
    EVAL_LAYER[EVAL_LAYER_CI_GATE]
    INTEGRATION_LAYER[INTEGRATION_LAYER_REAL_PROVIDER]
    QUALITY_CONFIDENCE[QUALITY_CONFIDENCE]

    UNIT_LAYER --> EVAL_LAYER
    EVAL_LAYER --> INTEGRATION_LAYER
    INTEGRATION_LAYER --> QUALITY_CONFIDENCE

    UNIT_LAYER -->|Deterministic Fixtures| QUALITY_CONFIDENCE
    EVAL_LAYER -->|Quality Thresholds| QUALITY_CONFIDENCE
```

## Reference URLs
- Vercel AI SDK: https://ai-sdk.dev/docs
- Mastra Studio: https://mastra.ai/docs/getting-started/studio
- OpenTelemetry: https://opentelemetry.io/docs/
- Vitest: https://vitest.dev/
- Ollama: https://ollama.com/

## Navigation
---
Previous: [27 — Security & Compliance](./27-security-compliance.md) | Next: [29 — API Governance & Consumer Migration](./29-api-governance.md)
