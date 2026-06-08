# FINAL Code Impact & Migration Plan
## Curator — Old Architecture → New Architecture

**Prepared:** 2026-06-08
**Scope:** Python sidecar + Swift UI
**Based on:** direct read of app.py, pipeline.py, ingest.py, db.py, embeddings.py, requirements_prod.txt, and 36 Swift source files

---

## Section 1: Codebase Inventory

### Python files

| File | What it currently does | Maps to in new system | Action |
|---|---|---|---|
| `app.py` | FastAPI app with 30+ endpoints: ingest, proposals, pipeline, clusters, search, duplicates, inbox, UMAP, shadow, rules, biography, antilibrary, health, errors, extract. Two-speed ingest worker (deep_queue + _worker_loop). PID lock. Watchdog. 2014 lines. | New scan endpoint layer + Review Hub endpoints. Worker loop redesigned for adaptive scan. | REDESIGN |
| `pipeline.py` | Nightly batch: FLRS decay → FAISS kNN graph + Leiden clustering → incremental NN fallback → conformal prediction → proposal generation. Hungarian UUID matching. SNF/Louvain legacy code still present (snf import with fallback). 1580 lines. | Adaptive scan replaces batch pipeline. Rough grouping via embeddings replaces SNF+Louvain. Proposals replaced by group-level hub staging. | REDESIGN |
| `ingest.py` | Text extraction (PDF via pdfminer+PyMuPDF fallback, DOCX, plain text). Greek detection. Spotlight metadata (kMDItemUseCount, kMDItemWhereFroms). NER via spaCy en_core_web_sm. build_embed_text. 301 lines. | Same role in new system. Needs OCR via ocrmac, markitdown for additional formats, xxhash for fast dedup gate. | EXTEND |
| `db.py` | SQLite/SQLCipher schema: files, cluster_identity, ner_entities, behavioral_edges_current, proposals, undo_log, cp_calibration, file_biography. upsert_file (inode+mtime dedup). WAL mode. ~456 lines shown; many more helpers inferred from call sites in app.py. | New tables: review_groups, wal_intents, file_states, activity_log. Current proposals table replaced by group-based staging. | EXTEND |
| `embeddings.py` | all-MiniLM-L6-v2 via sentence-transformers. FAISS IndexIDMap2 + IndexFlatIP. id_map / rev_map dicts. Persist every 10 upserts + flush on shutdown. Thread-safe with _index_lock and _embed_sem(2). 290 lines. | Same role. Model stays. Index management unchanged. unisim added for near-dedup (SimHash). | EXTEND |

### Swift files

