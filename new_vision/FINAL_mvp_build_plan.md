# FINAL MVP Build Plan — Curator

> **Purpose:** Answers "What exactly do we build first, second, and third?" Concrete, ordered, actionable.
> **Date:** 2026-06-08
> **Scope:** MVP = Phase 1 only. Phase 2 additions listed in Section 4.

---

## Section 1: What MVP Is NOT (Explicit Deferrals)

These are intentionally excluded. Mentioning them in MVP code is a scope violation.

| Deferred Feature | Reason | Phase |
|---|---|---|
| Sparse SNF + Leiden community detection | Replaces rough FAISS grouping with principled graph clustering | Phase 2 |
| Context Birth Detection / ADWIN | Per-community drift detection (`river` library) | Phase 2 |
| Folder Coherence Score | Requires Leiden communities to be meaningful | Phase 2 |
| Safe Learning / Pattern Validation (R5 full pipeline) | Requires stable group assignments first | Phase 2 |
| Multilingual embeddings (`multilingual-e5-small`) | MVP uses `all-MiniLM-L6-v2` only | Phase 2 |
| Greek NER (`spaCy el_core_news_sm`) | MVP uses `en_core_web_sm` + custom UCY regex | Phase 2 |
| `unisim` / RETSim version detection | MVP uses SHA-256 + SimHash only | Phase 2 |
| LLM coherence verification (Islam 2026) | Research-only; no API dependency in MVP | Research only |
| ProxAnn gate (Hoyle 2025) | Research-only evaluation metric | Research only |
| IUI 2027 metrics (OBI, DCR, DQDI) | Evaluation instrumentation, not product | Research only |
| Failure Graph | Requires stable error history baseline | Phase 2 |
| Adaptive Safety Mode | Requires failure graph | Phase 2 |
| Error Learning | Requires failure graph | Phase 2 |
| Conformal Prediction / Mondrian CP | Requires ≥20 calibrated examples per context | Phase 2 |
| SetFit fine-tuning | Requires ≥16 confirmed files per context | Phase 2 |
| Context lifecycle management (7 states) | Requires Leiden-based stable communities | Phase 2 |
| LabelSpreading | Requires stable confirmed labels | Phase 2 |
| 6-edge-type context graph | Full graph requires SNF + Leiden | Phase 2 |
| Behavioral co-access tracking | Requires sustained usage baseline | Phase 2 |
| OCR (`ocrmac` / Apple Vision) | Scanned PDF OCR is Tier 2+ edge case | Phase 2 |
| `moondream2` image understanding | Tier 3 — too expensive for MVP | Phase 2 |
| `nomic-embed-text` higher-dim embeddings | MVP uses `all-MiniLM-L6-v2` (384-dim) | Phase 2 |
| Seeded re-clustering (confirmed centroid seeds) | Requires stable confirmed context set | Phase 2 |
| Timeline view in Organized section | Nice-to-have; not blocking core workflow | Phase 2 |
| Power User Mode toggle | Build after core UX is stable | Phase 2 |
| Cmd+K command palette | Build after keyboard shortcuts are stable | Phase 2 |

---

## Section 2: MVP Feature List

Every item here is concrete, specific, and buildable.

### Identity & Storage

1. **File identity:** `(inode, device_id)` primary key. SHA-256 secondary content key. Path stored but not PK. NFC-normalized path for all string storage.
2. **Exclusion/lock gate:** Prefix trie backed by `exclusions` SQLite table. Files matching the trie never enter DB. No xattrs written to locked files.
3. **FSEvents watcher + daily mtime+size delta scan:** FSEvents real-time with stored event ID for restart recovery. Daily full mtime+size scan as fallback. Weekly SHA-256 integrity pass.
4. **SQLite schema v1:** `files`, `file_states`, `processing_queue`, `staged_files`, `staged_groups`, `curator_events`, `curator_event_files`, `scan_sessions`, `exclusions`, `file_errors`, `undo_log`. Schema version table. WAL mode. STRICT.
5. **SQLCipher:** Plain SQLite in dev (env var `CURATOR_DB_ENCRYPTED=false`). AES-256 encrypted in prod. Key in macOS Keychain `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`.
6. **DB migration framework:** `schema_version` table. Apply migrations on startup. Migrations are irreversible append-only (no DROP TABLE in migrations).

