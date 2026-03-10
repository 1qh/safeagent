# Domain Playbooks

This document provides domain-by-domain implementation paths so teams can execute in parallel without losing dependency discipline.

Batch execution control is defined in [Batch Delivery Playbook](./batch-delivery-playbook.md), and scheduling authority remains in [Execution Plan](./execution.md).

## How to Use Playbooks

1. Find your task in [Execution Plan](./execution.md).
2. Use the matching domain playbook below.
3. Follow the read order, then execute task specs, then complete verification.
4. Do not bypass dependency prerequisites from the execution matrix.

## Foundation and Contracts

### Scope

- [Foundation](./foundation.md)
- [Coding Standards](./coding-standards.md)
- [Developer Experience & Onboarding](./developer-experience.md)
- [API Governance](./api-governance.md)
- [Extensibility](./extensibility.md)

### Read Order

1. [Requirements & Constraints](./requirements.md)
2. [Foundation](./foundation.md)
3. [Coding Standards](./coding-standards.md)
4. [Developer Experience & Onboarding](./developer-experience.md)
5. [API Governance](./api-governance.md)
6. [Extensibility](./extensibility.md)

### Completion Checks

- Task acceptance criteria and QA scenarios complete.
- Required standards and governance constraints satisfied.
- Cross-domain contracts remain stable.

## Conversation and Agent Intelligence

### Scope

- [Conversation Pipeline](./conversation.md)
- [Agents & Orchestration](./agents.md)
- [Memory & Intelligence](./memory.md)

### Read Order

1. [Conversation Pipeline](./conversation.md)
2. [Agents & Orchestration](./agents.md)
3. [Memory & Intelligence](./memory.md)
4. [Testing](./testing.md)

### Completion Checks

- Multi-intent and dependent-intent behavior is verified.
- Memory, context budget, and calibration behavior are verified.
- Task-linked test assertions in domain test sections are satisfied.

## Retrieval, Documents, and Safety

### Scope

- [Document Intelligence](./documents.md)
- [Retrieval & Evidence](./retrieval.md)
- [Guardrails & Safety](./guardrails.md)

### Read Order

1. [Document Intelligence](./documents.md)
2. [Retrieval & Evidence](./retrieval.md)
3. [Guardrails & Safety](./guardrails.md)
4. [Unified Security Strategy and Regulatory Compliance](./security-compliance.md)

### Completion Checks

- Evidence sufficiency and citation behavior verified.
- Input and output safety paths verified.
- Security and compliance controls remain intact.

## Transport, Server, and Surfaces

### Scope

- [Streaming & Transport](./transport.md)
- [Server Implementation](./server.md)
- [TUI App](./tui.md)
- [Frontend SDK](./frontend-sdk.md)
- [Demos](./demos.md)

### Read Order

1. [Streaming & Transport](./transport.md)
2. [Server Implementation](./server.md)
3. [Frontend SDK](./frontend-sdk.md)
4. [TUI App](./tui.md)
5. [Demos](./demos.md)

### Completion Checks

- Streaming event behavior and suppression rules verified.
- Thin-server boundaries preserved.
- Surface clients remain compatible with shared contracts.

## Runtime, Scale, and Operations

### Scope

- [Infrastructure](./infrastructure.md)
- [Observability](./observability.md)
- [Monitoring](./monitoring.md)
- [AI Operations Plan](./ai-operations.md)
- [Durable Execution](./durable-execution.md)
- [Unified Security Strategy and Regulatory Compliance](./security-compliance.md)

### Read Order

1. [Infrastructure](./infrastructure.md)
2. [Observability](./observability.md)
3. [Monitoring](./monitoring.md)
4. [AI Operations Plan](./ai-operations.md)
5. [Durable Execution](./durable-execution.md)
6. [Unified Security Strategy and Regulatory Compliance](./security-compliance.md)

### Completion Checks

- Runtime resilience and fallback behavior verified.
- Observability and incident controls verified.
- Cost, governance, and durability controls verified.

## Delivery, Docs, and Release

### Scope

- [Documentation](./documentation.md)
- [Release Pipeline](./release-pipeline.md)
- [Testing](./testing.md)
- [Execution Plan](./execution.md)

### Read Order

1. [Documentation](./documentation.md)
2. [Release Pipeline](./release-pipeline.md)
3. [Testing](./testing.md)
4. [Execution Plan](./execution.md)

### Completion Checks

- Verification gates and release gates complete.
- Audit tasks complete with no unresolved scope drift.
- Delivery readiness confirmed for both library and thin server paths.
