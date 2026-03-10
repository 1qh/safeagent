# Quality Gates

This document defines task-level completion gates and separate project-level readiness gates.

---

## Task-Level Gate Sequence

1. Requirement and constraint compliance
2. Task acceptance criteria completion
3. QA scenario completion
4. Domain test assertion coverage
5. Cross-domain regression validation
6. Task handoff evidence completeness

---

## Project-Level Readiness Gates

These gates are not per-task prerequisites. They are project-level completion gates aligned with later execution batches.

1. Release pipeline quality checks
2. Final audit readiness

---

## Required Evidence per Task

- Task acceptance criteria satisfied in the domain task spec
- QA scenarios covered and validated
- Test assertions mapped and passing for the owning domain
- No dependency-order violation against execution matrix
- No safety or compliance rule regression

---

## Task-Level Gate Evidence Artifacts

| Task-Level Gate | Required Evidence Artifact | Authoritative Source |
|---|---|---|
| Requirement and constraint compliance | Constraint Compliance Record | [Requirements & Constraints](./requirements.md), [Constraints](./constraints.md) |
| Task acceptance criteria completion | Acceptance Criteria Checklist | Domain task specification file |
| QA scenario completion | QA Scenario Evidence Record | Domain task specification file |
| Domain test assertion coverage | Test Assertion Validation Record | Mapped owner in [Task Verification Index](./task-verification-index.md) |
| Cross-domain regression validation | Cross-Domain Regression Note | Context files listed in [Execution Plan](./execution.md) routing index |
| Task handoff evidence completeness | Completed handoff packet | [Handoff Packet Template](./handoff-packet-template.md) |

---

## Release Readiness Gates

- Coding standards gate is satisfied
- CI pipeline gate is satisfied
- Release pipeline gate is satisfied
- Security and compliance gate is satisfied
- Monitoring and observability readiness is satisfied

---

## Audit Completion Gates

- Plan audit scope is complete
- Code-quality audit scope is complete
- QA audit scope is complete
- Scope-fidelity audit scope is complete
- API governance audit scope is complete

---

## Project-Level Gate Evidence Artifacts

| Project-Level Gate | Required Evidence Artifact | Authoritative Source |
|---|---|---|
| Release pipeline quality checks | Release Gate Decision Record | [Release Pipeline](./release-pipeline.md), [Quality Gates](./quality-gates.md), [Readiness Decision Record Template](./readiness-decision-record-template.md) |
| Final audit readiness | Audit Closure Decision Record | [Testing](./testing.md), [Execution Plan](./execution.md), [Readiness Decision Record Template](./readiness-decision-record-template.md) |

---

## Source Documents

- Requirement-level done criteria: [Requirements & Constraints](./requirements.md)
- Test policy and coverage maps: [Testing](./testing.md)
- Task-level verification ownership: [Task Verification Index](./task-verification-index.md)
- Task completion evidence model: [Implementation Workflow](./implementation-workflow.md)
- Batch-level execution sequencing support: [Batch Delivery Playbook](./batch-delivery-playbook.md)
- Task handoff evidence contract: [Handoff Packet Template](./handoff-packet-template.md)
- Readiness rollup visibility: [Readiness Scoreboard](./readiness-scoreboard.md)
- Project-level readiness decision records: [Readiness Decision Record Template](./readiness-decision-record-template.md)
- Capability stress validation matrix: [Eval and Adversarial Matrix](./eval-adversarial-matrix.md)
- Compact writing policy for gate documentation updates: [Compact Authoring Standard](./compact-authoring-standard.md)
- CI and release controls: [Release Pipeline](./release-pipeline.md)
- Security and compliance controls: [Unified Security Strategy and Regulatory Compliance](./security-compliance.md)
- Execution dependency gates: [Execution Plan](./execution.md)
