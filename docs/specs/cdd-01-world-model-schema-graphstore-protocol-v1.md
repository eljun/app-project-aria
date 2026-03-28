# CDD-01: World Model Schema & GraphStore Protocol

> **Status:** APPROVED — All Reviewing Systems Signed Off  
> **Version:** 1.0 (Incorporates Multi-LLM Review Feedback)  
> **Author:** Jun (Architect) + Claude Opus 4.6 (Design Partner)  
> **Depends On:** Project ARIA Foundational Specification v2.1 Final, Phase 1 Implementation Plan v1.0  
> **Sprint:** S0 (Protocol) + S1 (Schema & Graph Operations)  
> **Classification:** Confidential — Core Team & Designated Review Partners  
> **Review Contributors:** GPT-5 (OpenAI) · Gemini (Google) · Grok (xAI) · Claude Opus 4.6

---

## Review & Approval Record

| System | Status | Conditions | Resolution |
|--------|--------|------------|------------|
| GPT-5 (OpenAI) | ✅ CONDITIONALLY APPROVED → APPROVED | 5 conditions raised | All 5 incorporated into v1.0 |
| Gemini (Google) | ✅ APPROVED | No conditions | Clean sign-off |
| Grok (xAI) | ✅ APPROVED | No conditions | Suggestions incorporated |
| Claude Opus 4.6 | ✅ 4th REVIEWER | 4 observations raised | All 4 confirmed and incorporated |

**Consensus:** 4/4 approved. One contention (version history storage) resolved via diff-based compromise — accepted by all parties. 3 additional hidden risks identified by GPT-5 in final round incorporated. All changes from v0.1 marked with ★.

---

## 1. Purpose & Scope

CDD-01 is the foundation of Project ARIA. Every other CDD imports from this document's data structures. Every component reads or writes through this document's interfaces.

This CDD specifies three things:

**1.1 The GraphStore Protocol** — the abstract interface through which all components access the causal graph. Defined as a Python `Protocol` (PEP 544), enabling future backend swaps without changing consuming code. Built in Sprint 0.

**1.2 The World Model Schema** — Pydantic models for all 20 node properties and all 12 causal relationship types defined in v2.1 Section 4. ★ Plus the `WorldNodeVersion` diff model for version history, the `CausalSubgraph` query return type, and partial update models. Built in Sprint 1.

**1.3 The Neo4j GraphStore Implementation** — the concrete implementation of the GraphStore Protocol for Neo4j 5.x Community Edition. Handles CRUD operations, graph queries, subgraph extraction, soft-delete semantics, version management, ★ optimistic concurrency, and ★ post-decay BSM reconciliation. Built in Sprint 1.

### 1.1 What This CDD Does NOT Cover

| Excluded Component | Covered In |
|-------------------|------------|
| Belief State Machine (transitions, preconditions, audit) | CDD-03 |
| Seed Ontology (content, loader, tier protection) | CDD-02 |
| Quarantine graph operations + ★ promotion atomicity | CDD-05 |
| Causal subgraph extraction with complexity budgets | CDD-08 |
| Formal semantics (confidence thresholds, calibration) | CDD-06 |

---

## 2. Specification References

| Spec Section | Content | Relevance |
|-------------|---------|-----------|
| v2.1 §4.1 | Final Node Schema Properties (20 properties) | Direct implementation target |
| v2.1 §4.2 | Complete Causal Relationship Vocabulary (12 types) | Direct implementation target |
| v2.1 §4.3 | Ontological Commitment — External Truth Pragmatism | Confidence and belief_state semantics |
| v2.1 §3.1 | Seed Ontology — 3-Tier Hybrid | Source enum values, immutability semantics |
| v2.1 §3.3 | Simulation Quarantine Layer | SIMULATED belief_state, simulation_trust_score semantics |
| v2.1 §5.2 | Causal Graph Complexity Manager | causal_depth property, subgraph query foundations |
| Impl Plan §2.3 | GraphStore Abstraction Layer | Protocol-based design requirement |
| Impl Plan §2.4 | Structural Code Isolation | Separate Protocol implementations for WM and Quarantine |

---

## 3. Interface Contract — GraphStore Protocol

### 3.1 Protocol Definition

The `GraphStore` Protocol defines the abstract interface for all graph operations. All components interact with the World Model exclusively through this protocol — never by importing Neo4j operations directly.

★ Changes from v0.1: Added `health_check()`, `get_node_history()`, `expected_version` support. Updated `apply_confidence_decay` to include BSM reconciliation contract.

