# CDD-04: Predictive Loop Engine

> **Status:** APPROVED — All Reviewers Signed Off (4/4)  
> **Version:** 1.0  
> **Author:** Jun (Architect) + Claude Opus 4.6 (Design Partner)  
> **Depends On:** CDD-01 v1.1 (GraphStore), CDD-03 v1.0 (BSM), CDD-06 v1.0 (Formal Semantics), CDD-08 v1.0 (Causal Complexity Manager)  
> **Consumed By:** CDD-05 (Quarantine), CDD-07 (Adversary Simulator), CDD-10 (DriftMonitor), CDD-11 (Observability)  
> **Sprint:** S4 (Predictive Loop + Physics Simulator + Observability)  
> **Classification:** Confidential — Core Team & Designated Review Partners  
> **Reviewed By:** GPT-5 (OpenAI) · Gemini (Google) · Grok (xAI) · Claude Opus 4.6 (Anthropic)  
> **Changes from v0.1:** 28 changes marked with ★

---

## 1. Purpose & Scope

ARIA runs continuously. The predict-observe-error-update cycle is the fundamental mechanism of all learning. This is not triggered by queries — it is the permanent operating state of the system. (v2.1 §2.2)

CDD-04 specifies the engine that drives this cycle. Every tick:

1. **Predict** — Given the current World Model state and an incoming observation context, extract a relevant causal subgraph (via CDD-08) and generate a prediction of what should happen next.
2. **Observe** — Receive the actual observation from the environment (CDD-09 Physics Simulator in Phase 1) via Redis pub/sub.
3. **Error** — Compute the quantified difference between prediction and observation. This is the learning signal.
4. **Update** — Adjust the World Model: modify confidences via weighted credit assignment, trigger BSM transitions (CDD-03), record predictions for calibration (CDD-06), and optionally generate counterfactual hypotheses for quarantine (CDD-05).

CDD-04 also specifies:

**1.1 Per-Tick Timing Instrumentation** — Every tick logs a structured timing breakdown across 4 phases (graph read, prediction, error compute, graph write) for observability and performance optimization. (Impl Plan §2.6, GPT-5 condition)

**1.2 Simplified Counterfactual Generator** — Phase 1's lightweight substitute for the Dream Engine (Phase 3). When prediction error exceeds a threshold, it generates candidate hypotheses by querying the causal neighborhood for confounders, unobserved variables, and violated invariants. Outputs are tagged SIMULATED and sent to Quarantine (CDD-05). ★ Mental simulation runs exclusively on in-memory deep copies — zero GraphStore interaction. (Impl Plan §8.1, Gemini BLOCKING 3)

**1.3 Local Predictive Coding Stub Interface** — Interface contracts for Phase 2's distributed cortical hierarchy. Raises `NotImplementedError` in Phase 1 but defines the contract so Phase 2 integration is seamless. (Gemini addition, Impl Plan §6 S6)

★ **1.4 Self-Regulation Mechanisms** — The loop monitors its own epistemic health: persistent surprise escalation detects thrashing, periodic recalibration checks catch drift before collapse, and weighted credit assignment prevents catastrophic forgetting. (GPT-5 + Gemini + Grok review feedback)

### 1.5 Neuroscience Grounding

| Component | Cognitive Analogue | Reference |
|-----------|-------------------|-----------|
| Predict-Observe-Error-Update cycle | Predictive coding / Free Energy minimization | Karl Friston — the brain continuously generates predictions and updates its model to minimize prediction error (surprise) |
| Causal subgraph extraction as attention | Selective attention / Prefrontal filtering | The brain doesn't reason over all knowledge simultaneously — it focuses on the relevant causal neighborhood |
| Counterfactual generation | Hippocampal replay / Imagination | The hippocampus generates counterfactual scenarios to explain unexpected outcomes, feeding back into the world model |
| Asymmetric confidence update | Loss aversion in Bayesian updating | Disconfirming evidence is more informative than confirming evidence — mirrors how biological belief systems update |
| ★ Weighted credit assignment | Precision-weighted prediction error | In predictive coding, error signals are weighted by precision (inverse uncertainty) — uncertain predictions propagate less blame |

### 1.6 What This CDD Does NOT Cover

| Excluded Component | Covered In |
|-------------------|------------|
| World Model schema and graph operations | CDD-01 |
| Belief state transitions and invariant checking | CDD-03 |
| Confidence thresholds and calibration metrics | CDD-06 |
| Subgraph extraction and complexity budgets | CDD-08 |
| Physics simulator and observation generation | CDD-09 |
| Quarantine storage and promotion rules | CDD-05 |
| Adversarial variant generation | CDD-07 |
| Calibration drift detection | CDD-10 |
| Dashboard and metrics aggregation | CDD-11 |

---

## 2. Specification References

| Spec Section | Content | Relevance |
|-------------|---------|-----------|
| v2.1 §2.2 | The Continuous Inner Loop | Direct specification source |
| v2.1 §3.2 | Acid Test | Must pass predict→fail→dream→update cycle |
| v2.1 §3.5 | External Truth Pragmatism | Epistemic framework for updates |
| v2.1 §5.2 | Causal Complexity Manager | Subgraph extraction for predictions |
| Impl Plan §2.6 | AsyncIO Event Loop + Per-tick timing | Technology decision + instrumentation |
| Impl Plan §2.7 | Redis Pub/Sub for observations | Observation delivery mechanism |
| Impl Plan §8.1 | SimplifiedCounterfactualGenerator | Phase 1 Dream Engine substitute |
| Impl Plan §6 S4 | Sprint 4 deliverables | Acceptance criteria source |
| CDD-01 §3.1 | GraphStore Protocol | Graph read/write operations |
| CDD-01 §4.5 | CausalSubgraph model | Prediction input structure |
| ★ CDD-01 §5.4 | Confidence decay + BSM reconciliation | Passive decay ownership (not CDD-04) |
| CDD-03 §3.2 | BSM valid transitions | Triggered by confidence updates |
| CDD-03 §5 | Invariant Checker | Called on every graph write |
| ★ CDD-03 §5.1 | reconcile_after_decay | Explicit BSM reconciliation after updates |
| CDD-06 §6.2 | Brier score, ECE, prediction variance | Calibration metrics |
| CDD-06 §6.4 | Recalibration triggers | Error-driven recalibration |
| CDD-06 §6.6 | PredictionRecord | Prediction tracking schema |
| ★ CDD-06 §6.5 | Recalibration cycle | Active calibration integration |
| CDD-08 §3.1 | CausalComplexityManager.extract_subgraph | Bounded attention per tick |
| CDD-08 §4.2 | BudgetedSubgraph | Extraction result with metadata |
| CDD-08 §4.7 | derived_from_chain flag | Must filter from evidence counting |
| CDD-08 §6 | Mixed-resolution contract | Prefer concrete over abstract when both present |

---

## 3. Interface Contract

### 3.1 PredictiveLoopEngine Protocol

