# TECH — Operation Passports & Compute Budget

> **Core principle: "Every background action must declare its cost, risk, reversibility, and completeness value before it runs."**
>
> If a job has no passport, it does not run. No exceptions.

---

## The Problem This Document Solves

Curator's background processing engine (R13) uses a priority queue with thermal awareness (R11). But priority and thermal state alone are not enough to decide whether a job should run right now. A job can be:

- High priority but extremely expensive (OCR of a 300-page PDF)
- Low cost but irreversible (moving a file to Staging)
- Medium cost but only safe on AC power (embedding generation for a large batch)
- Cheap but not allowed during a Zoom call (disk I/O that could cause stutter)

Without a formal declaration of these properties, the system cannot make safe, informed scheduling decisions. It can only check thermal state and priority — which is not enough.

The **Operation Passport** is the solution. Every job in Curator's queue must carry a passport that declares, upfront, what it will do, what it will cost, when it is allowed to run, and what completeness value it delivers. The **PassportGate** checks the passport against current system state before every job begins.

---

## 1. Prior Art

### 1.1 Apple BGProcessingTask — Declared Preconditions

Apple's `BGProcessingTask` framework (iOS/macOS BackgroundTasks framework) requires apps to declare task requirements before the OS schedules them:

```swift
let request = BGProcessingTaskRequest(identifier: "com.curator.deep-extract")
request.requiresExternalPower = true     // only run when plugged in
request.requiresNetworkConnectivity = false  // local-only operation
request.earliestBeginDate = Date(timeIntervalSinceNow: 60)
BGTaskScheduler.shared.submit(request)
```

The OS uses these declared properties to decide when to run the task — not the task itself. [Source: Apple Developer Documentation — BGProcessingTask]

On macOS specifically, XPC activities use `XPC_ACTIVITY_ALLOW_BATTERY` (boolean flag, default `false` for maintenance-priority activities) to declare whether the task is permitted to run on battery. [Source: Apple Developer Forums — BGProcessingTask macOS behavior]

**Lesson for Curator:** The caller declares requirements. The scheduler enforces them. The task itself does not check its own prerequisites.

### 1.2 Kubernetes Resource Requests & Limits — Declared Cost

Kubernetes requires every pod to declare `requests` (minimum resources needed, used for scheduling) and `limits` (maximum resources allowed, enforced at runtime) before it can be scheduled:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "2000m"
    memory: "1Gi"
```

The scheduler only places pods on nodes with sufficient available capacity matching `requests`. Pods that do not declare resources cannot be placed on resource-constrained clusters. [Source: Kubernetes documentation — Resource requests and limits]

**Admission webhooks** validate and mutate job specs before any pod runs. A pod whose declared resources exceed cluster policy is rejected before execution. [Source: Kubernetes — Admission Controllers]

**Lesson for Curator:** Resources must be declared, not discovered. A job that discovers it needs more CPU than expected mid-run is a design failure. The passport is the declaration; the PassportGate is the admission webhook.

### 1.3 Circuit Breaker — Pre-flight Failure

The circuit breaker pattern (Hystrix, Resilience4j) checks system health before attempting an operation. If the circuit is open (system is unhealthy), the operation fails fast without execution. [Source: Circuit Breaker Pattern — Wikipedia]

**Lesson for Curator:** The PassportGate is Curator's circuit breaker. If current system state (thermal, battery, active app load) does not satisfy the passport's requirements, the gate rejects the job immediately — it does not attempt partial execution.

### 1.4 macOS WWDC 2025 — BGContinuedProcessingTask

WWDC 2025 introduced `BGContinuedProcessingTask` (iOS 26 / macOS 26), a new API for extended background processing that allows tasks to declare longer runtime requirements with explicit energy and resource constraints. [Source: WWDC 2025 — Finish tasks in the background]

This confirms Apple's direction: the trend is toward more explicit, pre-declared background task constraints, not less.

---

## 2. Operation Passport — Data Structure

Every job in Curator's `processing_queue` must have an associated passport. The passport is stored as a JSON blob in the queue entry, or as a separate `operation_passports` table.

```python
from dataclasses import dataclass, field
from enum import Enum

