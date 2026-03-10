# Compact Authoring Standard

Purpose: make plan docs token-efficient for agents and still readable for humans.

## Authority

- Scheduling/dependency/task routing authority: [Execution Plan](./execution.md)
- Gate order and evidence authority: [Quality Gates](./quality-gates.md)
- Verification owner mapping authority: [Task Verification Index](./task-verification-index.md)
- This file defines writing style only. It does not define scheduling or gate logic.

## Compression Policy

Decision: `HYBRID_COMPRESSION`

- Keep all normative schemas explicit.
- Compress explanatory prose aggressively.
- Replace repeated narrative with pointers to authoritative tables.

## MUST Keep Explicit

- Task IDs, batch IDs, dependency rows, routing rows.
- Ordered gate sequences and required evidence artifacts.
- Acceptance Criteria and QA Scenarios in task specs.
- Authority statements (`authoritative`, `derived-only`, source hierarchy).

## Compress Aggressively

- Intro/context paragraphs that do not add unique semantics.
- Repeated "how to use" prose where table/list already exists.
- Repeated explanation of the same table row.
- Non-authoritative rollup narration.
- Redundant cross-reference prose.

## Writing Rules (Compact-First)

- One rule per line.
- Prefer tables/lists over prose blocks.
- Prefer fixed labels: `Task`, `Batch`, `Deps`, `Spec`, `Test`, `Gate`, `Evd`, `Owner`, `Status`.
- Use short imperative bullets; remove filler words.
- Keep paragraph blocks short; split long prose into bullets.
- If sentence repeats a nearby table/list meaning, delete sentence.
- If content is derived-only, state source and stop.

## Forbidden Patterns

- Long narrative restating authoritative table content.
- Multiple sections that define the same authority.
- Ambiguous synonyms for core fields (`deps`, `dependencies`, `prereqs`) in same doc.
- New standalone scheduling logic outside [Execution Plan](./execution.md).

## Compliance Check (Per Edit)

- Does this edit add new normative meaning? If yes, place it in authoritative schema doc.
- Does any prose duplicate nearby table/list semantics? If yes, remove prose.
- Are authority statements still unambiguous? If no, reconcile.
- Are links valid and drift checks still pass? If no, fix before commit.

## Related Documents

- [Plan Drift Control](./plan-drift-control.md)
- [Start Here](./start-here.md)
- [Constraints](./constraints.md)
