# R1 — File State Machine & Pipeline Gates

**Curator Research Series | New Vision Pipeline**
*Written: 2026-06-08 | Model: Claude Sonnet 4.6*

---

## Executive Summary

This document specifies the minimal, complete file state machine and mandatory pipeline gates for Curator's group-first file organizer. The design synthesizes prior art from Apache NiFi, LlamaIndex, Airflow, FSEvents documentation, and APFS internals into concrete, actionable decisions. Every state is defined, every transition is justified, every gate is ordered by cost/benefit.

The core constraint: **30,000+ files must be processed without ambiguity, infinite loops, or lost files.** The core philosophy: **fail early, quarantine cleanly, never reprocess what is already settled.**

---

## 1. File State Machine Specification

### 1.1 Design Principles

Prior art from Apache NiFi's FlowFile architecture establishes the gold standard for pipeline state tracking: every item must be in exactly one state at all times, all state changes are recorded before they are executed (write-ahead log pattern), and the system must recover to a consistent state after any crash (Apache NiFi Documentation, https://nifi.apache.org/docs/nifi-docs/html/nifi-in-depth.html).

LlamaIndex's ingestion pipeline uses a simpler three-state model for documents: new, modified (hash changed), unchanged (hash same, skip). This is sufficient for RAG ingestion but insufficient for Curator, which must track physical disposition, user decisions, and failure modes (LlamaIndex Ingestion Pipeline Docs, https://developers.llamaindex.ai/python/examples/ingestion/document_management_pipeline/).

Apache Airflow's task state machine (none → scheduled → queued → running → {success, failed, skipped, upstream_failed}) demonstrates how terminal states must be explicit and non-ambiguous, and how upstream failures must propagate cleanly (Airflow Task States, https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html).

**Curator decision:** Adopt a write-ahead log approach for state transitions: record the intended new state in the DB before executing the operation. If the app crashes mid-operation, the next startup detects the inconsistency and resets to the last stable state.

---

### 1.2 Complete State Definitions

The following 13 states form the minimal complete set. Each state has a single, unambiguous meaning.

| State | Description | Terminal? |
|---|---|---|
| `new` | File discovered by scanner; inode+device_id recorded; no other processing done | No |
| `identity_checked` | Stable identity established (inode+dev+sha256 or inode+dev+mtime+size); duplicate gate not yet run | No |
| `duplicate` | Content-identical copy of another file already in DB; no further processing | Yes* |
| `version` | Near-duplicate or variant of another file (same base name, different mtime/content); flagged for version grouping | Yes* |
| `metadata_only` | File-level metadata extracted (MIME type, size, mtime, extension); content not yet read | No |
| `partially_understood` | Some content extracted (e.g., first 512 bytes, PDF title, image EXIF) but full extraction incomplete | No |
| `fully_understood` | All feasible content extracted; file ready for grouping | No |
| `grouped` | File assigned to one or more groups by the context graph; awaiting staging | No |
| `staged_review` | File is physically present in `_Curator Review Hub`; user has not yet acted on its group | No |
| `committed` | User approved group action; file moved/renamed/tagged; final disposition recorded | Yes |
| `ignored` | User explicitly dismissed this file or its group; no further processing ever | Yes |
| `failed_to_read` | File could not be read after all retry attempts; sent to Review Hub with failure reason | Yes* |
| `user_corrected` | User manually overrode a group assignment or state; system must not revert | Yes* |

*Terminal with asterisk means terminal for normal pipeline processing but can be reset by explicit user action or admin override.

**What was cut and why:**

The originally proposed `locked` state is removed from the state machine. A locked file never enters the state machine at all — it is filtered at the exclusion-list check before any DB record is created. Putting `locked` in the state machine would require the system to know about the file's existence, violating the "total ignore" requirement (see Section 4).

**Curator decision:** 13 states, not 12. `identity_checked` is added as a discrete state between `new` and `metadata_only` because identity establishment (inode + SHA-256) is a mandatory gate that can fail independently of metadata extraction. Collapsing these would make it impossible to distinguish "never hashed" from "hashed but MIME detection failed."

---

### 1.3 Valid State Transitions

The following transition graph defines all legal moves. Any transition not listed here is invalid and must raise an error that gets logged and surfaced in the Review Hub.

