# CDD-03: Belief State Machine & Invariant Checker

> **Status:** APPROVED — All Reviewing Systems Signed Off  
> **Version:** 1.0 (Incorporates Multi-LLM Review Feedback)  
> **Author:** Jun (Architect) + Claude Opus 4.6 (Design Partner)  
> **Depends On:** CDD-01 v1.1 (World Model Schema & GraphStore Protocol), CDD-02 v1.0 (Seed Ontology & Tier Protection)  
> **Consumed By:** CDD-04 (Predictive Loop), CDD-05 (Quarantine), CDD-06 (Formal Semantics), CDD-07 (Adversary Simulator), CDD-10 (DriftMonitor)  
> **Sprint:** S1 (BSM core) + S2 (integration with Seed Ontology)  
> **Classification:** Confidential — Core Team & Designated Review Partners  
> **Review Contributors:** GPT-5 (OpenAI) · Gemini (Google) · Grok (xAI) · Claude Opus 4.6

---

## Review & Approval Record

| System | Status | Conditions | Resolution |
|--------|--------|------------|------------|
| GPT-5 (OpenAI) | ✅ APPROVED | 7 conditions (all accepted) | Bounded cascade, reconsideration op, threshold adjustment, audit archival plan, perf targets, promotion scope, context signature |
| Gemini (Google) | ✅ APPROVED | No conditions | Clean sign-off |
| Grok (xAI) | ✅ APPROVED | No conditions | Clean sign-off |
| Claude Opus 4.6 | ✅ 4th REVIEWER | Design partner | Authored spec, incorporated all feedback |

**Consensus:** 4/4 approved. All changes from v0.1 marked with ★.

---

## 1. Purpose & Scope

CDD-03 specifies two tightly coupled components:

**1.1 The Belief State Machine (BSM)** — The finite state machine that governs how every node in the World Model transitions between epistemic states. If CDD-01 defines the *structure* of nodes and CDD-02 defines the *starting content*, CDD-03 defines the *rules for how beliefs evolve over time*. Every confidence change, every observation, every decay cycle must pass through the BSM. There are no backdoors.

**1.2 The Invariant Checker** — The runtime enforcement engine that monitors every graph mutation for violations of Tier 0 logical axioms (defined in CDD-02). If the BSM is the system's epistemic conscience, the Invariant Checker is its logical immune system — it catches contradictions, temporal cycle violations, and tier protection breaches before they corrupt the World Model.

Together, these components ensure that ARIA's beliefs evolve *lawfully* — respecting both logical constraints (Tier 0) and epistemic governance rules (state transition preconditions).

### 1.1 Neuroscience Grounding

| Component | Cognitive Analogue | Reference |
|-----------|-------------------|-----------|
| BSM | Epistemic updating / Bayesian belief revision (Karl Friston, Free Energy Principle) | Beliefs update toward states that minimize surprise, but transitions are constrained by prior certainty |
| Invariant Checker | Error monitoring / Conflict detection (Anterior Cingulate Cortex) | The ACC detects conflicts between expectations and observations; the Invariant Checker detects conflicts between axioms and graph state |

### 1.2 What This CDD Does NOT Cover

| Excluded | Covered In |
|----------|------------|
| GraphStore Protocol and data structures | CDD-01 |
| Seed Ontology content and YAML loading | CDD-02 |
| Predictive Loop (the cycle that triggers BSM transitions) | CDD-04 |
| Quarantine layer, promotion orchestration, ★ WAL + rollback | CDD-05 |
| Formal Semantics (threshold values, ★ context signature refinement) | CDD-06 |
| Adversary Simulator (generates scenarios that test BSM) | CDD-07 |

---

## 2. Specification References

| Spec Section | Content | Relevance |
|-------------|---------|-----------|
| v2.1 §4.1 | belief_state enum: 6 states | Direct implementation target |
| v2.1 §3.3 | Simulation Quarantine Layer | SIMULATED state constraints |
| v2.1 §3.5 | External Truth Pragmatism | What "confirmed" operationally means |
| v2.1 §3.2 | Acid Test | BSM must support full predict→fail→update cycle |
| Impl Plan §2.5 | BSM: Custom Python Module + Neo4j Triggers | Technology decision |
| Impl Plan §6 S1 | Sprint 1 deliverables | BSM with all 6 states, preconditions, audit logging |
| CDD-01 §3.1 | GraphStore.update_node, apply_confidence_decay | BSM wraps these operations |
| CDD-01 §3.4 | InvariantViolationError | Invariant Checker raises this |
| CDD-01 §5.4 | Confidence Decay with BSM reconciliation | Synchronous reconciliation contract |
| CDD-01 §8.2 | Invariant Checking integration points | Called on every write |
| CDD-02 §4–6 | Tier 0/1/2 nodes with belief states and invariants | Invariant content source |
| CDD-02 §9 | Tier Protection Rules | BSM enforces tier-specific transition rules |

---

## 3. Belief State Machine

### 3.1 The Six States

```
                    ┌──────────────────────────────────────────────┐
                    │          ARIA Belief State Machine            │
                    │                                              │
                    │  ┌──────────┐     ┌──────────┐              │
                    │  │HYPOTHESIS│────▶│TENTATIVE │              │
                    │  └──────────┘     └──────────┘              │
                    │       │ ▲              │ ▲                   │
                    │       │ │              │ │                   │
                    │       │ │              ▼ │                   │
                    │       │ │         ┌──────────┐              │
                    │       │ └─────────│CONFIRMED │              │
                    │       │           └──────────┘              │
                    │       │                │                     │
                    │       ▼                ▼                     │
                    │  ┌──────────┐    ┌───────────┐              │
                    │  │ REFUTED  │    │DEPRECATED │              │
                    │  └──────────┘    └───────────┘              │
                    │                                              │
                    │  ┌──────────┐  (Quarantine Only)            │
                    │  │SIMULATED │──────────────────▶ HYPOTHESIS │
                    │  └──────────┘  (via promotion)              │
                    │                                              │
                    │  ★ REFUTED ─ ─ ─(reconsider)─ ─ ─▶ new     │
                    │               HYPOTHESIS (new node,         │
                    │               RECONSIDERS edge)              │
                    └──────────────────────────────────────────────┘
```

| State | Meaning | Typical Confidence Range | Can Decay? |
|-------|---------|-------------------------|------------|
| HYPOTHESIS | Initial belief. Unvalidated. | 0.1 – 0.5 | Yes |
| TENTATIVE | Some supporting evidence. Not yet robust. | 0.4 – 0.8 | Yes |
| CONFIRMED | Survived adversarial testing in multiple contexts. | 0.7 – 1.0 | Yes (triggers demotion if drops below threshold) |
| DEPRECATED | Was confirmed but evidence has weakened. | 0.2 – 0.6 | Yes |
| REFUTED | Directly contradicted by observation. ★ Terminal but reconsidarable via new node. | 0.0 – 0.2 | No (terminal) |
| SIMULATED | Dream Engine output. Quarantine only. | 0.0 – 1.0 (trust score) | No (managed by trust_score instead) |

**Key principle:** Confidence range overlaps are intentional. A HYPOTHESIS at 0.5 and a TENTATIVE at 0.5 are different — the TENTATIVE has passed validation checks that the HYPOTHESIS hasn't. Belief state encodes *epistemic quality*, not just certainty magnitude.

### 3.2 Valid Transitions

| # | From | To | Trigger | Preconditions |
|---|------|----|---------|---------------|
| T1 | HYPOTHESIS | TENTATIVE | Supporting evidence observed | confidence ≥ `min_tentative_confidence` AND context_diversity ≥ `min_tentative_context_diversity` |
| T2 | TENTATIVE | CONFIRMED | Survives adversarial testing | confidence ≥ `min_confirmed_confidence` AND context_diversity ≥ `min_confirmed_context_diversity` AND len(falsifiability_criteria) > 0 AND adversarial_survival_count ≥ `min_adversarial_survival` |
| T3 | CONFIRMED | DEPRECATED | Confidence decays below threshold | confidence < `confirmed_decay_threshold` (triggered by BSM reconciliation after decay) |
| T4 | DEPRECATED | CONFIRMED | Re-confirmed with new evidence | Same preconditions as T2 (full re-confirmation) |
| T5 | DEPRECATED | REFUTED | Direct contradicting observation | Contradicting observation has higher confidence AND comes from independent source |
| T6 | HYPOTHESIS | REFUTED | Direct contradicting observation | Same as T5 |
| T7 | TENTATIVE | REFUTED | Direct contradicting observation | Same as T5 |
| T8 | TENTATIVE | HYPOTHESIS | Confidence drops below threshold | confidence < `min_tentative_confidence` (triggered by decay reconciliation) |
| T9 | CONFIRMED | TENTATIVE | Partial disconfirmation | Evidence weakens but not refuted. confidence drops below `min_confirmed_confidence` but above `min_tentative_confidence` |
| T10 | SIMULATED | HYPOTHESIS | Promotion from quarantine | Real-world corroboration OR ≥ `min_independent_confirmations` context-aligned confirmations. Source ≠ simulated after promotion. Via CDD-05 promotion contract. |