### Ingestion Pipeline

7. **Processing queue:** SQLite table with `(file_id, gate, priority, status, worker_id)`. Priority = `0.6×recency + 0.3×(1/size_mb) + 0.1×access_freq`. WAL-recoverable on crash.
8. **Tier 0 reader:** `osxmetadata` + xattr. Batch `NSMetadataQuery`. Extracts `kMDItemWhereFroms`, `kMDItemCreator`, `kMDItemKind`, `kMDItemLastUsedDate`. Resolves 60–70% of files.
9. **Tier 1 reader:** `python-magic` MIME detection. Archive central directory (never extract). DOCX headings via `python-docx`.
10. **Tier 2 reader:** `markitdown` for PDF, DOCX, XLSX text extraction. No OCR in MVP — if `markitdown` cannot extract, file becomes `failed_to_read` reason `unsupported_format`.
11. **Basic NER — content:** `spaCy en_core_web_sm` for extracted text. Entities: PERSON, ORG, GPE, DATE, PRODUCT.
12. **UCY course code regex:** Always active. Pattern: `EPL\d{3}`, `ECE\d{3}`, `MCS\d{3}`, `EPL\d{3}[A-Z]?`. Applied to filename and extracted text. Stored as NER entity with type `COURSE_CODE`.
13. **File state machine transitions:** Implement all 13 states and all valid transitions from R1 §1.3, plus the C3 correction (`committed → staged_review` for Phase 2, transition exists in schema from day one).
14. **Stuck state watchdog:** `asyncio` periodic task every 5 minutes. Thresholds: `new`=5min, `identity_checked`=10min, `metadata_only`=30min, `partially_understood`=2h, `fully_understood`=1h, `grouped`=6h. Stuck files → `failed_to_read` reason `stuck_timeout`.
15. **Error containment:** Per-file `try/except` around every processing step. No exception propagates past file boundary. All errors written to `file_errors` table. Pipeline continues.

### Duplicate Gate

16. **Exact duplicate detection (full pipeline):** Partition by extension+size range → partial hash first 64KB (xxHash3) → full SHA-256 → Union-Find family grouping (path compression + union-by-rank). Exact duplicates: excluded from embedding, routed to `Review Hub/Duplicates`.
17. **Near-duplicate detection:** SimHash over extracted text tokens. Hamming distance ≤5 = near-duplicate. Near-duplicates: excluded from embedding, routed to `Review Hub/Duplicates`.
18. **No RETSim in MVP:** Version detection deferred. SimHash + SHA-256 only.

### Embeddings & Rough Grouping

19. **`all-MiniLM-L6-v2` embeddings:** Batch MPS inference on Apple Silicon. 384-dim float32. Input: filename tokens + top-N content tokens from Tier 0/1/2 extraction. Cache embeddings in `embeddings` table alongside `metadata` fingerprint.
20. **FAISS IVFFlat index:** `nlist=100`, `nprobe=10`, `k=20`. Build index when ≥100 files are embedded. Rebuild index on delta when >10% of vectors have changed.
21. **Rough group assignment:** FAISS cluster centroids only (no Leiden in MVP). Files within cosine distance 0.40 of a centroid are assigned to that group. Files with no centroid match within 0.40 → `Review Hub/Uncertain`.
22. **usearch for incremental inserts:** `usearch` (transitive via `unisim` install) handles delta inserts after initial FAISS index build. No `hnswlib`.

### Greek & Multilingual Handling

23. **Greek filename normalization:** NFC normalization on all file paths. Accent-strip index for search (store both NFC original and accent-stripped form). Greeklish transliteration index (`α→a`, `β→b`, etc.) for search.
24. **No `el_core_news_sm` in MVP:** Greek NER deferred. UCY course codes (all-ASCII) detected via regex regardless of filename language.

### Review Hub

