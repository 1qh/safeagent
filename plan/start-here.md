# Start Here

This document is the reader-first entry point for the plan. It tells builders what to read, in what order, and where each decision source lives.

---

## Fast Reading Order

1. [Requirements & Constraints](./requirements.md)
2. [Constraints](./constraints.md)
3. [System Plan Overview](./overview.md)
4. [System Architecture](./architecture.md)
5. [Execution Plan](./execution.md)
6. Assigned domain document (for the task you are implementing)
7. [Quality Gates](./quality-gates.md)
8. [Testing](./testing.md)
9. [Operations](./operations.md)

---

## If You Are Assigned a Task

1. Open [Execution Plan](./execution.md).
2. Find your task in the routing index and dependency matrix.
3. Read the task spec in the mapped domain document.
4. Read the linked test assertions and quality gates.
5. Implement only after dependency prerequisites are complete.
6. Verify against task acceptance criteria, QA scenarios, and quality gates.

---

## Single Source Rules

- Scheduling and dependency order: [Execution Plan](./execution.md)
- Product requirements and boundary constraints: [Requirements & Constraints](./requirements.md)
- Consolidated operating constraints: [Constraints](./constraints.md)
- Technical rationale and research decisions: [Research & Decisions](./research.md)
- Verification policy and audit tasks: [Testing](./testing.md) and [Quality Gates](./quality-gates.md)
- Runtime operation and incident handling: [Operations](./operations.md)

---

## Domain Reading Map

- Core interaction and orchestration: [Conversation Pipeline](./conversation.md), [Agents & Orchestration](./agents.md), [Memory & Intelligence](./memory.md)
- Retrieval and evidence: [Retrieval & Evidence](./retrieval.md), [Document Intelligence](./documents.md), [Guardrails & Safety](./guardrails.md)
- Delivery surface: [Streaming & Transport](./transport.md), [Server Implementation](./server.md), [TUI App](./tui.md), [Frontend SDK](./frontend-sdk.md)
- Runtime and scale: [Infrastructure](./infrastructure.md), [Observability](./observability.md), [Monitoring](./monitoring.md), [Security](./security-compliance.md), [AI Operations](./ai-operations.md)
- Release and governance: [Coding Standards](./coding-standards.md), [Release Pipeline](./release-pipeline.md), [Documentation](./documentation.md), [Developer Experience](./developer-experience.md), [API Governance](./api-governance.md)

---

## No-Loss Guarantee

The current workspace preserves all original planning content from `plan_v0` and layers this reader-first entry document on top. See [Coverage Matrix](./coverage-matrix.md) for one-to-one preservation tracking.
