# TECH — Engineering Foundation
## Curator: Concrete Technical Spec

> **Audience:** Developer (Haralambos). This is a concrete engineering spec with decisions, not a research paper. Every section ends with **Curator decision:**. Open questions are explicitly marked.

---

## A. Database Schema — MVP vs Phase 2

### A.1 MVP Tables (must exist from day 1)

```sql
-- File identity: inode+device_id is PK, not path
CREATE TABLE files (
    file_id     TEXT PRIMARY KEY,  -- UUID
    inode       INTEGER NOT NULL,
    device_id   INTEGER NOT NULL,
    path        TEXT NOT NULL,     -- current path (NOT PK)
    sha256      TEXT,              -- computed lazily
    size_bytes  INTEGER,
    mtime       REAL,
    mime_type   TEXT,
    state       TEXT NOT NULL DEFAULT 'new',
    failure_code TEXT,
    created_at  TEXT DEFAULT (datetime('now')),
    updated_at  TEXT DEFAULT (datetime('now')),
    UNIQUE(inode, device_id)
) STRICT;

CREATE TABLE file_path_history (
    id          INTEGER PRIMARY KEY,
    file_id     TEXT NOT NULL,
    path        TEXT NOT NULL,
    valid_from  TEXT NOT NULL,
    valid_to    TEXT            -- NULL = current path
);

CREATE TABLE processing_queue (
    queue_id    TEXT PRIMARY KEY,
    file_id     TEXT NOT NULL,
    priority    REAL NOT NULL,   -- 0.6*recency + 0.3*(1/size_mb) + 0.1*access_freq
    tier        INTEGER DEFAULT 0,  -- which reading tier to attempt
    enqueued_at TEXT DEFAULT (datetime('now')),
    status      TEXT DEFAULT 'pending',  -- pending/processing/done/failed
    worker_id   TEXT
) STRICT;

CREATE TABLE staged_files (
    staged_id       TEXT PRIMARY KEY,
    file_id         TEXT NOT NULL,
    inode           INTEGER NOT NULL,
    device_id       INTEGER NOT NULL,
    original_path   TEXT NOT NULL,
    staged_path     TEXT NOT NULL,
    wal_intent      TEXT,           -- JSON: {expected_sha256, started_at, step}
    status          TEXT DEFAULT 'pending',  -- pending/complete/evicted_skip
    reason          TEXT,
    reason_type     TEXT,
    confidence      REAL,
    failure_code    TEXT,
    group_id        TEXT,
    committed_path  TEXT,
    staged_at       TEXT DEFAULT (datetime('now'))
) STRICT;

CREATE TABLE staged_groups (
    group_id        TEXT PRIMARY KEY,
    group_type      TEXT NOT NULL,  -- duplicate_exact/duplicate_near/version/new_context/mixed/failed/unknown
    group_name      TEXT NOT NULL,
    hub_path        TEXT,
    member_count    INTEGER DEFAULT 0,
    user_action     TEXT DEFAULT 'pending',  -- pending/approved/committed/restored/split/merged
    parent_group_id TEXT,
    finder_tag_color INTEGER,
    created_at      TEXT DEFAULT (datetime('now'))
) STRICT;

CREATE TABLE curator_events (
    event_id        TEXT PRIMARY KEY,
    event_type      TEXT NOT NULL,
    timestamp       TEXT NOT NULL,
    status          TEXT DEFAULT 'pending',  -- pending/complete/failed/undone
    file_id         TEXT,
    batch_id        TEXT,
    source_path     TEXT,
    destination_path TEXT,
    source_inode    INTEGER,
    dest_inode      INTEGER,
    group_id        TEXT,
    reason          TEXT,
    reason_type     TEXT,
    confidence      REAL,
    triggered_by    TEXT,  -- user_action/auto_pipeline/learning_rule
    rule_id         TEXT,
    reversible      INTEGER DEFAULT 1,
    undone_at       TEXT,
    undo_event_id   TEXT,
    app_version     TEXT,
    session_id      TEXT
) STRICT;

CREATE TABLE exclusions (
    exclusion_id    TEXT PRIMARY KEY,
    pattern         TEXT NOT NULL UNIQUE,  -- path prefix or glob
    pattern_type    TEXT NOT NULL,  -- prefix/glob
    added_by        TEXT DEFAULT 'user',
    added_at        TEXT DEFAULT (datetime('now'))
) STRICT;

CREATE TABLE file_errors (
    error_id        TEXT PRIMARY KEY,
    file_id         TEXT NOT NULL,
    error_class     TEXT NOT NULL,
    error_count     INTEGER DEFAULT 1,
    first_seen      TEXT,
    last_seen       TEXT,
    extractor       TEXT,
    session_id      TEXT,
    detail          TEXT,  -- JSON: technical details (internal only)
    resolved_at     TEXT,
    resolved_by     TEXT,
    UNIQUE(file_id, error_class)
) STRICT;

CREATE TABLE schema_version (
    version INTEGER NOT NULL,
    applied_at TEXT DEFAULT (datetime('now'))
);
INSERT INTO schema_version VALUES (1, datetime('now'));
```