### 3.3 Invalid Transitions (Explicitly Blocked)

| From | To | Reason |
|------|----|--------|
| REFUTED | Any state | ★ REFUTED is terminal. Use `reconsider()` to create new linked node (GPT-5 Condition 2). |
| SIMULATED | TENTATIVE / CONFIRMED | Simulated nodes cannot skip HYPOTHESIS. Must go through full epistemic pipeline after promotion. |
| SIMULATED | DEPRECATED / REFUTED | Simulated nodes are not in the epistemic pipeline — they use trust_score instead. |
| Any state | SIMULATED | Nodes cannot be demoted to SIMULATED. SIMULATED is an origin state, not a destination. |
| HYPOTHESIS | CONFIRMED | Cannot skip TENTATIVE. Beliefs must accumulate evidence gradually. No shortcuts. |
| HYPOTHESIS | DEPRECATED | Cannot deprecate what was never confirmed. |

### 3.4 Tier-Specific Transition Overrides

CDD-02's tier protection rules constrain BSM transitions for seed ontology nodes:

| Node Source | Allowed Transitions | Enforcement |
|-------------|-------------------|-------------|
| `seed-ontology-t0` | NONE. Stays CONFIRMED forever. | BSM rejects any transition request. ImmutableNodeError raised. |
| `seed-ontology-t1` | Only via hard_review_override | BSM checks for override flag before executing transition. Logs tier1.bsm_hard_review event. |
| `seed-ontology-t2` | All normal transitions | Standard BSM rules apply. Starts TENTATIVE. |
| `simulated` | T10 only (SIMULATED → HYPOTHESIS via promotion) | ★ Promotion orchestration handled by CDD-05 (WAL + idempotent steps). BSM provides the transition; CDD-05 provides the orchestration wrapper. |
| All other sources | All applicable transitions | Standard BSM rules. |

---

## 4. BSM Implementation

### 4.1 State Machine Architecture

The BSM uses the `transitions` Python library for clean state machine patterns, wrapped in a service class that enforces preconditions and produces audit trails.

```python
# src/aria/core/belief_state_machine.py

from transitions import Machine
from aria.world_model.schema import WorldNode, BeliefState, NodeSource
from aria.core.protocols import GraphStore
from aria.core.invariants import InvariantChecker
from aria.core.config import BSMConfig


class BeliefStateMachine:
    """Governs all epistemic state transitions in the ARIA World Model.

    No belief_state change can occur outside this class. All transitions
    are precondition-gated, audited, and invariant-checked.

    ★ Uses expected_version on all GraphStore.update_node() calls for
    optimistic concurrency (GPT-5 additional observation 1).
    """

    # Valid transitions map
    TRANSITIONS = [
        # T1: HYPOTHESIS → TENTATIVE
        {"trigger": "promote_to_tentative", "source": "hypothesis", "dest": "tentative",
         "conditions": ["_check_tentative_preconditions"],
         "before": ["_check_tier_protection"],
         "after": ["_log_transition", "_emit_transition_event"]},

        # T2: TENTATIVE → CONFIRMED
        {"trigger": "confirm", "source": "tentative", "dest": "confirmed",
         "conditions": ["_check_confirmed_preconditions"],
         "before": ["_check_tier_protection"],
         "after": ["_log_transition", "_emit_transition_event"]},

        # T3: CONFIRMED → DEPRECATED
        {"trigger": "deprecate_from_confirmed", "source": "confirmed", "dest": "deprecated",
         "conditions": ["_check_deprecation_preconditions"],
         "before": ["_check_tier_protection"],
         "after": ["_log_transition", "_emit_transition_event"]},

        # T4: DEPRECATED → CONFIRMED
        {"trigger": "reconfirm", "source": "deprecated", "dest": "confirmed",
         "conditions": ["_check_confirmed_preconditions"],
         "before": ["_check_tier_protection"],
         "after": ["_log_transition", "_emit_transition_event"]},

        # T5: DEPRECATED → REFUTED
        {"trigger": "refute_from_deprecated", "source": "deprecated", "dest": "refuted",
         "conditions": ["_check_refutation_preconditions"],
         "before": ["_check_tier_protection"],
         "after": ["_log_transition", "_emit_transition_event"]},

        # T6: HYPOTHESIS → REFUTED
        {"trigger": "refute_from_hypothesis", "source": "hypothesis", "dest": "refuted",
         "conditions": ["_check_refutation_preconditions"],
         "before": ["_check_tier_protection"],
         "after": ["_log_transition", "_emit_transition_event"]},

        # T7: TENTATIVE → REFUTED
        {"trigger": "refute_from_tentative", "source": "tentative", "dest": "refuted",
         "conditions": ["_check_refutation_preconditions"],
         "before": ["_check_tier_protection"],
         "after": ["_log_transition", "_emit_transition_event"]},

        # T8: TENTATIVE → HYPOTHESIS (demotion)
        {"trigger": "demote_to_hypothesis", "source": "tentative", "dest": "hypothesis",
         "conditions": ["_check_demotion_preconditions"],
         "before": ["_check_tier_protection"],
         "after": ["_log_transition", "_emit_transition_event"]},

        # T9: CONFIRMED → TENTATIVE (partial disconfirmation)
        {"trigger": "demote_to_tentative", "source": "confirmed", "dest": "tentative",
         "conditions": ["_check_partial_disconfirmation_preconditions"],
         "before": ["_check_tier_protection"],
         "after": ["_log_transition", "_emit_transition_event"]},

        # T10: SIMULATED → HYPOTHESIS (promotion from quarantine)
        {"trigger": "promote_from_quarantine", "source": "simulated", "dest": "hypothesis",
         "conditions": ["_check_promotion_preconditions"],
         "after": ["_log_transition", "_emit_transition_event"]},
    ]

    def __init__(
        self,
        graph_store: GraphStore,
        invariant_checker: InvariantChecker,
        config: BSMConfig,
    ) -> None:
        self.graph = graph_store
        self.invariants = invariant_checker
        self.config = config
        self._transition_log: list[TransitionRecord] = []

    def transition(
        self,
        node: WorldNode,
        target_state: BeliefState,
        trigger: TransitionTrigger,
    ) -> TransitionResult:
        """Execute a belief state transition.

        This is the single entry point for ALL state changes.
        ★ Uses expected_version on GraphStore.update_node() to prevent
        lost updates under concurrent requests (GPT-5 observation 1).

        Returns TransitionResult with success/failure and audit data.
        """
        ...

    def reconcile_after_decay(
        self,
        decay_result: DecayResult,
    ) -> ReconciliationResult:
        """★ Bounded cascade reconciliation after confidence decay (GPT-5 Condition 1).

        Called by GraphStore.apply_confidence_decay() BEFORE returning.
        Evaluates all nodes that crossed below their state's minimum
        threshold and executes appropriate demotions.

        Cascade behavior (configurable via reconciliation_cascade_depth):
          D=0: Only directly affected nodes (those that crossed thresholds)
          D=1 (default): Affected nodes + their immediate dependents
          D=2+: Further dependent re-evaluation

        Nodes beyond depth D are enqueued via reconciliation.pending
        structured events for background processing in the next cycle.

        From CDD-01 §5.4: "This happens BEFORE returning — graph is
        never inconsistent."
        """
        ...

    def reconsider(                                                    # ★ GPT-5 Condition 2
        self,
        refuted_node: WorldNode,
        new_evidence: TransitionTrigger,
    ) -> ReconsiderationResult:
        """Create a new HYPOTHESIS node linked to a REFUTED node.

        REFUTED is terminal — no transitions FROM REFUTED are allowed.
        This operation creates a NEW node with:
          - belief_state = HYPOTHESIS
          - RECONSIDERS relationship → refuted_node
          - Copied metadata: source, provenance, falsifiability_criteria
          - Fresh confidence (from new evidence)

        Preserves lineage without enabling zombie beliefs.

        Returns ReconsiderationResult with new_node_id and audit record.
        """
        ...
```

### 4.2 Precondition Functions

Each precondition is a pure function that takes the node and trigger context, returning True/False. If False, the transition is rejected with a specific reason.