```python
# src/aria/core/protocols.py

from __future__ import annotations
from typing import Protocol, Sequence
from aria.world_model.schema import (
    WorldNode,
    WorldNodeVersion,
    CausalRelationship,
    CausalSubgraph,
    NodeType,
    BeliefState,
    RelationshipType,
    RelationshipDirection,
    NodeUpdate,
    RelationshipUpdate,
)


class GraphStore(Protocol):
    """Abstract interface for World Model graph operations.

    All components access the causal graph through this Protocol.
    Concrete implementations exist for:
    - Neo4j (world_model/graph.py) — primary World Model
    - Neo4j (quarantine/store.py) — Simulation Quarantine Layer

    Design constraints:
    - All write operations MUST be atomic (single transaction)
    - All write operations MUST increment node.version
    - All write operations MUST update node.updated_at
    - All write operations MUST validate against Pydantic schema before persisting
    - ★ All write operations MUST record a WorldNodeVersion diff entry
    - ★ All write operations MUST use parameterized Cypher (no string formatting)
    - Soft-deleted nodes MUST NOT appear in standard queries
    - ★ Version increments MUST occur only after diff commit succeeds
    """

    # ── Health & Status ────────────────────────────────────────────

    def health_check(self) -> bool:                                    # ★ Grok
        """Verify the graph backend is reachable and operational.

        Returns:
            True if connection is healthy, False otherwise.
        """
        ...

    # ── Node Operations ────────────────────────────────────────────

    def create_node(self, node: WorldNode) -> str:
        """Create a new node in the graph.

        Also creates the initial WorldNodeVersion record (version 1).

        Args:
            node: Validated WorldNode instance. node_id must be pre-generated.

        Returns:
            The node_id of the created node.

        Raises:
            NodeAlreadyExistsError: If node_id already exists in graph.
            SchemaValidationError: If node fails Pydantic validation.
            InvariantViolationError: If node contradicts a Tier 0 invariant.
        """
        ...

    def get_node(self, node_id: str) -> WorldNode:
        """Retrieve a single node by ID.

        Args:
            node_id: UUID string of the node.

        Returns:
            Validated WorldNode instance.

        Raises:
            NodeNotFoundError: If node_id does not exist or is soft-deleted.
        """
        ...

    def get_nodes_batch(self, node_ids: Sequence[str]) -> list[WorldNode]:
        """Retrieve multiple nodes by ID in a single round-trip.

        Args:
            node_ids: Sequence of UUID strings.

        Returns:
            List of validated WorldNode instances (order matches input).

        Raises:
            NodeNotFoundError: If any node_id does not exist or is soft-deleted.
        """
        ...

    def update_node(self, node_id: str, updates: NodeUpdate) -> WorldNode:
        """Update specific properties of an existing node.

        Automatically increments version, updates updated_at.
        Validates the resulting node against Pydantic schema.
        ★ Records a WorldNodeVersion diff entry before committing.
        ★ If expected_version is set on the NodeUpdate, performs
          optimistic concurrency check.

        Args:
            node_id: UUID string of the node to update.
            updates: NodeUpdate instance containing only the fields to change.

        Returns:
            The updated WorldNode instance (post-validation).

        Raises:
            NodeNotFoundError: If node_id does not exist or is soft-deleted.
            SchemaValidationError: If resulting node fails validation.
            InvariantViolationError: If update violates a Tier 0 invariant.
            ImmutableNodeError: If node is a Tier 0 seed ontology node.
            ★ ConcurrentWriteError: If expected_version does not match current.
        """
        ...

    def soft_delete_node(self, node_id: str, reason: str) -> None:
        """Mark a node as soft-deleted. Does not remove from graph.

        Soft-deleted nodes:
        - Are excluded from all standard queries
        - Retain full history for audit purposes
        - Can be referenced in deletion audit logs
        - Are NEVER physically removed in Phase 1

        Args:
            node_id: UUID string of the node to soft-delete.
            reason: Human-readable reason for deletion (audit trail).

        Raises:
            NodeNotFoundError: If node_id does not exist.
            ImmutableNodeError: If node is a Tier 0 seed ontology node.
        """
        ...

    def node_exists(self, node_id: str) -> bool:
        """Check if a non-deleted node exists."""
        ...

    # ── Relationship Operations ────────────────────────────────────

    def create_relationship(
        self, relationship: CausalRelationship
    ) -> str | tuple[str, str]:
        """Create a causal relationship between two nodes.

        Both source and target nodes must exist and not be soft-deleted.
        ★ For CONTRADICTS type: automatically creates the symmetric
          reverse edge (B→A) in the same transaction. Returns a tuple
          of both relationship_ids.

        Args:
            relationship: Validated CausalRelationship instance.

        Returns:
            The relationship_id of the created relationship.
            ★ For CONTRADICTS: tuple of (forward_id, reverse_id).

        Raises:
            NodeNotFoundError: If source or target node does not exist.
            DuplicateRelationshipError: If identical relationship already exists.
            SchemaValidationError: If relationship fails validation.
            InvariantViolationError: If relationship creates a logical contradiction.
        """
        ...

    def get_relationships(
        self,
        node_id: str,
        direction: RelationshipDirection,
        rel_types: Sequence[RelationshipType] | None = None,
    ) -> list[CausalRelationship]:
        """Get all relationships for a node, optionally filtered by type.

        Args:
            node_id: UUID string of the node.
            direction: OUTGOING, INCOMING, or BOTH.
            rel_types: Optional filter — only return these relationship types.

        Returns:
            List of validated CausalRelationship instances.

        Raises:
            NodeNotFoundError: If node_id does not exist.
        """
        ...

    def update_relationship(
        self, relationship_id: str, updates: RelationshipUpdate
    ) -> CausalRelationship:
        """Update properties of an existing relationship.

        Args:
            relationship_id: UUID string of the relationship.
            updates: RelationshipUpdate instance with fields to change.

        Returns:
            Updated CausalRelationship instance.

        Raises:
            RelationshipNotFoundError: If relationship_id does not exist.
            SchemaValidationError: If resulting relationship fails validation.
        """
        ...

    def delete_relationship(self, relationship_id: str, reason: str) -> None:
        """Soft-delete a relationship.

        Args:
            relationship_id: UUID string of the relationship.
            reason: Human-readable reason for deletion.

        Raises:
            RelationshipNotFoundError: If relationship_id does not exist.
        """
        ...

    # ── Query Operations ───────────────────────────────────────────

    def query_by_type(
        self,
        node_type: NodeType,
        belief_states: Sequence[BeliefState] | None = None,
        limit: int = 100,
    ) -> list[WorldNode]:
        """Query nodes by type, optionally filtered by belief state."""
        ...

    def query_causal_neighborhood(
        self,
        node_id: str,
        depth: int,
        direction: RelationshipDirection = RelationshipDirection.BOTH,
        rel_types: Sequence[RelationshipType] | None = None,
    ) -> CausalSubgraph:
        """Extract the causal neighborhood of a node up to a given depth.

        ★ Default direction is BOTH. Callers (especially the Predictive
          Loop) should specify OUTGOING for forward prediction and
          INCOMING for causal diagnosis when performance matters.

        Args:
            node_id: Root node for neighborhood extraction.
            depth: Maximum traversal depth (hops).
            direction: ★ Traversal direction (default BOTH).
            rel_types: Optional — only traverse these relationship types.

        Returns:
            CausalSubgraph containing all reached nodes and relationships.

        Raises:
            NodeNotFoundError: If node_id does not exist.
            ★ QueryTimeoutError: If query exceeds configured timeout.
        """
        ...

    def query_contradictions(self, node_id: str) -> list[CausalRelationship]:
        """Find all CONTRADICTS relationships involving a node."""
        ...

    def query_nodes_by_source(
        self, source: str, limit: int = 1000
    ) -> list[WorldNode]:
        """Query all nodes from a specific source (e.g., "seed-ontology-t0")."""
        ...

    # ── Confidence Operations ──────────────────────────────────────

    def update_confidence(
        self,
        node_id: str,
        new_confidence: float,
        new_epistemic_uncertainty: float | None = None,
        new_aleatoric_uncertainty: float | None = None,
    ) -> WorldNode:
        """Update confidence and uncertainty scores for a node.

        ★ Records a WorldNodeVersion diff. Supports expected_version
          via the general update path for safety-critical callers.

        Args:
            node_id: UUID string of the node.
            new_confidence: New confidence value (0.0–1.0).
            new_epistemic_uncertainty: Optional new epistemic uncertainty.
            new_aleatoric_uncertainty: Optional new aleatoric uncertainty.

        Returns:
            Updated WorldNode instance.

        Raises:
            NodeNotFoundError: If node_id does not exist.
            ValueError: If confidence is outside [0.0, 1.0].
        """
        ...

    def apply_confidence_decay(self, decay_cutoff: float) -> DecayResult:
        """Apply confidence decay to all eligible nodes.

        ★ Supports two decay modes (configured via WorldModelConfig):
        - exponential (default): confidence *= (1 - decay_rate)
        - additive: confidence -= decay_rate, clamped to 0.0

        ★ CRITICAL CONTRACT (Claude Obs 1, confirmed by all reviewers):
        After batch decay, this method MUST perform synchronous BSM
        reconciliation: query all nodes whose confidence dropped below
        their current belief_state's minimum threshold, and invoke BSM
        transition evaluation BEFORE returning. The graph must never
        be left in an inconsistent state where a CONFIRMED node has
        sub-threshold confidence.

        ★ Diff recording: Only record version diffs when confidence
        change exceeds epsilon (default 0.0001) to prevent diff
        explosion from high-frequency decay cycles (GPT-5 risk).

        Args:
            decay_cutoff: Minimum confidence threshold. Nodes below this
                after decay are flagged for BSM review.

        Returns:
            ★ DecayResult with affected_count and threshold_crossings.
        """
        ...

    # ── Version History ────────────────────────────────────────────

    def get_node_history(                                              # ★ NEW
        self, node_id: str, limit: int = 100
    ) -> list[WorldNodeVersion]:
        """Retrieve the version history (diffs) for a node.

        Returns diff entries in reverse chronological order (newest first).

        Args:
            node_id: UUID string of the node.
            limit: Maximum number of version entries to return.

        Returns:
            List of WorldNodeVersion diff records.

        Raises:
            NodeNotFoundError: If node_id does not exist.
        """
        ...

    # ── Graph Metrics ──────────────────────────────────────────────

    def count_nodes(
        self,
        node_type: NodeType | None = None,
        belief_state: BeliefState | None = None,
    ) -> int:
        """Count nodes, optionally filtered by type and/or belief state."""
        ...

    def count_relationships(
        self, rel_type: RelationshipType | None = None
    ) -> int:
        """Count relationships, optionally filtered by type."""
        ...
```

### 3.2 Protocol Design Rationale