```
new
  → identity_checked          (identity gate passes)
  → failed_to_read            (file unreadable at identity stage: zero-byte, permission denied, evicted and unavailable)

identity_checked
  → duplicate                 (SHA-256 matches existing record)
  → version                   (same base name, similar size, different hash, or fuzzy-hash near-match)
  → metadata_only             (no duplication detected; proceed to metadata extraction)

metadata_only
  → partially_understood      (content extraction started but incomplete: large file, encrypted sections)
  → fully_understood          (complete extraction in one pass: small plain-text file, image with EXIF)
  → failed_to_read            (readability gate failure: password-protected PDF, corrupted archive)

partially_understood
  → fully_understood          (extraction completed on subsequent pass)
  → failed_to_read            (too many retries or persistent extraction error)

fully_understood
  → grouped                   (context graph assigns file to group(s))

grouped
  → staged_review             (group promoted to Review Hub)

staged_review
  → committed                 (user approves group action)
  → ignored                   (user dismisses group)
  → grouped                   (user reassigns file to different group — only valid re-entry to non-terminal)

committed
  → user_corrected            (user reverses a committed action)

ignored
  → user_corrected            (user re-engages a previously ignored file)

duplicate
  → staged_review             (system surfaces duplicate cluster to user for resolution)
  → ignored                   (user dismisses duplicate)

version
  → staged_review             (version cluster surfaced to user)
  → ignored                   (user dismisses version cluster)

failed_to_read
  → staged_review             (always: failed files always appear in Review Hub)
  → ignored                   (user dismisses permanently unreadable file)

user_corrected
  → grouped                   (user reassigns to a group)
  → ignored                   (user re-ignores)
  → committed                 (user re-commits with correction)
```

**Curator decision:** `grouped → staged_review` is one-directional except when the user explicitly reassigns. The system must never automatically move a file backwards from `staged_review` to `grouped` on a delta scan — once staged, only the user drives the next state.

---

### 1.4 What Triggers Each Transition

| Transition | Trigger |
|---|---|
| `new → identity_checked` | Identity gate worker reads inode, device_id, mtime, size, optionally SHA-256 |
| `new → failed_to_read` | `stat()` returns EACCES, ENOENT, or file size is 0 bytes |
| `identity_checked → duplicate` | SHA-256 found in content_hash index |
| `identity_checked → version` | Fuzzy-hash (ssdeep/tlsh) score > threshold OR filename similarity + size proximity |
| `identity_checked → metadata_only` | No match; MIME type detected via libmagic |
| `metadata_only → fully_understood` | Extraction worker completes full parse |
| `metadata_only → failed_to_read` | Extraction worker returns error code (see Section 3) after retry |
| `fully_understood → grouped` | Context graph run assigns cluster membership |
| `grouped → staged_review` | Group promoted to Review Hub batch |
| `staged_review → committed` | User clicks "Accept" or "Move" on group |
| `staged_review → ignored` | User clicks "Dismiss" on group |
| `committed → user_corrected` | User undoes a committed action (undo window: configurable, default 48 hours) |

---

### 1.5 Stuck State Detection

