# Eval and Adversarial Matrix

This matrix is supplementary to [Testing](./testing.md) and domain test specifications. It does not define new test categories or override test policy.

---

## Purpose

- Validate that behavior quality is stable across normal and adversarial prompts.
- Detect regressions before release and before audit closure.
- Tie every stress test group to clear ownership and response actions.

---

## Capability Evaluation Matrix

| Capability Area | Normal Evaluation Focus | Adversarial Evaluation Focus | Primary Owner Docs |
|---|---|---|---|
| Intent and Orchestration | Intent routing accuracy, sub-agent coordination, synthesis quality | Intent confusion prompts, dependency-trap prompts, contradiction prompts | [Conversation Pipeline](./conversation.md), [Agents & Orchestration](./agents.md) |
| Memory and Context | Recall precision, context budgeting behavior, consistency across long threads | Prompt injection through memory context, stale-reference traps, conflict memories | [Memory & Intelligence](./memory.md), [Agents & Orchestration](./agents.md) |
| Retrieval and Evidence | Citation relevance, evidence sufficiency, source prioritization | Hallucination pressure prompts, fabricated source prompts, low-evidence coercion | [Retrieval & Evidence](./retrieval.md), [Document Intelligence](./documents.md) |
| Guardrails and Safety | Input/output guard behavior, language safety, zero-leak behavior | Policy bypass prompts, obfuscation attacks, multilingual toxicity evasions | [Guardrails & Safety](./guardrails.md), [Unified Security Strategy and Regulatory Compliance](./security-compliance.md) |
| Transport and Server Surface | Streaming coherence, endpoint behavior, auth boundary consistency | Stream abuse prompts, event-flood behavior, unauthorized endpoint attempts | [Streaming & Transport](./transport.md), [Server Implementation](./server.md) |
| Runtime and Reliability | Circuit-breaker behavior, retry patterns, incident detection fidelity | Degradation storms, cascading timeouts, noisy failure bursts | [Infrastructure](./infrastructure.md), [Monitoring](./monitoring.md), [Operations](./operations.md) |

---

## Adversarial Test Families

- Prompt injection and policy override pressure
- Citation forgery and false-grounding pressure
- Multi-intent ambiguity and contradiction pressure
- Memory poisoning and stale-context pressure
- Streaming abuse and output-channel corruption pressure
- Safety bypass through obfuscation and multilingual evasion

---

## Execution Cadence

- Per task completion: run relevant normal evaluation checks.
- Per batch closeout: run batch-relevant adversarial family checks.
- Per release readiness cycle: run full matrix sweep.
- Before final audits: run full matrix sweep with documented closure evidence.

---

## Failure Response Policy

1. Log failing case with task and capability ownership.
2. Block readiness progression for affected dimension.
3. Apply remediation in owning domain.
4. Re-run impacted matrix sections until stable.
5. Record closure in handoff packet and readiness scoreboard.

---

## Related Documents

- [Readiness Scoreboard](./readiness-scoreboard.md)
- [Quality Gates](./quality-gates.md)
- [Implementation Workflow](./implementation-workflow.md)
- [Handoff Packet Template](./handoff-packet-template.md)
- [Plan Drift Control](./plan-drift-control.md)