### A.2 Phase 2 Tables (add when building Context Graph + Learning)

- `context_graph` — edge list: `(file_id_a, file_id_b, edge_type, weight)`
- `communities` — `(community_id, files JSON, gamma, lifecycle_state)`
- `boundary_constraints` — cannot_link/must_link pairs
- `learned_patterns` — `(rule_id, state, trigger_count, specificity)`
- `pattern_corrections` — correction accumulator per rule
- `rule_applications` — which rule moved which file
- `curator_event_summaries` — monthly compaction of `curator_events`

Do not create these tables at startup. They are added via migration when the feature branch lands.

### A.3 PRAGMA Settings (applied on every connection open)

```sql
PRAGMA journal_mode=WAL;
PRAGMA synchronous=NORMAL;
PRAGMA foreign_keys=ON;
PRAGMA busy_timeout=5000;
```

`synchronous=NORMAL` is safe with WAL: a crash can lose the last committed transaction but will never corrupt the database. `FULL` is not worth the write amplification for local personal use.

### A.4 Indexes (MVP)

```sql
CREATE INDEX idx_files_inode_device ON files(inode, device_id);
CREATE INDEX idx_files_sha256 ON files(sha256) WHERE sha256 IS NOT NULL;
CREATE INDEX idx_files_state ON files(state);
CREATE INDEX idx_files_path ON files(path);
CREATE INDEX idx_queue_status_priority ON processing_queue(status, priority DESC);
CREATE INDEX idx_staged_files_group ON staged_files(group_id);
CREATE INDEX idx_events_timestamp ON curator_events(timestamp);
CREATE INDEX idx_events_type ON curator_events(event_type);
CREATE INDEX idx_events_batch ON curator_events(batch_id) WHERE batch_id IS NOT NULL;
CREATE INDEX idx_errors_file ON file_errors(file_id);
```

### A.5 FTS5 Virtual Table

```sql
CREATE VIRTUAL TABLE fts_files USING fts5(
    file_id UNINDEXED,
    filename,
    group_name,
    reason,
    original_path,
    content='files',
    tokenize='unicode61 remove_diacritics 1'
);
```

Used for SwiftUI search bar over file names, group names, and move reasons. `content='files'` makes it a content table — FTS index must be kept in sync with `files` via triggers or explicit re-index on write.

### A.6 Schema Migration System

```python
MIGRATIONS = {
    1: [],  # initial schema — applied by create_schema()
    2: [    # Phase 2: context graph
        "CREATE TABLE context_graph (...)",
        "CREATE TABLE communities (...)",
    ],
    3: [    # Phase 2: learning
        "CREATE TABLE learned_patterns (...)",
        "CREATE TABLE pattern_corrections (...)",
        "CREATE TABLE rule_applications (...)",
    ],
}

def migrate(conn, target_version: int):
    current = conn.execute(
        "SELECT MAX(version) FROM schema_version"
    ).fetchone()[0]
    for v in range(current + 1, target_version + 1):
        backup_db(conn)  # copy .db to .db.bak.{timestamp}
        for sql in MIGRATIONS[v]:
            conn.execute(sql)
        conn.execute(
            "INSERT INTO schema_version VALUES (?, datetime('now'))", (v,)
        )
    conn.commit()
```

`backup_db` copies the `.db` file to `curator.db.bak.{ISO8601}` before executing any DDL. Never migrate in-place without a backup. Never run multiple migrations in a single transaction — each version gets its own backup + commit.

### A.7 Corruption Detection on Startup

