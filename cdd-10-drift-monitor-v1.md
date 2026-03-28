# CDD-10: DriftMonitor (Calibration Drift Detection)

> **Status:** APPROVED — All Reviewers Signed Off (4/4)
> **Version:** 1.0
> **Author:** Jun (Architect) + Claude Opus 4.6 (Design Partner)
> **Depends On:** CDD-06 v1.0 (Formal Semantics), CDD-08 v1.0 (Causal Complexity Manager)
> **Consumed By:** CDD-03 (BSM, advisory), CDD-04 (Predictive Loop), CDD-06 (Recalibration), CDD-07 (Adversary, Phase 2), CDD-11 (Observability)
> **Sprint:** S4 (Predictive Loop + Physics Simulator + Observability + DriftMonitor)
> **Classification:** Confidential — Core Team & Designated Review Partners
> **Reviewed By:** GPT-5 (OpenAI) · Gemini (Google) · Grok (xAI) · Claude Opus 4.6 (Anthropic)
> **Changes from v0.1:** 52 changes marked with ★

---

## Table of Contents

1. [Purpose & Scope](#1-purpose--scope)
2. [Specification References](#2-specification-references)
3. [Interface Contract](#3-interface-contract)
4. [Data Structures](#4-data-structures)
5. [Algorithm / Logic Flow](#5-algorithm--logic-flow)
6. [Integration Points](#6-integration-points)
7. [Configuration Parameters](#7-configuration-parameters)
8. [Error Handling & Edge Cases](#8-error-handling--edge-cases)
9. [Module Structure](#9-module-structure)
10. [Performance Targets](#10-performance-targets)
11. [Test Plan](#11-test-plan)
12. [Acceptance Criteria](#12-acceptance-criteria)
13. [Open Questions — Resolved](#13-open-questions--resolved)
14. [Review Feedback Incorporation Record](#14-review-feedback-incorporation-record)

---

## 1. Purpose & Scope

### 1.1 What This Component Does

The DriftMonitor is ARIA's continuous calibration watchdog. Its responsibility is to detect when the system's predictions are diverging from reality — not through individual prediction errors (which the Predictive Loop handles per-tick), but through **sustained statistical degradation** in prediction quality over time and across domains.

DriftMonitor operates as a background observer. It does not make predictions, does not modify the World Model directly, and does not initiate recalibration itself. Its job is to:

1. **Monitor** calibration metrics emitted by CDD-06 across sliding windows
2. **Detect** distribution shift: statistically significant changes in the statistical properties of incoming observations
3. **Classify** drift by type: gradual concept drift, sudden domain shift, systematic bias, or distribution shift
4. **Emit** structured drift alerts with enough diagnostic context for the Predictive Loop, BSM, and Observability layer to act on
5. **Respect** CDD-06's `cooldown_protected` state — a temporarily elevated Brier score during recalibration cooldown is expected noise, not fresh drift
6. ★ **Advise** downstream components via hint fields — BSM demotion hints, Curiosity Drive exploration hints, and Adversary trigger hints (Grok Cond 1, Gemini Cond 2)
7. ★ **Degrade gracefully** when infrastructure fails — local fallback mode on Redis outage, PH state persistence across restarts (Grok Cond 2, GPT-5 Fix E)

### 1.2 Neuroscience Grounding

| Component | Cognitive Analogue | Reference |
|-----------|-------------------|-----------|
| Distribution shift detection | Allostatic regulation / Homeostatic alarm | The brain maintains a predictive baseline and escalates when the environment's statistical properties shift too far from learned priors |
| Cooldown awareness | Refractory period after learning | Biological systems temporarily desensitize to error signals after active relearning to prevent oscillatory overcorrection |
| Domain-scoped monitoring | Cortical column specialization | Drift is domain-local — the visual cortex degrading doesn't imply auditory cortex is degraded |
| Page-Hinkley detection | Change-point detection in neural circuits | The brain maintains running statistics and triggers alert when deviation exceeds a statistical threshold |
| ★ Warmup suppression | Neural circuit maturation | New cortical circuits require a stabilization period before contributing to alarm signals (Q1 consensus) |
| ★ Bidirectional detection | Opponent-process monitoring | Neural systems detect both increases and decreases in signal — a sudden absence of expected input is as alarming as a spike (Q4 consensus) |

### 1.3 What This CDD Does NOT Cover

| Excluded | Covered In |
|----------|------------|
| Recalibration cycle execution | CDD-06 §6.5 |
| BSM node demotion logic | CDD-03 |
| Prediction error per-tick | CDD-04 |
| Dashboard visualization of drift events | CDD-11 |
| ★ Adversarial test generation (CDD-07 consumes `aria.drift.adversarial_trigger` in S6) | CDD-07 |
| World Model schema | CDD-01 |
| ★ Auto-recalibration decision logic (opt-in policy lives in CDD-06 config) | CDD-06 |

---

## 2. Specification References

| Spec Section | Content | Relevance |
|-------------|---------|-----------|
| v2.1 §5.6 | Calibration Collapse Detection | Primary mandate for DriftMonitor |
| v2.1 §5.6 | Drift detectors | Continuous monitoring across domain subgraphs |
| v2.1 §5.6 | Context change detection | Statistical property shift in incoming observations |
| v2.1 §5.6 | Dynamic recalibration | DriftMonitor triggers; CDD-06 executes |
| v2.1 §5.6 | Resilience metric | Graceful degradation on massive aleatoric uncertainty |
| CDD-06 §6.3 | Calibration windows | Metric sources: recent, medium, domain |
| CDD-06 §6.4 | Recalibration triggers | Conditions DriftMonitor monitors |
| CDD-06 §6.4.1 | Recalibration cooldown | Cooldown state DriftMonitor must respect |
| CDD-06 §10 | DriftMonitor integration contract | `accuracy_drop_trigger`, cooldown_protected tag |
| CDD-08 §6 | `compute_graph_statistics()` | Graph-level structural metrics consumed by DriftMonitor |
| CDD-04 §4.6 | TickResult | Prediction outcomes consumed in batch |
| CDD-04 §1.4 | Self-regulation mechanisms | DriftMonitor complements per-tick self-regulation with long-window drift |
| Impl Plan §1 | DriftMonitor as Phase 1 component | Added from GPT-5 condition |
| Impl Plan §6 S4 | Sprint allocation | DriftMonitor delivered in S4 alongside CDD-04, CDD-09 |

---

## 3. Interface Contract

### 3.1 Inputs

| Input | Source | Type | Description |
|-------|--------|------|-------------|
| `CalibrationMetrics` | CDD-06 via Redis pub/sub | `CalibrationMetrics` | Per-window Brier scores, ECE, prediction variance, domain scope |
| `RecalibrationResult` | CDD-06 via Redis pub/sub | `RecalibrationResult` | Recalibration completion events including cooldown state |
| `PredictionRecord` | CDD-04 via Redis pub/sub | `PredictionRecord` | Individual prediction outcomes (for running statistics) |
| `GraphComplexityStats` | CDD-08 via Redis pub/sub | `GraphComplexityStats` | Graph-level structural metrics |
| `ObservationEvent` | CDD-09 via Redis Streams | `ObservationEvent` | Raw observations (for distribution shift detection) |

### 3.2 Outputs

| Output | Consumer | Type | Trigger |
|--------|----------|------|---------|
| `DriftAlert` | CDD-06 (informational — CDD-06 decides whether to recalibrate), CDD-03 (advisory hints), CDD-11 (dashboard) | `DriftAlert` | Any detected drift condition |
| `DistributionShiftReport` | CDD-11 (dashboard), CDD-04 (awareness) | `DistributionShiftReport` | Significant shift in observation statistics |
| `DriftMonitorHeartbeat` | CDD-11 | `DriftMonitorHeartbeat` | Periodic health signal (every N cycles, even if no drift) |
| ★ `SystemWideDriftSummary` | CDD-11 | `SystemWideDriftSummary` | ≥ 3 domains exhibit same drift type within dedup window (Grok Cond 3) |

★ **Authority boundary (Q3 consensus — 3/3):** DriftAlert is informational. DriftMonitor does NOT trigger recalibration directly. CDD-06 subscribes to `aria.drift.alert` and applies its own decision logic (cooldown check, severity threshold, optional auto-recalibrate if enabled in CDD-06 config). "CDD-10 is the fire alarm; CDD-06 is the fire department." (Gemini)

### 3.3 Redis Channel Contracts

All outputs published to Redis pub/sub using namespaced channels:

```
aria.drift.alert                  → DriftAlert (consumers: CDD-06, CDD-03, CDD-11)
aria.drift.distribution_shift     → DistributionShiftReport (consumers: CDD-04, CDD-11)
aria.drift.heartbeat              → DriftMonitorHeartbeat (consumers: CDD-11)
★ aria.drift.system_wide          → SystemWideDriftSummary (consumers: CDD-11)
★ aria.drift.adversarial_trigger  → DriftAlert subset (consumer: CDD-07, Phase 2)
```

Subscriptions:
```
aria.calibration.metrics                    ← CalibrationMetrics from CDD-06
aria.calibration.recalibration_complete     ← RecalibrationResult from CDD-06
aria.calibration.cooldown_suppressed        ← Cooldown notification from CDD-06
aria.prediction.record                      ← PredictionRecord from CDD-04
aria.graph.complexity_stats                 ← GraphComplexityStats from CDD-08
aria.observation.*                          ← ObservationEvents from CDD-09
```

### 3.4 Error Cases

| Error | Condition | Behavior |
|-------|-----------|----------|
| `DriftMonitorMetricsUnavailable` | CDD-06 metrics not received within `metrics_timeout_cycles` | Log warning, continue. Do not emit false drift alerts. |
| `InsufficientObservationHistory` | Fewer than `min_observations_for_shift_detection` observations in domain | Skip distribution shift analysis for that domain. Log debug. |
| `CooldownStateUnknown` | RecalibrationResult not received, but domain is in expected cooldown window | Treat as cooldown_protected. Conservative — prevents false positives. |
| `StaleMetrics` | Metrics timestamp exceeds `metrics_staleness_threshold` | Emit `drift.monitor.stale_metrics` warning. Do not act on stale data. |
| ★ `WarmupInProgress` | Domain has not completed warmup period | Skip all drift analysis for that domain. Log debug. (Q1) |
| ★ `RedisUnavailable` | Redis subscription fails for > `redis_fallback_threshold_seconds` | Switch to local fallback mode. Continue monitoring via in-memory buffer. (Grok Cond 2) |

---

## 4. Data Structures

### 4.1 DriftType Enum

```python
class DriftType(str, Enum):
    """Classification of detected drift."""
    GRADUAL_CONCEPT_DRIFT = "gradual_concept_drift"
    # Slow degradation of Brier score over medium_window.
    # Cause: world model drifting from real-world distribution.

    SUDDEN_DOMAIN_SHIFT = "sudden_domain_shift"
    # Abrupt change in Brier score between consecutive windows.
    # Cause: domain conditions changed (e.g., new confounder class active).

    SYSTEMATIC_BIAS = "systematic_bias"
    # Confidence consistently overestimates or underestimates accuracy.
    # Detected via ECE exceeding threshold.

    DISTRIBUTION_SHIFT = "distribution_shift"
    # Statistical properties of incoming observations have changed.
    # Detected via Page-Hinkley on observation feature statistics.

    CALIBRATED_USELESSNESS = "calibrated_uselessness"
    # Brier score acceptable but prediction_variance too low.
    # System is predicting ~0.5 everywhere. Technically calibrated, practically useless.

    ALEATORIC_SATURATION = "aleatoric_saturation"
    # Domain exhibits massive inherent uncertainty; further improvement not productive.
    # System directed to deprioritize domain per v2.1 §5.6 resilience metric.
```

### 4.2 DriftSeverity Enum

```python
class DriftSeverity(str, Enum):
    """Severity classification for routing and urgency."""
    INFO = "info"         # Soft trend worth watching
    WARNING = "warning"   # Action recommended
    HIGH = "high"         # Active recalibration needed
    CRITICAL = "critical" # Predictive loop pause may be needed
```

### 4.3 DriftAlert

★ Changes from v0.1: Added `alert_confidence`, `bsm_demotion_hint`, `curiosity_trigger_hint`, `adversarial_trigger_hint`.

```python
class DriftAlert(BaseModel):
    """Structured alert for detected calibration drift.

    ★ This is an INFORMATIONAL broadcast. DriftMonitor does not
    trigger recalibration directly — CDD-06 decides whether to act.
    (Q3 consensus: 3/3 reviewers)
    """
    model_config = ConfigDict(extra="forbid")

    alert_id: str = Field(description="UUID for this alert.")
    timestamp: datetime
    domain_scope: str = Field(description="Affected domain, or 'global' for system-wide drift.")
    drift_type: DriftType
    severity: DriftSeverity

    brier_score_current: float
    brier_score_baseline: float = Field(
        description="Baseline Brier score this window is compared against."
    )
    ece_current: float | None = Field(
        default=None,
        description="Expected Calibration Error at detection time. Present for SYSTEMATIC_BIAS."
    )
    prediction_variance: float | None = Field(
        default=None,
        description="Present for CALIBRATED_USELESSNESS alerts."
    )

    cooldown_protected: bool = Field(
        description=(
            "If True, elevated Brier score is within an active recalibration cooldown "
            "window. DriftAlert is informational only — do not trigger recalibration."
        )
    )

    window_type: Literal["recent", "medium", "domain"] = Field(
        description="Which calibration window triggered this alert."
    )

    diagnostic_context: dict[str, Any] = Field(
        default_factory=dict,
        description=(
            "Additional structured context for dashboard and investigation. "
            "★ Includes CDD-08 GraphComplexityStats when available (Q6: informational only)."
        )
    )

    recommended_action: str = Field(
        description=(
            "Human-readable action recommendation. "
            "Not an imperative — CDD-06 decides whether to act."
        )
    )

    # ★ NEW — Multi-signal confidence (GPT-5 Rec 4, all reviewers)
    alert_confidence: float = Field(
        ge=0.0, le=1.0,
        description=(
            "Confidence this is a genuine drift signal (0–1). Computed from "
            "Brier delta magnitude + PH z-score + affected feature count. "
            "Allows CDD-06 to prioritize alerts."
        )
    )

    # ★ NEW — Advisory hints for downstream components (Grok Cond 1)
    bsm_demotion_hint: dict[str, Any] | None = Field(
        default=None,
        description=(
            "Advisory hint for CDD-03 BSM. Populated when severity >= HIGH. "
            "Contains domain_scope and recommended_demotion_scope."
        )
    )
    curiosity_trigger_hint: dict[str, Any] | None = Field(
        default=None,
        description=(
            "Advisory hint for Curiosity Drive. Populated on DISTRIBUTION_SHIFT. "
            "Contains shifted_feature and shift_magnitude."
        )
    )

    # ★ NEW — CDD-07 adversarial trigger interface (Gemini Cond 2, scoped as stub)
    adversarial_trigger_hint: dict[str, Any] | None = Field(
        default=None,
        description=(
            "Advisory hint for CDD-07 Adversary Simulator (Phase 2). "
            "Populated on SUDDEN_DOMAIN_SHIFT or DISTRIBUTION_SHIFT with "
            "severity >= HIGH. Contains shifted feature and magnitude for "
            "targeted counterfactual generation."
        )
    )
```

### 4.4 DistributionShiftReport

```python
class DistributionShiftReport(BaseModel):
    """Report of detected shift in observation statistics."""
    model_config = ConfigDict(extra="forbid")

    report_id: str
    timestamp: datetime
    domain_scope: str

    feature_name: str = Field(
        description="Which observation feature exhibited the shift."
    )
    shift_magnitude: float = Field(
        description="Page-Hinkley cumulative sum at detection."
    )
    baseline_mean: float
    current_mean: float
    detection_cycle: int = Field(
        description="Prediction cycle at which shift was confirmed."
    )

    # ★ UPDATED — direction field (Q4 bidirectional detection)
    shift_direction: Literal["upward", "downward"] = Field(
        description="Direction of the detected mean shift."
    )

    confidence: float = Field(
        ge=0.0, le=1.0,
        description="DriftMonitor's confidence this is a genuine shift vs noise."
    )
```

### 4.5 DriftMonitorHeartbeat

★ Changes from v0.1: Added cumulative exposure metrics (Grok Cond 5).

```python
class DriftMonitorHeartbeat(BaseModel):
    """Periodic health signal even when no drift detected."""
    model_config = ConfigDict(extra="forbid")

    timestamp: datetime
    cycles_monitored: int
    domains_active: list[str]
    alerts_emitted_since_last_heartbeat: int
    all_domains_healthy: bool

    # ★ NEW — Cumulative exposure metrics (Grok Cond 5)
    total_drift_exposure_cycles: int = Field(
        default=0,
        description="Cumulative monitoring cycles where at least one domain was in drift."
    )
    domains_in_drift: list[str] = Field(
        default_factory=list,
        description="Domains currently in active drift (non-cooldown, non-deduplicated)."
    )
    longest_active_drift_domain: str | None = Field(
        default=None,
        description="Domain with longest continuous drift streak. None if all healthy."
    )
    longest_active_drift_cycles: int = Field(
        default=0,
        description="Cycle count of longest active drift streak."
    )

    # ★ NEW — Infrastructure health (Grok Cond 2)
    local_mode: bool = Field(
        default=False,
        description="True if DriftMonitor is operating in Redis-fallback local mode."
    )
```

### 4.6 PageHinkleyState

★ Changes from v0.1: Added `direction` field (Q4), normalization context.

Internal state for the Page-Hinkley change-point detector, maintained per monitored feature per domain per direction.

```python
class PageHinkleyState(BaseModel):
    """Running state for one Page-Hinkley detector instance.

    ★ Two instances per feature: one upward, one downward (Q4 consensus).
    ★ Input is z-score normalized before accumulation (Gemini Cond 1).
    """
    model_config = ConfigDict(extra="forbid")

    feature_key: str = Field(description="Composite key: '{domain}:{feature_name}'")
    direction: Literal["upward", "downward"] = Field(                   # ★ NEW (Q4)
        description="Which direction this detector monitors."
    )
    cumulative_sum: float = Field(default=0.0)
    min_cumulative_sum: float = Field(default=0.0)
    observation_count: int = Field(default=0)
    running_mean: float = Field(default=0.0)
    running_variance: float = Field(default=0.0)
    last_reset_cycle: int = Field(default=0)
```

### 4.7 SystemWideDriftSummary

★ NEW — Global alert deduplication output (Grok Cond 3).

```python
class SystemWideDriftSummary(BaseModel):
    """Emitted when multiple domains exhibit the same drift type simultaneously.

    Replaces per-domain alert spam with a single summary when ≥ threshold
    domains drift within the dedup window.
    """
    model_config = ConfigDict(extra="forbid")

    summary_id: str
    timestamp: datetime
    drift_type: DriftType
    affected_domains: list[str]
    severity: DriftSeverity = Field(
        description="Max severity across affected domains."
    )
    domain_count: int
    recommended_action: str
```

### 4.8 DomainWarmupState

★ NEW — Warmup tracking per domain (Q1 consensus).

```python
class DomainWarmupState(BaseModel):
    """Tracks warmup progress for a single domain."""
    model_config = ConfigDict(extra="forbid")

    domain_scope: str
    warmup_cycles_remaining: int
    brier_samples: list[float] = Field(
        default_factory=list,
        description="Medium-window Brier samples collected during warmup."
    )
    warmup_complete: bool = Field(default=False)
    computed_baseline: float | None = Field(
        default=None,
        description="Median of brier_samples once warmup completes."
    )
```

---

## 5. Algorithm / Logic Flow

### 5.1 Main Monitoring Loop

★ Changes from v0.1: Added warmup check, Redis health check, local fallback, PH checkpointing, global dedup, Phase 2 stubs.

DriftMonitor runs as an AsyncIO task. It does not run on every prediction tick — it operates on a configurable cycle that processes batched incoming events.

```
On startup:
    Load recalibration_cooldown_cycles from FormalSemanticsConfig
        ★ FAIL-FAST: if mismatch with local config, raise ConfigMismatchError (GPT-5 editorial)
    Initialize PageHinkleyState registry (empty)
    ★ Attempt PH state recovery from Redis:
        For each key matching aria:drift:ph_state:*:
            If state age < ph_state_ttl_seconds: load into registry
            Else: discard stale state, cold-start with warmup (GPT-5 Fix E)
    Initialize domain cooldown state registry (empty)
    ★ Initialize domain warmup state registry (empty) (Q1)
    ★ Initialize drift exposure counters (Grok Cond 5)
    ★ Initialize Redis health monitor (Grok Cond 2)
    Subscribe to Redis channels (§3.3)
    Start heartbeat timer

Every monitoring_cycle_ticks (default: 10):
    ★ Check Redis health:
        IF Redis unavailable for > redis_fallback_threshold_seconds:
            Set local_mode = True
            Log [LOCAL_MODE] warning via structlog
            Process from in-memory circular buffer instead
        IF Redis reconnected after outage:
            Set local_mode = False
            Emit heartbeat with redis_reconnected=True

    Process queued CalibrationMetrics events
        ★ For each domain: check warmup state (Q1)
            IF warmup not complete: collect Brier sample, decrement counter, skip drift analysis
            IF warmup just completed: compute baseline as median(brier_samples)
        → call detect_calibration_drift() for warmed-up domains only

    Process queued PredictionRecord batch
        → call update_page_hinkley_statistics()

    Process queued ObservationEvent batch
        → call detect_distribution_shift()

    Process queued RecalibrationResult events
        → update domain cooldown registry

    ★ Run global alert deduplication (Grok Cond 3):
        IF ≥ system_wide_drift_domain_threshold domains emitted same drift_type
        within global_alert_dedup_window_seconds:
            Suppress individual alerts (log at DEBUG)
            Emit SystemWideDriftSummary to aria.drift.system_wide

    ★ Update drift exposure counters (Grok Cond 5)

    ★ Periodic PH state checkpoint (GPT-5 Fix E):
        IF cycles_since_last_checkpoint >= ph_checkpoint_interval_cycles:
            Persist all PH state to Redis with TTL
            Include ph_state_version for schema compatibility (GPT-5 v2 suggestion)

    Emit DriftMonitorHeartbeat (with ★ cumulative exposure fields)

    ★ Emit internal observability metrics via structlog (GPT-5 Rec 7):
        drift_alert_rate, ph_score_distribution, features_monitored_count,
        cooldown_suppressed_count

On graceful shutdown:
    ★ Flush all PH state to Redis (GPT-5 Fix E)
    Emit final heartbeat
    Log shutdown event

★ Phase 2 stub methods (Q7 — GPT-5 + Grok, 2:1):
    handle_hierarchical_metrics(region_id: str, local_brier: float) → None:
        raise NotImplementedError("Phase 2: cortical hierarchy metrics")

    handle_cortical_drift(layer_id: str, metrics: dict) → None:
        raise NotImplementedError("Phase 2: multi-layer drift detection")
```

### 5.2 detect_calibration_drift(metrics: CalibrationMetrics) → DriftAlert | None

★ Changes from v0.1: Added warmup gate, alert_confidence computation, advisory hint population.

```
Receive CalibrationMetrics from CDD-06.

★ IF domain not in warmup registry: initialize with warmup_cycles remaining
★ IF domain warmup not complete:
    Collect brier_samples.append(metrics.brier_scores["medium"])
    Decrement warmup_cycles_remaining
    IF warmup_cycles_remaining == 0:
        computed_baseline = median(brier_samples)
        domain_brier_baseline[domain_scope] = computed_baseline
        warmup_complete = True
    Return None.  ← No alerts during warmup

IF metrics.cooldown_protected is True:
    Log: calibration metrics in cooldown window, drift detection skipped.
    ★ Increment cooldown_suppressed_count
    Return None.  ← CRITICAL: never emit drift alert during cooldown

FOR EACH window IN [recent, medium, domain]:
    brier = metrics.brier_scores[window]

    CHECK: Gradual Concept Drift
        IF medium window brier > brier_drift_gradual_threshold (0.17):
            severity = WARNING if brier < 0.20 else HIGH
            alert = DriftAlert(drift_type=GRADUAL_CONCEPT_DRIFT, severity=severity)

    CHECK: Sudden Domain Shift
        delta = brier - domain_brier_baseline[domain_scope]
        IF delta > brier_sudden_shift_delta (0.05) in consecutive windows:
            alert = DriftAlert(drift_type=SUDDEN_DOMAIN_SHIFT, severity=HIGH)

    CHECK: Systematic Bias (ECE-based)
        IF metrics.ece > ece_systematic_bias_threshold (0.10):
            alert = DriftAlert(drift_type=SYSTEMATIC_BIAS, severity=WARNING)

    CHECK: Calibrated Uselessness (variance floor — from CDD-06 GPT-5 Cond 2)
        IF metrics.prediction_variance < prediction_variance_floor (0.01):
            alert = DriftAlert(drift_type=CALIBRATED_USELESSNESS, severity=WARNING)

    CHECK: Calibration Collapse (matches CDD-06 §6.4 CRITICAL threshold)
        IF medium_window brier > 0.30 OR accuracy < 0.50:
            alert = DriftAlert(drift_type=SUDDEN_DOMAIN_SHIFT, severity=CRITICAL,
                            recommended_action="Consider pausing predictive loop for domain")

    CHECK: Aleatoric Saturation
        IF domain brier has been > 0.25 for > aleatoric_saturation_window (50) cycles
        AND no downward trend detectable:
            alert = DriftAlert(drift_type=ALEATORIC_SATURATION, severity=INFO,
                            recommended_action="Deprioritize domain per v2.1 §5.6 resilience metric")

★ For any emitted alert:
    Compute alert_confidence (GPT-5 Rec 4):
        brier_delta_score = min(1.0, abs(brier - baseline) / 0.20)
        feature_count_score = min(1.0, affected_features / auto_top_k)
        alert.alert_confidence = 0.6 * brier_delta_score + 0.4 * feature_count_score

    Populate advisory hints (Grok Cond 1):
        IF alert.severity >= HIGH:
            alert.bsm_demotion_hint = {
                "domain_scope": domain_scope,
                "recommended_demotion_scope": "nodes updated in last 50 cycles"
            }
        IF alert.drift_type in (DISTRIBUTION_SHIFT, SUDDEN_DOMAIN_SHIFT):
            alert.curiosity_trigger_hint = {
                "domain_scope": domain_scope,
                "exploration_priority": "high"
            }
        IF alert.drift_type in (SUDDEN_DOMAIN_SHIFT, DISTRIBUTION_SHIFT)
        AND alert.severity >= HIGH:
            alert.adversarial_trigger_hint = {
                "domain_scope": domain_scope,
                "shifted_features": [list of affected feature names],
                "shift_magnitude": delta
            }
            ★ Publish to aria.drift.adversarial_trigger (Gemini Cond 2)

    Check alert deduplication:
        IF same (domain, drift_type) emitted within alert_dedup_window_cycles:
            Suppress. Emit drift.alert.suppressed to CDD-11. Return None.

    Include CDD-08 GraphComplexityStats in diagnostic_context if available (Q6)

Update domain_brier_baseline[domain_scope] with exponential moving average.
```

### 5.3 Page-Hinkley Distribution Shift Detection

★ Changes from v0.1: Normalized z-score input (Gemini Cond 1), bidirectional detection (Q4), feature selection (Q2).

The Page-Hinkley test is a sequential change-point detection algorithm suited to online, streaming data. ★ It detects when the mean of a distribution has shifted in either direction by more than a threshold λ, using normalized z-score input for scale invariance.

```
★ Feature Selection (Q2 — hybrid mode):
    During warmup:
        Track running variance for ALL numeric features per domain.
    On warmup completion:
        configured = configured_features.get(domain, [])
        IF feature_selection_mode == "configured":
            monitored = configured
        ELIF feature_selection_mode == "auto":
            monitored = top_k_by_variance(all_features, k=auto_top_k)
        ELIF feature_selection_mode == "hybrid":
            auto_selected = top_k_by_variance(all_features, k=auto_top_k)
            monitored = set(configured) | set(auto_selected)
        Apply feature_overrides (add/remove)
        ★ Instantiate TWO PageHinkleyState per monitored feature: upward + downward (Q4)

For each monitored observation feature f in domain d:

    ★ For each direction in [upward, downward]:
        ph = PageHinkleyState['{d}:{f}:{direction}']

        ★ Determine effective input (Q4):
            IF direction == "upward": x_eff = x
            IF direction == "downward": x_eff = -x

        Update Welford running mean and variance:
            ph.observation_count += 1
            delta = x_eff - ph.running_mean
            ph.running_mean += delta / ph.observation_count
            ph.running_variance += delta * (x_eff - ph.running_mean)

        IF ph.observation_count < min_observations_for_shift_detection:
            Skip.  ← insufficient history

        ★ Normalize input (Gemini Cond 1 — CRITICAL FIX):
            sample_variance = ph.running_variance / ph.observation_count
            z = (x_eff - ph.running_mean) / sqrt(sample_variance + page_hinkley_epsilon)

        Compute Page-Hinkley statistic on normalized z:
            ph.cumulative_sum += (z - page_hinkley_delta)
            ph.min_cumulative_sum = min(ph.min_cumulative_sum, ph.cumulative_sum)
            page_hinkley_score = ph.cumulative_sum - ph.min_cumulative_sum

        IF page_hinkley_score > page_hinkley_threshold (default: 50.0):
            Emit DistributionShiftReport(
                shift_direction=direction,
                ★ confidence=min(1.0, page_hinkley_score / (page_hinkley_threshold * 2))
            )
            Reset PageHinkleyState for this feature+direction (last_reset_cycle = current_cycle)
```

**Why Page-Hinkley:** It is parameter-efficient (O(1) memory per feature), requires no assumption on shift magnitude, and naturally handles gradual vs sudden shifts. Appropriate for Phase 1 with limited compute budget.

**★ Why z-score normalization (Gemini Cond 1):** Without normalization, `page_hinkley_threshold` is scale-dependent — a deviation of 5.0 is noise for a feature with σ=100 but a massive shift for σ=0.1. Normalizing makes the threshold meaningful across all features regardless of their natural variance.

### 5.4 Cooldown State Tracking

```
On receive RecalibrationResult(domain_scope=D, started_at=T):
    domain_cooldown_registry[D] = {
        cooldown_until_cycle: current_cycle + recalibration_cooldown_cycles,
        trigger_brier: result.brier_before
    }

is_in_cooldown(domain_scope: str, current_cycle: int) → bool:
    state = domain_cooldown_registry.get(domain_scope)
    IF state is None: return False
    return current_cycle < state.cooldown_until_cycle
```

### 5.5 Baseline Rolling Update

```
After each monitoring cycle per domain (warmup-complete domains only):
    domain_brier_baseline[domain] = (
        baseline_ema_alpha * metrics.brier_scores["medium"] +
        (1 - baseline_ema_alpha) * domain_brier_baseline[domain]
    )
```

This prevents the baseline from permanently anchoring to a historically-good Brier score in domains that have fundamentally changed (legitimate distribution shift).

★ **Baseline initialization (Q1):** During warmup, `domain_brier_baseline` is initialized to `median(warmup_brier_samples)` — robust to startup transients.

### 5.6 Redis Fallback Mode

★ NEW (Grok Cond 2).

```
Redis health monitor runs alongside main loop:

    Track last_redis_success_timestamp

    IF now - last_redis_success_timestamp > redis_fallback_threshold_seconds:
        local_mode = True
        Log: [LOCAL_MODE] Redis unavailable, switching to local analysis
        ★ Continue monitoring using in-memory circular buffer:
            local_metrics_buffer: deque(maxlen=local_buffer_size) per domain
            Collect metrics from local buffer instead of Redis subscription
        ★ Emit alerts via structlog with [LOCAL_MODE] tag
            CDD-11 can scrape logs for missed alerts on reconnect

    On Redis reconnection:
        local_mode = False
        Flush local buffer state into normal pipeline
        Emit heartbeat with local_mode=False, redis_reconnected=True
        Log: [LOCAL_MODE] Redis reconnected, resuming normal operation
```

---

## 6. Integration Points

★ Changes from v0.1: Added CDD-03 advisory, CDD-04/CDD-08 dual ownership for aleatoric saturation, CDD-07 Phase 2 trigger.

| Component | What DriftMonitor Uses | How |
|-----------|------------------------|-----|
| **CDD-06: Formal Semantics** | `CalibrationMetrics` (Brier scores, ECE, variance, cooldown_protected), `RecalibrationResult` | DriftMonitor subscribes to `aria.calibration.*`. The `cooldown_protected` flag is the critical gate — DriftMonitor will not emit alerts on protected metrics. ★ CDD-06 subscribes to `aria.drift.alert` as an informational input signal. CDD-06 decides whether to recalibrate (Q3). Auto-recalibrate opt-in policy lives in `FormalSemanticsConfig`, not `DriftMonitorConfig`. |
| **CDD-04: Predictive Loop** | `PredictionRecord` stream | DriftMonitor consumes prediction records in batch for Page-Hinkley running statistics. Does NOT block the loop — reads asynchronously. ★ CDD-04 subscribes to `aria.drift.alert`, filters for `ALEATORIC_SATURATION`, reduces domain scheduling frequency by 50% until Brier improves (Q5 dual ownership). |
| **CDD-08: Causal Complexity Manager** | `GraphComplexityStats` (growth rate, compression ratio, orphan count, abstraction lifecycle metrics) | Structural graph metrics provide context for drift alerts. ★ Informational only — does NOT influence severity scoring in Phase 1 (Q6 consensus: 3/3). ★ CDD-08 subscribes to `aria.drift.alert`, filters for `ALEATORIC_SATURATION`, reduces `SubgraphBudget` by 50% for affected domain (Q5 dual ownership). |
| **CDD-09: Physics Simulator** | `ObservationEvent` | Raw observation features are the input to Page-Hinkley distribution shift detection. |
| ★ **CDD-03: BSM** | Advisory consumer of `DriftAlert` | When `bsm_demotion_hint` is populated (severity ≥ HIGH), CDD-03 receives advisory signal for potential demotion evaluation. CDD-03 decides — hint is not imperative. (Grok Cond 1) |
| ★ **CDD-07: Adversary Simulator (Phase 2)** | Advisory consumer of `aria.drift.adversarial_trigger` | When `SUDDEN_DOMAIN_SHIFT` or `DISTRIBUTION_SHIFT` with severity ≥ HIGH, DriftMonitor publishes `adversarial_trigger_hint`. CDD-07 subscribes in S6 and generates targeted counterfactuals. Phase 1: channel declared but no consumer. (Gemini Cond 2) |
| **CDD-11: Observability** | `DriftAlert`, `DistributionShiftReport`, `DriftMonitorHeartbeat`, ★ `SystemWideDriftSummary` | All DriftMonitor outputs flow to CDD-11 for dashboard display, trend storage, and alert history. ★ CDD-11 uses `total_drift_exposure_cycles` and `longest_active_drift_cycles` to trigger human review after prolonged exposure (Grok Cond 5). |

★ **Phase 2 design notes:**
- CDD-08 graph stats may become a secondary severity factor if empirical Phase 1 data shows correlation with calibration degradation (Q6).
- Variance drift detection (Levene/rolling variance ratio) complements PH mean-shift detection (GPT-5 Rec 2).
- Ensemble detectors (KS test, population classifier performance) with signal fusion for higher confidence (GPT-5 Rec 3).
- Multi-domain scaling with sharding and async worker pool for > 20 domains (GPT-5 Rec 6).

---

## 7. Configuration Parameters

★ Changes from v0.1: Added warmup, feature selection, PH normalization, Redis fallback, global dedup, PH persistence, alert dedup fields.

```python
class DriftMonitorConfig(BaseModel):
    """Configuration for the DriftMonitor component.

    ★ recalibration_cooldown_cycles MUST be read from
    FormalSemanticsConfig at startup. Mismatch = fail-fast startup error.
    """
    model_config = ConfigDict(extra="forbid")

    # ── Cycle Control ──────────────────────────────────────────────
    monitoring_cycle_ticks: int = Field(
        default=10,
        ge=1,
        description=(
            "How many prediction ticks between each DriftMonitor evaluation cycle. "
            "DriftMonitor is a background observer — does not run every tick."
        ),
    )
    heartbeat_interval_cycles: int = Field(
        default=50,
        ge=1,
        description="Emit DriftMonitorHeartbeat every N monitoring cycles.",
    )

    # ── ★ Warmup (Q1 consensus: 3/3) ──────────────────────────────
    warmup_cycles: int = Field(
        default=30,
        ge=10,
        description=(
            "Number of monitoring cycles per domain before drift detection activates. "
            "During warmup, Brier samples are collected and baseline is computed as median. "
            "30 cycles × 10 ticks/cycle = 300 ticks. Compromise: GPT-5 (10) too aggressive, "
            "Gemini (100) too conservative."
        ),
    )
    baseline_seed: dict[str, float] | None = Field(
        default=None,
        description=(
            "Optional pre-configured Brier baseline per domain. If provided, "
            "domain uses this seed instead of waiting for warmup median. "
            "Format: {'physics': 0.12, 'spatial': 0.14}."
        ),
    )

    # ── Calibration Drift Thresholds ───────────────────────────────
    brier_drift_gradual_threshold: float = Field(
        default=0.17,
        description=(
            "Medium-window Brier score above this triggers GRADUAL_CONCEPT_DRIFT WARNING. "
            "Set between CDD-06's warning threshold (0.15) and recalibration trigger (0.20)."
        ),
    )
    brier_sudden_shift_delta: float = Field(
        default=0.05,
        description=(
            "Absolute Brier score delta between consecutive domain windows "
            "that triggers SUDDEN_DOMAIN_SHIFT detection."
        ),
    )
    ece_systematic_bias_threshold: float = Field(
        default=0.10,
        description=(
            "ECE (Expected Calibration Error) above this indicates systematic "
            "overconfidence or underconfidence — SYSTEMATIC_BIAS alert."
        ),
    )
    prediction_variance_floor: float = Field(
        default=0.01,
        description=(
            "Prediction variance below this triggers CALIBRATED_USELESSNESS. "
            "Matches CDD-06 FormalSemanticsConfig.prediction_variance_floor."
        ),
    )
    aleatoric_saturation_window: int = Field(
        default=50,
        ge=10,
        description=(
            "Number of consecutive monitoring cycles a domain's Brier score must "
            "remain above 0.25 with no downward trend before ALEATORIC_SATURATION is declared."
        ),
    )

    # ── ★ Feature Selection (Q2 — hybrid) ─────────────────────────
    feature_selection_mode: Literal["configured", "auto", "hybrid"] = Field(
        default="hybrid",
        description=(
            "How to select features for Page-Hinkley monitoring. "
            "'configured': use configured_features only. "
            "'auto': warmup auto-selects top-K by variance. "
            "'hybrid': configured features + auto top-K fills remaining slots."
        ),
    )
    configured_features: dict[str, list[str]] = Field(
        default_factory=dict,
        description=(
            "Explicit per-domain feature lists for PH monitoring. "
            "Format: {'physics': ['position_error', 'velocity_error']}."
        ),
    )
    auto_top_k: int = Field(
        default=5,
        ge=1, le=50,
        description=(
            "Number of features to auto-select by normalized variance during warmup. "
            "K=5 for Phase 1 (Gemini). Phase 2: increase to 10+ for richer domains."
        ),
    )
    feature_overrides: dict[str, list[str]] = Field(
        default_factory=dict,
        description="Operator override: add/remove features at runtime (GPT-5 suggestion).",
    )

    # ── Distribution Shift Detection (Page-Hinkley) ────────────────
    page_hinkley_delta: float = Field(
        default=0.005,
        description=(
            "Page-Hinkley sensitivity parameter. Smaller = more sensitive to gradual shifts. "
            "★ Operates on z-score normalized input — scale-invariant (Gemini Cond 1)."
        ),
    )
    page_hinkley_threshold: float = Field(
        default=50.0,
        description=(
            "Cumulative sum threshold triggering a distribution shift report. "
            "Higher = less sensitive to noise, more confidence in genuine shift. "
            "★ Operates on z-score normalized input — consistent meaning across features."
        ),
    )
    page_hinkley_epsilon: float = Field(
        default=1e-8,
        description="Epsilon for variance normalization to prevent division by zero (Gemini Cond 1).",
    )
    min_observations_for_shift_detection: int = Field(
        default=30,
        ge=10,
        description=(
            "Minimum per-feature observations in a domain before Page-Hinkley "
            "distribution shift detection is active."
        ),
    )

    # ── Baseline Tracking ──────────────────────────────────────────
    baseline_ema_alpha: float = Field(
        default=0.05,
        ge=0.01, le=0.50,
        description=(
            "Exponential moving average alpha for rolling Brier baseline update. "
            "Low alpha = slow baseline adaptation (conservative). "
            "High alpha = fast adaptation (may mask gradual drift)."
        ),
    )

    # ── ★ Alert Deduplication ──────────────────────────────────────
    alert_dedup_window_cycles: int = Field(
        default=30,
        ge=1,
        description=(
            "Same (domain, drift_type) alert suppressed within this many monitoring cycles. "
            "Default = monitoring_cycle_ticks × 3."
        ),
    )
    global_alert_dedup_window_seconds: int = Field(
        default=60,
        ge=10,
        description="Window for detecting system-wide drift across multiple domains (Grok Cond 3).",
    )
    system_wide_drift_domain_threshold: int = Field(
        default=3,
        ge=2,
        description="Minimum domains with same drift_type to trigger SystemWideDriftSummary.",
    )

    # ── Operational Safeguards ─────────────────────────────────────
    metrics_timeout_cycles: int = Field(
        default=20,
        description=(
            "If no CalibrationMetrics received from CDD-06 within this many "
            "monitoring cycles, emit stale_metrics warning."
        ),
    )
    metrics_staleness_threshold_seconds: float = Field(
        default=60.0,
        description="Metrics older than this are treated as stale.",
    )
    recalibration_cooldown_cycles: int = Field(
        default=5,
        description=(
            "Must match CDD-06 FormalSemanticsConfig.recalibration_cooldown_cycles. "
            "Single-sourced from FormalSemanticsConfig at startup — do not set independently. "
            "★ Startup fails fast if mismatch detected (GPT-5 editorial)."
        ),
    )

    # ── ★ Redis Fallback (Grok Cond 2) ────────────────────────────
    redis_fallback_threshold_seconds: int = Field(
        default=30,
        ge=5,
        description="Seconds of Redis unavailability before switching to local fallback mode.",
    )
    local_buffer_size: int = Field(
        default=50,
        ge=10,
        description="Per-domain circular buffer size for local fallback metrics.",
    )

    # ── ★ PH State Persistence (GPT-5 Fix E) ──────────────────────
    ph_checkpoint_interval_cycles: int = Field(
        default=100,
        ge=10,
        description="Monitoring cycles between periodic PH state checkpoints to Redis.",
    )
    ph_state_ttl_seconds: int = Field(
        default=3600,
        ge=300,
        description="TTL for persisted PH state in Redis. Stale state is discarded on startup.",
    )
    ph_state_version: int = Field(
        default=1,
        description=(
            "Schema version for persisted PH state. On startup, ignore state from "
            "prior versions to prevent silent corruption (GPT-5 v2 suggestion)."
        ),
    )
```

**Critical:** `recalibration_cooldown_cycles` MUST be read from `FormalSemanticsConfig` at startup and NOT independently configured. ★ Startup fails fast if mismatch — `ConfigMismatchError` raised with diagnostic message.

---

## 8. Error Handling & Edge Cases

★ Changes from v0.1: Added warmup, Redis outage, DriftMonitor restart, global drift edge cases.

| Edge Case | Behavior | Rationale |
|-----------|----------|-----------|
| `CalibrationMetrics` arrives with `cooldown_protected=True` | Skip all drift analysis. Log at DEBUG. Return None. | Brier elevation during cooldown is expected. False drift alert here would cause oscillatory recalibration. |
| ★ Domain in warmup period | Skip all drift analysis. Collect Brier sample. Return None. | Insufficient history for meaningful baseline (Q1). |
| Domain not seen before | ★ Initialize warmup state with `warmup_cycles` remaining. If `baseline_seed` configured for domain, use it and skip warmup. | No prior baseline to compare against. |
| All predictions in window are correct | Brier ≈ 0. No drift alerts. | Correct behavior — no action needed. |
| Rapid successive drift alerts on same domain | Deduplicate: if DriftAlert for same `(domain, drift_type)` emitted within `alert_dedup_window_cycles`, suppress duplicate. Emit `drift.alert.suppressed` event to CDD-11. | Prevents alert storm flooding CDD-11 dashboard and CDD-06 recalibration queue. |
| ★ Multiple domains emit same drift type simultaneously | If ≥ `system_wide_drift_domain_threshold` domains within `global_alert_dedup_window_seconds`: suppress per-domain alerts, emit `SystemWideDriftSummary`. Individual alerts logged at DEBUG. | Prevents alert storm on global system stress (Grok Cond 3). |
| CDD-06 metrics unavailable (startup lag) | Wait `metrics_timeout_cycles`. If still unavailable, emit `drift.monitor.stale_metrics`. Do not false-alarm. | CDD-06 may not have enough predictions yet. |
| Observation stream temporarily interrupted | Pause Page-Hinkley update for affected domain. Mark domain as `observation_gap`. Resume on stream restoration. | Gap in observations makes running mean unreliable. |
| Multiple domains in cooldown simultaneously | Track cooldown per-domain independently. Other domains not in cooldown proceed normally. | Per-domain isolation — consistent with CDD-06 design. |
| Aleatoric saturation declared | Emit INFO alert. Continue monitoring. Do NOT stop receiving data for the domain. | v2.1 §5.6 directs the system to deprioritize, not ignore — DriftMonitor observes; CDD-04 and CDD-08 decide deprioritization (Q5). |
| ★ Redis outage | Switch to `local_mode` after `redis_fallback_threshold_seconds`. Monitor via in-memory buffer. Emit alerts via structlog. Auto-resume on reconnection. | DriftMonitor must never go blind — drift is most likely during system stress (Grok Cond 2). |
| ★ DriftMonitor restart | Attempt PH state recovery from Redis. If state found and not stale (age < `ph_state_ttl_seconds`) and version matches (`ph_state_version`): resume from checkpoint. Otherwise: cold-start with warmup. | Detection continuity across restarts (GPT-5 Fix E). |
| ★ PH state version mismatch on recovery | Discard stale state, cold-start with warmup. Log warning. | Prevents silent corruption from prior schema versions (GPT-5 v2 suggestion). |

---

## 9. Module Structure

★ Changes from v0.1: Added warmup.py, fallback.py, dedup.py, persistence.py.

```
src/aria/drift_monitor/
├── __init__.py
├── monitor.py              # DriftMonitor main class, AsyncIO task, ★ Phase 2 stubs
├── calibration_watcher.py  # detect_calibration_drift(), cooldown tracking, ★ alert_confidence
├── page_hinkley.py         # PageHinkleyState, ★ normalized PH update, ★ bidirectional detection
├── baseline.py             # domain_brier_baseline rolling update, EMA, ★ warmup median
├── warmup.py               # ★ DomainWarmupState, warmup management, feature selection (Q1, Q2)
├── fallback.py             # ★ Redis health monitor, local_mode circular buffer (Grok Cond 2)
├── dedup.py                # ★ Alert deduplication, global SystemWideDriftSummary (Grok Cond 3)
├── persistence.py          # ★ PH state checkpoint/recovery to/from Redis (GPT-5 Fix E)
├── models.py               # DriftAlert, DistributionShiftReport, DriftMonitorHeartbeat,
│                           #   ★ SystemWideDriftSummary, ★ DomainWarmupState,
│                           #   DriftType, DriftSeverity
└── config.py               # DriftMonitorConfig (sources recalibration_cooldown_cycles from CDD-06)
```

---

## 10. Performance Targets

★ Changes from v0.1: Updated memory estimate for bidirectional PH, added local buffer cost.

| Metric | Target | Rationale |
|--------|--------|-----------|
| Monitoring cycle overhead | < 5ms per cycle | Background task; must not compete with CDD-04 loop budget |
| Page-Hinkley update | O(1) memory per feature per direction | Ring-buffer-free — PH state is constant size per feature |
| Redis subscription lag | < 50ms from CDD-06 emit to DriftMonitor receipt | Async pub/sub — no synchronous blocking |
| Alert deduplication | In-memory dict lookup, O(1) | No database required |
| ★ Max monitored features | 50 features × 20 domains × 2 directions = 2,000 PH state instances | Each PH state is ~100 bytes → ~200KB max memory (Q4) |
| ★ Local fallback buffer | 50 samples × 20 domains × ~200 bytes/sample = ~200KB | In-memory circular buffer for Redis outage (Grok Cond 2) |
| ★ PH checkpoint size | ~200KB serialized (2,000 PH states) | Redis persistence per checkpoint (GPT-5 Fix E) |
| ★ Feature selection warmup | < 2ms per domain | Variance computation is O(N) on feature count during warmup |

**★ Memory cost formula for PH (GPT-5 editorial):** `2 × num_features × num_domains × sizeof(PageHinkleyState)` where `sizeof(PageHinkleyState) ≈ 100 bytes`.

---

## 11. Test Plan

★ Changes from v0.1: 12 new tests added from reviewer feedback.

### 11.1 Unit Tests

| Test | Validates |
|------|-----------|
| `test_cooldown_protected_metrics_never_alert` | Metrics with `cooldown_protected=True` produce no DriftAlert under any Brier score |
| `test_gradual_drift_threshold` | Brier at 0.16 → no alert; 0.18 → WARNING; 0.21 → HIGH |
| `test_sudden_shift_delta` | Consecutive windows with Δ < 0.05 → no alert; Δ ≥ 0.05 → SUDDEN_DOMAIN_SHIFT |
| `test_ece_systematic_bias` | ECE = 0.09 → no alert; ECE = 0.11 → SYSTEMATIC_BIAS |
| `test_calibrated_uselessness` | Brier 0.12 (acceptable) but variance 0.005 → CALIBRATED_USELESSNESS |
| `test_page_hinkley_step_shift` | Inject step function in observation mean → PH detects after sufficient accumulation |
| `test_page_hinkley_no_false_alarm_on_noise` | Gaussian noise without mean shift → no distribution shift report |
| `test_alert_deduplication` | Same (domain, drift_type) emitted twice in window → second suppressed |
| `test_cooldown_sync_with_cdd06` | `recalibration_cooldown_cycles` loaded from `FormalSemanticsConfig`, not independently set |
| `test_aleatoric_saturation_window` | Brier > 0.25 for < 50 cycles → no alert; ≥ 50 cycles → ALEATORIC_SATURATION INFO |
| ★ `test_warmup_false_positive` | Wildly bad Brier scores during warmup → no DriftAlert emitted (GPT-5 test #1) |
| ★ `test_warmup_baseline_median` | After warmup, baseline = median of collected samples, not mean |
| ★ `test_synthetic_gradual_drift` | Slowly increase Brier over 100 cycles → GRADUAL_CONCEPT_DRIFT triggers at threshold (GPT-5 test #2) |
| ★ `test_page_hinkley_downward_shift` | Inject negative step function → PH downward detector fires (Q4, GPT-5 test #4) |
| ★ `test_page_hinkley_scale_invariance` | Same relative shift detected regardless of absolute feature scale (Gemini Cond 1) |
| ★ `test_feature_selection_stability` | Synthetic 20-feature dataset → warmup top-K picks expected high-variance features (GPT-5 test #5) |
| ★ `test_global_drift_deduplication` | 4 domains emit same drift_type in 60s → SystemWideDriftSummary emitted, per-domain suppressed (Grok Cond 3) |
| ★ `test_alert_confidence_computation` | alert_confidence computed from Brier delta + feature count, always in [0, 1] |
| ★ `test_bsm_hint_populated_on_high_severity` | severity=HIGH → bsm_demotion_hint populated; severity=WARNING → None |
| ★ `test_adversarial_hint_on_sudden_shift` | SUDDEN_DOMAIN_SHIFT + severity=HIGH → adversarial_trigger_hint populated |
| ★ `test_config_mismatch_fails_fast` | Mismatched `recalibration_cooldown_cycles` raises `ConfigMismatchError` at startup |

### 11.2 Integration Tests

| Test | Validates |
|------|-----------|
| `test_cdd06_to_driftmonitor_pipeline` | CDD-06 emits CalibrationMetrics → DriftMonitor receives via Redis, processes, emits DriftAlert → CDD-11 receives alert |
| `test_cooldown_round_trip` | CDD-06 triggers recalibration → emits `cooldown_protected=True` metrics → DriftMonitor correctly suppresses drift alert during cooldown window |
| `test_cdd09_observation_feed` | CDD-09 observations feed Page-Hinkley; injected distribution shift → DistributionShiftReport emitted |
| `test_cdd08_graph_stats_correlation` | Rapid orphan growth in GraphComplexityStats co-occurs with Brier degradation → diagnostic_context populated in DriftAlert |
| ★ `test_ph_state_persistence_and_recovery` | Run PH, persist state, restart monitor, continue stream → detection continuity, no missed shifts (GPT-5 test #6) |
| ★ `test_ph_state_version_mismatch` | Persist v1 state, change ph_state_version to 2, restart → old state discarded, cold start |
| ★ `test_redis_outage_fallback` | Kill Redis → DriftMonitor switches to local_mode within threshold → alerts via structlog → reconnect → resume normal (Grok Cond 2) |
| ★ `test_high_severity_alert_reaches_cdd06` | DriftAlert with severity=HIGH → CDD-06 receives on aria.drift.alert → CDD-06 decides independently |

### 11.3 Property-Based Tests (Hypothesis)

| Test | Validates |
|------|-----------|
| `test_no_alert_during_cooldown_for_any_brier` | For any Brier score in [0.0, 1.0], if `cooldown_protected=True`, no DriftAlert emitted |
| ★ `test_no_alert_during_warmup_for_any_brier` | For any Brier score in [0.0, 1.0], if warmup incomplete, no DriftAlert emitted (Q1) |
| `test_page_hinkley_reset_after_detection` | After PH threshold exceeded and report emitted, state resets to zero |
| `test_baseline_ema_bounded` | Regardless of Brier sequence, baseline EMA stays in [0.0, 1.0] |
| ★ `test_bidirectional_detection_symmetry` | For any shift magnitude m, upward detector on +m fires iff downward detector on -m fires (Q4) |

---

## 12. Acceptance Criteria

★ Changes from v0.1: Expanded from 17 → 28 criteria.

| # | Criterion | Sprint | Verification |
|---|-----------|--------|-------------|
| 1 | DriftMonitor starts as AsyncIO background task | S4 | Integration test |
| 2 | Subscribes to all 6 Redis channels defined in §3.3 | S4 | Startup integration test |
| 3 | `cooldown_protected=True` metrics never produce DriftAlert | S4 | Unit + property test |
| 4 | `recalibration_cooldown_cycles` loaded from `FormalSemanticsConfig` at startup | S4 | Unit test: config sync |
| 5 | GRADUAL_CONCEPT_DRIFT detected at threshold 0.17 | S4 | Unit test |
| 6 | SUDDEN_DOMAIN_SHIFT detected at Δ ≥ 0.05 | S4 | Unit test |
| 7 | SYSTEMATIC_BIAS detected via ECE ≥ 0.10 | S4 | Unit test |
| 8 | CALIBRATED_USELESSNESS detected when variance < 0.01 | S4 | Unit test |
| 9 | ALEATORIC_SATURATION emitted as INFO after sustained high Brier | S4 | Unit test |
| 10 | Page-Hinkley detects step-function distribution shift | S4 | Unit test |
| 11 | Page-Hinkley produces no false alarms on Gaussian noise | S4 | Property test |
| 12 | Alert deduplication suppresses repeat alerts within window | S4 | Unit test |
| 13 | DriftAlert published to `aria.drift.alert` on Redis | S4 | Integration test |
| 14 | DistributionShiftReport published to `aria.drift.distribution_shift` | S4 | Integration test |
| 15 | DriftMonitorHeartbeat emitted every 50 monitoring cycles | S4 | Integration test |
| 16 | Monitoring cycle overhead < 5ms | S4 | Performance benchmark |
| 17 | Full CDD-06 → DriftMonitor → CDD-11 pipeline integration passes | S4 | End-to-end integration test |
| ★ 18 | No drift alerts emitted during warmup period | S4 | Unit + property test |
| ★ 19 | Bidirectional PH detection (upward and downward shifts) | S4 | Unit test |
| ★ 20 | PH uses normalized z-score input (scale-invariant) | S4 | Unit test |
| ★ 21 | PH state persisted to Redis and recovered on restart | S4 | Integration test |
| ★ 22 | Local fallback mode activates on Redis outage (> 30s) | S4 | Integration test |
| ★ 23 | Global alert dedup suppresses per-domain spam when ≥ 3 domains drift simultaneously | S4 | Unit test |
| ★ 24 | `alert_confidence` field computed and populated on all DriftAlerts | S4 | Unit test |
| ★ 25 | `bsm_demotion_hint` populated when severity ≥ HIGH | S4 | Unit test |
| ★ 26 | `DriftMonitorHeartbeat` includes cumulative exposure metrics | S4 | Unit test |
| ★ 27 | Startup fails fast if `recalibration_cooldown_cycles` mismatches CDD-06 | S4 | Unit test |
| ★ 28 | Feature selection in hybrid mode: configured features + auto top-K | S4 | Unit + integration test |

---

## 13. Open Questions — Resolved

★ All 7 original questions resolved through the multi-LLM review process.

| Q | Resolution | Decided By |
|---|-----------|------------|
| Q1: Baseline initialization strategy | Warmup suppression (30 cycles) + optional seed baseline | 3/3 consensus (GPT-5 + Gemini + Grok all chose Option C) |
| Q2: Page-Hinkley feature selection | Hybrid: configurable whitelist + warmup auto top-K (K=5) | GPT-5 hybrid approach + Grok conservatism respected (2:1 over Gemini pure auto) |
| Q3: DriftAlert → recalibration authority | Recommendation only. CDD-06 decides. Auto-recalibrate opt-in lives in CDD-06 config. | 3/3 consensus |
| Q4: Bidirectional shift detection | Yes — two PH detectors per feature (upward + downward) | 3/3 consensus |
| Q5: Aleatoric saturation response | Dual ownership: CDD-04 deprioritizes scheduling, CDD-08 reduces SubgraphBudget | Compromise: Gemini (CDD-08) + Grok (CDD-04) → both |
| Q6: CDD-08 structural correlation | Informational only in Phase 1 (diagnostic_context). Phase 2: empirical evaluation. | 3/3 consensus |
| Q7: Phase 2 upgrade path | Include minimal stubs (`handle_hierarchical_metrics`, `handle_cortical_drift`) with `NotImplementedError` | 2:1 (GPT-5 + Grok vs Gemini "premature") |

---

## 14. Review Feedback Incorporation Record

### 14.1 GPT-5 (OpenAI) — 5 Required Fixes + 7 Recommendations

| # | Fix / Recommendation | Section | Status |
|---|---------------------|---------|--------|
| A | Baseline initialization & warmup strategy | §5.2, §7 | ★ RESOLVED (Q1) |
| B | Feature selection for Page-Hinkley | §5.3, §7 | ★ RESOLVED (Q2) |
| C | Two-sided detection (upward & downward) | §5.3, §4.6 | ★ RESOLVED (Q4) |
| D | DriftAlert → recalibration authority | §3.2, §6 | ★ RESOLVED (Q3) |
| E | Persist PH state and checkpointing | §5.1, §7, §9 | ★ RESOLVED |
| Rec 1 | Feature normalization (z-score) | §5.3 | ★ ADOPTED (via Gemini Cond 1) |
| Rec 2 | Variance drift detection (Levene) | §6 design note | DEFERRED Phase 2 |
| Rec 3 | Ensemble detectors (KS test) | §6 design note | DEFERRED Phase 2 |
| Rec 4 | Alert confidence scoring | §4.3, §5.2 | ★ ADOPTED |
| Rec 5 | Graph stats correlation | §6 | Matches Q6 |
| Rec 6 | Multi-domain scaling | §6 design note | DEFERRED Phase 2 |
| Rec 7 | Internal observability metrics | §5.1 | ★ ADOPTED |

### 14.2 GPT-5 v2 Final Review — Sanity Checks & Suggestions

| Item | Section | Status |
|------|---------|--------|
| CI preflight for cross-CDD config consistency | §7, §12 #27 | ★ ADOPTED (fail-fast startup) |
| Runtime telemetry in ops dashboard before live runs | §5.1, §10 | ★ ADOPTED (observability metrics) |
| Runbook / drift-playbook.md for operators | §9 deliverable note | ★ ADOPTED (S4 deliverable) |
| Load/canary test under stressed conditions | §11.2 | ★ ADOPTED (integration test coverage) |
| `ph_state_version` for schema compatibility | §7 | ★ ADOPTED |
| Canonical JSON hashing utility | Implementation note | NOTED for implementation |

### 14.3 Gemini (Google) — 2 Blocking Conditions

| # | Condition | Section | Status |
|---|-----------|---------|--------|
| 1 | Page-Hinkley variance normalization (z-score) | §5.3 | ★ RESOLVED — algorithm rewritten with normalized input |
| 2 | Missing CDD-07 adversarial trigger loop | §3.2, §4.3, §6 | ★ RESOLVED — stub interface with `adversarial_trigger_hint` + Phase 2 channel |

### 14.4 Grok (xAI) — 6 Conditions

| # | Condition | Section | Status |
|---|-----------|---------|--------|
| 1 | BSM & Curiosity escalation path | §4.3, §5.2, §6 | ★ RESOLVED — hint fields in DriftAlert |
| 2 | Local fallback on Redis outage | §5.6, §7, §8, §9 | ★ RESOLVED — local_mode with circular buffer |
| 3 | Global alert deduplication | §4.7, §5.1, §7 | ★ RESOLVED — SystemWideDriftSummary |
| 4 | Resilience metric enforcement hook | §6 (Q5 resolution) | ★ RESOLVED — CDD-04 + CDD-08 dual ownership |
| 5 | Heartbeat cumulative exposure | §4.5, §5.1 | ★ RESOLVED — exposure fields in heartbeat |
| 6 | PH bidirectional detection | §4.6, §5.3 (Q4) | ★ RESOLVED |

### 14.5 Claude Opus 4.6 — Design Partner Observations

| # | Observation | Section |
|---|------------|---------|
| 1 | Warmup cycle compromise (30) balances GPT-5 (10) vs Gemini (100) | §7 |
| 2 | Feature selection hybrid preserves Grok's conservatism while adding GPT-5's flexibility | §7 |
| 3 | Aleatoric saturation dual ownership resolves Gemini vs Grok split cleanly | §6 |
| 4 | CDD-07 trigger stub avoids hard dependency on S6 component | §3.2, §6 |
| 5 | GPT-5 v2 `ph_state_version` prevents silent state corruption on schema evolution | §7 |

---

## 15. Deliverables

| Deliverable | Sprint | Notes |
|-------------|--------|-------|
| `src/aria/drift_monitor/` (all modules per §9) | S4 | Core implementation |
| Unit tests (`tests/unit/test_drift_monitor/`) | S4 | 21 unit tests per §11.1 |
| Integration tests (`tests/integration/test_drift_monitor/`) | S4 | 8 integration tests per §11.2 |
| Property tests (`tests/property/test_drift_monitor/`) | S4 | 5 property tests per §11.3 |
| Performance benchmark | S4 | Validate < 5ms cycle overhead |
| ★ `docs/drift-playbook.md` | S4 | Operator runbook: common alert patterns, recommended responses, auto-recalibrate opt-in procedure (GPT-5 v2) |

---

*End of CDD-10 v1.0: DriftMonitor (Calibration Drift Detection)*

**Project ARIA · Adaptive Reasoning Integrated Architecture**
Component Design Document 10 — Calibration Drift Detection
Approved by All Reviewing Systems — Ready for Implementation

★ **Total tracked changes from v0.1: 52**