| Decision | Rationale |
|----------|-----------|
| **Protocol, not ABC** | Structural subtyping (duck typing). Implementations don't need to inherit — they just need to match the interface. Quarantine store is completely independent. |
| **Pydantic models in, Pydantic models out** | Every method accepts and returns validated Pydantic instances. No raw dicts cross the Protocol boundary. Schema is the contract. |
| **Write operations return updated model** | After every write, the caller receives the validated, post-write state. Prevents read-after-write inconsistencies. |
| **Batch read, no batch write** | Batch reads are common (subgraph extraction). Batch writes are dangerous — each write needs individual validation and invariant checking. |
| **Separate confidence update method** | Confidence updates are the most frequent write operation (every predictive loop tick). Dedicated method avoids overhead of general-purpose update validation for this hot path. |
| **Soft-delete only** | Phase 1 never physically removes nodes. Full audit trail. |
| ★ **Optimistic concurrency opt-in** | `expected_version` prevents lost updates in retry patterns without adding overhead when not needed (GPT-5). |
| ★ **Synchronous decay reconciliation** | Decay must not leave graph in inconsistent epistemic state. BSM reconciliation is mandatory before return (Claude Obs 1, all reviewers confirmed). |
| ★ **CONTRADICTS auto-symmetry** | Logical contradiction is bidirectional. Both edges created atomically with independent evidence metadata (GPT-5 + Grok). |
| ★ **health_check() on Protocol** | Runtime backend validation without requiring a full query (Grok). |

### 3.3 Supporting Types

```python
# ── Direction Enum ─────────────────────────────────────────────────

class RelationshipDirection(str, Enum):
    """Direction filter for relationship queries."""
    OUTGOING = "outgoing"
    INCOMING = "incoming"
    BOTH = "both"


# ── Neo4j Boundary Types ──────────────────────────────────────────

Neo4jPropertyValue = str | float | int | bool | list[str] | datetime | None  # ★ Claude Obs 2
Neo4jPropertyMap = dict[str, Neo4jPropertyValue]                              # ★ Claude Obs 2
"""Type alias for Neo4j driver boundary. Keeps Any out of core modules.
The # type: ignore stays only at the driver call site (one line)."""


# ── Partial Update Models ──────────────────────────────────────────

class NodeUpdate(BaseModel):
    """Partial update payload for a node.

    Only fields that are set (not None) are applied.
    ★ expected_version enables optimistic concurrency when set.
    """
    expected_version: int | None = None                                # ★ GPT-5 Condition 1

    node_type: NodeType | None = None
    belief_state: BeliefState | None = None
    confidence: float | None = Field(None, ge=0.0, le=1.0)
    epistemic_uncertainty: float | None = Field(None, ge=0.0, le=1.0)
    aleatoric_uncertainty: float | None = Field(None, ge=0.0, le=1.0)
    falsifiability_criteria: list[str] | None = None
    confidence_decay_rate: float | None = Field(None, ge=0.0)
    source: str | None = None
    simulation_trust_score: float | None = Field(None, ge=0.0, le=1.0)
    real_sim_weight: float | None = Field(None, ge=0.0)
    domain_scope: list[str] | None = None
    salience_score: float | None = Field(None, ge=0.0, le=1.0)
    emotional_valence: float | None = Field(None, ge=-1.0, le=1.0)
    identity_invariant: bool | None = None
    context_diversity: int | None = Field(None, ge=0)
    causal_depth: int | None = Field(None, ge=0)

    model_config = ConfigDict(extra="forbid")


class RelationshipUpdate(BaseModel):
    """Partial update payload for a relationship."""
    strength: float | None = Field(None, ge=0.0, le=1.0)
    confidence: float | None = Field(None, ge=0.0, le=1.0)
    context_tags: list[str] | None = None
    observed_count: int | None = Field(None, ge=0)
    probability: float | None = Field(None, ge=0.0, le=1.0)
    threshold: float | None = Field(None, ge=0.0, le=1.0)

    model_config = ConfigDict(extra="forbid")


# ── Decay Result ───────────────────────────────────────────────────

class DecayResult(BaseModel):                                          # ★ NEW
    """Result of a confidence decay cycle."""
    affected_count: int = Field(..., description="Nodes whose confidence changed.")
    threshold_crossings: list[str] = Field(
        default_factory=list,
        description="node_ids that crossed below their state's minimum threshold.",
    )
    bsm_transitions_triggered: int = Field(
        default=0,
        description="Number of BSM transitions invoked during reconciliation.",
    )
```

### 3.4 Error Hierarchy

```python
# src/aria/exceptions.py

class AriaError(Exception):
    """Base exception for all ARIA errors."""


class GraphStoreError(AriaError):
    """Base exception for graph store operations."""


class NodeNotFoundError(GraphStoreError):
    """Node does not exist or is soft-deleted."""
    def __init__(self, node_id: str) -> None:
        self.node_id = node_id
        super().__init__(f"Node not found: {node_id}")


class NodeAlreadyExistsError(GraphStoreError):
    """Attempted to create a node with an existing ID."""
    def __init__(self, node_id: str) -> None:
        self.node_id = node_id
        super().__init__(f"Node already exists: {node_id}")


class RelationshipNotFoundError(GraphStoreError):
    """Relationship does not exist."""
    def __init__(self, relationship_id: str) -> None:
        self.relationship_id = relationship_id
        super().__init__(f"Relationship not found: {relationship_id}")


class DuplicateRelationshipError(GraphStoreError):
    """Identical relationship already exists between the same nodes."""
    def __init__(self, source_id: str, target_id: str, rel_type: str) -> None:
        self.source_id = source_id
        self.target_id = target_id
        self.rel_type = rel_type
        super().__init__(
            f"Duplicate relationship: {source_id} -[{rel_type}]-> {target_id}"
        )


class SchemaValidationError(GraphStoreError):
    """Node or relationship failed Pydantic schema validation."""


class InvariantViolationError(GraphStoreError):
    """Operation would violate a Tier 0 logical invariant.

    HARD ERROR. Operation rejected. Logged as critical structured event.
    Maps to GPT-5's Invariant Violation Alarm.
    """
    def __init__(self, invariant: str, detail: str) -> None:
        self.invariant = invariant
        self.detail = detail
        super().__init__(f"Invariant violation [{invariant}]: {detail}")


class ImmutableNodeError(GraphStoreError):
    """Attempted to modify or delete an immutable node (Tier 0 Seed Ontology)."""
    def __init__(self, node_id: str) -> None:
        self.node_id = node_id
        super().__init__(f"Cannot modify immutable node: {node_id}")


class ConcurrentWriteError(GraphStoreError):                          # ★ GPT-5 Condition 1
    """Node version mismatch — another write occurred since the caller's read.

    Raised when expected_version is set on NodeUpdate and the current
    node version does not match.
    """
    def __init__(self, node_id: str, expected: int, actual: int) -> None:
        self.node_id = node_id
        self.expected_version = expected
        self.actual_version = actual
        super().__init__(
            f"Concurrent write on {node_id}: "
            f"expected version {expected}, found {actual}"
        )


class QueryTimeoutError(GraphStoreError):                             # ★ Claude Obs 3
    """Graph query exceeded the configured timeout."""
    def __init__(self, operation: str, timeout_ms: float) -> None:
        self.operation = operation
        self.timeout_ms = timeout_ms
        super().__init__(f"Query timeout ({timeout_ms}ms) on: {operation}")
```

---

## 4. Data Structures — World Model Schema

### 4.1 Enumerations

All enumerations are defined as Python `str, Enum` subclasses for JSON serialization compatibility and Neo4j string storage. Unchanged from v0.1.