A file is "stuck" when it has been in a non-terminal state longer than a configurable timeout without any transition. This is the watchdog pattern: a background heartbeat checks for stale state records (inspired by: AWS Data Pipeline stuck state documentation, https://docs.aws.amazon.com/datapipeline/latest/DeveloperGuide/dp-check-when-run-fails.html).

**Stuck detection thresholds (configurable):**

| State | Stuck after |
|---|---|
| `new` | 5 minutes |
| `identity_checked` | 10 minutes |
| `metadata_only` | 30 minutes |
| `partially_understood` | 2 hours |
| `fully_understood` | 1 hour |
| `grouped` | 6 hours |

**On stuck detection:** transition to `failed_to_read` with reason `stuck_timeout`, log the last attempted operation, and surface in Review Hub. Do not retry automatically more than once — stuck files are likely symptomatic of a systemic issue (large file, resource contention, iCloud download pending).

**Curator decision:** Implement a `state_entered_at` timestamp on every row. The watchdog runs every 5 minutes as a SQLite query: `SELECT * FROM files WHERE state NOT IN ('committed','ignored','failed_to_read','duplicate','version','user_corrected') AND state_entered_at < datetime('now', '-X minutes')`.

---

## 2. Mandatory Pipeline Gates

### 2.1 Gate Ordering Rationale

The fail-fast literature is unambiguous: validate cheaply before processing expensively. A Spark engineering guide (https://medium.com/towards-data-engineering/fail-fast-or-quarantine-two-data-quality-patterns-every-spark-engineer-should-know-111598f31ada) distinguishes two patterns: **fail-fast** (halt everything) and **quarantine** (isolate and continue). For Curator, the quarantine pattern is correct — a corrupted file should not block 29,999 other files, but it must never be silently dropped.

Cost ordering (cheapest to most expensive):

1. Exclusion list check: O(1) hash lookup
2. State gate (already processed?): O(1) DB lookup
3. `stat()` call: O(1) syscall
4. SHA-256: O(file_size), amortized over I/O — expensive for large files
5. MIME detection: O(first 512 bytes)
6. Full content extraction: O(file_size × complexity)
7. Embedding: O(tokens × model_cost)
8. Graph and clustering: O(n²) worst case

**Curator decision:** Gates must execute strictly in cost order. Never run a downstream gate if an upstream gate rejects the file.

---

### 2.2 Gate 1: Exclusion List Check

**What it does:** Before creating any DB record, check if the file path matches any locked path or pattern.

**Implementation:** Store locked paths and glob patterns in a dedicated `exclusions` table (or a flat file for portability). At scan time, check each discovered path against the prefix tree / compiled glob set before calling `stat()`. If matched, skip entirely — no DB record, no log entry about the file's content.

**Why DB or flat file, not extended attributes:** Time Machine uses `com.apple.metadata:com_apple_backup_excludeItem` as an extended attribute (https://eclecticlight.co/2024/07/09/excluding-folders-and-files-from-time-machine-spotlight-and-icloud-drive/), but this requires writing to the file system object, which is invasive and may not survive iCloud round-trips. A separate exclusion table is non-invasive and survives file moves. Spotlight uses a privacy list stored in `~/Library/Preferences/com.apple.Spotlight.plist` — a system-managed plist. Curator's exclusion store should be a dedicated SQLite table for queryability, with an optional export to a `.curatorignore` flat file for human readability (inspired by .gitignore, https://git-scm.com/docs/gitignore).

**Pattern matching:** gitignore-style glob matching. Key rule from git's implementation: excluded directories are never recursed into, which is a critical performance optimization — if `/Users/me/Work/` is locked, Curator must not stat any file inside it (Git gitignore docs, https://git-scm.com/docs/gitignore).

**Curator decision:** Implement exclusions as a prefix trie over absolute paths plus a compiled glob list. At scan time, check prefix match first (O(path_length)), then glob (O(n patterns)). A matched file is never mentioned in any log that the user sees.

---

### 2.3 Gate 2: State Gate (Already Processed?)

**What it does:** After establishing file path, look up the (inode, device_id) pair in the DB. If the file is already in a terminal state (committed, ignored, failed_to_read, duplicate, version, user_corrected), skip all further processing.

**Why before SHA-256:** Computing SHA-256 on a file already marked `committed` wastes I/O. The state gate is a single indexed DB lookup.

**Idempotency consideration:** LlamaIndex's pipeline achieves idempotency by storing doc_id → hash mappings and only reprocessing when the hash changes (LlamaIndex docs). Curator extends this: for non-terminal states, the state gate also checks if the file's mtime+size have changed since last processing. If mtime+size are unchanged and the file is in `fully_understood` or `grouped`, skip re-extraction.

**Curator decision:** State gate uses (inode, device_id) as primary key, not path. Paths change on rename; inodes do not (within the same volume, see Section 2.4 on APFS inode stability).

---

### 2.4 Gate 3: Identity Gate

**What it does:** Establishes stable, content-based identity for the file.

**APFS inode stability:** Inodes are stable across renames and moves within the same volume (The Eclectic Light Company, https://eclecticlight.co/2019/09/04/apfs-safe-saves-inodes-and-the-volfs-file-system/). Inodes change when a file is replaced via a traditional safe-save (write temp → delete original → rename temp). This is the critical edge case: an app saves a new version of a document, the inode changes, and Curator would treat it as a new file.

**iCloud and inode stability:** Under macOS Sonoma's FileProvider system, inode numbers are preserved through eviction and re-materialization of dataless files. APFS atomically swaps extents without changing the inode (The Eclectic Light Company, https://eclecticlight.co/2023/10/25/macos-sonoma-has-changed-icloud-drive-radically/). This means iCloud-evicted files that re-download will have the same inode — Curator can safely use (inode, device_id) as a stable key for iCloud files.

**SHA-256 cost vs. benefit:**
- A benchmark from the CCNet paper (https://arxiv.org/pdf/1911.00359) reports hashing at ~600 documents/second on one CPU core for paragraph-level hashing.
- Dropbox hashes files in ~4MB blocks with SHA-256 for deduplication (Dropbox streaming sync, https://blogs.dropbox.com/tech/2014/07/streaming-file-synchronization/).
- For a 30,000 file corpus averaging 2MB per file, full SHA-256 of all files ≈ 60GB of I/O. At typical SSD read throughput (3GB/s), this takes ~20 seconds — acceptable for initial scan.
- For delta scans, SHA-256 is only computed when mtime+size differ from the cached value, making it nearly free.

**Two-phase identity:**
1. Fast check: (inode, device_id, mtime, size) — O(1) stat call.
2. Slow check: SHA-256 — only when fast check shows a change or on first scan.

**Curator decision:** Identity = (inode, device_id) as primary key. Compute SHA-256 on initial scan and when mtime+size change. Cache SHA-256 alongside mtime+size so delta scans skip rehashing unchanged files. For safe-saved files (inode changes), detect via SHA-256 match against prior inode and merge the records.

---

### 2.5 Gate 4: Duplicate / Version Gate

**What it does:** After identity, check if SHA-256 is already in the content hash index.

**Must run before embedding or clustering:** Embedding is O(tokens × model_cost). Running embedding on 5,000 duplicate screenshots and then deduplicating would be wasteful. The duplicate gate must be the first content-based check.

**Version detection:** Beyond exact duplicates, Curator needs near-duplicate detection. Tools:
- `ssdeep` (fuzzy hashing, context-triggered piecewise hashes): detects files that share 50%+ of content blocks. Used in malware analysis and document similarity (ssdeep project, https://ssdeep-project.github.io/ssdeep/).
- `tlsh` (Trend Micro Locality Sensitive Hash): produces a hash where Hamming distance approximates content similarity. Better than ssdeep for documents.
- Filename heuristics: `report_v1.pdf` and `report_v2.pdf` with close mtime are version candidates even without content similarity.

**Curator decision:** Duplicate gate = exact SHA-256 match → state `duplicate`. Version gate = tlsh distance < 50 OR (base filename match + mtime within 30 days + size within 20%) → state `version`. Both gates exit early; no embedding for duplicates or versions unless user explicitly requests it.

---

### 2.6 Gate 5: Readability Gate

**What it does:** Before full content extraction, verify the file can be read at all.

**Failure categories** (see Section 3 for full taxonomy):
- Zero-byte file → `failed_to_read` immediately, reason `zero_bytes`
- Permission denied (EACCES) → `failed_to_read`, reason `permission_denied`
- Password-protected PDF: `fitz.open()` returns `doc.needs_pass == True` → `failed_to_read`, reason `encrypted`
- Corrupted archive: zipfile.BadZipFile, etc. → `failed_to_read`, reason `corrupted`
- File too large: configurable threshold (default 500MB) → `failed_to_read`, reason `too_large`
- iCloud not downloaded: `os.stat()` returns size > 0 but `open()` triggers download — detect via `NSURLUbiquitousItemIsDownloadingKey` or presence of `.icloud` extension (pre-Sonoma) → state `partially_understood` (pending download), not `failed_to_read`

**PyMuPDF detection pattern for encrypted PDFs** (https://medium.com/@seffa.b/detecting-protected-pdf-files-with-pymupdf-7de9fc6ac8e1):
```python
doc = fitz.open(path)
if doc.needs_pass:
    # → failed_to_read, reason=encrypted
```

**Curator decision:** Readability gate runs before any content extraction worker is dispatched. Classify failure reason at detection time and store in `files.failure_reason` column. Never retry a `zero_bytes`, `encrypted`, or `corrupted` file automatically — these require user intervention.

---

## 3. Failed File Classification

### 3.1 Taxonomy of Failure Types

Prior art: AWS SQS Dead Letter Queues (https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) and general DLQ literature classify failures into two buckets — transient (retry-able) and permanent (require human intervention). Curator adopts this bifurcation.

| Failure Code | Bucket | Description | Auto-Retry? | User Action Needed |
|---|---|---|---|---|
| `zero_bytes` | Permanent | File has zero bytes | No | Review: delete or await population |
| `permission_denied` | Permanent | EACCES on open | No | Fix permissions, then re-trigger |
| `encrypted` | Permanent | Password-protected file | No | User provides password or dismisses |
| `corrupted` | Permanent | File fails format validation (bad magic bytes, incomplete structure) | No | Delete or restore from backup |
| `too_large` | Configurable | Exceeds size threshold | No (by default) | Lower threshold or mark as ignored |
| `unsupported_format` | Permanent | Binary format with no registered parser | No | User decides: ignore or add parser |
| `extraction_timeout` | Transient | Parser timed out | Yes (1×) | If retry fails → permanent |
| `icloud_unavailable` | Transient | File is evicted, download failed or timed out | Yes (3×) | If download fails → surface to user |
| `stuck_timeout` | Transient | Watchdog fired: file stuck in non-terminal state too long | Yes (1×) | If retry fails → surface to user |
| `unknown_error` | Transient | Unexpected exception caught | Yes (1×) | If retry fails → surface with traceback |

### 3.2 Retry Policy

Adapted from Dead Letter Queue pattern (https://codelit.io/blog/dead-letter-queue-patterns):

- Transient failures: retry up to N times (configurable, default: 2 for `extraction_timeout`, 3 for `icloud_unavailable`).
- After max retries exhausted: reclassify as permanent, set state to `failed_to_read`, store all attempt records.
- Permanent failures: no retry. Send directly to `failed_to_read` state.

### 3.3 What to Store on Failure

Every `failed_to_read` record must store:
```
failure_reason: TEXT (one of the codes above)
failure_detail: TEXT (exception message or system error string)
failure_at: DATETIME
attempt_count: INTEGER
last_attempt_at: DATETIME
file_size_at_failure: INTEGER
mime_type_detected: TEXT (may be null if detection itself failed)
```

### 3.4 How Failed Files Appear in Review Hub

Failed files appear in a dedicated "Could Not Read" section of the Review Hub, grouped by failure type. The user sees:
- Count per failure type (e.g., "12 encrypted PDFs", "3 corrupted ZIPs")
- File names and paths
- One-click actions: Dismiss All, Retry, or Open in Finder

**Curator decision:** Failed files are never silently dropped. Every failure is surfaced. The Review Hub's "Could Not Read" section is sorted by failure type, then by file size (largest first, as large unreadable files may be high-value items the user wants to recover).

---

## 4. Lock (Exclusion) Implementation

### 4.1 Total Ignore Semantics

A locked path must be treated as if it does not exist. The system must not:
- Compute its hash
- Record its inode in the DB
- Read its filename or size
- Generate any graph signal from its content
- Allow its content to influence grouping of unlocked files

This is the Spotlight Privacy List model: items in System Settings > Spotlight > Search Privacy are not indexed at all (Apple Spotlight documentation). Curator must implement the same guarantee.

### 4.2 Implementation

**Exclusion store:** A dedicated `exclusions` SQLite table with columns `(id, path_prefix TEXT, pattern TEXT, added_at DATETIME, note TEXT)`. This keeps exclusions in the same DB as the rest of the state, queryable and transactional. An optional export to `~/.config/curator/exclusions.curatorignore` supports human inspection.

**Why not extended attributes:** Writing `com.apple.metadata:com_apple_backup_excludeItem` to locked files would require reading their metadata first (violating the guarantee), and the attribute could be lost on iCloud round-trips or file copies.

**Prefix trie for performance:** Build a prefix trie from all `path_prefix` values at scan startup. For each discovered path, walk the trie: O(path_length). This is how git avoids recursing into excluded directories — by checking the directory node before listing its children (Git gitignore docs, https://git-scm.com/docs/gitignore).

**Glob patterns:** Compile glob patterns (e.g., `**/node_modules/**`, `**/.git/**`) into a finite automaton at startup. Check each path against the automaton after prefix trie miss.

**Cross-context leakage:** If a locked folder contains files that are semantically related to an unlocked context (e.g., a locked `~/Work/ClientProject/` contains files similar to unlocked `~/Downloads/`), the system must not know. This is guaranteed by the scan-time exclusion check: if the path is excluded, no content, no hash, no filename enters the DB. The context graph only sees unlocked files.

**Curator decision:**
- Exclusions checked at scan time, before `stat()`.
- Stored in `exclusions` table, not as extended attributes.
- Prefix trie + compiled glob list.
- No DB record created for any excluded file.
- UI: a dedicated "Locked Paths" panel where users add/remove exclusions. Changes take effect on next scan.

---

## 5. Delta Scanning Strategy

### 5.1 The Three Approaches

**Approach A: mtime + size comparison**
- For each file discovered in a scan, compare current (mtime, size) against cached (mtime, size).
- If unchanged: skip. If changed: reprocess from `identity_checked` state.
- Cost: one `stat()` syscall per file per scan. For 30,000 files: ~30,000 syscalls ≈ trivial on SSD.
- Weakness: mtime can be wrong after NFS mounts, rsync with `--times`, or manual `touch`. Dropbox engineering notes that clock skew between systems makes timestamp-only detection unreliable (Dropbox streaming sync, https://blogs.dropbox.com/tech/2014/07/streaming-file-synchronization/).

**Approach B: FSEvents delta**
- Subscribe to FSEvents stream with `sinceWhen` = last processed event ID.
- Only process paths that received `kFSEventStreamEventFlagItemCreated`, `kFSEventStreamEventFlagItemModified`, `kFSEventStreamEventFlagItemRenamed`, or `kFSEventStreamEventFlagItemRemoved` events.
- Cost: near-zero between events; process only changed files.
- Weakness 1: FSEvents are advisory, not guaranteed. The kernel documentation explicitly states events should be "treated as advisory rather than a definitive list" (Apple FSEvents Programming Guide, https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/FSEvents_ProgGuide/). When events are dropped, `kFSEventStreamEventFlagMustScanSubDirs` fires and a full scan is required.
- Weakness 2: `kFSEventStreamEventFlagItemRenamed` does NOT provide both old and new paths. You receive only the new path. Correlating the rename requires comparing DB state to current filesystem (FSEvents API docs, https://developer.apple.com/documentation/coreservices/1455361-fseventstreameventflags/kfseventstreameventflagitemrenamed).
- Weakness 3: FSEvents stream must be active when changes occur. If Curator is not running overnight and files change, the stored event ID allows catching up — but only if the fseventsd log has not been rotated.

**Approach C: SHA-256 change detection**
- Recompute SHA-256 for every file on every delta scan.
- Most reliable but expensive: 60GB I/O for 30,000 × 2MB files per scan.
- Only appropriate as a periodic full-integrity check, not as daily delta strategy.

### 5.2 Recommended Strategy: Layered

```
FSEvents (real-time) → mtime+size gate → SHA-256 (on mtime+size change only) → reprocess
         ↓
Periodic full mtime+size scan (daily, to catch FSEvents misses)
         ↓
Weekly SHA-256 integrity pass (to catch mtime-manipulated files)
```

**Rename handling:** When FSEvents fires `kFSEventStreamEventFlagItemRenamed` for a path, look up that path in the DB. If not found, look up the (inode, device_id) combination (which is stable across renames). If found by inode, update the stored path — the file state remains unchanged (it was already processed). This correctly handles moves within the same volume without reprocessing.

**State reset on change:**
- `mtime+size unchanged` → no state reset
- `mtime+size changed, SHA-256 unchanged` → no state reset (file content identical despite timestamp change; update cached mtime+size)
- `SHA-256 changed` → reset to `identity_checked`; re-run from duplicate gate forward
- `File removed` (FSEvents `ItemRemoved` or missing from scan) → set state to a soft-deleted marker; if already `committed`, log the external deletion; if `staged_review`, remove from hub

**Rename does not reset grouping:** A file renamed from `report_draft.pdf` to `report_final.pdf` has the same inode and same SHA-256. Its group assignment remains valid. Do not re-run grouping on a pure rename.

**Curator decision:**
- Primary delta: FSEvents with stored event ID for real-time.
- Fallback: full mtime+size scan daily (runs at low priority, e.g., 2am or on idle).
- Integrity: weekly SHA-256 pass of all `committed` and `ignored` files to detect external modifications.
- Rename detection: by inode lookup, not path comparison.

---

## 6. Concurrency & Queue Design

### 6.1 Queue Architecture

Inspiration: Apache Airflow's task queue (scheduled → queued → running → terminal) and `persist-queue`'s SQLite-backed durable queue (persist-queue PyPI, https://pypi.org/project/persist-queue/).

**Curator's queue:** A SQLite table `processing_queue` with columns:
```
(id INTEGER PRIMARY KEY, file_id INTEGER, gate TEXT, priority REAL,
 enqueued_at DATETIME, started_at DATETIME, worker_id TEXT, status TEXT)
```

Status: `pending`, `running`, `done`, `failed`.

On crash recovery: any row with `status='running'` and `started_at < now - timeout` is reset to `pending`. This is the NiFi WAL pattern applied to a queue.

### 6.2 Priority Scoring

The priority score for a file in the queue is computed as:

```
priority = w_recency * recency_score + w_size * (1 / file_size_mb) + w_access * access_frequency
```

Where:
- `recency_score` = 1 / (hours since last access + 1): files accessed recently are processed first
- `1 / file_size_mb`: small files are processed first (faster to complete, frees queue slots)
- `access_frequency`: from macOS `NSFileResourceType` / spotlight metadata `kMDItemLastUsedDate`

This is inspired by LRFU (Least Recently and Frequently Used) cache replacement (Redis LRU/LFU documentation, https://redis.io/blog/lfu-vs-lru-how-to-choose-the-right-cache-eviction-policy/).

**Curator decision:** Default weights: recency 0.6, size 0.3, frequency 0.1. Configurable. Files older than 2 years with no recent access get lowest priority unless the user explicitly triggers a scan.

### 6.3 Worker Count

The file processing pipeline is I/O bound (stat, read, hash) and CPU bound (extraction, embedding). Python's `concurrent.futures.ThreadPoolExecutor` is appropriate for I/O-bound stages; `ProcessPoolExecutor` for CPU-bound extraction.

Recommendations from Python concurrency literature (https://superfastpython.com/concurrency-file-io/):
- I/O bound workers (identity gate, state gate, metadata read): min(32, os.cpu_count() + 4) threads
- CPU bound workers (SHA-256, content extraction): os.cpu_count() processes
- Embedding workers: 1–2 (rate-limited by model)

**Curator decision:**
- Gate workers (exclusion, state, identity): 16 threads (I/O bound, fast)
- Extraction workers: 4 processes (CPU bound)
- Embedding: 2 threads (network/model bound; can batch)
- All worker counts configurable via Settings.

### 6.4 Queue Persistence and Crash Recovery

`persist-queue` (SQLite-backed, https://github.com/peter-wangxu/persist-queue) provides thread-safe, crash-recoverable queues. Its WAL mode (SQLite WAL journal) ensures queue state survives crashes.

On Curator startup:
1. Check `processing_queue` for rows with `status='running'` older than 10 minutes.
2. Reset those rows to `status='pending'`.
3. Resume processing from the queue.

This is the NiFi write-ahead log pattern: no file can be "in flight" across a crash without a recovery path.

**Curator decision:** Use SQLite WAL mode for the main `curator.db`. The `processing_queue` table is in the same DB, so all state changes are transactional. No separate queue library needed if SQLite WAL is used correctly.

---

## 7. Prior Art Summary

| System | Key Contribution to Curator Design |
|---|---|
| Apache NiFi FlowFile (https://nifi.apache.org/docs/nifi-docs/html/nifi-in-depth.html) | Write-ahead log for state transitions; provenance tracking; crash recovery via WAL snapshot |
| LlamaIndex Ingestion Pipeline (https://developers.llamaindex.ai/python/examples/ingestion/document_management_pipeline/) | doc_id → hash mapping for deduplication; three-state (new/modified/unchanged) as baseline |
| Apache Airflow task states (https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html) | Explicit terminal states; upstream failure propagation; scheduler-worker-state separation |
| Dropbox streaming sync (https://blogs.dropbox.com/tech/2014/07/streaming-file-synchronization/) | Block-level SHA-256 hashing; layered change detection (metadata first, hash on change) |
| FSEvents Programming Guide (https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/FSEvents_ProgGuide/) | sinceWhen for delta scanning; advisory-not-guaranteed events; snapshot + events combined strategy |
| APFS inode stability (https://eclecticlight.co/2019/09/04/apfs-safe-saves-inodes-and-the-volfs-file-system/) | Inodes stable across moves/renames but not safe-saves; (inode, device_id) as stable key |
| iCloud dataless files / Sonoma FileProvider (https://eclecticlight.co/2023/10/25/macos-sonoma-has-changed-icloud-drive-radically/) | Inode preserved through eviction; dataless files keep inode; iCloud-safe to use inode as key |
| Dead Letter Queue pattern (https://codelit.io/blog/dead-letter-queue-patterns) | Transient vs. permanent failure classification; retry limits; quarantine without pipeline halt |
| Fail-fast vs. quarantine (https://medium.com/towards-data-engineering/fail-fast-or-quarantine-two-data-quality-patterns-every-spark-engineer-should-know-111598f31ada) | Quarantine (not fail-fast) is correct for file organizers: bad file must not block others |
| PyMuPDF encrypted PDF detection (https://medium.com/@seffa.b/detecting-protected-pdf-files-with-pymupdf-7de9fc6ac8e1) | `doc.needs_pass` for pre-processing encrypted PDF detection |
| Git gitignore (https://git-scm.com/docs/gitignore) | Prefix-trie exclusion; don't recurse excluded directories; glob + negation patterns |
| Time Machine exclusion xattr (https://eclecticlight.co/2024/07/09/excluding-folders-and-files-from-time-machine-spotlight-and-icloud-drive/) | Extended attribute approach to exclusion; why Curator should NOT use xattr (invasive, iCloud-fragile) |
| persist-queue (https://pypi.org/project/persist-queue/) | SQLite WAL queue; crash recovery; thread-safe task_done semantics |
| Redis LFU/LRU (https://redis.io/blog/lfu-vs-lru-how-to-choose-the-right-cache-eviction-policy/) | LRFU scoring for priority queue; recency + frequency combined weight |

---

## 8. Design Decisions (Numbered, Actionable)

**D1.** The file state machine has exactly 13 states. The `locked` concept is implemented as a scan-time exclusion, not a state.

**D2.** All state transitions are recorded in the DB before execution (write-ahead pattern). On startup, the system scans for inconsistencies and resets in-flight states.

**D3.** File identity primary key is (inode, device_id). SHA-256 is a secondary content key. Path is stored but is not the primary key — renames do not create new DB records.

**D4.** Gates execute in strict cost order: exclusion check → state check → stat → duplicate/SHA-256 → MIME/readability → extraction. No gate is skipped except by an upstream gate's rejection.

**D5.** Exclusions are stored in the `exclusions` SQLite table and implemented as a prefix trie + compiled glob set. No extended attributes are written to locked files.

**D6.** Delta scanning is layered: FSEvents real-time → daily full mtime+size scan → weekly SHA-256 integrity pass. FSEvents event IDs are persisted so the daemon can catch up after being offline.

**D7.** A rename (same inode, new path) does not reset a file's state or group assignment. The DB record is updated with the new path.

**D8.** A safe-save (new inode, same content = same SHA-256) is detected by SHA-256 match after identity gate. The old record's inode is updated; state is not reset if SHA-256 matches.

**D9.** Failed files are classified into 10 failure codes (Section 3.1). Transient failures retry up to a configurable limit. Permanent failures go directly to `failed_to_read`.

**D10.** The processing queue is a SQLite table with WAL mode. Crashed workers are recovered by the startup watchdog. No separate queue library is required.

**D11.** Queue priority = 0.6 × recency + 0.3 × (1/size_mb) + 0.1 × access_frequency. Weights are configurable.

**D12.** Worker pool: 16 threads for I/O gates, 4 processes for CPU extraction, 2 threads for embedding. All configurable.

**D13.** The stuck-state watchdog runs every 5 minutes. Files stuck in non-terminal states beyond their threshold are transitioned to `failed_to_read` with reason `stuck_timeout` and surfaced in Review Hub.

**D14.** `icloud_unavailable` is a transient failure (not permanent) because the file may download later. Retry 3 times with exponential backoff. After 3 failures, surface to user.

**D15.** The Review Hub's "Could Not Read" section groups failures by type and sorts by file size descending. No failed file is ever silently dropped.

---

## 9. Open Questions

**OQ1. Safe-save inode churn rate:** How often do common macOS apps (Pages, Preview, BBEdit, VS Code) use safe-save vs. APFS copy-on-write? If most apps use safe-save, inode-based identity will churn constantly and SHA-256 matching will be the de-facto primary key. This changes the cost profile significantly. *Needs measurement on a real corpus.*

**OQ2. FSEvents event ID persistence across volume remounts:** If the user unmounts and remounts a volume (e.g., an external drive), the FSEvents sinceWhen event ID may be invalid. Does FSEvents fire `kFSEventStreamEventFlagHistoryDone` reliably in this case, triggering a full scan? *Needs testing.*

**OQ3. Fuzzy hash thresholds for version detection:** What tlsh or ssdeep distance threshold correctly identifies document versions without false positives (e.g., two unrelated PDFs that happen to share boilerplate)? *Needs empirical tuning on a real Downloads corpus.*

**OQ4. iCloud download on scan:** When Curator encounters a dataless iCloud file, should it trigger a download to extract content, or mark it `icloud_unavailable` and wait? Triggering downloads could use significant bandwidth without user consent. *Needs a UX policy decision.*

**OQ5. Cross-volume moves:** If a file is moved from one volume to another, it gets a new inode on the destination volume. The DB record will show the old inode as disappeared and a new file as appeared. With SHA-256, these can be correlated — but this requires a SHA-256 lookup by content rather than by identity. Is this worth implementing? *Low priority for v1; document as known limitation.*

**OQ6. APFS snapshots and Time Machine as sources of files:** If `~/Downloads` is on a volume with Time Machine enabled, `stat()` calls during scan may traverse Time Machine snapshots. Are these suppressed by the APFS snapshot layer, or could they appear as duplicate inodes? *Needs testing.*

**OQ7. Optimal SHA-256 chunking:** Should Curator hash entire files or use Dropbox-style block hashing (~4MB blocks)? Block hashing enables partial-file deduplication (two files that share a common block) but adds complexity. For a file organizer (not a backup tool), whole-file SHA-256 is likely sufficient. *Mark as future enhancement.*

**OQ8. Quarantine extended attribute and Gatekeeper:** macOS adds `com.apple.quarantine` to downloaded files. Should Curator read this attribute to inform file source (downloaded vs. local-created)? This could improve provenance signal for grouping. *No prior art found for using quarantine xattr in file organization; mark as research opportunity.*

---

*End of R1 — File State Machine & Pipeline Gates*
