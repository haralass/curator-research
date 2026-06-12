# R0 — Curator Master Synthesis

> **Single source of truth for what we're building and in what order.**  
> Audience: developer. Every claim cites its source R-file. No padding.

---

## 0. What Curator Is

Curator is a local macOS filesystem autopilot for neurodivergent users. It does not organize files — it organizes decisions. Groups of files are proposed, the user approves or corrects at group granularity, and Curator commits physical moves. No cloud, no LLM dependency for core function, no auto-delete.

**Core constraints (non-negotiable):**
- All data local. SQLite at `~/Library/Application Support/Curator/curator.db` (SQLCipher, AES-256, key in macOS Keychain). (R6)
- Never `removeItem`. Always `NSFileManager.trashItem`. (R7)
- File identity = `(inode, device_id)`. Path is not the primary key. (R1)
- All state transitions write a WAL record (`status='pending'`) before execution; mark `'complete'` after. (R1, R6, R8)
- Review Hub physically at `~/Library/Application Support/Curator/Review Hub/`; Finder symlink at `~/Desktop/_Curator Review`. (R6)

---

## 1. Full Pipeline — Annotated

```
INPUT: File event (FSEvents / daily mtime scan / weekly SHA-256 pass)
│
├─[EXCLUSION CHECK]─────────────────────────────────────────────────
│  prefix trie + compiled glob list → excluded files never reach DB  (R1)
│
├─[ADAPTIVE READING ENGINE]──────────────────────────────────────────
│  Tier 0: osxmetadata + xattr (batch NSMetadataQuery) — resolves 60-70%   (R3)
│  Tier 1: python-magic MIME, archive central dir, DOCX headings            (R3)
│  Tier 2: markitdown / ocrmac / tree-sitter  — ~10% of files reach here    (R3)
│  Tier 3: all-MiniLM-L6-v2 → nomic-embed-text, spaCy NER, moondream2      (R3)
│  Concurrency: 16 I/O threads, 4 extraction processes, 2 embedding threads (R1)
│
├─[DUPLICATE / VERSION GATE]────────────────────────────────────────
│  Partition by extension → size bucket → inode dedup                        (R2)
│  → partial hash (xxHash) → full SHA-256                                    (R2)
│  → SimHash Hamming ≤5 → dHash Hamming ≤10 (images) / pHash ≤8 confirm    (R2)
│  → RETSim (unisim) cosine: ≥0.90 = near-dup; 0.72-0.89 = version          (R2)
│  → Union-Find family grouping (path compression + union-by-rank)           (R2)
│  EXACT/NEAR-DUPS: excluded from embedding; staged to Review Hub/Duplicates (R2)
│  VERSION MEMBERS: embedded (unlike exact dups)                             (R2)
│  Precision ≥ 0.90 target; recall ≥ 0.70 acceptable                        (R9)
│
├─[CONTEXT GRAPH]───────────────────────────────────────────────────
│  FAISS IVFFlat (nlist=100, nprobe=10, k=20) for batch kNN                 (R4)
│  usearch for incremental delta inserts (transitive via unisim)        (R4)
│  Sparse SNF: K=20, 10 iterations, scipy.sparse.csr_matrix                 (R4)
│  (Full SNF ruled out: 3.6 GB/matrix at 30k files)                         (R4)
│  6 edge types overlaid (see §6)                                            (R4)
│  Leiden (leidenalg), NOT Louvain (25% disconnected community risk)         (R4)
│  Multi-resolution sweep γ=[0.5, 0.75, 1.0, 1.25, 1.5]                    (R4)
│  Min community size: 3 files                                               (R4)
│  Cannot-link: negative weight −1.0 in igraph before Leiden                 (R4)
│  igraph C backend ~0.3s for 30k nodes; NOT NetworkX (45s)                 (R4)
│
├─[CONTEXT BIRTH DETECTION]────────────────────────────────────────
│  ADWIN per community (delta=0.002, river library)                          (R4)
│  θ_birth=0.55, θ_coherence=0.10, gap_weeks=8, min_batch=5                 (R4)
│  ≥3 votes → NEW_CONTEXT event                                              (R4)
│
├─[LLM PRE-FILTER]──────────────────────────────────────────────────
│  Coherence Verification (Islam 2026) — incoherent groups auto-refined      (R9)
│  ProxAnn (Hoyle 2025) — groups with held-out accuracy < 0.70 flagged      (R9)
│
├─[REVIEW HUB STAGING]──────────────────────────────────────────────
│  WAL → physical move to Review Hub (APFS atomic rename if same volume)    (R6)
│  Cross-volume: copy → verify (size+hash) → delete original                 (R6, R8)
│  Finder symlink at ~/Desktop/_Curator Review                               (R6)
│  Max nesting depth: 3 levels                                               (R6)
│  Spotlight excluded: .metadata_never_index + mdutil -i off                 (R6)
│  iCloud files: NSFileCoordinator required                                  (R6)
│
├─[USER REVIEW]─────────────────────────────────────────────────────
│  4 primary actions: Approve, Rename, Commit, Restore                       (R7)
│  Session cap: 15 groups soft pause (configurable 5-30)                     (R7, R9)
│  Group card: 4 pieces of info (name, type+count+size, 4-dot confidence,   (R7)
│              primary actions) — never numeric percentages                  (R7)
│  Ordering: uncertain (●○○○) first, duplicates, new contexts, confident last(R7)
│
├─[COMMIT]──────────────────────────────────────────────────────────
│  Group-atomic commit with per-file rollback                                (R6)
│  Every event logged to curator_events (append-only, SQLCipher)            (R8)
│  Undo window: 90d (staging_move); 180d (commit_move); indefinitely archived (R6, R8)
│
├─[LEARNING]────────────────────────────────────────────────────────
│  7-state rule machine: observed→candidate→suggested_rule→confirmed_rule    (R5)
│                         →auto_rule→suspended→rejected (+discarded)         (R5)
│  N=3 for approve/rename/merge; N=5 for split/mark-wrong                   (R5)
│  3 automation levels: L1 (suggest only, default), L2 (auto+notify),       (R5)
│                        L3 (silent auto, ≤5 files)                          (R5)
│  - L1/L2/L3 are internal classification labels only — not exposed as labeled levels in the UI. (C4)
│
└─[FEEDBACK LOOP]───────────────────────────────────────────────────
   LabelSpreading(alpha=0.2) after each review session                       (R5)
   SetFit fine-tuning: ≥16 confirmed + ≥1 correction in 30 days             (R5)
   Mondrian CP: min 20 calibration examples per context; α=0.15              (R5)
```