```python
class TransitionPreconditions:
    """All precondition checks for BSM transitions.

    Threshold values come from BSMConfig, which in turn loads from
    CDD-06 (Formal Semantics). Until CDD-06 is implemented, Phase 1
    defaults are used.
    """

    def check_tentative_preconditions(
        self, node: WorldNode, trigger: TransitionTrigger, config: BSMConfig
    ) -> PreconditionResult:
        """T1: HYPOTHESIS → TENTATIVE

        Requirements:
        1. confidence ≥ min_tentative_confidence
        2. context_diversity ≥ min_tentative_context_diversity
        3. At least one supporting observation exists
        """
        checks = [
            (node.confidence >= config.min_tentative_confidence,
             f"confidence {node.confidence:.4f} < threshold {config.min_tentative_confidence}"),
            (node.context_diversity >= config.min_tentative_context_diversity,
             f"context_diversity {node.context_diversity} < threshold {config.min_tentative_context_diversity}"),
            (trigger.supporting_evidence_count >= 1,
             "No supporting evidence provided"),
        ]
        return PreconditionResult.from_checks(checks)

    def check_confirmed_preconditions(
        self, node: WorldNode, trigger: TransitionTrigger, config: BSMConfig
    ) -> PreconditionResult:
        """T2: TENTATIVE → CONFIRMED (and T4: DEPRECATED → CONFIRMED)

        Requirements:
        1. confidence ≥ min_confirmed_confidence
        2. context_diversity ≥ min_confirmed_context_diversity
        3. falsifiability_criteria is non-empty
        4. adversarial_survival_count ≥ min_adversarial_survival
        5. Node source is NOT simulated
        """
        checks = [
            (node.confidence >= config.min_confirmed_confidence,
             f"confidence {node.confidence:.4f} < threshold {config.min_confirmed_confidence}"),
            (node.context_diversity >= config.min_confirmed_context_diversity,
             f"context_diversity {node.context_diversity} < threshold {config.min_confirmed_context_diversity}"),
            (len(node.falsifiability_criteria) > 0,
             "falsifiability_criteria is empty — CONFIRMED requires falsifiable claims"),
            (trigger.adversarial_survival_count >= config.min_adversarial_survival,
             f"adversarial_survival {trigger.adversarial_survival_count} < threshold {config.min_adversarial_survival}"),
            (node.source != NodeSource.SIMULATED,
             "SIMULATED nodes cannot be directly confirmed — must promote first"),
        ]
        return PreconditionResult.from_checks(checks)

    def check_refutation_preconditions(
        self, node: WorldNode, trigger: TransitionTrigger, config: BSMConfig
    ) -> PreconditionResult:
        """T5/T6/T7: Any active state → REFUTED

        Requirements:
        1. Contradicting observation provided
        2. Contradicting observation confidence > node confidence
        3. Contradicting observation is from independent source
        4. Node is NOT Tier 0 (axioms cannot be refuted)
        """
        checks = [
            (trigger.contradicting_observation is not None,
             "No contradicting observation provided"),
            (trigger.contradicting_observation is not None and
             trigger.contradicting_observation.confidence > node.confidence,
             f"Contradicting evidence confidence must exceed node confidence ({node.confidence:.4f})"),
            (trigger.is_independent_source,
             "Contradicting evidence must come from independent source"),
            (node.source != NodeSource.SEED_ONTOLOGY_T0,
             "Tier 0 axioms cannot be refuted — observation is suspect, not the axiom"),
        ]
        return PreconditionResult.from_checks(checks)

    def check_deprecation_preconditions(
        self, node: WorldNode, trigger: TransitionTrigger, config: BSMConfig
    ) -> PreconditionResult:
        """T3: CONFIRMED → DEPRECATED

        Requirements:
        1. confidence dropped below confirmed_decay_threshold
        2. Triggered by decay reconciliation (not arbitrary)
        """
        checks = [
            (node.confidence < config.confirmed_decay_threshold,
             f"confidence {node.confidence:.4f} ≥ threshold {config.confirmed_decay_threshold} — not yet deprecated"),
            (trigger.trigger_type in ("decay_reconciliation", "manual_review"),
             f"Deprecation requires decay_reconciliation or manual_review trigger, got {trigger.trigger_type}"),
        ]
        return PreconditionResult.from_checks(checks)

    def check_demotion_preconditions(
        self, node: WorldNode, trigger: TransitionTrigger, config: BSMConfig
    ) -> PreconditionResult:
        """T8: TENTATIVE → HYPOTHESIS

        Requirements:
        1. confidence dropped below min_tentative_confidence
        2. Triggered by decay reconciliation
        """
        checks = [
            (node.confidence < config.min_tentative_confidence,
             f"confidence {node.confidence:.4f} ≥ threshold {config.min_tentative_confidence} — still TENTATIVE"),
            (trigger.trigger_type in ("decay_reconciliation", "manual_review"),
             f"Demotion requires decay_reconciliation or manual_review trigger"),
        ]
        return PreconditionResult.from_checks(checks)

    def check_partial_disconfirmation_preconditions(
        self, node: WorldNode, trigger: TransitionTrigger, config: BSMConfig
    ) -> PreconditionResult:
        """T9: CONFIRMED → TENTATIVE

        Requirements:
        1. confidence dropped below min_confirmed_confidence
        2. confidence still above min_tentative_confidence (otherwise → DEPRECATED)
        3. Some disconfirming evidence exists but not outright contradiction
        """
        checks = [
            (node.confidence < config.min_confirmed_confidence,
             f"confidence still above confirmed threshold"),
            (node.confidence >= config.min_tentative_confidence,
             f"confidence below tentative threshold — should be DEPRECATED, not TENTATIVE"),
            (trigger.trigger_type in ("decay_reconciliation", "disconfirming_evidence", "manual_review"),
             f"Partial disconfirmation requires appropriate trigger"),
        ]
        return PreconditionResult.from_checks(checks)

    def check_promotion_preconditions(
        self, node: WorldNode, trigger: TransitionTrigger, config: BSMConfig
    ) -> PreconditionResult:
        """T10: SIMULATED → HYPOTHESIS (quarantine promotion)

        Requirements:
        1. Real-world corroboration OR sufficient independent confirmations
        2. No conflict with Tier 0 or Tier 1 seed ontology
        3. simulation_trust_score above minimum threshold
        4. Counterfactual error reduction demonstrated (GPT-5 constraint)
        5. ★ Promotion orchestration via CDD-05 (WAL + idempotent steps)

        Note: After promotion, source changes from 'simulated' to 'learned'
        and real_sim_weight resets to 1.0 (CDD-01 §6.1 promotion contract).
        """
        has_corroboration = trigger.has_real_world_corroboration
        has_confirmations = (trigger.independent_confirmation_count >=
                           config.min_independent_confirmations)
        checks = [
            (has_corroboration or has_confirmations,
             f"Requires real-world corroboration or ≥{config.min_independent_confirmations} independent confirmations"),
            (not trigger.conflicts_with_seed_ontology,
             "Node conflicts with Tier 0/1 seed ontology — flagged for human review, cannot auto-promote"),
            (node.simulation_trust_score >= config.min_promotion_trust_score,
             f"simulation_trust_score {node.simulation_trust_score:.4f} < threshold {config.min_promotion_trust_score}"),
            (trigger.reduces_prediction_error,
             "Counterfactual must reduce prediction error before promotion (GPT-5 constraint)"),
        ]
        return PreconditionResult.from_checks(checks)
```

### 4.3 Transition Data Structures

★ Changes from v0.1: Added reconsideration-related types, RECONSIDERS relationship.