```python
# src/aria/predictive_loop/protocols.py

from __future__ import annotations
from typing import Protocol, AsyncIterator
from aria.world_model.schema import WorldNode, CausalSubgraph


class PredictiveLoopEngine(Protocol):
    """The continuous cognitive cycle.

    Runs as an asyncio event loop. Each tick:
    predict → observe → compute error → update model.
    """

    async def start(self) -> None:
        """Start the continuous loop.

        Subscribes to observation channels on Redis.
        Runs until stop() is called or the process is terminated.
        Emits 'loop.started' structlog event.
        """
        ...

    async def stop(self) -> None:
        """Gracefully stop the loop after the current tick completes.

        Does NOT interrupt a tick mid-execution.
        ★ Flushes PredictionRecordBuffer before shutdown.
        Emits 'loop.stopped' structlog event with total_ticks.
        """
        ...

    async def run_single_tick(
        self,
        observation: Observation,
    ) -> TickResult:
        """Execute one complete predict-observe-error-update cycle.

        This is the atomic unit of cognition. Exposed for testing —
        the continuous loop calls this internally.

        Args:
            observation: The environmental observation for this tick.

        Returns:
            TickResult with prediction, error signal, model updates,
            timing breakdown, ★ and error trend metadata.
        """
        ...

    async def get_tick_stream(self) -> AsyncIterator[TickResult]:
        """Yields TickResults as they complete.

        Used by CDD-11 (Observability) to stream results to the dashboard.
        """
        ...


class Predictor(Protocol):
    """Generates predictions from World Model state."""

    def predict(
        self,
        context: PredictionContext,
    ) -> Prediction:
        """Given a causal subgraph and current state, predict next state.

        Uses causal graph traversal: follow CAUSES and
        PROBABILISTICALLY_CAUSES relationships from active nodes
        to compute expected outcomes.

        Must filter derived_from_chain=True relationships from
        independent evidence counting. (CDD-08 GPT-5 Cond 1)

        Must handle mixed-resolution subgraphs: if abstract and
        concrete children coexist, prefer concrete. (CDD-08 Gemini A)

        ★ Prediction confidence = harmonic_mean of node confidences
        AND relationship strengths as joint inputs. (Claude Obs 1,
        GPT-5 v2 refinement)
        """
        ...


class ErrorComputer(Protocol):
    """Computes error signal between prediction and observation."""

    def compute(
        self,
        prediction: Prediction,
        observation: Observation,
    ) -> ErrorSignal:
        """Quantified difference between predicted and observed state.

        Returns per-variable error magnitudes and an aggregate error score.
        """
        ...


class ModelUpdater(Protocol):
    """Updates the World Model based on error signals."""

    async def update(
        self,
        error_signal: ErrorSignal,
        prediction_context: PredictionContext,
        tick_id: int,
    ) -> UpdateResult:
        """Apply error signal to World Model.

        1. ★ Adjust confidence via weighted credit assignment
           (causal contribution + uncertainty absorption). (Gemini BLOCKING 1)
        2. ★ Explicitly trigger BSM reconciliation. (Grok Cond 1)
        3. Create new nodes for previously unmodeled observations.
        4. ★ Enqueue PredictionRecord asynchronously. (Gemini BLOCKING 2)
        5. If error exceeds threshold, invoke counterfactual generator.

        All writes go through GraphStore → Invariant Checker (CDD-03).
        """
        ...


class CounterfactualGenerator(Protocol):
    """Phase 1 simplified Dream Engine substitute.

    Generates candidate hypotheses when prediction error is high.
    Full Dream Engine replaces this in Phase 3 without changing
    the Predictive Loop's contract.
    """

    def generate(
        self,
        error_signal: ErrorSignal,
        prediction_context: PredictionContext,
    ) -> list[CounterfactualHypothesis]:
        """Generate ranked candidate explanations for the error.

        ★ Uses CounterfactualSimulator protocol for sandboxed
        evaluation. All simulation runs on deep-copied subgraphs.
        (Gemini BLOCKING 3)

        ★ Applies hypothesis deduplication via signature check
        to prevent near-duplicate hypothesis spam. (GPT-5 v2)

        Returns candidates ranked by explanatory power.
        Candidates are tagged SIMULATED for quarantine.
        """
        ...


class CounterfactualSimulator(Protocol):                               # ★ NEW (GPT-5 Q7 + Gemini BLOCKING 3)
    """Sandboxed simulation for evaluating hypotheses.

    Phase 1: single-step forward pass on deep-copied subgraph.
    Phase 3: replaced by Dream Engine with temporal replay.

    INVARIANT: This method NEVER reads from or writes to GraphStore.
    All operations occur on in-memory deep copies.
    """

    def simulate(
        self,
        hypothesis: CounterfactualHypothesis,
        original_context: PredictionContext,
        original_error: ErrorSignal,
    ) -> SimulationResult:
        """Run hypothesis in sandbox. Returns error reduction estimate."""
        ...


class EntityMatcher(Protocol):                                         # ★ NEW (GPT-5 Q5)
    """Matches observation entities to World Model nodes.

    Phase 1: exact entity_id string match (confidence=1.0).
    Phase 2+: fuzzy matching with confidence < 1.0 for
    multi-modal sensor fusion.
    """

    def match(
        self,
        observation_entity: EntityState,
        candidate_nodes: list[WorldNode],
    ) -> EntityMatch:
        """Find the best matching WorldNode for an observed entity."""
        ...


class LocalPredictiveCodingStub(Protocol):
    """Phase 2 local predictive coding interface.

    Raises NotImplementedError in Phase 1. Defines the contract
    for distributed cortical hierarchy integration.
    (Gemini addition, Impl Plan §6 S6)
    """

    def generate_local_prediction(
        self,
        region_id: str,
        context: CausalSubgraph,
    ) -> Prediction:
        """Phase 1: raises NotImplementedError."""
        ...

    def propagate_error(
        self,
        region_id: str,
        error: ErrorSignal,
    ) -> None:
        """Phase 1: raises NotImplementedError."""
        ...
```

### 3.2 Error Cases

| Error | Condition | Recovery |
|-------|-----------|----------|
| `ObservationTimeoutError` | No observation received within tick_interval + grace period | Skip tick, log gap, continue loop |
| `PredictionFailedError` | Subgraph extraction returned empty or timed out | Use last successful prediction context (degraded mode) |
| `GraphWriteError` | Neo4j write failed during update | Retry once, then log and continue (next tick will correct) |
| `InvariantViolationError` | Update would violate Tier 0 axiom | Reject update, log violation, skip the offending node update |
| `CalibrationCollapseError` | Brier > 0.30 (from CDD-06 §6.4) | Pause loop for affected domain, emit critical alert |
| `RedisConnectionError` | Observation channel unavailable | Retry with exponential backoff, pause after 5 failures |
| ★ `EpistemicThrashingWarning` | Persistent surprise without improvement (§H) | Raise surprise threshold, emit diagnostic, flag for human review |

---

## 4. Data Structures

### 4.1 Observation

```python
class Observation(BaseModel):
    """An environmental observation received from the Physics Simulator (CDD-09).

    Published to Redis channel `aria.observation.physics` as structured JSON.
    """
    model_config = ConfigDict(extra="forbid")

    observation_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    scenario_id: str = Field(
        ...,
        description="Which physics scenario generated this (e.g., 'free_fall_01').",
    )
    tick_id: int = Field(
        ..., ge=0,
        description="Monotonically increasing tick counter from simulator.",
    )
    timestamp: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    entity_states: dict[str, EntityState] = Field(
        ...,
        description=(
            "Current state of each observed entity. "
            "Key = entity_id, Value = state vector."
        ),
    )
    active_forces: list[str] = Field(
        default_factory=list,
        description="Forces currently active in the scenario (e.g., 'gravity', 'friction').",
    )
    confounder_active: bool = Field(
        default=False,
        description=(
            "Whether a hidden confounder is active this tick. "
            "The loop does NOT see this — it is metadata for test validation only."
        ),
    )
    metadata: dict[str, str] = Field(
        default_factory=dict,
        description="Scenario-specific metadata (e.g., 'phase': 'pre_confounder').",
    )


class EntityState(BaseModel):
    """State vector for a single observed entity."""
    model_config = ConfigDict(extra="forbid")

    entity_id: str
    position: tuple[float, float, float] = (0.0, 0.0, 0.0)
    velocity: tuple[float, float, float] = (0.0, 0.0, 0.0)
    acceleration: tuple[float, float, float] = (0.0, 0.0, 0.0)
    mass: float | None = None  # May be unknown/hidden
    custom_properties: dict[str, float] = Field(default_factory=dict)
```

### ★ 4.2 EntityMatch (NEW — GPT-5 Q5 + Claude Obs 2)

```python
class EntityMatch(BaseModel):
    """Result of matching an observation entity to a WorldNode."""
    model_config = ConfigDict(extra="forbid")

    node_id: str = Field(..., description="Matched WorldNode ID.")
    match_confidence: float = Field(                                    # ★ NEW
        ..., ge=0.0, le=1.0,
        description="Phase 1: always 1.0 (exact match). Phase 2+: < 1.0 for fuzzy.",
    )
    is_novel: bool = Field(                                             # ★ NEW
        default=False,
        description="True if no match found — entity is new to the World Model.",
    )
```

### 4.3 Prediction

```python
class Prediction(BaseModel):
    """The system's prediction of what should happen next.

    Generated by traversing the causal subgraph from the current state.
    """
    model_config = ConfigDict(extra="forbid")

    prediction_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    tick_id: int
    predicted_entity_states: dict[str, EntityState] = Field(
        ...,
        description="Predicted state for each entity at the next tick.",
    )
    confidence: float = Field(
        ..., ge=0.0, le=1.0,
        description=(
            "Aggregate confidence in this prediction. "
            "★ Harmonic mean of node confidences AND relationship strengths "
            "as joint inputs. (Claude Obs 1, GPT-5 v2 refinement)"
        ),
    )
    causal_chain_used: list[str] = Field(
        default_factory=list,
        description="Node IDs of the causal chain that generated this prediction.",
    )
    causal_relationships_used: list[str] = Field(                       # ★ NEW (for credit assignment)
        default_factory=list,
        description="Relationship IDs used in prediction (for blame attribution).",
    )
    subgraph_budget_used: BudgetConsumption | None = Field(
        default=None,
        description="How much of the CDD-08 extraction budget was consumed.",
    )
    abstraction_level: int = Field(
        default=0,
        description="Max causal_depth of nodes used in prediction. 0 = all concrete.",
    )
```

### 4.4 ErrorSignal

```python
class ErrorSignal(BaseModel):
    """Quantified difference between prediction and observation.

    This is ARIA's learning signal — the gap that drives model improvement.
    """
    model_config = ConfigDict(extra="forbid")

    tick_id: int
    aggregate_error: float = Field(
        ..., ge=0.0,
        description="Aggregate error magnitude (L2 norm across all entities, normalized).",
    )
    per_entity_errors: dict[str, EntityError] = Field(
        default_factory=dict,
        description="Error breakdown per entity.",
    )
    is_surprising: bool = Field(
        default=False,
        description=(
            "True if aggregate_error > surprise_threshold. "
            "Triggers counterfactual generation."
        ),
    )
    surprise_magnitude: float = Field(
        default=0.0, ge=0.0,
        description="How far above the surprise threshold (0.0 if not surprising).",
    )
    prediction_id: str = Field(...)
    observation_id: str = Field(...)


class EntityError(BaseModel):
    """Error for a single entity between predicted and observed state."""
    model_config = ConfigDict(extra="forbid")

    entity_id: str
    position_error: float = 0.0
    velocity_error: float = 0.0
    acceleration_error: float = 0.0
    dominant_error_axis: str = ""
```

### 4.5 PredictionContext

