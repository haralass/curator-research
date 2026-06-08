# R13 — Progressive Completeness & Thermal-Safe Full Understanding

> **Core principle: "No file left behind, no Mac burned."**
>
> R11 makes Curator thermally calm. R13 must make it complete.
>
> Every eligible file must eventually be understood. Not all at once, not at maximum speed, not at the cost of the user's experience — but all of them, eventually, without exception.

---

## The Problem This Document Solves

R11 defines the Semantic Memory Pyramid and the Thermal Governor. It correctly establishes that not every file needs the same depth of understanding at the same time. But R11 does not answer a harder question:

**How does Curator guarantee that every eligible file is eventually understood?**

Without a completeness model, Curator risks:
- Files that are scanned but never extracted because they were always behind higher-priority work
- Thermal pauses that accumulate and are never resumed
- Crash recovery that loses position in the processing queue
- "Forgotten" files that live in Layer 1 forever because deep extraction was always deferred

This document defines the architecture that closes that gap.

---

## 1. Conceptual Model

### 1.1 Three Guarantees

Curator makes three guarantees about every eligible (non-excluded, non-locked) file:

| Guarantee | What it means | Layer |
|---|---|---|
| **Inventory** | Every eligible file is recorded | Layer 1 always |
| **Minimum Understanding** | Every eligible file eventually reaches Layer 2 sketch | Layer 2, no exception |
| **Deep Understanding** | Every file whose **target tier is Layer 3** eventually reaches Layer 3 | Layer 3, when thermally safe |

"Eventually" is not "soon." It means: before the file is trashed, renamed beyond recognition, or manually archived. A file ingested today must reach Minimum Understanding within the current session or the next idle window — not within a year.

### 1.1a Target Tier Definition

**Not every file has a target tier of Layer 3.** The target tier is assigned at ingest time based on budget level (R11 §2) and file characteristics. Only files with budget ≥ Medium AND an extractable content type reach Layer 3.

**Extractable** means: Curator can produce a meaningful embedding from the file's content. A file is extractable if it has at least one of:
- Extractable text (PDF with selectable text, DOCX, TXT, code file)
- OCR-able image content (scanned PDF, PNG, JPG with text)
- Structured data with semantic headers (XLSX, CSV with meaningful column names)

**Not extractable** (Layer 2 maximum, no embedding):
- Binary files with no text content (compiled executables, `.dmg`, `.pkg`, `.zip` without manifest)
- Corrupt or empty files
- Files under the minimum size threshold (< 1KB, configurable)
- Files the user has explicitly excluded from deep indexing
- `failed_unreadable` files (permanently, after retry exhaustion)

**Target tier by budget level:**

| Budget level | Target tier | Deep Understanding queue? |
|---|---|---|
| High (active context) | Layer 3 if extractable | Yes, high priority |
| Medium (recent/uncertain) | Layer 3 if extractable | Yes, normal priority |
| Low (dormant) | Layer 2 only | No — re-evaluate if context reactivates |
| Minimal (duplicate family) | Layer 2 only | No — resolved after deduplication |
| None (locked/excluded) | Not in system | Not applicable |

**A dormant-context file is not a failed file.** Its target tier is Layer 2. When its context reactivates, its target tier is upgraded to Layer 3 and it enters the Deep Understanding queue automatically.

**An old screenshot is not necessarily extractable.** A screenshot with no detectable text (OCR returns < 20 chars) is flagged `low_content` and assigned Layer 2 maximum — it is not given an embedding, because the embedding would carry almost no information. This is not a failure; it is the correct classification.

### 1.2 Completeness Ledger

The **Completeness Ledger** is the persistent record of where every file stands against the three guarantees.

It answers:
- How many files have Inventory only?
- How many have Minimum Understanding?
- How many have Deep Understanding?
- How many have been paused due to thermal pressure?
- How many have failed extraction and are waiting for retry?

The Ledger is not a separate database. It is a view over the existing `files` table + `processing_queue` + `file_errors`. No new schema is required — the Ledger is a query, not a table.

