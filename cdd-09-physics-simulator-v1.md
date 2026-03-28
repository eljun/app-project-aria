# CDD-09: Physics Simulator & Scenario Engine

> **Status:** APPROVED — All Reviewers Signed Off (4/4)  
> **Version:** 1.0  
> **Author:** Jun (Architect) + Claude Opus 4.6 (Design Partner)  
> **Depends On:** — (No CDD dependencies. Uses CDD-04 Observation schema.)  
> **Consumed By:** CDD-04 (Predictive Loop), CDD-07 (Adversary Simulator), CDD-11 (Observability)  
> **Sprint:** S4 (Predictive Loop + Physics Simulator + Observability)  
> **Classification:** Confidential — Core Team & Designated Review Partners  
> **Reviewed By:** GPT-5 (OpenAI) · Gemini (Google) · Grok (xAI) · Claude Opus 4.6 (Anthropic)  
> **Changes from v0.1:** 25 changes marked with ★

---

## 1. Purpose & Scope

ARIA learns by predicting and observing. In Phase 1, the "world" is a deterministic physics simulator that publishes observations to Redis for the Predictive Loop (CDD-04) to consume.

CDD-09 is ARIA's training environment — not a physics engine for production use. Its requirements are unusual for a simulator:

**1.1 Full Determinism** — Given the same scenario configuration and random seed, the simulator produces exactly the same sequence of observations every time. ★ Uses numpy PCG64 PRNG for cross-platform reproducibility. Canonical JSON serialization with fixed float precision. CI cross-runner determinism tests mandatory. (GPT-5 Cond 5 + Grok Cond 3)

**1.2 Injectable Confounders** — Every scenario has a hidden variable that activates at a configurable tick (or condition). Before activation, ARIA's predictions should be correct. After activation, predictions fail — and ARIA must explain the discrepancy. The confounder is the learning signal.

**1.3 YAML-Configured Scenarios** — Each of the 7 scenarios is defined entirely in a YAML file. No scenario-specific code. ★ Expressions use a safe whitelist enum — no `eval()`. New scenarios can be added by writing YAML alone. (GPT-5 BLOCKING 2 + Grok Cond 6)

**1.4 Relationship Type Coverage** — The 7 scenarios collectively exercise all 12 causal relationship types from v2.1 §4.2. This validates that CDD-01's schema can represent all the causal structures ARIA needs for Phase 1.

**1.5 Redis Streaming** — Each simulation step publishes an `Observation` (CDD-04 §4.1) to ★ Redis Streams (`aria.observation.physics`) with ordering guarantees and backpressure support. (Grok Cond 1 + GPT-5 BLOCKING 3)

★ **1.6 Epistemic Firewall** — The simulator enforces strict separation between what ARIA sees (Observation) and what the test runner sees (GroundTruth). Confounder state, hidden forces, and test metadata are NEVER included in published observations. (GPT-5 BLOCKING 1 + Grok Cond 2)

### 1.7 What This CDD Does NOT Cover

| Excluded Component | Covered In |
|-------------------|------------|
| Observation schema and deserialization | CDD-04 §4.1 |
| Predictive Loop consumption of observations | CDD-04 §5.1 |
| Adversarial variant generation from scenarios | CDD-07 |
| MuJoCo/PyBullet integration (Phase 5+) | Future CDD |
| Multi-domain observation sources | Phase 5+ |

---

## 2. Specification References

| Spec Section | Content | Relevance |
|-------------|---------|-----------|
| v2.1 §3.2 | Acid Test | Scenarios must support predict→fail→dream→update cycle |
| v2.1 §4.2 | 12 causal relationship types | All 12 must be exercised across scenarios |
| Impl Plan §2.10 | Custom Lightweight Simulator | Technology decision and rationale |
| Impl Plan §2.10 | 7 scenarios table | Scenario definitions and confounder types |
| Impl Plan §2.7 | Redis Pub/Sub | ★ Upgraded to Redis Streams for ordering |
| CDD-04 §4.1 | Observation model | ★ Schema for published observations (confounder fields removed) |
| CDD-04 §4.1 | EntityState model | State vector format |
| CDD-02 §5 | Tier 1 physics nodes | Ground truth that predictions are compared against |
| ★ CDD-02 §5.9 | t1-elasticity | Restitution model for collision scenarios |

---

## 3. Interface Contract

### 3.1 PhysicsSimulator Protocol