```python
# src/aria/world_model/schema.py
# Requires: pydantic >= 2.2 (★ Gemini: ValidationInfo requirement)

from __future__ import annotations
import uuid
from datetime import datetime, timezone
from enum import Enum
from pydantic import BaseModel, ConfigDict, Field, field_validator, ValidationInfo


class NodeType(str, Enum):
    """Classification of what a node represents. From v2.1 §4.1."""
    ENTITY = "entity"
    RELATIONSHIP = "relationship"
    STATE = "state"
    EVENT = "event"
    CONSTRAINT = "constraint"
    PREDICTION = "prediction"
    OBSERVATION = "observation"


class BeliefState(str, Enum):
    """Epistemic status. Governed by BSM (CDD-03). From v2.1 §4.1."""
    HYPOTHESIS = "hypothesis"
    TENTATIVE = "tentative"
    CONFIRMED = "confirmed"
    DEPRECATED = "deprecated"
    REFUTED = "refuted"
    SIMULATED = "simulated"


class NodeSource(str, Enum):
    """Origin of a node. Determines mutability rules. From v2.1 §4.1."""
    LEARNED = "learned"
    INFERRED = "inferred"
    OBSERVED = "observed"
    MODULE_PROVIDED = "module-provided"
    SEED_ONTOLOGY_T0 = "seed-ontology-t0"
    SEED_ONTOLOGY_T1 = "seed-ontology-t1"
    SEED_ONTOLOGY_T2 = "seed-ontology-t2"
    SIMULATED = "simulated"


class RelationshipType(str, Enum):
    """Complete causal relationship vocabulary. From v2.1 §4.2."""
    # Original (9)
    CAUSES = "CAUSES"
    PROBABILISTICALLY_CAUSES = "PROBABILISTICALLY_CAUSES"
    ENABLES = "ENABLES"
    PREVENTS = "PREVENTS"
    REQUIRES = "REQUIRES"
    IS_PART_OF = "IS_PART_OF"
    PRECEDES = "PRECEDES"
    CONTRADICTS = "CONTRADICTS"
    CORRELATES_WITH = "CORRELATES_WITH"
    # Added in v2.0 (3)
    CONFOUNDER = "CONFOUNDER"
    MEDIATOR = "MEDIATOR"
    STOCHASTIC_THRESHOLD = "STOCHASTIC_THRESHOLD"
```

### 4.2 WorldNode Model

The primary data structure of the entire system. 20 properties from v2.1 §4.1, plus soft-delete fields.

```python
class WorldNode(BaseModel):
    """A single node in the ARIA World Model causal graph.

    All 20 properties from v2.1 §4.1. Immutability rules:
    - source = SEED_ONTOLOGY_T0: cannot be modified after creation
    - source = SEED_ONTOLOGY_T1: only modified via hard review
    - node_id is NEVER reused, even after soft-delete
    """
    model_config = ConfigDict(
        extra="forbid",
        validate_assignment=True,
        frozen=False,
        str_strip_whitespace=True,
    )

    # ── Identity ───────────────────────────────────────────────────
    node_id: str = Field(
        default_factory=lambda: str(uuid.uuid4()),
        description="Globally unique. Stable. Never reused.",
    )
    node_type: NodeType = Field(
        ..., description="What this node represents in the World Model.",
    )

    # ── Epistemics ─────────────────────────────────────────────────
    belief_state: BeliefState = Field(
        default=BeliefState.HYPOTHESIS,
        description="Epistemic status. Governed by BSM (CDD-03).",
    )
    confidence: float = Field(
        default=0.5, ge=0.0, le=1.0,
        description="Certainty score. Meaningful only in context of belief_state.",
    )
    epistemic_uncertainty: float = Field(
        default=0.5, ge=0.0, le=1.0,
        description="Uncertainty reducible by information. Drives Curiosity Drive.",
    )
    aleatoric_uncertainty: float = Field(
        default=0.0, ge=0.0, le=1.0,
        description="Irreducible uncertainty. Drives conservative action selection.",
    )
    falsifiability_criteria: list[str] = Field(
        default_factory=list,
        description="What observations would refute this node. Required for CONFIRMED.",
    )
    confidence_decay_rate: float = Field(
        default=0.0, ge=0.0,
        description="Rate at which confidence decays. 0.0 = no decay (Tier 0).",
    )

    # ── Provenance ─────────────────────────────────────────────────
    source: NodeSource = Field(
        ..., description="Origin. Determines mutability rules.",
    )

    # ── Simulation (v2.1) ──────────────────────────────────────────
    simulation_trust_score: float = Field(
        default=0.0, ge=0.0, le=1.0,
        description="For SIMULATED nodes: accuracy track record.",
    )
    real_sim_weight: float = Field(
        default=1.0, ge=0.0,
        description="Real-to-simulated weighting. 1.0 = real. <1.0 = simulated.",
    )

    # ── Temporality ────────────────────────────────────────────────
    created_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc),
    )
    updated_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc),
    )
    version: int = Field(default=1, ge=1, description="Monotonic. Incremented on every update.")

    # ── Scope & Salience ───────────────────────────────────────────
    domain_scope: list[str] = Field(default_factory=list)
    salience_score: float = Field(default=0.5, ge=0.0, le=1.0)
    emotional_valence: float = Field(default=0.0, ge=-1.0, le=1.0)

    # ── Structural ─────────────────────────────────────────────────
    identity_invariant: bool = Field(default=False)
    context_diversity: int = Field(default=0, ge=0)
    causal_depth: int = Field(default=0, ge=0)

    # ── Soft Delete ────────────────────────────────────────────────
    is_deleted: bool = Field(default=False)
    deleted_at: datetime | None = Field(default=None)
    deleted_reason: str | None = Field(default=None)

    # ── Validators ─────────────────────────────────────────────────

    @field_validator("real_sim_weight")
    @classmethod
    def real_sim_weight_bounds(cls, v: float, info: ValidationInfo) -> float:
        """Simulated nodes must have real_sim_weight < 1.0."""
        source = info.data.get("source")
        if source == NodeSource.SIMULATED and v >= 1.0:
            raise ValueError(
                f"SIMULATED source nodes must have real_sim_weight < 1.0 (got {v})."
            )
        return v
```

### 4.3 CausalRelationship Model

★ Changes from v0.1: Added `evidence_trace` field (GPT-5 Q1).

```python
class CausalRelationship(BaseModel):
    """A causal relationship (edge) between two WorldNodes.

    In Neo4j: (source)-[rel_type {properties}]->(target)

    ★ CONTRADICTS relationships: when created via GraphStore, the
    symmetric reverse edge is auto-created in the same transaction.
    Each direction carries independent evidence metadata.
    """
    model_config = ConfigDict(extra="forbid", validate_assignment=True)

    # ── Identity ───────────────────────────────────────────────────
    relationship_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    rel_type: RelationshipType = Field(...)
    source_node_id: str = Field(...)
    target_node_id: str = Field(...)

    # ── Strength ───────────────────────────────────────────────────
    strength: float = Field(default=0.5, ge=0.0, le=1.0)
    confidence: float = Field(default=0.5, ge=0.0, le=1.0)

    # ── Evidence ───────────────────────────────────────────────────
    observed_count: int = Field(default=1, ge=0)
    context_tags: list[str] = Field(default_factory=list)
    evidence_trace: str | None = Field(                                # ★ GPT-5 Q1
        default=None,
        description=(
            "Reference to the audit log event that established this "
            "relationship. Improves forensic traceability."
        ),
    )

    # ── Type-Specific Properties ───────────────────────────────────
    probability: float | None = Field(default=None, ge=0.0, le=1.0)
    threshold: float | None = Field(default=None, ge=0.0, le=1.0)
    confounder_node_id: str | None = Field(default=None)
    mediator_node_id: str | None = Field(default=None)

    # ── Temporality & Soft Delete ──────────────────────────────────
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    is_deleted: bool = Field(default=False)
    deleted_at: datetime | None = Field(default=None)
    deleted_reason: str | None = Field(default=None)

    # ── Validators ─────────────────────────────────────────────────

    @field_validator("probability")
    @classmethod
    def probability_required_for_probabilistic(
        cls, v: float | None, info: ValidationInfo
    ) -> float | None:
        rel = info.data.get("rel_type")
        if rel == RelationshipType.PROBABILISTICALLY_CAUSES and v is None:
            raise ValueError("PROBABILISTICALLY_CAUSES requires a probability value.")
        return v

    @field_validator("threshold")
    @classmethod
    def threshold_required_for_stochastic(
        cls, v: float | None, info: ValidationInfo
    ) -> float | None:
        rel = info.data.get("rel_type")
        if rel == RelationshipType.STOCHASTIC_THRESHOLD and v is None:
            raise ValueError("STOCHASTIC_THRESHOLD requires a threshold value.")
        return v

    @field_validator("confounder_node_id")
    @classmethod
    def confounder_id_for_confounder_type(
        cls, v: str | None, info: ValidationInfo
    ) -> str | None:
        rel = info.data.get("rel_type")
        if rel == RelationshipType.CONFOUNDER and v is None:
            raise ValueError("CONFOUNDER requires confounder_node_id.")
        return v
```

