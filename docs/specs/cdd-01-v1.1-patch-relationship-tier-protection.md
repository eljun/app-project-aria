# CDD-01 v1.1 Patch: Relationship Tier Protection

> **Status:** APPROVED — All Reviewing Systems Signed Off  
> **Version:** 1.1 (Patch — applies on top of CDD-01 v1.0)  
> **Triggered By:** CDD-02 review — Claude Observation 1, confirmed 🔴 critical by GPT-5  
> **Scope:** Adds tier-source enforcement to relationship mutation operations  

---

## Patch Summary

CDD-01 v1.0's `delete_relationship()` and `update_relationship()` do not check whether the source node of the relationship is a protected tier. This allows silent deletion or modification of Tier 0 defining relationships — a structural integrity breach.

This patch adds 3 changes:

1. **Tier-source checks on relationship mutations**
2. **New exception: CrossTierReferenceError**
3. **Updated error handling table**

---

## Change 1: Tier-Source Checks

### §3.1 GraphStore Protocol — `delete_relationship()` update

Add to docstring and implementation:

```python
def delete_relationship(self, relationship_id: str, reason: str) -> None:
    """Soft-delete a relationship.

    ★ v1.1: Before deleting, checks the source node's tier:
    - If source is seed-ontology-t0: raises ImmutableNodeError
      ("Cannot delete relationships from Tier 0 node: {source_node_id}")
    - If source is seed-ontology-t1 and hard_review_override not set:
      raises ImmutableNodeError
      ("Tier 1 relationship deletion requires hard_review_override")

    Args:
        relationship_id: UUID string of the relationship.
        reason: Human-readable reason for deletion.

    Raises:
        RelationshipNotFoundError: If relationship_id does not exist.
        ★ ImmutableNodeError: If source node is Tier 0, or Tier 1 without override.
    """
    ...
```

### §3.1 GraphStore Protocol — `update_relationship()` update

Add to docstring and implementation:

```python
def update_relationship(
    self, relationship_id: str, updates: RelationshipUpdate
) -> CausalRelationship:
    """Update properties of an existing relationship.

    ★ v1.1: Before updating, checks the source node's tier:
    - If source is seed-ontology-t0: raises ImmutableNodeError
      ("Cannot modify relationships from Tier 0 node: {source_node_id}")
    - If source is seed-ontology-t1 and hard_review_override not set:
      raises ImmutableNodeError
      ("Tier 1 relationship modification requires hard_review_override")

    Args:
        relationship_id: UUID string of the relationship.
        updates: RelationshipUpdate instance with fields to change.

    Returns:
        Updated CausalRelationship instance.

    Raises:
        RelationshipNotFoundError: If relationship_id does not exist.
        SchemaValidationError: If resulting relationship fails validation.
        ★ ImmutableNodeError: If source node is Tier 0, or Tier 1 without override.
    """
    ...
```

### Implementation pattern (both methods)

```python
# ★ v1.1 — Tier protection check before mutation
rel = self._get_raw_relationship(relationship_id)
source_node = self.get_node(rel.source_node_id)

if source_node.source == NodeSource.SEED_ONTOLOGY_T0:
    raise ImmutableNodeError(
        f"Cannot modify relationships from Tier 0 node: {source_node.node_id}"
    )
if source_node.source == NodeSource.SEED_ONTOLOGY_T1:
    if not getattr(updates, 'hard_review_override', False):
        raise ImmutableNodeError(
            f"Tier 1 relationship modification requires hard_review_override"
        )
    # Log the hard review action
    structlog.get_logger().info(
        "tier1.relationship_hard_review",
        relationship_id=relationship_id,
        source_node_id=source_node.node_id,
        operation=operation_name,
    )
```

---

## Change 2: New Exception

### §3.4 Error Hierarchy — add CrossTierReferenceError

```python
class CrossTierReferenceError(SchemaValidationError):              # ★ v1.1
    """A YAML relationship references a node_id that does not exist
    in any loaded tier's node registry.

    Raised during pre-load validation, not at Neo4j operation time.
    Provides explicit diagnostics: source file, relationship, missing target.
    """
    def __init__(
        self, source_file: str, rel_type: str, missing_node_id: str
    ) -> None:
        self.source_file = source_file
        self.rel_type = rel_type
        self.missing_node_id = missing_node_id
        super().__init__(
            f"Cross-tier reference broken in {source_file}: "
            f"{rel_type} targets non-existent node '{missing_node_id}'"
        )
```

---

## Change 3: Updated Error Handling Table

### §8.1 — add rows

| Edge Case | Behavior | Rationale |
|-----------|----------|-----------|
| ★ Delete relationship from Tier 0 source | **Reject** ImmutableNodeError | Tier 0 semantic structure is protected (v1.1) |
| ★ Update relationship from Tier 0 source | **Reject** ImmutableNodeError | Tier 0 semantic structure is protected (v1.1) |
| ★ Delete relationship from Tier 1 source without override | **Reject** ImmutableNodeError | Tier 1 requires hard review (v1.1) |
| ★ Cross-tier reference to non-existent node in YAML | **Reject** CrossTierReferenceError | Pre-load validation catches broken refs (v1.1) |

---

## Tests Added

| Test | Validates |
|------|-----------|
| `test_delete_relationship_tier0_blocked` | Deleting relationship with Tier 0 source raises ImmutableNodeError |
| `test_update_relationship_tier0_blocked` | Updating relationship with Tier 0 source raises ImmutableNodeError |
| `test_delete_relationship_tier1_requires_override` | Deleting Tier 1 source relationship without override raises error |
| `test_delete_relationship_tier1_with_override` | Deleting Tier 1 source relationship with override succeeds and logs |
| `test_cross_tier_reference_error_diagnostic` | CrossTierReferenceError includes file, rel_type, and missing node_id |

---

## Updated Acceptance Criteria

Add to CDD-01 acceptance criteria:

| # | Criterion | Sprint | Verification |
|---|-----------|--------|-------------|
| 26 | ★ Relationship delete/update checks source node tier | S1 | Tier 0 blocked, Tier 1 requires override |
| 27 | ★ CrossTierReferenceError in exception hierarchy | S1 | Importable, tested with diagnostics |

---

*CDD-01 v1.1 Patch — Relationship Tier Protection*  
*Approved by all reviewing systems via CDD-02 review cycle*