25. **Physical Review Hub:** Created at `~/Library/Application Support/Curator/Review Hub/`. Subfolders: `Duplicates/`, `Uncertain/`, `Protected/`, `Corrupted/`, `EmptyFiles/`, `Encrypted/`, `TooLarge/`, `Unsupported/`.
26. **WAL-protected staging moves:** Same-volume (APFS): `moveItem` atomic rename. Cross-volume: copy → verify (size+SHA-256) → delete source. WAL record written with `status='pending'` before operation; updated to `'complete'` after verification. `expected_sha256` stored in WAL payload before the delete step.
27. **Finder symlink:** `~/Desktop/_Curator Review` → Review Hub physical path. Created on first launch, re-created on startup if missing.
28. **Spotlight exclusion:** `.metadata_never_index` sentinel file in Review Hub root. `mdutil -i off` on Review Hub path on first launch.
29. **`.curator_group.json` sidecar:** Written to each group subfolder at staging time. Contains group UUID, group name, member file list (by inode), staging timestamp. Never read back — write only.
30. **Finder color tags on group subfolders:** Set via `NSWorkspace.shared.setIcon` or `xattr com.apple.FinderInfo`. Color = group type: orange=duplicate, purple=uncertain, blue=new-context, green=high-confidence.
31. **Orphan detection on startup:** Scan Review Hub physical tree for files with no corresponding `staged_files` DB record. Orphaned files shown as Home problem card: "X files in Review Hub have no record — review them."
32. **Max nesting depth:** 3 levels inside Review Hub. No group subfolder may be nested deeper.
33. **Destination collision:** Rename with `_curator_N` suffix (N=1,2,3…). Never silent overwrite.

### User Review

34. **4 primary actions:** Approve, Rename, Commit, Restore. All other actions (Split, Lock, Merge) in secondary "…" menu accessible from group detail.
35. **Group card anatomy:** Name + type/count/size + 4-dot confidence + Approve/Rename buttons. Nothing else at card level.
36. **4-dot confidence scale:**
    - `●●●●` = "Curator is confident"
    - `●●●○` = "Curator is fairly confident"
    - `●●○○` = "Please review carefully"
    - `●○○○` = "Curator is uncertain — needs your input"
    - **Never percentages on group cards.**
37. **Group ordering in Review Hub:** `●○○○` uncertain first → duplicates second → high-confidence last. Within tier: largest groups first.
38. **Session cap:** 15 groups soft pause. After 15: "Great work — 9 groups are waiting for another day." Options: Commit What You Approved / Keep Going / Save and Come Back Later. No hard block.
39. **Session cap configurable:** Settings range 5–30. Default 15. Fixed in MVP at 15 with no UI to change (Settings option deferred to Phase 2 if needed).
40. **Duplicate group view:** Side-by-side for 2 files. Vertical list with ★ keeper for 3+ files. Auto-suggest keeper: most recently accessed, then better path tier, then most recently modified. Show reasoning in plain language. Never auto-delete — always `send2trash`.
41. **Group detail view:** Inline expansion (card expands). File list: icon + name + size + date, compact. "And N more" link for groups >10 files. Quick Look integration (Space bar on selected file). Similarity explanation sentence above file list.
42. **Approve separate from Commit:** Approve = grouping is correct (no file move). Commit = physical move to final destination. "Commit All Approved" button appears when ≥1 group is approved.

### Activity Log & Undo

43. **Append-only `curator_events` table:** All fields per R8 §1.3 schema. `status='pending'` WAL signal written before every file operation.
44. **`curator_event_files` table:** Per-file rows within batch operations. Enables future per-file undo without schema migration.
45. **"Why" explanation templates:** 8 reason types (semantic, ner, provenance, behavioral, duplicate, version, rule, coherence). Default: one sentence. `reason_details` JSON array for full signal breakdown in Details drawer.
46. **SQLite-backed undo stack (not NSUndoManager):** Reconstructed from `curator_events WHERE status != 'undone' ORDER BY timestamp DESC` on each launch. Survives restarts.
47. **Cmd+Z linear undo:** Pops most recent reversible event, executes inverse, records new `undo` event, updates original event `undone_at`.
48. **Group undo = one Cmd+Z step:** All files sharing a `batch_id` are one undo step. Partial undo (missing files) requires explicit user confirmation dialog.
49. **Two-tier undo window:** `staging_move` = 90 days active. `commit_move` = 180 days active. Both: indefinitely archived.
50. **`send2trash` for all duplicate trash:** `send2trash>=2.1.0`. Never `os.unlink` on user files. `NSFileManager.trashItem()` result URL stored as `destination_path` for reversibility check.
51. **Activity view:** Day + session grouped display. Filter bar: All / Staging / Commits / Auto-commits / Duplicates / Learning / Failed. FTS5 full-text search over filenames, paths, reasons.

