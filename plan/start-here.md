# Start Here

This is the primary entry point for implementation work. Use it to choose a reading path, find task ownership quickly, and avoid cross-file searching.

---

## First Hour Path

1. Read [Requirements & Constraints](./requirements.md) and [Constraints](./constraints.md).
2. Read [System Plan Overview](./overview.md) and [System Architecture](./architecture.md).
3. Read [Execution Plan](./execution.md) and locate your assigned task.
4. Open the domain document listed in the execution routing index.
5. Read [Implementation Workflow](./implementation-workflow.md).
6. Read [Task Verification Index](./task-verification-index.md) for behavioral test ownership.
7. Read [Batch Delivery Playbook](./batch-delivery-playbook.md) for batch-level execution rhythm.
8. Read [Quality Gates](./quality-gates.md) and [Testing](./testing.md) before implementation.
9. Use [Operations](./operations.md) if your task touches runtime behavior.

---

## Role-Based Reading Paths

### Builder (assigned TASK_X)

1. [Execution Plan](./execution.md) → locate TASK_X in the per-task routing index.
2. Open the mapped domain document and complete task acceptance criteria and QA scenarios.
3. Use [Task Verification Index](./task-verification-index.md) to find the behavioral test owner file.
4. Follow [Implementation Workflow](./implementation-workflow.md) for completion evidence sequencing.
5. Validate against [Quality Gates](./quality-gates.md).

### Technical Lead (ordering and risk)

1. [Execution Plan](./execution.md) for dependency order and critical path.
2. [Requirements & Constraints](./requirements.md) for non-negotiable boundaries.
3. [Research & Decisions](./research.md) for decision rationale.
4. [Domain Playbooks](./domain-playbooks.md) for domain execution planning.
5. [Batch Delivery Playbook](./batch-delivery-playbook.md) for batch-level control.
6. [Plan Drift Control](./plan-drift-control.md) for consistency checks.
7. [AI Operations Plan](./ai-operations.md) for runtime optimization strategy.

### QA and Release

1. [Quality Gates](./quality-gates.md)
2. [Testing](./testing.md)
3. [Release Pipeline](./release-pipeline.md)
4. [Task Verification Index](./task-verification-index.md)
5. [Handoff Packet Template](./handoff-packet-template.md)
6. [Readiness Scoreboard](./readiness-scoreboard.md)
7. [Eval and Adversarial Matrix](./eval-adversarial-matrix.md)
8. [Coverage Matrix](./coverage-matrix.md)

### Operations and Security

1. [Operations](./operations.md)
2. [Infrastructure](./infrastructure.md)
3. [Observability](./observability.md)
4. [Monitoring](./monitoring.md)
5. [Unified Security Strategy and Regulatory Compliance](./security-compliance.md)

---

## Single Source Rules

- Scheduling, dependencies, and task routing: [Execution Plan](./execution.md)
- Product boundaries and done criteria: [Requirements & Constraints](./requirements.md)
- Consolidated non-negotiables: [Constraints](./constraints.md)
- Testing and verification policy: [Testing](./testing.md), [Quality Gates](./quality-gates.md)
- Task completion sequencing and evidence model: [Implementation Workflow](./implementation-workflow.md)
- Task-level behavioral ownership mapping: [Task Verification Index](./task-verification-index.md)
- Batch-level execution rhythm: [Batch Delivery Playbook](./batch-delivery-playbook.md)
- Readiness scoring and release posture: [Readiness Scoreboard](./readiness-scoreboard.md)
- Adversarial and capability stress testing: [Eval and Adversarial Matrix](./eval-adversarial-matrix.md)
- Runtime improvement cycle: [AI Operations Plan](./ai-operations.md)
- Ongoing consistency control: [Plan Drift Control](./plan-drift-control.md)
- Operational readiness and incidents: [Operations](./operations.md)
- Architectural rationale and validated decisions: [Research & Decisions](./research.md)

---

## Domain Reading Map

- Core interaction and orchestration: [Conversation Pipeline](./conversation.md), [Agents & Orchestration](./agents.md), [Memory & Intelligence](./memory.md)
- Retrieval and evidence: [Retrieval & Evidence](./retrieval.md), [Document Intelligence](./documents.md), [Guardrails & Safety](./guardrails.md)
- Delivery surface: [Streaming & Transport](./transport.md), [Server Implementation](./server.md), [TUI App](./tui.md), [Frontend SDK](./frontend-sdk.md)
- Runtime and scale: [Infrastructure](./infrastructure.md), [Observability](./observability.md), [Monitoring](./monitoring.md), [AI Operations Plan](./ai-operations.md), [Unified Security Strategy and Regulatory Compliance](./security-compliance.md)
- Governance and release: [Coding Standards](./coding-standards.md), [Developer Experience](./developer-experience.md), [Documentation](./documentation.md), [Release Pipeline](./release-pipeline.md), [API Governance](./api-governance.md)

For domain-by-domain implementation routes, see [Domain Playbooks](./domain-playbooks.md).

---

## Preservation Note

All original planning documents from `plan_v0` are preserved in `plan`, with reader-first guidance layered on top. See [Coverage Matrix](./coverage-matrix.md) for preservation tracking.