```python
result = conn.execute("PRAGMA quick_check(100)").fetchone()
if result[0] != 'ok':
    raise DBCorruptionError(result[0])
```

Run `quick_check(100)` (checks first 100 pages) on every startup before any other operation. On `DBCorruptionError`: attempt WAL recovery (`PRAGMA wal_checkpoint(TRUNCATE)`), re-run check. If still failing: copy DB to `curator.db.corrupt.{timestamp}`, restore from most recent `.bak`, alert user via SwiftUI banner.

### A.8 Dev vs Production Encryption

Controlled by env var `CURATOR_DB_ENCRYPTED=1`.

- **Dev (default):** plain `sqlite3` — no SQLCipher, faster iteration, readable with DB Browser.
- **Production:** `pysqlcipher3` with key from macOS Keychain, `ThisDeviceOnly` attribute. Key is 32 random bytes generated on first launch, never leaves the device.

The DB layer wraps both behind a single `get_connection()` function. No application code distinguishes between dev and prod at the query level.

**Curator decision:** inode+device_id as file identity; `STRICT` tables throughout; WAL+NORMAL sync; SQLCipher behind env var; FTS5 on filename/reason/group_name; migration per version with pre-migration backup; `quick_check(100)` on every startup; Phase 2 tables deferred until feature branch.

---

## B. File Identity & macOS Filesystem Edge Cases

### B.1 Core Identity Rule

Identity is `(inode, device_id)` — never path, never filename. A file's `file_id` (UUID) is assigned once on first discovery and never changes, regardless of renames, moves, or safe-saves on the same volume.

The `path` column in `files` is the current known location. It is updated on every FSEvents rename event. It is a cache, not truth.

### B.2 Safe-Save (Atomic Save Changes Inode)

Many macOS apps (TextEdit, Pages, Xcode, VS Code) write to a temp file then `rename()` to the original path. The inode of the file at that path changes, but the content is logically the same document.

FSEvents reports this as: `kFSEventStreamEventFlagItemRemoved` for old path, `kFSEventStreamEventFlagItemCreated` for new file at same path. The new inode is different.

Detection and reconciliation:

```python
def reconcile_safe_save(old_inode: int, new_inode: int, path: str, device_id: int):
    old_record = db.get_by_inode(old_inode, device_id)
    if old_record is None:
        # No prior record — treat as new file
        db.create_file_record(new_inode, device_id, path)
        return

    new_sha256 = compute_sha256(path)

    if old_record.sha256 and old_record.sha256 == new_sha256:
        # Safe-save confirmed: same content, inode changed
        db.update_inode(old_record.file_id, new_inode)
        db.add_path_history(old_record.file_id, path, reason='safe_save')
    else:
        # Content changed or SHA not yet computed — new file
        db.mark_file_removed(old_record.file_id)
        db.create_file_record(new_inode, device_id, path)
```

Timing: SHA-256 for safe-save reconciliation is computed synchronously in the FSEvents callback thread (not deferred) because the decision must be made before the old inode record is garbage-collected. This is the one case where SHA is not lazy.

### B.3 Cross-Volume Move

APFS does not support atomic rename across volumes. A cross-volume move is always a copy+delete at the OS level. inode changes. `device_id` changes.

Detection strategy:
1. FSEvents on source volume: `ItemRemoved` for `old_inode` on `device_id_A`
2. FSEvents on destination volume: `ItemCreated` for `new_inode` on `device_id_B`
3. If SHA-256 was already computed for `old_inode`: match against `new_inode` SHA → same file, update record with new `(inode, device_id)`, add path history.
4. If SHA-256 not computed: treat conservatively as two independent events (file removed, new file created). The old record is marked `removed`. A new record is created. Re-processing is cheap.

Cross-volume reconciliation is best-effort. False positives (same content, different logical file) are rare enough to accept. False negatives (missed identity continuity) are safe — worst case is re-embedding.

### B.4 iCloud Drive Eviction

iCloud stubs (`.icloud` placeholder files) have stable inodes — CloudDocs preserves them across eviction/download cycles. However, the file content is not present on disk when evicted.

Detection before any read operation:

```python
import subprocess

def is_icloud_downloaded(path: str) -> bool:
    # Check NSURLUbiquitousItemDownloadingStatusKey via mdls
    result = subprocess.run(
        ['mdls', '-name', 'kMDItemFSContentChangeDate', path],
        capture_output=True, text=True
    )
    # Primary: check for .icloud stub extension
    if path.endswith('.icloud') or '/.icloud' in path:
        return False
    # Check size as secondary signal
    size = os.path.getsize(path)
    return size > 0
```