### Crash Recovery

52. **WAL 4-case recovery on startup (B1 algorithm):** For every `status='pending'` WAL record on startup: (a) src exists, dst missing → restart copy; (b) src missing, dst exists → verify SHA-256 → mark complete or quarantine; (c) both exist → verify SHA-256 of dst → delete src and mark complete, or delete corrupt dst and restart; (d) neither exists → discard WAL, log warning. Run before accepting any new work.
53. **Crash recovery UX:** 0 recovered → silent. 1–5 recovered → transient status bar message (auto-dismiss 4s). 6+ recovered → persistent Home problem card with [Review] action.

### Home Screen

54. **Home screen elements:** Personal greeting. Primary CTA card ("N groups ready for review" or "Nothing needs attention"). Two stats: total organized files + files organized this week. Ambient scan progress bar. Problem cards appear ONLY when action is needed.
55. **Home problem cards (conditional):** Appear only when: folder permission denied and scan paused >24h; >5% of files in batch failed to read; committed group destination no longer exists; orphan files detected in Review Hub; crash recovery ≥6 operations. Language: "N decisions" not "N files need attention." No red color, no alarm framing.

### Settings

56. **Settings v1 — two options only:**
    - Allowed folders (add/remove scan roots)
    - Review Hub location (default `~/Library/Application Support/Curator/Review Hub/`, can be changed; symlink updated on change)
57. No other settings in MVP. Learning controls, power mode, session length configurability — all Phase 2.

### Search

58. **FTS5 search:** Over filenames, group names, reason text. Greek/Greeklish support: query normalized via same NFC + accent-strip + Greeklish transliteration before FTS5 match. Results ranked by recency.

### Error Memory

59. **`file_errors` table:** `(file_id, error_type, attempt_count, last_attempt_at, last_error_message)`. Upsert on each failure.
60. **Skip-after-N policy:** Skip OCR attempts after 2 failures (deferred to Phase 2 when OCR is added). Skip iCloud download after 3 timeouts (`evicted_skip` state). Skip ZIP extraction after 2 `BadZipFile` errors.

### Failure Intelligence (R10 MVP Subset)

61. **Error taxonomy classification:** 20 error classes as defined in R10. Per-file classification at the time of failure.
62. **Per-file error bubble containment:** No exception propagates past the file processing boundary. Each file's failure is isolated.
63. **Retry policy table:** Configurable in code (not UI in MVP). Transient failures: 2 retries for `extraction_timeout`, 3 for `icloud_unavailable`. Permanent failures: no retry.
64. **Failed-to-read routing:** Correct Review Hub subfolder per failure type: `Protected/` (permission denied), `Corrupted/` (bad format), `EmptyFiles/` (zero bytes), `Encrypted/` (password protected), `TooLarge/` (exceeds threshold), `Unsupported/` (no parser).
65. **Home problem card generation:** Aggregate failed file counts → problem cards on Home. Never show raw error strings. Plain language: "12 encrypted PDFs couldn't be read — [View]."
66. **Basic self-healing:** Watchdog process restart on unexpected exit (via `launchd` KeepAlive). Review Hub subfolder re-creation on startup if any subfolders are missing.
67. **Black box recorder:** Circular buffer (last 1,000 log lines in memory). Flush to `~/Library/Application Support/Curator/crash_log.txt` on crash signal (SIGTERM, SIGINT, unhandled exception).
68. **Preflight check before batch moves:** Before any `commit_move` batch: verify destination parent directory exists and is writable. If not → Home problem card, no silent failure.

---

## Section 3: Build Order

Work items are ordered by dependency. No layer can begin until its prerequisite layer is stable and passing tests.

### Layer 0 — Infrastructure (nothing else works without this)

