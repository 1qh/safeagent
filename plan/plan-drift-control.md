# Plan Drift Control

This document defines ongoing checks that keep the plan coherent as iterations continue.

---

## Drift Risks to Control

- Task count drift between execution routing and verification ownership.
- Batch label drift across task specifications and execution routing.
- Dependency drift that introduces later-batch prerequisites.
- Navigation drift caused by broken or stale cross-links.
- Authority drift where scheduling truth appears outside [Execution Plan](./execution.md).

---

## Drift Check Cadence

- Per change set: run local consistency checks before commit.
- Per iteration wave: run full plan consistency and link integrity review.
- Before release readiness: run final consistency review across reader-first docs and execution files.

---

## Mandatory Drift Checks

### Task and Coverage Consistency

- Task count in routing index equals expected execution total.
- Task count in [Task Verification Index](./task-verification-index.md) equals routing index count.
- No duplicate task IDs in routing or verification ownership index.

### Batch and Dependency Consistency

- Batch labels in task specs align with execution routing entries.
- Dependency matrix does not reference later-batch prerequisites.
- No unknown dependency identifiers in dependency matrix tables.

### Navigation and Ownership Consistency

- Reader-first layer links resolve to existing files.
- [Start Here](./start-here.md) and [Overview](./overview.md) remain consistent with current onboarding flow.
- Scheduling/dependency order statements remain centralized in [Execution Plan](./execution.md).

### Preservation and Scope Consistency

- Source preservation map in [Coverage Matrix](./coverage-matrix.md) remains accurate.
- Additive reader-first files are listed accurately in coverage metadata.
- No documented core scope is removed without explicit migration notes.

---

## Drift Response Protocol

1. Log the mismatch with exact file and section references.
2. Determine authoritative source (execution for scheduling, domain spec for behavior, constraints for boundaries).
3. Apply minimal corrective edits to restore consistency.
4. Re-run all drift checks impacted by the correction.
5. Record resolution in commit message scope.

---

## Related Documents

- [Execution Plan](./execution.md)
- [Task Verification Index](./task-verification-index.md)
- [Coverage Matrix](./coverage-matrix.md)
- [Quality Gates](./quality-gates.md)