| File | What it currently does | Maps to in new system | Action |
|---|---|---|---|
| `ContentView.swift` | NavigationSplitView with SidebarView + WorkspaceView. 16 sections in CuratorSection enum: Dashboard, Inbox, Review, ScanProgress, Duplicates, Organized, Search, Extract, History, Rules, Settings, Antilibrary, FileMap, Projects, Shadow, Repair. | New 6-section sidebar: Home, Review Hub, Organized, Search, Activity, Settings. WorkspaceView routing updated. | REDESIGN |
| `Models.swift` | CuratorStore @Observable (1347 lines). CuratorSection enum, FileProposal, ProposalStatus, ConfidenceTier, InboxItem, ActivityEvent, ShadowMove, HealthIssue, ScanPhase. All scan+approval logic, FSEvents watcher launch, sidecar client calls. | CuratorStore extended with ReviewGroup, FileState, WALIntent models. Scan flow rewired to new endpoints. Proposal model replaced by group model. | REDESIGN |
| `ReviewWorkspace.swift` | Per-file proposal list with ProposalRow, FocusReviewMode, InspectorView. Approve/Skip per file. Group-by-destination option. 485 lines. | Replaced by Review Hub: group-level cards with Approve/Rename/Commit/Restore/Split actions. Physical staging in hub folder. | REDESIGN |
| `SidecarClient.swift` | URLSession wrapper for all current sidecar endpoints. | Extended with new Review Hub endpoints, scan/start, scan/status, activity, undo. | EXTEND |
| `FSEventsWatcher.swift` | FSEvents callback → /ingest/realtime. inode + mtime capture. Protected folders check. | Add iCloud stub detection (0-byte + xattr). Add safe-save detection (same SHA-256 at new path). | EXTEND |
| `ScanProgressWorkspace.swift` | Progress display for current scan (total, indexed, skipped, errors, phase). | Rewired to GET /scan/status. File state breakdown: queued / hashing / embedding / in_hub / committed / failed. | EXTEND |
| `DashboardWorkspace.swift` | Stats, entropy score, lifecycle summary, top clusters, pending proposals badge. | Home section: scan trigger, hub badge, quick stats. | EXTEND |
| `InboxWorkspace.swift` | Downloads inbox view with state buckets (new/ready/needs_review/waiting/stale). | Absorbed into Home or deprecated; Review Hub replaces per-file inbox triage. | DEPRECATE |
| `SearchWorkspace.swift` | Vector + keyword search via /search. | Stays as Search section. Wire to new FTS5-backed GET /search endpoint. | EXTEND |
| `ClusterWorkspace.swift` | Shows clusters with file lists, rename cluster. | Organized section: committed group view, rename still works. | KEEP |
| `HistoryWorkspace.swift` | Undo log entries with restore button. | Activity section: paginated event log + undo per group action. | EXTEND |
| `DuplicatesWorkspace.swift` | /duplicates endpoint, semantic + filename modes. | Duplicate gate now runs at scan time (exact SHA-256 + SimHash). Duplicates workspace shows pre-detected groups. | EXTEND |
| `SettingsView.swift` | Monitored folders, protected folders, safe mode, local only toggles. | Settings section. Add hub folder location display. | KEEP |
| `SidecarLifecycle.swift` | Sidecar process launch, port detection, health check loop. | No change needed. | KEEP |
| `CuratorSQLiteStore.swift` | Local Swift-side SQLite for undo log mirror + behavioral edges. | Extend with WAL intent table for crash recovery. | EXTEND |
| `VisionOCR.swift` | On-device Vision OCR for images before ingest. | No change. ocrmac used on Python side for PDFs. | KEEP |
| `BehavioralTracker.swift` | Co-access tracking for behavioral edges. | Keep; feeds context graph in new system. | KEEP |
| `RulesWorkspace.swift` | Rule CRUD, rule suggestions. | Keep; rule-matched files still bypass hub. | KEEP |
| `AntiLibraryWorkspace.swift` | Unread/unused files list. | DEPRECATE in new 6-section nav; accessible from Settings or Organized. | DEPRECATE |
| `FileMapWorkspace.swift` | UMAP scatter plot. | Keep as tab within Organized section or accessible via search. | KEEP |
| `ProjectsWorkspace.swift` | Co-access connected components. | Keep; displayed in Organized section. | KEEP |
| `ShadowModeWorkspace.swift` | Shadow preview of predicted moves. | DEPRECATE; Review Hub shows pending staging directly. | DEPRECATE |
| `RepairCenterWorkspace.swift` | Health issues + clean actions. | Absorb into Settings section or Home. | DEPRECATE |
| `CommandPaletteView.swift` | Cmd+K palette. | KEEP. | KEEP |
| `InspectorView.swift` | File detail panel (biography, related files, NER entities). | KEEP; shown in Review Hub inspector panel. | KEEP |
| `OnboardingView.swift` | First-run onboarding. | KEEP. | KEEP |
| `SavePanelObserver.swift` | Intercept Save panels for save-point interception. | KEEP; high-value future feature. | KEEP |

---

## Section 2: Python API Changes

### Current endpoints (from app.py) — keep / remove / replace