```python
class PredictionContext(BaseModel):
    """Everything the predictor needs to generate a prediction.

    Assembled at the start of each tick by extracting a bounded
    subgraph (CDD-08) and combining it with the current observation.

    ★ This object is IMMUTABLE once assembled. Counterfactual
    generation uses this pre-update snapshot. (Grok Cond 3 / §C)
    """
    model_config = ConfigDict(extra="forbid")

    tick_id: int
    observation: Observation
    subgraph: BudgetedSubgraph  # From CDD-08 extract_subgraph()
    previous_prediction: Prediction | None = None
    previous_error: ErrorSignal | None = None
    scenario_id: str
    domain_scope: str = "physics"
```

### 4.6 TickResult

★ Changes from v0.1: Added error trend and surprise state for observability (Claude Obs 3).

```python
class TickResult(BaseModel):
    """Complete result of a single predict-observe-error-update tick.

    Emitted for every tick. Consumed by CDD-11 (Observability).
    """
    model_config = ConfigDict(extra="forbid")

    tick_id: int
    timestamp: datetime
    observation: Observation
    prediction: Prediction
    error_signal: ErrorSignal
    update_result: UpdateResult
    timing: TickTiming
    counterfactual_count: int = 0

    # ★ Self-regulation metadata (Claude Obs 3 + §H)
    recent_error_trend: float = Field(                                  # ★ NEW
        default=0.0,
        description="Slope of aggregate_error over last N ticks. Negative = improving.",
    )
    consecutive_surprising_ticks: int = Field(                          # ★ NEW
        default=0,
        description="How many ticks in a row have been surprising.",
    )


class UpdateResult(BaseModel):
    """Outcome of applying error signal to World Model."""
    model_config = ConfigDict(extra="forbid")

    nodes_confidence_adjusted: int
    nodes_created: int
    bsm_transitions_triggered: int
    prediction_records_enqueued: int = 0                                # ★ CHANGED: "enqueued" not "stored"
    counterfactuals_generated: list[str] = Field(default_factory=list)
    novel_entities_created: int = 0                                     # ★ NEW
    novel_entities_deduplicated: int = 0                                # ★ NEW
```

### 4.7 TickTiming

```python
class TickTiming(BaseModel):
    """Per-tick timing breakdown.

    Logged for every tick. CDD-11 tracks p50/p95/p99.
    (Impl Plan §2.6, GPT-5 condition)
    """
    model_config = ConfigDict(extra="forbid")

    t_graph_read_ms: float = Field(
        ..., description="Time reading from Neo4j (subgraph extraction).",
    )
    t_prediction_ms: float = Field(
        ..., description="Time generating prediction from subgraph.",
    )
    t_error_compute_ms: float = Field(
        ..., description="Time computing error signal.",
    )
    t_graph_write_ms: float = Field(
        ..., description="Time writing updates to Neo4j (excludes async telemetry).",
    )
    t_total_ms: float = Field(
        ..., description="Total tick latency (sum of above + overhead).",
    )
    budget_starvation: bool = Field(
        default=False,
        description="True if CDD-08 entered starvation mode this tick.",
    )
```

### 4.8 CounterfactualHypothesis

```python
class CounterfactualHypothesis(BaseModel):
    """A candidate explanation for a surprising prediction error.

    Generated by SimplifiedCounterfactualGenerator.
    ★ Evaluated via CounterfactualSimulator on deep-copied subgraph.
    Tagged SIMULATED and sent to Quarantine (CDD-05).
    """
    model_config = ConfigDict(extra="forbid")

    hypothesis_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    source_tick_id: int
    hypothesis_type: str = Field(
        ...,
        pattern=r"^(confounder|unobserved_variable|violated_invariant|"
                r"missing_relationship|strength_miscalibration)$",
        description="Category of explanatory hypothesis.",
    )
    explanation: str = Field(
        ..., max_length=500,
        description="Human-readable description of the hypothesis.",
    )
    candidate_node: WorldNode | None = Field(
        default=None,
        description="The proposed new node (if hypothesis suggests unmodeled entity).",
    )
    candidate_relationship: CausalRelationship | None = Field(
        default=None,
        description="The proposed new/modified relationship.",
    )
    error_reduction_estimate: float = Field(
        ..., ge=0.0, le=1.0,
        description=(
            "Estimated fraction of prediction error this hypothesis explains. "
            "Must be > 0 to survive the error reduction rule. (GPT-5 condition)"
        ),
    )
    confidence: float = Field(
        default=0.3,
        ge=0.0, le=1.0,
        description="Initial confidence in this hypothesis.",
    )
    signature: tuple[str, str, str] | None = Field(                     # ★ NEW (GPT-5 v2)
        default=None,
        description=(
            "(type, source_node_id, target_node_id) for deduplication. "
            "Hypotheses with identical signatures within a rolling window "
            "are discarded to prevent hypothesis spam."
        ),
    )
```

### ★ 4.9 SimulationResult (NEW — Gemini BLOCKING 3)

```python
class SimulationResult(BaseModel):
    """Outcome of a sandboxed counterfactual simulation."""
    model_config = ConfigDict(extra="forbid")

    hypothesis_id: str
    original_error: float
    simulated_error: float
    error_reduction: float  # (original - simulated) / original
    contradicts_confirmed: bool = False  # ★ Consistency check (GPT-5 Cond 3)
```

### ★ 4.10 PredictionRecordBuffer (NEW — Gemini BLOCKING 2)

```python
class PredictionRecordBuffer:
    """Asynchronous buffer for PredictionRecord writes.

    Records are enqueued synchronously during tick execution (O(1)).
    A background asyncio task flushes to Neo4j in batches.
    Decouples telemetry from the synchronous cognitive tick.
    """

    def __init__(
        self,
        flush_interval_seconds: float = 5.0,
        max_buffer_size: int = 500,
        hard_buffer_cap: int = 2000,                                    # ★ Grok v2 safety
    ):
        self._buffer: list[PredictionRecord] = []
        self._flush_interval = flush_interval_seconds
        self._max_buffer = max_buffer_size
        self._hard_cap = hard_buffer_cap

    def enqueue(self, record: PredictionRecord) -> None:
        """Called synchronously during tick. O(1)."""
        if len(self._buffer) >= self._hard_cap:
            # Drop oldest to prevent unbounded memory growth
            dropped = self._buffer[:100]
            self._buffer = self._buffer[100:]
            structlog.get_logger().warning(
                "loop.prediction_records_dropped",
                count=len(dropped),
            )
        self._buffer.append(record)
        if len(self._buffer) >= self._max_buffer:
            asyncio.create_task(self._flush())

    async def _flush(self) -> None:
        """Batch write to Neo4j. Runs in background."""
        if not self._buffer:
            return
        batch = self._buffer.copy()
        self._buffer.clear()
        try:
            await graphstore.batch_create_prediction_records(batch)
        except Exception:
            structlog.get_logger().error(
                "loop.prediction_record_flush_failed",
                batch_size=len(batch),
            )
            # Re-enqueue failed batch (up to hard cap)
            self._buffer = batch[:self._hard_cap - len(self._buffer)] + self._buffer

    async def flush_and_shutdown(self) -> None:
        """Final flush on loop shutdown."""
        await self._flush()
```

---

## 5. Algorithm / Logic Flow

### 5.1 Continuous Loop Engine

★ Changes from v0.1: Added persistent surprise escalation (§H), rolling error window, escalation reset.

```
start() called
    │
    ├── Subscribe to Redis channel: aria.observation.{domain_scope}
    ├── Initialize: tick_counter = 0, last_prediction = None, last_error = None
    ├── ★ Initialize: previous_budget_nodes = config.default_budget_nodes
    ├── ★ Initialize: recent_errors = RollingWindow(size=surprise_escalation_window)
    ├── ★ Initialize: consecutive_surprising_ticks = 0
    ├── ★ Initialize: current_surprise_threshold = config.surprise_threshold
    ├── ★ Initialize: recent_hypothesis_signatures = RollingWindow(size=100)
    ├── Initialize: consecutive_gap_count = 0, consecutive_error_count = 0
    ├── ★ Initialize: prediction_record_buffer = PredictionRecordBuffer(...)
    ├── Log: structlog "loop.started" with config summary
    │
    ▼
LOOP (runs until stop() called):
    │
    ├── Wait for next observation from Redis
    │     Timeout: tick_interval_ms + observation_grace_ms
    │     If timeout:
    │       Log "loop.observation_gap"
    │       consecutive_gap_count += 1
    │       If consecutive_gap_count > max_observation_gaps: emit warning
    │       Continue to next iteration
    │
    ├── consecutive_gap_count = 0
    ├── Deserialize Observation from JSON
    │     If deserialization fails: log error, continue
    ├── tick_counter += 1
    │
    ├── TRY:
    │     result = await run_single_tick(observation)
    │     consecutive_error_count = 0
    │
    │     ★ Update rolling error window:
    │       recent_errors.append(result.error_signal.aggregate_error)
    │
    │     ★ Update surprise tracking:
    │       If result.error_signal.is_surprising:
    │         consecutive_surprising_ticks += 1
    │       Else:
    │         consecutive_surprising_ticks = 0
    │         ★ Reset escalated threshold after 5 non-surprising ticks
    │         current_surprise_threshold = config.surprise_threshold (Grok v2)
    │
    │     ★ Persistent surprise escalation check (GPT-5 missing mode):
    │       If consecutive_surprising_ticks > config.surprise_escalation_window:
    │         trend = compute_error_trend(recent_errors)
    │         If trend.improvement < config.surprise_escalation_improvement_threshold:
    │           emit "loop.epistemic_thrashing" alert
    │           current_surprise_threshold *= 1.5 (raise bar)
    │           Log full causal context for human diagnosis
    │
    │     ★ Periodic recalibration check (Grok Cond 2):
    │       If tick_counter % config.recalibration_check_interval_ticks == 0:
    │         brier = CDD06.compute_brier_score(domain_scope, window=N)
    │         If brier > recalibration_warning_threshold (0.20):
    │           emit "loop.calibration_drift_detected"
    │           CDD06.trigger_recalibration(domain_scope)
    │         If brier > 0.30:
    │           Pause loop (calibration collapse)
    │
    │     Emit result to tick stream (for CDD-11)
    │     Log: structlog "loop.tick_complete" with tick_id, timing, aggregate_error
    │
    ├── EXCEPT:
    │     Log error with full context
    │     consecutive_error_count += 1
    │     If consecutive_error_count > max_consecutive_errors:
    │       Pause loop, emit "loop.error_cascade"
    │     Continue
```

