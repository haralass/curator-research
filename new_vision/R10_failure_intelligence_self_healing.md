# R10 — Failure Intelligence & Self-Healing

> Cross-cutting layer. Monitors all pipeline stages. Surfaces only what matters to users.
>
> **Core principle: "Errors are internal. Problems are user-facing."**
>
> The user NEVER sees: `PDFSyntaxError`, `EXDEV`, `sqlite pending WAL`, `sidecar timeout`, `hash mismatch`.
> The user ONLY sees: "5 files need help.", "3 files could not be read.", "1 interrupted move was recovered."

---

## Architecture Overview

This is NOT a separate module that runs alongside the pipeline. It is a **woven layer** — every pipeline stage participates in failure intelligence by reporting into shared infrastructure:

```
Scan ─────────────────────────────────────────────────────────────────────┐
Extract ──────────────────── [file_errors table]  [black_box buffer] ─────┤
Embed ────────────────────── [failure_patterns]   [watchdog timers] ──────┤→ Home, Activity, Review Hub
Duplicate Gate ───────────── [pattern detector]   [adaptive safety mode] ─┤
Staging Move ────────────────[WAL recovery]       [preflight checks] ─────┤
Commit ──────────────────────[severity model]     [self-heal actions] ─────┘
Learning ────────────────────[error learning]
```

Failures surface through exactly three user-visible channels:
- **Home** — calm problem cards (only when action is required from the user)
- **Review Hub** — failed files physically routed to correct subfolder
- **Activity** — recovery events, staging moves, undo log

Technical diagnostics go to internal logs only. They are never shown in UI.

---

## Layer 1 — Error Detection

### 1.1 Prior Art

**Kubernetes liveness probes** — distinguish "process alive" from "process functional". A pod can be running but serving 100% errors. Liveness checks the process; readiness checks the function. Curator needs the same distinction: the sidecar process can be alive but the extractor worker can be deadlocked.

**Erlang supervisor trees** — every worker is monitored by a supervisor that owns a restart policy. Workers can crash without bringing down the system. The supervisor decides: restart immediately, restart with delay, or give up and escalate. The key insight: **supervision is structural, not reactive**. You don't add try/except when things go wrong — you design supervision before deployment.

**Watchdog timers** — used in embedded systems and OS kernels. A timer resets when a process "kicks" it. If the timer fires before the next kick, the process is presumed dead. This is the correct model for detecting silent hangs vs. crashes.

**Heartbeat patterns (distributed systems)** — Redis Sentinel, etcd, and RabbitMQ all use heartbeat intervals with configurable miss thresholds. Typical pattern: heartbeat every N seconds, alarm after M misses, action after M+K misses.

**Circuit breaker** (Hystrix, Resilience4j) — detects patterns of failure, not individual failures. After N failures in a window, open the circuit (stop trying) and enter a half-open probe period. Prevents thundering herd on a failing resource.

### 1.2 Silent Failure Types

The hardest failures to detect are the ones that produce no exception:

| Silent failure | Observable signal | Detection method |
|---|---|---|
| Sidecar heartbeat lost | No `POST /heartbeat` for >30s | Swift-side timer |
| Extractor returned "" from non-empty file | `len(text) == 0 AND file_size > 10KB` | Post-extraction validator |
| FSEvents stalled / disk I/O hung | Scan progress counter frozen for >5min | Watchdog on `files_scanned_this_session` counter |
| Embedding worker silently died | Embed queue grows, no progress for >5min | Queue depth + throughput monitor |
| WAL `pending` without transition | WAL entry `last_updated` > N minutes | Startup check + periodic watchdog |
| File stuck in `new` state | State age > 5 minutes | Already in R1 watchdog |
| File stuck in `grouped` state | State age > 6 hours | Already in R1 watchdog |

### 1.3 Curator Decision — Watchdog Intervals

```python
WATCHDOG_THRESHOLDS = {
    # Sidecar / process health
    'sidecar_heartbeat_interval_s': 10,       # sidecar sends heartbeat every 10s
    'sidecar_heartbeat_alarm_s': 30,          # alarm if missed for 30s
    'sidecar_heartbeat_kill_s': 60,           # restart if missed for 60s

    # Extraction
    'extractor_timeout_s': 45,                # per-file extraction timeout
    'extractor_empty_min_bytes': 10_240,      # 10KB — file this large should yield text

    # Scan progress
    'scan_stall_alarm_s': 300,                # 5 minutes without progress → stuck
    'scan_stall_kill_s': 600,                 # 10 minutes → abort scan, alert

    # Embed queue
    'embed_stall_alarm_s': 300,               # 5 min no embed progress → worker dead?
    'embed_stall_kill_s': 600,                # 10 min → restart embed worker

    # State machine
    'state_new_max_s': 300,                   # from R1
    'state_grouped_max_s': 21_600,            # 6h, from R1

    # WAL
    'wal_pending_max_s': 120,                 # 2 minutes — if WAL is pending longer, alert
}
```

**Swift-side watchdog** (`WatchdogMonitor.swift`):
```swift
class WatchdogMonitor {
    private var lastHeartbeat = Date()
    private var timer: Timer?

    func startMonitoring() {
        timer = Timer.scheduledTimer(withTimeInterval: 15, repeats: true) { [weak self] _ in
            guard let self else { return }
            let elapsed = Date().timeIntervalSince(self.lastHeartbeat)
            if elapsed > SIDECAR_HEARTBEAT_KILL_S {
                self.restartSidecar(reason: "heartbeat_lost")
            } else if elapsed > SIDECAR_HEARTBEAT_ALARM_S {
                CuratorLogger.internal("sidecar_heartbeat_alarm elapsed=\(elapsed)")
            }
        }
    }

    func didReceiveHeartbeat() { lastHeartbeat = Date() }
}
```

**Python-side extractor timeout guard** (wraps every extraction call):
```python
import signal
from contextlib import contextmanager

@contextmanager
def extraction_timeout(seconds: int):
    def handler(signum, frame):
        raise TimeoutError(f"extraction exceeded {seconds}s")
    signal.signal(signal.SIGALRM, handler)
    signal.alarm(seconds)
    try:
        yield
    finally:
        signal.alarm(0)

def extract_with_guard(file_path: Path, extractor: Extractor) -> ExtractionResult:
    with extraction_timeout(WATCHDOG_THRESHOLDS['extractor_timeout_s']):
        try:
            text = extractor.extract(file_path)
        except TimeoutError:
            return ExtractionResult.error('extractor_timeout', extractor.name)

    # Silent failure check: non-empty file returned empty text
    file_size = file_path.stat().st_size
    if len(text.strip()) == 0 and file_size > WATCHDOG_THRESHOLDS['extractor_empty_min_bytes']:
        return ExtractionResult.error('empty_extraction', extractor.name)

    return ExtractionResult.ok(text)
```

---

## Layer 2 — Error Classification (Failure Taxonomy)

### 2.1 Prior Art

**HTTP status codes** — the canonical example of a flat taxonomy: 4xx (client fault), 5xx (server fault), with sub-codes for specificity. Usable precisely because every class has exactly one meaning. Problems arise when systems use 400 for everything or 500 for user errors — the same ambiguity Curator must avoid.

**AWS error codes** — hierarchical: service prefix + error name + HTTP status. `S3.NoSuchKey` is unambiguous. The pattern: `domain.ErrorClass` with a human message attached. This is the model for Curator's internal taxonomy.

**PostgreSQL SQLSTATE** — five-character codes with a class (first 2 chars) and a subclass (last 3). `23505` = unique violation (class 23 = integrity constraint). Curator doesn't need five-character codes, but the class/subclass model is useful for routing.

**Sentry event taxonomy** — separates `error` (exception), `warning` (non-fatal), `info` (expected outcome), and `debug` (internal trace). Maps directly to Curator's internal/user distinction.

**What makes a taxonomy good vs. unusable:**
- Every error maps to EXACTLY one class (no overlap)
- Class names are self-describing (not `ERR_001`)
- Classes determine behavior (retry policy, routing, user message) — not just logging
- Stable: classes don't change without migration
- Exhaustive: there is always a fallback class (`unknown_error`)

### 2.2 Complete Taxonomy