---

## 2. File State Machine — 13 States

| State | Trigger to enter | Trigger to leave | Notes |
|---|---|---|---|
| `new` | File first seen by scanner | Reading engine starts | Priority = 0.6×recency + 0.3×(1/size_mb) + 0.1×access_freq (R1) |
| `identity_checked` | Tier 0 metadata resolved | Dup gate or Tier 1 | (inode, device_id) recorded (R1) |
| `duplicate` | SHA-256 / SimHash match | Manual keep decision | Excluded from embedding (R2) |
| `version` | RETSim 0.72–0.89 | User assigns version role | 9-class taxonomy (R2) |
| `metadata_only` | Only Tier 0 data available | Tier 1 attempted | 60-70% of files stop here (R3) |
| `partially_understood` | Tier 1 complete | Tier 2 triggers | ~30% of files |
| `fully_understood` | Tier 2/3 complete | Context graph assignment | ~10% reach Tier 2+ (R3) |
| `grouped` | Leiden community assigned | Staged for review | Min 3 files/community (R4) |
| `staged_review` | Physical move to Review Hub | User action | `wal_intent` set on `staged_files` row (R6) |
| `committed` | User commits group | → `user_corrected` \| → `staged_review` (Phase 2) | Physical move to final path; stable, re-evaluable (R6) |
| `ignored` | User dismisses file/group from Review Hub | `ignored → user_corrected` | In DB with `ignored` state; distinct from `locked` (never in DB) |
| `failed_to_read` | Any unrecoverable read error | Manual retry | 10 failure codes; quarantine, not fail-fast (R1) |
| `user_corrected` | User splits/renames/moves | Re-enters pipeline | Learning signal: weight 2.0–3.0 (R5) |