```python
class TransitionTrigger(BaseModel):
    """Context for a requested BSM transition."""
    model_config = ConfigDict(extra="forbid")

    trigger_type: str = Field(
        ..., description="What caused this transition request.",
        pattern=r"^(observation|decay_reconciliation|manual_review|"
                r"disconfirming_evidence|adversarial_survival|promotion|"
                r"reconsideration)$",                                  # ★ added reconsideration
    )
    supporting_evidence_count: int = Field(default=0, ge=0)
    adversarial_survival_count: int = Field(default=0, ge=0)
    contradicting_observation: WorldNode | None = None
    is_independent_source: bool = False

    # Promotion-specific (T10)
    has_real_world_corroboration: bool = False
    independent_confirmation_count: int = Field(default=0, ge=0)
    conflicts_with_seed_ontology: bool = False
    reduces_prediction_error: bool = False

    # Tier protection
    hard_review_override: bool = False
    reviewer_id: str | None = None
    justification: str | None = None


class TransitionResult(BaseModel):
    """Outcome of a BSM transition attempt."""
    model_config = ConfigDict(extra="forbid")

    success: bool
    node_id: str
    previous_state: BeliefState
    new_state: BeliefState | None = None  # None if transition rejected
    rejection_reasons: list[str] = Field(default_factory=list)
    timestamp: datetime
    transition_id: str = Field(
        default_factory=lambda: str(uuid.uuid4()),
        description="Unique ID for this transition attempt (pass or fail).",
    )


class TransitionRecord(BaseModel):
    """Append-only audit record. One per transition attempt."""
    model_config = ConfigDict(extra="forbid")

    record_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    node_id: str
    previous_state: BeliefState
    requested_state: BeliefState
    actual_state: BeliefState  # May differ from requested if rejected
    success: bool
    trigger: TransitionTrigger
    precondition_results: list[str]  # Human-readable check results
    timestamp: datetime
    confidence_at_transition: float
    context_diversity_at_transition: int
    linked_node_id: str | None = None                                  # ★ for reconsideration linkage


class ReconciliationResult(BaseModel):
    """Outcome of post-decay BSM reconciliation."""
    model_config = ConfigDict(extra="forbid")

    nodes_evaluated: int
    transitions_executed: int
    transitions_rejected: int
    demotions: list[TransitionResult]  # Individual results
    cascade_depth_reached: int                                         # ★ GPT-5 Condition 1
    pending_deeper_evaluations: int                                    # ★ enqueued beyond D
    duration_ms: float


class ReconsiderationResult(BaseModel):                                # ★ GPT-5 Condition 2
    """Outcome of a reconsideration operation on a REFUTED node."""
    model_config = ConfigDict(extra="forbid")

    original_node_id: str
    new_node_id: str
    reconsiders_relationship_id: str
    new_belief_state: BeliefState  # Always HYPOTHESIS
    audit_record: TransitionRecord
```

---

## 5. Confidence Decay Integration

### 5.1 Reconciliation Protocol

★ Changes from v0.1: Bounded cascade with configurable depth D (GPT-5 Condition 1).

When `GraphStore.apply_confidence_decay()` is called, it modifies confidence values in batch. After the batch completes — but BEFORE returning — it invokes the BSM reconciliation protocol. This is the contract established in CDD-01 §5.4.

```
apply_confidence_decay(cutoff) called
    │
    ▼
Batch update: all eligible nodes' confidence decayed
    │
    ▼
★ BOUNDED CASCADE BSM RECONCILIATION (GPT-5 Condition 1)
    │
    ├── Pass 0: Query all nodes where confidence < state_minimum_threshold
    │     (using thresholds from BSMConfig)
    │
    ├── For each threshold-crossing node:
    │     ├── CONFIRMED + confidence < confirmed_decay_threshold
    │     │     → If confidence ≥ min_tentative_confidence: T9 (CONFIRMED → TENTATIVE)
    │     │     → If confidence < min_tentative_confidence: T3 (CONFIRMED → DEPRECATED)
    │     │
    │     ├── TENTATIVE + confidence < min_tentative_confidence
    │     │     → T8 (TENTATIVE → HYPOTHESIS)
    │     │
    │     └── HYPOTHESIS + confidence < hypothesis_review_flag_threshold
    │           → Stays HYPOTHESIS. Flag for review if < 0.05.
    │
    ├── Each transition goes through full BSM pipeline:
    │     preconditions checked, audit logged, tier protection enforced
    │
    ├── ★ Pass 1..D: Cascade dependent re-evaluation
    │     For each node demoted in previous pass:
    │       Query immediate dependents (via REQUIRES, ENABLES, CAUSES)
    │       If dependent's confidence preconditions are now violated:
    │         Execute appropriate demotion
    │     Repeat up to reconciliation_cascade_depth (default D=1)
    │
    ├── ★ Beyond depth D: Enqueue pending evaluations
    │     For nodes that would need re-evaluation beyond D:
    │       Emit reconciliation.pending structured event
    │       These are processed in the next decay cycle
    │
    ▼
Return ReconciliationResult
  (includes cascade_depth_reached, pending_deeper_evaluations)
```

### 5.2 Reconciliation Thresholds

These are Phase 1 defaults. Definitive values are specified in CDD-06 (Formal Semantics).

| Threshold | Default Value | Meaning |
|-----------|--------------|---------|
| `min_tentative_confidence` | 0.35 | Below this, TENTATIVE → HYPOTHESIS |
| `min_confirmed_confidence` | 0.65 | Below this, CONFIRMED → TENTATIVE (or DEPRECATED) |
| `confirmed_decay_threshold` | 0.40 | Below this, CONFIRMED → DEPRECATED (skipping TENTATIVE) |
| `min_tentative_context_diversity` | 2 | Minimum contexts for HYPOTHESIS → TENTATIVE |
| `min_confirmed_context_diversity` | 3 | Minimum contexts for TENTATIVE → CONFIRMED |
| `min_adversarial_survival` | 1 | Minimum adversarial challenges survived for CONFIRMED |
| `min_independent_confirmations` | 3 | Minimum independent confirmations for quarantine promotion |
| `min_promotion_trust_score` | 0.60 | Minimum simulation_trust_score for promotion eligibility |
| `hypothesis_review_flag_threshold` | 0.05 | Below this, flag for review (possible cleanup) |
| ★ `reconciliation_cascade_depth` | 1 | How many dependent-node passes per reconciliation (GPT-5) |

**Design rationale for the confirmed_decay_threshold vs min_confirmed_confidence gap:**

There is a deliberate gap between `min_confirmed_confidence` (0.65) and `confirmed_decay_threshold` (0.40). When a CONFIRMED node's confidence drops below 0.65, it first transitions to TENTATIVE (T9) — giving the system a chance to re-gather evidence. Only if confidence continues dropping below 0.40 does it reach DEPRECATED (T3). This two-stage demotion prevents abrupt loss of confirmed knowledge due to temporary evidence gaps.

---

## 6. ★ Context Diversity (GPT-5 Condition 7)

### 6.1 Context Signature Definition

Two observations of the same node count as different contexts if their context signatures differ. The context signature is a deterministic hash of orthogonal axes:

```python
def compute_context_signature(
    scenario_id: str,
    observer_source_id: str,
    domain_scope_id: str,
    time_window_bucket: str,
) -> str:
    """Compute a deterministic context signature.

    Two observations are different contexts if their signatures differ.
    Phase 1 uses 4 axes. CDD-06 may refine or extend.

    Args:
        scenario_id: Physics scenario or simulation ID (e.g., "freefall_air_resistance")
        observer_source_id: Which sensor/system observed it (e.g., "physics_sim_v1")
        domain_scope_id: Semantic domain (e.g., "physics", "kitchen")
        time_window_bucket: Temporal bucket to avoid duplicate counts from
            same episode. Phase 1 granularity: hourly (e.g., "2025-06-15T14")

    Returns:
        SHA-256 hex digest of the concatenated axes.
    """
    import hashlib
    raw = f"{scenario_id}|{observer_source_id}|{domain_scope_id}|{time_window_bucket}"
    return hashlib.sha256(raw.encode()).hexdigest()
```

### 6.2 Incrementing context_diversity

When a new observation references a WorldNode:
1. Compute the observation's context_signature
2. Check if this signature already exists in the node's observed_contexts set
3. If new: increment `context_diversity` by 1, store signature
4. If duplicate: no increment (same context, repeated observation)

```python
class ContextDiversityTracker:
    """Tracks unique context signatures per node.

    Phase 1: observed_contexts stored as a Neo4j array property on WorldNode.
    Phase 2+: may move to a separate index for performance.
    """

    def record_observation(
        self, node_id: str, context_signature: str
    ) -> bool:
        """Record an observation context. Returns True if context was new."""
        ...
```

**★ GPT-5 note (non-blocking):** `time_window_bucket` granularity must be documented clearly. Phase 1 default: hourly buckets (ISO 8601 truncated to hour, e.g., `"2025-06-15T14"`). This prevents the same one-hour observation episode from inflating context_diversity while allowing observations across different hours to count. CDD-06 may adjust granularity based on scenario analysis.

---

## 7. Invariant Checker

### 7.1 Architecture

The Invariant Checker runs on every graph mutation. It is called by the GraphStore (as specified in CDD-01 §8.2) before the mutation is committed.