```
error_class (string, primary key of taxonomy)

FILE-LEVEL
  permission_denied          — file exists, can't read (EPERM/EACCES)
  file_disappeared           — seen by scanner, gone by extraction time
  icloud_evicted_timeout     — NSFileCoordinator timed out (B2 from R0)
  protected_pdf              — PDF has DRM or password
  corrupted_file             — file fails format validation
  unsupported_format         — no extractor registered for this MIME type
  archive_too_large          — zip/tar/7z exceeds size threshold
  ocr_failed                 — ocrmac returned error or empty
  extractor_timeout          — extraction exceeded N seconds
  empty_extraction           — extractor returned "" from non-empty file (silent failure)
  embedding_failed           — embedding model returned error or NaN vector

MOVE-LEVEL
  hash_mismatch              — SHA-256 of dest doesn't match expected (R0/B1)
  move_failed_permission     — rename() or copy() returned EPERM
  cross_volume_partial       — copy succeeded, delete failed (WAL recovery case)
  destination_collision      — file already exists at exact target path
  disk_full                  — ENOSPC on destination volume
  network_volume_dropped     — NFS/SMB disconnect detected mid-operation

SYSTEM-LEVEL
  sidecar_crashed            — Python process exited unexpectedly
  sidecar_heartbeat_lost     — no heartbeat for > threshold
  db_locked                  — SQLite WAL lock timeout after retries
  stuck_state                — file in a state longer than watchdog threshold

LOGIC-LEVEL
  rule_conflict              — two rules claim the same file with equal priority
  context_conflict           — file matches two contexts with equal confidence
  cannot_link_violation      — Leiden would merge user-separated groups

FALLBACK
  unknown_error              — error not matching any class above (log + alert)
```

**Rule:** the classification function must be called at every catch site. Every exception becomes one of these classes. `unknown_error` should appear in less than 1% of errors — if it's higher, the taxonomy is incomplete.

```python
def classify_error(exc: Exception, context: dict) -> str:
    if isinstance(exc, PermissionError):
        return 'permission_denied'
    if isinstance(exc, FileNotFoundError):
        return 'file_disappeared'
    if isinstance(exc, TimeoutError) and context.get('stage') == 'extract':
        return 'extractor_timeout'
    if isinstance(exc, TimeoutError) and context.get('stage') == 'icloud_download':
        return 'icloud_evicted_timeout'
    if isinstance(exc, OSError) and exc.errno == 28:   # ENOSPC
        return 'disk_full'
    if isinstance(exc, OSError) and exc.errno == 1:    # EPERM
        return 'move_failed_permission'
    # ... full match table
    return 'unknown_error'
```

---

## Layer 3 — Severity / Risk Model

### 3.1 Prior Art

**Sentry severity levels** — `debug`, `info`, `warning`, `error`, `fatal`. The key design principle: severity describes **impact on the system**, not difficulty of fixing. A `fatal` error in Sentry doesn't mean the data is gone — it means the process crashed.

**PagerDuty urgency** — `high` (page someone now), `low` (surface in morning review). Maps cleanly to: does this need a human in the next 5 minutes?

**macOS error domains** — `NSCocoaErrorDomain`, `NSPOSIXErrorDomain`, `NSURLErrorDomain`. Domain encodes severity implicitly: URL errors are often transient, POSIX errors are often systemic. Curator's taxonomy domains (file, move, system, logic) play the same role.

**Escalation by pattern** — production systems rarely assign severity statically. A single 500 error is `warning`; 100 in 60 seconds is `critical`. Rate-based escalation is the correct model.

### 3.2 Four-Level Severity Model

| Level | Meaning | Example | User effect |
|---|---|---|---|
| **Low** | One file fails, batch continues | One PDF is password-protected | No UI interruption. File routed to Review Hub. |
| **Medium** | One group can't complete | OCR fails on 30% of a folder's files | Home card appears after batch. Group goes to Review Hub. |
| **High** | Current batch operation stops | Cross-volume copy fails mid-way | Home card appears immediately. Batch paused. User can resume or dismiss. |
| **Critical** | All file operations halt | WAL shows move started, file missing at both paths | Persistent Home card. No new moves until acknowledged. Cautious/Safe mode activated. |

### 3.3 Rate-Based Escalation

```python
def compute_effective_severity(
    base_severity: str,
    error_class: str,
    recent_errors: list[dict],   # last 60 minutes
    session_move_count: int
) -> str:
    same_class_count = sum(1 for e in recent_errors if e['error_class'] == error_class)

    # Escalate low → medium
    if base_severity == 'low' and same_class_count >= 5:
        return 'medium'

    # Escalate medium → high
    if base_severity == 'medium' and same_class_count >= 10:
        return 'high'

    # Move failures are elevated
    if error_class in MOVE_ERRORS and same_class_count >= 3:
        return 'high'

    # disk_full is always critical
    if error_class == 'disk_full':
        return 'critical'

    return base_severity

MOVE_ERRORS = {
    'hash_mismatch', 'cross_volume_partial', 'destination_collision',
    'move_failed_permission', 'disk_full', 'network_volume_dropped'
}
```

### 3.4 Base Severity by Error Class

```python
SEVERITY_MAP = {
    # Low
    'permission_denied':        'low',
    'file_disappeared':         'low',
    'protected_pdf':            'low',
    'corrupted_file':           'low',
    'unsupported_format':       'low',
    'archive_too_large':        'low',
    'ocr_failed':               'low',
    'extractor_timeout':        'low',
    'empty_extraction':         'low',
    'embedding_failed':         'low',
    'icloud_evicted_timeout':   'low',      # escalates to medium after 3x

    # Medium
    'destination_collision':    'medium',
    'rule_conflict':            'medium',
    'context_conflict':         'medium',
    'network_volume_dropped':   'medium',
    'db_locked':                'medium',
    'stuck_state':              'medium',
    'sidecar_crashed':          'medium',   # auto-recovers → may drop to low
    'sidecar_heartbeat_lost':   'medium',

    # High
    'move_failed_permission':   'high',
    'cross_volume_partial':     'high',     # drops to low after successful recovery
    'hash_mismatch':            'high',
    'cannot_link_violation':    'high',

    # Critical
    'disk_full':                'critical',
    'unknown_error':            'high',     # fail loud until classified
}
```

---

## Layer 4 — Containment

### 4.1 Prior Art

**Bulkhead pattern** (Michael Nygard, "Release It!") — named after ship bulkheads that prevent flooding one compartment from sinking the whole vessel. In software: isolate thread pools, connection pools, or process groups so that a failure in one partition doesn't exhaust shared resources.

**rsync `--partial`** — rsync treats each file as an independent unit. A single file permission error writes to `--log-file` and increments a counter, but rsync continues to the next file. It accumulates errors and reports them at the end. The batch is not contaminated.

**Time Machine error handling** — uses `hfs_clonefile` and falls back to regular copy. Per-file errors are logged to `~/Library/Logs/com.apple.backupd.log` and skipped. The backup completes with a partial success notification ("Some items could not be backed up") rather than a full failure. This is the UX model for Curator.

**Subprocess isolation** — Python's `subprocess` module can run extractors as child processes. A crash in the child doesn't kill the parent. The parent detects non-zero exit code or timeout and classifies the error. This is the correct model for unreliable third-party extractors.

### 4.2 Curator Containment Strategy

**Per-file error bubble** — the extraction loop is structured so exceptions never propagate to the batch:

```python
async def process_batch(files: list[FileRecord]) -> BatchResult:
    results = []
    for file in files:
        try:
            result = await process_single_file(file)
            results.append(result)
        except Exception as exc:
            # Classify, record, route — never re-raise
            error_class = classify_error(exc, {'file_id': file.file_id})
            await record_error(file.file_id, error_class, exc)
            await route_to_review_hub(file, error_class)
            results.append(FileResult.failed(file, error_class))
    return BatchResult(results)
```

**Extractor subprocess isolation** — untrusted extractors (ocrmac, pdfminer, openpyxl) run in a subprocess with a timeout. If the subprocess crashes, the parent records `extractor_timeout` or `corrupted_file` and continues:

