# CDD-02: Seed Ontology & Tier Protection

> **Status:** APPROVED — All Reviewing Systems Signed Off  
> **Version:** 1.0 (Incorporates Multi-LLM Review Feedback)  
> **Author:** Jun (Architect) + Claude Opus 4.6 (Design Partner)  
> **Depends On:** CDD-01 v1.1 (World Model Schema & GraphStore Protocol)  
> **Sprint:** S2 (Seed Ontology + Formal Semantics)  
> **Classification:** Confidential — Core Team & Designated Review Partners  
> **Review Contributors:** GPT-5 (OpenAI) · Gemini (Google) · Grok (xAI) · Claude Opus 4.6

---

## Review & Approval Record

| System | Status | Conditions | Resolution |
|--------|--------|------------|------------|
| GPT-5 (OpenAI) | ✅ APPROVED | 2 additions required (Tier 1 revalidation, relationship tier check) | Both incorporated into v1.0 |
| Gemini (Google) | ✅ APPROVED | No conditions | Clean sign-off |
| Grok (xAI) | ✅ APPROVED | No conditions | Clean sign-off |
| Claude Opus 4.6 | ✅ 4th REVIEWER | 3 observations raised | All 3 confirmed and incorporated |

**Consensus:** 4/4 approved. No contentions remain. All changes from v0.1 marked with ★.

---

## 1. Purpose & Scope

CDD-02 defines what ARIA *knows before it learns anything.* If CDD-01 is the empty graph structure, CDD-02 is the content that populates it — the foundational beliefs that shape how ARIA interprets every future observation.

This CDD specifies four things:

**1.1 Seed Ontology Content** — The actual nodes and relationships that define ARIA's starting knowledge, organized into three tiers. These are the beliefs ARIA is "born with," analogous to how a human infant has innate expectations about object permanence and causality before any learning occurs.

**1.2 YAML Schema Format** — The standardized format for defining ontology nodes and relationships in human-readable YAML files. All seed knowledge lives in version-controlled YAML, not hardcoded in Python.

**1.3 Ontology Loader** — The pipeline that reads YAML files, validates them through Pydantic (★ including cross-tier reference pre-validation), and loads them into Neo4j through the GraphStore Protocol. The loader enforces tier-specific rules during loading and produces an audit trail of every node created.

**1.4 Tier Protection Rules** — The enforcement mechanisms that make Tier 0 truly immutable, Tier 1 revisable-only-via-hard-review, and Tier 2 continuously updateable. ★ These rules now extend to relationship mutations (CDD-01 v1.1 patch) and include a Tier 1 revalidation clause.

### 1.1 What This CDD Does NOT Cover

| Excluded | Covered In |
|----------|------------|
| GraphStore Protocol and WorldNode schema | CDD-01 |
| Belief State Machine transitions and preconditions | CDD-03 |
| Formal semantics (confidence thresholds, calibration metrics, ★ revalidation cycle count N) | CDD-06 |
| Adversarial de-biasing validation for Tier 2 | CDD-07 |
| Causal Complexity Manager (abstraction, budgets) | CDD-08 |
| ★ Tier Health Metrics (full implementation) | CDD-10 (DriftMonitor) |

### 1.2 Neuroscience Grounding

The 3-tier structure maps to established cognitive science:

| Tier | Cognitive Analogue | Reference |
|------|--------------------|-----------|
| Tier 0 — Logical Absolutes | Core knowledge / innate constraints (Spelke, 2007) | Infants expect physical objects to obey spatiotemporal continuity from birth |
| Tier 1 — Physical Bedrock | Naïve physics (Baillargeon, 1987) | Object permanence, gravity expectations emerge in first months |
| Tier 2 — Structural Priors | Statistical learning biases (Saffran et al., 1996) | Pattern extraction capabilities that bootstrap from exposure |

---

## 2. Specification References

| Spec Section | Content | Relevance |
|-------------|---------|-----------|
| v2.1 §3.1 | Seed Ontology — 3-Tier Hybrid (RFC-01, Unanimous 4:0) | Tier definitions, content, mutability rules |
| v2.1 §3.4 | Constitutional Immutability + Evolving Interpretation | Tier 0 lives alongside Constitutional Value Layer |
| v2.1 §4.1 | Node Schema Properties | source enum values for each tier |
| v2.1 §7.1 | Open Source vs Sovereign | Tier 0 protection is sovereign |
| Impl Plan §6 Sprint 2 | Seed Ontology + Formal Semantics deliverables | Acceptance criteria |
| CDD-01 §3.1 | GraphStore Protocol | create_node, ImmutableNodeError, InvariantViolationError |
| CDD-01 §3.4 | Error Hierarchy | ImmutableNodeError, ★ CrossTierReferenceError (v1.1) |
| CDD-01 §4.1 | NodeSource enum | SEED_ONTOLOGY_T0, SEED_ONTOLOGY_T1, SEED_ONTOLOGY_T2 |
| ★ CDD-01 v1.1 | Relationship Tier Protection patch | Tier checks on delete/update relationship |

---

## 3. Tier Architecture

### 3.1 Tier Summary

| Property | Tier 0 — Logical Absolutes | Tier 1 — Physical Bedrock | Tier 2 — Structural Priors |
|----------|---------------------------|--------------------------|---------------------------|
| **Source** | Hand-coded only | Hand-coded, revisable via hard review | Meta-learned from curated datasets |
| **Mutability** | Immutable. Cannot be modified or deleted. | Revisable only through ★ CLI hard review workflow | Continuously updated. Subject to adversarial de-biasing |
| **Nature** | Mathematically necessary truths | Operational assumptions about physical reality | Statistical regularities, not truths |
| **Confidence** | 1.0 (absolute) | 0.95 (very high, but acknowledges edge cases) | ★ 0.65–0.75 (non-uniform — see §6.1) |
| **Belief State** | CONFIRMED (pre-validated) | CONFIRMED (pre-validated) | TENTATIVE (awaiting reinforcement) |
| **Decay Rate** | 0.0 (never decays) | 0.0 (never decays without review) | ★ 0.005–0.02 (non-uniform — see §6.1) |
| **identity_invariant** | true | false | false |
| **NodeSource** | `seed-ontology-t0` | `seed-ontology-t1` | `seed-ontology-t2` |
| **Protection Level** | `ImmutableNodeError` on any modification ★ including relationships | Modification requires ★ CLI hard review | Standard update rules apply |
| **Falsifiability** | Not applicable — these are axioms | Must have criteria (even if extreme) | Required ★ + optional machine_testable_criteria |
| **★ Formal Expression** | Required (mathematical logic) | ★ Required (physics equations) | Not required |
| **★ Revalidation** | Never (axioms) | ★ Flagged if no supporting evidence in N cycles | Natural decay handles revalidation |
| **Phase 1 Count** | ★ ~9 nodes, ~10 relationships | ★ ~12 nodes, ~17 relationships | ~6 nodes, ~3 relationships |

### 3.2 Design Principle: Why Three Tiers?

