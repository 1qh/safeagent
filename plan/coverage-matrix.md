# Coverage Matrix

This matrix tracks preservation from `plan_v0` into the current `plan` workspace so no original planning content is lost while the new reader-first layer is added.

---

## Preservation Status

- Source files in `plan_v0`: 30
- Preserved source files in `plan`: 30
- Missing files: 0
- Additional reader-first files in `plan`: `start-here.md`, `constraints.md`, `quality-gates.md`, `operations.md`, `coverage-matrix.md`, `domain-playbooks.md`, `implementation-workflow.md`, `task-verification-index.md`, `batch-delivery-playbook.md`, `handoff-packet-template.md`, `plan-drift-control.md`, `readiness-scoreboard.md`, `readiness-decision-record-template.md`, `eval-adversarial-matrix.md`, `compact-authoring-standard.md`

---

## Source-to-Current Preservation Map

| Source (`plan_v0`) | Preserved (`plan`) | Status |
|---|---|---|
| `overview.md` | `overview.md` | Preserved (additive reader-first links) |
| `requirements.md` | `requirements.md` | Preserved |
| `research.md` | `research.md` | Preserved |
| `architecture.md` | `architecture.md` | Preserved |
| `coding-standards.md` | `coding-standards.md` | Preserved |
| `api-governance.md` | `api-governance.md` | Preserved |
| `developer-experience.md` | `developer-experience.md` | Preserved |
| `foundation.md` | `foundation.md` | Preserved |
| `extensibility.md` | `extensibility.md` | Preserved |
| `conversation.md` | `conversation.md` | Preserved |
| `agents.md` | `agents.md` | Preserved |
| `memory.md` | `memory.md` | Preserved |
| `documents.md` | `documents.md` | Preserved |
| `retrieval.md` | `retrieval.md` | Preserved |
| `guardrails.md` | `guardrails.md` | Preserved |
| `transport.md` | `transport.md` | Preserved |
| `server.md` | `server.md` | Preserved |
| `tui.md` | `tui.md` | Preserved |
| `observability.md` | `observability.md` | Preserved |
| `monitoring.md` | `monitoring.md` | Preserved |
| `infrastructure.md` | `infrastructure.md` | Preserved |
| `durable-execution.md` | `durable-execution.md` | Preserved |
| `ai-operations.md` | `ai-operations.md` | Preserved |
| `security-compliance.md` | `security-compliance.md` | Preserved |
| `frontend-sdk.md` | `frontend-sdk.md` | Preserved |
| `demos.md` | `demos.md` | Preserved |
| `documentation.md` | `documentation.md` | Preserved |
| `release-pipeline.md` | `release-pipeline.md` | Preserved |
| `testing.md` | `testing.md` | Preserved |
| `execution.md` | `execution.md` | Preserved (additive workflow/source-of-truth sections) |

---

## Reader-First Layer

Use this sequence to start implementation quickly:

1. [Start Here](./start-here.md)
2. [Constraints](./constraints.md)
3. [Execution Plan](./execution.md)
4. [Implementation Workflow](./implementation-workflow.md)
5. [Task Verification Index](./task-verification-index.md)
6. [Domain Playbooks](./domain-playbooks.md)
7. [Batch Delivery Playbook](./batch-delivery-playbook.md)
8. Domain task specification document
9. [Handoff Packet Template](./handoff-packet-template.md)
10. [Quality Gates](./quality-gates.md)
11. [Readiness Scoreboard](./readiness-scoreboard.md)
12. [Readiness Decision Record Template](./readiness-decision-record-template.md)
13. [Eval and Adversarial Matrix](./eval-adversarial-matrix.md)
14. [Operations](./operations.md)
15. [Plan Drift Control](./plan-drift-control.md)
16. [Compact Authoring Standard](./compact-authoring-standard.md)

---

## Verification Procedure

- File parity check: all source filenames in `plan_v0` exist in `plan`.
- Missing check: no source file is absent from `plan`.
- Additions check: reader-first files are additive and do not replace source files.
- Content loss check: no removed lines from source files unless explicitly documented.
- Scheduling authority check: task ordering and dependency truth remain centralized in [Execution Plan](./execution.md).

---

## Documented Source Deltas

- `execution.md`: count wording and gate-layer wording were corrected for consistency with the current 129-task model and task-level versus project-level gate separation.
- `ai-operations.md`: top metadata label was clarified from task-style wording to internal-layer wording to avoid execution-task ambiguity.
- 24 module docs: repeated Task/Test relationship boilerplate was shortened to a compact equivalent sentence.
- 30 source docs: per-file `Table of Contents` blocks were removed under [Compact Authoring Standard](./compact-authoring-standard.md) to reduce token overhead without changing normative schemas.