| Endpoint | Verdict | Notes |
|---|---|---|
| `GET /health` | KEEP | Heartbeat; also needed by SidecarLifecycle.swift |
| `GET /health/full` | KEEP | Issues list; move to Settings |
| `POST /health/clean` | KEEP | Clean orphans/stale proposals |
| `GET /errors` | KEEP | Error log |
| `POST /errors/client` | KEEP | Swift error reporting |
| `POST /ingest` | KEEP | Single-file ingest; FSEvents path |
| `POST /ingest/realtime` | KEEP | Realtime fast-path; adapt to set file state |
| `POST /ingest/batch` | KEEP | Bulk scan path |
| `POST /pipeline/run` | REPLACE | Replaced by `POST /scan/start` |
| `POST /pipeline/stop` | KEEP | Rename to `POST /scan/stop` or keep both |
| `GET /pipeline/status` | REPLACE | Replaced by `GET /scan/status` (richer file-state breakdown) |
| `GET /proposals` | REMOVE | Replaced by `GET /review_hub/groups` |
| `POST /proposals/expire` | REMOVE | No longer needed (hub staging replaces proposal queue) |
| `POST /proposals/clear-stale` | REMOVE | |
| `POST /proposals/{id}/approve` | REMOVE | Replaced by group-level commit |
| `POST /proposals/approve-batch` | REMOVE | |
| `POST /proposals/{id}/undo` | REMOVE | Replaced by `POST /review_hub/group/{id}/restore` |
| `POST /proposals/{id}/reject` | REMOVE | Groups are split/discarded, not rejected per-file |
| `POST /recalibrate` | KEEP | Internal calibration |
| `POST /session/update` | KEEP | Behavioral tracking |
| `GET /search` | KEEP | Wire to FTS5 + vector hybrid |
| `GET /history` | REPLACE | Replaced by `GET /activity` |
| `GET /biography/{file_id}` | KEEP | File timeline; shown in Inspector |
| `GET /files/{file_id}/rename-suggestion` | KEEP | |
| `GET /duplicates` | KEEP | Backend for DuplicatesWorkspace |
| `GET /projects` | KEEP | Projects workspace |
| `GET /stats` | KEEP | Dashboard stats |
| `GET /antilibrary` | KEEP | Can deprecate later |
| `GET /clusters` | KEEP | |
| `GET /clusters/lifecycle` | KEEP | |
| `GET /clusters/{uuid}/dna` | KEEP | |
| `POST /clusters/{uuid}/dna/refresh` | KEEP | |
| `PATCH /clusters/{uuid}/label` | KEEP | |
| `GET /umap` | KEEP | |
| `GET /shadow` | REMOVE | Shadow mode deprecated |
| `GET /inbox` | REMOVE | Inbox replaced by Review Hub |
| `GET /extract` | KEEP | Text extraction debug tool |
| `GET/POST/PATCH/DELETE /rules*` | KEEP | |

### New endpoints required

```
POST /scan/start           — Start adaptive scan; returns scan_id
GET  /scan/status          — Progress by file state: {queued, hashing, duplicate_gate,
                             embedding, staged, committed, failed_read, locked}

GET  /review_hub/groups    — All staged groups with member file lists, suggested label,
                             duplicate flags, error subfolders
POST /review_hub/group/{id}/approve  — Stage group (move files to hub folder)
POST /review_hub/group/{id}/rename   — Update group label
POST /review_hub/group/{id}/commit   — Commit group to final destination
POST /review_hub/group/{id}/restore  — Restore all files to original paths
POST /review_hub/group/{id}/split    — Split group into two by provided file_id list

GET  /activity             — Paginated event log: {events, total, page, per_page}
POST /undo                 — Undo last committed group action (uses WAL)

GET  /search?q=            — FTS5 + vector hybrid (replaces current LIKE fallback)
GET  /health               — Already exists; keep as heartbeat
```

---

## Section 3: Database Migration

### Current tables (from db.py _create_schema)

| Table | Purpose |
|---|---|
| `files` | Core file registry: id, path, inode, device_id, sha256, mtime, size_bytes, indexed_at, cluster_uuid, ingest_status, flrs_base, flrs_last_active, flrs_score, filename_date, source_domain |
| `cluster_identity` | Stable cluster UUIDs: uuid, current_cluster_id, label, created_at, last_matched_jaccard, top_keywords, file_types, file_count, last_active, lifecycle, accepted_moves, rejected_moves, gravity |
| `ner_entities` | Extracted NER: file_id, entity_text, entity_type, source |
| `behavioral_edges_current` | Co-access edges: file_id_a, file_id_b, co_access_count, last_session |
| `proposals` | Per-file move proposals: id, file_id, proposed_path, rationale, prediction_set, coverage, confidence_tier, status, signals, fingerprint, created_at, decided_at |
| `undo_log` | Move undo: id, proposal_id, original_path, executed_path, executed_at, undone_at |
| `cp_calibration` | Conformal prediction scores: cluster_uuid, file_id, nonconformity, added_at |
| `file_biography` | Per-file event log: id, file_id, event_type, payload, recorded_at |
| `rules` | User-defined routing rules: id, name, condition, destination, enabled, created_at, hit_count |