- [ ] SQLite schema v1: all MVP tables, indexes, FTS5 virtual table
- [ ] `PRAGMA journal_mode=WAL; PRAGMA synchronous=NORMAL; PRAGMA STRICT`
- [ ] `schema_version` table + migration framework (`apply_pending_migrations()` on startup)
- [ ] SQLCipher setup: plain in dev (`CURATOR_DB_ENCRYPTED=0`), encrypted in prod; key in Keychain
- [ ] File identity utilities: `(inode, device_id)` lookup, SHA-256 hashing, mtime+size fingerprint
- [ ] NFC normalization utility: `normalize_path(p: str) -> str`
- [ ] Exclusion trie: build from `exclusions` table at startup; `is_excluded(path: str) -> bool`; gitignore-style glob fallback
- [ ] WAL intent utilities: `write_intent(event_id, src, dst, sha256) -> None`; `mark_complete(event_id) -> None`; `get_pending_intents() -> list`
- [ ] WAL 4-case crash recovery function: `recover_pending_intents() -> RecoverySummary`
- [ ] Greek string utilities: `normalize_for_storage(s: str) -> str` (NFC); `normalize_for_search(s: str) -> str` (NFC + accent-strip); `greek_to_latin(s: str) -> str` (Greeklish); `UCY_COURSE_REGEX` constant
- [ ] FTS5 virtual table + insert/update triggers on `files` and `staged_groups`
- [ ] **`thermal.py`** — `ThermalGovernor`, `ThermalState` enum, `SystemSnapshot`; polls `NSProcessInfo.thermalState` + `psutil.cpu_percent` + `psutil.sensors_battery()`; `os.nice(15)` at import; Apple Silicon caveat: `fair + cpu > 60%` → `reduced` (see R11 §3.2)
- [ ] **`passports.py`** — `OperationPassport` dataclass, `CostTier`, `ThermalFloor`, `GateDecision` enums; `PASSPORT_REGISTRY` with all 12 standard operation types; `passport_gate(passport, thermal, system) → GateDecision`; `enqueue_job()` calls `PASSPORT_REGISTRY[op_type]` — raises `UnknownOperationType` if missing (see TECH_operation_passports_compute_budget.md)
- [ ] Test fixtures: synthetic filesystem factory with 100+ files including Greek filenames, exact duplicates, near-duplicates, zero-byte files, encrypted PDFs

### Layer 1 — Ingestion

- [ ] FSEvents watcher (Swift): subscribe with stored event ID, emit file events to Python sidecar via Unix domain socket; handle `kFSEventStreamEventFlagMustScanSubDirs` with full scan fallback
- [ ] Processing queue: `processing_queue` table with priority scoring; `enqueue(file_id, gate)`, `dequeue(worker_id) -> QueueItem`, `mark_done(item_id)`, `recover_stuck_workers(timeout=600)`
- [ ] Tier 0 reader: `osxmetadata` batch read; `kMDItemWhereFroms`, `kMDItemCreator`, `kMDItemKind`, `kMDItemLastUsedDate`; write to `files.tier0_metadata` JSON column
- [ ] Tier 1 reader: `python-magic` MIME; archive central dir (zipfile, tarfile); DOCX headings; write to `files.tier1_metadata`
- [ ] Tier 2 reader: `markitdown` for PDF/DOCX/XLSX; 500-token content summary; write to `files.content_summary`
- [ ] State machine transition engine: `transition(file_id, new_state, reason=None)` — validates transition against legal graph, writes `file_states` row with `entered_at` timestamp
- [ ] UCY course code extractor: apply `UCY_COURSE_REGEX` to filename + content_summary; store as NER entity rows
- [ ] `spaCy en_core_web_sm` content NER: PERSON, ORG, GPE, DATE, PRODUCT entities from `content_summary`; store in `file_entities` table
- [ ] Stuck state watchdog: `asyncio.create_task(watchdog_loop(interval=300))` — query stuck files, transition to `failed_to_read(stuck_timeout)`, write to `file_errors`
- [ ] Error containment wrapper: `@safe_process(file_id)` decorator — catches all exceptions, writes to `file_errors`, marks file `failed_to_read`, never re-raises