`locked` is NOT a state — exclusion check happens before DB record is created. (R1)

**Stuck watchdog:** runs every 5 minutes; thresholds: `new`=5 min, `grouped`=6h. (R1)

---

## 3. Data Model — Core Tables

```
curator.db (SQLCipher AES-256)
│
├── files                  (inode, device_id PK; sha256 secondary; path is NOT PK)
├── file_states            (file_id FK, state, entered_at, failure_code, wal_intent)
├── processing_queue       (SQLite WAL mode; priority scored; crash-recoverable)
│
├── staged_files           (UUID PK, inode, device_id, sha256, original_path,
│                           staged_path, wal_intent, file_state, reason,
│                           confidence, failure_code, group_id, committed_path)
├── staged_groups          (UUID PK, group_type, group_name, hub_path,
│                           member_count, user_action, parent_group_id,
│                           finder_tag_color)
│
├── context_graph          (edges: file_id_a, file_id_b, edge_type, weight)
├── communities            (community_id, files, gamma, lifecycle_state)
├── boundary_constraints   (file_id_a, file_id_b, type: cannot_link|must_link)
│
├── learned_patterns       (rule_id, specificity 0-5, state, trigger_count,
│                           confidence, domain_scope)
├── pattern_corrections    (correction_id, rule_id, action_type, weight,
│                           session_position_weight)
├── rule_applications      (application_id, rule_id, file_id, timestamp, undone)
│
├── curator_events         (event_id UUID, event_type, timestamp ISO8601+TZ,
│                           status: pending|complete|failed|undone,
│                           file_id, source_path, destination_path,
│                           source_inode, dest_inode, group_id, batch_id,
│                           reason, reason_type, confidence, triggered_by,
│                           rule_id, reversible, irreversible_reason,
│                           undone_at, undo_event_id, redo_event_id,
│                           app_version, session_id)
├── curator_event_files    (batch_id FK, file_id, source_path, dest_path,
│                           source_inode, dest_inode, status, error_message)
├── scan_sessions          (session_id, scan_type, started_at, completed_at,
│                           root_path, files_scanned, files_staged, status)
├── curator_event_summaries (compacted monthly after 1 year)
│
├── exclusions             (prefix trie + compiled glob list; no xattrs written)
└── undo_log               (append-only; soft-archived after 90 days, never deleted)
```

PRAGMA: `journal_mode=WAL`, `synchronous=NORMAL`, `STRICT`. (R8)

---

## 4. Implementation Layers

### Layer 0 — File Identity & Delta Scanning
- Identity: `(inode, device_id)` primary key; SHA-256 secondary content key. (R1)
- FSEvents real-time (stored event ID) → daily mtime+size scan → weekly SHA-256 pass. (R1)
- iCloud: `NSFileCoordinator` required; dataless files keep inode (stable key). (R6)
- Exclusion check: prefix trie + compiled glob; `exclusions` SQLite table; no xattrs. (R1)
- WAL crash recovery on startup: query all `wal_intent IS NOT NULL` / `status='pending'`. (R1, R8)