### Schema additions needed

```sql
-- File state machine (replaces ingest_status + proposal status combo)
ALTER TABLE files ADD COLUMN file_state TEXT NOT NULL DEFAULT 'queued'
    CHECK (file_state IN (
        'queued','hashing','duplicate_gate','embedding',
        'staged','committed','failed_read','locked','ignored'
    ));

ALTER TABLE files ADD COLUMN simhash INTEGER;        -- 64-bit SimHash for near-dedup
ALTER TABLE files ADD COLUMN content_hash TEXT;      -- xxhash for fast exact dedup
ALTER TABLE files ADD COLUMN hub_path TEXT;          -- current path in Review Hub (if staged)
ALTER TABLE files ADD COLUMN original_path TEXT;     -- path before any staging

-- Write-Ahead Log for safe move operations
CREATE TABLE IF NOT EXISTS wal_intents (
    id           TEXT PRIMARY KEY,
    file_id      TEXT NOT NULL REFERENCES files(id),
    src_path     TEXT NOT NULL,
    dst_path     TEXT NOT NULL,
    operation    TEXT NOT NULL CHECK (operation IN ('stage','commit','restore')),
    status       TEXT NOT NULL DEFAULT 'pending'
                 CHECK (status IN ('pending','complete','failed')),
    created_at   TEXT NOT NULL,
    completed_at TEXT
);
CREATE INDEX IF NOT EXISTS idx_wal_pending ON wal_intents(status) WHERE status='pending';

-- Review Hub groups (replaces proposals)
CREATE TABLE IF NOT EXISTS review_groups (
    id           TEXT PRIMARY KEY,
    label        TEXT NOT NULL,
    status       TEXT NOT NULL DEFAULT 'staged'
                 CHECK (status IN ('staged','committed','restored','split')),
    hub_dir      TEXT,           -- physical folder in Review Hub
    created_at   TEXT NOT NULL,
    committed_at TEXT,
    split_from   TEXT REFERENCES review_groups(id)
);

-- Group membership (many files per group)
CREATE TABLE IF NOT EXISTS review_group_members (
    group_id TEXT NOT NULL REFERENCES review_groups(id),
    file_id  TEXT NOT NULL REFERENCES files(id),
    PRIMARY KEY (group_id, file_id)
);

-- Activity / event log (replaces undo_log + biography for UI purposes)
CREATE TABLE IF NOT EXISTS activity_log (
    id          TEXT PRIMARY KEY,
    event_type  TEXT NOT NULL,  -- 'staged','committed','restored','renamed','split','undo'
    group_id    TEXT REFERENCES review_groups(id),
    file_id     TEXT REFERENCES files(id),
    payload     TEXT,           -- JSON
    recorded_at TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_activity_time ON activity_log(recorded_at DESC);

-- FTS5 virtual table for full-text search (replaces LIKE fallback)
CREATE VIRTUAL TABLE IF NOT EXISTS files_fts USING fts5(
    file_id UNINDEXED,
    filename,
    content_text,
    nfc_text,
    greeklish_text,
    tokenize = 'unicode61'
);
```

### Tables to keep unchanged
`cluster_identity`, `ner_entities`, `behavioral_edges_current`, `cp_calibration`, `file_biography`, `rules`

### Tables to deprecate (keep for reference during transition)
`proposals`, `undo_log` — keep read-only; new code writes to `review_groups` + `wal_intents` + `activity_log`

---

## Section 4: Swift UI Changes

### Current sections (16)
Dashboard, Inbox, Review, ScanProgress, Duplicates, Organized, Search, Extract, History, Rules, Settings, Antilibrary, FileMap, Projects, Shadow, Repair