### Layer 2 — Duplicate Gate

- [ ] Size bucketing: group by `(extension, size_range)` where size ranges are 0–1KB, 1KB–1MB, 1MB–100MB, 100MB+
- [ ] xxHash3 partial hash: first 64KB of file content; cached in `files.partial_hash`
- [ ] SHA-256 full hash: full file content; cached in `files.sha256`; skip if `mtime+size` unchanged since last hash
- [ ] SimHash computation: 64-bit SimHash over filename tokens + top content tokens; store in `files.simhash`
- [ ] Hamming distance check: `popcount(a XOR b) <= 5` for near-duplicate pairs; SQL query with `simhash` index
- [ ] Union-Find family grouping: path compression + union-by-rank; `families` table with `(file_id, family_id, is_canonical)`
- [ ] Duplicate family routing: `family_size >= 2` → create `staged_groups` row of type `duplicate`; move representative to `Review Hub/Duplicates/`
- [ ] Format twin detection: same stem, different extension (`.docx` + `.pdf`), same folder → mark as format_twin pair, do NOT route to duplicates

### Layer 3 — Embeddings & Rough Grouping

- [ ] **`vector_index.py`** — `VectorIndex` Protocol; `UsearchVectorIndex` (default, usearch HNSW, memory-mapped, external IDs, filtered search); `NumpyFallbackIndex` (brute-force, test environments / < 5k vectors); `get_vector_index(config)` factory from `CURATOR_VECTOR_BACKEND` env var (see R11 §4.1 and FINAL_code_impact_plan §8.3)
- [ ] `all-MiniLM-L6-v2` embedding: `sentence-transformers` library, MPS device, batch size 32; input = `filename_tokens + content_summary[:256]`; output = 384-dim float32 vector
- [ ] Embedding cache: store vector in `files.embedding` BLOB; invalidate when `content_summary` changes
- [ ] Vector index via `VectorIndex` abstraction (not FAISS directly): `vector_index.add(file_id, vector)` on embed, `vector_index.search(query, k, allowlist)` for retrieval; persist via `vector_index.save(path)` — **no new direct FAISS calls** (FAISS is the current legacy backend only; migrate to `UsearchVectorIndex`)
- [ ] `usearch` incremental insert: for delta files after initial index build; sync back to FAISS on next rebuild cycle
- [ ] Rough group assignment: `faiss.search(embedding, k=1)` → nearest centroid; if `distance <= 0.40` → assign to existing group; else → route to `Review Hub/Uncertain`
- [ ] `staged_groups` population: one row per proposed group; `group_type` ∈ {duplicate, uncertain, new_context, high_confidence}; `confidence` derived from mean centroid distance → mapped to 1–4 dot scale
- [ ] Similarity explanation generator: template-based (no LLM in MVP); pick dominant signal (provenance > NER course code > content NER > semantic) → one-sentence template filled from file metadata
- [ ] Centroid update on commit: when user commits a group, update FAISS centroid as EMA: `centroid_new = 0.9×centroid_old + 0.1×mean(committed_embeddings)`

### Layer 4 — Review Hub

- [ ] Physical folder structure: create all 8 subfolders on first launch; `ensure_hub_structure()` on every startup
- [ ] WAL-protected staging moves: same-volume `FileManager.moveItem`; cross-volume copy-verify-delete with `expected_sha256` in WAL payload
- [ ] `staged_files` table population: one row per staged file with `original_path`, `staged_path`, `wal_intent`, `group_id`
- [ ] Spotlight exclusion: write `.metadata_never_index` to hub root; `mdutil -i off <hub_path>` via subprocess
- [ ] `.curator_group.json` sidecar: write JSON on group folder creation; fields: `group_id`, `group_name`, `group_type`, `staged_at`, `member_inodes`
- [ ] Finder color tags: `NSWorkspace.shared` xattr `com.apple.FinderInfo` on group subfolders
- [ ] Orphan detection: on startup, walk Review Hub tree; for each file check `staged_files` by inode; orphans → Home problem card
- [ ] Finder symlink: `~/Desktop/_Curator Review` → hub path; create on first launch, verify on every startup, re-create if broken