### ★ 4.4 WorldNodeVersion Model (NEW — GPT-5 Condition 3)

Diff-based version history. Stores only changed fields, not full snapshots. Every mutation path (update, decay, promotion, BSM transition) MUST emit a diff entry. The diff model supports deterministic replay.

```python
class FieldDiff(BaseModel):                                            # ★ NEW
    """A single field change in a version diff.

    ★ GPT-5 requirement: store old_value and new_value for
    deterministic replay capability.
    ★ GPT-5 risk: confidence values stored at fixed precision
    (6 decimal places) to prevent floating-point replay drift.
    """
    model_config = ConfigDict(extra="forbid")

    field_name: str
    old_value: str = Field(description="JSON-serialized previous value.")
    new_value: str = Field(description="JSON-serialized new value.")


class WorldNodeVersion(BaseModel):                                     # ★ NEW
    """A version diff record for a WorldNode.

    Stored as :WorldNodeVersion nodes in Neo4j, linked to the
    active WorldNode via a VERSIONED_AS relationship.

    Design constraints (GPT-5 Condition 3 requirements):
    - Every mutation path MUST emit a diff entry
    - No mutation path may bypass diff recording
    - Version increments ONLY after diff commit succeeds
    - Diffs must enable deterministic state reconstruction
    """
    model_config = ConfigDict(extra="forbid")

    version_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    node_id: str = Field(..., description="The WorldNode this diff belongs to.")
    version: int = Field(..., ge=1, description="The version number this diff produced.")
    timestamp: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    trigger: str = Field(
        ...,
        description=(
            "What caused this change: 'create', 'update', 'decay', "
            "'promotion', 'bsm_transition', 'soft_delete'."
        ),
    )
    changed_fields: list[FieldDiff] = Field(
        ...,
        description="List of fields that changed in this version.",
    )
```

### 4.5 CausalSubgraph Model

★ Changes from v0.1: Added O(n) performance comment (Claude Obs 4, deferred to Phase 2).

```python
class CausalSubgraph(BaseModel):
    """A subgraph extracted from the World Model.

    Used by Predictive Loop, Causal Complexity Manager, and
    SimplifiedCounterfactualGenerator.
    """
    model_config = ConfigDict(extra="forbid")

    root_node_id: str = Field(...)
    nodes: dict[str, WorldNode] = Field(default_factory=dict)
    relationships: list[CausalRelationship] = Field(default_factory=list)
    depth: int = Field(..., ge=0)

    @property
    def node_count(self) -> int:
        return len(self.nodes)

    @property
    def relationship_count(self) -> int:
        return len(self.relationships)

    def get_node(self, node_id: str) -> WorldNode:
        if node_id not in self.nodes:
            raise NodeNotFoundError(node_id)
        return self.nodes[node_id]

    def get_relationships_for(
        self, node_id: str, direction: RelationshipDirection = RelationshipDirection.BOTH
    ) -> list[CausalRelationship]:
        # ★ O(n) scan — acceptable for Phase 1 graph sizes (<100 edges).
        # Replace with pre-built adjacency dict in Phase 2 if
        # subgraphs grow > 500 edges. (Claude Obs 4, all reviewers agree defer.)
        result: list[CausalRelationship] = []
        for rel in self.relationships:
            if direction in (RelationshipDirection.OUTGOING, RelationshipDirection.BOTH):
                if rel.source_node_id == node_id:
                    result.append(rel)
            if direction in (RelationshipDirection.INCOMING, RelationshipDirection.BOTH):
                if rel.target_node_id == node_id:
                    result.append(rel)
        return result
```

---

## 5. Algorithm / Logic Flow

### 5.1 Node Creation Flow

```
Caller creates WorldNode (Pydantic validates all 20 properties)
    │
    ▼
GraphStore.create_node(node) called
    │
    ├── Check: node_id does not already exist → NodeAlreadyExistsError
    ├── Check: if belief_state == SIMULATED → SchemaValidationError
    │         (SIMULATED nodes cannot be created in World Model store)
    ├── Check: Invariant Checker → InvariantViolationError if contradicts Tier 0
    │
    ▼
Neo4j Transaction (★ parameterized Cypher only):
    CREATE (n:WorldNode {all properties + is_deleted: false})
    CREATE (v:WorldNodeVersion {version_id, node_id, version: 1,
            trigger: "create", changed_fields: [all fields]})
    CREATE (n)-[:VERSIONED_AS]->(v)
    │
    ▼
Return node_id
Log: structlog event "node.created"
```

### ★ 5.2 Node Update Flow (with optimistic concurrency)

```
Caller creates NodeUpdate (only changed fields set)
    │
    ├── ★ If expected_version is set: optimistic concurrency mode
    │
    ▼
GraphStore.update_node(node_id, updates) called
    │
    ├── Check: node exists and is not soft-deleted → NodeNotFoundError
    ├── Check: node is not Tier 0 immutable → ImmutableNodeError
    ├── Check: if Tier 1, verify hard_review flag → ImmutableNodeError
    ├── ★ Check: if expected_version set AND node.version != expected_version
    │         → ConcurrentWriteError
    │
    ▼
Merge updates into existing node:
    existing = get_node(node_id)
    merged = apply_updates(existing, updates)
    merged.version += 1
    merged.updated_at = now()
    │
    ├── Validate merged node with Pydantic → SchemaValidationError
    ├── Check: Invariant Checker → InvariantViolationError
    │
    ▼
★ Compute diff: compare existing vs merged, build FieldDiff list
★ Confidence values rounded to 6 decimal places in diff (GPT-5 replay drift)
    │
    ▼
Neo4j Transaction (★ parameterized Cypher only):
    MATCH (n:WorldNode {node_id: $id, is_deleted: false})
    [★ WHERE n.version = $expected_version  -- if optimistic concurrency]
    SET n += {merged properties}
    CREATE (v:WorldNodeVersion {diff data})
    CREATE (n)-[:VERSIONED_AS]->(v)
    │
    ├── ★ If no rows matched (version mismatch) → ConcurrentWriteError
    │
    ▼
Return updated WorldNode
Log: structlog event "node.updated" with changed fields and version
★ Log: structlog event "node.write_slow" if write > slow_write_threshold_ms
```

### 5.3 Relationship Creation Flow

★ Changes: CONTRADICTS auto-symmetry.

```
Caller creates CausalRelationship (Pydantic validates type-specific fields)
    │
    ▼
GraphStore.create_relationship(rel) called
    │
    ├── Check: source node exists → NodeNotFoundError
    ├── Check: target node exists → NodeNotFoundError
    ├── ★ Check: no self-loop for REQUIRES type → InvariantViolationError
    ├── Check: no duplicate (same source, target, type) → DuplicateRelationshipError
    ├── Check: Invariant — CONTRADICTS creates investigation flag
    │
    ▼
★ If rel_type == CONTRADICTS:
    Create BOTH forward (A→B) and reverse (B→A) edges in same transaction
    Each carries independent observed_count, context_tags, confidence
    Return tuple(forward_id, reverse_id)
    │
Otherwise:
    Create single edge
    Return relationship_id
    │
    ▼
Log: structlog event "relationship.created"
```

### ★ 5.4 Confidence Decay Flow (with BSM reconciliation)