Preferred: use `NSURL.resourceValues(forKeys:)` from the Swift side with `URLResourceKey.ubiquitousItemDownloadingStatusKey`. If status is `.notDownloaded` or `.downloading`, do not pass to Python sidecar. Mark as `state='icloud_evicted'` in DB. Re-queue when FSEvents reports the file as downloaded (size > 0 at same path+inode).

### B.5 Finder Rename / Move (Same Volume)

FSEvents `kFSEventStreamEventFlagItemRenamed` fires with both old and new path available (via the renamed pair API). inode is unchanged.

Handler:

```python
def handle_rename(old_path: str, new_path: str, inode: int, device_id: int):
    record = db.get_by_inode(inode, device_id)
    if record:
        db.update_path(record.file_id, new_path)
        db.add_path_history(record.file_id, old_path, valid_to=datetime.utcnow().isoformat())
    else:
        # First time seeing this file
        db.create_file_record(inode, device_id, new_path)
```

If a staged file is renamed by the user while it is in Review Hub: detect via FSEvents on the Hub directory, update `staged_files.staged_path`, do not treat as a new file.

### B.6 Manual Delete

FSEvents `kFSEventStreamEventFlagItemRemoved` for a path that had a `files` record.

```python
def handle_delete(path: str, inode: int, device_id: int):
    record = db.get_by_inode(inode, device_id)
    if record is None:
        return
    staged = db.get_staged_by_file_id(record.file_id)
    if staged and staged.status == 'pending':
        # File deleted while staged — surface in Activity log
        db.mark_staged_disappeared(staged.staged_id)
        emit_event('file_disappeared_while_staged', file_id=record.file_id)
    db.mark_file_removed(record.file_id)
```

`mark_file_removed` sets `state='removed'` and stamps `updated_at`. Records are never hard-deleted from `files` — they become tombstones for audit trail.

### B.7 Hardlinks

Multiple paths can share the same `(inode, device_id)`. Curator does not attempt to track all hardlink aliases. Only the first-discovered path is recorded. If a second path with the same inode is seen, it is logged to `file_errors` with `error_class='hardlink_alias'` and skipped. Moving a hardlinked file is refused (`failure_code='hardlink_refused'`).

**Curator decision:** `(inode, device_id)` as immutable identity; safe-save reconciliation via synchronous SHA-256; cross-volume moves are best-effort via SHA match; iCloud eviction checked via stub extension + size before any read; FSEvents renamed-pair API for same-volume moves; manual deletes become tombstones; hardlinks refused from move pipeline.

---

## C. File Movement Safety

### C.1 Same-Volume Atomic Rename

```python
import os
from pathlib import Path

def move_same_volume(src: Path, dst: Path, file_id: str, db: DB):
    dst = safe_destination(dst)
    expected_sha256 = db.get_sha256(file_id)

    # WAL intent written BEFORE any filesystem operation
    db.set_wal_intent(file_id, {
        'src': str(src),
        'dst': str(dst),
        'expected_sha256': expected_sha256,
        'started_at': datetime.utcnow().isoformat(),
        'step': 'rename'
    })

    os.rename(src, dst)  # atomic on same APFS volume

    db.clear_wal_intent(file_id)
    db.update_path(file_id, str(dst))
    db.add_path_history(file_id, str(src))
```

`os.rename` on the same APFS volume is a single directory-entry update — it is atomic at the filesystem level. The WAL intent exists only to handle the window between intent-write and rename completion (crash recovery, see B1 in `R0_open_questions_resolved.md`).

### C.2 Cross-Volume Copy-Verify-Delete

