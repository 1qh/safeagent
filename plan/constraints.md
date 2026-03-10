# Constraints

This document consolidates non-negotiable constraints so builders can validate decisions quickly before implementation.

---

## Product and Scope Boundaries

- The core product is one library package named `safeagent`.
- A separate thin server project consumes the library.
- Core logic belongs in the library; server behavior is configuration and transport wiring.
- Scale target is 10 million users.

---

## Runtime and Dependency Policy

- Bun-only runtime and tooling.
- TypeScript-first and strongly typed contracts.
- Dependency policy is latest and unpinned.
- No raw SQL in Postgres paths; typed ORM paths are required.
- No raw Surreal query strings in SurrealDB paths; typed query builder paths are required.

---

## Safety, Reliability, and Quality Policy

- Safety and security are first-class concerns, not post-processing steps.
- Every feature path requires corresponding comprehensive tests.
- Development speed may be traded for breakage prevention and correctness.
- Verification gates are mandatory before release and before audit completion.

---

## Extensibility Policy

- Bring strong defaults out of the box.
- Keep all major subsystems modular and customizable.
- Avoid lock-in by keeping extension points explicit and typed.

---

## Plan Authoring Policy

- The plan should remain implementation-stable and avoid volatile scaffolding details.
- Focus on logic, requirements, constraints, findings, and verification behavior.
- Keep cross-document references explicit so builders do not search blindly.
- Keep task IDs semantic and consistent across execution, domain specs, and testing.
- Apply compact-first writing policy from [Compact Authoring Standard](./compact-authoring-standard.md).

---

## Detailed Sources

- Full boundary and requirement definitions: [Requirements & Constraints](./requirements.md)
- Security and governance requirements: [Unified Security Strategy and Regulatory Compliance](./security-compliance.md)
- Testing doctrine and audit scope: [Testing](./testing.md)
- Delivery sequencing and dependency truth: [Execution Plan](./execution.md)