### Target sections (6)
**Home | Review Hub | Organized | Search | Activity | Settings**

### Mapping

| Old section | New section | Change |
|---|---|---|
| Dashboard | Home | Redesign: scan trigger, hub badge, quick stats |
| Inbox | Home | Absorbed into Home quick-view |
| Review | Review Hub | Full redesign: group cards, 5 actions per group |
| ScanProgress | Home (inline) or slide-out | Rewire to GET /scan/status file-state breakdown |
| Duplicates | Review Hub (sub-view) | Duplicate groups shown as flagged cards in hub |
| Organized | Organized | Keep ClusterWorkspace; show committed groups |
| Search | Search | Wire to new FTS5 + vector search |
| Extract | Settings (debug panel) | Move to Settings; not a primary user section |
| History | Activity | Replace undo_log with activity_log; paginated |
| Rules | Settings | Move to Settings sub-section |
| Settings | Settings | Keep, add hub folder display |
| Antilibrary | Deprecated | Remove from sidebar; accessible via Organized filter |
| FileMap | Organized (tab) | Embed as tab in Organized section |
| Projects | Organized (tab) | Embed as tab in Organized section |
| Shadow | Deprecated | Remove; Review Hub covers this use case |
| Repair | Settings | Absorb into Settings health panel |

### Key Swift changes required

1. **CuratorSection enum** — reduce from 16 to 6 cases.
2. **CuratorStore** — add `reviewGroups: [ReviewGroup]`, `scanFileStates: ScanStatus`, `activityLog: [ActivityEvent]`. Remove `proposals`, `duplicateGroups`, `inboxData`, `shadowMoves`, `historyEntries` (replace with `activityLog`).
3. **ReviewHubWorkspace** (new file) — group card list, approve/rename/commit/restore/split actions, physical hub folder path display, per-file inspector panel.
4. **HomeWorkspace** (replaces DashboardWorkspace) — scan trigger button, hub pending badge, quick stats (files scanned, groups staged, committed today).
5. **SidecarClient.swift** — add 11 new endpoint methods listed in Section 2.
6. **Models.swift** — add `ReviewGroup`, `ReviewGroupMember`, `ScanStatus`, `WALIntent` structs. FileProposal model can be removed after migration.

---

## Section 5: Requirements Diff

### Current requirements_prod.txt
```
fastapi, uvicorn[standard], pydantic, sentence-transformers, faiss-cpu, numpy,
spacy, snfpy, igraph, leidenalg, python-igraph, scikit-learn, scipy, crepes,
pulearn, ruptures, pdfminer.six, python-docx, python-magic, dateparser,
rapidfuzz, sqlcipher3
```

### Remove
- `snfpy` — replaced by sparse_snf.py (custom sparse implementation already present in pipeline.py comments); SNF+Louvain path is dead code in new architecture

### Add
```
river==0.25.0          # online learning for adaptive scan priority model
ocrmac>=1.0.1          # native macOS OCR for scanned PDFs (replaces no-OCR gap)
markitdown>=0.1.6      # Office/web document → markdown text extraction
send2trash>=2.1.0      # safe delete via macOS Trash (never os.unlink in new code)
unisim==1.0.1          # SimHash near-duplicate detection (64-bit hash, Hamming distance)
moondream>=1.2.2        # local vision model for image captioning (optional, lazy-loaded)
xxhash>=3.4.0          # fast 64-bit content hash for exact duplicate gate (10x faster than SHA-256 for dedup)
```

### Keep unchanged (justified)
- `sentence-transformers` + `faiss-cpu` — embedding backbone stays
- `igraph` + `leidenalg` — rough grouping via Leiden still used in new pipeline
- `spacy` — NER still needed for cluster label inference
- `pdfminer.six` + PyMuPDF (via fitz, add explicitly) — PDF extraction chain
- `rapidfuzz` — filename similarity in duplicate detection
- `sqlcipher3` — encrypted DB stays
- `crepes` + `pulearn` + `scikit-learn` — conformal prediction and PU learning kept