```python
# simulator/protocols.py

from __future__ import annotations
from typing import Protocol, AsyncIterator


class PhysicsSimulator(Protocol):
    """Deterministic physics simulation engine.

    Produces observations for the Predictive Loop.
    Not a production physics engine — a controlled training environment.
    """

    def load_scenario(
        self,
        scenario_path: str,
    ) -> ScenarioConfig:
        """Load a scenario from YAML.

        ★ Uses yaml.safe_load() — never yaml.Loader. (GPT-5 v2)
        Validates scenario structure via Pydantic (extra="forbid").
        ★ Validates schema_version field. (Claude Obs 3)
        ★ Validates MagnitudeType enum and ActivationCondition. (GPT-5 BLOCKING 2)
        Configures confounder injection.

        Raises:
            ScenarioLoadError: Invalid YAML, unknown schema version, or missing fields.
        """
        ...

    def reset(
        self,
        *,
        seed: int = 42,
    ) -> SimulatorState:
        """Reset simulator to initial conditions.

        Deterministic: same seed + same scenario = same sequence.

        ★ CLEARS: all entity states, accumulated_values,
        confounder_ramp_progress, confounder_active, tick_id (→ 0),
        elapsed_time (→ 0.0). REINITIALIZES: entities from
        ScenarioConfig, PRNG from numpy PCG64(seed).
        Post-condition: state is byte-identical to freshly-loaded scenario.
        (Gemini COND 2)
        """
        ...

    def step(self) -> TestObservation:
        """Advance simulation by one tick.

        ★ Returns TestObservation (includes ground-truth metadata).
        The publisher strips this to plain Observation before Redis.

        1. Apply forces to entities.
        2. ★ Run substeps if configured. (GPT-5 Cond 6)
        3. Update positions/velocities (Euler integration).
        4. ★ Collision detection (swept-sphere if configured). (GPT-5 Cond 4 + Gemini COND 1)
        5. Check confounder activation condition.
        6. If confounder active: apply hidden force/variable.
        7. Assemble TestObservation from current state.
        8. Increment tick counter.
        """
        ...

    async def run_and_publish(
        self,
        num_ticks: int,
        tick_interval_ms: int = 100,
    ) -> SimulationSummary:
        """Run N ticks and publish each observation to Redis Streams.

        ★ Uses XADD for ordered, persistent delivery. (Grok Cond 1)
        ★ Backpressure: pauses if queue > max_redis_queue_length. (Grok Cond 4)
        ★ Strips TestObservation → Observation before publish. (GPT-5 BLOCKING 1)
        ★ Safety: raises TestDataLeakError if TestObservation fields detected. (Grok v2)
        ★ If redis_required=False, writes to local NDJSON archive. (GPT-5 BLOCKING 3)
        """
        ...

    def get_ground_truth(
        self,
        tick_id: int,
    ) -> GroundTruth:
        """Return the full ground truth for a given tick.

        ★ Requires test_mode=True on simulator instance. (Grok Cond 5)
        Raises GroundTruthAccessError if test_mode=False.
        ★ Every call logged: structlog "simulator.ground_truth_accessed". (GPT-5 Obs 6)

        Includes hidden variables, actual forces, confounder state.
        Used by test assertions — NOT accessible to the Predictive Loop.
        """
        ...


class ConfounderInjector(Protocol):
    """Injects hidden variables into scenarios at configured ticks."""

    def is_active(self, tick_id: int) -> bool: ...

    def apply(self, state: SimulatorState, tick_id: int) -> SimulatorState:
        """Modify state with the hidden confounder's effect.
        NOT visible in the Observation's active_forces. Truly hidden.
        """
        ...


class CollisionDetector(Protocol):                                     # ★ NEW (GPT-5 Cond 4 + Gemini COND 1)
    """Detects collisions between entities."""

    def detect(
        self,
        entities: dict[str, EntityState],
        dt: float,
    ) -> list[CollisionEvent]:
        """Swept-sphere continuous detection. Prevents tunneling."""
        ...
```

### 3.2 Error Cases

| Error | Condition | Recovery |
|-------|-----------|----------|
| `ScenarioLoadError` | Invalid YAML, missing fields, unknown physics type, ★ unknown schema_version | Fail fast with descriptive error |
| `SimulationDivergenceError` | Entity exceeds safety bounds | ★ Clamp + emit structured "simulator.entity_clamped" event (GPT-5 Obs 8) |
| `RedisPublishError` | Redis unavailable during run_and_publish | ★ If redis_required: retry 3×, then raise. If !redis_required: write to NDJSON archive. (GPT-5 BLOCKING 3) |
| `DeterminismViolation` | Same seed produces different results | Test failure — indicates float or PRNG bug |
| ★ `TestDataLeakError` | TestObservation fields detected in publish path | Fail immediately — epistemic firewall breach (Grok v2) |
| ★ `GroundTruthAccessError` | get_ground_truth() called with test_mode=False | Fail — production cannot access hidden data (Grok Cond 5) |
| ★ `UnsafeExpressionError` | Unknown MagnitudeType or malformed ActivationCondition | Fail at YAML load time, not runtime (GPT-5 BLOCKING 2) |

---

## 4. Data Structures

### 4.1 ScenarioConfig

★ Changes from v0.1: Added schema_version, substeps, noise, expected_discovery, collision model, safe expression types.

```python
class ScenarioConfig(BaseModel):
    """A complete physics scenario loaded from YAML."""
    model_config = ConfigDict(extra="forbid")

    schema_version: str = Field(                                        # ★ NEW (Claude Obs 3, GPT-5 Obs 10)
        default="1.0",
        description="YAML schema version. Loader rejects unknown versions.",
    )
    scenario_id: str = Field(
        ...,
        description="Unique identifier (e.g., 'freefall_air_resistance').",
    )
    description: str = Field(
        ..., max_length=500,
        description="Human-readable description of what this scenario tests.",
    )
    total_ticks: int = Field(default=100, ge=10, le=10000)

    # ── Entities ───────────────────────────────────────────────────
    entities: list[EntityConfig] = Field(..., min_length=1)

    # ── Physics ────────────────────────────────────────────────────
    physics_type: str = Field(
        ...,
        pattern=r"^(newtonian|threshold|correlation)$",
    )
    dt: float = Field(default=0.01)
    integration_method: str = Field(                                    # ★ NEW (GPT-5 Q1)
        default="euler",
        pattern=r"^(euler|symplectic_euler|rk4)$",
        description="Phase 1: euler default. RK4 stubbed for Phase 2.",
    )
    simulation_substeps_per_observation: int = Field(                   # ★ NEW (GPT-5 Cond 6)
        default=1, ge=1,
        description="Inner physics steps per published observation.",
    )
    global_forces: list[ForceConfig] = Field(default_factory=list)

    # ── Collision ──────────────────────────────────────────────────
    collision_model: str = Field(                                       # ★ NEW (GPT-5 Cond 4 + Gemini COND 1)
        default="none",
        pattern=r"^(none|swept_sphere)$",
    )
    restitution_coefficient: float = Field(                             # ★ NEW
        default=0.8, ge=0.0, le=1.0,
        description="Coefficient of restitution for collision scenarios.",
    )

    # ── Confounder ─────────────────────────────────────────────────
    confounder: ConfounderConfig

    # ── Noise ──────────────────────────────────────────────────────
    noise: NoiseConfig = Field(default_factory=NoiseConfig)             # ★ NEW (GPT-5 Cond 7)

    # ── Expected Outcomes ──────────────────────────────────────────
    expected_relationship_types: list[str] = Field(..., min_length=1)
    expected_discovery: ExpectedDiscovery | None = Field(               # ★ NEW (Claude Obs 1)
        default=None,
        description="Expected causal graph ARIA should discover. Used by Acid Test.",
    )
    expected_prediction_accuracy_before_confounder: float = Field(default=0.8)
    expected_prediction_accuracy_after_confounder: float = Field(default=0.3)
    seed: int = Field(default=42)


class EntityConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")
    entity_id: str
    label: str
    initial_position: tuple[float, float, float] = (0.0, 0.0, 0.0)
    initial_velocity: tuple[float, float, float] = (0.0, 0.0, 0.0)
    mass: float = Field(default=1.0, gt=0.0)
    properties: dict[str, float] = Field(default_factory=dict)
    observable: bool = Field(default=True)
    radius: float = Field(                                              # ★ NEW (for swept-sphere)
        default=0.1, gt=0.0,
        description="Entity radius for collision detection.",
    )
```