```python
def run_extractor_isolated(file_path: Path, extractor_cmd: list[str]) -> str:
    try:
        result = subprocess.run(
            extractor_cmd + [str(file_path)],
            capture_output=True,
            text=True,
            timeout=WATCHDOG_THRESHOLDS['extractor_timeout_s']
        )
        if result.returncode != 0:
            raise RuntimeError(f"extractor exited {result.returncode}: {result.stderr}")
        return result.stdout
    except subprocess.TimeoutExpired:
        raise TimeoutError("extractor_timeout")
```

**Quarantine queue** — failed files are immediately written to `failed_to_read` state in the DB. The scan continues. At no point does the scan loop check "are there failures?" before continuing:

```python
# Correct: write failure state, then continue
async def on_extraction_failure(file: FileRecord, error_class: str):
    await db.execute(
        "UPDATE files SET state = 'failed_to_read' WHERE file_id = ?",
        (file.file_id,)
    )
    await record_error(file.file_id, error_class)
    # fall through — scan loop continues
```

**Move atomicity** — WAL prevents partial moves from corrupting state. Containment at the move layer is provided by Layer 6 (WAL recovery). The containment principle here: if a move fails, the WAL record is left in `pending` state. The file is NOT marked as moved. Recovery on next startup resolves the state.

---

## Layer 5 — Retry & Backoff Policy

### 5.1 Prior Art

**Exponential backoff with jitter** (AWS SDK) — the canonical pattern for transient errors. Without jitter, all retrying clients synchronize and hit the backend simultaneously (thundering herd). Jitter randomizes the retry window. Formula: `min(cap, base * 2^attempt) * random(0, 1)`.

**Circuit breaker for extractors** — if an extractor fails 5 times in a session, open the circuit: stop sending files to it, reduce concurrency, alert the internal log. Half-open probe: after 30 minutes, try one file. If it succeeds, close the circuit.

**Apple BackgroundTasks framework** — `BGAppRefreshTask` is rescheduled on failure with a system-managed delay. The key insight: the OS controls retry timing for battery/network-sensitive tasks. Curator doesn't use BGT, but the principle applies: **the system, not the failing component, controls retry**.

**Dead Letter Queue (Kafka DLQ)** — messages that fail N times are moved to a separate topic (the DLQ) for human inspection. They are never silently discarded. This is Review Hub's purpose for Curator.

### 5.2 Per-Error-Class Retry Table

| Error class | Max retries | Backoff | Final action |
|---|---|---|---|
| `icloud_evicted_timeout` | 3 | 1h, 4h, 24h (+ jitter) | `failed_to_read` (retriable in future session) |
| `extractor_timeout` | 2 | immediate, 60min | `failed_to_read` (permanent for this extractor) |
| `protected_pdf` | 0 | — | `failed_to_read` (permanent) |
| `file_disappeared` | 1 | 30s | discard if still gone |
| `permission_denied` | 0 | — | `failed_to_read` (permanent) |
| `db_locked` | 5 | 100ms, 200ms, 400ms, 800ms, 1600ms | `stuck_state` → alert |
| `network_volume_dropped` | 3 | 30s, 5min, 30min | staging paused, home card |
| `ocr_failed` | 1 | immediate (try fallback extractor) | `failed_to_read` |
| `empty_extraction` | 1 | immediate (try next tier) | Tier 1 only, continue |
| `embedding_failed` | 2 | 5s, 30s | file placed in context by metadata only |
| `sidecar_crashed` | 3 (restarts/session) | 5s, 15s, 60s | critical alert, safe mode |
| `hash_mismatch` | 0 | — | quarantine dest, alert |
| `destination_collision` | 0 | — | auto-rename (Layer 11) |
| `disk_full` | 0 | — | safe mode, home card |

### 5.3 Retry Infrastructure

```python
@dataclass
class RetryPolicy:
    max_retries: int
    backoffs_s: list[float]    # length must equal max_retries
    jitter: bool = True
    final_state: str = 'failed_to_read'

RETRY_POLICIES: dict[str, RetryPolicy] = {
    'icloud_evicted_timeout': RetryPolicy(3, [3600, 14400, 86400], jitter=True),
    'extractor_timeout':      RetryPolicy(2, [0, 3600], jitter=False),
    'db_locked':              RetryPolicy(5, [0.1, 0.2, 0.4, 0.8, 1.6], jitter=True),
    'network_volume_dropped': RetryPolicy(3, [30, 300, 1800], jitter=True),
    'embedding_failed':       RetryPolicy(2, [5, 30], jitter=False),
    'sidecar_crashed':        RetryPolicy(3, [5, 15, 60], jitter=False, final_state='critical'),
    'protected_pdf':          RetryPolicy(0, []),
    'permission_denied':      RetryPolicy(0, []),
    'hash_mismatch':          RetryPolicy(0, []),
}

def next_retry_delay(policy: RetryPolicy, attempt: int) -> float | None:
    if attempt >= policy.max_retries:
        return None
    base = policy.backoffs_s[attempt]
    if policy.jitter and base > 0:
        import random
        return base * random.uniform(0.5, 1.5)
    return base
```

---

## Layer 6 — WAL Recovery

### 6.1 Core Algorithm (from R0/B1)

The 4-case recovery algorithm runs on every startup before accepting new work:

```python
# For every WAL record with status != 'complete' on startup:
src_exists = path_exists(intent.src_path)
dst_exists = path_exists(intent.dest_path)

if src_exists and not dst_exists:
    action = RESTART_FULL_PIPELINE         # copy → verify → delete

elif not src_exists and dst_exists:
    if sha256(intent.dest_path) == intent.expected_sha256:
        action = MARK_COMPLETE             # normal: delete happened, WAL update crashed
    else:
        action = QUARANTINE_DEST_ALERT     # corrupt dest, source gone — alert user

elif src_exists and dst_exists:
    if sha256(intent.dest_path) == intent.expected_sha256:
        action = DELETE_SOURCE_THEN_MARK_COMPLETE  # copy done, delete not attempted
    else:
        action = DELETE_DEST_THEN_RESTART  # partial/corrupt copy, source intact

else:  # neither exists
    action = DISCARD_INTENT_LOG_WARNING    # stale WAL
```

### 6.2 Extended Cases

**Case: Review Hub folder missing at recovery time**

If the destination folder in the WAL intent does not exist (e.g., Review Hub was deleted by user), recovery creates the folder structure before resuming the operation:

```python
def ensure_review_hub_structure(review_hub_root: Path):
    """Re-create Review Hub folder tree from expected schema. Never touches existing files."""
    required_folders = [
        review_hub_root / "Ready to Commit",
        review_hub_root / "Failed to Read" / "Protected",
        review_hub_root / "Failed to Read" / "Corrupted",
        review_hub_root / "Failed to Read" / "Unsupported",
        review_hub_root / "Failed to Read" / "Too Large",
        review_hub_root / "Failed to Read" / "iCloud Pending",
        review_hub_root / "Failed to Read" / "Timeout",
        review_hub_root / "Learning Conflicts",
        review_hub_root / "New Contexts",
        review_hub_root / "Archive",
    ]
    for folder in required_folders:
        folder.mkdir(parents=True, exist_ok=True)
    CuratorLogger.internal("review_hub_structure_restored", path=str(review_hub_root))
    emit_activity_event("recovery", "Review Hub folder structure was restored.")
```

**Case: DB record missing but file is in Review Hub (orphan file)**

On startup, after WAL recovery, scan the Review Hub for files that have no corresponding DB record. These are orphans — they were moved but the DB write failed, or they were placed manually by the user.

```python
async def detect_orphan_files(review_hub_root: Path, db: Database):
    for staged_file in review_hub_root.rglob("*"):
        if staged_file.is_dir():
            continue
        if staged_file.name.startswith("."):
            continue  # skip .DS_Store, hidden files
        sha = compute_sha256(staged_file)
        record = await db.fetchone(
            "SELECT file_id FROM staged_files WHERE sha256 = ?", (sha,)
        )
        if record is None:
            await db.execute("""
                INSERT INTO staged_files (file_id, current_path, sha256, state, notes)
                VALUES (?, ?, ?, 'orphan', 'Detected at startup, no DB record')
            """, (generate_id(), str(staged_file), sha))
            CuratorLogger.internal("orphan_file_detected", path=str(staged_file))
    # Orphan files appear in Review Hub and Activity. User decides what to do with them.
```

