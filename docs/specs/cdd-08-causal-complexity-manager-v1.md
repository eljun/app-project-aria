# CDD-08: Causal Complexity Manager

> **Status:** APPROVED — All Reviewers Signed Off (4/4)  
> **Version:** 1.0  
> **Author:** Jun (Architect) + Claude Opus 4.6 (Design Partner)  
> **Depends On:** CDD-01 v1.1 (World Model Schema & GraphStore Protocol)  
> **Consumed By:** CDD-04 (Predictive Loop), CDD-07 (Adversary Simulator), CDD-09 (Physics Simulator)  
> **Sprint:** S3 (Causal Complexity Manager + Graph Operations)  
> **Classification:** Confidential — Core Team & Designated Review Partners  
> **Reviewed By:** GPT-5 (OpenAI) · Gemini (Google) · Grok (xAI) · Claude Opus 4.6 (Anthropic)  
> **Changes from v0.1:** 72 changes marked with ★

---

## 1. Purpose & Scope

As ARIA's World Model grows, the causal graph can explode in complexity. A physics domain alone can produce hundreds of nodes and thousands of relationships by mid-Phase 1. The Predictive Loop (CDD-04) cannot reason over the entire graph every tick — it needs a relevant, bounded slice of the world for each prediction cycle.

CDD-08 is ARIA's attention mechanism for causal reasoning. It answers the question: **"Given a query about node X, what is the smallest subgraph that contains enough causal context to reason well?"**

This CDD specifies four things:

**1.1 Budgeted Subgraph Extraction** — Given a query node (typically from an observation or prediction error), extract the most relevant causal neighborhood within a configurable complexity budget. The budget bounds both the number of nodes and the computational cost of extraction. If the full neighborhood exceeds budget, the system automatically abstracts up the causal hierarchy.

