# Quality Gates

This document defines the verification sequence that every implementation task must pass before it is considered complete.

---

## Gate Sequence

1. Requirement and constraint compliance
2. Task acceptance criteria completion
3. QA scenario completion
4. Domain test assertion coverage
5. Cross-domain regression validation
6. Release pipeline quality checks
7. Final audit readiness

---

## Required Evidence per Task

- Task acceptance criteria satisfied in the domain task spec
- QA scenarios covered and validated
- Test assertions mapped and passing for the owning domain
- No dependency-order violation against execution matrix
- No safety or compliance rule regression

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

## Source Documents

- Requirement-level done criteria: [Requirements & Constraints](./requirements.md)
- Test policy and coverage maps: [Testing](./testing.md)
- Task-level verification ownership: [Task Verification Index](./task-verification-index.md)
- Task completion evidence model: [Implementation Workflow](./implementation-workflow.md)
- Batch-level execution sequencing support: [Batch Delivery Playbook](./batch-delivery-playbook.md)
- Task handoff evidence contract: [Handoff Packet Template](./handoff-packet-template.md)
- CI and release controls: [Release Pipeline](./release-pipeline.md)
- Security and compliance controls: [Unified Security Strategy and Regulatory Compliance](./security-compliance.md)
- Execution dependency gates: [Execution Plan](./execution.md)