### 5.2 Single Tick — The Core Cognitive Cycle

This is the atomic unit of cognition.

★ Major changes from v0.1: Weighted credit assignment (§A), explicit BSM reconciliation (§I), async PredictionRecord (§B), novel entity hygiene (§L), pre-update context for counterfactuals (§C/§K).

```
run_single_tick(observation) called
    │
    ├── start_timer(total)
    │
    ▼
PHASE 1: PREDICT (Graph Read + Prediction Generation)
    │
    ├── start_timer(graph_read)
    ├── ★ Match observation entities to WorldNodes via EntityMatcher:
    │     For each entity in observation.entity_states:
    │       match = EntityMatcher.match(entity, candidate_nodes)
    │       If match.is_novel: mark for creation in Phase 4
    │       Else: record (entity_id → match.node_id, match.match_confidence)
    │     Select primary entity (highest confidence or first observed)
    │
    ├── ★ budget = allocate_budget(tick_id, previous_timing, previous_budget_nodes)
    │     Adapts from PREVIOUS budget, not default (GPT-5 Cond 5)
    │     Safety floor on high total latency (Grok Cond 4)
    │
    ├── subgraph = CausalComplexityManager.extract_subgraph(
    │     query_node_id=primary_entity_node,
    │     budget=budget,
    │     direction=BOTH,  # Extract both causes and effects for context
    │     min_confidence=min_prediction_confidence,
    │   )
    ├── stop_timer(graph_read) → t_graph_read_ms
    │
    ├── start_timer(prediction)
    ├── context = PredictionContext(
    │     tick_id, observation, subgraph,
    │     previous_prediction=last_prediction,
    │     previous_error=last_error,
    │     scenario_id=observation.scenario_id,
    │   )
    │   ★ Context is IMMUTABLE from this point forward (Grok Cond 3)
    │
    ├── Resolve mixed-resolution subgraph (CDD-08 §6 Gemini A):
    │     If abstract node AND its concrete children both present:
    │       Remove abstract, keep concrete (higher resolution)
    │
    ├── Filter derived relationships:
    │     Exclude relationships where derived_from_chain=True
    │     from causal traversal (CDD-08 §4.7 GPT-5 Cond 1)
    │
    ├── prediction = Predictor.predict(context)
    │     ★ Prediction follows OUTGOING relationships only (Q8 consensus)
    │     ★ Confidence = harmonic_mean(node_confidences + rel_strengths) (Claude Obs 1)
    ├── stop_timer(prediction) → t_prediction_ms
    │
    ▼
PHASE 2: OBSERVE (already received — passed as argument)
    │
    ▼
PHASE 3: ERROR SIGNAL
    │
    ├── start_timer(error_compute)
    ├── error_signal = ErrorComputer.compute(prediction, observation)
    │
    ├── Classify surprise:
    │     ★ if error_signal.aggregate_error > current_surprise_threshold:
    │       (uses escalated threshold if in thrashing state)
    │       error_signal.is_surprising = True
    │       error_signal.surprise_magnitude = aggregate_error - threshold
    │
    ├── stop_timer(error_compute) → t_error_compute_ms
    │
    ▼
PHASE 4: UPDATE MODEL
    │
    ├── start_timer(graph_write)
    │
    ├── ★ 4a. Weighted credit assignment (Gemini BLOCKING 1 + GPT-5 Cond 1+2):
    │     For each node_id in prediction.causal_chain_used:
    │       node = GraphStore.get_node(node_id)
    │       If node is Tier 0/1: skip (immutable confidence)
    │       
    │       ★ Compute blame_weight:
    │         causal_weight = relationship_strength / total_chain_strength
    │         uncertainty_weight = node.epistemic_uncertainty / total_chain_uncertainty
    │         blame_weight = 0.5 * causal_weight + 0.5 * uncertainty_weight
    │       
    │       Determine if prediction was locally correct for this node:
    │         Compare predicted vs observed values for this node's effects
    │         prediction_correct = (local_error < correct_threshold)
    │       
    │       delta = compute_confidence_delta(
    │         node, prediction_correct, local_error,
    │         config.learning_rate, config.confidence_asymmetry_factor,
    │         blame_weight
    │       )
    │       ★ delta *= prediction.confidence  (GPT-5 Cond 1: scale by prediction certainty)
    │       
    │       new_confidence = clamp(node.confidence + delta, 0.0, 1.0)
    │       GraphStore.update_node(node_id, {confidence: new_confidence})
    │       → Triggers Invariant Checker (CDD-03 §5)
    │       
    │       ★ BSM.evaluate_transitions(node_id, new_confidence, trigger)
    │         → Explicit reconciliation call (Grok Cond 1)
    │         → If transition fires: increment bsm_transitions_triggered
    │         → If transition triggers CDD-08 foundation check: synchronous
    │
    ├── 4b. Context diversity update:
    │     For nodes involved in prediction:
    │       Compute context_signature_hash (CDD-06 §4.4):
    │         Axes: scenario_id, time_bucket(hour), active_forces
    │       If new context (hash not seen before):
    │         Increment node.context_diversity
    │
    ├── ★ 4c. Enqueue PredictionRecords asynchronously (Gemini BLOCKING 2):
    │     For each node that contributed to the prediction:
    │       prediction_record_buffer.enqueue(PredictionRecord(...))
    │     → Buffer flushes in background. O(1) per enqueue.
    │     → Does NOT block t_graph_write_ms.
    │
    ├── ★ 4d. Create nodes for unmodeled entities (with hygiene — Grok Cond 5):
    │     For each entity marked as novel in Phase 1:
    │       ★ Deduplication: check if HYPOTHESIS node with matching label
    │         + domain_scope already exists → if so, increment observation_count
    │       ★ Cap: if novel_entities_this_tick >= max_novel_entities_per_tick (3):
    │         Log excess, skip creation
    │       ★ Tier 0/1 protection: provisional CORRELATES_WITH rels
    │         CANNOT target Tier 0 or Tier 1 nodes
    │       Create new WorldNode(
    │         label=entity_id, belief_state=HYPOTHESIS,
    │         source="learned",
    │         ★ confidence=config.novel_entity_initial_confidence (0.4),
    │         causal_depth=0,
    │         ★ decay_rate_multiplier=2.0 (elevated until reinforced)
    │       )
    │
    ├── ★ 4e. Counterfactual generation (uses PRE-UPDATE context — §C/§K):
    │     If error_signal.is_surprising:
    │       ★ hypotheses = CounterfactualGenerator.generate(
    │           error_signal, context  ← IMMUTABLE pre-update snapshot
    │         )
    │       For each hypothesis:
    │         Send to Quarantine (CDD-05) via QuarantineGraphStore
    │         Record hypothesis_id in update_result
    │
    ├── stop_timer(graph_write) → t_graph_write_ms
    │
    ▼
Assemble TickResult:
    timing = TickTiming(t_graph_read_ms, t_prediction_ms,
                        t_error_compute_ms, t_graph_write_ms,
                        t_total_ms=stop_timer(total))
    result = TickResult(
        tick_id, observation, prediction, error_signal,
        update_result, timing,
        counterfactual_count=len(hypotheses),
        ★ recent_error_trend=compute_error_trend(recent_errors),
        ★ consecutive_surprising_ticks=consecutive_surprising_ticks,
    )

Update loop state:
    last_prediction = prediction
    last_error = error_signal
    ★ previous_budget_nodes = budget.max_nodes

Return result
```

### 5.3 Prediction Algorithm — Causal Graph Traversal

★ Changes from v0.1: Relationship strengths included in confidence (Claude Obs 1, GPT-5 v2). OUTGOING-only traversal confirmed (Q8).

