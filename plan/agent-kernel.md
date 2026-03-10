# Agent Kernel

Minimal, agent-first operating view. Read this first.

## Authority Map

- Scheduling + deps + routing: [Execution Plan](./execution.md)
- Task behavior contract: task spec in routed `Spec File`
- Verification owner mapping: [Task Verification Index](./task-verification-index.md)
- Gate order + required evidence: [Quality Gates](./quality-gates.md)
- Constraints boundary: [Requirements & Constraints](./requirements.md), [Constraints](./constraints.md)

## Minimal Task Flow

1. Get `Task ID` in [Execution Plan](./execution.md) routing index.
2. Read routed `Spec File` + `Context Sections`.
3. Validate requirement + constraint compliance from [Requirements & Constraints](./requirements.md) and [Constraints](./constraints.md).
4. Complete `Acceptance Criteria` + `QA Scenarios`.
5. Validate mapped behavioral assertions from [Task Verification Index](./task-verification-index.md).
6. Validate cross-domain regression using routed `Context Sections`.
7. Produce handoff evidence via [Handoff Packet Template](./handoff-packet-template.md).
8. Confirm required task-level gates in [Quality Gates](./quality-gates.md).

## Hard Rules

- Do not define or infer alternate scheduling order outside `execution.md`.
- Do not replace authoritative gate criteria with derived rollups.
- Do not mark task complete without acceptance + QA + mapped test-owner checks.
- Keep docs compact per [Compact Authoring Standard](./compact-authoring-standard.md).

## Read-Only Support Views

- Readiness rollup: [Readiness Scoreboard](./readiness-scoreboard.md)
- Stress coverage map: [Eval and Adversarial Matrix](./eval-adversarial-matrix.md)
- Drift checks: [Plan Drift Control](./plan-drift-control.md)

## Scope

- Current routed task count target: 129
- If counts mismatch between docs, trust `execution.md` routing index and reconcile others.