### Updated requirements_prod.txt
```
fastapi
uvicorn[standard]
pydantic
sentence-transformers
faiss-cpu
numpy
spacy
igraph
leidenalg
python-igraph
scikit-learn
scipy
crepes
pulearn
ruptures
pdfminer.six
PyMuPDF
python-docx
python-magic
dateparser
rapidfuzz
sqlcipher3
river==0.25.0
ocrmac>=1.0.1
markitdown>=0.1.6
send2trash>=2.1.0
unisim==1.0.1
moondream>=1.2.2
xxhash>=3.4.0
```

---

## Section 6: Migration Strategy

### Recommendation: Branch + incremental replacement with a compatibility shim

**Rationale based on the codebase:**

The current app.py is 2014 lines with 30+ endpoints tightly coupled to the `proposals` table and per-file approval flow. The pipeline.py Louvain/SNF code (complete with sklearn compatibility patch at the top) is a known performance bottleneck that is already partially replaced (graph_cluster.py + Leiden). The Swift CuratorStore is 1347 lines with `proposals: [FileProposal]` woven into every workspace.

A full branch rewrite is appropriate here because:

1. The data model changes are breaking: `proposals` → `review_groups` + `wal_intents`. These cannot coexist cleanly with the old approval endpoints without a proxy layer.
2. The sidebar navigation changes from 16 sections to 6. This is a root-level change in ContentView and Models.swift that cascades everywhere.
3. The pipeline run model changes fundamentally: poll-based nightly batch → streaming adaptive scan with per-file state. The current `/pipeline/run` + `/pipeline/status` loop in Models.swift must be replaced entirely.

**Recommended approach:**

1. Create branch `feature/new-architecture` from main.
2. Add new endpoints (`/scan/*`, `/review_hub/*`, `/activity`, `/undo`) to app.py alongside old endpoints. Keep old endpoints alive behind a feature flag (`CURATOR_LEGACY_API=1`).
3. Add new DB tables via migration guards (ALTER TABLE IF NOT EXISTS pattern already established in db.py). Old tables remain read-only.
4. Build new Swift workspaces (HomeWorkspace, ReviewHubWorkspace, ActivityWorkspace) as new files alongside old ones.
5. Switch CuratorSection enum to 6 cases once all new workspaces pass manual testing.
6. After two-week soak on branch, remove old endpoints and deprecated Swift files.

This preserves the existing 30k-file ingested state (FAISS index + SQLite DB) and avoids a fresh scan.

---

## Section 7: Risk Assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| WAL crash leaves files in hub with no DB record (orphan) | Medium | High — user sees phantom files | Recovery scan on startup: scan hub folder, match by SHA-256, re-create WAL record or restore to original path |
| SimHash false positive marks unique file as near-duplicate | Low-Medium | Medium — user loses access to file via hub confusion | Hamming distance threshold 5 is conservative; show both files side-by-side in UI before any action; never auto-delete |
| ocrmac + markitdown add 2-3s latency per file | High | Low — scan speed degrades | Lazy extraction: run in background worker after hash+embed; do not block staging |
| FTS5 table out of sync with files table | Low | Low — search misses | Rebuild trigger on init_db(); incremental INSERT after each embed |
| FAISS index and new file_state column diverge | Medium | Medium — re-scan needed | Startup sync_faiss_with_db() already exists; extend to also sync file_state |
| CuratorSection reduction breaks any persisted @AppStorage selection | Low | Low — UX glitch on upgrade | Add migration: if stored selection not in new enum, reset to .home |
| moondream model (vision captioning) is 2GB download | Medium | Medium — first-run stall | Lazy load behind `CURATOR_VISION=1` env var; not enabled by default |
| snfpy removal breaks import in pipeline.py for users with old venv | High | Low — only affects legacy path | try/except import already present (`_SNF_AVAILABLE` flag); pipeline.py never calls SNF in new architecture |
| send2trash unavailable on some macOS CI runners | Low | Low — test failures only | Fall back to os.unlink in test environment with explicit override |
| SQLCipher key rotation during migration | Low | High — DB corruption | Never migrate key and schema simultaneously; use separate migration steps |