### Layer 5 — Activity & Undo

- [ ] `curator_events` append-only logging: `log_event(type, file_id, src, dst, reason, reason_type, confidence, batch_id) -> event_id`; always write `status='pending'` first, then `'complete'`
- [ ] 8 reason template functions: one per reason_type; return `(reason_str, reason_details_json)`
- [ ] SQLite-backed undo stack: `get_undo_stack() -> list[EventId]` from `curator_events WHERE status NOT IN ('undone','failed') AND reversible=1 ORDER BY timestamp DESC`
- [ ] `undo(event_id) -> UndoResult`: look up event, compute inverse operation (move dst→src), execute inverse, log new `undo` event, mark original `undone_at`
- [ ] Group undo: `undo_batch(batch_id) -> UndoResult` — gather all `curator_event_files` for batch, check each file exists, confirm partial undo if missing, execute
- [ ] Undo window enforcement: nightly job marks `reversible=0` for events where `(event_type='staging_move' AND age>90d) OR (event_type='commit_move' AND age>180d)`
- [ ] `send2trash` integration: `from send2trash import send2trash` — all duplicate trash operations use this; store `resultingItemURL` as `destination_path` for trash undo check

### Layer 6 — Swift UI

- [ ] **Home:** Scan status bar (animated progress). Stats row (total organized, this week). CTA card ("N groups ready" or calm empty state). Problem cards rendered from `home_problem_cards` view query.
- [ ] **Review Hub:** Flat list with sticky non-navigable category headers (DUPLICATES, NEW CONTEXTS, REVIEW NEEDED). Group card: name, type/count/size, 4-dot confidence, Approve + Rename buttons. Session counter "X of 15". Pause screen at 15.
- [ ] **Group detail:** Inline card expansion. Compact file list (icon, name, size, date). Similarity explanation sentence. Quick Look integration (Space bar). Secondary "…" menu with Split, Lock.
- [ ] **Duplicate family:** Side-by-side (2 files) with keeper pre-selected + reasoning. Vertical list with ★ (3+ files). "Keep Both" option always available.
- [ ] **Activity:** Day + session grouped view. Session summary rows (collapsed by default). Individual event rows (expand session to see). Filter bar. FTS5 search field.
- [ ] **Settings:** Allowed folders (add/remove with folder picker). Review Hub location (change with folder picker, symlink updates on change).
- [ ] **Keyboard shortcuts:** A=Approve, R=Rename, C=Commit, Backspace=Restore, J/K=next/prev group, Cmd+Z=undo. Tooltips on hover (600ms delay).
- [ ] **Commit All Approved button:** Appears in Review Hub top-right when ≥1 group approved. Summary dialog before executing ("This will move N files to X locations"). Undo available after.
- [ ] **Language:** All copy uses "N groups ready" / "decisions to make" framing. Forbidden: "files need attention", "backlog", "unorganized".
- [ ] **Completion animation:** Group card collapses with 100ms checkmark animation on Approve. Next group slides up.

### Layer 7 — Failure Intelligence (R10 MVP Subset)

- [ ] Error taxonomy: classify every caught exception into one of the 20 R10 error classes at catch time
- [ ] `file_errors` upsert: `upsert_file_error(file_id, error_class, message, attempt_count) -> None`
- [ ] Retry policy execution: `should_retry(file_id, error_class) -> bool` — check `attempt_count` against policy table; return False for permanent classes
- [ ] Review Hub routing for failed files: `route_failed_file(file_id, error_class) -> hub_subfolder` — maps error class to subfolder constant
- [ ] `home_problem_cards` view: SQL view that aggregates counts and conditions → returns list of `ProblemCard(title, body, action_destination)` for Home screen
- [ ] Watchdog restart: `launchd` plist with `KeepAlive = true`; sidecar is a LaunchAgent
- [ ] Hub folder re-creation: `ensure_hub_structure()` on every startup, called before any staging work
- [ ] Black box recorder: `CircularLogBuffer(capacity=1000)` in Python sidecar; `atexit` + `signal.signal(SIGTERM, flush_handler)` flush to `crash_log.txt`
- [ ] Preflight check: `preflight_batch_move(group_id) -> PreflightResult` — verify destination parent writable; abort batch if failed, generate Home problem card