```sql
-- Completeness Ledger query (proposed)
SELECT
  COUNT(*) FILTER (WHERE state = 'new')                          AS inventory_only,
  COUNT(*) FILTER (WHERE tier_reached >= 1)                      AS minimum_understanding,
  COUNT(*) FILTER (WHERE tier_reached >= 2)                      AS deep_understanding,
  COUNT(*) FILTER (WHERE processing_state = 'paused_thermal')    AS paused_thermal,
  COUNT(*) FILTER (WHERE processing_state = 'paused_retry')      AS paused_retry,
  COUNT(*) FILTER (WHERE state = 'failed_unreadable')            AS permanently_failed
FROM files
LEFT JOIN processing_queue USING (file_id);
```

This Ledger is surfaced in the Home view as a calm aggregate — not a list of files, not error messages, just: "X files understood, Y in progress."

### 1.3 Completion Debt

**Completion Debt** is the number of files that have not yet reached their target understanding tier, weighted by priority.

```
completion_debt = Σ (target_tier - current_tier) × priority_weight
```

Where `priority_weight` reflects:
- Active context → weight 3
- Recent download (< 7 days) → weight 2
- Dormant context → weight 0.5
- Failed files on retry hold → weight 0.1

Completion Debt is the primary metric for the Idle Completion Mode (§4). When Debt > 0, there is work to do. When Debt == 0, every eligible file has been understood to its target tier.

The user never sees "Completion Debt." They see: "All files understood" or "X files being processed."

---

## 2. Processing Queue Architecture

### 2.1 Prior Art

**SQLite as a persistent job queue** — A SQLite-backed queue can process up to 15,000 jobs/second [Source: justplainstuff/plainjob benchmarks]. For Curator, the queue is not high-throughput; it is durable and resumable. SQLite WAL mode ensures that queue state survives crashes without data loss.