**Orphan file handling rule:** Curator never deletes orphan files. It registers them in the DB with state `orphan`, routes them to their current Review Hub subfolder (detected by path), and surfaces one Activity entry: "Curator found files in Review Hub that had no record. They are now tracked."

---

## Layer 7 — User-Facing Explanation Layer

### 7.1 Design Rules

1. Never show error codes, class names, stack traces, errno values, or internal terms
2. Maximum 2 sentences
3. Always say what Curator did about it (routed, will retry, paused, waiting)
4. Use "Curator" as subject: "Curator couldn't read…" — not "Error:", "Failed to:", "Exception:"
5. Tense: past perfect for completed actions ("Curator moved…"), present for ongoing states ("Curator is waiting…")
6. Never use the word "error" or "failure" in user-visible text — use "couldn't", "needs help", "had trouble"

### 7.2 Translation Table

| Internal error class | User sees |
|---|---|
| `permission_denied` | "Curator couldn't access this file — it may be locked by another app. It's been set aside in Review Hub." |
| `file_disappeared` | "This file was moved or deleted before Curator could read it." |
| `icloud_evicted_timeout` | "This file is stored in iCloud and couldn't be downloaded right now. Curator will try again later." |
| `protected_pdf` | "This PDF is password-protected and can't be read. It's been set aside in Review Hub." |
| `corrupted_file` | "This file appears to be damaged. Curator set it aside in Review Hub." |
| `unsupported_format` | "Curator doesn't know how to read this file type yet. It's been set aside in Review Hub." |
| `archive_too_large` | "This archive is too large to open safely. Curator set it aside in Review Hub." |
| `ocr_failed` | "This scanned document couldn't be converted to text. It's been set aside in Review Hub." |
| `extractor_timeout` | "This file took too long to read and was skipped. It's been set aside in Review Hub." |
| `empty_extraction` | "Curator couldn't extract any content from this file. It's been set aside in Review Hub." |
| `embedding_failed` | "Curator had trouble analysing this file and placed it by file type and name only." |
| `hash_mismatch` | "A file move didn't complete safely. Curator has set the file aside for your review." |
| `move_failed_permission` | "Curator couldn't move this file — it may be in use by another app. It's been set aside." |
| `cross_volume_partial` (recovered) | "An interrupted file move was safely recovered." |
| `destination_collision` | "A file with the same name already exists in the destination. Curator renamed it to avoid overwriting." |
| `disk_full` | "Your disk is almost full. Curator has paused to avoid any issues. Please free up some space." |
| `network_volume_dropped` | "The external drive or network folder became unavailable. Curator paused and will retry when it reconnects." |
| `sidecar_crashed` (recovered) | "Curator restarted after an unexpected issue and is resuming where it left off." |
| `sidecar_heartbeat_lost` (recovered) | "Curator restarted after an unexpected issue and is resuming where it left off." |
| `db_locked` (resolved) | (no user message — resolves within milliseconds, only logged internally) |
| `stuck_state` | "A file has been waiting for a while. Curator is looking into it." |
| `rule_conflict` | "Two of your patterns match this file. Curator is waiting for your guidance." |
| `context_conflict` | "Curator isn't sure which group this file belongs to. It's been set aside for your review." |
| `cannot_link_violation` | "Curator noticed a grouping conflict with your previous decisions. It's waiting for your guidance." |
| `unknown_error` | "Curator had trouble with this file and set it aside in Review Hub." |

### 7.3 Message Aggregation

Individual per-file messages are only shown if the user drills into a file. Home cards aggregate:

- "5 files need help." (links to Review Hub > Failed to Read)
- "2 files are waiting for your guidance." (links to Review Hub > Learning Conflicts)
- "1 interrupted move was safely recovered." (links to Activity)
- "Curator is paused — your disk is almost full." (persistent card, no dismiss without action)

---

## Layer 8 — Error Memory

### 8.1 Schema

```sql
CREATE TABLE file_errors (
    error_id        TEXT PRIMARY KEY,           -- UUID
    file_id         TEXT NOT NULL,              -- FK to files table
    error_class     TEXT NOT NULL,              -- from taxonomy (Layer 2)
    severity        TEXT NOT NULL,              -- low / medium / high / critical
    error_count     INTEGER DEFAULT 1,
    first_seen      TEXT NOT NULL,              -- ISO 8601
    last_seen       TEXT NOT NULL,
    extractor       TEXT,                       -- which extractor failed (if applicable)
    session_id      TEXT,                       -- which session this occurred in
    detail          TEXT,                       -- JSON: technical details (internal only, never shown in UI)
    resolved_at     TEXT,                       -- NULL if still active
    resolved_by     TEXT,                       -- 'user' | 'auto' | 'retry_success' | NULL

    FOREIGN KEY (file_id) REFERENCES files(file_id) ON DELETE CASCADE
);

CREATE INDEX idx_file_errors_file_id ON file_errors(file_id);
CREATE INDEX idx_file_errors_class ON file_errors(error_class);
CREATE INDEX idx_file_errors_last_seen ON file_errors(last_seen);
CREATE INDEX idx_file_errors_unresolved ON file_errors(resolved_at) WHERE resolved_at IS NULL;
```

### 8.2 Key Behaviors

**Permanent skip after repeated failure:**
```python
async def should_skip_ocr(file_id: str, db: Database) -> bool:
    row = await db.fetchone("""
        SELECT error_count FROM file_errors
        WHERE file_id = ? AND error_class = 'ocr_failed' AND resolved_at IS NULL
    """, (file_id,))
    return row is not None and row['error_count'] >= 2

async def should_skip_icloud_retry(file_id: str, db: Database) -> bool:
    row = await db.fetchone("""
        SELECT error_count FROM file_errors
        WHERE file_id = ? AND error_class = 'icloud_evicted_timeout' AND resolved_at IS NULL
    """, (file_id,))
    return row is not None and row['error_count'] >= 3
```

**Error count increment (upsert pattern):**
```python
async def record_error(file_id: str, error_class: str, exc: Exception, extractor: str | None, db: Database):
    now = utcnow_iso()
    detail = json.dumps({'msg': str(exc), 'type': type(exc).__name__})
    await db.execute("""
        INSERT INTO file_errors (error_id, file_id, error_class, severity, first_seen, last_seen, extractor, detail)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        ON CONFLICT (file_id, error_class) DO UPDATE SET
            error_count = error_count + 1,
            last_seen = excluded.last_seen,
            detail = excluded.detail
    """, (new_uuid(), file_id, error_class, SEVERITY_MAP[error_class], now, now, extractor, detail))
```

Note: the `ON CONFLICT` clause requires a unique index on `(file_id, error_class)` for active (unresolved) errors:

```sql
CREATE UNIQUE INDEX idx_file_errors_active ON file_errors(file_id, error_class)
    WHERE resolved_at IS NULL;
```

**On retry success:**
```python
async def mark_error_resolved(file_id: str, error_class: str, resolved_by: str, db: Database):
    await db.execute("""
        UPDATE file_errors
        SET resolved_at = ?, resolved_by = ?
        WHERE file_id = ? AND error_class = ? AND resolved_at IS NULL
    """, (utcnow_iso(), resolved_by, file_id, error_class))
```

Error history is kept permanently. `resolved_at` being non-NULL does not delete the row — the history informs Layer 16 (Error Learning).

---

## Layer 9 — Error Pattern Detection

### 9.1 Design Principles

Pattern detection runs as a background query, not on every error. It fires:
- After every batch completes
- Every 15 minutes during active sessions
- On session start (looking for patterns from the last 24 hours)

The goal: detect **systemic failures** that indicate an external condition (network, disk, iCloud) rather than a single bad file. Systemic failures need a different response than per-file failures.

### 9.2 Detection Queries

**Folder-level failure pattern:**
```sql
SELECT
    f.folder_path,
    fe.error_class,
    COUNT(*) AS n,
    MIN(fe.first_seen) AS earliest,
    MAX(fe.last_seen) AS latest
FROM file_errors fe
JOIN files f ON fe.file_id = f.file_id
WHERE fe.last_seen > datetime('now', '-1 hour')
  AND fe.resolved_at IS NULL
GROUP BY f.folder_path, fe.error_class
HAVING n >= 5
ORDER BY n DESC;
```