```python
# src/aria/core/invariants.py

class InvariantChecker:
    """Runtime enforcement of Tier 0 logical axioms.

    Called on every graph write operation. Violations raise
    InvariantViolationError — a hard error that rejects the operation.

    Every violation also emits a structured alert event for observability.

    ★ Performance target: < 10ms per check at Phase 1 scale (<10K nodes).
    Emits invariant.check.duration_ms metric. Alert if p95 > 10ms.
    (GPT-5 Condition 5)
    """

    def __init__(self, graph_store: GraphStore, config: InvariantConfig) -> None:
        self.graph = graph_store
        self.config = config
        self._violation_count: int = 0

    def check_node_mutation(
        self, node: WorldNode, operation: str
    ) -> None:
        """Check invariants before a node create/update.

        Raises InvariantViolationError if any check fails.
        """
        self._check_tier0_immutability(node, operation)
        self._check_quarantine_boundary(node, operation)
        self._check_belief_confidence_coherence(node)

    def check_relationship_mutation(
        self, rel: CausalRelationship, operation: str
    ) -> None:
        """Check invariants before a relationship create/update/delete.

        Raises InvariantViolationError if any check fails.
        """
        self._check_non_contradiction(rel)
        self._check_temporal_ordering(rel)
        self._check_no_requires_self_loop(rel)
        self._check_tier0_relationship_protection(rel, operation)
        self._check_causal_asymmetry(rel)

    def check_post_transition(
        self, node: WorldNode, previous_state: BeliefState
    ) -> None:
        """Check invariants after a BSM transition completes.

        Catches any state that shouldn't exist (e.g., SIMULATED in WM).
        """
        self._check_quarantine_boundary(node, "post_transition")
        self._check_belief_confidence_coherence(node)
```

### 7.2 Invariant Definitions

Each invariant maps to a Tier 0 node from CDD-02.

#### INV-01: Non-Contradiction (`t0-non-contradiction`)

```
Rule: If a CONTRADICTS relationship exists between nodes A and B,
      A and B cannot BOTH be in CONFIRMED state within the same
      domain_scope.

Check trigger: create_relationship(CONTRADICTS), update_node(belief_state)

Implementation:
  On CONTRADICTS relationship creation:
    If source.belief_state == CONFIRMED and target.belief_state == CONFIRMED:
      If overlap(source.domain_scope, target.domain_scope):
        RAISE InvariantViolationError("non_contradiction",
          f"Nodes {source.node_id} and {target.node_id} are both CONFIRMED "
          f"with CONTRADICTS relationship in overlapping domain scope")

  On belief_state transition TO CONFIRMED:
    Query all CONTRADICTS neighbors of this node
    For each neighbor:
      If neighbor.belief_state == CONFIRMED:
        If overlap(node.domain_scope, neighbor.domain_scope):
          RAISE InvariantViolationError(...)

Resolution path (when invariant fires):
  The system does NOT auto-resolve contradictions. It:
  1. Rejects the operation that would create the violation
  2. Emits invariant.violation structured event
  3. Logs both nodes and their evidence for human review
  4. The Predictive Loop (CDD-04) can use this signal to
     prioritize investigation of the conflicting beliefs
```

#### INV-02: Temporal Ordering / DAG Enforcement (`t0-temporal-ordering`)

```
Rule: PRECEDES and CAUSES relationships must form a Directed Acyclic
      Graph. No temporal cycles.

Check trigger: create_relationship(PRECEDES), create_relationship(CAUSES)

Implementation:
  On PRECEDES or CAUSES relationship creation (A → B):
    Run cycle detection from B back to A using BFS/DFS
    (bounded by max_cycle_check_depth from config)
    If path from B back to A exists:
      RAISE InvariantViolationError("temporal_ordering",
        f"Adding {rel_type} from {A} to {B} would create a temporal cycle")

Performance note:
  Cycle detection is O(V+E) in the worst case but bounded by
  max_cycle_check_depth (default: 20 hops). For Phase 1 graph
  sizes (<10K nodes), this is sub-millisecond.
  ★ Short-circuit heuristic: skip BFS for nodes with degree < 3
  (low-degree nodes rarely participate in cycles). Phase 2 adds
  topological index / SCC cache. (GPT-5 Condition 5)
```

#### INV-03: Causal Asymmetry (`t0-causal-asymmetry`)

```
Rule: If A CAUSES B exists, B CAUSES A cannot exist in the same
      temporal context. Feedback loops require temporal separation.

Check trigger: create_relationship(CAUSES)

Implementation:
  On CAUSES relationship creation (A → B):
    Query: does B CAUSES A exist (not soft-deleted)?
    If yes AND both are in same temporal window:
      RAISE InvariantViolationError("causal_asymmetry",
        f"A CAUSES B and B CAUSES A cannot coexist in same temporal context. "
        f"Model feedback as temporal sequence instead.")

  Note: B CAUSES A is allowed if it references a different temporal
  context (different event timestamps). The check compares
  created_at timestamps within a configurable window.
```

#### INV-04: Tier 0 Immutability (`t0-*` nodes)

```
Rule: Nodes with source = seed-ontology-t0 cannot be modified,
      deleted, or have their outgoing relationships changed.

Check trigger: update_node, soft_delete_node, delete_relationship,
               update_relationship (when source is T0)

Implementation: Already enforced by GraphStore (CDD-01) and Tier
  Protection (CDD-02). The Invariant Checker is the secondary
  enforcement layer — defense in depth.

  On any mutation targeting a Tier 0 node:
    RAISE InvariantViolationError("tier0_immutability",
      f"Cannot modify Tier 0 node {node.node_id}")
```

#### INV-05: Quarantine Boundary

```
Rule: Nodes with belief_state = SIMULATED cannot exist in the
      World Model GraphStore. They can ONLY exist in the
      Quarantine GraphStore.

Check trigger: create_node, update_node (in World Model store)

Implementation:
  On any node creation or update in World Model store:
    If node.belief_state == SIMULATED:
      RAISE InvariantViolationError("quarantine_boundary",
        f"SIMULATED node {node.node_id} cannot exist in World Model")
    If node.source == NodeSource.SIMULATED and node.belief_state != HYPOTHESIS:
      RAISE InvariantViolationError("quarantine_boundary",
        f"Promoted node must have belief_state=HYPOTHESIS, not {node.belief_state}")
```

#### INV-06: No REQUIRES Self-Loop (Grok, CDD-01)

```
Rule: A node cannot REQUIRE itself. "B requires B" is logically vacuous.

Check trigger: create_relationship(REQUIRES)

Implementation:
  If rel.source_node_id == rel.target_node_id:
    RAISE InvariantViolationError("no_self_loop",
      f"REQUIRES self-loop on {rel.source_node_id} is logically vacuous")
```

#### INV-07: Belief-Confidence Coherence

★ Changes from v0.1: Adjusted confirmed_hard_minimum to 0.20 (GPT-5 Condition 3).

```
Rule: A node's confidence should be broadly consistent with its
      belief_state. This is a soft invariant — it warns rather
      than blocks, except in extreme cases.

Check trigger: update_node, post_transition

★ Hard violations (BLOCK):
  - CONFIRMED with confidence < 0.20 (★ raised from 0.10 — GPT-5 Condition 3)
  - REFUTED with confidence > 0.90 (clearly inconsistent)
  - HYPOTHESIS with confidence = 1.0 (unvalidated belief at max certainty)

Soft violations (WARN + structured event, do not block):
  - CONFIRMED with confidence < min_confirmed_confidence
    (should trigger decay reconciliation)
  - TENTATIVE with confidence > 0.95
    (should probably be promoted to CONFIRMED)

★ All thresholds configurable via BSMConfig:
  confirmed_hard_minimum (default 0.20)
  refuted_hard_maximum (default 0.90)
```

### 7.3 Invariant Registry

All invariants are registered in a central registry that the checker iterates through. This makes invariants extensible — Phase 2 adds new invariants without modifying the checker's core logic.

```python
class InvariantRegistry:
    """Registry of all active invariants.

    Phase 1 registers INV-01 through INV-07.
    Phase 2+ can add invariants dynamically.
    """

    def __init__(self) -> None:
        self._node_invariants: list[NodeInvariant] = []
        self._relationship_invariants: list[RelationshipInvariant] = []

    def register_node_invariant(self, invariant: NodeInvariant) -> None: ...
    def register_relationship_invariant(self, invariant: RelationshipInvariant) -> None: ...

    def check_all_node_invariants(
        self, node: WorldNode, operation: str
    ) -> list[InvariantViolation]: ...

    def check_all_relationship_invariants(
        self, rel: CausalRelationship, operation: str
    ) -> list[InvariantViolation]: ...


class NodeInvariant(Protocol):
    """Interface for a single node-level invariant check."""
    invariant_id: str
    tier0_node_id: str  # Which Tier 0 axiom this enforces
    severity: str  # "hard" (block) or "soft" (warn)

    def check(self, node: WorldNode, operation: str) -> InvariantViolation | None: ...


class RelationshipInvariant(Protocol):
    """Interface for a single relationship-level invariant check."""
    invariant_id: str
    tier0_node_id: str
    severity: str

    def check(self, rel: CausalRelationship, operation: str) -> InvariantViolation | None: ...


class InvariantViolation(BaseModel):
    """Record of a detected invariant violation."""
    invariant_id: str
    tier0_node_id: str
    severity: str
    message: str
    involved_nodes: list[str]
    operation: str
    timestamp: datetime
```