class CostTier(Enum):
    TRIVIAL   = "trivial"   # < 10ms CPU, < 1MB RAM
    CHEAP     = "cheap"     # 10ms–1s CPU, < 50MB RAM
    MODERATE  = "moderate"  # 1s–30s CPU, < 200MB RAM
    EXPENSIVE = "expensive" # > 30s CPU or > 200MB RAM

class ThermalFloor(Enum):
    ANY       = "any"       # runs at any thermal state
    ATTENTIVE = "attentive" # requires thermalState ≤ fair
    CALM      = "calm"      # requires thermalState == nominal
    IDLE      = "idle"      # requires idle detection + nominal

@dataclass
class OperationPassport:
    # Identity
    operation_type: str          # e.g. "tier0_scan", "ocr_pdf", "embed_file", "stage_move"

    # Cost declaration
    cost_tier: CostTier
    estimated_cpu_seconds: float  # rough upper bound (not a SLA)
    estimated_ram_mb: int         # rough upper bound
    estimated_io_mb: int          # disk bytes read/written

    # Safety constraints
    thermal_floor: ThermalFloor
    battery_allowed: bool         # False = requires AC power
    touches_user_files: bool      # True = reads/writes files outside Curator's own DB
    is_reversible: bool           # True = can be undone without data loss

    # Scheduling hints
    requires_idle: bool           # True = only during user idle detection
    interruptible: bool           # True = can be paused mid-operation safely
    max_duration_seconds: int     # watchdog timeout (R10)

    # Completeness value
    completeness_gain: str        # e.g. "Layer1", "Layer2", "Layer3", "none"
    file_id: int | None           # which file this advances (None = system operation)

    # Validation (auto-computed)
    def is_valid(self) -> bool:
        """Basic sanity check — passport must be internally consistent."""
        if self.touches_user_files and self.is_reversible:
            # Moves/renames are touches but reversible via WAL
            pass
        if self.cost_tier == CostTier.EXPENSIVE and self.battery_allowed:
            # Expensive + battery: requires explicit justification
            raise ValueError(
                f"Expensive operation '{self.operation_type}' marked battery_allowed=True. "
                "Expensive operations should not run on battery without explicit override."
            )
        return True