```
Predictor.predict(context) called
    │
    ├── Extract active entity nodes from subgraph:
    │     Match observation.entity_states keys → WorldNode labels
    │     These are the "source" nodes for forward prediction
    │
    ├── For each source entity node:
    │     ├── Follow OUTGOING causal relationships in subgraph (Q8):
    │     │     Include: CAUSES, PROBABILISTICALLY_CAUSES, ENABLES
    │     │     Exclude: is_structural=True, derived_from_chain=True
    │     │
    │     ├── For each reachable effect node:
    │     │     Compute predicted_value based on:
    │     │       - Source entity's current state (from observation)
    │     │       - Relationship strength and type
    │     │       - Effect node's domain_scope properties
    │     │
    │     │     Relationship-type-specific rules:
    │     │       CAUSES: predicted = source_state * relationship.strength
    │     │       PROBABILISTICALLY_CAUSES: predicted *= probability
    │     │       ENABLES: if enabler present → allow effect, else → 0
    │     │       PREVENTS: if preventer active → reduce by strength
    │     │       STOCHASTIC_THRESHOLD: if source < threshold → 0, else → effect
    │     │
    │     └── Aggregate multiple effects on same entity:
    │           predicted_entity_state = weighted sum of all effects
    │           weights = relationship.strength * source_node.confidence
    │
    ├── ★ Compute aggregate prediction confidence (Claude Obs 1, GPT-5 v2):
    │     Combine node confidences AND relationship strengths as
    │     joint inputs to harmonic mean:
    │     
    │     inputs = [node.confidence for node in causal_chain_used]
    │            + [rel.strength for rel in relationships_used]
    │     prediction.confidence = harmonic_mean(inputs)
    │     
    │     This ensures a chain of high-confidence nodes connected by
    │     weak relationships produces LOW prediction confidence.
    │     A single weak link (node OR relationship) penalizes the whole chain.
    │
    ├── Record causal_chain_used, causal_relationships_used
    ├── Record abstraction_level = max(causal_depth) of nodes used
    │
    ▼
Return Prediction
```

### 5.4 Error Signal Computation

(Unchanged from v0.1 — no review feedback required changes.)

```
ErrorComputer.compute(prediction, observation) called
    │
    ├── For each entity in observation.entity_states:
    │     ├── If entity exists in prediction.predicted_entity_states:
    │     │     position_error = euclidean_distance(pred.position, obs.position)
    │     │     velocity_error = euclidean_distance(pred.velocity, obs.velocity)
    │     │     acceleration_error = euclidean_distance(pred.accel, obs.accel)
    │     │     dominant_error_axis = max(position, velocity, acceleration)
    │     │
    │     └── If entity NOT in prediction (novel/unmodeled):
    │           entity_error = EntityError(entity_id, novelty_penalty, ...)
    │
    ├── aggregate_error = sqrt(sum(e² for all entity errors)) / num_entities
    ├── is_surprising = (aggregate_error > current_surprise_threshold)
    ├── surprise_magnitude = max(0, aggregate_error - current_surprise_threshold)
    │
    ▼
Return ErrorSignal
```

### 5.5 Confidence Update Rules — Weighted Credit Assignment

★ Complete rewrite from v0.1. Addresses GPT-5 Cond 1+2 + Gemini BLOCKING 1.

```python
def compute_confidence_delta(
    node: WorldNode,
    prediction_correct: bool,
    error_magnitude: float,
    learning_rate: float,
    asymmetry_factor: float,
    blame_weight: float,
) -> float:
    """Weighted credit assignment with uncertainty absorption.

    ★ Three multipliers shape the final delta:
    1. Base delta (from correctness and error magnitude)
    2. Blame weight (causal contribution × uncertainty absorption)
    3. Prediction confidence (applied by caller: delta *= prediction.confidence)

    Final equation: delta = base_delta * blame_weight * prediction.confidence

    This ensures:
    - Stable nodes (low uncertainty) absorb less blame (Gemini BLOCKING 1)
    - Weak causal links absorb more blame (GPT-5 Cond 2)
    - Weak predictions produce weak updates (GPT-5 Cond 1)
    """
    if prediction_correct:
        # Correct: small increase, diminishing as confidence grows
        base_delta = learning_rate * (1.0 - node.confidence) * (1.0 - error_magnitude)
    else:
        # Incorrect: larger decrease, proportional to error
        base_delta = -learning_rate * node.confidence * error_magnitude * asymmetry_factor

    return base_delta * blame_weight


def compute_blame_weight(
    node: WorldNode,
    chain_nodes: list[WorldNode],
    chain_relationships: list[CausalRelationship],
) -> float:
    """Distribute blame proportionally across the causal chain.

    Two factors, equally blended:
    1. Causal contribution: node's relationship strength / total strength
    2. Uncertainty absorption: node's epistemic_uncertainty / total uncertainty

    50/50 blend ensures neither structure nor epistemic state dominates.
    """
    # Causal contribution
    node_strength = _get_relationship_strength_for_node(node, chain_relationships)
    total_strength = sum(r.strength for r in chain_relationships) or 1.0
    causal_weight = node_strength / total_strength

    # Uncertainty absorption
    total_uncertainty = sum(n.epistemic_uncertainty for n in chain_nodes) or 1.0
    uncertainty_weight = node.epistemic_uncertainty / total_uncertainty

    return 0.5 * causal_weight + 0.5 * uncertainty_weight
```

**Asymmetry rationale (unchanged):** Disconfirming evidence is more informative. The asymmetry_factor (default 1.5) means wrong predictions hurt more than correct predictions help.

**Blame weight rationale:** In predictive coding, prediction errors are precision-weighted — high-precision (low-uncertainty) signals dominate learning while noisy signals are downweighted. The 50/50 blend of causal and uncertainty weighting is a Phase 1 simplification; Phase 2+ could tune the blend dynamically.

### 5.6 Simplified Counterfactual Generator

★ Changes from v0.1: In-memory sandbox (Gemini BLOCKING 3), consistency check (GPT-5 Cond 3), hypothesis deduplication (GPT-5 v2), uses pre-update context (Grok Cond 3).

```
CounterfactualGenerator.generate(error_signal, context) called
    │
    ├── If NOT error_signal.is_surprising: return []
    │
    ├── ★ Context is the IMMUTABLE pre-update PredictionContext (Grok Cond 3)
    │     Counterfactuals explain why the prediction was wrong given
    │     what we knew at prediction time, not after update.
    │
    ├── Strategy 1: CONFOUNDER search
    │     Find CONFOUNDER relationships in subgraph
    │     For each unobserved confounder_node_id:
    │       Generate hypothesis: "Unobserved confounder {label} may be active"
    │
    ├── Strategy 2: Unobserved variable enumeration
    │     Find ENABLES/REQUIRES targets not in observation
    │     Generate hypothesis: "Missing enabler/requirement {label}"
    │
    ├── Strategy 3: Invariant violation check
    │     Check error pattern against physical law violations
    │     Generate hypothesis: "Violated assumption: {invariant}"
    │
    ├── Strategy 4: Relationship strength miscalibration
    │     If error concentrated on one axis:
    │       Generate hypothesis: "Relationship {A→B} strength is wrong"
    │
    ├── Strategy 5: Missing relationship
    │     If entities correlated in observation but no graph relationship:
    │       Generate hypothesis: "Missing causal link between {A} and {B}"
    │
    ├── ★ Hypothesis deduplication (GPT-5 v2):
    │     For each hypothesis, compute signature = (type, source_node, target_node)
    │     If signature in recent_hypothesis_signatures: discard
    │     Else: add signature to rolling window
    │
    ├── ★ Sandboxed error reduction check (Gemini BLOCKING 3):
    │     For each hypothesis:
    │       result = CounterfactualSimulator.simulate(
    │         hypothesis, context, error_signal
    │       )
    │       ★ INVARIANT: simulate() operates ONLY on deep-copied subgraph.
    │         Zero interaction with GraphStore.
    │       hypothesis.error_reduction_estimate = result.error_reduction
    │       If error_reduction ≤ min_error_reduction_for_hypothesis: discard
    │
    │     ★ Consistency check (GPT-5 Cond 3):
    │       If result.contradicts_confirmed:
    │         hypothesis.error_reduction_estimate *= 0.5 (penalty)
    │       (Don't discard — contradicting confirmed beliefs is sometimes
    │        correct, but should be penalized)
    │
    ├── Rank by error_reduction_estimate (descending)
    ├── Cap at max_counterfactuals_per_tick (default: 5)
    │
    ├── For each surviving hypothesis:
    │     Assign belief_state = HYPOTHESIS, source = "counterfactual"
    │     Assign confidence = counterfactual_initial_confidence (0.3)
    │     → Caller sends to QuarantineGraphStore (CDD-05)
    │
    ├── Log: structlog "loop.counterfactuals_generated" with
    │     tick_id, count, types, top_error_reduction
    │
    ▼
Return list[CounterfactualHypothesis]
```

### 5.7 Budget Allocation Strategy

★ Changes from v0.1: Adapts from previous budget (GPT-5 Cond 5), safety floor on high latency (Grok Cond 4).

```python
def allocate_budget(
    tick_id: int,
    previous_timing: TickTiming | None,
    previous_budget_nodes: int | None,
    config: PredictiveLoopConfig,
) -> SubgraphBudget:
    """Adaptive budget with memory and safety floor.

    ★ Three stabilizers:
    1. Hysteresis: asymmetric 0.8× down / 1.1× up (prevents oscillation)
    2. Memory: adapts from previous actual budget, not default
    3. Safety floor: forces minimum budget during high-load periods
    """
    budget_nodes = previous_budget_nodes or config.default_budget_nodes

    if previous_timing:
        if previous_timing.t_graph_read_ms > config.graph_read_target_ms:
            budget_nodes = int(budget_nodes * 0.8)
        elif previous_timing.t_graph_read_ms < config.graph_read_target_ms * 0.5:
            budget_nodes = int(budget_nodes * 1.1)

        # ★ Safety floor: if total tick was too slow, force minimum (Grok Cond 4)
        if previous_timing.t_total_ms > config.max_tick_latency_ms:
            budget_nodes = config.min_budget_nodes

    budget_nodes = max(config.min_budget_nodes, min(config.max_budget_nodes, budget_nodes))

    return SubgraphBudget(
        max_nodes=budget_nodes,
        max_relationships=budget_nodes * 5,
        max_depth=config.default_budget_depth,
        time_limit_ms=config.graph_read_target_ms,
        allow_abstraction_fallback=True,
    )
```