**1.2 Causal Depth Assignment & Hierarchical Layering** — Every node carries a `causal_depth` property (defined in v2.1 §4.1, stored in CDD-01's WorldNode schema) indicating its position in the abstraction hierarchy. Depth 0 = raw observation-level facts. Higher depths = compressed abstractions. The Complexity Manager assigns and maintains these depths as the graph evolves.

**1.3 Causal Chain Compression** — Repeatedly-validated low-level causal chains are compressed into higher-level abstract nodes. This is not lossy — the original chain remains in the graph. Compression creates an abstract node at a higher `causal_depth` that summarizes the chain, with an `ABSTRACTS` meta-relationship linking it to the original nodes. This enables multi-resolution reasoning.

**★ 1.4 Abstraction Lifecycle Management** — Abstract nodes participate in the full epistemic lifecycle. When underlying concrete nodes are demoted or refuted, the abstract node's confidence is re-evaluated. Abstractions that lose their foundation can be dissolved, restoring the system to granular evidence. This prevents "zombie abstractions" — high-confidence concepts whose supporting evidence has disappeared.

### 1.5 What This CDD Does NOT Cover

| Excluded Component | Covered In |
|-------------------|------------|
| Raw graph CRUD operations | CDD-01 (GraphStore Protocol) |
| Belief state transitions during extraction | CDD-03 (BSM) |
| Confidence thresholds for abstraction eligibility | CDD-06 (Formal Semantics) |
| Prediction generation from extracted subgraphs | CDD-04 (Predictive Loop) |
| Adversarial variant generation from subgraphs | CDD-07 (Adversary Simulator) |
| Graph-level drift monitoring | CDD-10 (DriftMonitor) |

---

## 2. Specification References

| Spec Section | Content | Relevance |
|-------------|---------|-----------|
| v2.1 §5.2 | Causal Graph Complexity Manager | Direct specification source |
| v2.1 §4.1 | `causal_depth` property on WorldNode | Schema field this CDD manages |
| v2.1 §4.2 | 12 causal relationship types | Relationship traversal semantics |
| Impl Plan §6 S3 | Sprint 3 deliverables | Acceptance criteria source |
| CDD-01 §4.5 | CausalSubgraph model | Data structure we return |
| CDD-01 §3.1 | GraphStore.query_causal_neighborhood | Low-level query we build on |
| CDD-01 §5.5 | Neighborhood query flow | Underlying traversal mechanism |
| CDD-02 §4.1 | Tier 0 nodes (causal_depth: 0) | Protected from abstraction |
| CDD-02 §5.1 | Tier 1 nodes (causal_depth: 1) | Protected from auto-compression |
| CDD-02 §6.1 | Tier 2 nodes (causal_depth: 2) | Subject to compression when validated |
| ★ CDD-02 §9.4 | Tier protection matrix | ABSTRACTS relationship constraints |
| ★ CDD-03 §5.1 | BSM reconcile_after_decay | Foundation validation hook target |
| ★ CDD-06 §6.5 | Recalibration cycle | Abstract node calibration scope |

---

## 3. Interface Contract

### 3.1 CausalComplexityManager Protocol

```python
# src/aria/causal/protocols.py

from __future__ import annotations
from typing import Protocol
from aria.world_model.schema import (
    CausalSubgraph,
    WorldNode,
    RelationshipType,
    RelationshipDirection,
)


class CausalComplexityManager(Protocol):
    """Interface for budgeted causal subgraph extraction and abstraction.

    All consumers (Predictive Loop, Adversary Simulator, etc.) access
    causal context through this protocol — never by querying the full graph.
    """

    def extract_subgraph(
        self,
        query_node_id: str,
        budget: SubgraphBudget,
        *,
        direction: RelationshipDirection = RelationshipDirection.BOTH,
        rel_types: list[RelationshipType] | None = None,
        min_confidence: float = 0.0,
        max_causal_depth: int | None = None,
        include_abstractions: bool = True,
    ) -> BudgetedSubgraph:
        """Extract the most relevant causal subgraph within budget.

        This is the primary method. All reasoning queries go through here.

        ★ Operates within a single Neo4j read transaction for snapshot
        isolation. Concurrent compression operations do not affect
        the extraction in progress. (GPT-5 observation)

        Args:
            query_node_id: The focal node (typically an observation or prediction error).
            budget: Maximum node count, relationship count, and time limit.
            direction: OUTGOING (effects), INCOMING (causes), or BOTH.
            rel_types: Filter to specific relationship types. None = all types.
            min_confidence: Exclude nodes below this confidence threshold.
            max_causal_depth: Only include nodes up to this abstraction level.
                None = include all depths.
            include_abstractions: If True and budget is exceeded at depth 0,
                automatically include higher-depth abstract nodes instead.

        Returns:
            BudgetedSubgraph with the extracted subgraph plus budget metadata.
            ★ Budget is deterministic: result node count ≤ max_nodes always.
            ★ No nodes receive automatic priority override. (GPT-5 Cond 2)

        Raises:
            NodeNotFoundError: query_node_id does not exist.
            BudgetExhaustedError: Budget constraints could not be satisfied
                even at the highest abstraction level.
        """
        ...

    def assign_causal_depth(
        self,
        node_id: str,
        depth: int,
        *,
        reason: str = "",
    ) -> WorldNode:
        """Set the causal_depth of a node.

        Depth 0: Raw observation-level facts.
        Depth 1: Seed ontology physical bedrock (Tier 1).
        Depth 2: Structural priors and first-order abstractions.
        Depth 3+: Higher-order compressed abstractions.

        Tier 0 and Tier 1 nodes cannot have their depth changed
        (enforced by tier protection from CDD-02).

        ★ Depth capped at max_allowed_causal_depth (default: 10).
        (GPT-5 Cond 4)

        Raises:
            TierProtectionError: Attempting to change depth of Tier 0/1 node.
            ★ DepthCapExceededError: depth > max_allowed_causal_depth.
        """
        ...

    def compress_chain(
        self,
        chain_node_ids: list[str],
        *,
        abstract_label: str,
        abstract_description: str,
    ) -> CompressionResult:
        """Compress a validated causal chain into an abstract node.

        Creates a new node at causal_depth = max(chain depths) + 1
        with ABSTRACTS relationships to each original node.
        Original chain is NOT deleted — both levels coexist.

        ★ Mirrored boundary relationships are marked
        derived_from_chain=True and excluded from calibration,
        adversarial validation, and independent evidence counting.
        (GPT-5 Cond 1)

        Preconditions:
            - Chain must have ≥ 2 nodes.
            - All nodes must be CONFIRMED belief state.
            - All chain relationships must be CAUSES or PROBABILISTICALLY_CAUSES.
            - No Tier 0 or Tier 1 nodes in chain (tier protection).
            - ★ Resulting depth ≤ max_allowed_causal_depth. (GPT-5 Cond 4)

        Returns:
            CompressionResult with abstract node ID, new depth, and
            the ABSTRACTS relationships created.
            ★ Includes chain_confidences and confidence_method for
            traceability. (Claude Obs 2)

        Raises:
            CompressionPreconditionError: Chain doesn't meet requirements.
            TierProtectionError: Chain includes protected tier nodes.
            ★ DepthCapExceededError: Would exceed max_allowed_causal_depth.
        """
        ...

    def dissolve_abstraction(                                           # ★ NEW (Gemini Cond C)
        self,
        abstract_node_id: str,
    ) -> DissolutionResult:
        """Safely dissolve an abstraction, restoring concrete evidence.

        Soft-deletes the abstract node, all ABSTRACTS relationships
        from it, and all mirrored boundary relationships
        (derived_from_chain=True). Original concrete chain remains
        untouched.

        ★ Triggers BSM reconciliation event for boundary concrete
        nodes to maintain downstream consistency. (Grok v2 addition)

        ★ Operation is transactional — all-or-nothing. If any step
        fails, the entire dissolution rolls back. (GPT-5 v2 suggestion)

        Raises:
            NodeNotFoundError: abstract_node_id does not exist.
            NotAnAbstractionError: Node is not source="compressed".
        """
        ...

    def get_abstraction_hierarchy(
        self,
        node_id: str,
        *,
        direction: str = "up",  # "up" = find abstractions, "down" = find components
    ) -> list[WorldNode]:
        """Navigate the abstraction hierarchy for a node.

        "up": Find all abstract nodes that ABSTRACTS this node (parents).
        "down": Find all nodes that this node ABSTRACTS (children).

        Returns nodes ordered by causal_depth (ascending for "down",
        descending for "up").
        """
        ...

    def compute_graph_statistics(self) -> GraphComplexityStats:
        """Return current complexity metrics for the graph.

        ★ Results cached with TTL (default 60s). (Q7 consensus)

        Used by CDD-11 (Observability) for dashboard and by
        stress tests for performance benchmarking.
        """
        ...

    def check_abstraction_foundation(                                   # ★ NEW (Gemini B + Grok 1)
        self,
        abstract_node_id: str,
    ) -> FoundationCheckResult:
        """Re-evaluate an abstract node's confidence based on its children.

        Called by BSM reconciliation (CDD-03 §5.1) when a concrete
        node that has ABSTRACTS parents is demoted.

        Recalculates confidence from current children's confidences.
        If any child is REFUTED, flags abstract for dissolution.
        If recalculated confidence < confirmed_decay_threshold (0.40),
        triggers BSM demotion evaluation of the abstract node.
        """
        ...
```

### 3.2 Error Cases

| Error | Condition | Recovery |
|-------|-----------|----------|
| `NodeNotFoundError` | Query node doesn't exist | Caller retries with valid ID |
| `BudgetExhaustedError` | Cannot satisfy budget even at max abstraction | Caller uses fallback (e.g., reason about root node only) |
| `CompressionPreconditionError` | Chain has unconfirmed nodes, wrong rel types, or < 2 nodes | Caller waits for nodes to reach CONFIRMED, or selects different chain |
| `TierProtectionError` | Attempting to compress or re-depth Tier 0/1 nodes | Not recoverable — design constraint |
| `QueryTimeoutError` | Extraction exceeds time budget | Return partial subgraph with `budget.time_exhausted = True` |
| ★ `DepthCapExceededError` | Compression would exceed `max_allowed_causal_depth` | Caller uses existing abstraction level (GPT-5 Cond 4) |
| ★ `NotAnAbstractionError` | `dissolve_abstraction` called on non-abstract node | Caller checks source field first (Gemini Cond C) |

---

## 4. Data Structures

### 4.1 SubgraphBudget

```python
class SubgraphBudget(BaseModel):
    """Complexity budget for a single subgraph extraction.

    The Predictive Loop allocates a budget per prediction tick.
    The Adversary Simulator allocates a budget per test variant.

    ★ Budget is deterministic: extraction result ALWAYS satisfies
    node_count ≤ max_nodes AND rel_count ≤ max_relationships.
    No nodes receive automatic priority override. (GPT-5 Cond 2)
    """
    model_config = ConfigDict(extra="forbid")

    max_nodes: int = Field(
        default=100,
        ge=1,
        le=10000,
        description="Maximum nodes in the extracted subgraph.",
    )
    max_relationships: int = Field(
        default=500,
        ge=1,
        le=50000,
        description="Maximum relationships in the extracted subgraph.",
    )
    max_depth: int = Field(
        default=5,
        ge=1,
        le=20,
        description=(
            "Maximum traversal depth from the query node. "
            "Deeper = more context but more expensive."
        ),
    )
    time_limit_ms: int = Field(
        default=50,
        ge=1,
        le=5000,
        description=(
            "Maximum wall-clock time for extraction in milliseconds. "
            "Phase 1 target: < 50ms for budget ≤ 100 nodes."
        ),
    )
    allow_abstraction_fallback: bool = Field(
        default=True,
        description=(
            "If the concrete subgraph exceeds max_nodes, automatically "
            "fall back to higher causal_depth abstractions. If False, "
            "the extraction truncates and returns a partial result."
        ),
    )
```

### 4.2 BudgetedSubgraph

```python
class BudgetedSubgraph(BaseModel):
    """A CausalSubgraph with budget consumption metadata.

    Extends CDD-01's CausalSubgraph with information about how
    much of the budget was used and whether abstraction fallback
    was triggered.
    """
    model_config = ConfigDict(extra="forbid")

    subgraph: CausalSubgraph
    budget_used: BudgetConsumption
    abstraction_triggered: bool = Field(
        default=False,
        description="True if concrete graph exceeded budget and abstractions were included.",
    )
    effective_causal_depth_range: tuple[int, int] = Field(
        ...,
        description="(min_depth, max_depth) of nodes in the result.",
    )
    truncated: bool = Field(
        default=False,
        description="True if extraction was cut short by budget before completion.",
    )
    extraction_strategy: str = Field(
        default="relevance_first",
        description=(
            "Which strategy was used: 'relevance_first', 'breadth_first', "
            "'abstraction_only', ★ 'root_node_only' (starvation fallback)."
        ),
    )


class BudgetConsumption(BaseModel):
    """How much of the allocated budget was actually consumed."""
    model_config = ConfigDict(extra="forbid")

    nodes_used: int
    nodes_limit: int
    relationships_used: int
    relationships_limit: int
    depth_reached: int
    depth_limit: int
    time_ms: float
    time_limit_ms: int
    time_exhausted: bool = False
```

### 4.3 CompressionResult

★ Changes from v0.1: Added confidence traceability fields (Claude Obs 2, all reviewers confirmed).

```python
class CompressionResult(BaseModel):
    """Outcome of a causal chain compression operation."""
    model_config = ConfigDict(extra="forbid")

    abstract_node_id: str
    abstract_node: WorldNode
    causal_depth: int
    source_chain_ids: list[str]
    abstracts_relationships: list[str]  # Relationship IDs for ABSTRACTS edges
    compressed_relationship_types: list[str]  # What rel types were in the original chain

    # ★ Confidence traceability (Claude Obs 2, GPT-5 v2 confirmed)
    chain_confidences: list[float] = Field(                             # ★ NEW
        ...,
        description="Individual confidences of source chain nodes at compression time.",
    )
    confidence_method: str = Field(                                     # ★ NEW
        default="min",
        description="Method used: 'min' (Phase 1) | 'geometric_mean' (Phase 2+).",
    )
    computed_confidence: float = Field(                                  # ★ NEW
        ...,
        description="The resulting abstract node confidence from the method.",
    )
```

### ★ 4.4 DissolutionResult (NEW — Gemini Cond C)

```python
class DissolutionResult(BaseModel):
    """Outcome of dissolving an abstraction."""
    model_config = ConfigDict(extra="forbid")

    abstract_node_id: str
    children_restored: list[str]         # Node IDs of concrete chain members
    abstracts_rels_removed: int          # ABSTRACTS relationships soft-deleted
    derived_rels_removed: int            # Mirrored boundary rels soft-deleted
    bsm_reconciliation_triggered: bool   # ★ Whether BSM event emitted (Grok v2)
```

### ★ 4.5 FoundationCheckResult (NEW — Gemini B + Grok 1)

```python
class FoundationCheckResult(BaseModel):
    """Outcome of checking an abstract node's foundation."""
    model_config = ConfigDict(extra="forbid")

    abstract_node_id: str
    children_checked: int
    children_refuted: int
    previous_confidence: float
    recalculated_confidence: float
    action_taken: str  # "none" | "confidence_updated" | "demotion_triggered" | "dissolution_flagged"
```

### 4.6 GraphComplexityStats

★ Changes from v0.1: Added abstraction-specific metrics (Grok Cond 4).

```python
class GraphComplexityStats(BaseModel):
    """Current complexity metrics for the World Model graph.

    ★ Cached with TTL (default 60s). Stale reads acceptable
    for dashboard; force-refresh available for tests. (Q7 consensus)

    Fed to CDD-11 (Observability) dashboard.
    """
    model_config = ConfigDict(extra="forbid")

    total_nodes: int
    total_relationships: int
    active_nodes: int  # Not soft-deleted
    active_relationships: int
    nodes_by_belief_state: dict[str, int]
    nodes_by_causal_depth: dict[int, int]
    max_causal_depth: int
    avg_node_degree: float  # Average number of relationships per node
    max_node_degree: int
    abstract_node_count: int  # Nodes with causal_depth > 2
    compression_ratio: float  # abstract_nodes / total_active_nodes
    largest_connected_component_size: int
    orphan_node_count: int  # Nodes with zero relationships

    # ★ Abstraction lifecycle metrics (Grok Cond 4)
    abstractions_dissolved_total: int = 0                               # ★ NEW
    foundation_checks_triggered: int = 0                                # ★ NEW
    active_abstraction_chains: int = 0                                  # ★ NEW
    avg_chain_length: float = 0.0                                       # ★ NEW
```

### 4.7 ABSTRACTS Relationship Type & Schema Extensions

CDD-08 introduces a new meta-relationship type that does not exist in the v2.1 §4.2 vocabulary:

```python
# Extension to CDD-01 RelationshipType enum

ABSTRACTS = "abstracts"
# Meta-relationship: (abstract_node)-[ABSTRACTS]->(concrete_node)
# Created by compress_chain(). Not a causal relationship —
# it is a structural relationship for hierarchical navigation.
# Does NOT participate in causal reasoning, prediction, or decay.
# Does NOT count toward CDD-08's relationship budget.
```

★ **Schema extensions to CDD-01 CausalRelationship** (GPT-5 Cond 1 + Q2):

```python
# Added to CDD-01 CausalRelationship model

is_structural: bool = Field(                                            # ★ NEW (Q2 consensus)
    default=False,
    description=(
        "True for ABSTRACTS and other non-causal structural relationships. "
        "Structural relationships are excluded from causal reasoning, "
        "prediction, and decay."
    ),
)

derived_from_chain: bool = Field(                                       # ★ NEW (GPT-5 Cond 1)
    default=False,
    description=(
        "True if this relationship was auto-created by compress_chain() "
        "to mirror boundary connections at the abstract level. "
        "MUST be excluded from: calibration counting (CDD-06), "
        "adversarial validation counting (CDD-07), and "
        "independent evidence tallying (CDD-04 prediction). "
        "Prevents epistemic double-counting from compression."
    ),
)
```

**Justification:** The 12 relationship types in v2.1 §4.2 are all causal. ABSTRACTS is structural — it links abstraction levels, not causal dependencies. Keeping it in the same enum with `is_structural=True` avoids polymorphic schema explosion while clearly distinguishing it from causal relationships. (Q2 — GPT-5 recommendation adopted.)

**Tier Protection:** ABSTRACTS relationships are auto-created by `compress_chain()` and ★ auto-deleted when abstract nodes are soft-deleted (Claude Obs 1 cascade rule). They cannot be manually created or modified.

★ **ABSTRACTS and Tier 0/1 boundary interactions** (Grok Cond 5): ABSTRACTS relationships and mirrored boundary relationships NEVER modify Tier 0/1 nodes. Tier protection from CDD-02 §9.4 is preserved: abstract structural relationships are read-only references, not modifications. If compression creates a mirrored boundary relationship connecting an abstract node to a Tier 0/1 node (because the original chain connected to an axiom), that mirrored relationship is `derived_from_chain=True` and tagged as structural. It does NOT alter the Tier 0/1 node's properties, belief state, or confidence.

---

## 5. Algorithm / Logic Flow

### 5.1 Budgeted Subgraph Extraction

This is the core algorithm — the one the Predictive Loop calls every tick.

★ **Snapshot isolation** (GPT-5 observation): `extract_subgraph()` operates within a single Neo4j read transaction. This guarantees snapshot isolation — concurrent compression operations do not affect the extraction in progress. This matches CDD-01's existing transactional model.

★ **Budget determinism** (GPT-5 Cond 2): No nodes receive automatic priority override. Tier 0 axiom nodes compete for relevance like any other node. Their inherently high salience (≈1.0 from CDD-02 loader) means they are naturally included in most reasonably-sized budgets, but the budget contract is deterministic: result ≤ max_nodes, always.

★ **Starvation fallback** (Grok Cond 3): If `extract_subgraph()` returns `time_exhausted=True` for 3 consecutive calls from the same caller, emit `causal.budget_starvation_warning` event. Next call automatically uses `extraction_strategy="root_node_only"` (returns query node + immediate neighbors only). Starvation mode persists until a successful normal extraction completes.

```
extract_subgraph(query_node_id, budget) called
    │
    ├── Validate: query node exists in graph
    ├── ★ Check starvation state for caller → if starvation, use root_node_only strategy
    ├── ★ Begin read transaction (snapshot isolation — GPT-5 obs)
    ├── Initialize: visited = set(), result_nodes = {}, result_rels = []
    ├── Initialize: priority_queue = [(relevance_score(query_node), query_node)]
    ├── Start timer
    │
    ▼
Phase 1: Relevance-First Expansion (concrete graph)
    │
    WHILE priority_queue NOT empty
      AND len(result_nodes) < budget.max_nodes
      AND len(result_rels) < budget.max_relationships
      AND current_depth ≤ budget.max_depth
      AND elapsed_ms < budget.time_limit_ms:
    │
    ├── Pop highest-relevance node from queue
    ├── If node.confidence < min_confidence: skip
    ├── If max_causal_depth set AND node.causal_depth > max_causal_depth: skip
    ├── Add node to result_nodes
    ├── Mark visited
    │
    ├── Query neighbors via GraphStore.query_causal_neighborhood(node, depth=1)
    │     (Filtered by rel_types, direction, is_deleted=false)
    │
    ├── For each neighbor not in visited:
    │     ├── Compute relevance_score(neighbor, query_node, current_depth)
    │     ├── Push to priority_queue with score
    │     └── Add connecting relationship to result_rels
    │
    └── Increment current_depth if expanding to new ring
    │
    ▼
Phase 2: Abstraction Fallback (if budget exceeded AND allow_abstraction_fallback)
    │
    IF budget exceeded AND include_abstractions:
    │
    ├── Identify nodes in result that have ABSTRACTS parents
    ├── ★ For each FULLY-CONTAINED cluster of concrete nodes with a shared
    │     abstract parent (all children present — GPT-5 obs):
    │     ├── Remove concrete nodes from result
    │     ├── Add abstract parent instead (saves N-1 node slots)
    │     ├── ★ Rebuild relationships at abstract level (derived_from_chain=True)
    │     ├── ★ Abstract parent treated as boundary node — new connections
    │     │     outside result are NOT followed (GPT-5 obs)
    │     └── Mark abstraction_triggered = True
    │
    ├── Re-check budget compliance
    └── If still over budget: truncate by lowest relevance score
    │
    ▼
★ Track starvation state: if time_exhausted, increment consecutive_timeout_count
Return BudgetedSubgraph(
    subgraph=CausalSubgraph(root=query_node_id, nodes, rels, depth),
    budget_used=BudgetConsumption(...),
    abstraction_triggered,
    effective_causal_depth_range,
    truncated,
    extraction_strategy
)
Log: structlog "causal.subgraph_extracted" with metrics
```

### 5.2 Relevance Scoring

The relevance score determines which nodes are included first. Higher relevance = included earlier = survives budget cuts.

★ **Normalization invariant** (GPT-5 Cond 3): All relevance inputs MUST be normalized to [0.0, 1.0]. Relevance output is guaranteed [0.0, 1.0]. Defensive clamps applied to guard against implementation bugs.

```python
def compute_relevance(
    candidate: WorldNode,
    query_node: WorldNode,
    hop_distance: int,
    relationship: CausalRelationship,
) -> float:
    """Compute relevance of a candidate node to the query.

    Score in [0.0, 1.0]. Higher = more relevant.
    ★ All components clamped to [0.0, 1.0] before weighting. (GPT-5 Cond 3)

    Components (weighted sum):
    - Distance decay:   0.4 * (1 / (1 + hop_distance))
    - Confidence:       0.25 * candidate.confidence
    - Relationship str: 0.2 * relationship.strength
    - Salience:         0.15 * candidate.salience_score
    """
    # ★ Defensive normalization clamps (GPT-5 Cond 3)
    distance_score = min(1.0, max(0.0, 1.0 / (1.0 + hop_distance)))
    confidence_score = min(1.0, max(0.0, candidate.confidence))
    strength_score = min(1.0, max(0.0, relationship.strength if relationship else 0.5))
    salience_score = min(1.0, max(0.0, candidate.salience_score))

    return (
        0.40 * distance_score
        + 0.25 * confidence_score
        + 0.20 * strength_score
        + 0.15 * salience_score
    )
```

**Weight rationale:**

- **Distance (0.40):** Proximity in the causal graph is the strongest relevance signal. Immediate neighbors are almost always relevant. This aligns with CDD-02's Tier 2 "Temporal Proximity Prior."
- **Confidence (0.25):** CONFIRMED nodes contribute more reliably than TENTATIVE ones. Avoids building reasoning chains on shaky foundations.
- **Relationship strength (0.20):** Strong causal links matter more than weak correlations. A `CAUSES` with strength 0.9 is more relevant than a `CORRELATES_WITH` at 0.3. ★ No hardcoded penalty for CORRELATES_WITH — the strength field calibration is sufficient. (Q5 — GPT-5 recommendation adopted.)
- **Salience (0.15):** The Salience Engine scores contextual importance. This lets the system attend to "what matters now" rather than just structural proximity.

These weights are configurable via `ComplexityManagerConfig`. ★ Global for Phase 1. Domain override support stubbed but not active. (Q1 consensus.)

### 5.3 Causal Depth Assignment

```
assign_causal_depth(node_id, depth) called
    │
    ├── Validate: node exists
    ├── Check: node.source NOT IN (SEED_ONTOLOGY_T0, SEED_ONTOLOGY_T1)
    │     → TierProtectionError if protected
    │
    ├── Validate: depth ≥ 0
    ├── ★ Validate: depth ≤ max_allowed_causal_depth (GPT-5 Cond 4)
    │     → DepthCapExceededError if exceeded
    ├── If depth < node.causal_depth:
    │     → Log warning: "depth_demotion" (unusual but allowed for Tier 2+)
    │
    ├── GraphStore.update_node(node_id, {causal_depth: depth}, expected_version)
    ├── Log: structlog "causal.depth_assigned" with node_id, old_depth, new_depth, reason
    │
    ▼
Return updated WorldNode
```

**Initial depth assignment rules:**

| Node Source | Initial causal_depth | Assigned By |
|------------|---------------------|-------------|
| `seed-ontology-t0` | 0 | CDD-02 loader (immutable) |
| `seed-ontology-t1` | 1 | CDD-02 loader (immutable) |
| `seed-ontology-t2` | 2 | CDD-02 loader |
| `learned` (from observation) | 0 | CDD-04 Predictive Loop (raw observations are depth 0) |
| `simulated` → `learned` (promoted) | 0 | CDD-05 Quarantine promotion (starts at raw level) |
| Compressed abstractions | max(chain_depths) + 1 | CDD-08 compress_chain() |

### 5.4 Causal Chain Compression

★ Changes from v0.1: Depth cap enforcement, `derived_from_chain` on mirrored rels, confidence traceability, tier protection clarification for ABSTRACTS, foundation validation integration.

```
compress_chain(chain_node_ids, abstract_label, abstract_description) called
    │
    ├── Validate: len(chain_node_ids) ≥ 2
    ├── Validate: all nodes exist and are not soft-deleted
    ├── Validate: all nodes are CONFIRMED belief state
    ├── Validate: no Tier 0 or Tier 1 nodes in chain
    ├── ★ Validate: no existing abstract node already ABSTRACTS these exact nodes
    │     (idempotency check — prevents duplicate abstractions)
    │
    ├── Validate: chain forms a connected path via
    │     CAUSES or PROBABILISTICALLY_CAUSES relationships
    │     (walk the chain and verify edges exist)
    │
    ├── Compute: abstract_depth = max(n.causal_depth for n in chain) + 1
    ├── ★ Validate: abstract_depth ≤ max_allowed_causal_depth (GPT-5 Cond 4)
    │     → DepthCapExceededError if exceeded
    │     → Log: structlog "causal.depth_cap_reached"
    │
    ├── ★ Collect chain_confidences = [n.confidence for n in chain] (Claude Obs 2)
    ├── ★ Compute confidence using confidence_inheritance_mode (default: "min")
    │     Phase 1: min(chain_confidences)
    │     Phase 2+: geometric_mean(chain_confidences) [documented upgrade path]
    │
    ├── Create abstract WorldNode:
    │     node_type: "abstraction"
    │     label: abstract_label
    │     description: abstract_description
    │     confidence: computed_confidence
    │     belief_state: CONFIRMED  # Inherits from all-confirmed precondition
    │     source: "compressed"
    │     causal_depth: abstract_depth
    │     epistemic_uncertainty: max(n.epistemic_uncertainty for n in chain)
    │     aleatoric_uncertainty: max(n.aleatoric_uncertainty for n in chain)
    │     domain_scope: intersection(n.domain_scope for n in chain)
    │
    ├── Create ABSTRACTS relationships:
    │     (abstract_node)-[ABSTRACTS]->(chain_node) for each chain member
    │     ★ All ABSTRACTS relationships: is_structural=True (Q2 consensus)
    │
    ├── Mirror causal relationships at abstract level:
    │     For relationships entering/exiting the chain boundary:
    │       Create equivalent relationships to/from the abstract node
    │       with strength = product of chain relationship strengths
    │       ★ All mirrored rels: derived_from_chain=True (GPT-5 Cond 1)
    │       ★ All mirrored rels: is_structural=False (they are causal but derived)
    │       ★ Mirrored rels connecting to Tier 0/1 nodes: read-only reference,
    │         DOES NOT modify the Tier 0/1 node (Grok Cond 5)
    │
    ├── GraphStore transactions: single atomic operation
    │     (abstract node creation + all ABSTRACTS + mirrored rels)
    │
    ├── Log: structlog "causal.chain_compressed" with
    │     chain_ids, abstract_id, depth, ★ chain_confidences,
    │     ★ confidence_method, ★ computed_confidence (Grok Cond 4)
    │
    ▼
Return CompressionResult(
    ...,
    ★ chain_confidences=chain_confidences,
    ★ confidence_method=config.confidence_inheritance_mode,
    ★ computed_confidence=computed_confidence,
)
```

**Compression triggers (Phase 1):**

Compression is NOT automatic in Phase 1. It is triggered manually by the architect or by explicit API call. Phase 2+ will add automatic compression when:
- A chain has been validated > N times (configurable, default: 10)
- All nodes in chain are CONFIRMED for > M decay cycles (default: 50)
- Chain length exceeds configurable threshold (default: 5 nodes)
- ★ Chain appears in > 20% of extractions (cognitive load trigger — GPT-5 Q3 addition)

Phase 1 focus is on getting the extraction and hierarchy navigation right. Automatic compression adds complexity that can wait.

### ★ 5.5 Abstraction Dissolution (NEW — Gemini Cond C)

```
dissolve_abstraction(abstract_node_id) called
    │
    ├── Validate: node exists and has source="compressed"
    │     → NotAnAbstractionError if not abstract
    ├── Validate: node has ABSTRACTS relationships
    │
    ├── ★ Begin write transaction (atomic — GPT-5 v2)
    │
    ├── Collect: children = all nodes targeted by ABSTRACTS from this node
    ├── Soft-delete all mirrored boundary relationships (derived_from_chain=True)
    ├── Soft-delete all ABSTRACTS relationships from this node
    ├── Soft-delete the abstract node itself
    ├── Concrete chain remains untouched — original nodes and rels preserved
    │
    ├── ★ Emit BSM reconciliation event for boundary concrete nodes
    │     to maintain downstream consistency (Grok v2 addition)
    │
    ├── ★ Commit transaction (all-or-nothing)
    │     On failure: full rollback, no partial deletes
    │
    ├── Log: structlog "causal.abstraction_dissolved" with
    │     abstract_id, child_ids, rels_removed
    │
    ▼
Return DissolutionResult(
    abstract_node_id,
    children_restored=child_ids,
    abstracts_rels_removed=count,
    derived_rels_removed=count,
    bsm_reconciliation_triggered=True,
)
```

### ★ 5.6 Foundation Validation Hook (NEW — Gemini B + Grok 1)

Called by CDD-03's BSM reconciliation when a concrete node with ABSTRACTS parents is demoted.

```
check_abstraction_foundation(abstract_node_id) called
    │
    ├── Validate: node exists, source="compressed"
    │
    ├── Query: all children via ABSTRACTS relationships
    ├── For each child: read current confidence and belief_state
    │
    ├── ★ If ANY child is REFUTED:
    │     ├── Flag abstract node for dissolution
    │     ├── action_taken = "dissolution_flagged"
    │     └── Caller (BSM) invokes dissolve_abstraction()
    │
    ├── ★ Otherwise: recalculate confidence from children
    │     Using current confidence_inheritance_mode (Phase 1: min)
    │     If recalculated_confidence < confirmed_decay_threshold (0.40 from CDD-06):
    │       ├── Trigger BSM demotion evaluation of abstract node
    │       └── action_taken = "demotion_triggered"
    │     Else if recalculated != previous:
    │       ├── Update abstract node confidence
    │       └── action_taken = "confidence_updated"
    │     Else:
    │       └── action_taken = "none"
    │
    ├── Log: structlog "causal.foundation_check_triggered" with
    │     abstract_id, children_checked, action_taken
    │
    ▼
Return FoundationCheckResult(...)
```

**Integration with CDD-03:** After any BSM demotion, `reconcile_after_decay` checks for ABSTRACTS parents of the demoted node. If found, it calls `check_abstraction_foundation()` for each abstract parent. This is synchronous — same pattern as CDD-01 §5.4's BSM reconciliation. CDD-03 v1.1 gets a one-line addition: "After any demotion, check for ABSTRACTS parents and trigger foundation re-evaluation via CDD-08."

### 5.7 Hierarchical Query Navigation

```
get_abstraction_hierarchy(node_id, direction="up") called
    │
    ├── Validate: node exists
    │
    ├── If direction == "up":
    │     ├── Query: MATCH (abstract)-[r:ABSTRACTS]->(n {node_id: $id})
    │     ├── Recurse: for each abstract node, find its own parents
    │     ├── Depth limit: max_abstraction_hierarchy_depth (default: 10)
    │     └── Return: list sorted by causal_depth descending
    │
    ├── If direction == "down":
    │     ├── Query: MATCH (n {node_id: $id})-[r:ABSTRACTS]->(concrete)
    │     ├── Recurse: for each concrete node, find its own children
    │     ├── Depth limit: max_abstraction_hierarchy_depth (default: 10)
    │     └── Return: list sorted by causal_depth ascending
    │
    ▼
Return list[WorldNode]
```

---

## 6. Integration Points

★ Changes from v0.1: Added mixed-resolution contract, calibration scope, budget allocation split, tier protection cross-ref, dissolution BSM event.

| Component | What It Uses From CDD-08 | How |
|-----------|--------------------------|-----|
| **CDD-01: GraphStore** | ABSTRACTS relationship type extension, ★ `is_structural` field, ★ `derived_from_chain` field | Added to RelationshipType enum and CausalRelationship model |
| **CDD-02: Seed Ontology** | Tier protection rules respected by depth assignment | TierProtectionError on Tier 0/1. ★ ABSTRACTS may reference Tier 0/1 nodes as boundary neighbors but cannot modify their properties (Grok Cond 5) |
| **CDD-03: BSM** | Confidence thresholds gate compression eligibility. ★ Foundation validation hook | Only CONFIRMED nodes can be compressed. ★ After any demotion, BSM checks for ABSTRACTS parents and triggers `check_abstraction_foundation()` (Gemini B + Grok 1) |
| **CDD-04: Predictive Loop** | `extract_subgraph()` — called every tick for prediction context | Budget allocated per tick from loop timing budget. ★ **Mixed-resolution contract:** BudgetedSubgraph may contain nodes at multiple `causal_depth` levels. CDD-04 MUST handle cross-depth relationships by treating abstract nodes as semantic equivalents of their concrete children for prediction. If abstract + concrete children both appear, prefer concrete (higher resolution). ★ CDD-04 MUST filter `derived_from_chain=True` relationships from independent evidence counting. ★ CDD-04 MAY implement adaptive budget allocation — CDD-08 provides the tool, CDD-04 owns allocation strategy (Gemini A, GPT-5 Cond 1, Grok Cond 3) |
| **CDD-06: Formal Semantics** | `min_confirmed_confidence` gates abstraction eligibility. ★ Abstract node calibration scope | ★ Abstract nodes enter CDD-06 calibration scope ONLY if they are used in a prediction context by CDD-04. Unused abstract nodes are exempt from calibration. (GPT-5 obs + Grok 2) |
| **CDD-07: Adversary Simulator** | `extract_subgraph()` — called for generating adversarial variants | Adversary operates on bounded subgraphs. ★ Must filter `derived_from_chain=True` from adversarial validation counting (GPT-5 Cond 1) |
| **CDD-09: Physics Simulator** | Observation nodes created at causal_depth 0 | Raw observations feed into the hierarchy from the bottom |
| **CDD-10: DriftMonitor** | `compute_graph_statistics()` — graph-level metrics | Monitors growth rate, compression ratio, orphan count, ★ abstraction lifecycle metrics (Grok Cond 4) |
| **CDD-11: Observability** | `GraphComplexityStats` metrics, BudgetConsumption per extraction | Dashboard displays complexity trends and budget utilization. ★ Receives compression/dissolution/foundation_check events (Grok Cond 4) |

---

## 7. Configuration Parameters

★ Changes from v0.1: Added depth cap, cache TTL, confidence mode, domain weight stub, starvation limit, Phase 2 extraction frequency trigger.

```python
class ComplexityManagerConfig(BaseSettings):
    """Configuration for the Causal Complexity Manager."""

    # ── Budget Defaults ────────────────────────────────────────────
    default_max_nodes: int = 100
    default_max_relationships: int = 500
    default_max_depth: int = 5
    default_time_limit_ms: int = 50
    budget_starvation_consecutive_limit: int = 3                        # ★ NEW (Grok Cond 3)

    # ── Relevance Weights ──────────────────────────────────────────
    relevance_weight_distance: float = 0.40
    relevance_weight_confidence: float = 0.25
    relevance_weight_strength: float = 0.20
    relevance_weight_salience: float = 0.15
    # ★ Phase 2+: domain-specific overrides (Q1 consensus — global for Phase 1)
    # relevance_weights_per_domain: dict[str, dict] | None = None

    # ── Compression ────────────────────────────────────────────────
    min_chain_length_for_compression: int = 2
    max_chain_length_for_compression: int = 20
    confidence_inheritance_mode: str = "min"                            # ★ NEW (§H resolution)
    # Phase 2+: confidence_inheritance_mode = "geometric_mean"
    # Phase 2+ automatic compression:
    # auto_compress_validation_count: int = 10
    # auto_compress_stable_cycles: int = 50
    # auto_compress_chain_length_threshold: int = 5
    # ★ auto_compress_extraction_frequency: float = 0.20  # GPT-5 Q3 addition

    # ── Abstraction Lifecycle ──────────────────────────────────────
    abstraction_fallback_enabled: bool = True
    max_abstraction_hierarchy_depth: int = 10
    max_allowed_causal_depth: int = 10                                  # ★ NEW (GPT-5 Cond 4)

    # ── Caching ────────────────────────────────────────────────────
    graph_stats_cache_ttl_seconds: int = 60                             # ★ NEW (Q7 consensus)

    # ── Performance ────────────────────────────────────────────────
    stress_test_node_count: int = 10_000
    stress_test_relationship_count: int = 50_000
    stress_test_extraction_target_ms: int = 100
    partitioning_benchmark_target_nodes: int = 100_000  # Phase 7 projection

    model_config = SettingsConfigDict(env_prefix="ARIA_CCM_")
```

---

## 8. Error Handling & Edge Cases

★ Changes from v0.1: Tier 0 override removed, starvation fallback added, soft-delete cascade, dissolution edge cases.

| Edge Case | Behavior | Rationale |
|-----------|----------|-----------|
| Query node has zero relationships | Return subgraph with just the query node | Valid result — isolated node is still a valid subgraph |
| Budget.max_nodes = 1 | Return only the query node, no relationships | Degenerate case but valid — caller gets minimal context |
| Entire graph fits within budget | Return full graph, no abstraction needed | Phase 1 likely scenario — small graph |
| Abstraction fallback still exceeds budget | Truncate by relevance score, set `truncated=True` | Caller knows result is incomplete |
| compress_chain with circular path | Reject — chain must be a DAG path (validated by CDD-03 INV-02) | Circular compression would create logical contradiction |
| Compress same chain twice | Reject — if abstract node already ABSTRACTS these nodes | Idempotency check prevents duplicate abstractions |
| Abstract node has confidence decay below threshold | BSM demotes abstract node independently of source chain | Abstract nodes participate in normal epistemic lifecycle |
| ★ Source chain node demoted after compression | ★ Foundation validation hook triggers — abstract confidence recalculated, possibly demoted or dissolved (Gemini B + Grok 1) | Prevents zombie abstractions |
| ABSTRACTS relationships during budget counting | NOT counted against relationship budget | Meta-relationships are structural, not causal — don't consume reasoning budget |
| Time budget exhausted mid-extraction | Return partial result with `time_exhausted=True` | Caller can choose to retry with larger budget or use partial result |
| ★ Tier 0 node in extraction path | Competes for relevance like any other node; high salience ensures natural inclusion (GPT-5 Cond 2) | Budget determinism preserved |
| Graph has disconnected components | Each extraction is rooted at query node — other components not reached | By design — extraction is query-local, not global |
| ★ 3 consecutive extraction timeouts | Enter starvation mode: next extraction uses root_node_only strategy (Grok Cond 3) | Prevents I/O death spiral in Predictive Loop |
| ★ Compression would exceed max_allowed_causal_depth | Reject with DepthCapExceededError (GPT-5 Cond 4) | Prevents abstraction ladder explosion |
| ★ Soft-deleting abstract node | Cascade: auto soft-delete all ABSTRACTS rels and derived_from_chain=True rels from that node. Cascade is idempotent. (Claude Obs 1, GPT-5 v2) | Prevents orphaned structural edges |
| ★ Dissolution failure mid-transaction | Full rollback — no partial deletes (GPT-5 v2) | Atomic dissolution prevents graph corruption |
| ★ Foundation check finds REFUTED child | Abstract node flagged for dissolution (Gemini B + Grok 1) | Abstractions must have valid foundations |

---

## 9. Module Structure

★ Changes from v0.1: Added dissolution.py.

```
src/aria/causal/
├── __init__.py
├── protocols.py          # CausalComplexityManager Protocol
├── subgraph.py           # Budgeted subgraph extraction (§5.1, §5.2)
├── abstraction.py        # Causal chain compression, hierarchy navigation (§5.3, §5.4, §5.7)
├── dissolution.py        # ★ Abstraction dissolution & foundation validation (§5.5, §5.6)
├── budget.py             # SubgraphBudget, BudgetConsumption, allocation strategies
├── stats.py              # GraphComplexityStats computation + ★ TTL caching
└── config.py             # ComplexityManagerConfig
```

---

## 10. Performance Targets

| Metric | Phase 1 Target | Measurement |
|--------|---------------|-------------|
| Subgraph extraction (≤100 nodes) | < 50ms | `budget_used.time_ms` |
| Subgraph extraction (≤500 nodes) | < 200ms | Stress test |
| Stress test: 10K nodes, 50K rels | < 100ms for budget=100 extraction | `tests/stress/test_complexity_stress.py` |
| Graph partitioning benchmark: 100K nodes (projected) | < 500ms for budget=100 extraction | Benchmark script (measures scaling curve) |
| Compression (chain of 5 nodes) | < 20ms | Atomic transaction duration |
| ★ Dissolution | < 15ms | Atomic transaction duration |
| Hierarchy navigation (depth 10) | < 10ms | Recursive ABSTRACTS query |
| Graph statistics computation | < 100ms at 10K nodes | Aggregate Cypher queries |
| ★ Foundation check | < 5ms per abstract node | Single-hop ABSTRACTS query + confidence calc |

**Profiling approach:** All extraction calls emit `structlog` events with `time_ms`. Percentile tracking (p50, p95, p99) is enabled by CDD-11. If p95 exceeds 2x target, investigation is logged but not blocking in Phase 1.

---

## 11. Test Plan

### 11.1 Unit Tests (`tests/unit/test_causal/`)

★ Changes from v0.1: Added normalization, depth cap, dissolution, foundation check tests. Updated weight tolerance.

| Test | Validates |
|------|-----------|
| `test_budget_defaults` | SubgraphBudget defaults match config |
| `test_budget_validation` | Invalid budgets rejected (max_nodes < 1, etc.) |
| `test_relevance_score_range` | Score always in [0.0, 1.0] |
| `test_relevance_distance_dominates` | Closer nodes score higher than far nodes (all else equal) |
| ★ `test_relevance_weights_sum_to_one` | Config weights sum to 1.0 with float tolerance (abs(sum - 1.0) < 1e-3). Also assert each weight ∈ [0,1]. (Claude Obs 3, GPT-5 v2) |
| ★ `test_relevance_normalization_clamps` | Inputs > 1.0 or < 0.0 are clamped. Output always ∈ [0,1]. (GPT-5 Cond 3) |
| `test_compression_result_fields` | CompressionResult captures all required data |
| ★ `test_compression_result_confidence_traceability` | chain_confidences, confidence_method, computed_confidence all recorded (Claude Obs 2, Grok v2) |
| `test_graph_stats_computation` | Stats computed correctly from known graph |
| `test_tier_protection_on_depth_assign` | Tier 0/1 depth assignment raises TierProtectionError |
| `test_depth_assignment_tier2` | Tier 2+ nodes accept depth changes |
| `test_abstracts_not_counted_in_budget` | ABSTRACTS relationships excluded from budget counting |
| ★ `test_depth_cap_enforced` | Compression rejected when depth > max_allowed_causal_depth (GPT-5 Cond 4) |
| ★ `test_derived_from_chain_flag` | Mirrored boundary rels have derived_from_chain=True (GPT-5 Cond 1) |
| ★ `test_is_structural_flag` | ABSTRACTS rels have is_structural=True (Q2) |
| ★ `test_dissolution_result_fields` | DissolutionResult captures all required data (Gemini Cond C) |
| ★ `test_foundation_check_result_fields` | FoundationCheckResult captures all required data (Gemini B + Grok 1) |

### 11.2 Integration Tests (`tests/integration/test_causal_integration.py`)

★ Changes from v0.1: Tier 0 test updated, added dissolution, foundation, starvation, snapshot, cascade, e2e tests.

| Test | Validates |
|------|-----------|
| `test_extract_simple_chain` | Linear A→B→C chain extracted within budget |
| `test_extract_with_budget_limit` | Budget=2 nodes → only 2 most relevant returned |
| `test_extract_with_confidence_filter` | min_confidence filters out low-confidence nodes |
| `test_extract_with_depth_filter` | max_causal_depth limits abstraction levels |
| `test_abstraction_fallback` | Budget exceeded → abstract nodes substituted for concrete clusters |
| ★ `test_abstraction_fallback_fully_contained` | Only fully-contained clusters replaced (GPT-5 obs) |
| `test_compression_creates_abstract_node` | compress_chain creates node + ABSTRACTS rels in Neo4j |
| `test_compression_preconditions_enforced` | Unconfirmed, wrong rel type, too short → rejected |
| `test_hierarchy_navigation_up` | get_abstraction_hierarchy("up") returns abstract parents |
| `test_hierarchy_navigation_down` | get_abstraction_hierarchy("down") returns concrete children |
| `test_compression_atomicity` | Partial failure → entire compression rolled back |
| ★ `test_tier0_competes_on_relevance` | Tier 0 nodes included via high salience, not override. Budget=5 with 9 Tier 0 nodes → only highest-relevance ones included. (GPT-5 Cond 2) |
| `test_extract_disconnected_node` | Isolated node → subgraph with just that node |
| `test_budget_consumption_tracking` | BudgetConsumption accurately reflects actual usage |
| ★ `test_dissolution_removes_abstract` | dissolve_abstraction soft-deletes abstract + ABSTRACTS + derived rels (Gemini Cond C) |
| ★ `test_dissolution_preserves_concrete_chain` | After dissolution, original chain nodes and rels untouched (Gemini Cond C) |
| ★ `test_dissolution_atomicity` | Partial failure mid-dissolve → full rollback (GPT-5 v2) |
| ★ `test_foundation_check_refuted_child` | One child REFUTED → abstract flagged for dissolution (Gemini B + Grok 1) |
| ★ `test_foundation_check_confidence_drop` | Children's confidence drops → abstract confidence recalculated (Gemini B + Grok 1) |
| ★ `test_starvation_fallback_triggers` | 3 consecutive timeouts → root_node_only mode (Grok Cond 3) |
| ★ `test_starvation_recovery` | Successful normal extraction exits starvation mode (Grok Cond 3) |
| ★ `test_snapshot_isolation_during_compression` | Concurrent compression doesn't affect in-progress extraction (GPT-5 obs) |
| ★ `test_soft_delete_cascade` | Soft-deleting abstract node cascades to ABSTRACTS + derived rels. Cascade is idempotent. (Claude Obs 1, GPT-5 v2) |
| ★ `test_derived_rels_excluded_from_evidence` | Extraction includes derived rels but they are filterable by consumers (GPT-5 Cond 1) |
| ★ `test_end_to_end_compress_demote_dissolve` | Compress chain → run prediction (derived rels ignored) → demote child to REFUTED → foundation check triggers → dissolve → verify predictions use concrete chain. (GPT-5 v2 e2e test) |

### 11.3 Stress Tests (`tests/stress/test_complexity_stress.py`)

| Test | Validates |
|------|-----------|
| `test_10k_nodes_50k_rels_extraction` | Budget=100 extraction < 100ms at scale |
| `test_scaling_curve_1k_to_10k` | Extraction time grows sub-linearly with graph size |
| `test_100k_projection_benchmark` | Phase 7 projection: budget=100 extraction < 500ms |
| `test_deep_hierarchy_10_levels` | Hierarchy navigation at depth 10 < 10ms |
| `test_concurrent_extractions` | 10 simultaneous extractions → no deadlocks, all complete |

### 11.4 Property-Based Tests (`tests/property/test_causal_properties.py`)

| Test | Validates |
|------|-----------|
| ★ `test_extraction_never_exceeds_budget` | For any random graph + budget → result ≤ budget limits. Unconditional — no Tier 0 override exception. (GPT-5 Cond 2) |
| `test_relevance_ordering_consistent` | Same graph + same budget → same ordering (deterministic) |
| `test_compression_preserves_reachability` | After compression, paths through abstract node reach same endpoints as original chain |
| `test_tier_protection_universal` | Random tier assignments → Tier 0/1 never compressed or re-depthed |
| ★ `test_relevance_always_normalized` | For any input values → output ∈ [0.0, 1.0] (GPT-5 Cond 3) |
| ★ `test_depth_never_exceeds_cap` | For any compression sequence → max depth ≤ max_allowed_causal_depth (GPT-5 Cond 4) |

---

## 12. Acceptance Criteria

★ Changes from v0.1: Expanded from 15 → 25 criteria.

| # | Criterion | Sprint | Verification |
|---|-----------|--------|-------------|
| 1 | Subgraph extraction by relevance within budget | S3 | Integration test: budget=100, result ≤ 100 nodes |
| 2 | Complexity budget enforcement with abstraction fallback | S3 | Integration test: small budget → abstractions substituted |
| 3 | Causal depth assignment with tier protection | S3 | Unit test: Tier 0/1 rejected, Tier 2+ accepted |
| 4 | Causal chain compression creates abstract node | S3 | Integration test: compress 3-node chain → abstract node at depth+1 |
| 5 | ABSTRACTS relationship type functional | S3 | Hierarchy navigation tests pass |
| 6 | Compression preconditions enforced | S3 | Unconfirmed chain rejected, Tier 0/1 chain rejected |
| 7 | Graph growth stress test: 10K/50K < 100ms | S3 | Stress test passes |
| 8 | Graph partitioning benchmark: 100K projection | S3 | Benchmark runs, results documented |
| 9 | Hierarchical abstraction queries functional | S3 | Navigate up and down hierarchy correctly |
| 10 | GraphComplexityStats computable | S3 | Stats match known graph state |
| 11 | Budget consumption accurately tracked | S3 | BudgetConsumption matches actual result size |
| 12 | ★ Budget determinism: no Tier 0 override, result ≤ max_nodes always | S3 | Property test passes (GPT-5 Cond 2) |
| 13 | Time budget enforcement | S3 | Extraction stops within time_limit_ms + tolerance |
| 14 | Relevance scoring deterministic | S3 | Same inputs → same ordering |
| 15 | ComplexityManagerConfig loads from environment | S3 | ARIA_CCM_ prefix works |
| ★ 16 | `derived_from_chain` flag on mirrored boundary rels | S3 | Unit + integration tests (GPT-5 Cond 1) |
| ★ 17 | Relevance normalization: all inputs clamped [0,1], output [0,1] | S3 | Property test (GPT-5 Cond 3) |
| ★ 18 | Depth cap: compression rejected above max_allowed_causal_depth | S3 | Unit test (GPT-5 Cond 4) |
| ★ 19 | `dissolve_abstraction()` safely removes abstract + structural rels | S3 | Integration test (Gemini Cond C) |
| ★ 20 | Foundation validation hook: child demotion recalculates abstract confidence | S3 | Integration test (Gemini B + Grok 1) |
| ★ 21 | Starvation fallback: 3 timeouts → root_node_only mode | S3 | Integration test (Grok Cond 3) |
| ★ 22 | Compression audit events in structlog | S3 | Log assertions (Grok Cond 4) |
| ★ 23 | Abstraction lifecycle metrics in GraphComplexityStats | S3 | Stats include dissolved/foundation/active counts (Grok Cond 4) |
| ★ 24 | Soft-delete cascade for abstract nodes | S3 | Integration test (Claude Obs 1) |
| ★ 25 | End-to-end: compress → predict → demote → foundation check → dissolve | S3 | Integration test (GPT-5 v2) |

---

## 13. Open Questions — Resolved

★ All 7 original questions resolved through reviewer consensus.

| # | Question | Decision | Rationale | Voters |
|---|----------|----------|-----------|--------|
| Q1 | Domain-specific relevance weights? | Global for Phase 1. Domain override stubbed but not active. | Premature domain weighting complicates debugging. | GPT-5 + Grok + Gemini (unanimous) |
| Q2 | ABSTRACTS: single enum or MetaRelationshipType? | Single enum + `is_structural: bool` flag on CausalRelationship. | Simple, no polymorphic schema explosion. Clear structural/causal distinction via flag. | GPT-5 (recommended) + Grok + Gemini |
| Q3 | Automatic compression triggers (Phase 2)? | Original 3 triggers + extraction frequency (>20% of extractions). | Ties abstraction to cognitive load, not just epistemic validation. | GPT-5 addition, accepted by all |
| Q4 | Tier 0 budget treatment? | Count against budget like all nodes. High salience handles inclusion. | Deterministic budgets more important than axiom auto-inclusion. | GPT-5 (recommended) + Grok (2:1) |
| Q5 | CORRELATES_WITH relevance penalty? | No hardcoded penalty. Strength field calibration is sufficient. | Double-penalizing makes weak relations invisible. | GPT-5 (recommended) + all |
| Q6 | Abstract node confidence: min vs geometric mean? | min() for Phase 1. Geometric mean documented for Phase 2. | Conservative prevents overconfident abstractions. Phase 2 has data to justify switch. | GPT-5 + design intent (2:1 vs Grok) |
| Q7 | Graph statistics caching? | TTL cache, 60 seconds. | Stats don't need millisecond precision. Predictable cost at scale. | GPT-5 (recommended) + all |

---

## 14. ★ Implementation Risks for Sprint 3 (NEW — Gemini v2)

These are not Phase 1 blockers but should be monitored during implementation.

### 14.1 Shadow Graph Budget Dilution

**Risk:** If abstraction fallback inserts many `derived_from_chain=True` or `is_structural=True` nodes into a BudgetedSubgraph, the effective reasoning budget for CDD-04 shrinks because those nodes don't contribute to causal reasoning.

**Mitigation:** Monitor the ratio of structural-to-causal nodes in extraction results. If structural nodes consistently exceed 30% of budget in Phase 1, add a "reasoning_budget" concept that counts only non-derived nodes. Phase 1 physics graphs are small enough that this is unlikely to manifest.

**Source:** Gemini v2 observation.

### 14.2 BSM Cascade Latency

**Risk:** In a deep abstraction hierarchy (A → Abstract B → Abstract C), a single decay event on node A triggers synchronous foundation checks cascading up the hierarchy. Under high-frequency ticks with many simultaneous decays, this could spike BSM latency beyond the <10ms target from CDD-03.

**Mitigation:** Phase 1 has manual compression only and shallow hierarchies (depth ≤ 3 expected). If latency spikes are detected during S3 implementation, add a "lazy validation" flag that queues non-critical foundation checks for the next tick instead of blocking the current transition.

**Source:** Gemini v2 observation.

### 14.3 Confounder Visibility at Distance

**Risk:** The distance-heavy relevance scoring (0.40 weight) may prune distant but important confounders from extractions.

**Mitigation:** Phase 1 confounders are structurally modeled via `confounder_node_id` on relationships and are typically 1–2 hops from affected nodes. If confounder detection accuracy is low (measurable via CDD-06 calibration), Phase 2 adds a "confounder-seeker" expansion pass that prioritizes high-salience nodes along CORRELATES_WITH paths.

**Source:** Gemini Cond D (deferred to Phase 2, accepted by Gemini with caveat).

---

## 15. ★ Review Feedback Incorporation Record (NEW)

### GPT-5 (OpenAI) — 4 Structural Conditions + 2 v2 Additions

| Item | Resolution | Section |
|------|-----------|---------|
| Cond 1: Double-counting from mirrored rels | `derived_from_chain: bool` field added | §4.7, §5.4 |
| Cond 2: Tier 0 budget determinism | Override removed, compete on relevance | §5.1, §8 |
| Cond 3: Relevance normalization | Clamps added, invariant stated | §5.2 |
| Cond 4: Compression depth cap | `max_allowed_causal_depth = 10` added | §5.3, §5.4, §7 |
| v2: Atomic dissolve | Transactional dissolution | §5.5 |
| v2: End-to-end integration test | Added as test + AC 25 | §11.2, §12 |

### Gemini (Google) — 3 Conditions + 2 v2 Observations

| Item | Resolution | Section |
|------|-----------|---------|
| Cond A: Mixed-resolution semantics | Design note + CDD-04 blocking requirement | §6 |
| Cond B: Zombie abstractions | Foundation validation hook | §5.6 |
| Cond C: Dissolve abstraction | `dissolve_abstraction()` method added | §3.1, §5.5 |
| Cond D: Confounder-seeker | Deferred to Phase 2 (accepted with caveat) | §14.3 |
| v2: Shadow Graph budget | Documented as implementation risk | §14.1 |
| v2: BSM Cascade latency | Documented as implementation risk | §14.2 |

### Grok (xAI) — 5 Conditions + 2 v2 Polish Items

| Item | Resolution | Section |
|------|-----------|---------|
| Cond 1: BSM integration for abstracts | Foundation validation hook | §5.6 |
| Cond 2: Confidence inheritance | min() Phase 1, geometric mean Phase 2 (outvoted 2:1) | §5.4, §7 |
| Cond 3: Dynamic budget & fallback | Starvation fallback in CDD-08, allocation in CDD-04 | §5.1, §7, §8 |
| Cond 4: Audit & observability | structlog events + abstraction metrics | §5.4, §5.5, §5.6, §4.6 |
| Cond 5: Tier protection for ABSTRACTS | Explicit clarification + CDD-02 cross-ref | §4.7, §6 |
| v2: Dissolution BSM event | Reconciliation event for boundary nodes | §5.5 |
| v2: Compression traceability test | test_compression_result_confidence_traceability added | §11.1 |

### Claude Opus 4.6 — 3 Observations (All Confirmed)

| Item | Resolution | Section |
|------|-----------|---------|
| Obs 1: ABSTRACTS soft-delete cascade | Cascade rule + idempotency | §4.7, §8 |
| Obs 2: CompressionResult traceability | chain_confidences, confidence_method, computed_confidence | §4.3 |
| Obs 3: Relevance weight test tolerance | Float tolerance (1e-3) + weight ∈ [0,1] check | §11.1 |

---

*End of CDD-08: Causal Complexity Manager*

**Project ARIA · Adaptive Reasoning Integrated Architecture**  
Component Design Document 08 — Causal Attention & Abstraction Engine  
v1.0 — Approved by GPT-5 (OpenAI), Gemini (Google), Grok (xAI), Claude Opus 4.6 (Anthropic)
