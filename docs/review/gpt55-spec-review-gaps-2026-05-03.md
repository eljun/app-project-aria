# GPT-5.5 Review: Spec Gaps & Recommendations (Planning Phase)

**Date:** 2026-05-03  
**Reviewer:** GPT-5.5 (Codex)  
**Scope reviewed:** CDD-01, CDD-02, CDD-03 (+ repo README positioning)

## Executive summary

The current architecture is strong on conceptual rigor and governance. The biggest remaining risks are **operational ambiguity** and **verifiability gaps** rather than missing concepts. In short: many invariants are defined, but some are not yet specified in machine-checkable acceptance terms.

Top priorities before build acceleration:
1. Add a **global consistency contract** across CDDs (single source of truth for thresholds, enums, and transition semantics).
2. Define **failure handling semantics** for partial writes, retries, and cross-component idempotency.
3. Add a **testability matrix** mapping each invariant and transition to concrete automated tests.
4. Lock down **observability and SLOs** (latency/error budgets, drift detection precision targets).
5. Add explicit **security/threat model** for ontology poisoning, replay, and data provenance tampering.

---

## High-value gaps identified

## 1) Cross-CDD contract drift risk

### Gap
CDD-01/02/03 are tightly coupled but there is no explicit canonical "contract table" with normative keys and owners (e.g., threshold names, enum allowed values, context diversity formula versioning).

### Why it matters
Even minor naming or formula drift across modules will create silent logic divergence (especially BSM transitions and tier protection pathways).

### Recommendation
Create `docs/specs/canonical-contract-matrix-v1.md` that includes:
- Field/parameter name
- Owning CDD
- Data type + units
- Allowed range
- Backward compatibility policy
- Change control process

---

## 2) Invariant enforcement granularity is under-specified

### Gap
Invariant behavior is described at design level, but boundary cases are not fully normed:
- When to hard-fail vs quarantine vs soft-flag
- Conflict resolution precedence if multiple invariant checks trigger
- Recovery semantics after `InvariantViolationError`

### Recommendation
Add a deterministic **Invariant Decision Table**:
- Input condition
- Severity
- Action
- Retry policy
- Audit event schema

---

## 3) Concurrency + idempotency semantics are incomplete

### Gap
Optimistic concurrency is referenced, but end-to-end race handling is not fully specified for multi-step flows (e.g., transitions + audit + version diff + derived updates).

### Recommendation
Add a "Write Semantics" section in CDD-01 with:
- idempotency keys per command
- safe retry behavior
- exactly-once vs at-least-once guarantees
- conflict resolution policy for stale `expected_version`

---

## 4) Missing explicit schema evolution strategy

### Gap
Versioning is present (node version history), but **schema migration lifecycle** is not fully defined.

### Recommendation
Define:
- migration classes (backward-compatible, breaking)
- mandatory migration tests
- downgrade policy
- data backfill strategy for required new fields

---

## 5) Epistemic metrics need calibration protocol

### Gap
Confidence/state thresholds exist, but no concrete calibration loop is defined for verifying that confidence values correspond to empirical correctness.

### Recommendation
Add a calibration spec:
- reliability diagrams
- expected calibration error (ECE) targets
- per-domain recalibration cadence
- stop-ship criteria if calibration degrades

---

## 6) Tier-1 hard-review governance is process-light

### Gap
Tier-1 override exists, but approval policy appears procedural rather than cryptographically/operationally enforceable.

### Recommendation
Specify a **policy-as-code approval gate**:
- minimum approvers
- signer identities / role constraints
- immutable approval logs
- emergency override policy and postmortem requirements

---

## 7) Security and adversarial model is implicit, not formalized

### Gap
There are references to adversarial testing, but no consolidated threat model for:
- ontology poisoning
- prompt/log injection into evidence pipeline
- replayed observation events
- provenance forgery

### Recommendation
Create `threat-model-v1.md` with STRIDE-style table and mitigation ownership.

---

## 8) Observability/SRE criteria are not yet production-grade

### Gap
Auditability is emphasized, but service-level goals are missing.

### Recommendation
Define SLOs + telemetry dictionary:
- p50/p95/p99 latency for read/write/transition
- transition failure rates
- invariant violation alert thresholds
- drift monitor precision/recall targets
- on-call runbook hooks

---

## 9) Test strategy does not yet map 1:1 to claims

### Gap
Strong claims (immutability, lawful transitions, no backdoors) need direct traceability to automated checks.

### Recommendation
Add a **Requirements-to-Tests Traceability Matrix**:
- requirement ID
- unit/integration/property/fuzz test IDs
- pass criteria
- ownership

---

## 10) Data provenance chain needs stricter formalism

### Gap
Evidence history is conceptually present, but there is no explicit minimal provenance envelope requirement.

### Recommendation
Require provenance fields for any confidence/state mutation:
- evidence source ID
- acquisition timestamp
- transformation lineage hash
- verifier identity
- confidence contribution weight

---

## Suggested next artifacts (in order)

1. Canonical Contract Matrix v1  
2. Invariant Decision Table v1  
3. Write Semantics & Idempotency Profile v1  
4. Calibration & Reliability Spec v1  
5. Threat Model v1  
6. Requirements-to-Tests Traceability Matrix v1

---

## Positive assessment

What is already excellent:
- Clear separation of concerns between schema, ontology, and epistemic transitions.
- Strong emphasis on immutable axioms + tiered mutability.
- Good early attention to auditability and version history.
- Architecture is unusually reviewable and intellectually coherent for planning stage.

With the gaps above closed, the design can move from **conceptual robustness** to **implementation robustness** with lower integration risk.