```
GraphStore.apply_confidence_decay(decay_cutoff) called
    │
    ▼
Read config: decay_mode ("exponential" | "additive")                   # ★ GPT-5 Condition 5
    │
    ▼
Neo4j Batch Query:
    MATCH (n:WorldNode)
    WHERE n.is_deleted = false
      AND n.confidence_decay_rate > 0
      AND n.belief_state IN ['hypothesis', 'tentative', 'confirmed']
    │
    ├── If exponential: SET n.confidence = n.confidence * (1 - n.confidence_decay_rate)
    ├── If additive:    SET n.confidence = MAX(0.0, n.confidence - n.confidence_decay_rate)
    │
    ▼
★ Diff recording (GPT-5 decay explosion risk):
    Only record WorldNodeVersion diff if |old_confidence - new_confidence| > epsilon
    epsilon = 0.0001 (configurable)
    OR batch under single "decay_batch" trigger type
    │
    ▼
★ SYNCHRONOUS BSM RECONCILIATION (Claude Obs 1, all reviewers confirmed):
    Query: all nodes where confidence < state_minimum_threshold
      (thresholds defined in CDD-06, consumed by CDD-03)
    For each: invoke BSM transition evaluation
      (typically CONFIRMED → DEPRECATED or TENTATIVE → HYPOTHESIS)
    This happens BEFORE returning — graph is never inconsistent
    │
    ▼
Return DecayResult(affected_count, threshold_crossings, bsm_transitions_triggered)
Log: structlog event "confidence.decay_applied"
```

### 5.5 Causal Neighborhood Query Flow

★ Changes: direction parameter, transaction timeout.

```
GraphStore.query_causal_neighborhood(node_id, depth, direction, rel_types) called
    │
    ├── Check: root node exists → NodeNotFoundError
    ├── ★ Set transaction timeout: query_timeout_ms (configurable)
    │     For depth >= 5: effective_timeout = base * (1 + depth/3)     # GPT-5 suggestion
    │
    ▼
Neo4j Cypher (★ parameterized):
    MATCH path = (root:WorldNode {node_id: $id})-[r*1..$depth]-(neighbor)
    WHERE root.is_deleted = false
      AND neighbor.is_deleted = false
      AND ALL(rel IN relationships(path) WHERE rel.is_deleted = false)
      [AND type(r) IN $rel_types]
      [★ AND direction filter applied]
    RETURN DISTINCT nodes(path), relationships(path)
    │
    ├── ★ If timeout exceeded → QueryTimeoutError
    │
    ▼
Deserialize into CausalSubgraph
Return CausalSubgraph
Log: structlog event "query.neighborhood" with root_id, depth, node_count, rel_count
```

---

## 6. Integration Points

| Component | What It Uses | How |
|-----------|-------------|-----|
| **CDD-03: BSM** | WorldNode, BeliefState, GraphStore.update_node, ★ DecayResult.threshold_crossings | Wraps writes in state transition logic, ★ reconciles after decay |
| **CDD-02: Seed Ontology** | WorldNode, NodeSource, GraphStore.create_node | Loads YAML → WorldNode → GraphStore |
| **CDD-04: Predictive Loop** | CausalSubgraph, GraphStore.query_causal_neighborhood, update_confidence | ★ Uses OUTGOING for prediction, INCOMING for diagnosis |
| **CDD-05: Quarantine** | GraphStore Protocol (separate impl), ★ promotion weight reset contract | ★ Promotion: real_sim_weight→1.0, archive trust_score, emit audit |
| **CDD-06: Formal Semantics** | BeliefState thresholds, confidence ranges | Defines numbers BSM preconditions check |
| **CDD-07: Adversary Simulator** | CausalSubgraph, RelationshipType enums | Generates variants by mutating subgraph |
| **CDD-08: Causal Complexity Manager** | CausalSubgraph, causal_depth, GraphStore queries | Manages abstraction, subgraph budgets |
| **CDD-10: DriftMonitor** | GraphStore.count_nodes, confidence distributions | Monitors aggregate statistics |
| **CDD-11: Observability** | GraphStore metrics, ★ DecayResult, ★ write latency events | Dashboard metrics |

### ★ 6.1 Promotion Contract (CDD-05 Cross-Reference)

When a SIMULATED node is promoted from Quarantine to World Model (GPT-5 Condition 2, Grok confirmed):

1. `real_sim_weight` := 1.0
2. `simulation_trust_score` archived to version history, then set to 0.0
3. `source` changes from `simulated` to `learned`
4. `belief_state` changes from `SIMULATED` to `HYPOTHESIS`
5. A `promotion_audit` structured log event captures full pre/post state

★ **Atomicity requirement (GPT-5 hidden risk 2):** Promotion is a cross-store operation that MUST be all-or-nothing. If any step fails (quarantine read, WM write, weight reset, audit emit, diff record), the entire operation rolls back. CDD-05 must implement a two-phase commit simulation or equivalent failure rollback design.

### ★ 6.2 Confounder Triplet Migration Note (Gemini)

In Phase 1, the CONFOUNDER relationship uses a `confounder_node_id` property on the edge. In Phase 2+, the Causal Complexity Manager may need to "traverse" the confounder itself, requiring migration to a three-node triplet structure (source → confounder_node → target). Plan this migration path in CDD-08.

---

## 7. Configuration Parameters

★ Changes: Added decay_mode, decay_epsilon, timeout configs, slow_write_threshold.

```python
class WorldModelConfig(BaseSettings):
    """Configuration for World Model graph operations."""

    # Neo4j connection
    neo4j_uri: str = "bolt://localhost:7687"
    neo4j_user: str = "neo4j"
    neo4j_password: str = "aria-dev"
    neo4j_database: str = "neo4j"

    # Query defaults
    default_query_limit: int = 100
    max_query_limit: int = 10000
    default_neighborhood_depth: int = 3
    max_neighborhood_depth: int = 10

    # ★ Transaction timeouts (Claude Obs 3, GPT-5 confirmed)
    query_timeout_ms: int = 5000
    write_timeout_ms: int = 10000
    deep_traversal_timeout_scaling: bool = True  # ★ GPT-5: auto-scale for depth >= 5

    # Confidence decay
    decay_cycle_interval_seconds: float = 60.0
    decay_cutoff_threshold: float = 0.1
    decay_mode: str = "exponential"            # ★ GPT-5 Condition 5: "exponential" | "additive"
    decay_diff_epsilon: float = 0.0001         # ★ GPT-5: skip diff if change < epsilon

    # Soft delete
    soft_delete_enabled: bool = True

    # ★ Confidence precision (GPT-5 replay drift risk)
    confidence_precision_decimals: int = 6

    # Logging
    log_all_writes: bool = True
    log_query_performance: bool = True
    slow_query_threshold_ms: float = 100.0
    slow_write_threshold_ms: float = 200.0     # ★ GPT-5: emit slow_write event

    model_config = SettingsConfigDict(env_prefix="ARIA_WM_")
```

---

## 8. Error Handling & Edge Cases

### 8.1 Critical Edge Cases