```

---

## 3. Standard Passports — Reference Definitions

These are the canonical passports for Curator's known operation types. Any new operation type added to the codebase must add its passport here before it can be queued.

### 3.1 Tier 0 — File Identity Scan

```python
OperationPassport(
    operation_type    = "tier0_scan",
    cost_tier         = CostTier.TRIVIAL,
    estimated_cpu_s   = 0.01,
    estimated_ram_mb  = 1,
    estimated_io_mb   = 0,
    thermal_floor     = ThermalFloor.ANY,
    battery_allowed   = True,
    touches_user_files= False,   # stat() only, no content access
    is_reversible     = True,
    requires_idle     = False,
    interruptible     = True,
    max_duration_s    = 5,
    completeness_gain = "Layer1",
)
```

### 3.2 Tier 1 — Text Extraction (PDF, DOCX, code)

```python
OperationPassport(
    operation_type    = "tier1_extract_text",
    cost_tier         = CostTier.CHEAP,
    estimated_cpu_s   = 2.0,
    estimated_ram_mb  = 50,
    estimated_io_mb   = 5,
    thermal_floor     = ThermalFloor.ATTENTIVE,
    battery_allowed   = True,
    touches_user_files= False,   # reads content, writes to DB only
    is_reversible     = True,
    requires_idle     = False,
    interruptible     = True,
    max_duration_s    = 30,
    completeness_gain = "Layer2",
)
```

### 3.3 OCR — Single Page

```python
OperationPassport(
    operation_type    = "ocr_single_page",
    cost_tier         = CostTier.MODERATE,
    estimated_cpu_s   = 5.0,
    estimated_ram_mb  = 100,
    estimated_io_mb   = 2,
    thermal_floor     = ThermalFloor.CALM,
    battery_allowed   = False,    # OCR is expensive on battery
    touches_user_files= False,
    is_reversible     = True,
    requires_idle     = False,
    interruptible     = True,     # can stop between pages
    max_duration_s    = 60,
    completeness_gain = "Layer2",
)
```

### 3.4 OCR — Full Document (> 50 pages)

```python
OperationPassport(
    operation_type    = "ocr_full_document",
    cost_tier         = CostTier.EXPENSIVE,
    estimated_cpu_s   = 120.0,   # 200-page PDF ≈ 2 minutes
    estimated_ram_mb  = 200,
    estimated_io_mb   = 20,
    thermal_floor     = ThermalFloor.IDLE,   # only during idle + nominal
    battery_allowed   = False,
    touches_user_files= False,
    is_reversible     = True,
    requires_idle     = True,
    interruptible     = True,    # stop between pages, resume later
    max_duration_s    = 600,     # 10-minute watchdog
    completeness_gain = "Layer2",
)
```

### 3.5 Embedding Generation — Single File

```python
OperationPassport(
    operation_type    = "embed_file",
    cost_tier         = CostTier.MODERATE,
    estimated_cpu_s   = 3.0,
    estimated_ram_mb  = 150,     # model loaded in memory
    estimated_io_mb   = 0,
    thermal_floor     = ThermalFloor.CALM,
    battery_allowed   = False,
    touches_user_files= False,
    is_reversible     = True,
    requires_idle     = False,
    interruptible     = False,   # do not interrupt mid-embedding
    max_duration_s    = 30,
    completeness_gain = "Layer3",
)
```

### 3.6 Embedding Generation — Batch (Idle Completion Mode)

```python
OperationPassport(
    operation_type    = "embed_batch_idle",
    cost_tier         = CostTier.EXPENSIVE,
    estimated_cpu_s   = 300.0,  # batch of 100 files
    estimated_ram_mb  = 200,
    estimated_io_mb   = 0,
    thermal_floor     = ThermalFloor.IDLE,
    battery_allowed   = False,
    touches_user_files= False,
    is_reversible     = True,
    requires_idle     = True,
    interruptible     = True,   # can pause after each file
    max_duration_s    = 1800,   # 30-minute watchdog
    completeness_gain = "Layer3",
)
```

### 3.7 Staging Move (WAL-protected rename)

```python
OperationPassport(
    operation_type    = "staging_move",
    cost_tier         = CostTier.CHEAP,
    estimated_cpu_s   = 0.1,
    estimated_ram_mb  = 2,
    estimated_io_mb   = 0,       # rename is metadata only
    thermal_floor     = ThermalFloor.ANY,
    battery_allowed   = True,
    touches_user_files= True,    # renames user file
    is_reversible     = True,    # WAL intent written before rename
    requires_idle     = False,
    interruptible     = False,   # atomic: do not interrupt mid-rename
    max_duration_s    = 10,
    completeness_gain = "none",  # completeness unchanged by move
)
```

### 3.8 Candidate Graph Update

```python
OperationPassport(
    operation_type    = "candidate_graph_update",
    cost_tier         = CostTier.CHEAP,
    estimated_cpu_s   = 0.5,
    estimated_ram_mb  = 20,
    estimated_io_mb   = 1,
    thermal_floor     = ThermalFloor.ATTENTIVE,
    battery_allowed   = True,
    touches_user_files= False,
    is_reversible     = True,
    requires_idle     = False,
    interruptible     = True,
    max_duration_s    = 15,
    completeness_gain = "none",  # internal graph maintenance
)
```

### 3.9 Vector Index Rebuild (per-context shard)

```python
OperationPassport(
    operation_type    = "vector_index_rebuild_shard",
    cost_tier         = CostTier.EXPENSIVE,
    estimated_cpu_s   = 60.0,
    estimated_ram_mb  = 300,
    estimated_io_mb   = 10,
    thermal_floor     = ThermalFloor.IDLE,
    battery_allowed   = False,
    touches_user_files= False,
    is_reversible     = True,    # old index retained until new one validated
    requires_idle     = True,
    interruptible     = False,   # index rebuild must complete atomically
    max_duration_s    = 300,
    completeness_gain = "none",
)
```

---

## 4. The PassportGate

The **PassportGate** is the function called before every job dequeues from `processing_queue`. It checks the job's passport against current system state and returns either `ALLOW`, `DEFER`, or `REJECT`.

```python
class GateDecision(Enum):
    ALLOW  = "allow"   # run now
    DEFER  = "defer"   # re-queue, check again later
    REJECT = "reject"  # permanent failure, surface to user