**Priority + status indexing** — A composite index on `(status, priority, created_at)` enables O(log n) dequeue without full table scans. Workers grab the highest-priority pending job that has been waiting longest. [Source: Jason Gorman's SQLite Background Job System]

**Checkpoint-based recovery** — If the system crashes mid-processing, the worker can recover from the last checkpoint by finding jobs in `status = 'processing'` and restarting them. This is the standard pattern for all durable queues.

**Android Doze / iOS Background Fetch** — Mobile OSes model deferred background work as discrete, resumable tasks with energy budgets. Curator adopts the same discipline: background work is enumerated, budgeted, and resumable, not continuous.

### 2.2 Queue Schema

The existing `processing_queue` table (TECH_engineering_foundation.md) already supports this. Key columns:

```sql
CREATE TABLE processing_queue (
    queue_id        INTEGER PRIMARY KEY,
    file_id         INTEGER NOT NULL REFERENCES files(file_id),
    tier_target     INTEGER NOT NULL,      -- 0, 1, or 2
    priority        INTEGER NOT NULL,      -- derived from budget level
    status          TEXT NOT NULL,         -- pending / processing / paused_thermal / paused_retry / done / failed
    attempts        INTEGER DEFAULT 0,
    next_attempt_at INTEGER,               -- Unix timestamp, for retry backoff
    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL
);
CREATE INDEX idx_queue_dequeue ON processing_queue (status, priority DESC, created_at ASC);
```

### 2.3 Queue States

```
pending
  │
  ├─ [worker picks up] → processing
  │                          │
  │              ┌───────────┴──────────────┐
  │              │                          │
  │           success                    failure
  │              │                          │
  │             done           ┌────────────┴──────────────┐
  │                            │                           │
  │                     retryable?                  permanent failure
  │                            │                           │
  │                     paused_retry                    failed
  │
  ├─ [thermal governor signals pause] → paused_thermal
  │       [governor clears] → back to pending
```

`paused_thermal` jobs return to `pending` automatically when the Thermal Governor transitions to `Calm` or `Attentive`. No file is ever marked `failed` due to thermal pressure alone.

### 2.4 Priority Assignment

| File situation | Queue priority |
|---|---|
| Active context, new file | 100 |
| Active context, old unprocessed file | 80 |
| Recent download (< 7 days), unknown context | 60 |
| Review Hub pending decision | 90 |
| Dormant context, unprocessed | 20 |
| Retry after extraction failure | 10 |
| Idle completion sweep (dormant) | 5 |

Priority is recalculated when context activity changes. A file that was priority 20 (dormant) becomes priority 80 when the user opens a file from the same context.

---

## 3. Resume After Pause & Crash Recovery

### 3.1 Thermal Pause Resume

When the Thermal Governor signals a state change to `Reduced` or `Paused`:
1. All workers finish their current file (no mid-file interruption)
2. Workers mark their current queue entries as `paused_thermal`
3. Workers stop polling

When the Thermal Governor returns to `Calm` or `Attentive`:
1. Workers resume polling
2. `paused_thermal` entries are reset to `pending`
3. Processing resumes from highest priority

No data is lost. No file is re-processed from scratch unless the extraction itself was incomplete.

### 3.2 Crash Recovery

If the sidecar process crashes mid-extraction:
1. On restart, any `processing_queue` entry with `status = 'processing'` is considered interrupted
2. Interrupted entries are reset to `pending` with `attempts += 1`
3. If `attempts` exceeds the retry limit (configurable, default 3), the entry transitions to `failed`
4. A `file_errors` record is created with `error_type = 'extraction_crash'`

This is consistent with R10 failure intelligence — crash recovery is structural, not reactive.

### 3.3 WAL Integrity

The `processing_queue` lives in the same WAL-mode SQLite database as the `files` table. The WAL guarantees that:
- A queue entry is never committed without its corresponding `files` record
- A crash between "start extraction" and "write result" leaves the queue entry in `processing`, which is the safe state for recovery
- No partial extraction result is committed to the `files` table

---

## 4. Idle Completion Mode

### 4.1 What It Is

Idle Completion Mode is the state in which Curator uses available idle time to drain Completion Debt. It is not a user-visible mode — it is an automatic behavior governed by the Thermal Governor.

Idle Completion Mode activates when:
- `thermalState == nominal`
- Mac screen is off OR user idle for > 5 minutes (no keyboard/mouse events)
- AC power connected
- No other heavy processes detected

Idle Completion Mode deactivates immediately when:
- User interaction detected (screen on, keyboard event)
- Thermal state rises above `nominal`
- Battery-only power detected

### 4.2 Apple Silicon E-Core Architecture

On Apple Silicon, macOS assigns threads with QoS level `Background` (value 9) exclusively to Efficiency (E) cores, running at ~1 GHz — well below the P-core maximum of 3+ GHz. [Source: The Eclectic Light Company, Apple silicon scheduling analysis, 2024–2026]

This is the correct substrate for Curator's Idle Completion workers:
- `os.nice(19)` + `IOPOL_THROTTLE` → guaranteed E-core scheduling
- Background workers never pre-empt user tasks
- The Mac stays responsive even while Curator is extracting, embedding, and indexing

```python
# Applied at worker startup for all Idle Completion workers
import os
os.nice(19)

# IOPOL_THROTTLE via ctypes (macOS-specific)
import ctypes
libc = ctypes.CDLL("libSystem.B.dylib")
IOPOL_SCOPE_PROCESS = 0
IOPOL_THROTTLE = 3
libc.setiopolicy_np(IOPOL_SCOPE_PROCESS, IOPOL_SCOPE_PROCESS, IOPOL_THROTTLE)
```

### 4.3 Overnight Completion

For very deep Completion Debt (e.g., after first run on a 30k-file corpus), Curator can run an overnight completion sweep.

Pattern:
- Mac is locked/sleeping is prevented via `caffeinate -i` subprocess (keeps CPU running, allows disk I/O)
- All Idle Completion workers run at full concurrency on E-cores
- Thermal Governor runs normally — if Mac gets warm, workers throttle back
- By morning, the corpus should be fully processed to the target tier

**When `caffeinate` is appropriate:**
- First-run full corpus ingestion
- User-initiated "process everything now" action
- Never automatic without user consent

**When not to use it:**
- Normal ongoing ingestion (FSEvents-driven, not corpus-wide)
- Battery-only situations

### 4.4 Completeness SLA (Soft Target)

| File type | Time to Minimum Understanding (Layer 2) | Time to Deep Understanding (Layer 3) |
|---|---|---|
| New file in active context | ≤ 30 seconds (real-time path) | ≤ 5 minutes (next idle window) |
| New file, unknown context | ≤ 2 minutes (background path) | ≤ 1 hour (idle queue) |
| Old unprocessed file, active context | ≤ 10 minutes (priority queue) | ≤ 24 hours (idle completion) |
| Old unprocessed file, dormant context | ≤ overnight (idle completion) | ≤ overnight (idle completion) |

These are soft targets — they can be exceeded under sustained thermal pressure or on a very large initial corpus. The guarantee is eventual completion, not bounded latency.

---

## 5. Minimum Understanding Guarantee

### 5.1 What Minimum Understanding Requires

Every non-excluded, non-locked file that Curator has inventoried must eventually reach **Layer 2 sketch** (Semantic Sketch Memory). This means:
- Filename tokens (NFC, accent-stripped, Greeklish)
- Folder context
- File type classification
- First-page snippet (≤ 512 chars) OR course code extraction OR code import list

Layer 2 is intentionally cheap. Even large scanned PDFs can be partially sketched from filename + folder context alone, without OCR. A file that Curator cannot extract at all (truly unreadable) still gets a Layer 2 sketch based on available metadata.

### 5.2 The Unreadable Case

If a file has `state = 'failed_unreadable'`, Curator still guarantees:
- Layer 1 identity is complete
- Layer 2 sketch contains filename tokens + folder context + file type
- No vector is generated (Layer 3 skipped)
- The file appears in the Completeness Ledger as `permanently_failed`

The user sees: "3 files could not be read." (R10 — user-facing problem card, not technical error.)

### 5.3 Extraction Failure Progression

```
failed_extraction (attempt 1)
  → retry after 5 minutes
  → failed_extraction (attempt 2)
  → retry after 30 minutes
  → failed_extraction (attempt 3)
  → paused_retry indefinitely
  → surfaced in Review Hub / Home card if user action required
```

Backoff prevents thermal damage from repeated failing extractors. Max 3 automatic retries. After that, manual user action can trigger a retry.

---

## 6. Completeness and the Semantic Attention Budget

R11 defines the Semantic Attention Budget. R13 extends it with completeness enforcement:

| Budget Level | Completeness guarantee | Queue priority |
|---|---|---|
| High (active context) | Layer 2 within 30s, Layer 3 within 5min | 80–100 |
| Medium (recent/uncertain) | Layer 2 within 2min, Layer 3 within 1h | 40–60 |
| Low (dormant) | Layer 2 within overnight, Layer 3 within overnight | 5–20 |
| Minimal (duplicate family) | Layer 2 eventually, no Layer 3 | 5 |
| None (locked/excluded) | **Not applicable — not in Curator's inventory** | — |

Budget level determines priority. Priority determines queue order. Queue order determines when completion happens. Thermal Governor determines at what rate the queue drains.

This is a closed loop: every eligible file has a budget, every budget has a queue entry, every queue entry has a priority, every priority has a thermal-aware execution path.

---

## 7. Implementation Checklist

- [ ] `processing_queue` table with priority, status, attempts, next_attempt_at
- [ ] Composite dequeue index: `(status, priority DESC, created_at ASC)`
- [ ] Thermal Governor → worker coordinator interface (R11 §3.3)
- [ ] `paused_thermal` → `pending` transition on governor signal
- [ ] Crash recovery on startup: `processing` → `pending` with `attempts += 1`
- [ ] Retry backoff: 5min → 30min → give up, surface to user
- [ ] `os.nice(19)` + `IOPOL_THROTTLE` for all Idle Completion workers
- [ ] Completeness Ledger query (view or named query in db.py)
- [ ] Home view card: "X files understood, Y in progress" (no technical detail)
- [ ] `caffeinate` integration for user-initiated overnight sweep
- [ ] Priority recalculation when context activity changes

---

## 8. Relationship to Other Documents

| Document | Relationship |
|---|---|
| R1_file_state_machine.md | Queue status mirrors state transitions. `failed_unreadable` is a terminal state that still gets Layer 2 sketch. |
| R3_adaptive_reading_engine.md | Tier 0/1/2 extraction corresponds to Layer 1/2/3 sketch targets |
| R10_failure_intelligence_self_healing.md | Retry logic and failure surfaces. R13 defines the queue mechanics; R10 defines what the user sees. |
| R11_calm_vector_memory_thermal_indexing.md | Thermal Governor drives queue drain rate. Budget levels set queue priority. |
| TECH_engineering_foundation.md | `processing_queue` schema, WAL mode, `file_errors` table |

---

## 9. Open Questions

| # | Question | Status |
|---|---|---|
| Q1 | What is the correct idle detection threshold? 5 minutes of no input? | Open — needs UX input |
| Q2 | Should Completion Debt be surfaced to the user as a number? | Likely no — calm aggregate only |
| Q3 | After `caffeinate` overnight sweep, should Curator notify the user? | Likely yes — one calm notification |
| Q4 | How to handle files that grow over time (log files, large downloads)? | Rescan trigger on size change |
| Q5 | Is `IOPOL_THROTTLE` sufficient on Apple Silicon, or should we verify via `powermetrics`? | Open — test on M-series under load |

---

## Summary

- Every eligible file receives three guarantees: **Inventory**, **Minimum Understanding**, **Deep Understanding**.
- The **Completeness Ledger** tracks progress against all three — it is a query, not a new table.
- The **Completion Debt** metric quantifies outstanding work and drives queue priority.
- The **processing_queue** is a SQLite-backed, WAL-durable, priority-ordered job queue that survives crashes and thermal pauses.
- Apple Silicon **E-cores** (QoS Background + `os.nice(19)` + `IOPOL_THROTTLE`) are the natural substrate for Idle Completion workers — they run without disturbing the user.
- **`caffeinate`** enables overnight full-corpus completion when AC-connected and user-approved.
- The Thermal Governor from R11 is the only entity that pauses queue draining. No file is ever permanently paused due to thermal pressure alone.
- No file is left unknown. No Mac is burned.

---

## Sources

- [Apple Developer — Optimize for Apple Silicon: performance and efficiency cores](https://developer.apple.com/news/?id=vk3m204o)
- [The Eclectic Light Company — Apple silicon: cores, clusters and performance (2024)](https://eclecticlight.co/2024/02/19/apple-silicon-1-cores-clusters-and-performance/)
- [The Eclectic Light Company — Why E cores make Apple silicon fast (2026)](https://eclecticlight.co/2026/02/08/last-week-on-my-mac-why-e-cores-make-apple-silicon-fast/)
- [Apple Developer — Mach Scheduling and Thread Interfaces](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/scheduler/scheduler.html)
- [macOS caffeinate — Running Uninterrupted Long Processes](https://joaquinurruti.com/en/blog/2026/01/03/running-uninterrupted-long-processes-with-macos-caffeinate-command/)
- [Jason Gorman — A SQLite Background Job System](https://jasongorman.uk/writing/sqlite-background-job-system/)
- [plainjob — SQLite-backed job queue (15k jobs/s)](https://github.com/justplainstuff/plainjob)
- [Alex DeLorenzo — Process Scheduling on Linux and macOS (ionice/nice)](https://alexdelorenzo.dev/programming/2018/08/23/ionice)
- [How macOS runs background activities — The Eclectic Light Company](https://eclecticlight.co/2017/05/13/how-macos-runs-background-activities-1-from-within-an-app/)