### Layer 1 — Adaptive Reading Engine
- Tier 0: `osxmetadata` + xattr; batch `NSMetadataQuery`. Highest-value free signals: `kMDItemWhereFroms`, `kMDItemCreator`. (R3)
- Tier 1: `python-magic` MIME; archive central directory (never extract); DOCX headings.
- Tier 2: `markitdown` (speed), `ocrmac` for scanned PDFs (207ms/page, Apple Vision M3 Max), `tree-sitter` (code).
- Tier 3: `nomic-embed-text` 384-dim ~20ms/doc batched MPS; `spaCy` NER; `moondream2` 4-bit (1-3s, sparingly).
- Cache: `~/.curator/cache.db`; 384×float32×30k = 46MB; total ~250MB uncompressed.
- Group sampling for large folders: filename TF-IDF k-means centroid; max 50 deep reads for groups >500 files. (R3)
- Progressive display: T+0-5min rough, T+5-12min refined, T+12-25min finalized. (R3)

### Layer 2 — Duplicate & Version Detection
- Pipeline: partition → size bucket → inode dedup → xxHash (partial) → SHA-256 (full) → SimHash (Hamming ≤5) → dHash (images, Hamming ≤10) → RETSim cosine → Union-Find. (R2)
- RETSim (`unisim`, archived May 2026, functional): cosine ≥0.90 = near-dup; 0.72-0.89 = version candidate; fallback = all-MiniLM-L6-v2. (R2)
- Version roles: 9 types (Draft, Revision, Final, Submitted, Archived, Superseded, Branch, Format Export, Reference Copy). (R2)
- Exact/near-dups: excluded from embedding. Version members: embedded. (R2)
- Minimum family size: 2 members for Review Hub entry.
- Never show Format Twins as duplicate families. (R7)

### Layer 3 — Context Graph & Community Detection
- FAISS IVFFlat batch; usearch for incremental inserts (transitive via unisim). (R4)
- Sparse SNF (K=20, 10 iter, `scipy.sparse.csr_matrix`). (R4)
- Composite edge weights: semantic 40%, NER 25%, behavioral 20%, provenance 10%, temporal 3%, filename 2%. (R4)
- Leiden via `leidenalg`; multi-resolution sweep γ=[0.5, 0.75, 1.0, 1.25, 1.5]. (R4)
- ADWIN (delta=0.002, `river`): one instance per community; ≥3 votes → NEW_CONTEXT. (R4)
- Folder Coherence Score: 0.35·MPS + 0.20·EC + 0.15·TSR + 0.20·SCD + 0.10·FTC. (R4)
- Context lifecycle: 7 states — new, active, stable, changing, dormant, finished, archived. (R4)
- `changing` state trigger: ≥3 new entities + new files OR ≥2 internal sub-communities. (R4)

### Layer 4 — Review Hub & Staging
- Physical move only (aliases/hardlinks ruled out). APFS `rename()` = atomic on same volume. (R6)
- Cross-volume: copy → verify (size+hash) → delete; WAL throughout. (R6, R8)
- `.curator_group.json` sidecar: generated from DB, never read back. (R6)
- Destination collision: rename with `_curator_N` suffix; never silent overwrite. (R6)
- Finder color tags on group subfolders (not individual files). (R6)
- FSEvents latency=0.5s during staging; path-prefix filter for hub events. (R6)