### ★ 4.2 Safe Force & Confounder Configuration (GPT-5 BLOCKING 2 + Grok Cond 6)

```python
class MagnitudeType(str, Enum):                                        # ★ NEW — replaces free-form string
    """Safe whitelist of magnitude computation methods.
    No eval(). No arbitrary expressions."""
    CONSTANT = "constant"
    PROPORTIONAL_TO_VELOCITY = "proportional_to_velocity"
    PROPORTIONAL_TO_MASS = "proportional_to_mass"
    THRESHOLD = "threshold"
    CENTRIPETAL = "centripetal"


class ForceConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")
    force_id: str
    label: str
    direction: tuple[float, float, float] = (0.0, -9.81, 0.0)
    magnitude_expression: MagnitudeType = MagnitudeType.CONSTANT        # ★ CHANGED: enum, not string
    applies_to: list[str] | None = None
    visible: bool = True


class ActivationCondition(BaseModel):                                  # ★ NEW — replaces free-form string
    """Safe structured condition for delayed confounders.
    No eval(). Validated at YAML load time."""
    model_config = ConfigDict(extra="forbid")
    entity_id: str
    property_name: str        # e.g., "temperature"
    comparison: str = Field(pattern=r"^(greater_than|less_than|equal_to)$")
    threshold_value: float


class ManipulationConfig(BaseModel):                                   # ★ NEW (Grok Cond 6 — Scenario 7)
    """Mid-scenario manipulation for forcing contradictions."""
    model_config = ConfigDict(extra="forbid")
    manipulation_tick: int = Field(..., ge=1)
    action: str
    description: str


class ConfounderConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")
    confounder_id: str
    label: str
    confounder_type: str = Field(pattern=r"^(immediate|delayed|correlation)$")
    activation_tick: int = Field(..., ge=1)
    activation_condition: ActivationCondition | None = None             # ★ CHANGED: structured, not string
    effect: ForceConfig
    ramp_ticks: int = Field(default=0)
    manipulation: ManipulationConfig | None = None                      # ★ NEW (Scenario 7)


class NoiseConfig(BaseModel):                                          # ★ NEW (GPT-5 Cond 7)
    """Optional observation noise. Disabled in Phase 1."""
    model_config = ConfigDict(extra="forbid")
    enabled: bool = False
    position_sigma: float = 0.0
    velocity_sigma: float = 0.0
    seed: int = 0  # Separate PRNG for reproducible noise
```

### ★ 4.3 ExpectedDiscovery (NEW — Claude Obs 1)

```python
class ExpectedDiscovery(BaseModel):
    """What ARIA should learn from this scenario.
    Used by Acid Test for structural causal graph assertions.
    """
    model_config = ConfigDict(extra="forbid")

    expected_nodes: list[str] = Field(
        ...,
        description="Labels of nodes ARIA should create/discover.",
    )
    expected_relationships: list[tuple[str, str, str]] = Field(
        ...,
        description="(source_label, rel_type, target_label) tuples.",
    )
    confounder_discoverable: bool = Field(
        default=True,
        description="Whether the confounder is identifiable from observations.",
    )
    matching_mode: str = Field(                                         # ★ GPT-5 v2 suggestion
        default="strict",
        pattern=r"^(strict|fuzzy)$",
        description="strict: exact label match. fuzzy: alias/substring match.",
    )
```

### ★ 4.4 CollisionEvent (NEW — GPT-5 Cond 4 + Gemini COND 1)

```python
class CollisionEvent(BaseModel):
    model_config = ConfigDict(extra="forbid")
    entity_a_id: str
    entity_b_id: str
    collision_time: float     # Fraction of dt at which collision occurred
    collision_point: tuple[float, float, float]
    restitution: float        # From scenario config
```

### 4.5 Observation & TestObservation

★ Critical change from v0.1: `confounder_active` and `metadata` REMOVED from published Observation. (GPT-5 BLOCKING 1 + Grok Cond 2)

```python
class Observation(BaseModel):
    """An environmental observation published to Redis.

    ★ This is what ARIA sees. No hidden variables, no test metadata.
    Pure sensory payload — only what a real sensor would report.
    """
    model_config = ConfigDict(extra="forbid")

    observation_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    scenario_id: str
    tick_id: int = Field(..., ge=0)
    timestamp: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    entity_states: dict[str, EntityState]
    active_forces: list[str] = Field(default_factory=list)
    # ★ REMOVED: confounder_active, metadata — epistemic firewall


class TestObservation(Observation):                                    # ★ NEW (GPT-5 BLOCKING 1 + Grok Cond 2)
    """Extended observation with ground-truth metadata.

    ONLY used internally by step() and in test harnesses.
    NEVER published to Redis. Publisher strips to plain Observation.
    ★ Safety: isinstance(TestObservation) check before publish. (Grok v2)
    """
    confounder_active: bool = False
    metadata: dict[str, str] = Field(default_factory=dict)
    ground_truth_forces: list[str] = Field(default_factory=list)
```

