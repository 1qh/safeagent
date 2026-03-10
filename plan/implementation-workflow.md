# Implementation Workflow

This document defines the execution workflow from task assignment to final completion evidence. It does not define task order; scheduling authority remains in [Execution Plan](./execution.md).

## Definition of Ready

A task is ready to implement only when all checks below are true:

- Task exists in [Execution Plan](./execution.md) routing index and dependency matrix.
- Task dependencies are completed according to execution ordering.
- Task specification is identified and read in its mapped spec file.
- Behavioral assertion ownership is identified in [Task Verification Index](./task-verification-index.md).
- Relevant constraints are reviewed in [Requirements & Constraints](./requirements.md) and [Constraints](./constraints.md).

## Task Delivery Packet

Before implementation starts, prepare this packet:

- `Task ID`: semantic UPPER_SNAKE identifier
- `Batch ID`: from [Execution Plan](./execution.md)
- `Spec File`: authoritative task specification location
- `Context Files`: required supporting documents from routing index
- `Behavioral Test Owner`: mapped test-spec file from [Task Verification Index](./task-verification-index.md)
- `Quality Gates`: required gates from [Quality Gates](./quality-gates.md)
- `Risk Notes`: security, scalability, and reliability concerns for this task

Packet identity fields are sourced from the [Execution Plan](./execution.md) per-task routing index. Do not create a separate task registry outside execution routing.

## Execution Loop

1. Read task specification and acceptance criteria.
2. Read QA scenarios for the same task.
3. Read behavioral assertions in the mapped test-spec owner file.
4. Implement task scope only after dependency prerequisites are satisfied.
5. Verify acceptance criteria and QA scenarios.
6. Verify behavioral assertions and quality gates.
7. Record completion evidence for handoff or review.

## Verification Order

Use this fixed order for every task:

1. Requirement and constraint compliance from [Requirements & Constraints](./requirements.md) and [Constraints](./constraints.md)
2. Task acceptance criteria in the task specification
3. Task QA scenarios in the task specification
4. Behavioral assertions in the mapped test-spec owner file
5. Cross-domain checks required by context files
6. Task handoff evidence completeness using [Handoff Packet Template](./handoff-packet-template.md)

Release and audit gates from [Quality Gates](./quality-gates.md) are tracked as project-level readiness impacts, not per-task completion prerequisites.

## Handoff Evidence Contract

A task is handoff-ready only when evidence includes:

- Task ID and completion timestamp
- Acceptance criteria checklist status
- QA scenario checklist status
- Behavioral assertion owner file and validated assertion groups
- Any exceptions, deferred scope notes, or risk follow-ups

## Escalation and Mismatch Policy

- If task spec and execution routing conflict, [Execution Plan](./execution.md) decides scheduling and dependency order.
- If test ownership appears ambiguous, [Task Verification Index](./task-verification-index.md) is the tie-breaker.
- If constraints conflict with implementation convenience, constraints win.
- Any unresolved mismatch is documented and escalated before marking task complete.

## Related Documents

- [Start Here](./start-here.md)
- [Execution Plan](./execution.md)
- [Task Verification Index](./task-verification-index.md)
- [Batch Delivery Playbook](./batch-delivery-playbook.md)
- [Handoff Packet Template](./handoff-packet-template.md)
- [Quality Gates](./quality-gates.md)
- [Testing](./testing.md)