### Layer 5 — User Review & Learning
- 4 primary actions: Approve, Rename, Commit, Restore (Hick's Law). (R7)
- Keyboard: A=approve, R=rename, C=commit, Backspace=restore, S=split, L=lock, J/K=navigate, Cmd+Z=undo. (R7)
- Confidence: 4-dot qualitative scale (never percentages — Lucivero 2020 over-reliance). (R7)
- Peripheral files (cosine <0.50 to centroid): confidence ×0.20 from approve. (R5)
- Decision fatigue weights: ×0.5 after 20 groups, ×0.25 after 40 groups. (R5)
- Undo <60s = discarded (pattern count NOT incremented). (R5)
- Learning weights: approve=1.0, rename=2.0, split=3.0, merge=3.0, mark-wrong=3.0. (R5)
- L2 opt-in requires: confirmed_rule ≥7 days + ≥5 manual approvals + <5% undo + explicit opt-in. (R5)
- L3 opt-in requires: L2 ≥30 days + ≥20 auto-triggers + <10% undo + ≤5 files/action. Annual trust reset. (R5)
- Bulk action ≥10 files: always show confirmation even at L3. (R5)
- Rule specificity levels 0–5; generalizes to N+1 only after N+2 confirming corrections. (R5)
- Confidence decay: 20% per 30-day inactive period; stale <0.30, suspended <0.10. (R5)

### Layer 6 — Activity Log & Undo
- Append-only `curator_events`. Undos are new events, not deletions. (R8)
- `status='pending'` is the WAL signal; written before every file op. (R8)
- `source_inode`/`dest_inode` recorded for every move; enables crash recovery via FSEvents correlation. (R8)
- Linear undo (Cmd+Z) backed by SQLite (not NSUndoManager — does not survive restarts). (R8)
- Group undo = one Cmd+Z step; partial undo (missing files) requires explicit user confirmation. (R8)
- Rule-scoped undo: `SELECT * FROM curator_events WHERE rule_id = ? AND status != 'undone'`. (R8)
- Selective undo (Berlage & Genau 1993): NOT in v1. (R8)
- `duplicate_trash` undo: checks Trash via `NSFileManager.trashItem()` resultingItemURL; reversibility computed dynamically. (R8)
- Activity view: day + session grouping; filter bar; SQLite FTS5 full-text search over paths and reasons. (R8)
- Explanation templates: 8 reason types (semantic, ner, provenance, behavioral, duplicate, version, rule, coherence); composite stores JSON `reason_details`. (R8)
- Retention: full detail 1 year; monthly compaction to summaries; undo window 90 days. (R8)

---

## 5. Key Numbers Table

| Constant | Value | Source |
|---|---|---|
| Session cap (default) | 15 groups | R7, R9 |
| SimHash Hamming threshold (personal files) | ≤5 | R2 |
| RETSim near-dup threshold | cosine ≥0.90 | R2 |
| RETSim version candidate threshold | cosine 0.72–0.89 | R2 |
| dHash image threshold | Hamming ≤10 | R2 |
| θ_semantic | 0.60 | R4 |
| ADWIN delta | 0.002 | R4 |
| CBD: θ_birth | 0.55 | R4 |
| CBD: min_batch | 5 | R4 |
| CBD: gap_weeks | 8 | R4 |
| Min community size | 3 files | R4 |
| Min ADWIN votes for new context | ≥3 | R4 |
| Folder coherence: coherent | ≥0.75 | R4 |
| Folder coherence: chaotic | <0.30 | R4 |
| FAISS: nlist / nprobe / k | 100 / 10 / 20 | R4 |
| Sparse SNF: K / iterations | 20 / 10 | R4 |
| Embedding dim | 384 | R3 |
| Embedding latency | ~20ms/doc batched MPS | R3 |
| Tier 2+ file reach | ~10% | R3 |
| Concurrency | 16 I/O / 4 extraction / 2 embedding | R1 |
| N for approve/rename/merge → candidate | 3 | R5 |
| N for split/mark-wrong → candidate | 5 | R5 |
| Fatigue weight after 20 groups | ×0.5 | R5 |
| Fatigue weight after 40 groups | ×0.25 | R5 |
| Mondrian CP α | 0.15 | R5 |
| Mondrian CP min calibration examples | 20 | R5 |
| LabelSpreading alpha | 0.2 | R5 |
| SetFit trigger | ≥16 confirmed + ≥1 correction / 30 days | R5 |
| Confidence decay per 30-day inactive | 20% | R5 |
| L2: min days / approvals / undo rate | 7 / 5 / <5% | R5 |
| L3: min days / triggers / undo rate / max files | 30 / 20 / <10% / ≤5 | R5 |
| L3 random audit rate | 1-in-20 | R5 |
| Max nesting depth in Review Hub | 3 levels | R6 |
| Undo retention (staging_move / commit_move) | 90 days / 180 days; both indefinitely archived | R6, R8 |
| WAL stuck watchdog interval | 5 min | R1 |
| Stuck threshold: `new` state | 5 min | R1 |
| Stuck threshold: `grouped` state | 6 h | R1 |
| Behavioral edge session gap | 30 min | R4 |
| Behavioral edge decay half-life | 30 days (after 90 days) | R4 |
| Provenance edge weight (specific) | 0.8 | R4 |
| Provenance edge weight (domain-only) | 0.4 | R4 |
| Temporal edge (day / week / month) | 0.3 / 0.2 / 0.1 | R4 |
| Leiden expected optimal γ | 0.8–1.1 | R4 |
| igraph 30k node processing time | ~0.3s | R4 |
| FLRS decay half-life | 30 days | R4 |
| Event log storage (1 year steady state) | ~43 MB | R8 |
| Embedding cache total (30k files) | ~250 MB | R3 |
| OCR speed (Apple Vision, M3 Max) | 207ms/page | R3 |
| Progressive display finish | T+25 min | R3 |
| HippoCamp baseline (user profiling) | 48.3% | R9 |
| DCR target (acceptable / excellent) | ≥30 / ≥50 | R9 |
| GCR minimum threshold | 0.70 | R9 |
| Undo Rate hard gate | <5% | R9 |
| ProxAnn gate threshold | 0.70 | R9 |
| SUS target Phase 1 | ≥70 | R9 |
| Re-engagement rate target | >70% within 48h | R9 |

---

## 6. Edge Types Table

| Edge Type | Signal | Weight Contribution | Notes | Source |
|---|---|---|---|---|
| Semantic | FAISS cosine (nomic-embed-text) | 40% | θ_semantic=0.60 cutoff; backbone kNN k=20 | R4 |
| NER | spaCy entity overlap (Jaccard) | 25% | Fires when ≥2 shared entities; course codes, names, projects | R4 |
| Behavioral | Co-access within 30-min session window | 20% | Raw timestamps in memory only; 30-day half-life after 90 days | R4 |
| Provenance | `kMDItemWhereFroms` netloc+first_path_component | 10% | 0.8 specific / 0.4 domain-only; cross-volume stable | R4 |
| Temporal | Modification time proximity | 3% | Additive bonus only, never standalone; day=0.3, week=0.2, month=0.1 | R4 |
| Filename | TF-IDF token overlap | 2% | Course codes, project prefixes; degenerate without other signals | R4 |

Cannot-link constraints: negative weight −1.0 inserted in igraph before Leiden; stored in `boundary_constraints` table. (R4)
Must-link constraints: weight boosted to 0.95. (R4)

---

## 7. Open Questions

### Blocking (must resolve before shipping)

| ID | Question | Blocker for |
|---|---|---|
| B1 | Two-phase cross-volume staging: what is the correct behavior when source is on a network volume and copy succeeds but the network drops before delete? | Layer 4 (R6) |
| B2 | iCloud dataless files: `NSFileCoordinator` async download can block staging for arbitrarily long. What is the timeout and fallback? | Layer 4 (R6) |
| B3 | Can `NSFileManager.trashItem()` be called from a background process without a UI? Sandbox restrictions unclear. | Layer 4, Layer 6 |
| B4 | Mondrian CP requires min 20 calibration examples per context. New contexts start with 0. What confidence is assigned before 20 examples are collected? Use global fallback with α=0.15? | Layer 5 (R5) |
| B5 | RETSim (`unisim`) archived May 2026. Confirm the pip-installable package still builds on current Python/Apple Silicon before committing to it as the version similarity backbone. | Layer 2 (R2) |

### Non-Blocking (design decisions needed before Phase 2)

| ID | Question | Affects |
|---|---|---|
| NB1 | Undo window for committed files: 90 days uniform, or longer for `commit_move` events specifically? | R8 |
| NB2 | Crash recovery UX: silent or surfaced ("3 operations recovered")? | R8 |
| NB3 | Per-file undo within a group commit: deferred to v1, but should the data model accommodate it now? | R8 |
| NB4 | `reason_details` as JSON array vs. normalized `curator_event_signals` table? JSON is flexible; normalized is queryable. | R8 |
| NB5 | Should the Activity log be accessible from `_Curator Review` as a `.curator_log.html` file? Security: unencrypted. | R8 |
| NB6 | OBI component weights (α, β, γ, δ) uncalibrated. Phase 2 must produce correlation with user satisfaction before OBI is published. | R9 |
| NB7 | DCR target for a 30k chaotic filesystem vs. an already-organized one — need baseline measurement in Phase 1. | R9 |
| NB8 | ProxAnn gate at 0.70: no empirical basis for Curator specifically. Calibrate from Phase 0 HippoCamp distribution. | R9 |
| NB9 | Undo <60s = discarded (R5). Should this threshold be configurable per user? | R5 |
| NB10 | `duplicate_trash`: Trash check is dynamic. Should reversibility be re-evaluated on every Activity view render or lazily on undo click? | R8 |

### Research (IUI 2027 open questions)

| ID | Question |
|---|---|
| RQ1 | Does the ADHD OAP (Overwhelm Abandonment Point) differ significantly from neurotypical OAP? Testable in Phase 2 with ANOVA. |
| RQ2 | Does Coherence Verification pre-filtering improve GCR enough to justify ~$2-5/session API cost? Measure Phase 1 with/without. |
| RQ3 | Does OBI correlate with subjective organizational stress (r ≥ 0.5)? Would validate OBI as a published metric. |
| RQ4 | How does HippoCamp QA accuracy map to human re-finding success? Needs Phase 2 dual measurement. |
| RQ5 | Can FSEvents + MDQuery intercept Spotlight queries that resolve to Curator-organized files without a kernel extension? (Search-After-Organization Rate). |

---

## 8. Prior Work

| System / Paper | What Curator takes from it | Curator decision |
|---|---|---|
| **HippoCamp** (arXiv:2604.01221) | 42.4 GB benchmark, 581 QA pairs, 48.3% baseline on user profiling | Primary Phase 0 benchmark; QA accuracy after Curator organization is the paper's headline automated result (R9) |
| **CFCOS** (Electronics 2026) | LLM-based per-file category classification baseline | Phase 0 baseline for file-level category accuracy (R9) |
| **ProxAnn** (Hoyle et al., ACL 2025) | LLM held-out classification accuracy ≈ human annotation quality | Integrated as Stage 3 of automated pipeline pre-filter; groups <0.70 flagged (R9) |
| **Islam 2026** (arXiv:2604.07562) | 3-stage LLM coherence verification + redundancy adjudication | Stage 1 coherence verification as pre-surface filter; Stage 2 redundancy adjudication as auto-merge (R9) |
| **fclones** | 6-stage duplicate model: size → partial hash → full hash → clustering | Architectural reference for R2's 8-stage pipeline; Curator adds SimHash + RETSim layers (R2) |
| **Leiden algorithm** (Traag et al. 2019) | Guarantees connected communities; Louvain produces disconnected up to 25% of cases | Curator uses `leidenalg` exclusively; Louvain ruled out (R4) |
| **Sparse SNF** (Wang et al. 2014, adapted) | K-nearest neighbor fusion across views | Full SNF infeasible (3.6 GB/matrix); Curator uses sparse variant (K=20, scipy.sparse) (R4) |
| **ADWIN** (Bifet & Gavalda 2007) | Adaptive windowing for concept drift detection | Used per-community for context birth detection; `river` library (R4) |
| **Bergman et al. 2010** | Folder depth ≤3, 8-15 files/folder, ~15s re-find time, 94% success | Phase 2 primary re-finding baseline; Curator Review Hub max depth=3 (R6, R9) |
| **Brackenbury et al. SIGIR 2021** | Perceived co-grouping as pairwise user preference | GCR as proxy; pairwise co-grouping sample (10%) in Phase 2 only (R9) |
| **Cushing 2022 (PIM-B)** | PIM burden = extra activity + negative affect + identity gap + extra seeking | "Ownership" survey item in Phase 2; Search-After-Organization Rate operationalizes component 4 (R9) |
| **Danziger et al. 2011** (Israeli judges) | Decision quality degrades within a session | Justification for 15-item session cap; Decision Quality Degradation Index metric (R9) |
| **Parasuraman & Sheridan** | Automation levels taxonomy (levels 1-10) | L1/L2/L3 map to Sheridan levels 3/5/7 (R5) |
| **Git reflog** | Append-only, log intent not just state, undo = new event | Curator's activity log is append-only; undo creates new events (R8) |
| **NSUndoManager** | Group undo as one step; linear stack; memory-only | Curator does not use NSUndoManager for undo logic (restarts would break it); own SQLite-backed stack (R8) |
| **Berlage & Genau 1993** | Selective undo theory; dependency detection required | Selective undo deferred from v1; rule-scoped undo as practical bounded case (R8) |
| **APFS copy-on-write** | Same-volume clone = near-instant + space-efficient | `copyItem` on APFS is COW; Curator uses for staging copy phase (R8) |
| **Hick's Law** | Reaction time ∝ log(N alternatives) | 4 primary actions only in Review Hub (R7) |
| **Lucivero 2020** | Numeric confidence causes over-reliance; qualitative preferable | 4-dot scale, never percentages in group card (R7) |

---

---

## §9 — Supplementary Research Modules (R11–R14)

These modules were added after the initial R0–R10 synthesis. They are **Phase 2** features unless noted. They do not block MVP Layer 0–7.

### R11 — Calm Vector Memory & Thermal-Aware Indexing
- 5-layer memory pyramid: Identity → Sketch → Compressed → Dynamic → Ephemeral
- Thermal Governor: IOPMLib state + CPU temperature → dynamically throttle background processing
- Files climb the pyramid only when thermal budget allows
- `thermal.py` module: monitors thermals, pauses/resumes workers
- MVP relevance: thermal monitoring is MVP (prevents Mac overheating during scan)

### R12 — External Research Acceleration Scan
- Scan Spotlight index, Git repos, browser downloads for provenance signals
- Feeds Tier 0 reader with richer `kMDItemWhereFroms` data
- Phase 2 only

### R13 — Progressive Completeness & Thermal-Safe Scan
- Files get a `completeness_score` (0.0–1.0) based on how much was extracted
- Scan resumes from last checkpoint on restart (no re-processing)
- Processing queue ordered by: priority × (1 − completeness) × thermal_headroom
- MVP relevance: checkpoint/resume behavior is MVP

### R14 — Spatially Sharded Candidate Graph
- At n>10,000 files, FAISS IVFFlat is partitioned into spatial shards
- Each shard is a geographic cluster of the embedding space
- Cross-shard edges only for high-weight relationships
- Phase 2 only (MVP uses single FAISS IVFFlat index)

### TECH — Operation Passports & Compute Budget
- Every background job must declare upfront: CPU cost tier, I/O cost tier, reversibility, power requirement, user-presence requirement, completeness value
- `PassportGate` checks passport against current system state before every job starts
- If job has no passport → does not run
- Integrates with Thermal Governor (R11) and processing queue (TECH_engineering_foundation)
- MVP relevance: passport system is MVP for file move operations (reversibility declaration)

### Integration with MVP Build Plan
- `thermal.py` → Layer 0 (Infrastructure)
- `passports.py` + `PassportGate` → Layer 0 (Infrastructure)  
- `vector_index.py` (FAISS wrapper with progressive completeness) → Layer 1 (Ingestion)
- R12, R14 → Phase 2

*Document length: ~550 lines. Sources: 00, R1–R14.*