```python
import shutil, hashlib

def move_cross_volume(src: Path, dst: Path, file_id: str, db: DB):
    dst = safe_destination(dst)
    dst_tmp = dst.with_suffix(dst.suffix + '.curator_tmp')
    expected_sha256 = compute_sha256(src)

    db.set_wal_intent(file_id, {
        'src': str(src),
        'dst': str(dst),
        'dst_tmp': str(dst_tmp),
        'expected_sha256': expected_sha256,
        'started_at': datetime.utcnow().isoformat(),
        'step': 'copy'
    })

    shutil.copy2(src, dst_tmp)  # preserves mtime/xattrs; APFS CoW if same container

    actual_sha256 = compute_sha256(dst_tmp)
    if actual_sha256 != expected_sha256:
        os.unlink(dst_tmp)
        db.clear_wal_intent(file_id)
        raise HashMismatchError(f"copy verification failed: {src}")

    db.update_wal_step(file_id, 'rename')
    os.rename(dst_tmp, dst)  # atomic within destination volume

    db.update_wal_step(file_id, 'delete')
    os.unlink(src)

    db.clear_wal_intent(file_id)
    db.update_path(file_id, str(dst))
    db.add_path_history(file_id, str(src))
```

The `.curator_tmp` suffix is distinctive enough to filter from FSEvents processing (add `*.curator_tmp` to the exclusion pattern list on startup).

### C.3 Destination Collision Handling

```python
def safe_destination(dst: Path) -> Path:
    if not dst.exists():
        return dst
    for n in range(1, 100):
        candidate = dst.parent / f"{dst.stem}_curator_{n}{dst.suffix}"
        if not candidate.exists():
            return candidate
    raise CollisionError(f"Too many collisions at {dst}")
```

`_curator_{n}` suffix is chosen to be: human-readable, sortable, and clearly machine-generated. If 99 collisions exist, something is wrong with grouping logic — treat as a pipeline failure, not a filesystem failure.

### C.4 WAL Recovery (4-Case Algorithm)

On startup, scan `staged_files` for records where `wal_intent IS NOT NULL`:

| `wal_intent.step` | Action |
|---|---|
| `'rename'` (same-volume) | Check if `dst` exists. If yes: clear intent, update DB path. If no: re-attempt `os.rename(src, dst)` if src still exists, else mark `failure_code='wal_src_gone'`. |
| `'copy'` (cross-volume) | `dst_tmp` may or may not exist. Delete `dst_tmp` if present. Re-run full copy from `src` if `src` still exists. |
| `'rename'` (cross-volume, post-copy) | Check if `dst` exists. If yes: delete `src`, clear intent. If no: re-run `os.rename(dst_tmp, dst)` if `dst_tmp` exists, else re-copy. |
| `'delete'` (cross-volume, post-rename) | `dst` exists, `src` may still exist. Delete `src` if present. Clear intent. |

This is the B1 resolution from `R0_open_questions_resolved.md`. Implement as `recover_wal_intents(db)` called once before the scan worker starts.

### C.5 Permissions Pre-Check

Before moving any file, verify:

```python
def can_move(src: Path, dst_parent: Path) -> tuple[bool, str]:
    if not os.access(src, os.R_OK | os.W_OK):
        return False, 'no_write_permission_src'
    if not os.access(dst_parent, os.W_OK | os.X_OK):
        return False, 'no_write_permission_dst'
    # Check not in exclusion list
    for excl in db.get_exclusions():
        if matches_exclusion(str(src), excl):
            return False, 'excluded_path'
    return True, ''
```

On failure: set `staged_files.failure_code`, emit `curator_events` with `status='failed'`, do not retry automatically.

### C.6 Extended Attributes Preservation

`shutil.copy2` copies extended attributes on macOS (including Finder tags, quarantine bits, custom metadata). For `os.rename` (same-volume), xattrs move with the inode — no action needed.

Quarantine attribute (`com.apple.quarantine`) is preserved as-is. Curator does not strip or add quarantine bits.

**Curator decision:** same-volume moves use `os.rename` (atomic); cross-volume uses copy+SHA-verify+rename-tmp+delete; `safe_destination` with `_curator_{n}` suffix for collisions; 4-case WAL recovery on startup; `shutil.copy2` for xattr preservation; `*.curator_tmp` excluded from FSEvents processing.

---

## D. Concurrency Model

### D.1 Process Architecture

```
macOS Swift App (main process)
└── Python sidecar (subprocess, launched at app start)
    ├── uvicorn (single thread, asyncio event loop)
    ├── Scan worker (asyncio Task) — FSEvents bridge → DB queue
    ├── I/O pool (ThreadPoolExecutor, max_workers=16) — stat/hash/mime
    ├── Extraction pool (ProcessPoolExecutor, max_workers=4) — markitdown/ocrmac
    ├── Embedding worker (2 threads, batched, MPS) — sentence-transformers
    ├── DB writer (single thread, serialized queue) — all SQLite writes
    └── Watchdog (asyncio periodic task, 60s interval)
```