A single tier would force a false choice: either everything is immutable (too rigid — can't correct mistaken physics assumptions) or everything is revisable (too fragile — core logic could be undermined by adversarial experience).

Three tiers create a gradient from absolute certainty to informed guesses, matching how human cognition operates: you can question whether gravity works the same way on another planet (Tier 1), but you cannot question whether A and not-A can both be true simultaneously (Tier 0).

---

## 4. Seed Ontology Content — Tier 0: Logical Absolutes

These are domain-independent logical truths that hold in any possible world. They cannot be learned, unlearned, or contradicted by observation. If an observation appears to violate a Tier 0 node, the observation is suspect — not the axiom.

### 4.1 Tier 0 Nodes

★ Changes from v0.1: Added t0-reflexivity (Grok recommendation).

```yaml
# seed_data/tier0_logical_absolutes.yml

metadata:
  tier: 0
  source: "seed-ontology-t0"
  description: "Logical absolutes — immutable, domain-independent"
  version: "1.0"
  author: "ARIA Core Architecture Team"

nodes:
  # ── Non-Contradiction ────────────────────────────────────────────
  - node_id: "t0-non-contradiction"
    node_type: "constraint"
    label: "Principle of Non-Contradiction"
    description: >
      A proposition and its negation cannot both be true in the same
      context at the same time. If the World Model contains nodes A
      and NOT-A both marked CONFIRMED in the same domain_scope, the
      system is in an invalid state.
    formal_expression: "∀P, ¬(P ∧ ¬P)"
    confidence: 1.0
    belief_state: "confirmed"
    epistemic_uncertainty: 0.0
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    identity_invariant: true
    falsifiability_criteria: []
    domain_scope: []
    enforcement: >
      Invariant Checker (CDD-03) monitors all CONTRADICTS relationships.
      If both endpoints are CONFIRMED in overlapping domain_scope,
      InvariantViolationError is raised as a hard error.

  # ── Temporal Ordering ────────────────────────────────────────────
  - node_id: "t0-temporal-ordering"
    node_type: "constraint"
    label: "Principle of Temporal Ordering"
    description: >
      Cause must precede or be simultaneous with effect. No backwards
      causation. The PRECEDES relationship defines a strict partial
      order that cannot contain cycles.
    formal_expression: "∀(A CAUSES B) → time(A) ≤ time(B)"
    confidence: 1.0
    belief_state: "confirmed"
    epistemic_uncertainty: 0.0
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    identity_invariant: true
    falsifiability_criteria: []
    domain_scope: []
    enforcement: >
      Invariant Checker verifies that PRECEDES relationships form a
      DAG (no cycles). CAUSES relationships must respect temporal order.
      Cycle detection runs on every new PRECEDES/CAUSES edge creation.

  # ── Causality Direction ──────────────────────────────────────────
  - node_id: "t0-causality-direction"
    node_type: "constraint"
    label: "Principle of Causal Directionality"
    description: >
      Causal relationships are directional. A CAUSES B does not imply
      B CAUSES A. Correlation (CORRELATES_WITH) is explicitly
      non-directional and must not be treated as causation.
    formal_expression: "(A CAUSES B) ↛ (B CAUSES A)"
    confidence: 1.0
    belief_state: "confirmed"
    epistemic_uncertainty: 0.0
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    identity_invariant: true
    falsifiability_criteria: []
    domain_scope: []
    enforcement: >
      CORRELATES_WITH relationships carry an investigation flag. The
      system must never auto-promote CORRELATES_WITH to CAUSES without
      confounder analysis. The Adversary Simulator (CDD-07) generates
      confounder alternatives for every correlation.

  # ── Identity Persistence ─────────────────────────────────────────
  - node_id: "t0-identity-persistence"
    node_type: "constraint"
    label: "Principle of Identity Persistence"
    description: >
      An entity retains its identity across state changes unless
      explicitly destroyed or transformed. Entity A at time T1 and
      Entity A at time T2 are the same entity even if properties
      change. node_id is stable and never reused.
    formal_expression: "∀E, identity(E, t1) = identity(E, t2) unless destroyed(E, t) where t1 < t < t2"
    confidence: 1.0
    belief_state: "confirmed"
    epistemic_uncertainty: 0.0
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    identity_invariant: true
    falsifiability_criteria: []
    domain_scope: []
    enforcement: >
      node_id is a UUID that is never reused, even after soft-delete.
      State nodes link to their parent Entity via relationships, not
      by copying the Entity's identity.

  # ── Conservation (Abstract Form) ─────────────────────────────────
  - node_id: "t0-conservation-abstract"
    node_type: "constraint"
    label: "Principle of Conservation (Abstract)"
    description: >
      In a closed system, measurable quantities are conserved across
      transformations. The specific conserved quantity depends on
      domain (energy in physics, mass in chemistry, value in
      economics). This is the abstract form — domain-specific
      instantiations live in Tier 1.
    formal_expression: "∀ closed_system S, ∃ quantity Q: Q(S, t1) = Q(S, t2)"
    confidence: 1.0
    belief_state: "confirmed"
    epistemic_uncertainty: 0.0
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    identity_invariant: true
    falsifiability_criteria: []
    domain_scope: []
    enforcement: >
      Domain-specific conservation checks are Tier 1 nodes. This Tier 0
      node establishes that conservation as a principle exists. Tier 1
      nodes instantiate it for specific domains.

  # ── Excluded Middle ──────────────────────────────────────────────
  - node_id: "t0-excluded-middle"
    node_type: "constraint"
    label: "Principle of Excluded Middle"
    description: >
      For any proposition, either it or its negation is true. There is
      no third option. This applies to the logical evaluation of
      propositions, not to the system's epistemic state (the system
      can be uncertain about which is true).
    formal_expression: "∀P, P ∨ ¬P"
    confidence: 1.0
    belief_state: "confirmed"
    epistemic_uncertainty: 0.0
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    identity_invariant: true
    falsifiability_criteria: []
    domain_scope: []
    enforcement: >
      Epistemic uncertainty is tracked separately from logical truth.
      A node can have high epistemic_uncertainty (we don't know which
      is true) while the excluded middle still holds (one must be true).

  # ── Transitivity of Causation ────────────────────────────────────
  - node_id: "t0-causal-transitivity"
    node_type: "constraint"
    label: "Transitivity of Deterministic Causation"
    description: >
      If A deterministically CAUSES B and B deterministically CAUSES C,
      then A deterministically CAUSES C (possibly through a MEDIATOR).
      This does NOT apply to PROBABILISTICALLY_CAUSES, CORRELATES_WITH,
      or ENABLES — transitivity is restricted to deterministic causation.
    formal_expression: "(A CAUSES B) ∧ (B CAUSES C) → (A CAUSES C)"
    confidence: 1.0
    belief_state: "confirmed"
    epistemic_uncertainty: 0.0
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    identity_invariant: true
    falsifiability_criteria: []
    domain_scope: []
    enforcement: >
      Causal chain compression in the Causal Complexity Manager (CDD-08)
      exploits this property. Transitive closure is computed only for
      CAUSES relationships, not for probabilistic types.

  # ── Asymmetry of Causation ───────────────────────────────────────
  - node_id: "t0-causal-asymmetry"
    node_type: "constraint"
    label: "Asymmetry of Causation"
    description: >
      If A CAUSES B, then B does not CAUSE A (in the same causal
      context). Causation is irreflexive and asymmetric. Feedback
      loops are modeled as temporal sequences (A at t1 CAUSES B at t2,
      B at t2 CAUSES A at t3), not as simultaneous bidirectional causation.
    formal_expression: "(A CAUSES B) → ¬(B CAUSES A) in same temporal context"
    confidence: 1.0
    belief_state: "confirmed"
    epistemic_uncertainty: 0.0
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    identity_invariant: true
    falsifiability_criteria: []
    domain_scope: []
    enforcement: >
      Invariant Checker prevents creation of A CAUSES B and B CAUSES A
      in the same temporal context. Feedback loops require temporal
      separation (modeled through State nodes at different timestamps).

  # ★ ── Reflexivity of Identity ────────────────────────────────────
  - node_id: "t0-reflexivity"                                         # ★ NEW (Grok)
    node_type: "constraint"
    label: "Reflexivity of Identity"
    description: >
      Every entity is identical to itself. A = A. This is the most
      basic identity axiom. Combined with identity_persistence, it
      ensures that self-referential checks are always valid and that
      the system never treats an entity as non-identical to itself.
    formal_expression: "∀A, A = A"
    confidence: 1.0
    belief_state: "confirmed"
    epistemic_uncertainty: 0.0
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    identity_invariant: true
    falsifiability_criteria: []
    domain_scope: []
    enforcement: >
      Strengthens identity persistence checks. If any operation
      produces a result where a node is evaluated as non-identical
      to itself (e.g., hash mismatch on unchanged node), this is
      a system integrity error, not a logical possibility.
```

### 4.2 Tier 0 Relationships

```yaml
relationships:
  - rel_type: "REQUIRES"
    source: "t0-causal-transitivity"
    target: "t0-causality-direction"
    strength: 1.0
    confidence: 1.0
    context_tags: ["foundational-logic"]
    rationale: "Transitivity presupposes directionality."

  - rel_type: "REQUIRES"
    source: "t0-causal-asymmetry"
    target: "t0-temporal-ordering"
    strength: 1.0
    confidence: 1.0
    context_tags: ["foundational-logic"]
    rationale: "Asymmetry is enforced through temporal ordering."

  - rel_type: "REQUIRES"
    source: "t0-causal-asymmetry"
    target: "t0-causality-direction"
    strength: 1.0
    confidence: 1.0
    context_tags: ["foundational-logic"]
    rationale: "Asymmetry presupposes directionality."

  - rel_type: "IS_PART_OF"
    source: "t0-excluded-middle"
    target: "t0-non-contradiction"
    strength: 1.0
    confidence: 1.0
    context_tags: ["classical-logic"]
    rationale: "Excluded middle and non-contradiction form the classical logic foundation."

  # ★ NEW — reflexivity supports identity persistence
  - rel_type: "REQUIRES"
    source: "t0-identity-persistence"
    target: "t0-reflexivity"
    strength: 1.0
    confidence: 1.0
    context_tags: ["foundational-logic"]
    rationale: "Identity persistence presupposes that A = A holds at all times."
```

---

## 5. Seed Ontology Content — Tier 1: Physical Bedrock

These are strong operational assumptions about physical reality. They are expected to hold in ARIA's Phase 1 physics domain but are acknowledged as potentially revisable. An infant "expects" a ball to fall — but this expectation can be revised if evidence overwhelmingly contradicts it.

★ Changes from v0.1: Added t1-elasticity (Gemini + Grok). Added formal_expression to all nodes (Gemini + Grok).

### 5.1 Tier 1 Nodes

```yaml
# seed_data/tier1_physical_bedrock.yml

metadata:
  tier: 1
  source: "seed-ontology-t1"
  description: "Physical bedrock — operational assumptions, revisable via hard review"
  version: "1.0"
  author: "ARIA Core Architecture Team"
  review_authority: "Multi-signature: Architect + 2 reviewing systems"

nodes:
  # ── Object Permanence ────────────────────────────────────────────
  - node_id: "t1-object-permanence"
    node_type: "constraint"
    label: "Object Permanence"
    description: >
      Physical objects continue to exist when not being observed.
      An entity that was observed at time T and is not observed at T+1
      still exists unless a destruction event is recorded.
    formal_expression: "∀E, observed(E, t) ∧ ¬destroyed(E, t') → exists(E, t+1) for t < t' or no t'"
    confidence: 0.95
    belief_state: "confirmed"
    epistemic_uncertainty: 0.05
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    falsifiability_criteria:
      - "An object consistently fails to reappear after occlusion in controlled conditions"
      - "Observations from multiple independent sensors confirm disappearance without destruction event"
    domain_scope: ["physics"]
    causal_depth: 0

  # ── Gravitational Attraction ─────────────────────────────────────
  - node_id: "t1-gravity"
    node_type: "constraint"
    label: "Gravitational Attraction"
    description: >
      Unsupported objects accelerate toward the ground. Objects with
      mass attract each other. In Phase 1 physics scenarios, this
      manifests as constant downward acceleration near a surface.
    formal_expression: "F = G·m₁·m₂/r² (general); a = g ≈ 9.81 m/s² (Phase 1 surface)"
    confidence: 0.95
    belief_state: "confirmed"
    epistemic_uncertainty: 0.05
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    falsifiability_criteria:
      - "An unsupported object consistently fails to fall in repeated controlled experiments"
      - "Acceleration measurements consistently contradict predicted gravitational model"
    domain_scope: ["physics"]
    causal_depth: 0

  # ── Contact Interaction ──────────────────────────────────────────
  - node_id: "t1-contact-interaction"
    node_type: "constraint"
    label: "Contact Interaction Principle"
    description: >
      Physical objects interact through contact or known force fields.
      Action at a distance requires a mediating mechanism (field, wave,
      particle). If two objects change state simultaneously without
      contact or known mediator, a hidden variable (CONFOUNDER) is
      the most likely explanation.
    formal_expression: "∀(A affects B) → ∃ mediator M: contact(A,M) ∧ contact(M,B) ∨ field(A,B)"
    confidence: 0.95
    belief_state: "confirmed"
    epistemic_uncertainty: 0.05
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    falsifiability_criteria:
      - "Repeated demonstrations of correlated state changes without any detectable mediating mechanism"
    domain_scope: ["physics"]
    causal_depth: 0

  # ── Inertia ──────────────────────────────────────────────────────
  - node_id: "t1-inertia"
    node_type: "constraint"
    label: "Principle of Inertia"
    description: >
      An object at rest stays at rest, and an object in motion stays
      in motion at constant velocity, unless acted upon by a net
      external force. Unexpected velocity changes imply unobserved forces.
    formal_expression: "ΣF = 0 → Δv = 0 (Newton's First Law)"
    confidence: 0.95
    belief_state: "confirmed"
    epistemic_uncertainty: 0.05
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    falsifiability_criteria:
      - "An isolated object changes velocity without any detectable force across multiple experiments"
    domain_scope: ["physics"]
    causal_depth: 0

  # ── Energy Conservation ──────────────────────────────────────────
  - node_id: "t1-energy-conservation"
    node_type: "constraint"
    label: "Conservation of Energy"
    description: >
      In a closed system, total energy is conserved. Energy can change
      form (kinetic → potential → thermal) but cannot be created or
      destroyed. This is the physics instantiation of the Tier 0
      abstract conservation principle.
    formal_expression: "ΣE_initial = ΣE_final (KE + PE + thermal + ... = constant)"
    confidence: 0.95
    belief_state: "confirmed"
    epistemic_uncertainty: 0.05
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    falsifiability_criteria:
      - "Energy accounting in a demonstrably closed system consistently fails to balance"
    domain_scope: ["physics"]
    causal_depth: 0

  # ── Momentum Conservation ────────────────────────────────────────
  - node_id: "t1-momentum-conservation"
    node_type: "constraint"
    label: "Conservation of Momentum"
    description: >
      In a closed system with no external forces, total momentum is
      conserved. Critical for collision scenario predictions in Phase 1.
    formal_expression: "Σp_initial = Σp_final (Σmv before = Σmv after)"
    confidence: 0.95
    belief_state: "confirmed"
    epistemic_uncertainty: 0.05
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    falsifiability_criteria:
      - "Momentum measurements before and after collision in closed system consistently fail to balance"
    domain_scope: ["physics"]
    causal_depth: 0

  # ── Spatial Continuity ───────────────────────────────────────────
  - node_id: "t1-spatial-continuity"
    node_type: "constraint"
    label: "Spatial Continuity"
    description: >
      An object moves through connected spatial positions. It cannot
      teleport — disappear from position A and appear at position B
      without traversing intermediate positions.
    formal_expression: "∀E, position(E) is a continuous function of time"
    confidence: 0.95
    belief_state: "confirmed"
    epistemic_uncertainty: 0.05
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    falsifiability_criteria:
      - "An object repeatedly appears at non-contiguous positions with no detectable traversal"
    domain_scope: ["physics"]
    causal_depth: 0

  # ── Deterministic Macro Physics ──────────────────────────────────
  - node_id: "t1-macro-determinism"
    node_type: "constraint"
    label: "Macroscopic Determinism"
    description: >
      At macroscopic scales, identical initial conditions produce
      identical outcomes (within measurement precision). Unexpected
      divergence implies unobserved variables, not fundamental randomness.
      Phase 1 physics operates entirely at macroscopic scales.
    formal_expression: "∀ macro_system S, state(S, t0) = state(S', t0) → state(S, t1) ≈ state(S', t1)"
    confidence: 0.90
    belief_state: "confirmed"
    epistemic_uncertainty: 0.10
    aleatoric_uncertainty: 0.0
    confidence_decay_rate: 0.0
    falsifiability_criteria:
      - "Repeated experiments with verified identical conditions produce statistically different outcomes"
    domain_scope: ["physics"]
    causal_depth: 0

  # ── Friction ─────────────────────────────────────────────────────
  - node_id: "t1-friction"
    node_type: "entity"
    label: "Friction"
    description: >
      Contact surfaces resist relative motion. Friction converts
      kinetic energy to thermal energy. It is a ubiquitous force
      that must be accounted for in all surface-contact scenarios.
    formal_expression: "F_friction = μ · F_normal"
    confidence: 0.95
    belief_state: "confirmed"
    epistemic_uncertainty: 0.05
    aleatoric_uncertainty: 0.05
    confidence_decay_rate: 0.0
    falsifiability_criteria:
      - "Objects in surface contact move without any energy dissipation in repeated experiments"
    domain_scope: ["physics"]
    causal_depth: 1

  # ── Air Resistance ───────────────────────────────────────────────
  - node_id: "t1-air-resistance"
    node_type: "entity"
    label: "Air Resistance"
    description: >
      Objects moving through air experience a resistive force
      proportional to velocity. A key hidden variable in the
      free-fall physics scenario.
    formal_expression: "F_drag = ½ · ρ · v² · C_d · A"
    confidence: 0.95
    belief_state: "confirmed"
    epistemic_uncertainty: 0.05
    aleatoric_uncertainty: 0.05
    confidence_decay_rate: 0.0
    falsifiability_criteria:
      - "Objects of different shapes fall at identical rates in atmosphere across repeated tests"
    domain_scope: ["physics"]
    causal_depth: 1

  # ★ ── Elasticity ─────────────────────────────────────────────────
  - node_id: "t1-elasticity"                                          # ★ NEW (Gemini + Grok)
    node_type: "entity"
    label: "Elasticity"
    description: >
      Objects deform under force and may return to original shape
      (elastic) or remain deformed (inelastic). In collisions,
      elasticity determines energy partition between kinetic retention
      and thermal dissipation. Coefficient of restitution ranges
      from 0 (perfectly inelastic) to 1 (perfectly elastic).
    formal_expression: "v₁' = ((m₁ - e·m₂)v₁ + (1+e)m₂·v₂) / (m₁+m₂) where e = coeff. of restitution"
    confidence: 0.95
    belief_state: "confirmed"
    epistemic_uncertainty: 0.05
    aleatoric_uncertainty: 0.05
    confidence_decay_rate: 0.0
    falsifiability_criteria:
      - "Repeated collisions show energy partition inconsistent with any restitution model"
    domain_scope: ["physics"]
    causal_depth: 1
```

### 5.2 Tier 1 Relationships

★ Changes: Added elasticity relationships.

```yaml
relationships:
  # Conservation instantiations
  - rel_type: "IS_PART_OF"
    source: "t1-energy-conservation"
    target: "t0-conservation-abstract"
    strength: 1.0
    confidence: 1.0
    context_tags: ["ontology-structure"]
    rationale: "Energy conservation is a physics instantiation of abstract conservation."

  - rel_type: "IS_PART_OF"
    source: "t1-momentum-conservation"
    target: "t0-conservation-abstract"
    strength: 1.0
    confidence: 1.0
    context_tags: ["ontology-structure"]
    rationale: "Momentum conservation is a physics instantiation of abstract conservation."

  # Physical dependencies
  - rel_type: "REQUIRES"
    source: "t1-spatial-continuity"
    target: "t1-object-permanence"
    strength: 1.0
    confidence: 0.95
    context_tags: ["physics-foundation"]
    rationale: "Spatial continuity presupposes that the object persists."

  - rel_type: "ENABLES"
    source: "t1-gravity"
    target: "t1-energy-conservation"
    strength: 0.8
    confidence: 0.90
    context_tags: ["physics-foundation"]
    rationale: "Gravitational potential energy is a form in the conservation equation."

  # Dissipation mechanisms
  - rel_type: "PREVENTS"
    source: "t1-friction"
    target: "t1-inertia"
    strength: 0.7
    confidence: 0.95
    context_tags: ["physics-forces"]
    rationale: "Friction opposes continued motion, partially preventing inertial behavior."

  - rel_type: "PREVENTS"
    source: "t1-air-resistance"
    target: "t1-inertia"
    strength: 0.5
    confidence: 0.90
    context_tags: ["physics-forces"]
    rationale: "Air resistance opposes motion through atmosphere."

  # Confounder preparedness
  - rel_type: "CORRELATES_WITH"
    source: "t1-friction"
    target: "t1-air-resistance"
    strength: 0.3
    confidence: 0.70
    context_tags: ["physics-forces"]
    rationale: >
      Both are dissipative forces that reduce kinetic energy. They
      often co-occur but have different mechanisms. This correlation
      is flagged for investigation — it should not be mistaken for causation.

  # ★ Elasticity relationships
  - rel_type: "ENABLES"
    source: "t1-elasticity"
    target: "t1-energy-conservation"
    strength: 0.8
    confidence: 0.90
    context_tags: ["physics-collisions"]
    rationale: "Elasticity determines how energy is partitioned in collisions."

  - rel_type: "ENABLES"
    source: "t1-elasticity"
    target: "t1-momentum-conservation"
    strength: 0.8
    confidence: 0.90
    context_tags: ["physics-collisions"]
    rationale: "Elasticity affects velocity exchange in collisions while momentum is conserved."
```

---

## 6. Seed Ontology Content — Tier 2: Structural Priors

Tier 2 nodes are statistical biases, not truths. They represent patterns ARIA expects to encounter based on the structure of causal systems in general.

★ Changes from v0.1: Non-uniform confidence values (0.65–0.75), non-uniform decay rates (0.005–0.02), added machine_testable_criteria field.

### 6.1 Tier 2 Nodes

```yaml
# seed_data/tier2_structural_priors.yml

metadata:
  tier: 2
  source: "seed-ontology-t2"
  description: "Structural priors — statistical biases, continuously updated"
  version: "1.0"
  author: "ARIA Core Architecture Team"
  update_policy: "Continuously revised based on observation. Subject to adversarial de-biasing."

nodes:
  # ── Causal Sparsity ─────────────────────────────────────────────
  - node_id: "t2-causal-sparsity"
    node_type: "constraint"
    label: "Causal Sparsity Prior"
    description: >
      Most effects have few direct causes. When predicting an outcome,
      a small number of causal variables typically explains most of the
      variance. This prior biases ARIA toward parsimonious causal models.
    confidence: 0.75                                                   # ★ raised (structural)
    belief_state: "tentative"
    epistemic_uncertainty: 0.25
    aleatoric_uncertainty: 0.10
    confidence_decay_rate: 0.005                                       # ★ slow (structural)
    falsifiability_criteria:
      - "Consistently, effects require many (>5) direct causes to explain observed variance"
      - "Parsimonious models consistently underperform complex models across domains"
    machine_testable_criteria:                                         # ★ NEW (Claude Obs 3)
      metric: "parsimonious_model_win_rate"
      threshold: 0.50
      comparison: "less_than"
      sample_size: 50
      evaluation: "If parsimonious models win <50% against complex models, weaken this prior"
    domain_scope: []
    causal_depth: 2

  # ── Confounder Prevalence ────────────────────────────────────────
  - node_id: "t2-confounder-prevalence"
    node_type: "constraint"
    label: "Confounder Prevalence Prior"
    description: >
      When two variables correlate, a hidden confounder is a common
      explanation. The system should actively generate confounder
      hypotheses whenever CORRELATES_WITH relationships are established.
    confidence: 0.75                                                   # ★ raised (structural)
    belief_state: "tentative"
    epistemic_uncertainty: 0.25
    aleatoric_uncertainty: 0.10
    confidence_decay_rate: 0.005                                       # ★ slow (structural)
    falsifiability_criteria:
      - "Across 100+ correlation investigations, confounders are found in <20% of cases"
    machine_testable_criteria:                                         # ★ NEW
      metric: "confounder_detection_rate"
      threshold: 0.20
      comparison: "less_than"
      sample_size: 100
      evaluation: "If confounder detection rate <20% across 100+ investigations, weaken this prior"
    domain_scope: []
    causal_depth: 2

  # ── Threshold Dynamics ───────────────────────────────────────────
  - node_id: "t2-threshold-dynamics"
    node_type: "constraint"
    label: "Threshold Dynamics Prior"
    description: >
      Many causal relationships exhibit threshold behavior — A only
      causes B after a critical threshold is exceeded. The system
      should consider STOCHASTIC_THRESHOLD as a candidate relationship
      type when effects appear discontinuous.
    confidence: 0.65                                                   # ★ (heuristic)
    belief_state: "tentative"
    epistemic_uncertainty: 0.35
    aleatoric_uncertainty: 0.15
    confidence_decay_rate: 0.015                                       # ★ moderate (heuristic)
    falsifiability_criteria:
      - "Effects are consistently proportional to causes with no threshold behavior observed"
    machine_testable_criteria:                                         # ★ NEW
      metric: "threshold_relationship_frequency"
      threshold: 0.05
      comparison: "less_than"
      sample_size: 50
      evaluation: "If <5% of discovered relationships are threshold-type, weaken this prior"
    domain_scope: []
    causal_depth: 2

  # ── Temporal Proximity Bias ──────────────────────────────────────
  - node_id: "t2-temporal-proximity"
    node_type: "constraint"
    label: "Temporal Proximity Prior"
    description: >
      Causes tend to be temporally close to their effects. When
      searching for the cause of an observed event, prioritize
      recent preceding events. However, delayed-effect causation
      exists (Tier 2 — this is a bias, not a rule).
    confidence: 0.65                                                   # ★ (heuristic)
    belief_state: "tentative"
    epistemic_uncertainty: 0.35
    aleatoric_uncertainty: 0.10
    confidence_decay_rate: 0.02                                        # ★ fast (volatile heuristic)
    falsifiability_criteria:
      - "Delayed causes (>10 time steps) are consistently more common than proximate causes"
    machine_testable_criteria:                                         # ★ NEW
      metric: "proximate_cause_rate"
      threshold: 0.40
      comparison: "less_than"
      sample_size: 100
      evaluation: "If proximate causes explain <40% of observations, weaken this prior"
    domain_scope: []
    causal_depth: 2

  # ── Common Cause Structure ───────────────────────────────────────
  - node_id: "t2-common-cause"
    node_type: "constraint"
    label: "Common Cause Structure Prior"
    description: >
      When multiple effects appear simultaneously, a single common
      cause is more likely than multiple independent causes.
    confidence: 0.75                                                   # ★ raised (structural)
    belief_state: "tentative"
    epistemic_uncertainty: 0.25
    aleatoric_uncertainty: 0.10
    confidence_decay_rate: 0.005                                       # ★ slow (structural)
    falsifiability_criteria:
      - "Simultaneous effects consistently have independent causes rather than common causes"
    machine_testable_criteria:                                         # ★ NEW
      metric: "common_cause_explanation_rate"
      threshold: 0.30
      comparison: "less_than"
      sample_size: 50
      evaluation: "If common-cause explains <30% of simultaneous effects, weaken this prior"
    domain_scope: []
    causal_depth: 2

  # ── Proportionality ──────────────────────────────────────────────
  - node_id: "t2-proportionality"
    node_type: "constraint"
    label: "Proportionality Prior"
    description: >
      Larger causes tend to produce larger effects. When an effect
      magnitude is surprising, the system should consider whether the
      cause magnitude was also surprising or whether an additional
      cause is present.
    confidence: 0.65                                                   # ★ raised from 0.60
    belief_state: "tentative"
    epistemic_uncertainty: 0.35
    aleatoric_uncertainty: 0.15
    confidence_decay_rate: 0.02                                        # ★ fast (volatile heuristic)
    falsifiability_criteria:
      - "Effect magnitudes are consistently unrelated to cause magnitudes across domains"
    machine_testable_criteria:                                         # ★ NEW
      metric: "cause_effect_correlation"
      threshold: 0.20
      comparison: "less_than"
      sample_size: 100
      evaluation: "If cause-effect magnitude correlation <0.20, weaken this prior"
    domain_scope: []
    causal_depth: 2
```

### 6.2 Tier 2 Relationships

```yaml
relationships:
  - rel_type: "ENABLES"
    source: "t2-causal-sparsity"
    target: "t2-common-cause"
    strength: 0.6
    confidence: 0.65
    context_tags: ["structural-reasoning"]
    rationale: "Sparsity prior makes common-cause explanations preferred over many-cause."

  - rel_type: "REQUIRES"
    source: "t2-confounder-prevalence"
    target: "t0-causality-direction"
    strength: 0.8
    confidence: 0.90
    context_tags: ["structural-reasoning"]
    rationale: "Confounder detection requires understanding that correlation is not causation."

  - rel_type: "CORRELATES_WITH"
    source: "t2-threshold-dynamics"
    target: "t2-proportionality"
    strength: 0.4
    confidence: 0.50
    context_tags: ["structural-reasoning"]
    rationale: >
      Threshold and proportionality are alternative models for the same
      phenomena. When proportionality fails, threshold dynamics may explain
      the discontinuity. Investigation flag: which model fits better?
```

---

## 7. YAML Schema Format

### 7.1 Validation Schema

★ Changes from v0.1: Added machine_testable_criteria model, formal_expression field.

```python
# src/aria/world_model/seed_ontology.py

class MachineTestCriteria(BaseModel):                                  # ★ NEW (Claude Obs 3)
    """Structured falsifiability criteria for automated evaluation.

    Defines conditions under which a Tier 2 prior should be weakened.
    Schema defined now; evaluation engine implemented in CDD-07.
    """
    model_config = ConfigDict(extra="forbid")

    metric: str = Field(..., description="Name of the measured quantity.")
    threshold: float = Field(..., description="Value at which the prior is weakened.")
    comparison: str = Field(..., pattern=r"^(less_than|greater_than|equals)$")
    sample_size: int = Field(..., ge=1, description="Minimum observations before evaluation.")
    evaluation: str = Field(..., description="Human-readable explanation of the test.")


class SeedNodeYAML(BaseModel):
    """Validation model for a single node in a Seed Ontology YAML file."""
    model_config = ConfigDict(extra="forbid")

    node_id: str = Field(..., pattern=r"^t[012]-.+$")
    node_type: NodeType
    label: str = Field(..., min_length=1, max_length=200)
    description: str = Field(..., min_length=10)
    confidence: float = Field(..., ge=0.0, le=1.0)
    belief_state: BeliefState
    epistemic_uncertainty: float = Field(..., ge=0.0, le=1.0)
    aleatoric_uncertainty: float = Field(..., ge=0.0, le=1.0)
    confidence_decay_rate: float = Field(..., ge=0.0)
    falsifiability_criteria: list[str] = Field(default_factory=list)
    domain_scope: list[str] = Field(default_factory=list)
    causal_depth: int = Field(default=0, ge=0)

    # Optional fields
    identity_invariant: bool = False
    formal_expression: str | None = None                               # ★ Now expected for T0 + T1
    enforcement: str | None = None
    machine_testable_criteria: MachineTestCriteria | None = None       # ★ NEW


class SeedRelationshipYAML(BaseModel):
    """Validation model for a relationship in Seed Ontology YAML."""
    model_config = ConfigDict(extra="forbid")

    rel_type: RelationshipType
    source: str = Field(..., description="node_id of source node.")
    target: str = Field(..., description="node_id of target node.")
    strength: float = Field(..., ge=0.0, le=1.0)
    confidence: float = Field(..., ge=0.0, le=1.0)
    context_tags: list[str] = Field(default_factory=list)
    rationale: str = Field(..., min_length=5)
    probability: float | None = Field(None, ge=0.0, le=1.0)
    threshold: float | None = Field(None, ge=0.0, le=1.0)


class SeedMetadata(BaseModel):
    """Metadata header for a Seed Ontology YAML file."""
    model_config = ConfigDict(extra="forbid")

    tier: int = Field(..., ge=0, le=2)
    source: str = Field(..., pattern=r"^seed-ontology-t[012]$")
    description: str
    version: str
    author: str
    review_authority: str | None = None
    update_policy: str | None = None


class SeedOntologyFileYAML(BaseModel):
    """Validation model for an entire Seed Ontology YAML file."""
    model_config = ConfigDict(extra="forbid")

    metadata: SeedMetadata
    nodes: list[SeedNodeYAML]
    relationships: list[SeedRelationshipYAML] = Field(default_factory=list)
```

### 7.2 Node ID Convention

All seed ontology node IDs follow a strict naming convention: `t{tier}-{descriptive-slug}`. Examples: `t0-non-contradiction`, `t1-gravity`, `t2-causal-sparsity`. This enables quick identification of any node's tier from its ID alone.

---

## 8. Ontology Loader

### 8.1 Loader Pipeline

★ Changes from v0.1: Added Step 2b (cross-tier reference pre-validation), version check stub (Grok), CLI hard review awareness.

```
seed_data/tier0_logical_absolutes.yml
seed_data/tier1_physical_bedrock.yml
seed_data/tier2_structural_priors.yml
    │
    ▼
Step 1: Read YAML files
    │
    ▼
Step 2a: Validate each file against SeedOntologyFileYAML (Pydantic)
    ├── Reject on any validation error — no partial loads
    ├── Verify node_id prefix matches tier
    │
    ▼
★ Step 2b: Cross-tier reference pre-validation (Claude Obs 2)
    ├── Build combined node_id registry across all three YAML files
    ├── For each relationship in all files:
    │     Verify source exists in combined registry
    │     Verify target exists in combined registry
    ├── If any endpoint missing → raise CrossTierReferenceError
    │     with source_file, rel_type, and missing node_id
    ├── This catches mistyped cross-tier references BEFORE Neo4j
    │
    ▼
Step 3: Convert SeedNodeYAML → WorldNode (CDD-01 schema)
    ├── Set source = metadata.source
    ├── Set salience_score = 1.0 (seed nodes start maximally salient)
    ├── Set emotional_valence = 0.0 (neutral at initialization)
    ├── Set version = 1
    ├── Set context_diversity = 1 (established in seed context)
    │
    ▼
Step 4: Tier-specific property injection
    ├── Tier 0: identity_invariant = true (if not already set)
    ├── Tier 0: confidence_decay_rate forced to 0.0
    ├── Tier 0: verify formal_expression is present
    ├── Tier 1: confidence_decay_rate forced to 0.0
    ├── ★ Tier 1: verify formal_expression is present
    ├── Tier 2: verify confidence_decay_rate > 0 (must decay)
    │
    ▼
Step 5: Load into Neo4j via GraphStore Protocol
    ├── Load order: Tier 0 first, then Tier 1, then Tier 2
    ├── Each node: GraphStore.create_node(world_node)
    ├── Each relationship: GraphStore.create_relationship(causal_rel)
    │
    ▼
Step 6: Post-load verification
    ├── Query all nodes by source → count matches YAML for each tier
    ├── Verify all Tier 0 nodes have belief_state = CONFIRMED
    ├── Verify all Tier 0 nodes have identity_invariant = true
    ├── Verify no Tier 0 nodes have confidence_decay_rate > 0
    ├── ★ Verify ontology version tag matches YAML metadata version
    │
    ▼
Step 7: Emit structured audit event
    Log: "seed_ontology.loaded" with tier counts, node_ids, version
```

### 8.2 Loader Interface

```python
class SeedOntologyLoader:
    """Loads Seed Ontology YAML files into the World Model via GraphStore.

    The loader is idempotent: running it twice with the same YAML files
    does not create duplicates (checks node_exists before creating).
    """

    def __init__(self, graph_store: GraphStore, seed_data_path: Path) -> None:
        self.graph = graph_store
        self.seed_path = seed_data_path

    def load_all(self) -> LoadResult:
        """Load all three tiers in order. Returns summary."""
        ...

    def load_tier(self, tier: int) -> TierLoadResult:
        """Load a specific tier. Tier 0 must be loaded before 1 or 2."""
        ...

    def verify_integrity(self) -> IntegrityReport:
        """Post-load verification. Returns discrepancies if any."""
        ...

    def is_loaded(self) -> bool:
        """Check if seed ontology has already been loaded."""
        ...

    def validate_cross_tier_references(                                # ★ NEW
        self, files: list[SeedOntologyFileYAML]
    ) -> None:
        """Pre-validate all relationship endpoints across all YAML files.

        Raises CrossTierReferenceError if any relationship references
        a node_id not found in any tier's node list.
        """
        ...


class LoadResult(BaseModel):
    """Summary of a full seed ontology load operation."""
    tier0_nodes: int
    tier0_relationships: int
    tier1_nodes: int
    tier1_relationships: int
    tier2_nodes: int
    tier2_relationships: int
    total_nodes: int
    total_relationships: int
    load_duration_ms: float
    ontology_version: str                                              # ★ NEW
    errors: list[str] = Field(default_factory=list)
    success: bool


class IntegrityReport(BaseModel):
    """Result of post-load integrity verification."""
    all_tiers_loaded: bool
    tier0_count_match: bool
    tier1_count_match: bool
    tier2_count_match: bool
    tier0_all_confirmed: bool
    tier0_all_invariant: bool
    tier0_no_decay: bool
    tier0_all_have_formal_expression: bool                             # ★ NEW
    tier1_all_have_formal_expression: bool                             # ★ NEW
    discrepancies: list[str] = Field(default_factory=list)
```

### 8.3 Idempotency

The loader checks `graph.node_exists(node_id)` before each `create_node`. If a node already exists, it is skipped (not updated). Seed Ontology updates in Phase 1 require a conscious act (version bump + loader update), not an automatic overwrite.

---

## 9. Tier Protection Rules

### 9.1 Tier 0 — Immutable

```
Protection level: ABSOLUTE
Any attempt to modify or delete a Tier 0 node → ImmutableNodeError
★ Any attempt to delete or update relationships FROM Tier 0 → ImmutableNodeError (CDD-01 v1.1)

Enforced at:
  1. GraphStore.update_node()     — checks source == seed-ontology-t0
  2. GraphStore.soft_delete_node() — checks source == seed-ontology-t0
  3. ★ GraphStore.delete_relationship() — checks source node tier (CDD-01 v1.1)
  4. ★ GraphStore.update_relationship() — checks source node tier (CDD-01 v1.1)
  5. Invariant Checker (CDD-03)   — runtime monitoring

Protection applies to:
  - All 20 node properties (none can be changed)
  - ★ Relationships originating FROM Tier 0 nodes (cannot be deleted or updated)
  - ★ Relationships BETWEEN Tier 0 nodes (cannot be auto-created)

What IS allowed:
  - Creating new relationships TO Tier 0 nodes from non-Tier-0 nodes
  - Reading Tier 0 nodes (all queries)
  - Using Tier 0 nodes in subgraph extraction
  - ★ Creating relationships FROM/BETWEEN Tier 0 via hard_review_override only
```

### 9.2 Tier 1 — Hard Review Required

★ Changes from v0.1: CLI workflow replaces bare flag. Revalidation clause added (GPT-5).

```
Protection level: GUARDED
Modification requires explicit hard_review_override with CLI confirmation

★ Hard Review CLI Workflow:
  1. Developer calls update_node() with hard_review_override=True
  2. System prints full diff to console
  3. System prompts: "Tier 1 modification requires hard review."
     "Reviewer ID: ___"
     "Justification: ___"
  4. Reviewer enters both interactively
  5. reviewer_id and justification logged in WorldNodeVersion diff
  6. "tier1.hard_review_modification" structured event emitted

★ Tier 1 Revalidation Clause (GPT-5):
  Tier 1 nodes must be flagged for revalidation during Tier Health
  Review if no supporting evidence has been observed within N
  operational cycles. This is NOT automatic decay — confidence and
  belief_state remain unchanged until a human reviewer evaluates the
  flag and decides whether revision is warranted. N is defined in
  CDD-06 (Formal Semantics). For Phase 1, the revalidation check is
  a manual step during sprint retrospectives.

Enforced at:
  1. GraphStore.update_node()       — requires hard_review_override
  2. GraphStore.soft_delete_node()  — requires hard_review_override
  3. ★ GraphStore.delete_relationship()  — if source is T1, requires override
  4. ★ GraphStore.update_relationship()  — if source is T1, requires override
```

### 9.3 Tier 2 — Standard Rules

```
Protection level: STANDARD
Normal update rules apply (Pydantic validation, version increment, diff logging)

Additional Tier 2 behaviors:
  - confidence_decay_rate MUST be > 0 (Tier 2 beliefs decay without reinforcement)
  - ★ Decay rates are non-uniform (0.005 structural, 0.015–0.02 heuristic)
  - Adversarial de-biasing (CDD-07) can challenge and revise Tier 2 nodes
  - ★ machine_testable_criteria enables automated evaluation (CDD-07)
  - Context diversity requirements apply for BSM state transitions
  - Tier 2 nodes start as TENTATIVE, not CONFIRMED
```

### 9.4 Protection Decision Matrix

★ Updated from v0.1 with relationship mutation protection.

| Operation | Tier 0 | Tier 1 | Tier 2 |
|-----------|--------|--------|--------|
| Create (via loader) | ✅ Allowed | ✅ Allowed | ✅ Allowed |
| Read | ✅ Allowed | ✅ Allowed | ✅ Allowed |
| Update properties | ❌ ImmutableNodeError | ⚠️ Requires hard_review_override | ✅ Standard rules |
| Soft-delete | ❌ ImmutableNodeError | ⚠️ Requires hard_review_override | ✅ Standard rules |
| Create relationship FROM | ❌ ★ Blocked (unless hard_review_override) | ⚠️ Requires hard_review_override | ✅ Standard rules |
| Create relationship TO | ✅ Allowed | ✅ Allowed | ✅ Allowed |
| ★ Delete relationship FROM | ❌ ImmutableNodeError | ⚠️ Requires hard_review_override | ✅ Standard rules |
| ★ Update relationship FROM | ❌ ImmutableNodeError | ⚠️ Requires hard_review_override | ✅ Standard rules |
| Create relationship BETWEEN T0 | ❌ ★ Blocked (unless hard_review_override) | N/A | N/A |
| Confidence decay | ❌ decay_rate = 0.0 | ❌ decay_rate = 0.0 | ✅ ★ Non-uniform rates |
| BSM transition | ❌ Stays CONFIRMED forever | ⚠️ Only via hard review | ✅ Normal BSM rules |
| ★ Revalidation | ❌ Never (axioms) | ★ Flag after N cycles without evidence | ✅ Natural decay handles |

---

## 10. Integration Points

### 10.1 CDD-01 Integration (GraphStore)

| Touchpoint | How |
|------------|-----|
| `GraphStore.create_node()` | Loader uses this for all node creation |
| `GraphStore.create_relationship()` | Loader uses this for all relationship creation |
| `ImmutableNodeError` | Raised for Tier 0/1 modification attempts |
| `NodeSource` enum | SEED_ONTOLOGY_T0, T1, T2 values |
| `WorldNodeVersion` | Initial creation diffs recorded for all seed nodes |
| ★ `CrossTierReferenceError` | Raised during pre-load validation (CDD-01 v1.1) |
| ★ Relationship tier checks | delete/update check source tier (CDD-01 v1.1) |

### 10.2 CDD-03 Integration (BSM & Invariant Checker)

| Touchpoint | How |
|------------|-----|
| Invariant Checker | Monitors that Tier 0 nodes remain unmodified |
| BSM transitions | Tier 0 never transitions. Tier 1 only via hard review. Tier 2 follows normal rules. |
| Non-contradiction enforcement | `t0-non-contradiction` defines the invariant for CONTRADICTS checks |
| Temporal ordering enforcement | `t0-temporal-ordering` defines the DAG invariant for PRECEDES |

### 10.3 CDD-06 Integration (Formal Semantics)

| Touchpoint | How |
|------------|-----|
| ★ Revalidation cycle count N | Defined in CDD-06, consumed by Tier 1 revalidation clause |
| Confidence thresholds | Tier 2 state transition thresholds defined in CDD-06 |

### 10.4 CDD-07 Integration (Adversary Simulator)

| Touchpoint | How |
|------------|-----|
| Tier 2 de-biasing | Adversary Simulator challenges Tier 2 priors |
| ★ machine_testable_criteria | Structured evaluation conditions for automated testing |
| Confounder generation | `t2-confounder-prevalence` drives confounder hypothesis generation |

### 10.5 CDD-10 Integration (DriftMonitor)

| Touchpoint | How |
|------------|-----|
| ★ Tier Health Metrics | Tier 1 growth rate, promotion frequency, confidence saturation |
| ★ Tier 1 growth cap | If Tier 1 exceeds 30 nodes, trigger tier health review |

---

## 11. Configuration Parameters

```python
class SeedOntologyConfig(BaseSettings):
    """Configuration for Seed Ontology loading and protection."""

    # File paths
    seed_data_path: Path = Path("seed_data/")
    tier0_filename: str = "tier0_logical_absolutes.yml"
    tier1_filename: str = "tier1_physical_bedrock.yml"
    tier2_filename: str = "tier2_structural_priors.yml"

    # Protection
    tier1_hard_review_enabled: bool = True
    tier1_require_justification: bool = True
    tier1_max_nodes_before_review: int = 30                            # ★ GPT-5

    # Tier 2 defaults
    tier2_default_decay_rate: float = 0.01
    tier2_minimum_decay_rate: float = 0.001

    # Loading
    loader_idempotent: bool = True
    loader_verify_after_load: bool = True
    loader_cross_tier_validation: bool = True                          # ★ Claude Obs 2

    model_config = SettingsConfigDict(env_prefix="ARIA_SEED_")
```

---

## 12. Design Notes — Future Phases

These are documented decisions that affect architecture beyond Phase 1. They are NOT implemented in Phase 1 but are recorded here to prevent scope surprise later.

### ★ 12.1 Tier 0 Migration Path (GPT-5)

Tier 0 is runtime-immutable but not eternally-immutable. The migration mechanism:

- Each YAML file carries a version in its metadata (currently "1.0")
- A future `SeedOntologyMigrator` can replace the entire Tier 0 set atomically — never live mutation, only full-version swap under the existing loader
- Migration requires all reviewing systems to sign off (same multi-LLM process used for CDDs)
- During migration, the old Tier 0 nodes are soft-deleted (with reason "ontology_migration_v{old}_to_v{new}") and new nodes are created
- No mixed-version Tier 0 states — the swap is all-or-nothing

**Implementation:** Phase 2+. The current loader already stores version metadata. The migrator is a future extension.

### ★ 12.2 Tier 1 Growth Cap (GPT-5)

If Tier 1 exceeds 30 nodes, a tier health review is triggered. This is a manual process in Phase 1 (check during sprint retrospectives). In Phase 2, the DriftMonitor (CDD-10) automates this check.

The 30-node cap is a soft limit — it triggers review, not rejection. The architect can approve growth beyond 30 if justified.

### ★ 12.3 Promotion Prerequisites (GPT-5)

If a Tier 2 node is ever promoted to Tier 1 (via hard review), it must:
- Have a `formal_expression` added (Tier 1 requirement)
- Have its falsifiability_criteria strengthened (Tier 1 standard)
- Not semantically duplicate an existing Tier 1 node
- Go through full multi-LLM review

### ★ 12.4 Cross-Domain Semantic Overlay (GPT-5)

Phase 5+ concern. When ARIA operates across domains, the `domain_scope` field on nodes enables domain-scoped semantic overlays. Same label (e.g., "clean") can have different operational meanings in different domain_scopes. The seed ontology's domain-agnostic Tier 0 and Tier 2 nodes remain shared; Tier 1 may need domain-specific extensions.

---

## 13. Test Plan

### 13.1 Unit Tests (`tests/unit/test_seed_ontology.py`)

| Test | Validates |
|------|-----------|
| `test_tier0_yaml_validates` | Tier 0 YAML passes SeedOntologyFileYAML validation |
| `test_tier1_yaml_validates` | Tier 1 YAML passes validation |
| `test_tier2_yaml_validates` | Tier 2 YAML passes validation |
| `test_node_id_convention` | All node_ids match `t{tier}-{slug}` pattern |
| `test_tier0_all_confirmed` | All Tier 0 nodes have belief_state = CONFIRMED |
| `test_tier0_all_invariant` | All Tier 0 nodes have identity_invariant = true |
| `test_tier0_no_decay` | All Tier 0 nodes have confidence_decay_rate = 0.0 |
| `test_tier0_max_confidence` | All Tier 0 nodes have confidence = 1.0 |
| ★ `test_tier0_all_have_formal_expression` | All Tier 0 nodes have formal_expression set |
| `test_tier1_all_confirmed` | All Tier 1 nodes have belief_state = CONFIRMED |
| `test_tier1_no_decay` | All Tier 1 nodes have confidence_decay_rate = 0.0 |
| `test_tier1_has_falsifiability` | All Tier 1 nodes have non-empty falsifiability_criteria |
| ★ `test_tier1_all_have_formal_expression` | All Tier 1 nodes have formal_expression set |
| `test_tier2_all_tentative` | All Tier 2 nodes have belief_state = TENTATIVE |
| `test_tier2_has_decay` | All Tier 2 nodes have confidence_decay_rate > 0 |
| `test_tier2_has_falsifiability` | All Tier 2 nodes have non-empty falsifiability_criteria |
| ★ `test_tier2_nonuniform_decay` | Structural priors (0.005) differ from heuristic priors (0.015–0.02) |
| ★ `test_tier2_nonuniform_confidence` | Structural (0.75) vs heuristic (0.65) confidence values |
| ★ `test_machine_testable_criteria_schema` | MachineTestCriteria validates correctly |
| `test_relationship_endpoints_exist` | All relationship source/target IDs exist in nodes list |
| `test_no_self_referential_relationships` | No relationship has source == target |
| `test_yaml_extra_fields_rejected` | YAML with unexpected fields fails validation |
| `test_seed_to_worldnode_conversion` | SeedNodeYAML converts to valid WorldNode |

### 13.2 Integration Tests (`tests/integration/test_seed_loading.py`)

| Test | Validates |
|------|-----------|
| `test_load_all_tiers` | Full load creates correct node and relationship counts |
| `test_load_order_enforced` | Loading Tier 1 before Tier 0 fails gracefully |
| `test_idempotent_reload` | Loading twice produces same counts |
| `test_tier0_immutable_after_load` | update_node on Tier 0 raises ImmutableNodeError |
| `test_tier0_undeletable_after_load` | soft_delete on Tier 0 raises ImmutableNodeError |
| ★ `test_tier0_relationship_deletion_blocked` | Deleting relationship from Tier 0 source raises error |
| ★ `test_tier0_relationship_update_blocked` | Updating relationship from Tier 0 source raises error |
| `test_tier1_requires_hard_review` | update_node on Tier 1 without override raises error |
| `test_tier1_allows_hard_review` | update_node on Tier 1 with override succeeds and logs |
| ★ `test_hard_review_cli_captures_metadata` | CLI prompts capture reviewer_id and justification |
| `test_tier2_normal_update` | update_node on Tier 2 succeeds with standard rules |
| `test_tier2_decay_applies` | Tier 2 nodes lose confidence after decay cycle |
| `test_cross_tier_relationships` | Tier 1 IS_PART_OF Tier 0 loads correctly |
| `test_new_relationship_to_tier0` | Creating relationship TO Tier 0 succeeds |
| ★ `test_cross_tier_reference_validation` | Mistyped cross-tier ref caught with CrossTierReferenceError |
| `test_integrity_report_clean` | verify_integrity returns all-clean after successful load |
| ★ `test_elasticity_node_loaded` | t1-elasticity present after load |
| ★ `test_reflexivity_axiom_loaded` | t0-reflexivity present after load |
| `test_invariant_alarm_on_tier0_modification` | Tier 0 modification triggers InvariantViolationError + alert |

### 13.3 Content Validation Tests (`tests/unit/test_seed_content.py`)

| Test | Validates |
|------|-----------|
| `test_non_contradiction_exists` | t0-non-contradiction node is defined |
| `test_temporal_ordering_exists` | t0-temporal-ordering node is defined |
| ★ `test_reflexivity_exists` | t0-reflexivity node is defined |
| `test_all_tier0_have_formal_expression` | Every Tier 0 node has a formal_expression |
| ★ `test_all_tier1_have_formal_expression` | Every Tier 1 node has a formal_expression |
| `test_conservation_has_physics_instantiation` | t1-energy-conservation IS_PART_OF t0-conservation-abstract |
| ★ `test_elasticity_enables_conservation` | t1-elasticity ENABLES t1-energy-conservation |
| `test_tier1_covers_physics_scenarios` | Tier 1 includes nodes relevant to all 7 physics scenarios |
| `test_confounder_prevalence_links_to_directionality` | t2-confounder-prevalence REQUIRES t0-causality-direction |
| `test_no_orphan_nodes` | Every node participates in at least one relationship |
| `test_no_circular_requires` | REQUIRES relationships form a DAG |

### ★ 13.4 Property-Based Tests (`tests/property/test_seed_properties.py`)

Grok recommendation: cross-tier integrity testing via Hypothesis.

| Test | Validates |
|------|-----------|
| `test_random_tier0_modification_always_rejected` | Hypothesis generates random updates → all rejected on Tier 0 |
| `test_random_cross_tier_relationships_valid` | Hypothesis generates random relationships → all endpoints exist |

---

## 14. Acceptance Criteria

★ Expanded from 14 to 20 criteria.

| # | Criterion | Sprint | Verification |
|---|-----------|--------|-------------|
| 1 | ★ 9 Tier 0 logical absolutes defined in YAML (including reflexivity) | S2 | All axioms present with formal_expression |
| 2 | Tier 0 loaded as immutable nodes | S2 | Modification raises ImmutableNodeError + alarm |
| 3 | ★ 12 Tier 1 physical bedrock nodes defined (including elasticity) | S2 | All nodes present with formal_expression + falsifiability |
| 4 | Tier 1 loaded with ★ CLI hard review protection | S2 | Override requires reviewer_id + justification |
| 5 | ★ 6 Tier 2 structural priors with non-uniform confidence and decay | S2 | Structural (0.75/0.005) vs heuristic (0.65/0.015–0.02) |
| 6 | Tier 2 nodes loaded as TENTATIVE with decay > 0 | S2 | All Tier 2 decay, start TENTATIVE |
| 7 | YAML validation rejects malformed files | S2 | Extra fields, missing required, wrong types all caught |
| 8 | ★ Cross-tier reference pre-validation catches broken refs | S2 | CrossTierReferenceError with file + rel + node diagnostics |
| 9 | Loader is idempotent | S2 | Loading twice = same result |
| 10 | Post-load integrity verification passes | S2 | IntegrityReport all clean |
| 11 | Cross-tier relationships load correctly | S2 | IS_PART_OF, REQUIRES between tiers verified |
| 12 | ★ Tier protection matrix fully enforced including relationship mutations | S2 | All cells in §9.4 verified by integration tests |
| 13 | ★ Relationship deletion from Tier 0 blocked | S2 | ImmutableNodeError raised (CDD-01 v1.1) |
| 14 | ★ Relationship update from Tier 0 blocked | S2 | ImmutableNodeError raised (CDD-01 v1.1) |
| 15 | Structured audit events for all load operations | S2 | seed_ontology.loaded event with counts + version |
| 16 | ★ machine_testable_criteria schema defined | S2 | MachineTestCriteria Pydantic model validates |
| 17 | ★ Tier 1 formal expressions present for all nodes | S2 | Physics equations stored as strings |
| 18 | ★ Tier 1 revalidation clause documented | S2 | Design rule in protection section |
| 19 | All unit + content validation tests pass | S2 | 100% of §13.1 and §13.3 green |
| 20 | All integration tests pass | S2 | 100% of §13.2 green |

---

## 15. Open Questions — All Resolved

All 7 original open questions have been answered through the multi-LLM review process.

| Q | Resolution | Decided By |
|---|-----------|------------|
| Q1: Tier 0 completeness | 8 original + added reflexivity = 9 axioms | Grok (suggested) + Gemini (sufficient) |
| Q2: Tier 1 physics scope | Added elasticity for collision scenarios | Gemini + Grok (both flagged) |
| Q3: Tier 2 confidence values | Non-uniform: 0.75 structural, 0.65 heuristic | Gemini (lower) + Grok (higher) → split |
| Q4: Outgoing relationship protection | Blocked for auto, allowed via hard_review_override | Gemini (strict) + Grok (relaxed) → compromise |
| Q5: Hard review implementation | CLI workflow: prompt for reviewer_id + justification | Grok (CLI) + Gemini (logging) |
| Q6: Tier 2 decay uniformity | Non-uniform: 0.005 structural, 0.015–0.02 heuristic | Gemini + Grok (consensus) |
| Q7: Formal expressions for Tier 1 | Added physics equations to all Tier 1 nodes | Gemini + Grok (consensus) |

---

## Appendix A: Review Feedback Incorporation Record

### GPT-5 (OpenAI)

| Item | Section |
|------|---------|
| Frozen ontology migration path | §12.1 |
| Promotion inflation guard (30-node cap) | §12.2 |
| Overconfidence lock-in → revalidation clause | §9.2 |
| Cross-domain contamination note | §12.4 |
| Tier health metrics | §10.5, §12.2 |
| Semantic inheritance constraints | §12.3 |
| Revalidation windows → Tier 1 revalidation clause | §9.2 |
| Tier quarantine → already exists (CDD-05) | §10 cross-reference |

### Gemini (Google)

| Item | Section |
|------|---------|
| Add elasticity to Tier 1 | §5.1 |
| Non-uniform Tier 2 decay rates | §6.1 |
| Formal expressions for Tier 1 | §5.1 |
| Block auto-creation between Tier 0 nodes | §9.1 |
| REASONING_LOG on hard review → via PropertyDiff | §9.2 |

### Grok (xAI)

| Item | Section |
|------|---------|
| Add reflexivity axiom | §4.1 |
| Add elasticity to Tier 1 | §5.1 |
| Non-uniform decay rates | §6.1 |
| Formal expressions for Tier 1 | §5.1 |
| CLI hard review workflow | §9.2 |
| YAML schema validator suggestion | §7.1 (Pydantic validation) |
| Property-based cross-tier tests | §13.4 |
| Loader batch operations note | §8.2 |
| Loader version check stub | §8.1 Step 6 |

### Claude Opus 4.6

| # | Observation | Section |
|---|------------|---------|
| 1 | Relationship deletion tier check (→ CDD-01 v1.1) | §9.1, §9.4 |
| 2 | Cross-tier reference pre-validation | §8.1 Step 2b |
| 3 | Machine-testable criteria for Tier 2 | §6.1, §7.1 |

---

*End of CDD-02 v1.0: Seed Ontology & Tier Protection*

**Project ARIA · Adaptive Reasoning Integrated Architecture**  
Component Design Document 02 — Knowledge Foundation  
Approved by All Reviewing Systems — Ready for Implementation