---

## 6. Integration Points

★ Changes from v0.1: Added explicit BSM reconciliation, active recalibration, EntityMatcher, CounterfactualSimulator, passive decay ownership note.

| Component | What It Uses / Provides | How |
|-----------|------------------------|-----|
| **CDD-01: GraphStore** | Read: `query_causal_neighborhood`. Write: `update_node`, `create_node`, `create_relationship`. ★ Batch PredictionRecord writes via buffer. | All graph I/O through Protocol. Writes trigger Invariant Checker. |
| **CDD-01: Decay** | ★ CDD-01 §5.4 owns passive confidence decay for ALL nodes (including unused ones). CDD-04 does NOT apply additional decay. | Decay applies independently of prediction activity. Prevents epistemic fossilization. (GPT-5 Cond 4 → deferred to CDD-01) |
| **CDD-03: BSM** | ★ `evaluate_transitions()` called EXPLICITLY after each confidence update. `reconcile_after_decay` may cascade to CDD-08 foundation checks. | Synchronous during Phase 4a. Rejected transitions logged, not fatal. (Grok Cond 1) |
| **CDD-03: Invariant Checker** | Automatically invoked on every GraphStore write. | Rejects invalid mutations before they reach Neo4j. |
| **CDD-06: Formal Semantics** | `PredictionRecord` ★ enqueued asynchronously per tick. `context_signature_hash` computed per prediction. ★ Periodic Brier check triggers recalibration. | CDD-06 defines metrics; CDD-04 produces data and ★ actively checks calibration every N ticks. (Grok Cond 2) |
| **CDD-08: Complexity Manager** | `extract_subgraph()` called per tick with ★ adaptive budget (memory + safety floor). Filters `derived_from_chain=True`. Handles mixed-resolution. | CDD-08 provides tool, CDD-04 provides policy. |
| **CDD-09: Physics Simulator** | Publishes `Observation` to Redis `aria.observation.physics`. | CDD-04 subscribes and deserializes via ★ EntityMatcher. |
| **CDD-05: Quarantine** | Counterfactual hypotheses sent to QuarantineGraphStore. | ★ Generated from pre-update context. Sandboxed simulation. |
| **CDD-07: Adversary Simulator** | Uses PredictionRecord data for adversarial variants. | Reads prediction history from Neo4j. |
| **CDD-10: DriftMonitor** | Consumes aggregate_error trends from TickResult stream. | CDD-10 does comprehensive trend detection; CDD-04 does ★ periodic spot checks. |
| **CDD-11: Observability** | Consumes TickResult + TickTiming for real-time dashboard. ★ Receives error_trend and consecutive_surprising_ticks. | get_tick_stream() yields real-time data. |

---

## 7. Configuration Parameters

★ Changes from v0.1: Added surprise escalation, recalibration interval, novel entity hygiene, blame weight, buffer settings, Phase 2 persistence note.

```python
class PredictiveLoopConfig(BaseSettings):
    """Configuration for the Predictive Loop Engine."""

    # ── Tick Timing ────────────────────────────────────────────────
    tick_interval_ms: int = 100
    observation_grace_ms: int = 50
    max_tick_latency_ms: int = 200

    # ── Budget Allocation ──────────────────────────────────────────
    default_budget_nodes: int = 100
    min_budget_nodes: int = 10
    max_budget_nodes: int = 500
    default_budget_depth: int = 5
    graph_read_target_ms: int = 50

    # ── Learning / Credit Assignment ───────────────────────────────
    learning_rate: float = 0.1
    confidence_asymmetry_factor: float = 1.5                            # ★ Exposed (GPT-5 Q2)
    blame_weight_causal_ratio: float = 0.5                              # ★ NEW: causal vs uncertainty blend
    correct_threshold: float = 0.1
    min_prediction_confidence: float = 0.1
    surprise_threshold: float = 0.3  # ★ Should align with CDD-06 accuracy_drop_trigger (Grok Cond 7)
    novelty_penalty: float = 1.0

    # ── Novel Entity Hygiene ───────────────────────────────────────
    novel_entity_initial_confidence: float = 0.4                        # ★ CHANGED from 0.3 (Q4 compromise)
    max_novel_entities_per_tick: int = 3                                 # ★ NEW (Grok Cond 5)
    novel_entity_decay_rate_multiplier: float = 2.0                     # ★ NEW (Grok Cond 5)

    # ── Counterfactual Generator ───────────────────────────────────
    max_counterfactuals_per_tick: int = 5
    min_error_reduction_for_hypothesis: float = 0.05
    counterfactual_initial_confidence: float = 0.3
    confirmed_contradiction_penalty: float = 0.5                        # ★ NEW (GPT-5 Cond 3)
    hypothesis_dedup_window_size: int = 100                             # ★ NEW (GPT-5 v2)

    # ── Self-Regulation ────────────────────────────────────────────
    surprise_escalation_window: int = 20                                # ★ NEW (GPT-5 missing mode)
    surprise_escalation_improvement_threshold: float = 0.1              # ★ NEW
    surprise_escalation_multiplier: float = 1.5                         # ★ NEW
    surprise_escalation_reset_ticks: int = 5                            # ★ NEW (Grok v2)
    recalibration_check_interval_ticks: int = 100                       # ★ NEW (Grok Cond 2)
    recalibration_warning_threshold: float = 0.20                       # ★ NEW

    # ── Prediction ─────────────────────────────────────────────────
    prediction_confidence_method: str = "harmonic_mean"

    # ── Telemetry Buffer ───────────────────────────────────────────
    prediction_record_flush_interval_seconds: float = 5.0               # ★ NEW (Gemini BLOCKING 2)
    prediction_record_buffer_size: int = 500                            # ★ NEW
    prediction_record_hard_cap: int = 2000                              # ★ NEW (Grok v2)

    # ── Resilience ─────────────────────────────────────────────────
    max_observation_gaps: int = 10
    redis_retry_backoff_base_ms: int = 100
    redis_max_retries: int = 5
    max_consecutive_errors: int = 20

    # ── Observation Channel ────────────────────────────────────────
    redis_observation_channel: str = "aria.observation.physics"
    observation_json_schema_version: str = "1.0"

    # ── Phase 2 Requirements (documented, not active) ──────────────
    # ★ Loop state persistence: persist last_prediction, last_error,
    #   previous_budget_nodes, consecutive_surprising_ticks to
    #   lightweight store on shutdown. Reload on start().
    #   Required for v2.1 §1.2 "genuine experience across sessions".
    #   (Grok Cond 6 → deferred to Phase 2)

    model_config = SettingsConfigDict(env_prefix="ARIA_LOOP_")
```

---

## 8. Error Handling & Edge Cases

★ Changes from v0.1: Updated Tier 0 handling, added thrashing, novel entity hygiene cases, async buffer cases.

| Edge Case | Behavior | Rationale |
|-----------|----------|-----------|
| Observation timeout | Skip tick, increment gap counter, continue | Simulator may be slow; don't crash the loop |
| Observation contains unknown entity | ★ Create HYPOTHESIS (max 3/tick) with dedup, confidence 0.4, elevated decay. No Tier 0/1 links. (Grok Cond 5) | Controlled growth prevents graph noise |
| Prediction has zero causal chain | Prediction with confidence=0.0, all-zero state | Large error triggers counterfactuals |
| Error signal exactly 0.0 | Log, no confidence adjustment, no counterfactuals | Perfect prediction needs no changes |
| Brier > 0.30 (calibration collapse) | Pause loop, critical alert, human acknowledge (CDD-06 §6.4) | Worse than random — stop predicting |
| ★ Brier > 0.20 (calibration drift) | Trigger recalibration, warning alert (Grok Cond 2) | Catch drift before collapse |
| BSM transition rejected | Log rejection, continue to next node | Not fatal — one rejection doesn't block tick |
| GraphStore write fails | Retry once, then skip node | Neo4j transient failures recoverable |
| Subgraph extraction empty | Use query node only, low-confidence prediction | Degenerate but valid |
| Redis connection lost | Exponential backoff, pause after max_retries | Don't burn CPU on unavailable service |
| Counterfactual generator 0 hypotheses | Normal — logged as "no_hypotheses" | Not all surprises have obvious explanations |
| Tick latency > max_tick_latency_ms | Log warning, ★ budget safety floor next tick (Grok Cond 4) | Auto-adapts under load |
| 20+ consecutive errors | Pause loop, critical alert | Fundamentally broken — investigate |
| ★ Persistent surprise without improvement | Raise surprise threshold (1.5×), emit diagnostic (GPT-5 §H) | Prevents epistemic thrashing |
| ★ Surprise threshold escalated for too long | Reset after 5 non-surprising ticks (Grok v2) | Prevents permanent desensitization |
| CDD-08 starvation mode | Reduced subgraph, tick continues | CDD-08 handles; CDD-04 adapts |
| Mixed-resolution subgraph | Prefer concrete over abstract (CDD-08 §6) | Higher resolution preferred |
| Tier 0/1 node in causal chain | Skip confidence update (immutable) | Axioms don't change from predictions |
| ★ Near-duplicate counterfactual hypothesis | Discarded via signature dedup (GPT-5 v2) | Prevents hypothesis spam |
| ★ PredictionRecord buffer overflow | Drop oldest records, log warning (Grok v2) | Telemetry loss < loop crash |
| ★ PredictionRecord flush fails | Re-enqueue up to hard cap (2000) | Graceful degradation |
| ★ High-uncertainty node in chain | Absorbs more blame via uncertainty_weight (Gemini BLOCKING 1) | Prevents catastrophic forgetting |