### 4.6 SimulatorState

```python
class SimulatorState(BaseModel):
    model_config = ConfigDict(extra="forbid")
    tick_id: int = 0
    elapsed_time: float = 0.0
    entities: dict[str, EntityState]
    active_forces: list[str]
    confounder_active: bool = False
    confounder_ramp_progress: float = 0.0
    accumulated_values: dict[str, float] = Field(default_factory=dict)
```

### 4.7 GroundTruth

```python
class GroundTruth(BaseModel):
    """Full ground truth including hidden variables.
    ★ Only accessible via get_ground_truth() with test_mode=True.
    """
    model_config = ConfigDict(extra="forbid")
    tick_id: int
    all_forces: list[str]
    confounder_active: bool
    confounder_magnitude: float
    true_entity_states: dict[str, EntityState]
    expected_relationship_types: list[str]
```

### 4.8 SimulationSummary

★ Changes from v0.1: Added actual activation tick, provenance metadata.

```python
class SimulationSummary(BaseModel):
    model_config = ConfigDict(extra="forbid")
    scenario_id: str
    total_ticks: int
    ticks_before_confounder: int
    ticks_after_confounder: int
    confounder_activation_tick: int        # Configured tick
    actual_confounder_activation_tick: int  # ★ NEW: actual tick (may differ for condition-based) (Claude Obs 2)
    activation_cause: str = ""             # ★ NEW: "tick_based" or condition description (GPT-5 v2)
    observations_published: int
    publish_failures: int
    total_duration_ms: float
    seed: int
    # ★ Provenance metadata (GPT-5 v2)
    python_version: str = ""
    numpy_version: str = ""
    host_id: str = ""
    run_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
```

---

## 5. Algorithm / Logic Flow

### 5.1 Simulation Step

★ Changes from v0.1: Substeps, swept-sphere collision, safe expression evaluation, TestObservation output, clamp events.

```
step() called
    │
    ├── tick_id += 1
    ├── elapsed_time += dt
    │
    ├── ★ For substep in range(simulation_substeps_per_observation):  (GPT-5 Cond 6)
    │
    │   ├── Phase 1: Apply global forces
    │   │     For each force in scenario.global_forces:
    │   │       If force.visible: add to active_forces list
    │   │       For each applicable entity:
    │   │         ★ Compute force via MagnitudeType enum (no eval):
    │   │           CONSTANT: F = force.direction * entity.mass
    │   │           PROPORTIONAL_TO_VELOCITY: F = force.direction * |velocity|
    │   │           PROPORTIONAL_TO_MASS: F = force.direction * entity.mass
    │   │           THRESHOLD: F = 0 if value < threshold, else force.direction
    │   │           CENTRIPETAL: F = computed from string length + velocity
    │   │         Accumulate force on entity
    │   │
    │   ├── Phase 2: Confounder evaluation
    │   │     If confounder_type == "immediate":
    │   │       If tick_id >= confounder.activation_tick: confounder_active = True
    │   │     
    │   │     If confounder_type == "delayed":
    │   │       Update accumulated_values from current state
    │   │       ★ Evaluate ActivationCondition (structured, not eval):
    │   │         Read entity.properties[condition.property_name]
    │   │         Compare against condition.threshold_value
    │   │       If condition met: confounder_active = True
    │   │         ramp_progress = min(1.0, (tick - activation_tick) / ramp_ticks)
    │   │     
    │   │     If confounder_type == "correlation":
    │   │       If tick_id >= confounder.activation_tick: confounder_active = True
    │   │
    │   │     ★ If manipulation defined AND tick_id >= manipulation.manipulation_tick:
    │   │       Apply manipulation action (e.g., block_light_only for Scenario 7)
    │   │       (Grok Cond 6 — forces genuine CONTRADICTS)
    │   │
    │   ├── Phase 3: Apply confounder (if active)
    │   │     ConfounderInjector.apply(state, tick_id)
    │   │     Hidden force NOT added to visible active_forces
    │   │
    │   ├── Phase 4: Physics integration (Euler method)
    │   │     For each entity:
    │   │       total_force = sum(all accumulated forces)
    │   │       acceleration = total_force / entity.mass
    │   │       velocity += acceleration * (dt / substeps)
    │   │       position += velocity * (dt / substeps)
    │   │       
    │   │       Bounds check: clamp position and velocity to safety limits
    │   │       ★ If clamped: emit structlog "simulator.entity_clamped" (GPT-5 Obs 8)
    │   │
    │   └── ★ Phase 5: Collision detection (if collision_model == "swept_sphere")
    │         collisions = CollisionDetector.detect(entities, dt/substeps)
    │         For each collision:
    │           Apply momentum transfer with restitution_coefficient
    │           (Standard elastic/inelastic collision physics)
    │
    ├── Phase 6: Assemble TestObservation
    │     observation = TestObservation(
    │       observation_id=uuid(),
    │       scenario_id=scenario.scenario_id,
    │       tick_id=tick_id,
    │       entity_states={observable entities only},
    │       active_forces=[visible force labels only],
    │       ★ confounder_active=confounder_active,  # TestObservation only
    │       ★ ground_truth_forces=[all force labels including hidden],
    │     )
    │
    │     ★ If noise.enabled:
    │       Apply Gaussian noise to positions/velocities
    │       Using separate noise PRNG (noise.seed)
    │
    ▼
Return TestObservation
```

### 5.2 Run and Publish

★ Changes from v0.1: Redis Streams, backpressure, TestObservation stripping, local archive fallback, safety gate.