Single Python process. All concurrency within that process. No separate worker processes except the `ProcessPoolExecutor` for extraction (isolation against native library crashes).

### D.2 Thread Responsibilities

**I/O pool (16 threads):** `os.stat`, `compute_sha256`, `mimetypes.guess_type`, reading file bytes for extraction. These are all blocking calls that release the GIL. 16 threads is enough to saturate APFS on an M-series Mac without exceeding memory pressure limits for 30k files.

**Extraction pool (4 processes):** `markitdown`, `ocrmac`, PDF parsing. Isolated in separate processes because: (a) native library crashes kill the worker, not the main process; (b) CPU-bound work benefits from true parallelism. Max 4 to leave headroom for system processes and Swift UI.

**Embedding worker (2 threads):** `sentence-transformers` with MPS backend. Batched inference: accumulate 32 embeddings before calling `model.encode()`. Two threads to overlap batch preparation with inference. Not a separate process — `sentence-transformers` manages MPS internally.

**DB writer (1 thread):** All `INSERT`, `UPDATE`, `DELETE` go through a `queue.Queue` consumed by a single thread. This eliminates `SQLITE_BUSY` errors entirely. Reads can happen on any thread via a separate read connection (WAL allows concurrent reads).

### D.3 Backpressure

```python
EMBEDDING_QUEUE_MAX = 1000
io_pause_event = asyncio.Event()
io_pause_event.set()  # not paused initially

async def embedding_monitor():
    while True:
        if embedding_queue.qsize() > EMBEDDING_QUEUE_MAX:
            io_pause_event.clear()  # pause I/O workers
        elif embedding_queue.qsize() < EMBEDDING_QUEUE_MAX // 2:
            io_pause_event.set()   # resume
        await asyncio.sleep(5)
```

I/O workers check `await io_pause_event.wait()` before dequeuing. This prevents unbounded memory growth when extraction is fast but embedding is slow (e.g., first scan of 30k files).

### D.4 Graceful Shutdown

```python
shutdown_event = asyncio.Event()

async def shutdown():
    shutdown_event.set()
    # Allow queues to drain (max 10s)
    await asyncio.wait_for(drain_queues(), timeout=10.0)
    io_executor.shutdown(wait=False, cancel_futures=True)
    extraction_executor.shutdown(wait=False, cancel_futures=True)
    db_writer.stop()
```

Swift sends `SIGTERM` to the sidecar on app quit. The sidecar's SIGTERM handler calls `shutdown()`. Any in-flight file moves complete before the process exits (WAL intent recovery handles any that do not).

### D.5 Crash Recovery for Extraction Workers

```python
def submit_extraction(file_id: str, path: str):
    attempt = db.get_extraction_attempt_count(file_id)
    if attempt >= 3:
        db.set_state(file_id, 'extraction_failed')
        return
    future = extraction_executor.submit(extract_text, path)
    future.add_done_callback(
        lambda f: handle_extraction_result(f, file_id, attempt + 1)
    )

def handle_extraction_result(future, file_id, attempt):
    try:
        result = future.result()
        db.save_extraction_result(file_id, result)
    except Exception as e:
        db.increment_extraction_attempt(file_id)
        db.log_error(file_id, error_class='extraction_crash', detail=str(e))
        if attempt < 3:
            submit_extraction(file_id, ...)  # retry
```

`ProcessPoolExecutor` automatically replaces a crashed worker process. The file is re-queued up to 3 times. After 3 failures: `state='extraction_failed'`, `failure_code` set, surfaced in Activity log.

### D.6 Heartbeat

```sql
CREATE TABLE sidecar_heartbeat (
    id          INTEGER PRIMARY KEY CHECK (id = 1),  -- singleton row
    updated_at  TEXT NOT NULL,
    pid         INTEGER NOT NULL,
    version     TEXT NOT NULL
);
```

Sidecar writes `updated_at = datetime('now')` every 30 seconds from the watchdog task. Swift polls this table every 60 seconds. If `updated_at` is more than 90 seconds stale: restart the sidecar subprocess, emit a `sidecar_restart` event.