---

## 9. Module Structure

★ Changes from v0.1: Added counterfactual simulator, entity matcher, telemetry buffer.

```
src/aria/predictive_loop/
├── __init__.py
├── protocols.py                    # PredictiveLoopEngine, Predictor, ErrorComputer,
│                                   # ModelUpdater, CounterfactualGenerator,
│                                   # ★ CounterfactualSimulator, ★ EntityMatcher,
│                                   # LocalPredictiveCodingStub
├── engine.py                       # AsyncIO main loop: start(), stop(), run_single_tick()
│                                   # ★ Surprise escalation, recalibration checks
├── predictor.py                    # Causal graph traversal prediction (§5.3)
├── error_signal.py                 # Error computation: predicted vs observed (§5.4)
├── updater.py                      # ★ Weighted credit assignment (§5.5)
├── counterfactual.py               # SimplifiedCounterfactualGenerator (§5.6)
├── simulator.py                    # ★ CounterfactualSimulator (in-memory sandbox)
├── entity_matcher.py               # ★ EntityMatcher (exact Phase 1, fuzzy Phase 2+)
├── timing.py                       # TickTiming instrumentation
├── budget.py                       # ★ Adaptive budget allocation (§5.7)
├── observation.py                  # Observation deserialization from Redis
├── telemetry.py                    # ★ PredictionRecordBuffer (async flush)
├── config.py                       # PredictiveLoopConfig
└── local_predictive_coding.py      # Stub interface (NotImplementedError in Phase 1)
```

---

## 10. Performance Targets

| Metric | Phase 1 Target | Measurement |
|--------|---------------|-------------|
| Total tick latency (p50) | < 100ms | TickTiming.t_total_ms |
| Total tick latency (p95) | < 200ms | TickTiming.t_total_ms |
| Graph read (subgraph extraction) | < 50ms | TickTiming.t_graph_read_ms |
| Prediction generation | < 20ms | TickTiming.t_prediction_ms |
| Error computation | < 5ms | TickTiming.t_error_compute_ms |
| Graph write (confidence + BSM) | < 50ms | ★ Excludes async PredictionRecord writes |
| 100-tick accuracy improvement | Aggregate error trend negative after tick 20 | Integration test |
| Counterfactual generation | < 30ms per tick (when triggered) | Includes sandbox simulation |
| Observation deserialization | < 2ms | Redis subscribe latency |
| Memory stability (1000 ticks) | No growth beyond O(log N) | ★ Buffer capped at 2000 records |
| ★ PredictionRecord buffer flush | < 200ms per batch | Background task, non-blocking |

---

## 11. Test Plan

### 11.1 Unit Tests (`tests/unit/test_predictive_loop/`)

★ Changes from v0.1: Added credit assignment, blame weight, dedup, buffer tests.

| Test | Validates |
|------|-----------|
| `test_error_signal_perfect_prediction` | Zero error → aggregate_error = 0.0 |
| `test_error_signal_complete_miss` | Maximum error → is_surprising = True |
| `test_error_signal_partial_error` | Partial error correctly computed per entity |
| `test_error_signal_novel_entity` | Unknown entity → novelty_penalty applied |
| ★ `test_blame_weight_high_uncertainty_absorbs_more` | High-uncertainty node gets higher blame_weight than low-uncertainty in same chain (Grok v2 explicit test) |
| ★ `test_blame_weight_strong_relationship_absorbs_more` | Higher causal_weight for stronger relationship |
| ★ `test_blame_weight_50_50_blend` | Combined weight = 0.5 * causal + 0.5 * uncertainty |
| `test_confidence_delta_correct_prediction` | Correct → small positive delta |
| `test_confidence_delta_wrong_prediction` | Wrong → larger negative delta |
| `test_confidence_delta_asymmetry` | Decrease > increase for same magnitude |
| ★ `test_confidence_delta_scaled_by_blame` | delta proportional to blame_weight |
| ★ `test_confidence_delta_scaled_by_prediction_confidence` | delta *= prediction.confidence |
| `test_confidence_bounded` | Confidence never exceeds [0.0, 1.0] |
| `test_confidence_high_stays_stable` | 0.95 + correct → tiny increase |
| `test_budget_allocation_adapts_slow` | Slow → reduced budget |
| `test_budget_allocation_adapts_fast` | Fast → increased budget |
| `test_budget_allocation_clamped` | Stays within min/max |
| ★ `test_budget_from_previous_not_default` | Adapts from previous_budget_nodes (GPT-5 Cond 5) |
| ★ `test_budget_safety_floor` | High total latency → min_budget_nodes (Grok Cond 4) |
| `test_prediction_filters_derived_rels` | derived_from_chain=True excluded |
| `test_prediction_deduplicates_mixed_resolution` | Concrete preferred over abstract |
| ★ `test_prediction_confidence_includes_rel_strengths` | Harmonic mean uses both node conf AND rel strength (Claude Obs 1) |
| `test_counterfactual_error_reduction_filter` | Zero-reduction hypotheses discarded |
| `test_counterfactual_cap` | Never exceeds max_counterfactuals_per_tick |
| ★ `test_counterfactual_dedup` | Duplicate signature → discarded (GPT-5 v2) |
| ★ `test_counterfactual_consistency_penalty` | Contradicting CONFIRMED → 0.5× penalty (GPT-5 Cond 3) |
| `test_tick_timing_all_phases` | All 4 phases + total populated |
| `test_tick_timing_total_ge_sum` | t_total ≥ sum of phases |
| `test_observation_deserialization` | Valid JSON → Observation |
| `test_observation_invalid_json` | Graceful error, tick skipped |
| ★ `test_prediction_record_buffer_enqueue` | O(1) enqueue, no blocking |
| ★ `test_prediction_record_buffer_hard_cap` | At 2000: drops oldest, logs warning |
| ★ `test_entity_match_exact` | Phase 1 exact match → confidence=1.0 |
| ★ `test_entity_match_novel` | No match → is_novel=True |

### 11.2 Integration Tests (`tests/integration/test_predictive_loop_integration.py`)

★ Changes from v0.1: Added credit assignment, BSM reconciliation, recalibration, novel hygiene, sandbox, dedup tests.

| Test | Validates |
|------|-----------|
| `test_single_tick_full_cycle` | predict → observe → error → update completes |
| `test_correct_prediction_increases_confidence` | Low error → confidence increases |
| `test_wrong_prediction_decreases_confidence` | High error → confidence decreases |
| ★ `test_stable_node_less_blame_than_uncertain` | A→B→C chain: high-uncertainty C penalized more than stable A (Gemini BLOCKING 1) |
| ★ `test_weak_prediction_weak_update` | Prediction at conf=0.3 → delta × 0.3 (GPT-5 Cond 1) |
| `test_surprising_error_generates_counterfactuals` | Error > threshold → hypotheses in quarantine |
| ★ `test_bsm_reconciliation_explicit` | Confidence crosses threshold → BSM transition fires WITHIN same tick (Grok Cond 1) |
| `test_prediction_record_stored` | ★ PredictionRecord enqueued; buffer flushes to Neo4j in background |
| `test_context_diversity_incremented` | New context → diversity +1 |
| ★ `test_novel_entity_dedup` | Same entity twice → one node, observation_count incremented (Grok Cond 5) |
| ★ `test_novel_entity_cap` | > 3 unknowns → only 3 created, rest logged |
| ★ `test_novel_entity_no_tier01_link` | Provisional rels cannot target Tier 0/1 |
| `test_100_tick_accuracy_improvement` | Aggregate error trend decreasing |
| `test_confounder_scenario_detects_hidden_variable` | Counterfactual identifies confounder |
| `test_loop_start_stop_graceful` | start → ticks → stop → no hanging tasks |
| `test_observation_gap_handling` | Missing observation → skipped, loop continues |
| `test_redis_reconnection` | Connection drops → recovers after backoff |
| `test_invariant_violation_during_update` | Tier 0 violation → rejected, tick continues |
| `test_calibration_collapse_pauses_loop` | Brier > 0.30 → loop pauses |
| ★ `test_calibration_drift_triggers_recalibration` | Brier > 0.20 → recalibration fires (Grok Cond 2) |
| `test_tick_stream_emits_results` | get_tick_stream() yields TickResults |
| `test_tier0_tier1_confidence_immutable` | Tier 0/1 skip confidence update |
| ★ `test_counterfactual_sandbox_isolation` | Simulation doesn't write to GraphStore (Gemini BLOCKING 3) |
| ★ `test_counterfactual_uses_pre_update_context` | Hypotheses generated from immutable snapshot (Grok Cond 3) |
| ★ `test_persistent_surprise_escalation` | 20+ surprising ticks → threshold raised, alert emitted (GPT-5 §H) |
| ★ `test_surprise_escalation_reset` | 5 non-surprising ticks → threshold restored (Grok v2) |
| ★ `test_hypothesis_dedup_prevents_spam` | Same signature within window → discarded (GPT-5 v2) |
| `test_mid_sprint_integration_checkpoint` | Simulator + Loop integrated end-to-end |

### 11.3 Stress Tests (`tests/stress/test_loop_stress.py`)