---

## Section 4: Phase 2 Additions

After MVP is stable and shipped, add these in roughly this order:

1. **Sparse SNF + Leiden** — replace rough FAISS centroid grouping with principled graph communities (`leidenalg`, `scipy.sparse`, custom `sparse_snf.py`)
2. **6-edge context graph** — semantic + NER + behavioral + provenance + temporal + filename edges
3. **ADWIN context birth detection** — per-community drift via `river` library
4. **Folder Coherence Score** — `0.35·MPS + 0.20·EC + 0.15·TSR + 0.20·SCD + 0.10·FTC`
5. **Safe learning full pipeline (R5)** — pattern accumulation, 7-state rule machine, L1/L2/L3 escalation, Mondrian CP
6. **`unisim`/RETSim version families** — version detection with 9-class taxonomy
7. **`multilingual-e5-small` embeddings** — replace `all-MiniLM-L6-v2`; add Greek semantic search
8. **`el_core_news_sm` Greek NER + langdetect** — Greek named entity recognition
9. **Mondrian CP per-context confidence** — calibrated per-community routing confidence (requires ≥20 confirmed files/community)
10. **Full R10 Failure Intelligence** — Failure Graph, Adaptive Safety Mode, Error Learning
11. **IUI 2027 evaluation instrumentation** — OBI, DCR, DQDI metrics collection
12. **OCR (`ocrmac` / Apple Vision)** — Tier 2 scanned PDF support
13. **`moondream2` image understanding** — Tier 3 image semantic analysis
14. **Power User Mode** — Settings toggle; reveals Tier 3 detail in all views
15. **Session length configurability** — Settings range 5–30 (fixed at 15 in MVP)
16. **Timeline view in Organized** — temporal organization view
17. **Cmd+K command palette** — universal action discovery
18. **SetFit fine-tuning** — per-context embedding fine-tuning (requires ≥16 confirmed files/context)
19. **LabelSpreading** — propagate confirmed labels to unconfirmed neighbors after each session
20. **Behavioral co-access tracking** — FSEvents co-access within 30-min session windows

---

## Section 5: Definition of MVP Done

MVP is done when **ALL** of the following are true:

1. **Scan coverage:** A folder with 1,000+ files is scanned → all files reach a state (not stuck in `new` or `identity_checked`) within 30 minutes on M-series MacBook.
2. **Duplicate detection:** Exact duplicate pairs (SHA-256 match) are detected and appear in `Review Hub/Duplicates/` before embedding runs.
3. **Failure routing:** Failed files appear in the correct Review Hub subfolder. A password-protected PDF appears in `Encrypted/`, a zero-byte file in `EmptyFiles/`. No failed file is silently dropped.
4. **Core user actions:** User can Approve, Rename, Commit, and Restore a group. Each action is reflected in the Activity log within 1 second.
5. **Activity logging completeness:** Every file operation (staging_move, commit_move, duplicate_trash, lock) has a corresponding `curator_events` row. No file is moved without a log entry.
6. **Undo correctness:** Cmd+Z undoes the last group action correctly (files return to original paths). Group undo is one Cmd+Z step. Undo works after app restart.
7. **Greek support:** Greek filenames (NFC, polytonic, Greeklish) normalize without `UnicodeDecodeError`. Greek filenames appear correctly in Activity log and Review Hub. FTS5 search finds Greek files by Greeklish query.
8. **Crash recovery:** Force-kill the sidecar mid-staging → relaunch → WAL 4-case algorithm runs → no orphan files, no data loss, no stuck records. Recovery summary shown if ≥1 operation recovered.
9. **Calm Home screen:** Home shows zero raw error strings, zero tracebacks, zero technical codes in any user-visible surface. All problem states are expressed in plain language with a specific actionable next step.
10. **No `os.unlink` on user files:** `grep -r 'os.unlink\|os.remove\|removeItem' engine/` returns zero results for any file that could be a user file (exclusion: temp files explicitly created by Curator itself).

---

*Build plan complete. Layer 0 starts tomorrow. Layer 7 completes MVP. Phase 2 starts when Section 5 is green.*