Curator interpretation: if this query returns a row, generate a systemic notice: "Folder '{folder}' has {n} files that {user_message_for_class}. They have all been set aside in Review Hub."

**Extractor degradation detection:**
```sql
SELECT extractor, COUNT(*) AS n
FROM file_errors
WHERE error_class IN ('extractor_timeout', 'empty_extraction', 'ocr_failed')
  AND last_seen > datetime('now', '-' || ? || ' minutes')
  AND resolved_at IS NULL
GROUP BY extractor
HAVING n >= 5;
```

If this fires: reduce extractor concurrency for that extractor, log `extractor_degraded` internally.

**iCloud systemic failure detection:**
```sql
SELECT COUNT(*) AS n
FROM file_errors
WHERE error_class = 'icloud_evicted_timeout'
  AND last_seen > datetime('now', '-30 minutes')
  AND resolved_at IS NULL;
-- If n > 5: iCloud connectivity issue. Pause all iCloud file operations.
```

**Post-restart failure spike:**
```sql
SELECT COUNT(*) AS n
FROM file_errors fe
JOIN curator_events ce ON ce.event_type = 'sidecar_restart'
WHERE fe.first_seen > ce.timestamp
  AND ce.timestamp > datetime('now', '-30 minutes');
-- If n > 10: sidecar version issue or environment problem. Alert internal log.
```

**Error rate computation (for Adaptive Safety Mode):**
```sql
SELECT
    COUNT(*) FILTER (WHERE fe.resolved_at IS NULL) AS active_errors,
    COUNT(*) AS total_processed
FROM files f
LEFT JOIN file_errors fe ON f.file_id = fe.file_id
WHERE f.last_seen > datetime('now', '-1 hour');
-- error_rate = active_errors / MAX(total_processed, 1)
-- IF error_rate > 0.20: trigger cautious mode
```

---

## Layer 10 — Failure Graph (Internal)

### 10.1 Design Philosophy

This is not a graph database — it is a set of diagnostic queries over existing tables (`file_errors`, `files`, `curator_events`, `staged_files`). No additional schema is needed. The "failure graph" is the set of joins that reveal why something failed.

The purpose: answer post-hoc diagnostic questions without requiring a developer to hand-write SQL. These queries are used internally (developer logs, support triage) and power Curator's own pattern detection.

### 10.2 Diagnostic Query Library

**"Why did the EPL342 folder mostly fail?"**
```sql
SELECT
    f.file_name,
    f.mime_type,
    f.file_size_bytes,
    fe.error_class,
    fe.extractor,
    fe.error_count,
    fe.detail
FROM files f
JOIN file_errors fe ON f.file_id = fe.file_id
WHERE f.folder_path LIKE '%EPL342%'
  AND fe.resolved_at IS NULL
ORDER BY fe.error_class, f.file_size_bytes DESC;
```

**"Why did staging fail yesterday?"**
```sql
SELECT
    sf.original_path,
    sf.wal_status,
    fe.error_class,
    fe.detail,
    ce.timestamp AS event_time,
    ce.detail AS event_detail
FROM staged_files sf
LEFT JOIN file_errors fe ON sf.file_id = fe.file_id
LEFT JOIN curator_events ce ON ce.file_id = sf.file_id
WHERE sf.staged_at BETWEEN datetime('now', '-25 hours') AND datetime('now', '-1 hour')
  AND (sf.wal_status != 'complete' OR fe.error_class IS NOT NULL)
ORDER BY ce.timestamp DESC;
```

**"Which extractors are failing most this session?"**
```sql
SELECT
    fe.extractor,
    fe.error_class,
    COUNT(*) AS failures,
    AVG(CAST(json_extract(fe.detail, '$.duration_ms') AS REAL)) AS avg_duration_ms
FROM file_errors fe
WHERE fe.session_id = (SELECT session_id FROM curator_sessions ORDER BY started_at DESC LIMIT 1)
GROUP BY fe.extractor, fe.error_class
ORDER BY failures DESC;
```

**"What is the complete error history for file X?"**
```sql
SELECT
    fe.error_class,
    fe.error_count,
    fe.first_seen,
    fe.last_seen,
    fe.resolved_at,
    fe.resolved_by,
    fe.extractor,
    fe.detail
FROM file_errors fe
WHERE fe.file_id = ?
ORDER BY fe.first_seen DESC;
```

**"Are there any files that succeeded after repeated failures? (Learning signal)"**
```sql
SELECT
    f.file_name,
    f.mime_type,
    fe.error_class,
    fe.error_count,
    fe.resolved_at,
    fe.resolved_by
FROM file_errors fe
JOIN files f ON fe.file_id = f.file_id
WHERE fe.resolved_by = 'retry_success'
  AND fe.error_count >= 2
ORDER BY fe.error_count DESC;
```

---

## Layer 11 — Self-Healing Actions

### 11.1 Design Principle

Self-healing is conservative. Curator acts autonomously only when:
1. The action is reversible (or provably safe)
2. The action is logged to Activity
3. The action has a guard that prevents infinite loops

Self-healing is NEVER:
- Deleting user files
- Overwriting existing files silently
- Making physical moves without WAL
- Ignoring `critical` severity
- Acting without recording the action

### 11.2 Self-Heal Action Table

| Problem detected | Self-heal action | Guard | User notification |
|---|---|---|---|
| Extractor worker crashed | Restart worker subprocess | Max 3 restarts per session | None (silent if recovered) |
| Sidecar heartbeat lost >30s | Swift launcher restarts Python sidecar | After 60s silence, escalate to critical | Activity: "Curator restarted after an unexpected issue." |
| Review Hub folder missing | Re-create full folder structure from expected schema | Never removes existing files | Activity: "Review Hub folder structure was restored." |
| `.curator_group.json` missing | Re-generate from DB state | Log recovery event | None (internal file) |
| Destination collision on commit | Auto-rename: `filename_curator_1.ext`, `_curator_2`, etc. | Never overwrites; if collision after 10 attempts, route to Review Hub instead | Activity: "File renamed to avoid a name conflict." |
| WAL `pending` on startup | Run 4-case recovery algorithm | Only on startup, one pass | Activity: "1 interrupted move was safely recovered." (if action taken) |
| Embedding cache corrupted | Rebuild cache for affected files only | Background, lowest priority, max 50 files/session | None |
| `db_locked` after retries | Wait for lock release (exponential backoff) | After 5 retries: log `stuck_state`, alert | Home card if db_locked persists >5min |
| Orphan file in Review Hub | Register in DB as `orphan` state | Never moves or deletes | Activity: "Curator found an untracked file in Review Hub." |
| iCloud systemic timeout | Pause all iCloud file staging | Resume when single iCloud file succeeds | Home card: "iCloud files are paused temporarily." |
| Extractor degraded (>5 failures) | Reduce concurrency for that extractor | Per extractor, per session | None (logged internally) |

### 11.3 Self-Heal Circuit Breaker

```python
class SelfHealCircuit:
    """Prevents self-heal actions from looping."""

    def __init__(self):
        self._counts: dict[str, int] = {}
        self._limits: dict[str, int] = {
            'extractor_restart': 3,
            'sidecar_restart': 3,
            'db_retry': 5,
            'destination_rename': 10,
        }

    def can_act(self, action: str) -> bool:
        current = self._counts.get(action, 0)
        limit = self._limits.get(action, 1)
        if current >= limit:
            CuratorLogger.internal(f"self_heal_circuit_open action={action} count={current}")
            return False
        return True

    def record_action(self, action: str):
        self._counts[action] = self._counts.get(action, 0) + 1
```

---

## Layer 12 — Preflight Simulation

### 12.1 Prior Art

**SMART disk pre-flight** — storage drivers run read-ahead checks before committing a large write. If SMART reports sectors near failure, the OS escalates before data loss.

**macOS disk image verification** — `hdiutil` verifies checksums before mounting. Preflight in Curator is the same concept: check the conditions before committing the operation.

**Dry-run mode** — `rsync --dry-run`, `terraform plan`, `ansible --check`. Run the operation in read-only simulation mode and report what would fail. Curator's preflight does not simulate moves but does predict failures from observable conditions.