---

## 8. ★ Reconsideration Operation (GPT-5 Condition 2)

### 8.1 Mechanics

REFUTED is terminal — no BSM transitions FROM REFUTED are permitted. If new evidence later suggests a refuted belief may have been wrongly dismissed, the system uses the `reconsider()` operation:

```
reconsider(refuted_node, new_evidence) called
    │
    ├── Verify: refuted_node.belief_state == REFUTED
    │     If not → ReconsiderationError("Can only reconsider REFUTED nodes")
    │
    ├── Create new WorldNode:
    │     - Fresh node_id (UUID)
    │     - belief_state = HYPOTHESIS
    │     - confidence from new_evidence
    │     - Copy: falsifiability_criteria, domain_scope, source (reset to LEARNED)
    │     - context_diversity = 1 (starts fresh)
    │
    ├── Create RECONSIDERS relationship:
    │     new_node -[RECONSIDERS]-> refuted_node
    │     Carries: new_evidence summary, reconsideration_reason
    │
    ├── Log special TransitionRecord:
    │     - trigger_type = "reconsideration"
    │     - linked_node_id = refuted_node.node_id
    │     - Both node IDs captured for audit
    │
    ▼
Return ReconsiderationResult(original_node_id, new_node_id, relationship_id)
```

### 8.2 RECONSIDERS Relationship Type

Added to the RelationshipType enum (CDD-01 extension):

```python
class RelationshipType(str, Enum):
    # ... existing 12 types ...
    RECONSIDERS = "RECONSIDERS"  # ★ Links reconsidered HYPOTHESIS to original REFUTED
```

Properties: not a causal relationship — purely structural/epistemic linkage. No strength, no confidence decay. Carries `reconsideration_reason` and `new_evidence_summary` as context.

### 8.3 Why Not Just Un-REFUTE?

1. **Zombie prevention:** If REFUTED nodes could transition back, adversarial inputs could repeatedly resurrect discredited beliefs, creating oscillatory instability.
2. **Audit clarity:** The REFUTED node's history is preserved intact. The new node has its own clean trajectory.
3. **Version lineage:** The RECONSIDERS relationship explicitly links the two, so the full epistemic history is traceable.

---

## 9. Transition Audit System

### 9.1 Audit Requirements

Every transition attempt (successful or rejected) produces an append-only `TransitionRecord` (§4.3). These records serve three purposes:

1. **Debugging**: Trace why a node is in its current state
2. **Calibration**: Compare predicted confidence trajectories against actual (CDD-10 DriftMonitor)
3. **Governance**: Prove that tier protection rules were enforced

★ Audit write is atomic with the transition: Neo4j transaction encompasses both node state change and TransitionAudit creation. If audit write fails, the entire transition rolls back. (GPT-5 additional observation 2)

### 9.2 Audit Storage

Phase 1: TransitionRecords are stored as Neo4j nodes (label: `TransitionAudit`) with relationships to the WorldNode they reference. This keeps the audit trail in the same graph as the data it governs.

```
(WorldNode)-[:HAS_TRANSITION]->(TransitionAudit)
```

Properties on TransitionAudit map directly to TransitionRecord fields.

### ★ 9.3 Audit Archival Plan (GPT-5 Condition 4)

Phase 1 graph size is small — all TransitionRecords stay in Neo4j. The schema is designed from day one to support future archival:

**Phase 2 Archival Pipeline (documented, not implemented):**

1. **Retention window:** Records older than 90 days (configurable) eligible for archival
2. **Export:** Periodic batch export to append-only cold store (S3/Parquet or Redis Stream → S3)
3. **Replace:** Archived Neo4j TransitionAudit nodes replaced with lightweight `ArchivedTransitionPointer` nodes containing only: `archive_batch_id`, `archive_timestamp`, `original_record_id`
4. **Reconstruct:** Tooling to retrieve and re-hydrate archived records from cold store back into Neo4j for forensic replay

**Schema forward-compatibility:** All TransitionRecord fields are self-contained (no foreign-key-only references). Every record can be serialized, exported, and reconstructed independently.

### 9.4 Audit Querying

```python
class TransitionAuditStore:
    """Query interface for BSM transition history."""

    def get_node_transitions(
        self, node_id: str, limit: int = 50
    ) -> list[TransitionRecord]:
        """All transitions for a specific node, newest first."""
        ...

    def get_transitions_by_type(
        self, from_state: BeliefState, to_state: BeliefState,
        since: datetime | None = None, limit: int = 100
    ) -> list[TransitionRecord]:
        """All transitions of a specific type across the graph."""
        ...

    def get_rejected_transitions(
        self, since: datetime | None = None, limit: int = 100
    ) -> list[TransitionRecord]:
        """All rejected transition attempts. Useful for diagnosing
        precondition failures and potential stuck states."""
        ...

    def count_transitions_by_state(self) -> dict[str, int]:
        """Aggregate: how many transitions to each state. Feed for
        DriftMonitor (CDD-10) calibration metrics."""
        ...

    def get_reconsideration_chain(                                     # ★ GPT-5 Condition 2
        self, node_id: str
    ) -> list[TransitionRecord]:
        """Trace the full reconsideration lineage for a node.
        Follows RECONSIDERS edges back through refuted ancestors."""
        ...
```

---

## 10. Configuration

★ Changes from v0.1: Added cascade depth, soft invariant thresholds, perf target configs.

```python
class BSMConfig(BaseSettings):
    """Configuration for the Belief State Machine.

    Phase 1 defaults provided. CDD-06 (Formal Semantics) defines
    the definitive threshold values.
    """

    # Transition thresholds
    min_tentative_confidence: float = 0.35
    min_confirmed_confidence: float = 0.65
    confirmed_decay_threshold: float = 0.40
    min_tentative_context_diversity: int = 2
    min_confirmed_context_diversity: int = 3
    min_adversarial_survival: int = 1
    hypothesis_review_flag_threshold: float = 0.05

    # Promotion thresholds
    min_independent_confirmations: int = 3
    min_promotion_trust_score: float = 0.60

    # ★ Reconciliation (GPT-5 Condition 1)
    reconciliation_cascade_depth: int = 1       # D=1: affected + immediate dependents
    reconciliation_cascade_max: int = 5         # Safety cap even if configured higher

    # ★ Soft invariant thresholds (GPT-5 Condition 3)
    confirmed_hard_minimum: float = 0.20        # CONFIRMED < this → hard block
    refuted_hard_maximum: float = 0.90          # REFUTED > this → hard block

    # ★ Context diversity (GPT-5 Condition 7)
    context_time_window_granularity: str = "hour"  # ISO truncation level

    # Audit
    audit_all_attempts: bool = True  # Log rejections too
    audit_storage: str = "neo4j"     # Phase 1: same graph
    audit_retention_days: int = 90   # ★ Phase 2 archival window (GPT-5 Condition 4)

    model_config = SettingsConfigDict(env_prefix="ARIA_BSM_")


class InvariantConfig(BaseSettings):
    """Configuration for the Invariant Checker."""

    # Cycle detection
    max_cycle_check_depth: int = 20
    causal_asymmetry_temporal_window_seconds: float = 0.001

    # Soft invariant behavior
    soft_violation_log_level: str = "warning"
    hard_violation_emit_alert: bool = True

    # ★ Performance (GPT-5 Condition 5)
    perf_target_ms: float = 10.0                # Soft target per check
    perf_alert_p95_ms: float = 10.0             # Alert threshold
    emit_check_duration_metric: bool = True     # invariant.check.duration_ms

    model_config = SettingsConfigDict(env_prefix="ARIA_INV_")
```

---

## 11. ★ Monitoring Metrics (GPT-5 Additional Observation 3)

The BSM and Invariant Checker emit the following structured metrics for CDD-11 (Observability):