```
run_and_publish(num_ticks, tick_interval_ms) called
    │
    ├── If config.redis_required:
    │     Connect to Redis
    ├── Else:
    │     ★ Open NDJSON archive file: ./simulator/archives/{scenario_id}/{seed}.ndjson
    │
    ├── Initialize: published = 0, failures = 0, start_timer
    │
    ├── For tick in range(num_ticks):
    │     test_obs = step()  # Returns TestObservation
    │     
    │     ★ Strip to plain Observation (epistemic firewall):
    │       observation = Observation(**{
    │         k: v for k, v in test_obs.dict().items()
    │         if k in Observation.model_fields
    │       })
    │     
    │     ★ Safety gate (Grok v2):
    │       assert not isinstance(observation, TestObservation)
    │       assert "confounder_active" not in observation.dict()
    │       If fails: raise TestDataLeakError
    │     
    │     ★ Canonical serialization (GPT-5 Cond 5 + Grok Cond 3):
    │       json_bytes = json.dumps(
    │         observation.dict(), sort_keys=True, ensure_ascii=True,
    │         default=canonical_float_serializer  # round(v, 8)
    │       )
    │     
    │     If config.redis_required:
    │       ★ Backpressure check (Grok Cond 4):
    │         queue_length = await redis.xlen(channel)
    │         If queue_length > max_redis_queue_length:
    │           emit "simulator.backpressure_applied"
    │           ★ Sleep with jitter (GPT-5 v2)
    │           Wait until queue drops below threshold/2
    │       
    │       ★ Publish via XADD (Grok Cond 1):
    │         await redis.xadd(channel, {"data": json_bytes})
    │     Else:
    │       Write json_bytes + "\n" to NDJSON archive
    │     
    │     published += 1
    │     await asyncio.sleep(tick_interval_ms / 1000)
    │
    ├── stop_timer
    │
    ▼
Return SimulationSummary(
    ...,
    ★ actual_confounder_activation_tick=actual_tick,
    ★ activation_cause=cause_description,
    ★ python_version, numpy_version, host_id, run_id,
)
```

### 5.3 Determinism Contract

★ Significantly strengthened from v0.1 based on GPT-5 Cond 5 + Grok Cond 3.

1. **No threading:** All computation is single-threaded sequential.
2. **★ PRNG:** `numpy.random.Generator(PCG64(seed))` — not Python's `random.Random()`. Cross-platform deterministic. (GPT-5 Cond 5)
3. **No time-dependent state:** `elapsed_time = tick_id * dt`, not wall clock.
4. **★ Canonical serialization:** `json.dumps(sort_keys=True, ensure_ascii=True)` with all floats rounded to 8 decimal places via `round(value, 8)`. (GPT-5 Cond 5 + Grok Cond 3)
5. **★ Canonical hashing:** Determinism tests compute `SHA-256(canonical_json_bytes)` and assert equality between runs. (GPT-5 Cond 5)
6. **★ Cross-runner CI:** Mandatory CI job runs each scenario twice on different runners. Hash comparison must match. (GPT-5 Cond 5)
7. **★ Provenance recording:** `SimulationSummary` records Python version, numpy version, host_id, and run_id for diagnosing cross-platform mismatches. (GPT-5 v2)
8. **Float safety:** Basic arithmetic only (no transcendental functions in Phase 1). Use `math.fsum` for summation.
9. **★ Noise determinism:** When noise.enabled, noise PRNG is separate from physics PRNG. Both seeded deterministically. (GPT-5 Cond 7)

---

## 6. The 7 Scenarios

### 6.1 Scenario 1: Free-Fall with Hidden Air Resistance

| Property | Value |
|----------|-------|
| scenario_id | freefall_air_resistance |
| physics_type | newtonian |
| Confounder | Air resistance (immediate, tick 30) |
| Expected types | CAUSES, PREVENTS, CONFOUNDER |
| Expected discovery | Gravity CAUSES fall; air resistance PREVENTS full acceleration; air resistance is CONFOUNDER |

### 6.2 Scenario 2: Collision with Hidden Mass Difference

| Property | Value |
|----------|-------|
| scenario_id | collision_hidden_mass |
| physics_type | newtonian |
| ★ collision_model | swept_sphere (Gemini COND 1) |
| ★ restitution | 0.8 (partially inelastic, per CDD-02 t1-elasticity) (GPT-5 Obs 9) |
| Confounder | Hidden extra mass (immediate, tick 40) |
| Expected types | CAUSES, REQUIRES, MEDIATOR |

### 6.3 Scenario 3: Projectile with Hidden Wind

| Property | Value |
|----------|-------|
| scenario_id | projectile_wind |
| physics_type | newtonian |
| Confounder | Crosswind (immediate, tick 25) |
| Expected types | PROBABILISTICALLY_CAUSES, CONFOUNDER, ENABLES |

### 6.4 Scenario 4: Ramp Roll with Hidden Friction Change

| Property | Value |
|----------|-------|
| scenario_id | ramp_friction |
| physics_type | newtonian |
| Confounder | Friction coefficient increase (immediate, tick 35) |
| Expected types | STOCHASTIC_THRESHOLD, IS_PART_OF, PRECEDES |

### 6.5 Scenario 5: Pendulum with Hidden Damping

| Property | Value |
|----------|-------|
| scenario_id | pendulum_damping |
| physics_type | newtonian |
| Confounder | Damping force (immediate, tick 40) |
| Expected types | CAUSES, CORRELATES_WITH, CONTRADICTS |

### 6.6 Scenario 6: Temperature Threshold — Delayed Effect (GPT-5)

| Property | Value |
|----------|-------|
| scenario_id | temperature_threshold |
| physics_type | threshold |
| ★ Confounder type | delayed (condition: surface.temperature > 50.0) |
| ★ activation_condition | ActivationCondition(entity_id="surface", property_name="temperature", comparison="greater_than", threshold_value=50.0) |
| ramp_ticks | 10 |
| Expected types | STOCHASTIC_THRESHOLD, PRECEDES, MEDIATOR |