def passport_gate(
    passport: OperationPassport,
    thermal: ThermalGovernor,
    system: SystemSnapshot,
) -> GateDecision:

    # 1. Thermal floor check
    if not thermal_satisfied(passport.thermal_floor, thermal.current_state()):
        return GateDecision.DEFER

    # 2. Battery check
    if not passport.battery_allowed and system.on_battery:
        return GateDecision.DEFER

    # 3. Idle check
    if passport.requires_idle and not system.user_is_idle:
        return GateDecision.DEFER

    # 4. Available RAM check (soft — don't reject, defer if tight)
    if passport.estimated_ram_mb > system.available_ram_mb * 0.8:
        return GateDecision.DEFER

    # 5. File touches during active use check
    if passport.touches_user_files and system.user_is_active:
        return GateDecision.DEFER

    # 6. Irreversible operations require explicit WAL intent
    #    (enforced at operation level, not gate level — gate trusts the passport)

    return GateDecision.ALLOW
```

```python
@dataclass
class SystemSnapshot:
    thermal_state: ThermalState
    on_battery: bool
    battery_pct: int
    user_is_idle: bool          # no keyboard/mouse for > threshold
    user_is_active: bool        # keyboard/mouse in last 30 seconds
    available_ram_mb: int
    active_app_load: float      # 0.0–1.0 (Zoom/Xcode = 0.8+)
```

The `SystemSnapshot` is refreshed every 30 seconds by the Thermal Governor (R11 §3.3). The PassportGate reads from the cached snapshot — it does not poll the OS on every job.

---

## 5. Gate Decision Handling

| Decision | Action |
|---|---|
| `ALLOW` | Job dequeues and runs. Watchdog timer set to `max_duration_seconds`. |
| `DEFER` | Job stays in queue with `status = 'pending'`. `next_attempt_at` set to `now + defer_interval`. No retry count incremented (deferral is not failure). |
| `REJECT` | Only for passports that fail internal validation (e.g., `is_valid()` raises). Logged as a system error. Never surfaced to user as a file problem. |

**Defer intervals by reason:**

| Reason | Defer interval |
|---|---|
| Thermal too high | 60 seconds (re-check after governor re-polls) |
| On battery | 300 seconds (re-check in 5 minutes) |
| User not idle (requires_idle) | 120 seconds |
| RAM too tight | 30 seconds |
| User active (file touches) | 60 seconds |

Deferred jobs do not count toward retry limits. Only `status = 'failed'` transitions count.

---

## 6. Passport Validation at Queue Time

When a job is enqueued in `processing_queue`, the passport is validated immediately:

```python
def enqueue_job(file_id: int, operation_type: str) -> int:
    passport = PASSPORT_REGISTRY[operation_type]
    passport.is_valid()  # raises if internally inconsistent
    # Insert into processing_queue with passport JSON
    ...
```

If the operation type is not in `PASSPORT_REGISTRY`, the enqueue call raises `UnknownOperationType`. No unregistered operation can be queued.

The `PASSPORT_REGISTRY` is a dict of `operation_type → OperationPassport`. It is the canonical list of all operations Curator performs. Adding a new operation requires adding its passport to the registry — there is no other path.

---

## 7. Completeness Gain Integration (R13)

The `completeness_gain` field on every passport integrates with the Completeness Ledger (R13 §1.2). When a job completes successfully:

```python
def on_job_complete(file_id: int, passport: OperationPassport):
    if passport.completeness_gain == "Layer2":
        db.execute("UPDATE files SET tier_reached = MAX(tier_reached, 1) WHERE file_id = ?", file_id)
    elif passport.completeness_gain == "Layer3":
        db.execute("UPDATE files SET tier_reached = MAX(tier_reached, 2) WHERE file_id = ?", file_id)
    # Completeness Ledger query auto-reflects the update