| Test | Validates |
|------|-----------|
| `test_1000_tick_sustained` | 1000 ticks without memory leak or degradation |
| `test_tick_latency_under_load` | p95 < 200ms with 10K-node graph |
| `test_concurrent_observation_bursts` | Rapid observations processed in order |
| `test_memory_stability_1000_ticks` | No unbounded growth (buffer capped) |
| ★ `test_prediction_record_buffer_under_neo4j_outage` | Buffer fills, drops oldest, recovers on reconnect |

### 11.4 Property-Based Tests (`tests/property/test_loop_properties.py`)

| Test | Validates |
|------|-----------|
| `test_confidence_always_bounded` | Any update sequence → confidence ∈ [0.0, 1.0] |
| `test_error_signal_non_negative` | Any prediction/observation → error ≥ 0.0 |
| `test_counterfactuals_always_reduce_error` | All surviving hypotheses have reduction > 0 |
| `test_tick_timing_consistent` | t_total ≥ sum of phase timings |
| `test_learning_monotonicity` | Many correct predictions → confidence monotonically increases |
| ★ `test_blame_weights_sum_to_one` | Blame weights across chain sum ≈ 1.0 (±tolerance) |

---

## 12. Acceptance Criteria

★ Expanded from 18 → 29 criteria.

| # | Criterion | Sprint | Verification |
|---|-----------|--------|-------------|
| 1 | Prediction engine generates predictions via causal traversal | S4 | Integration test |
| 2 | Error signal: quantified prediction-observation difference | S4 | Unit tests |
| 3 | ★ Weighted credit assignment: blame distributed by causal + uncertainty | S4 | Integration test (Gemini BLOCKING 1) |
| 4 | ★ Prediction confidence scales update magnitude | S4 | Integration test (GPT-5 Cond 1) |
| 5 | BSM transitions triggered by confidence threshold crossing | S4 | Integration test |
| 6 | ★ Explicit BSM reconciliation after every confidence update | S4 | Integration test (Grok Cond 1) |
| 7 | Full loop: 100 ticks with aggregate error decreasing | S4 | Integration test |
| 8 | Per-tick timing instrumentation (4 components + total) | S4 | All TickTiming fields populated |
| 9 | Tick rate configurable via ARIA_LOOP_ prefix | S4 | Config test |
| 10 | Observation gaps handled gracefully | S4 | Integration test |
| 11 | ★ PredictionRecord enqueued asynchronously (not in critical path) | S4 | Integration test (Gemini BLOCKING 2) |
| 12 | Context diversity updated per tick | S4 | Integration test |
| 13 | Counterfactual generation on surprising errors | S4 | Integration test |
| 14 | Error reduction rule enforced on counterfactuals | S4 | Unit test |
| 15 | ★ Counterfactual simulation runs in in-memory sandbox | S4 | Integration test (Gemini BLOCKING 3) |
| 16 | ★ Counterfactuals use pre-update context snapshot | S4 | Integration test (Grok Cond 3) |
| 17 | derived_from_chain relationships filtered from prediction | S4 | Unit + integration test |
| 18 | Mixed-resolution subgraphs deduplicated (concrete preferred) | S4 | Integration test |
| 19 | ★ Adaptive budget: memory + safety floor | S4 | Unit test (GPT-5 Cond 5 + Grok Cond 4) |
| 20 | Calibration collapse (Brier > 0.30) pauses loop | S4 | Integration test |
| 21 | ★ Calibration drift (Brier > 0.20) triggers recalibration | S4 | Integration test (Grok Cond 2) |
| 22 | Local Predictive Coding stub interface defined | S4 | Interface raises NotImplementedError |
| 23 | Mid-sprint integration checkpoint | S4 | Simulator + Loop integrated by week 9 |
| 24 | ★ Persistent surprise escalation: threshold raised after N ticks | S4 | Integration test (GPT-5 §H) |
| 25 | ★ Surprise escalation reset after improvement | S4 | Integration test (Grok v2) |
| 26 | ★ Novel entity hygiene: dedup + cap + decay + no Tier 0/1 links | S4 | Integration test (Grok Cond 5) |
| 27 | ★ EntityMatcher stub with match_confidence | S4 | Unit test (GPT-5 Q5 + Claude Obs 2) |
| 28 | ★ Hypothesis deduplication via signature | S4 | Unit test (GPT-5 v2) |
| 29 | ★ Prediction confidence includes relationship strengths | S4 | Unit test (Claude Obs 1) |

---

## 13. Open Questions — Resolved

★ All 8 original questions resolved through reviewer consensus.

| Q# | Question | Decision | Voters |
|----|----------|----------|--------|
| Q1 | Prediction confidence aggregation | Harmonic mean. ★ Include relationship strengths as inputs (Claude Obs 1, GPT-5 v2 refinement). | Unanimous |
| Q2 | Asymmetric confidence update | Keep 1.5×, expose in config. Phase 2+: dynamic asymmetry as confidence increases. | GPT-5 + Gemini |
| Q3 | Hypothesis type extensibility | Fixed 5 types for Phase 1. Extensible in Phase 3. | GPT-5 + Gemini (YAGNI) |
| Q4 | Novel entity confidence | ★ 0.4 (compromise: GPT-5 0.3, Gemini 0.5) | Compromise accepted |
| Q5 | EntityMatcher protocol | ★ Define stub now with match_confidence. Phase 1: exact match. | GPT-5 (refactor avoidance) |
| Q6 | Error history buffer | No ring buffer in loop. DriftMonitor owns trends. ★ Exception: small rolling window for surprise escalation only. | Unanimous |
| Q7 | Mental simulation formalization | ★ Extract CounterfactualSimulator protocol. Sandboxed. | GPT-5 + Gemini |
| Q8 | BOTH vs OUTGOING | BOTH extraction, OUTGOING prediction. Already correct in v0.1. | Unanimous |

---

## 14. ★ Review Feedback Incorporation Record (NEW)

### GPT-5 (OpenAI) — 5 Structural Conditions + 1 Missing Mode + 2 v2 Items

| Item | Resolution | Section |
|------|-----------|---------|
| Cond 1: Self-reinforcing confidence bias | delta *= prediction.confidence | §5.5, §5.2 Phase 4a |
| Cond 2: Credit assignment leakage | Weighted blame (causal + uncertainty) | §5.5 |
| Cond 3: Counterfactual narrative overfitting | Consistency check (contradiction penalty 0.5×) | §5.6 |
| Cond 4: Passive confidence decay | Deferred to CDD-01 (decay ownership) | §6 design note |
| Cond 5: Budget adaptation oscillation | Adapt from previous, not default | §5.7 |
| Missing mode: Persistent surprise | Escalation mechanism with threshold increase | §5.1, §8 |
| v2: Claude Obs 1 refinement | Rel strengths as harmonic mean inputs (not multipliers) | §5.3 |
| v2: Hypothesis deduplication | Signature-based dedup with rolling window | §5.6, §4.8 |

### Gemini (Google) — 3 BLOCKING Conditions (All Lifted)

| Item | Resolution | Section |
|------|-----------|---------|
| BLOCKING 1: Credit assignment / catastrophic forgetting | Weighted blame with causal + uncertainty weights | §5.5 |
| BLOCKING 2: PredictionRecord I/O death spiral | Async buffer with background flush | §4.10, §5.2 Phase 4c |
| BLOCKING 3: Mental simulation state leakage | CounterfactualSimulator on deep-copied subgraph | §3.1, §5.6, §4.9 |

### Grok (xAI) — 7 Conditions + 3 v2 Polish Items

| Item | Resolution | Section |
|------|-----------|---------|
| Cond 1: Explicit BSM reconciliation | evaluate_transitions() called after each update | §5.2 Phase 4a |
| Cond 2: Active CDD-06 recalibration | Periodic Brier check every N ticks | §5.1, §7 |
| Cond 3: Atomic update + counterfactual | Pre-update context + sandbox isolation | §5.6 |
| Cond 4: Budget strategy (bursty load) | Safety floor on high total latency | §5.7 |
| Cond 5: Novel entity hygiene | Dedup + cap(3) + elevated decay + no Tier 0/1 links | §5.2 Phase 4d |
| Cond 6: Loop state persistence | Deferred to Phase 2 (documented) | §7 |
| Cond 7: Config-driven thresholds | Cross-reference CDD-06, validation warnings | §7 |
| v2: Buffer hard cap | 2000 records, oldest-first drop | §4.10 |
| v2: Blame weight unit test | Explicit test for high-uncertainty absorbs more | §11.1 |
| v2: Surprise escalation reset | Reset after 5 non-surprising ticks | §5.1 |

### Claude Opus 4.6 — 3 Observations (All Confirmed)

| Item | Resolution | Section |
|------|-----------|---------|
| Obs 1: Prediction confidence includes rel strengths | Harmonic mean of node conf + rel strengths (GPT-5 v2 refined) | §5.3 |
| Obs 2: EntityMatch with match_confidence + is_novel | EntityMatch model with Phase 2+ future-proofing | §4.2 |
| Obs 3: TickResult error trend + consecutive surprising | Exposed for CDD-11 observability | §4.6 |

---

*End of CDD-04: Predictive Loop Engine*

**Project ARIA · Adaptive Reasoning Integrated Architecture**  
Component Design Document 04 — The Continuous Cognitive Cycle  
v1.0 — Approved by GPT-5 (OpenAI), Gemini (Google), Grok (xAI), Claude Opus 4.6 (Anthropic)