### 12.2 Preflight Implementation

```python
@dataclass
class PreflightCheck:
    name: str
    severity: str          # 'low', 'medium', 'high'
    passed: bool
    message: str | None    # None if passed

@dataclass
class PreflightResult:
    checks: list[PreflightCheck]

    @property
    def max_severity(self) -> str:
        failed = [c for c in self.checks if not c.passed]
        if not failed:
            return 'none'
        levels = {'low': 0, 'medium': 1, 'high': 2}
        return max(failed, key=lambda c: levels[c.severity]).severity

    @property
    def should_abort(self) -> bool:
        return self.max_severity == 'high'

    @property
    def should_warn(self) -> bool:
        return self.max_severity == 'medium'


def preflight_check(files: list[FileRecord], destination: Path) -> PreflightResult:
    checks = [
        _check_disk_space(files, destination),
        _check_same_volume(files, destination),
        _check_permissions(files, destination),
        _check_locked_files(files),
        _check_icloud_evicted(files),
        _check_filename_collisions(files, destination),
        _check_network_volume_stability(destination),
    ]
    return PreflightResult(checks)


def _check_disk_space(files: list[FileRecord], destination: Path) -> PreflightCheck:
    total_bytes = sum(f.file_size_bytes for f in files)
    stat = shutil.disk_usage(destination)
    free = stat.free
    # Require 110% of total file size to be available (10% buffer for metadata)
    needed = int(total_bytes * 1.1)
    passed = free >= needed
    return PreflightCheck(
        name='disk_space',
        severity='high',
        passed=passed,
        message=None if passed else f"Not enough space. Need {needed//1024//1024}MB, have {free//1024//1024}MB."
    )


def _check_same_volume(files: list[FileRecord], destination: Path) -> PreflightCheck:
    # Cross-volume means copy-verify-delete path, not atomic rename
    # This is informational, not a failure — just affects WAL strategy
    src_dev = os.stat(files[0].current_path).st_dev if files else None
    dst_dev = os.stat(destination).st_dev
    same = (src_dev == dst_dev)
    return PreflightCheck(
        name='same_volume',
        severity='low',
        passed=True,    # cross-volume is handled, not a failure
        message=None if same else "Files will be moved across volumes — Curator will verify each one."
    )


def _check_filename_collisions(files: list[FileRecord], destination: Path) -> PreflightCheck:
    collisions = [f for f in files if (destination / f.file_name).exists()]
    passed = len(collisions) == 0
    return PreflightCheck(
        name='filename_collisions',
        severity='medium',
        passed=passed,
        message=None if passed else f"{len(collisions)} file(s) already exist at the destination and will be renamed."
    )
```

### 12.3 Preflight → Action

```python
async def commit_batch_with_preflight(files: list[FileRecord], destination: Path):
    result = preflight_check(files, destination)

    if result.should_abort:
        # High severity: abort batch, explain why
        failed_checks = [c for c in result.checks if not c.passed]
        for check in failed_checks:
            await emit_home_card(check.message, severity='high')
        return BatchResult.aborted()

    if result.should_warn:
        # Medium severity: warn but user can still proceed
        # (In current design: proceed automatically, but log warning)
        CuratorLogger.internal("preflight_warnings", checks=[c.message for c in result.checks if not c.passed])

    # Low or none: proceed
    return await execute_batch_move(files, destination)
```

---

## Layer 13 — Black Box Recorder

### 13.1 Prior Art

**Flight data recorders** — overwrite oldest data when buffer is full. The last N minutes are always available. This is the circular buffer model.

**macOS CrashReporter** — writes a structured crash report to `~/Library/Logs/DiagnosticReports/` on process termination. The report includes the last call stack, thread states, and memory info. Curator's black box serves the same purpose for worker-level debugging.

**Linux `kdump` / kernel ring buffer** — the last N kernel log messages always available via `dmesg`. The ring buffer never blocks the kernel — it overwrites. Curator's in-memory buffer follows the same model: never write to disk during normal operation.

**OpenTelemetry spans** — structured records of operations with start time, duration, and attributes. Each entry in Curator's black box is equivalent to a completed span.

### 13.2 Schema and Implementation

```python
from collections import deque
from dataclasses import dataclass, field
import json, time

BLACKBOX_SIZE = 500     # entries per worker
BLACKBOX_FLUSH_DIR = Path.home() / "Library" / "Logs" / "Curator"

@dataclass
class BlackBoxEntry:
    timestamp: str
    worker_id: str
    file_path: str
    file_state: str
    operation: str          # 'extract', 'embed', 'move', 'hash', 'scan', 'wal_check'
    extractor: str
    result: str             # 'ok', 'error', 'timeout', 'skipped'
    error_class: str        # '' if result == 'ok'
    duration_ms: int
    detail: str             # JSON string


class BlackBoxRecorder:
    def __init__(self, worker_id: str):
        self.worker_id = worker_id
        self._buffer: deque[BlackBoxEntry] = deque(maxlen=BLACKBOX_SIZE)

    def record(self, **kwargs):
        entry = BlackBoxEntry(
            timestamp=utcnow_iso(),
            worker_id=self.worker_id,
            **kwargs
        )
        self._buffer.append(entry)

    def flush_to_disk(self, reason: str = 'crash'):
        BLACKBOX_FLUSH_DIR.mkdir(parents=True, exist_ok=True)
        ts = int(time.time())
        path = BLACKBOX_FLUSH_DIR / f"blackbox_{self.worker_id}_{ts}.json"
        data = {
            'reason': reason,
            'worker_id': self.worker_id,
            'flushed_at': utcnow_iso(),
            'entries': [asdict(e) for e in self._buffer]
        }
        path.write_text(json.dumps(data, indent=2))
        return path
```

**Flush triggers:**
1. Sidecar receives `SIGTERM` / `SIGABRT` — flush all worker buffers
2. Worker process exits with non-zero code — parent flushes that worker's buffer
3. Developer request via internal API (`GET /debug/blackbox`) — flush to disk, return path

**Never written to main SQLite DB** — too noisy, too transient. The black box is a crash forensics tool, not an audit log.

---

## Layer 14 — Error → Review Hub Routing Table

### 14.1 Physical Routing

| Error class | Physical destination | DB state |
|---|---|---|
| `protected_pdf` | `Review Hub/Failed to Read/Protected/` | `failed_to_read` |
| `corrupted_file` | `Review Hub/Failed to Read/Corrupted/` | `failed_to_read` |
| `ocr_failed` | `Review Hub/Failed to Read/Corrupted/` | `failed_to_read` |
| `empty_extraction` | `Review Hub/Failed to Read/Corrupted/` | `failed_to_read` |
| `permission_denied` | `Review Hub/Failed to Read/Protected/` | `failed_to_read` |
| `unsupported_format` | `Review Hub/Failed to Read/Unsupported/` | `failed_to_read` |
| `archive_too_large` | `Review Hub/Failed to Read/Too Large/` | `failed_to_read` |
| `icloud_evicted_timeout` (3x) | `Review Hub/Failed to Read/iCloud Pending/` | `failed_to_read` |
| `extractor_timeout` (2x) | `Review Hub/Failed to Read/Timeout/` | `failed_to_read` |
| `embedding_failed` | No routing — file placed by metadata only | `embedded_partial` |
| `rule_conflict` | `Review Hub/Learning Conflicts/` | `conflict` |
| `context_conflict` | `Review Hub/New Contexts/` | `uncertain` |
| `cannot_link_violation` | `Review Hub/Learning Conflicts/` | `conflict` |
| `cross_volume_partial` (recovered) | Activity entry only — no physical move | resolved |
| `disk_full` | Home card only — no staging | `pending` (paused) |
| `sidecar_crashed` (recovered) | Activity entry only | resolved |
| `sidecar_heartbeat_lost` (recovered) | Activity entry only | resolved |
| `stuck_state` (watchdog) | Home card only — investigate, don't move | still active |
| `hash_mismatch` | `Review Hub/Failed to Read/Corrupted/` | `failed_to_read` |
| `move_failed_permission` | `Review Hub/Failed to Read/Protected/` | `failed_to_read` |
| `network_volume_dropped` | No staging — batch paused | `pending` (paused) |
| `destination_collision` | Auto-renamed, committed normally | `committed` |
| `unknown_error` | `Review Hub/Failed to Read/` (root) | `failed_to_read` |

