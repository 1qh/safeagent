# Operations

This document gives operators and implementers a single operational reading path across observability, monitoring, infrastructure, security, and AI runtime operations.

---

## Operational Responsibility Map

- Trace quality, spans, prompt telemetry, and feedback signals: [Observability](./observability.md)
- Health checks, alerts, escalation, incident runbooks: [Monitoring](./monitoring.md)
- Runtime capacity, cache, rate limit, budget controls, resilience patterns: [Infrastructure](./infrastructure.md)
- Security governance, threat model, compliance operations: [Unified Security Strategy and Regulatory Compliance](./security-compliance.md)
- Runtime cost intelligence and model-operation controls: [AI Operations Plan](./ai-operations.md)

---

## Operational Lifecycle

1. Detect using health and telemetry signals
2. Classify severity and blast radius
3. Contain by policy and runtime controls
4. Recover through runbook-guided actions
5. Verify system health and quality signals
6. Record findings and feed back into prevention controls

---

## Deployment Readiness Checks

- Required health checks are defined and reachable
- Alert routes and escalation ownership are defined
- Budget and rate-limit controls are configured
- Security controls and audit trails are active
- Incident procedures are accessible and rehearsable

---

## Cross-Plan Links

- Execution and dependency timing: [Execution Plan](./execution.md)
- Server runtime integration boundaries: [Server Implementation](./server.md)
- Verification and release gates: [Quality Gates](./quality-gates.md)
- Runtime optimization governance: [AI Operations Plan](./ai-operations.md)
- Readiness visibility and escalation posture: [Readiness Scoreboard](./readiness-scoreboard.md)
- Release and audit decision capture: [Readiness Decision Record Template](./readiness-decision-record-template.md)