**Curator decision:** single Python process with asyncio + ThreadPoolExecutor(16) + ProcessPoolExecutor(4) + 2 embedding threads + 1 DB writer thread; backpressure at embedding queue > 1000; graceful SIGTERM drain with 10s timeout; extraction workers retry up to 3 times with process isolation; heartbeat via singleton DB row, checked by Swift every 60s.

---

## E. Security & Local Data

### E.1 Data Locations

| Data | Location | Encrypted | Notes |
|---|---|---|---|
| Main DB | `~/Library/Application Support/Curator/curator.db` | SQLCipher (prod) | Key in Keychain, `ThisDeviceOnly` |
| FAISS index | Same dir, `embeddings.faiss` | No | Binary, opaque to filesystem |
| Embedding metadata | Same dir, `embeddings_meta.db` | No | `file_id → FAISS index` mapping |
| Review Hub | `~/Library/Application Support/Curator/Review Hub/` | No | Physical files, user-accessible |
| Finder symlink | `~/Desktop/_Curator Review` | — | Symlink to Review Hub |
| `.curator_group.json` | Inside each group folder | No | Name, type, count — no file content |
| Activity export | User-chosen path | No | User-initiated only |
| Diagnostic log | `~/Library/Logs/Curator/curator.log` | No | Dev diagnostics, rotated weekly |
| Keychain entry | macOS Keychain | System | SQLCipher key, `ThisDeviceOnly` attribute |

### E.2 What Is and Is Not Stored

**Stored:**
- Embeddings (384-dim float32 vectors)
- NER entity lists (list of strings, e.g., `["Alice", "Project Alpha"]`)
- Summaries (max 500 chars, generated by local model only)
- File metadata (path, size, mtime, mime_type, sha256)
- Move reasons (human-readable, no file content)

**Not stored:**
- Raw extracted text from any file
- File content in any form
- OCR output beyond the summary/entities
- Any data transmitted off-device (Curator has no network calls)

### E.3 Keychain Integration

```python
import subprocess

KEYCHAIN_SERVICE = "com.curator.db"
KEYCHAIN_ACCOUNT = "sqlcipher_key"

def get_or_create_db_key() -> bytes:
    # Try to retrieve existing key
    result = subprocess.run(
        ['security', 'find-generic-password',
         '-s', KEYCHAIN_SERVICE, '-a', KEYCHAIN_ACCOUNT, '-w'],
        capture_output=True, text=True
    )
    if result.returncode == 0:
        return bytes.fromhex(result.stdout.strip())

    # Generate new key
    key = os.urandom(32)
    subprocess.run([
        'security', 'add-generic-password',
        '-s', KEYCHAIN_SERVICE,
        '-a', KEYCHAIN_ACCOUNT,
        '-w', key.hex(),
        '-T', ''  # no trusted apps beyond this process
    ], check=True)
    return key
```