### 14.2 Routing Logic

```python
REVIEW_HUB_ROUTES: dict[str, str | None] = {
    'protected_pdf':            'Failed to Read/Protected',
    'permission_denied':        'Failed to Read/Protected',
    'move_failed_permission':   'Failed to Read/Protected',
    'corrupted_file':           'Failed to Read/Corrupted',
    'ocr_failed':               'Failed to Read/Corrupted',
    'empty_extraction':         'Failed to Read/Corrupted',
    'hash_mismatch':            'Failed to Read/Corrupted',
    'unsupported_format':       'Failed to Read/Unsupported',
    'archive_too_large':        'Failed to Read/Too Large',
    'icloud_evicted_timeout':   'Failed to Read/iCloud Pending',
    'extractor_timeout':        'Failed to Read/Timeout',
    'rule_conflict':            'Learning Conflicts',
    'context_conflict':         'New Contexts',
    'cannot_link_violation':    'Learning Conflicts',
    'unknown_error':            'Failed to Read',
    # None = no physical routing
    'embedding_failed':         None,
    'cross_volume_partial':     None,
    'disk_full':                None,
    'sidecar_crashed':          None,
    'sidecar_heartbeat_lost':   None,
    'stuck_state':              None,
    'network_volume_dropped':   None,
    'destination_collision':    None,
}

async def route_failed_file(file: FileRecord, error_class: str, review_hub_root: Path, db: Database):
    subfolder = REVIEW_HUB_ROUTES.get(error_class)
    if subfolder is None:
        return  # No physical routing — handled by home card or activity only

    dest_dir = review_hub_root / subfolder
    dest_dir.mkdir(parents=True, exist_ok=True)

    dest_path = safe_destination(dest_dir, file.file_name)  # handles collisions
    await move_with_wal(file.current_path, dest_path, db)

    await db.execute(
        "UPDATE files SET state = 'failed_to_read', current_path = ? WHERE file_id = ?",
        (str(dest_path), file.file_id)
    )
```

---

## Layer 15 — Adaptive Safety Mode

### 15.1 Prior Art

**macOS low power mode** — the OS reduces CPU frequency and background activity when battery is low. The user can override. Curator's cautious mode is the equivalent: reduce operations when the environment is hostile.

**Kubernetes pod disruption budget** — limits the number of disruptions (pods killed) at once. If too many pods fail simultaneously, the rollout pauses. Curator's cautious mode is analogous: if too many file operations fail, pause staging.

**Circuit breaker half-open state** — after the circuit is open (failures are too high), the system waits, then tries one operation. If it succeeds, close the circuit. If it fails, stay open. Curator's cautious → normal transition uses the same model.

### 15.2 Mode Definitions

**Normal mode** — full pipeline active. Physical staging, batch moves, all extractors, background embedding.

**Cautious mode** — triggered by:
- Error rate > 20% in last 60 minutes, OR
- > 5 move-level failures in current session, OR
- `sidecar_crashed` twice in session

Actions in cautious mode:
- Physical staging PAUSED
- Scanning and classification CONTINUE
- Embedding CONTINUES
- Home card: "Curator is being careful — some moves are paused until issues resolve."
- Background: run error pattern detection every 5 minutes
- Auto-exit: 30 minutes of clean operation (error rate < 5%)

**Safe mode** — triggered by:
- Any `critical` severity error, OR
- `disk_full`, OR
- `hash_mismatch` with file at neither source nor dest (data loss risk)

Actions in safe mode:
- ALL file operations PAUSED
- Read-only scanning CONTINUES
- Persistent Home card (no dismiss button — requires user acknowledgment)
- Home card: "Curator has paused to protect your files. [See what happened]"

### 15.3 State Machine

```python
class SafetyMode(Enum):
    NORMAL = 'normal'
    CAUTIOUS = 'cautious'
    SAFE = 'safe'

class AdaptiveSafetyController:
    def __init__(self):
        self.mode = SafetyMode.NORMAL
        self._clean_run_since: datetime | None = None
        self._move_failures_this_session = 0
        self._sidecar_restarts_this_session = 0

    def on_error(self, error_class: str, severity: str, error_rate: float):
        if severity == 'critical' or error_class == 'disk_full':
            self._transition(SafetyMode.SAFE, reason=f"{error_class}")
        elif (error_rate > 0.20 or
              self._move_failures_this_session >= 5 or
              self._sidecar_restarts_this_session >= 2):
            if self.mode == SafetyMode.NORMAL:
                self._transition(SafetyMode.CAUTIOUS, reason=f"error_rate={error_rate:.0%}")

        if error_class in MOVE_ERRORS:
            self._move_failures_this_session += 1
        if error_class == 'sidecar_crashed':
            self._sidecar_restarts_this_session += 1

    def on_successful_operation(self):
        if self.mode == SafetyMode.CAUTIOUS:
            if self._clean_run_since is None:
                self._clean_run_since = datetime.utcnow()
            elif (datetime.utcnow() - self._clean_run_since).seconds >= 1800:  # 30 min
                self._transition(SafetyMode.NORMAL, reason="30min clean run")

    def on_user_acknowledged(self):
        if self.mode == SafetyMode.SAFE:
            self._transition(SafetyMode.CAUTIOUS, reason="user_acknowledged")

    def _transition(self, new_mode: SafetyMode, reason: str):
        CuratorLogger.internal(f"safety_mode_transition {self.mode.value} → {new_mode.value} reason={reason}")
        emit_activity_event("safety_mode", f"Curator switched to {new_mode.value} mode: {reason}")
        self.mode = new_mode
        if new_mode != SafetyMode.CAUTIOUS:
            self._clean_run_since = None
```

---

## Layer 16 — Error Learning

### 16.1 Design Principle

Curator should fail less over time, not just recover better. Error learning converts repeated failures into proactive decisions: if a pattern of failures is reliably associated with a signal (MIME type, file size, folder, time of day), encode that signal as a strategy override and apply it before the next failure occurs.

This is distinct from user correction learning (R5) — it is **system-level learning from its own failures**, not from user feedback.

### 16.2 Schema

```sql
CREATE TABLE failure_patterns (
    pattern_id          TEXT PRIMARY KEY,       -- UUID
    error_class         TEXT NOT NULL,
    signal              TEXT NOT NULL,          -- expression: 'mime_type=X AND size_mb>Y'
    learned_at          TEXT NOT NULL,
    observation_count   INTEGER DEFAULT 1,
    confidence          REAL DEFAULT 0.0,       -- 0.0 to 1.0
    strategy_override   TEXT NOT NULL,          -- what to do instead
    last_applied_at     TEXT,
    applied_count       INTEGER DEFAULT 0,
    success_rate        REAL                    -- how often the override succeeded
);

CREATE INDEX idx_failure_patterns_class ON failure_patterns(error_class);
```

### 16.3 Learned Patterns and Their Overrides

| Observation | Signal | Strategy override |
|---|---|---|
| `ocr_failed` on file | `mime_type=application/pdf AND size_mb > 5` | Skip OCR entirely, use Tier 1 only (metadata + filename) |
| `archive_too_large` on `.zip > 500MB` | `mime_type=application/zip AND size_mb > 500` | Skip central directory read; treat as opaque blob |
| `extractor_timeout` on `.xlsx` with 100+ sheets | `mime_type=application/vnd.ms-excel AND sheet_count > 100` | Read only first 3 sheets |
| `icloud_evicted_timeout` between 2–4 AM | `is_icloud=true AND hour_utc BETWEEN 2 AND 4` | Defer iCloud file staging until after 6 AM |
| `empty_extraction` on scanned PDF | `mime_type=application/pdf AND has_text_layer=false` | Route directly to OCR tier, skip pdfminer |
| `extractor_timeout` on `.pptx > 50MB` | `mime_type=application/vnd.ms-powerpoint AND size_mb > 50` | Use metadata + slide count only |

### 16.4 Pattern Learning Algorithm