| Edge Case | Behavior | Rationale |
|-----------|----------|-----------|
| Create node with SIMULATED belief_state in WM store | **Reject** SchemaValidationError | Structural quarantine firewall |
| Update a Tier 0 node | **Reject** ImmutableNodeError | Tier 0 immutable by definition |
| Create CONTRADICTS relationship | ★ **Auto-create symmetric edge** in same transaction | Contradiction is logically symmetric (GPT-5 + Grok) |
| ★ Update with expected_version mismatch | **Reject** ConcurrentWriteError | Optimistic concurrency (GPT-5 Condition 1) |
| ★ Confidence decay drops CONFIRMED below threshold | **Synchronous BSM reconciliation** triggers DEPRECATED transition | Epistemic integrity (Claude Obs 1, all confirmed) |
| ★ Decay changes confidence < epsilon | **Skip diff recording** | Prevent version history explosion (GPT-5 risk) |
| ★ REQUIRES relationship with self-loop | **Reject** InvariantViolationError | "B requires B" is logically vacuous (Grok) |
| Confidence update outside [0.0, 1.0] | **Reject** ValueError | Pydantic enforces bounds |
| Soft-delete node with active relationships | **Allow** — relationships remain | Relationships retained for audit |
| Query exceeds max_query_limit | **Truncate** at limit, log warning | Prevents memory exhaustion |
| ★ Query exceeds timeout | **Raise** QueryTimeoutError | Prevents connection pool starvation (Claude Obs 3) |
| Neo4j connection failure | **Raise** GraphStoreError | Caller decides retry strategy |
| Concurrent writes to same node | ★ **Detectable** via expected_version | Last-write-wins if no version check; ConcurrentWriteError if checked |
| ★ Promotion from quarantine | **Reset** real_sim_weight→1.0, archive trust_score | Simulated experience cannot masquerade as real (GPT-5 + Grok) |
| ★ String formatting in Cypher | **Banned** — parameterized queries only | Prevents injection (GPT-5) |

### 8.2 Invariant Checking Integration

The Invariant Checker (`core/invariants.py`, fully specified in CDD-03) is called on every write:

```
create_node           → invariant_check(node, operation="create")
update_node           → invariant_check(updated_node, operation="update")
create_relationship   → invariant_check_relationship(rel, operation="create")
```

Phase 1 invariants:
- **Non-contradiction**: A node cannot have CONTRADICTS relationships with itself
- **Temporal ordering**: PRECEDES relationships cannot create cycles
- **Tier 0 immutability**: seed-ontology-t0 nodes cannot be modified
- **Quarantine boundary**: SIMULATED nodes cannot exist in World Model store
- ★ **No REQUIRES self-loops**: A node cannot require itself (Grok)

★ Every invariant violation emits a structured alert event (`invariant.violation`) for observability (GPT-5).

---

## 9. Test Plan

### 9.1 Unit Tests (`tests/unit/test_schema.py`)

All v0.1 tests retained, plus:

| ★ New Test | Validates |
|-----------|-----------|
| `test_node_update_expected_version_field` | expected_version is optional, accepted by NodeUpdate |
| `test_field_diff_serialization` | FieldDiff round-trips correctly |
| `test_world_node_version_all_triggers` | All trigger types ("create", "update", "decay", etc.) valid |
| `test_decay_result_model` | DecayResult validates correctly |
| `test_evidence_trace_field` | CausalRelationship accepts evidence_trace |
| `test_contradicts_does_not_require_extra` | CONTRADICTS type doesn't mandate extra fields |
| `test_neo4j_property_value_types` | All Neo4jPropertyValue union members are valid |

### 9.2 Property-Based Tests (`tests/property/test_schema_properties.py`)

All v0.1 tests retained, plus:

| ★ New Test | Strategy |
|-----------|----------|
| `test_no_requires_self_loops` | Random relationship generation — REQUIRES with same source/target rejected (Grok) |
| `test_confidence_precision_roundtrip` | Confidence rounded to 6 decimals before diff → replay produces exact match (GPT-5 drift) |
| `test_exponential_decay_never_negative` | Hypothesis: random decay_rate × random cycles → confidence ≥ 0 |
| `test_additive_decay_clamps_to_zero` | Hypothesis: large decay_rate × many cycles → confidence == 0.0 |

### 9.3 Integration Tests

All v0.1 tests retained, plus:

| ★ New Test | Validates |
|-----------|-----------|
| `test_optimistic_concurrency_success` | Update with correct expected_version succeeds |
| `test_optimistic_concurrency_conflict` | Update with wrong expected_version raises ConcurrentWriteError |
| `test_contradicts_creates_both_edges` | CONTRADICTS creation produces forward + reverse edges |
| `test_contradicts_edges_independent_evidence` | Each direction has its own observed_count and context_tags |
| `test_version_history_recorded_on_update` | update_node creates WorldNodeVersion diff |
| `test_version_history_recorded_on_decay` | Decay creates diff only when change > epsilon |
| `test_version_history_deterministic_replay` | 100 updates → replay diffs → reconstruct final state matches |
| `test_decay_bsm_reconciliation` | CONFIRMED node decayed below threshold → BSM transition triggered synchronously |
| `test_decay_epsilon_skip` | Confidence change < epsilon → no diff recorded |
| `test_promotion_weight_reset` | Promoted node has real_sim_weight=1.0, trust_score=0.0 |
| `test_promotion_audit_event` | Promotion emits promotion_audit structured event |
| `test_query_timeout_enforcement` | Deep query exceeding timeout raises QueryTimeoutError |
| `test_health_check_healthy` | health_check() returns True on running Neo4j |
| `test_health_check_unhealthy` | health_check() returns False on stopped Neo4j |
| `test_get_node_history` | get_node_history returns diffs in reverse chronological order |
| `test_invariant_violation_emits_alert` | Invariant violation → InvariantViolationError + alert event |
| `test_graphstore_protocol_compliance` | Neo4j impl satisfies all Protocol methods including new ones |

### ★ 9.4 Concurrency Tests (`tests/integration/test_concurrent_updates.py`)

GPT-5 requirement: verify no deadlocks or lost writes under realistic Phase 1 load.

| Test | Target |
|------|--------|
| `test_100_concurrent_confidence_updates` | 100 concurrent update_confidence on 1000 nodes — zero failures |
| `test_concurrent_write_with_version_check` | 10 concurrent updates to same node with expected_version — exactly 1 succeeds, rest get ConcurrentWriteError |

### 9.5 Performance Tests

| Test | Target |
|------|--------|
| `test_create_10k_nodes` | < 30 seconds for 10,000 insertions |
| `test_neighborhood_query_at_10k` | < 100ms for depth-3 neighborhood at 10K nodes |
| `test_batch_read_100_nodes` | < 50ms for 100-node batch retrieval |
| ★ `test_write_latency_histogram` | Write latency histogram exists, slow_write events emitted above threshold |

---

## 10. Acceptance Criteria

★ Expanded from 17 to 25 criteria.

| # | Criterion | Sprint | Verification |
|---|-----------|--------|-------------|
| 1 | GraphStore Protocol defined with all methods including ★ health_check, get_node_history | S0 | Protocol importable, mypy verifies |
| 2 | All enums implemented | S1 | Unit tests pass for all enum values |
| 3 | WorldNode Pydantic model with all 20 properties | S1 | Serialization round-trip tests pass |
| 4 | CausalRelationship model with all 12 types + ★ evidence_trace | S1 | Type-specific validators pass |
| 5 | ★ WorldNodeVersion diff model | S1 | Diff creation and history retrieval tests pass |
| 6 | CausalSubgraph model with utility methods | S1 | Subgraph property tests pass |
| 7 | NodeUpdate with ★ expected_version field | S1 | Optimistic concurrency tests pass |
| 8 | Complete exception hierarchy including ★ ConcurrentWriteError, QueryTimeoutError | S1 | All error types importable, tested |
| 9 | Neo4j GraphStore satisfies Protocol | S1 | Protocol compliance integration test passes |
| 10 | Node CRUD operational | S1 | Integration tests pass |
| 11 | Relationship CRUD operational + ★ CONTRADICTS symmetry | S1 | Both edges created in single transaction |
| 12 | Causal neighborhood query with ★ direction and ★ timeout | S1 | Depth-1, depth-3, timeout tests pass |
| 13 | ★ Exponential confidence decay with BSM reconciliation | S1 | Decay + reconciliation integration test passes |
| 14 | ★ Decay diff epsilon filtering | S1 | Sub-epsilon changes produce no diff |
| 15 | ★ Version history recorded on all mutation paths | S1 | History tests pass for update, decay, create |
| 16 | ★ Deterministic replay from diffs | S1 | 100-version replay matches final state |
| 17 | ★ Optimistic concurrency detects conflicts | S1 | ConcurrentWriteError raised on version mismatch |
| 18 | ★ health_check operational | S1 | Returns True/False correctly |
| 19 | ★ Transaction timeouts enforced | S1 | QueryTimeoutError on exceeded timeout |
| 20 | All unit tests pass | S1 | 100% of Section 9.1 tests green |
| 21 | All property-based tests pass | S1 | Hypothesis runs 1000+ examples |
| 22 | ★ Concurrency stress tests pass | S1 | 100 concurrent updates, zero failures |
| 23 | mypy --strict passes, no `Any` in core modules | S1 | Zero mypy errors |
| 24 | structlog events for all writes, ★ slow writes, ★ invariant violations | S1 | Log output verified |
| 25 | ★ CI import-isolation check for quarantine modules | S1 | Build fails if quarantine imports WM writes |