The `ThisDeviceOnly` attribute is set via the Swift side on first launch using `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`. Python reads the key via the `security` CLI (which inherits the app's entitlements). The key is never written to disk, environment variables, or logs.

### E.4 Sandbox Entitlements Required

- `com.apple.security.files.user-selected.read-write` — user-granted folder access
- `com.apple.security.files.downloads.read-write` — Downloads folder
- `com.apple.security.temporary-exception.files.absolute-path.read-write` — `~/Library/Application Support/Curator/` (sidecar data dir)
- `com.apple.security.network.client` — not requested (Curator has no network calls)

Hardened runtime is enabled. Library validation is enabled. Sidecar is a helper tool bundled in `Contents/Helpers/`, not a framework, so it runs under the app's sandbox.

### E.5 `.curator_group.json` Sidecar File

Each staged group folder contains one `.curator_group.json`:

```json
{
  "group_id": "uuid",
  "group_type": "duplicate_exact",
  "group_name": "Project Alpha Reports",
  "member_count": 4,
  "created_at": "2026-06-08T14:23:00Z",
  "curator_version": "0.1.0"
}
```

This file contains no extracted text, no file content, no personal data beyond what Finder already shows (file count, name). It exists so that if the DB is lost, Curator can partially reconstruct group state from the filesystem.

**Curator decision:** extracted text is never persisted; only embeddings + NER entities + 500-char summaries stored; SQLCipher key in Keychain with `ThisDeviceOnly`; no network entitlements requested; `.curator_group.json` carries only structural metadata; sidecar runs as bundled helper under app sandbox.

---

## Design Decisions

1. **inode+device_id as identity, UUID as stable handle.** Path is a mutable cache column. This is the only correct approach for a filesystem agent on macOS — paths change constantly (Finder moves, safe-saves, iCloud sync).

2. **Single DB writer thread.** Eliminates `SQLITE_BUSY` without retry loops. Adds at most ~1ms latency per write (queue serialization). At 30k files, peak write rate is well within single-thread throughput.

3. **Extracted text is never stored.** Privacy by design. Embeddings are mathematically irreversible for practical purposes. This is also a storage decision — storing extracted text from 30k files would be gigabytes; embeddings are megabytes.

4. **WAL intent as crash journal.** SQLite WAL is for DB consistency; our WAL intent is for filesystem consistency. These are different layers. The DB can be consistent while a file move is half-done. The intent record bridges that gap.

5. **ProcessPoolExecutor for extraction, not threads.** `markitdown` and `ocrmac` call native libraries that can segfault. A crash in a ThreadPoolExecutor kills the sidecar. A crash in a ProcessPoolExecutor kills only that worker process, which is automatically replaced.

6. **Backpressure at embedding queue, not at scan.** Pausing the I/O pool (not the FSEvents listener) means filesystem events are still received and queued to DB, but no new hashing/extraction work is started. This prevents missing events while also preventing memory exhaustion.

7. **Phase 2 tables deferred until feature branch.** Adding unused tables at MVP bloats the migration history and tempts incomplete implementations. Context graph and learning tables are added when the code that writes to them is ready.

8. **`STRICT` tables throughout.** SQLite's `STRICT` mode (3.37+) enforces column type constraints. Without it, inserting a string into an INTEGER column silently succeeds. For a data-integrity-critical application, this is a safety net worth the minor syntax overhead.

9. **`safe_destination` with `_curator_{n}` suffix.** Not `(1)`, not `_copy`, not a timestamp. The suffix is clearly machine-generated, stable across runs, and human-readable in Finder. 99 collisions → pipeline failure, not silent overwrite.

10. **Heartbeat via DB singleton row, checked by Swift.** The sidecar cannot notify Swift that it has crashed (it's crashed). The pull model (Swift polls DB) is more reliable than the push model (sidecar sends IPC). 90-second stale threshold gives enough headroom for a slow M-series wake-from-sleep.

---

## Open Questions

**OQ-1: FTS5 sync strategy.** `content='files'` requires keeping the FTS index in sync with the `files` table. Options: (a) triggers on `files` INSERT/UPDATE/DELETE; (b) explicit `fts_files.insert()` call in the DB writer. Triggers are invisible and can cause subtle issues with `STRICT` tables. Explicit calls in the DB writer are more predictable. **Unresolved:** which approach, and how to handle the initial FTS population for 30k existing records (background job vs. blocking startup).

**OQ-2: FAISS index persistence under concurrent reads.** The embedding worker writes to FAISS periodically. SwiftUI queries FAISS for similarity search. FAISS is not thread-safe for concurrent read+write. Options: (a) read lock around all FAISS operations; (b) write a new index file atomically and `mmap` the new version; (c) use a separate FAISS query thread. **Unresolved:** chosen approach and whether `embeddings_meta.db` read connection is safe to share.

**OQ-3: Hardlink detection reliability.** The current approach (skip if inode seen twice) works for 2-link hardlinks. Files with 3+ hardlinks could create confusing state if the second and third are both discovered before the first is processed. **Unresolved:** whether `nlink > 1` check on `stat` result is sufficient to pre-screen all hardlinks before ingestion, rather than detecting them at collision time.

**OQ-4: iCloud eviction re-queue trigger.** FSEvents does not reliably fire `ItemCreated` when an iCloud file is downloaded after being evicted — it may fire `ItemModified` or nothing. **Unresolved:** whether a periodic `stat` poll on `state='icloud_evicted'` files is needed alongside FSEvents, and at what interval.

**OQ-5: `processing_queue` priority formula calibration.** Current formula `0.6*recency + 0.3*(1/size_mb) + 0.1*access_freq` is a guess. `access_freq` requires tracking `kMDItemLastUsedDate` from Spotlight metadata, which adds a dependency. **Unresolved:** whether access_freq is worth the complexity for MVP, or whether `0.7*recency + 0.3*(1/size_mb)` is sufficient until learning data is available.
