# Batch Delivery Playbook

This playbook operationalizes batch execution for implementation teams. It does not replace scheduling truth; [Execution Plan](./execution.md) remains authoritative for ordering, dependencies, and routing.

---

## How to Use

1. Open [Execution Plan](./execution.md) and identify the active batch.
2. Confirm dependencies are complete for all tasks planned in the batch.
3. Use this playbook for batch-level readiness, coordination rhythm, and closeout evidence.
4. Use [Implementation Workflow](./implementation-workflow.md) and [Handoff Packet Template](./handoff-packet-template.md) for task-level delivery evidence.

---

## Batch Execution Rhythm

### 1) Readiness

- Confirm predecessor batch exit criteria are satisfied.
- Confirm task owners have routing context and verification ownership.
- Confirm shared risks and constraints are understood by all implementers.

### 2) Kickoff

- Confirm batch scope (task IDs only from execution routing index).
- Confirm dependency-sensitive tasks run first inside the batch window.
- Confirm review and verification owners for each task.

### 3) In-Batch Delivery

- Track task completion against acceptance criteria and QA scenarios.
- Track behavioral assertion ownership completion per task.
- Surface dependency blockers immediately and rebalance parallel work.

### 4) Closeout

- Verify batch exit criteria and handoff evidence completeness.
- Record unresolved risks and deferred follow-ups.
- Release next batch only when batch exit checks are complete.

---

## Batch Readiness Checklist

- Previous batch completed and accepted.
- Active batch task IDs confirmed against [Execution Plan](./execution.md).
- Verification ownership confirmed in [Task Verification Index](./task-verification-index.md).
- Task-level handoff evidence structure agreed.
- Constraints and quality gates reviewed for relevant task scopes.

---

## Batch Exit Checklist

- All batch tasks meet acceptance criteria and QA scenarios.
- Behavioral assertions for each task are validated in mapped owner files.
- Required quality gates are complete for batch scope.
- Handoff packets complete for all finished tasks.
- Dependency matrix has no unresolved blocker for next batch.

---

## Batch-by-Batch Delivery Guide

| Batch ID | Delivery Focus | Entry Requirement | Exit Evidence |
|---|---|---|---|
| `BLOCKING_SPIKE_BATCH` | Core technology baseline validation | Initial planning complete | Baseline stack assumptions validated |
| `RAG_VALIDATION_BATCH` | Retrieval dependency validation | `BLOCKING_SPIKE_BATCH` complete | Retrieval dependency chain validated |
| `SCAFFOLDING_BATCH` | Repository scaffolding and standards baseline | `RAG_VALIDATION_BATCH` complete | Library/server scaffolds and standards baseline complete |
| `FOUNDATION_A_BATCH` | Core types, storage, MCP, platform foundations | `SCAFFOLDING_BATCH` complete | Foundation layer A tasks accepted |
| `FOUNDATION_B_BATCH` | Schemas, memory core, observability/cache foundations | `FOUNDATION_A_BATCH` complete | Foundation layer B tasks accepted |
| `CONFIG_GUARDS_BATCH` | Configuration, guardrails, extensibility, tenant controls | `FOUNDATION_B_BATCH` complete | Config and safety baseline accepted |
| `AGENT_PIPELINE_BATCH` | Agent factory and document/file pipeline core | `CONFIG_GUARDS_BATCH` complete | Agent/pipeline foundation accepted |
| `INTEGRATION_BATCH` | Core integration layer and runtime interaction | `AGENT_PIPELINE_BATCH` complete | Integration layer accepted |
| `SELFTEST_MIDINTEGRATION_BATCH` | Mid-integration hardening and self-test controls | `INTEGRATION_BATCH` complete | Hardening controls accepted |
| `SERVER_TUI_PIPELINE_BATCH` | Server/TUI/search/intent integration | `SELFTEST_MIDINTEGRATION_BATCH` complete | Server/TUI integration accepted |
| `EXTENDED_INTEGRATION_BATCH` | Extended orchestration and integration behaviors | `SERVER_TUI_PIPELINE_BATCH` complete | Extended orchestration accepted |
| `SERVER_ROUTES_SUBAGENT_BATCH` | Server route and sub-agent assembly layer | `EXTENDED_INTEGRATION_BATCH` complete | Route and sub-agent assembly accepted |
| `ENDPOINTS_BARREL_BATCH` | Endpoint surface and barrel exports | `SERVER_ROUTES_SUBAGENT_BATCH` complete | Endpoint and export surface accepted |
| `E2E_DEPLOY_BATCH` | End-to-end, deploy, release, ops infra readiness | `ENDPOINTS_BARREL_BATCH` complete | E2E, release, and ops readiness accepted |
| `FRONTEND_DEMOS_BATCH` | Demo delivery and incident/doc experience readiness | `E2E_DEPLOY_BATCH` complete | Demo and operational runbook readiness accepted |
| `FINAL_AUDIT_BATCH` | Final audits and governance closure | `FRONTEND_DEMOS_BATCH` complete | Final audit evidence complete |

---

## Team Split Guidance

- Foundation team: scaffolding, contracts, configuration baseline.
- Intelligence team: conversation, agents, memory, retrieval behavior.
- Surface team: transport, server, TUI, frontend SDK, demos.
- Reliability team: infrastructure, observability, monitoring, security, release.

Cross-team coordination is required for tasks with shared context files in routing index entries.

---

## Handoff Contract

Every completed task in a batch should provide a completed packet from [Handoff Packet Template](./handoff-packet-template.md). Batch closeout requires all packets for the active batch.

---

## Related Documents

- [Execution Plan](./execution.md)
- [Implementation Workflow](./implementation-workflow.md)
- [Task Verification Index](./task-verification-index.md)
- [Quality Gates](./quality-gates.md)