### 6.7 Scenario 7: Correlated Variables + Hidden Heat Source (Grok)

| Property | Value |
|----------|-------|
| scenario_id | correlation_heat_source |
| physics_type | correlation |
| Confounder | Hidden heat lamp (common cause, immediate, tick 20) |
| ★ manipulation | tick 60: block light, keep heat → forces genuine CONTRADICTS (Grok Cond 6) |
| Expected types | CORRELATES_WITH, CONFOUNDER, CONTRADICTS |

### 6.8 Relationship Coverage Matrix

| Relationship Type | S1 | S2 | S3 | S4 | S5 | S6 | S7 | Total |
|---|---|---|---|---|---|---|---|---|
| CAUSES | ✓ | ✓ | | | ✓ | | | 3 |
| PROBABILISTICALLY_CAUSES | | | ✓ | | | | | 1 |
| ENABLES | | | ✓ | | | | | 1 |
| PREVENTS | ✓ | | | | | | | 1 |
| REQUIRES | | ✓ | | | | | | 1 |
| IS_PART_OF | | | | ✓ | | | | 1 |
| PRECEDES | | | | ✓ | | ✓ | | 2 |
| CONTRADICTS | | | | | ✓ | | ★ ✓ | 2 |
| CORRELATES_WITH | | | | | ✓ | | ✓ | 2 |
| CONFOUNDER | ✓ | | ✓ | | | | ✓ | 3 |
| MEDIATOR | | ✓ | | | | ✓ | | 2 |
| STOCHASTIC_THRESHOLD | | | | ✓ | | ✓ | | 2 |

**All 12 relationship types covered.** ★ Scenario 7 CONTRADICTS now genuinely exercised via manipulation phase (Grok Cond 6).

---

## 7. Configuration Parameters

★ Changes from v0.1: Added Redis Streams config, backpressure, archive path, test_mode.

```python
class SimulatorConfig(BaseSettings):
    """Configuration for the Physics Simulator."""

    # ── Simulation ─────────────────────────────────────────────────
    default_dt: float = 0.01
    default_total_ticks: int = 100
    default_seed: int = 42
    safety_position_limit: float = 1000.0
    safety_velocity_limit: float = 100.0

    # ── Redis Streaming ────────────────────────────────────────────
    redis_channel: str = "aria.observation.physics"
    redis_required: bool = True                                         # ★ NEW (GPT-5 BLOCKING 3)
    redis_publish_retries: int = 3
    max_redis_queue_length: int = 50                                    # ★ NEW (Grok Cond 4)
    backpressure_jitter_ms: int = 10                                    # ★ NEW (GPT-5 v2)
    tick_interval_ms: int = 100

    # ── Archive ────────────────────────────────────────────────────
    archive_directory: str = "./simulator/archives/"                     # ★ NEW (GPT-5 BLOCKING 3)

    # ── Scenarios ──────────────────────────────────────────────────
    scenario_directory: str = "simulator/scenarios/"

    # ── Test Mode ──────────────────────────────────────────────────
    test_mode: bool = True                                              # ★ NEW (Grok Cond 5)

    model_config = SettingsConfigDict(env_prefix="ARIA_SIM_")
```

---

## 8. Error Handling & Edge Cases

★ Changes from v0.1: Added clamp events, tunneling prevention, TestObservation safety, archive fallback.

| Edge Case | Behavior | Rationale |
|-----------|----------|-----------|
| Entity exits safety bounds | Clamp + ★ emit "simulator.entity_clamped" event (GPT-5 Obs 8) | Prevents divergence; CDD-04 can treat clamped states as noise |
| Confounder activation_tick > total_ticks | Confounder never activates, log warning | Valid — tests no-confounder baseline |
| Zero mass entity | Reject at YAML load time | Division by zero in F=ma |
| ★ Unknown MagnitudeType in YAML | Reject at load time (UnsafeExpressionError) | Safe whitelist enforcement |
| Redis unavailable at start | ★ If redis_required: fail fast. If !redis_required: use archive. | Configurable resilience |
| ★ Redis queue overflow | Backpressure: pause + jitter + emit telemetry (Grok Cond 4) | Prevents consumer overwhelm |
| ★ Fast objects tunneling through each other | Swept-sphere detection prevents tunneling (Gemini COND 1) | Scenario 2 depends on collision detection |
| Delayed confounder condition never met | Confounder never activates, log info | Valid — variable didn't reach threshold |
| ★ TestObservation detected in publish path | Raise TestDataLeakError immediately (Grok v2) | Epistemic firewall breach |
| ★ get_ground_truth() in production mode | Raise GroundTruthAccessError (Grok Cond 5) | Hidden data must stay hidden |
| Float precision drift | ★ round(v, 8), canonical JSON, PCG64 (GPT-5 Cond 5) | Cross-platform determinism |
| YAML has extra fields | Rejected by Pydantic extra="forbid" | Prevents silent misconfiguration |
| ★ YAML uses yaml.Loader | ★ Always yaml.safe_load() (GPT-5 v2) | Prevents arbitrary object instantiation |

---

## 9. Module Structure

★ Changes from v0.1: Added collision, archiver, canonical serialization utility.