```

The Completeness Ledger never needs to be manually updated. It is always a live view of `tier_reached` values across the `files` table.

---

## 8. Watchdog Integration (R10)

Every job runs with a watchdog timer set to `passport.max_duration_seconds`. The watchdog is set when the job dequeues (`ALLOW` decision) and cancelled when the job completes or fails.

```python
def run_job(job: QueueEntry, passport: OperationPassport):
    watchdog = Watchdog(timeout=passport.max_duration_seconds, job_id=job.queue_id)
    watchdog.start()
    try:
        executor = EXECUTOR_REGISTRY[passport.operation_type]
        executor.run(job.file_id)
        watchdog.cancel()
        on_job_complete(job.file_id, passport)
    except WatchdogFired:
        on_job_timeout(job, passport)  # → R10 failure path
    except Exception as e:
        on_job_error(job, passport, e)  # → R10 failure path
```

The watchdog fires at `max_duration_seconds` regardless of thermal state. A thermally paused job should have been deferred before it ran — if it is running and the thermal state changes, the job continues until completion or watchdog. The Thermal Governor does not interrupt mid-execution; it defers the next job.

---

## 9. Passport Registry — Complete List

All currently defined operation types in Curator:

| operation_type | Cost | Thermal floor | Battery | Touches files | Reversible | Completeness |
|---|---|---|---|---|---|---|
| `tier0_scan` | Trivial | Any | Yes | No | Yes | Layer1 |
| `tier1_extract_text` | Cheap | Attentive | Yes | No | Yes | Layer2 |
| `tier1_extract_headers` | Cheap | Attentive | Yes | No | Yes | Layer2 |
| `ocr_single_page` | Moderate | Calm | No | No | Yes | Layer2 |
| `ocr_full_document` | Expensive | Idle | No | No | Yes | Layer2 |
| `embed_file` | Moderate | Calm | No | No | Yes | Layer3 |
| `embed_batch_idle` | Expensive | Idle | No | No | Yes | Layer3 |
| `staging_move` | Cheap | Any | Yes | Yes | Yes (WAL) | none |
| `candidate_graph_update` | Cheap | Attentive | Yes | No | Yes | none |
| `vector_index_rebuild_shard` | Expensive | Idle | No | No | Yes | none |
| `fts_index_sync` | Trivial | Any | Yes | No | Yes | none |
| `schema_migration` | Cheap | Any | Yes | No | Yes* | none |
| `wal_recovery` | Trivial | Any | Yes | Yes | Yes | none |

`*` Schema migration is reversible only to the previous version via migration table. Destructive schema changes are always gated with an explicit user confirmation step.

---

## 10. Design Constraints

**Every new operation type MUST:**
1. Add a passport to `PASSPORT_REGISTRY` before any code calls `enqueue_job`
2. Set `touches_user_files = True` if it reads or writes any file outside Curator's own data directory
3. Set `is_reversible = False` only if the operation cannot be undone without user data loss — this requires a mandatory code review comment explaining why
4. Set `battery_allowed = False` for any `CostTier.EXPENSIVE` operation (enforced by `passport.is_valid()`)
5. Set `max_duration_seconds` to a realistic upper bound — not infinity, not zero

**The PassportGate MUST NOT:**
- Be bypassed for any reason
- Be called with a `None` passport
- Be skipped for "quick" or "trivial" operations (trivial passports pass the gate in microseconds — the gate overhead is negligible)

**The system MUST NOT:**
- Run any operation without a prior `ALLOW` gate decision
- Increment retry count on `DEFER` decisions
- Surface passport validation failures to the user as file errors

---

## 11. Relationship to Other Documents

| Document | Relationship |
|---|---|
| R11_calm_vector_memory_thermal_indexing.md | `ThermalGovernor.current_state()` and `SystemSnapshot` are inputs to PassportGate |
| R13_progressive_completeness_thermal_safe_scan.md | `completeness_gain` field feeds the Completeness Ledger; `processing_queue` carries the passport |
| R10_failure_intelligence_self_healing.md | Watchdog timeout → R10 failure path. Passport `max_duration_seconds` sets the watchdog. |
| TECH_engineering_foundation.md | `processing_queue` schema extended with `passport_json` column |

---

## 12. Open Questions

| # | Question | Status |
|---|---|---|
| Q1 | Should passport be stored as JSON in `processing_queue.passport_json` or in a separate `operation_passports` table? | JSON column is simpler; separate table enables indexed queries on passport fields |
| Q2 | Should `estimated_cpu_seconds` be calibrated from real measurements? | Yes — initial values are guesses; the system should record actual duration and update estimates over time |
| Q3 | Should the PassportGate log every `DEFER` decision? | Yes for debugging; no for production (high volume). Use a configurable debug flag. |
| Q4 | Should expensive operations have a user-visible "scheduled for tonight" notification? | Possibly — for the first-run overnight sweep only |
| Q5 | How does the passport system interact with user-triggered actions (e.g., user manually requests OCR)? | User-triggered operations bypass `battery_allowed` and `requires_idle` — but not `thermal_floor`. User consent overrides scheduling constraints, not safety constraints. |

---

## Summary

- Every Curator background operation carries an **Operation Passport** declaring cost, safety, reversibility, and completeness value.
- The **PassportGate** checks every passport against real-time system state before a job runs. No passport = no run.
- Apple's `BGProcessingTask` (`requiresExternalPower`, `requiresNetworkConnectivity`) and Kubernetes resource requests are the direct prior art — both enforce declared-before-run constraints.
- The `PASSPORT_REGISTRY` is the canonical list of all Curator operations. Adding a new operation requires adding its passport. This is a code-review gate, not a runtime gate.
- `completeness_gain` connects every passport to the Completeness Ledger (R13), ensuring the queue knows what understanding each job delivers.
- Watchdog timeouts (R10) are set from `passport.max_duration_seconds` — no job runs indefinitely without detection.
- The result: Curator's background engine is predictable, thermally safe, and auditable. Any engineer reading a job entry can understand exactly what it will do, what it will cost, and when it is allowed to run — without reading the implementation.

---

## Sources

- [Apple Developer — BGProcessingTask](https://developer.apple.com/documentation/backgroundtasks/bgprocessingtask)
- [Apple Developer — BGTaskScheduler](https://developer.apple.com/documentation/backgroundtasks/bgtaskscheduler)
- [Apple Developer — Choosing Background Strategies for Your App](https://developer.apple.com/documentation/backgroundtasks/choosing-background-strategies-for-your-app)
- [Apple Developer Forums — BGProcessingTask macOS / XPC_ACTIVITY_ALLOW_BATTERY](https://developer.apple.com/forums/thread/119758)
- [WWDC 2025 — Finish tasks in the background (BGContinuedProcessingTask)](https://developer.apple.com/videos/play/wwdc2025/227/)
- [The Eclectic Light Company — Which tasks require mains power? (May 2026)](https://eclecticlight.co/2026/05/27/which-tasks-require-mains-power/)
- [Kubernetes — Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Kubernetes — Resource requests and limits](https://rx-m.com/lesson/cka-understand-how-resource-limits-can-affect-pod-scheduling/)
- [ScaleOps — Kubernetes Resource Requests and Limits: Complete 2026 Guide](https://scaleops.com/blog/kubernetes-resource-requests-and-limits/)
- [Circuit Breaker Pattern — Wikipedia](https://en.wikipedia.org/wiki/Circuit_breaker_design_pattern)
- [LeJOT: Intelligent Job Cost Orchestration for Databricks (arXiv 2025)](https://arxiv.org/pdf/2512.18266)