```python
MIN_OBSERVATIONS = 3          # minimum failures before encoding a pattern
MIN_CONFIDENCE = 0.75         # minimum signal reliability

async def learn_from_failures(db: Database):
    """Run after each session. Encode patterns from repeated failures."""

    # Find signals that reliably predict specific error classes
    rows = await db.fetchall("""
        SELECT
            fe.error_class,
            f.mime_type,
            CASE
                WHEN f.file_size_bytes > 500*1024*1024 THEN 'size_mb>500'
                WHEN f.file_size_bytes > 50*1024*1024  THEN 'size_mb>50'
                WHEN f.file_size_bytes > 5*1024*1024   THEN 'size_mb>5'
                ELSE 'size_mb<5'
            END AS size_bucket,
            COUNT(*) AS n,
            COUNT(*) FILTER (WHERE fe.resolved_at IS NULL) AS still_failing
        FROM file_errors fe
        JOIN files f ON fe.file_id = f.file_id
        WHERE fe.error_count >= 1
        GROUP BY fe.error_class, f.mime_type, size_bucket
        HAVING n >= ?
    """, (MIN_OBSERVATIONS,))

    for row in rows:
        confidence = row['still_failing'] / row['n']
        if confidence < MIN_CONFIDENCE:
            continue

        signal = f"mime_type={row['mime_type']} AND {row['size_bucket']}"
        override = STRATEGY_OVERRIDES.get((row['error_class'], row['mime_type']))
        if override is None:
            continue

        await db.execute("""
            INSERT INTO failure_patterns (pattern_id, error_class, signal, learned_at, observation_count, confidence, strategy_override)
            VALUES (?, ?, ?, ?, ?, ?, ?)
            ON CONFLICT (error_class, signal) DO UPDATE SET
                observation_count = excluded.observation_count,
                confidence = excluded.confidence,
                learned_at = excluded.learned_at
        """, (new_uuid(), row['error_class'], signal, utcnow_iso(), row['n'], confidence, override))
```

### 16.5 Applying Learned Patterns

```python
async def get_strategy_override(file: FileRecord, db: Database) -> str | None:
    """Before extraction: check if a learned pattern applies. Returns override or None."""
    rows = await db.fetchall("""
        SELECT strategy_override, confidence
        FROM failure_patterns
        WHERE error_class IN ('ocr_failed', 'extractor_timeout', 'empty_extraction', 'archive_too_large')
        ORDER BY confidence DESC
    """)
    for row in rows:
        if evaluate_signal(row['signal'], file):    # simple expression evaluator
            return row['strategy_override']
    return None
```

**Feedback loop:** when an override is applied and the file is processed successfully (reaches `embedded` or `committed` state), increment `applied_count` and update `success_rate`. If `success_rate < 0.5` after 10 applications, flag the pattern for review and stop applying it.

---

## Design Decisions

1. **Error taxonomy is behavior-determining, not just descriptive.** Every error class maps to exactly one retry policy, one severity level, one user message, and one Review Hub route. Adding a new error class requires updating all four tables — this is intentional friction that prevents incomplete error handling.

2. **Silent failures are first-class failures.** Empty extraction from a non-empty file, scan progress frozen, embedding queue backed up with no progress — these are detected via watchdog timers and post-extraction validators, not exceptions. They are classified and handled identically to thrown exceptions.

3. **The taxonomy has a fallback: `unknown_error`.** Any exception that doesn't match a known class maps to `unknown_error` with severity `high`. This fails loudly (Activity event, Home card) rather than silently. The metric "unknown_error rate" is a signal for taxonomy completeness.

4. **Error memory uses upsert, not append-only.** Active errors are deduplicated per `(file_id, error_class)`. The `error_count` column tracks how many times the same file has hit the same error. This drives the "skip OCR permanently after 2 failures" logic without requiring a separate query.

5. **Self-healing has a circuit breaker.** Every self-heal action has a per-session limit. If sidecar restarts 3 times in a session and still fails, the circuit opens and severity escalates to critical. Self-healing never loops silently.

6. **WAL recovery runs before accepting any new work on startup.** The 4-case algorithm from R0/B1 is the first thing executed after DB connection is established. No new moves are queued until all pending WAL records are resolved or quarantined.

7. **Review Hub routing is a physical action with a WAL record.** Routing a failed file to `Review Hub/Failed to Read/Protected/` is a move operation — it follows the same WAL pattern as all other moves. It is not just a DB label change.

8. **User-facing messages never use the word "error" or "failure".** The words "couldn't", "needs help", "had trouble", "set aside" are the vocabulary. The word "error" is reserved for internal logs. This is enforced by code review convention.

9. **The black box is in-memory only during normal operation.** It is never written to the main SQLite DB. It flushes to `~/Library/Logs/Curator/` only on crash or developer request. This keeps the main DB clean and fast.

10. **Adaptive safety mode transitions are logged to Activity.** Every mode transition (Normal → Cautious, Cautious → Normal, Cautious → Safe) generates an Activity event. Users can see why Curator changed behavior. Transitions are also in internal logs with the specific triggering condition.

11. **Error learning requires MIN_OBSERVATIONS=3 and MIN_CONFIDENCE=0.75.** A single failure never generates a pattern. Three failures with the same signal and 75% still-failing rate are required before encoding a strategy override. This prevents over-fitting to one-off failures.

12. **`cross_volume_partial` recovered = Activity only, no Review Hub routing.** A successfully recovered interrupted move does not appear in Review Hub — it would be confusing to see a file in "Failed to Read" when the move actually succeeded. Recovery events go to Activity only, with the message "1 interrupted move was safely recovered."

13. **Preflight is synchronous and blocking.** The preflight check runs before any batch commit and blocks until complete. It is fast (all checks are stat() calls or DB queries, no I/O to the files themselves). High-severity preflight failures abort the batch before WAL records are written.

14. **Orphan files in Review Hub are registered, never deleted.** A file in Review Hub with no DB record is an orphan (possible causes: DB write failed after move, user placed file manually). Curator registers it as `orphan` state and surfaces it in Activity. The user decides what to do. Curator never silently deletes files it doesn't recognize.

15. **Severity escalates by rate, not by count alone.** Five `protected_pdf` errors in 60 minutes escalate from low to medium. This is a systemic signal (a whole folder of DRM-protected PDFs) that deserves a different response than five isolated errors across five days.

---

## Open Questions

1. **What is the right iCloud defer window?** The pattern "iCloud eviction timeouts cluster between 2–4 AM" is hypothetical — it depends on users' iCloud sync schedules. Should Curator learn this per-user, or is a fixed heuristic (avoid staging iCloud files between midnight–6 AM) good enough?

2. **Should orphan files in Review Hub be auto-classified?** Currently, orphan files are registered and surfaced as `orphan` state without further processing. Should Curator attempt to re-classify them (match against existing groups) or leave that decision entirely to the user?

3. **What is the right BLACKBOX_SIZE?** 500 entries per worker is an estimate. On a high-throughput session (10,000 files, 4 workers), the buffer cycles in ~2 minutes. Is this sufficient for crash diagnosis? Should the buffer be time-bounded (last 5 minutes) rather than count-bounded?

4. **Should cautious mode notify the user proactively, or only on request?** The current design shows a Home card. But for a background autopilot, is a visible card the right signal, or should Curator be silent unless the user opens the app? This is a UX question that depends on user research with neurodivergent users.

5. **How does error learning interact with user corrections (R5)?** If the user manually commits a file that Curator marked as `failed_to_read`, should that override a learned pattern? If the user commits 5 files that OCR marked as failed, should Curator unlearn the OCR-skip pattern for that MIME type? The interaction between failure learning (R10) and user correction learning (R5) is not fully specified.

6. **What happens when `expected_sha256` is missing from a WAL record?** The B1 recovery algorithm relies on `expected_sha256` being stored before the delete. If an older WAL record (written before this field was added) is found on startup, the recovery algorithm cannot verify the dest file. Migration strategy needed: treat as `QUARANTINE_DEST_ALERT` conservatively, or attempt recovery without SHA verification?

7. **Should failure patterns be user-visible?** The `failure_patterns` table encodes learned strategies. A power user might want to review, edit, or reset these (e.g., "stop skipping OCR for PDFs > 5MB — my scanner improved"). Should there be a developer/advanced settings view for learned patterns, or is that too much complexity for the target user?