```
simulator/
├── __init__.py
├── protocols.py              # PhysicsSimulator, ConfounderInjector, CollisionDetector
├── engine.py                 # PhysicsEngine: step(), reset(), integration
├── publisher.py              # ★ Redis Streams publication + NDJSON archive fallback
├── confounder.py             # ConfounderInjector implementations
├── collision.py              # ★ SweptSphereDetector (Gemini COND 1)
├── loader.py                 # YAML loading with safe_load + Pydantic validation
├── ground_truth.py           # GroundTruth generation (test_mode gated)
├── canonical.py              # ★ Canonical JSON serialization + SHA-256 hashing
├── config.py                 # SimulatorConfig
├── scenarios/
│   ├── freefall_air_resistance.yml
│   ├── collision_hidden_mass.yml
│   ├── projectile_wind.yml
│   ├── ramp_friction.yml
│   ├── pendulum_damping.yml
│   ├── temperature_threshold.yml
│   └── correlation_heat_source.yml
└── stubs/
    └── mujoco_stub.py        # MuJoCo interface stub (NotImplementedError)
```

---

## 10. Performance Targets

| Metric | Phase 1 Target | Measurement |
|--------|---------------|-------------|
| Single step() execution | < 1ms | Timer within step |
| 100-tick simulation (no Redis) | < 100ms total | Unit test |
| 1000-tick simulation (with Redis) | < 10s total | Integration test |
| Redis XADD latency (per observation) | < 5ms | Publisher timing |
| Scenario YAML load time | < 50ms | Loader timing |
| ★ Swept-sphere collision detection | < 0.5ms per step | For collision scenarios |
| Determinism: two runs produce identical SHA-256 | Exact match | ★ Cross-runner CI test |

---

## 11. Test Plan

### 11.1 Unit Tests (`tests/unit/test_simulator/`)

★ Changes from v0.1: Added safe expression, collision, determinism hash, clamp, reset, TestObservation safety tests.

| Test | Validates |
|------|-----------|
| `test_freefall_no_confounder` | Ball falls with constant acceleration (gravity only) |
| `test_freefall_with_air_resistance` | After confounder tick, acceleration decreases |
| `test_collision_symmetric` | Equal-mass collision produces symmetric result |
| `test_collision_asymmetric` | Hidden mass produces asymmetric result |
| ★ `test_collision_no_tunneling` | Fast objects detected by swept-sphere (parametrized dt) |
| `test_confounder_activation_tick` | Confounder activates exactly at configured tick |
| `test_delayed_confounder_ramp` | Delayed confounder ramps from 0 to full |
| ★ `test_delayed_confounder_condition` | Condition-based activation fires at correct tick |
| `test_correlation_scenario_no_causation` | Correlated variables without direct causal link |
| ★ `test_scenario7_manipulation_contradicts` | After manipulation tick, light/temp correlation breaks |
| `test_scenario_yaml_loading` | All 7 scenarios load without error |
| `test_scenario_yaml_extra_fields_rejected` | Extra YAML fields cause ScenarioLoadError |
| ★ `test_unknown_magnitude_type_rejected` | Invalid MagnitudeType raises at load time |
| ★ `test_malformed_activation_condition_rejected` | Invalid ActivationCondition raises at load time |
| `test_entity_bounds_clamping` | Entity exceeding limit is clamped |
| ★ `test_clamp_emits_event` | Clamping emits structlog event |
| `test_zero_mass_rejected` | Zero mass entity rejected at load |
| `test_observation_matches_cdd04_schema` | step() output (stripped) is valid CDD-04 Observation |
| `test_ground_truth_includes_hidden` | get_ground_truth() includes confounder forces |
| ★ `test_ground_truth_requires_test_mode` | test_mode=False raises GroundTruthAccessError |
| ★ `test_published_observation_no_confounder` | Published JSON has no confounder_active field |
| ★ `test_test_observation_detected_in_publish` | isinstance(TestObservation) raises TestDataLeakError |
| ★ `test_reset_produces_identical_state` | Load + run 50 + reset = freshly loaded state |
| ★ `test_yaml_safe_load` | yaml.safe_load used, yaml.Loader rejected |
| ★ `test_schema_version_validated` | Unknown schema_version raises ScenarioLoadError |

### 11.2 Integration Tests (`tests/integration/test_simulator_integration.py`)

| Test | Validates |
|------|-----------|
| ★ `test_redis_stream_publication` | Observations published via XADD to correct stream |
| ★ `test_redis_stream_ordering` | XREAD returns observations in tick order |
| `test_redis_deserialization` | Published JSON deserializes to valid Observation |
| `test_100_tick_complete_run` | run_and_publish(100) completes without error |
| `test_simulator_loop_integration` | Simulator publishes → CDD-04 loop receives → processes |
| `test_all_7_scenarios_run` | Each scenario loads, runs to completion |
| `test_confounder_causes_prediction_failure` | ARIA's accuracy drops after confounder activation |
| ★ `test_ndjson_archive_fallback` | redis_required=False → observations archived to file |
| ★ `test_archive_replay_idempotency` | Replaying archive produces same CDD-04 results as live (GPT-5 v2) |
| ★ `test_backpressure_pauses_simulator` | Queue > limit → simulator pauses, resumes on drain |

### 11.3 Property-Based Tests (`tests/property/test_simulator_properties.py`)

| Test | Validates |
|------|-----------|
| ★ `test_determinism_all_scenarios_sha256` | Same seed → identical SHA-256 hash of observation sequence |
| ★ `test_cross_runner_determinism` | CI: two different runners produce identical hashes |
| `test_energy_conservation_pre_confounder` | Total energy approximately conserved (tolerance tied to dt) |
| `test_observation_always_valid` | For any tick: observation matches Pydantic schema |
| `test_confounder_hidden_from_observation` | Confounder forces never appear in active_forces |

### 11.4 Coverage Tests (`tests/coverage/test_relationship_coverage.py`)

| Test | Validates |
|------|-----------|
| `test_all_12_relationship_types_covered` | Union across 7 scenarios = all 12 types |
| `test_each_scenario_exercises_at_least_2_types` | No scenario < 2 types |
| `test_immediate_delayed_correlation_all_present` | At least 1 of each confounder type |

---

## 12. Acceptance Criteria

★ Expanded from 12 → 18 criteria.