| Metric | Type | Source |
|--------|------|--------|
| `bsm.transitions_per_second` | Counter/rate | BSM transition() |
| `bsm.rejected_rate` | Counter/rate | BSM transition() on rejection |
| `bsm.reconciliation_latency_ms` | Histogram | reconcile_after_decay() |
| `bsm.reconciliation_cascade_depth` | Gauge | reconcile_after_decay() |
| `bsm.reconciliation_pending_count` | Gauge | Nodes enqueued beyond D |
| `invariant.violation_rate` | Counter/rate | InvariantChecker on violation |
| `invariant.check.duration_ms` | Histogram | InvariantChecker per check |
| `bsm.state_distribution` | Gauge per state | Periodic snapshot |
| `bsm.reconsideration_count` | Counter | reconsider() calls |

---

## 12. Integration Points

| Component | What It Uses From CDD-03 | How |
|-----------|-------------------------|-----|
| **CDD-01: GraphStore** | InvariantChecker.check_node_mutation, check_relationship_mutation | Called on every write (§8.2 of CDD-01) |
| **CDD-01: Decay** | BSM.reconcile_after_decay | Synchronous call after confidence batch update |
| **CDD-02: Tier Protection** | BSM tier-specific overrides (§3.4) | BSM enforces tier protection on transitions |
| **CDD-04: Predictive Loop** | BSM.transition (observation triggers) | Loop calls BSM when observations change confidence |
| **CDD-05: Quarantine** | BSM.transition (T10 promotion), ★ promotion atomicity is CDD-05 scope | ★ BSM provides transition; CDD-05 provides orchestration (WAL + idempotent steps). (GPT-5 Condition 6) |
| **CDD-06: Formal Semantics** | BSMConfig threshold values, ★ context signature refinement | CDD-06 defines the numbers and may extend context axes, BSM enforces them |
| **CDD-07: Adversary Simulator** | TransitionTrigger.adversarial_survival_count | Adversary results feed CONFIRMED preconditions |
| **CDD-10: DriftMonitor** | TransitionAuditStore.count_transitions_by_state, ★ metrics | Monitors transition patterns for drift detection |
| **CDD-11: Observability** | ★ All metrics from §11, transition events, invariant violation events | Dashboard metrics |

---

## 13. Error Handling & Edge Cases

| Edge Case | Behavior | Rationale |
|-----------|----------|-----------|
| Transition requested on non-existent node | NodeNotFoundError | Fail fast — don't create phantom transitions |
| Transition requested on soft-deleted node | Reject with reason "node is soft-deleted" | Dead nodes don't transition |
| Multiple transitions requested simultaneously | Sequential execution (Python GIL + Neo4j transaction isolation) | Phase 1 doesn't need concurrent BSM — single event loop |
| Decay reconciliation finds Tier 0 node below threshold | Skip — Tier 0 decay_rate is 0.0, should never happen | Defense in depth: even if bug sets T0 confidence low, skip it |
| Promotion attempted for node conflicting with Tier 0 | Reject + flag for human review | Never auto-promote nodes that contradict axioms |
| ★ Invariant violation during cascade reconciliation | Cascade halts at violation depth, partial results returned, alert emitted | Partial cascade is safer than corrupted state (GPT-5) |
| ★ BSM transition succeeds but audit write fails | Transaction rolls back entire transition | Audit is non-optional — same Neo4j transaction (GPT-5 obs 2) |
| REFUTED node receives supporting evidence | ★ Use reconsider() to create new linked node (GPT-5 Condition 2) | Prevents zombie belief resurrection while preserving lineage |
| CONFIRMED Tier 1 node — confidence stable for N cycles | Flag for revalidation (CDD-02 §9.2) | Epistemic fossilization prevention (GPT-5, from CDD-02 review) |
| ★ Cascade reconciliation exceeds depth D | Enqueue remaining via reconciliation.pending event | Prevents O(n²) stalls (GPT-5 Condition 1) |
| ★ concurrent expected_version mismatch during transition | ConcurrentWriteError, transition retried by caller | Optimistic concurrency (GPT-5 obs 1, CDD-01) |

---

## 14. Test Plan

### 14.1 Unit Tests (`tests/unit/test_belief_state.py`)

| Test | Validates |
|------|-----------|
| `test_all_valid_transitions` | T1–T10 succeed when preconditions met |
| `test_hypothesis_to_tentative_requires_confidence` | T1 fails below min_tentative_confidence |
| `test_hypothesis_to_tentative_requires_context_diversity` | T1 fails below min_tentative_context_diversity |
| `test_confirmed_requires_falsifiability` | T2 fails with empty falsifiability_criteria |
| `test_confirmed_requires_adversarial_survival` | T2 fails below min_adversarial_survival |
| `test_refuted_is_terminal` | No transition from REFUTED succeeds |
| `test_simulated_cannot_skip_hypothesis` | SIMULATED → TENTATIVE/CONFIRMED rejected |
| `test_cannot_enter_simulated` | Any → SIMULATED rejected |
| `test_hypothesis_cannot_skip_to_confirmed` | HYPOTHESIS → CONFIRMED rejected |
| `test_hypothesis_cannot_deprecate` | HYPOTHESIS → DEPRECATED rejected |
| `test_tier0_blocks_all_transitions` | Any transition on T0 node raises ImmutableNodeError |
| `test_tier1_requires_hard_review` | Transition on T1 without override rejected |
| `test_tier1_with_hard_review_succeeds` | Transition on T1 with override succeeds and logs |
| `test_tier2_normal_transitions` | T2 nodes follow standard BSM rules |
| `test_decay_reconciliation_confirmed_to_deprecated` | Confidence drop triggers T3 |
| `test_decay_reconciliation_confirmed_to_tentative` | Partial confidence drop triggers T9 |
| `test_decay_reconciliation_tentative_to_hypothesis` | Confidence drop triggers T8 |
| `test_transition_audit_recorded` | Every transition attempt produces TransitionRecord |
| `test_rejected_transition_audit_recorded` | Failed transitions also audited |
| `test_refutation_requires_higher_confidence` | Contradicting evidence must exceed node confidence |
| `test_refutation_requires_independent_source` | Same-source contradiction rejected |
| `test_tier0_cannot_be_refuted` | Refutation of T0 node rejected with clear message |
| ★ `test_reconsideration_creates_new_node` | reconsider() creates HYPOTHESIS with RECONSIDERS edge |
| ★ `test_reconsideration_only_on_refuted` | reconsider() rejects non-REFUTED nodes |
| ★ `test_reconsideration_preserves_lineage` | New node links to refuted node with metadata |
| ★ `test_context_signature_uniqueness` | Different axes produce different signatures |
| ★ `test_context_diversity_increment` | New context increments, duplicate does not |

### 14.2 Unit Tests (`tests/unit/test_invariants.py`)

| Test | Validates |
|------|-----------|
| `test_non_contradiction_both_confirmed` | CONTRADICTS between two CONFIRMED nodes raises error |
| `test_non_contradiction_one_not_confirmed` | CONTRADICTS with non-CONFIRMED node allowed |
| `test_non_contradiction_different_domains` | CONTRADICTS in non-overlapping domain_scope allowed |
| `test_temporal_cycle_detected` | A→B→C→A via PRECEDES raises error |
| `test_causal_cycle_detected` | A CAUSES B, B CAUSES A (same temporal context) raises error |
| `test_causal_cycle_temporal_separation_allowed` | A CAUSES B (t1), B CAUSES A (t2, t2 > t1) allowed |
| `test_tier0_immutability_on_update` | Update to T0 node raises InvariantViolationError |
| `test_tier0_immutability_on_delete` | Delete of T0 node raises InvariantViolationError |
| `test_quarantine_boundary_create` | SIMULATED node in WM store raises error |
| `test_quarantine_boundary_post_promotion` | Promoted node must be HYPOTHESIS not SIMULATED |
| `test_requires_self_loop_blocked` | A REQUIRES A raises error |
| ★ `test_belief_confidence_hard_violation_confirmed` | CONFIRMED at 0.15 raises hard error (< 0.20 threshold) |
| ★ `test_belief_confidence_hard_violation_refuted` | REFUTED at 0.95 raises hard error (> 0.90 threshold) |
| `test_belief_confidence_soft_violation` | CONFIRMED at 0.55 emits warning, does not block |
| `test_invariant_registry_extensible` | New invariant registered and checked |
| ★ `test_invariant_check_emits_duration_metric` | invariant.check.duration_ms metric present |

### 14.3 Property-Based Tests (`tests/property/test_bsm_properties.py`)

