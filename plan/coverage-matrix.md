# Coverage Matrix

This matrix tracks preservation from the historical `plan_v0` baseline into the current `plan` workspace so no original planning content is lost while the new reader-first layer is added.

## Preservation Status

- Source files in `plan_v0`: 30
- Preserved source files in `plan`: 30
- Missing files: 0
- Additional reader-first files in `plan`: `start-here.md`, `constraints.md`, `quality-gates.md`, `operations.md`, `coverage-matrix.md`, `domain-playbooks.md`, `implementation-workflow.md`, `task-verification-index.md`, `batch-delivery-playbook.md`, `handoff-packet-template.md`, `plan-drift-control.md`, `readiness-scoreboard.md`, `readiness-decision-record-template.md`, `eval-adversarial-matrix.md`, `compact-authoring-standard.md`, `agent-kernel.md`

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

## Reader-First Layer

Use this sequence to start implementation quickly:

1. [Agent Kernel](./agent-kernel.md)
2. [Start Here](./start-here.md)
3. [Constraints](./constraints.md)
4. [Execution Plan](./execution.md)
5. [Implementation Workflow](./implementation-workflow.md)
6. [Task Verification Index](./task-verification-index.md)
7. [Domain Playbooks](./domain-playbooks.md)
8. [Batch Delivery Playbook](./batch-delivery-playbook.md)
9. Domain task specification document
10. [Handoff Packet Template](./handoff-packet-template.md)
11. [Quality Gates](./quality-gates.md)
12. [Readiness Scoreboard](./readiness-scoreboard.md)
13. [Readiness Decision Record Template](./readiness-decision-record-template.md)
14. [Eval and Adversarial Matrix](./eval-adversarial-matrix.md)
15. [Operations](./operations.md)
16. [Plan Drift Control](./plan-drift-control.md)
17. [Compact Authoring Standard](./compact-authoring-standard.md)

## Verification Procedure

- File parity check: all source filenames from the historical `plan_v0` baseline exist in `plan`.
- Missing check: no source file is absent from `plan`.
- Additions check: reader-first files are additive and do not replace source files.
- Content loss check: no removed lines from source files unless explicitly documented.
- Scheduling authority check: task ordering and dependency truth remain centralized in [Execution Plan](./execution.md).

## Documented Source Deltas

- `execution.md`: count wording and gate-layer wording were corrected for consistency with the current 129-task model and task-level versus project-level gate separation.
- `ai-operations.md`: top metadata label was clarified from task-style wording to internal-layer wording to avoid execution-task ambiguity.
- 24 module docs: repeated Task/Test relationship boilerplate was shortened to a compact equivalent sentence.
- 30 source docs: per-file `Table of Contents` blocks were removed under [Compact Authoring Standard](./compact-authoring-standard.md) to reduce token overhead without changing normative schemas.
- 4 source docs: Mermaid `style ... fill:` directives were removed as non-semantic visual metadata.
- 46 docs in `plan`: markdown separator lines and redundant blank-line runs were removed for compactness without changing schemas, IDs, dependencies, or gate order.
- 21 docs in `plan`: `Cross-References` sections were removed as redundant navigation blocks after centralizing navigation in agent-kernel/start-here/domain-playbooks.