| # | Criterion | Sprint | Verification |
|---|-----------|--------|-------------|
| 1 | 7 scenarios implemented as YAML | S4 | All 7 load without error |
| 2 | Each scenario deterministic (same seed = same SHA-256) | S4 | ★ Cross-runner property test |
| 3 | Confounders injectable at configurable ticks/conditions | S4 | Unit tests |
| 4 | Delayed-effect confounder works (Scenario 6) | S4 | Unit test: condition-based activation |
| 5 | Non-causal correlation scenario works (Scenario 7) | S4 | Unit test |
| 6 | ★ Scenario 7 manipulation forces genuine CONTRADICTS | S4 | Unit test (Grok Cond 6) |
| 7 | Observations published to ★ Redis Streams as canonical JSON | S4 | Integration test |
| 8 | ★ Observation schema has NO confounder fields (epistemic firewall) | S4 | Unit + integration test |
| 9 | All 12 relationship types covered across scenarios | S4 | Coverage test |
| 10 | Ground truth available for test assertions (test_mode only) | S4 | Unit test |
| 11 | ★ Confounder forces hidden from published observations | S4 | Property test |
| 12 | MuJoCo interface stub defined | S4 | Stub exists, raises NotImplementedError |
| 13 | Mid-sprint: simulator integrated with CDD-04 loop | S4 | End-to-end integration |
| 14 | ★ Swept-sphere collision detection for Scenario 2 | S4 | Unit test: no tunneling |
| 15 | ★ Safe expression evaluation (no eval) | S4 | Unit test: unknown types rejected |
| 16 | ★ Redis optional with NDJSON archive fallback | S4 | Integration test |
| 17 | ★ Backpressure mechanism functional | S4 | Integration test |
| 18 | ★ ExpectedDiscovery defined for Acid Test scenarios | S4 | YAML validation |

---

## 13. Open Questions — Resolved

| Q# | Question | Decision | Voters |
|----|----------|----------|--------|
| Q1 | Integration method | Euler default, configurable (RK4 stubbed). Document stability limits. | GPT-5 + Gemini |
| Q2 | Collision detection | ★ Swept-sphere for collision scenarios. AABB removed. | GPT-5 + Gemini (unanimous) |
| Q3 | Tick vs condition triggers | Both supported. ★ Safe ActivationCondition model, no eval. | Unanimous |
| Q4 | Observation frequency | ★ Add simulation_substeps_per_observation, default=1. | GPT-5 |
| Q5 | Difficulty progression | Independent execution. No sequential dependency Phase 1. | Gemini (strong) + GPT-5 |
| Q6 | Noise injection | ★ NoiseConfig toggle, disabled by default. Not active in S4 tests. | GPT-5 (add) vs Gemini (no) → compromise |

---

## 14. ★ Review Feedback Incorporation Record (NEW)

### GPT-5 (OpenAI) — 3 Blocking + 4 High + 6 Medium/Low + v2 Polish

| Item | Resolution | Section |
|------|-----------|---------|
| BLOCKING 1: Confounder leak in Observation | TestObservation split, publish-time safety gate | §4.5, §5.2 |
| BLOCKING 2: Unsafe eval in expressions | MagnitudeType enum + ActivationCondition model | §4.2 |
| BLOCKING 3: Redis required (no fallback) | redis_required flag + NDJSON archive | §5.2, §7 |
| Cond 4: Collision tunneling | Swept-sphere CollisionDetector | §3.1, §5.1 |
| Cond 5: Determinism tightening | PCG64, canonical JSON, SHA-256, cross-runner CI | §5.3 |
| Cond 6: Substeps per observation | simulation_substeps_per_observation config | §4.1, §5.1 |
| Cond 7: Noise toggle | NoiseConfig (disabled default) | §4.2, §7 |
| v2: Provenance metadata | python_version, numpy_version, host_id in summary | §4.8 |
| v2: Publish safety assertion | isinstance check before Redis write | §5.2 |
| v2: Replay idempotency test | Archive replay produces same results | §11.2 |
| v2: Backpressure jitter | Randomized sleep during pause | §5.2, §7 |

### Gemini (Google) — 2 Conditions

| Item | Resolution | Section |
|------|-----------|---------|
| COND 1: Continuous collision detection | Swept-sphere for collision scenarios | §3.1, §5.1, §6.2 |
| COND 2: Explicit state reset | reset() clears ALL accumulated state | §3.1 |

### Grok (xAI) — 6 Conditions + v2 Polish

| Item | Resolution | Section |
|------|-----------|---------|
| Cond 1: Redis ordering | Redis Streams (XADD/XREAD) | §5.2, §7 |
| Cond 2: Test field leakage | TestObservation split | §4.5 |
| Cond 3: Deterministic serialization | Canonical JSON + round(8dp) | §5.3 |
| Cond 4: Flow control | Backpressure + max_queue_length | §5.2, §7 |
| Cond 5: GroundTruth access | test_mode flag + logging | §3.1, §7 |
| Cond 6: Scenario 7 CONTRADICTS | Manipulation phase at tick 60 | §6.7 |
| v2: TestObservation isinstance guard | Safety gate before publish | §5.2 |

### Claude Opus 4.6 — 3 Observations (All Confirmed)

| Item | Resolution | Section |
|------|-----------|---------|
| Obs 1: ExpectedDiscovery in ScenarioConfig | Structural graph assertions for Acid Test | §4.3 |
| Obs 2: actual_confounder_activation_tick | Dynamic activation timing in summary | §4.8 |
| Obs 3: YAML schema_version | Version field + loader validation | §4.1 |

---

*End of CDD-09: Physics Simulator & Scenario Engine*

**Project ARIA · Adaptive Reasoning Integrated Architecture**  
Component Design Document 09 — The Training Environment  
v1.0 — Approved by GPT-5 (OpenAI), Gemini (Google), Grok (xAI), Claude Opus 4.6 (Anthropic)