| Test | Validates |
|------|-----------|
| `test_random_transition_sequences_no_crash` | Hypothesis generates 1000+ random valid/invalid transition sequences — no crashes, no invalid final states |
| `test_refuted_is_always_terminal` | From any random sequence of transitions, once REFUTED, no further transition succeeds |
| `test_simulated_always_exits_to_hypothesis` | SIMULATED can only become HYPOTHESIS, never anything else |
| `test_tier0_never_transitions` | Random transitions on T0 nodes always rejected |
| `test_confidence_always_bounded` | After any sequence of transitions and decays, confidence stays in [0.0, 1.0] |
| ★ `test_cascade_reconciliation_bounded` | Cascade never exceeds configured depth D |
| ★ `test_cascade_convergence` | Background reconciliation events eventually resolve all pending nodes |

### 14.4 Integration Tests (`tests/integration/test_bsm_integration.py`)

| Test | Validates |
|------|-----------|
| `test_full_lifecycle_hypothesis_to_confirmed` | H → T → C with real Neo4j |
| `test_full_lifecycle_confirmed_to_refuted` | C → D → R with real Neo4j |
| ★ `test_full_lifecycle_refuted_reconsidered` | R → reconsider() → new H with RECONSIDERS edge |
| `test_decay_reconciliation_with_live_graph` | Decay batch + reconciliation on 100 nodes |
| ★ `test_cascade_reconciliation_depth_1` | Demotion propagates to immediate dependents |
| ★ `test_cascade_reconciliation_depth_0` | No propagation beyond directly affected nodes |
| `test_audit_trail_complete` | Full lifecycle produces correct TransitionRecord chain |
| `test_invariant_checker_on_live_graph` | Contradiction detection with real Neo4j |
| `test_cycle_detection_on_deep_graph` | DAG check on graph with 1000 nodes, 5000 edges |
| `test_seed_ontology_bsm_consistency` | All seed nodes from CDD-02 are in correct states |
| `test_concurrent_transitions_sequential` | 50 rapid transitions execute sequentially, no data corruption |
| ★ `test_audit_transaction_atomicity` | Audit write failure rolls back entire transition |

### ★ 14.5 Stress Tests (`tests/stress/test_bsm_stress.py`) (GPT-5 Condition 5)

| Test | Validates |
|------|-----------|
| `test_invariant_check_1k_nodes` | INV-01 through INV-07 all < 10ms on 1K node graph |
| `test_invariant_check_5k_nodes` | INV-02 cycle detection < 10ms on 5K node graph |
| `test_invariant_check_10k_nodes` | INV-02 cycle detection < 10ms on 10K node graph |
| `test_reconciliation_100_demotions` | Reconciliation of 100 simultaneous threshold crossings |
| `test_cascade_depth_2_performance` | D=2 cascade completes within acceptable latency |

---

## 15. Acceptance Criteria

★ Expanded from 22 to 30 criteria.

| # | Criterion | Sprint | Verification |
|---|-----------|--------|-------------|
| 1 | All 6 belief states defined | S1 | BeliefState enum with all 6 values |
| 2 | All 10 valid transitions implemented | S1 | T1–T10 succeed when preconditions met |
| 3 | All invalid transitions explicitly blocked | S1 | Matrix of blocked transitions verified |
| 4 | Preconditions enforced for every transition | S1 | Each transition has ≥1 precondition test |
| 5 | CONFIRMED requires falsifiability_criteria | S1 | Empty criteria → rejection |
| 6 | CONFIRMED requires adversarial_survival | S1 | Zero survival → rejection |
| 7 | SIMULATED restricted to quarantine | S1 | SIMULATED in WM → InvariantViolationError |
| 8 | REFUTED is terminal | S1 | No transitions from REFUTED succeed |
| 9 | Tier 0 nodes never transition | S1 | All BSM requests on T0 nodes rejected |
| 10 | Tier 1 transitions require hard_review_override | S1 | Without override → rejected with log |
| 11 | Transition audit logging for all attempts | S1 | Pass and fail both produce TransitionRecord |
| 12 | Non-contradiction invariant enforced (INV-01) | S1 | Both CONFIRMED + CONTRADICTS → InvariantViolationError |
| 13 | Temporal ordering / DAG invariant enforced (INV-02) | S1 | Cycle in PRECEDES/CAUSES → InvariantViolationError |
| 14 | Causal asymmetry invariant enforced (INV-03) | S1 | A→B and B→A (same context) → InvariantViolationError |
| 15 | REQUIRES self-loop invariant enforced (INV-06) | S1 | A REQUIRES A → InvariantViolationError |
| 16 | Quarantine boundary invariant enforced (INV-05) | S1 | SIMULATED in WM store → InvariantViolationError |
| 17 | Confidence decay reconciliation synchronous | S1 | Decay + reconciliation in same call |
| 18 | Two-stage confirmed demotion works | S1 | CONFIRMED → TENTATIVE → HYPOTHESIS via progressive decay |
| 19 | Invariant violations emit structured alerts | S1 | invariant.violation event observed in logs |
| 20 | Property-based tests: 1000+ random sequences | S1 | No crashes, no invalid terminal states |
| 21 | Seed ontology + BSM consistency verified | S2 | All T0 CONFIRMED, T1 CONFIRMED, T2 TENTATIVE |
| 22 | Invariant registry extensible | S1 | New invariant can be registered without modifying checker core |
| ★ 23 | Bounded cascade reconciliation (D=1 default) | S1 | Cascade re-evaluates dependents, bounded by D |
| ★ 24 | Cascade enqueues beyond-D nodes | S1 | reconciliation.pending events emitted |
| ★ 25 | Reconsideration operation creates linked HYPOTHESIS | S1 | New node + RECONSIDERS edge + audit record |
| ★ 26 | INV-07 hard thresholds configurable (confirmed_hard_minimum=0.20) | S1 | Config overrides work, defaults correct |
| ★ 27 | Context signature computation deterministic | S1 | Same inputs → same hash |
| ★ 28 | Context diversity increments on new contexts only | S1 | Duplicate signatures don't inflate count |
| ★ 29 | BSM uses expected_version on all writes | S1 | ConcurrentWriteError on version mismatch |
| ★ 30 | Invariant check performance < 10ms at Phase 1 scale | S1 | Stress tests pass on 10K node graphs |

---

## 16. Open Questions — All Resolved

All 7 original open questions have been answered through the multi-LLM review process.

| Q | Resolution | Decided By |
|---|-----------|------------|
| Q1: Reconciliation cascade depth | Bounded cascade D=1 default, enqueue beyond D | GPT-5 (proposed) |
| Q2: REFUTED terminal permanence | Keep terminal + reconsider() operation with RECONSIDERS edge | GPT-5 (proposed) |
| Q3: Soft invariant thresholds | confirmed_hard_minimum raised to 0.20, configurable | GPT-5 (proposed) |
| Q4: Audit storage location | Neo4j Phase 1, documented archival pipeline Phase 2 | GPT-5 (proposed) |
| Q5: Invariant checker perf budget | < 10ms soft target, p95 alert, stress tests | GPT-5 (proposed) |
| Q6: Promotion atomicity | BSM provides transition, CDD-05 provides orchestration (WAL) | GPT-5 (proposed, scope boundary) |
| Q7: Context diversity definition | 4-axis context signature, hourly time buckets, CDD-06 refinement | GPT-5 (proposed, simplified by Claude) |

---

## Appendix A: Review Feedback Incorporation Record

### GPT-5 (OpenAI) — 7 Conditions

| # | Condition | Section |
|---|-----------|---------|
| 1 | Bounded cascade reconciliation (D=1) | §5.1 |
| 2 | Reconsideration operation (RECONSIDERS) | §8 |
| 3 | Soft invariant threshold adjustment (0.20) | §7.2 INV-07 |
| 4 | Hybrid audit archival plan | §9.3 |
| 5 | Invariant checker perf target + instrumentation | §7.1, §10, §14.5 |
| 6 | Promotion orchestration scoped to CDD-05 | §3.4, §12 |
| 7 | Context diversity signature definition | §6 |

### GPT-5 — Additional Observations

| Observation | Section |
|-------------|---------|
| expected_version on BSM writes | §4.1 |
| Audit transaction atomicity | §9.1 |
| Monitoring metrics | §11 |
| Property tests for cascade | §14.3 |
| Promotion stress tests → CDD-05 | §12 (cross-ref) |
| Threshold configurability | §10 |

### Gemini (Google)

Clean approval, no conditions.

### Grok (xAI)

Clean approval, no conditions.

---

*End of CDD-03 v1.0: Belief State Machine & Invariant Checker*

**Project ARIA · Adaptive Reasoning Integrated Architecture**  
Component Design Document 03 — Epistemic Governance Engine  
Approved by All Reviewing Systems — Ready for Implementation