---

## 11. Open Questions — All Resolved

All 7 original open questions have been answered through the multi-LLM review process. No open questions remain.

| Q | Resolution | Decided By |
|---|-----------|------------|
| Q1: Relationship properties | Sufficient + added `evidence_trace` | GPT-5 + Grok + Gemini |
| Q2: Soft-delete strategy | Keep for both nodes and relationships | Unanimous |
| Q3: CONTRADICTS symmetry | Auto-create both edges, independent evidence | GPT-5 + Grok |
| Q4: real_sim_weight promotion | Reset to 1.0, archive trust_score | GPT-5 + Grok |
| Q5: Confidence decay model | Exponential default, additive configurable | GPT-5 + Grok |
| Q6: Traversal direction | Default BOTH, callers specify when needed | GPT-5 + Grok |
| Q7: Version history storage | Diff-based WorldNodeVersion, real get_node_history() | GPT-5 (modified) + Grok |

---

## Appendix A: Neo4j Schema Mapping

### A.1 Node Labels and Properties

| Pydantic Type | Neo4j Type | Notes |
|--------------|------------|-------|
| `str` | String | UUIDs stored as strings |
| `float` | Float | ★ Confidence rounded to 6 decimals in diffs |
| `int` | Integer | Native 64-bit |
| `bool` | Boolean | Native |
| `datetime` | DateTime | Neo4j temporal type |
| `list[str]` | String[] | Neo4j list property |
| `Enum` (str) | String | Stored as enum value string |

### A.2 Index Strategy

★ Expanded from v0.1 with relationship indexes and composite indexes.

```cypher
-- Primary lookup
CREATE INDEX idx_node_id FOR (n:WorldNode) ON (n.node_id)
CREATE CONSTRAINT uniq_node_id FOR (n:WorldNode) REQUIRE n.node_id IS UNIQUE

-- Query optimization
CREATE INDEX idx_node_type FOR (n:WorldNode) ON (n.node_type)
CREATE INDEX idx_belief_state FOR (n:WorldNode) ON (n.belief_state)
CREATE INDEX idx_source FOR (n:WorldNode) ON (n.source)
CREATE INDEX idx_is_deleted FOR (n:WorldNode) ON (n.is_deleted)

-- ★ Composite indexes (Grok recommendation)
CREATE INDEX idx_type_state FOR (n:WorldNode) ON (n.node_type, n.belief_state)
CREATE INDEX idx_state_source FOR (n:WorldNode) ON (n.belief_state, n.source)

-- ★ Version history indexes
CREATE INDEX idx_version_node FOR (v:WorldNodeVersion) ON (v.node_id)
CREATE INDEX idx_version_num FOR (v:WorldNodeVersion) ON (v.version)

-- ★ Relationship indexes (GPT-5)
CREATE CONSTRAINT uniq_rel_id FOR ()-[r]-() REQUIRE r.relationship_id IS UNIQUE
CREATE INDEX idx_rel_deleted FOR ()-[r]-() ON (r.is_deleted)

-- ★ Evidence trace index (Grok)
CREATE INDEX idx_evidence_trace FOR ()-[r]-() ON (r.evidence_trace)
```

### A.3 Serialization

★ Updated with `Neo4jPropertyMap` type alias.

```python
def node_to_neo4j_props(node: WorldNode) -> Neo4jPropertyMap:          # ★ typed
    """Convert a WorldNode to Neo4j property map.

    ★ Uses Neo4jPropertyMap (not dict[str, Any]) to maintain
    type safety at the driver boundary (Claude Obs 2).
    ★ All Cypher queries use parameterized queries only (GPT-5).
    """
    props = node.model_dump(mode="python")
    for key in ("created_at", "updated_at", "deleted_at"):
        if props.get(key) is not None:
            props[key] = props[key].isoformat()
    return {k: v for k, v in props.items() if v is not None}
```

---

## Appendix B: ★ Operational Notes

### B.1 Backup & Restore (GPT-5 recommendation)

Both Neo4j instances (World Model and Quarantine) require documented backup/restore procedures:

- **Backup frequency:** Daily for Phase 1 development
- **Backup method:** Neo4j `neo4j-admin dump` for offline backup, or `neo4j-admin backup` (Enterprise) when available
- **Restore procedure:** Document in `docs/operations/backup-restore.md`
- **Recovery testing:** Test restore from backup at least once during Sprint 1

### B.2 CI Import Isolation Check (GPT-5 Condition 4)

A CI script or flake8 plugin verifies that:
- `aria.quarantine.*` modules do NOT import from `aria.world_model.graph`
- `aria.quarantine.promotion` does NOT import World Model write methods
- Violation fails the build

---

## Appendix C: Review Feedback Incorporation Record

### GPT-5 (OpenAI) — 5 Conditions

| # | Condition | Section |
|---|-----------|---------|
| 1 | Optimistic concurrency (expected_version) | §3.1, §3.3, §3.4, §5.2 |
| 2 | Promotion weight reset + audit | §6.1 |
| 3 | Diff-based version history (WorldNodeVersion) | §4.4, §5.2, §5.4 |
| 4 | CI import isolation | Appendix B.2 |
| 5 | Exponential decay mode | §5.4, §7 |

### GPT-5 — Additional Items

| Item | Section |
|------|---------|
| Parameterized Cypher only | §3.1 design constraints, §5.1–5.5, §8.1 |
| Slow-write log event | §5.2, §7 |
| Relationship indexes | Appendix A.2 |
| Concurrency stress test | §9.4 |
| Backup/restore docs | Appendix B.1 |
| Decay diff epsilon (explosion risk) | §5.4, §7, §8.1 |
| Promotion atomicity (cross-store) | §6.1 |
| Confidence precision for replay drift | §4.4, §7 |

### Grok (xAI) — Suggestions

| Item | Section |
|------|---------|
| health_check() method | §3.1 |
| No self-loop on REQUIRES | §8.2, §9.2 |
| Composite index belief_state + source | Appendix A.2 |
| Index evidence_trace | Appendix A.2 |
| Index changed_fields for version queries | Appendix A.2 |

### Gemini (Google) — Observations

| Item | Section |
|------|---------|
| Pin Pydantic ≥ 2.2 | §4.1 header comment |
| Confounder triplet migration note | §6.2 |
| Invariant Library in CDD-03, not CDD-01 | §8.2 |

### Claude Opus 4.6 — 4th Reviewer Observations

| # | Observation | Section |
|---|------------|---------|
| 1 | BSM reconciliation after decay (synchronous) | §3.1 (apply_confidence_decay), §5.4 |
| 2 | Neo4jPropertyValue type alias | §3.3, Appendix A.3 |
| 3 | Transaction timeout configuration | §3.1 (query_causal_neighborhood), §3.4, §7 |
| 4 | CausalSubgraph linear scan (defer to Phase 2) | §4.5 |

---

*End of CDD-01 v1.0: World Model Schema & GraphStore Protocol*

**Project ARIA · Adaptive Reasoning Integrated Architecture**  
Component Design Document 01 — Foundation Layer  
Approved by All Reviewing Systems — Ready for Implementation