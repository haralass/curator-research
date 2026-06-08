# R6 — Review Hub Physical Staging Design

**Curator Research Series | New Vision Pipeline**
*Written: 2026-06-08 | Model: Claude Sonnet 4.6*

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Physical vs Virtual — Why Physical is Correct](#2-physical-vs-virtual)
3. [Folder Structure Specification](#3-folder-structure-specification)
4. [Per-File Metadata Schema](#4-per-file-metadata-schema)
5. [Group Metadata Schema](#5-group-metadata-schema)
6. [Commit Semantics](#6-commit-semantics)
7. [Restore Semantics & Undo Log](#7-restore-semantics--undo-log)
8. [Spotlight & FSEvents Management](#8-spotlight--fsevents-management)
9. [Finder Integration](#9-finder-integration)
10. [iCloud & Locked Volume Handling](#10-icloud--locked-volume-handling)
11. [Prior Art](#11-prior-art)
12. [Design Decisions](#12-design-decisions)
13. [Open Questions](#13-open-questions)

---

## 1. Executive Summary

The Review Hub is the physical manifestation of Curator's "many files → few groups → few decisions" philosophy. Every file that needs user guidance is physically moved to `_Curator Review/` — a real folder on disk — not represented by aliases, symlinks, or virtual views. The user can open Finder, see exactly what is staged, make decisions in the folder directly if they choose, and trust that Curator's database faithfully records what is there and where it came from.

This document specifies:
- Why physical move is the correct choice on APFS and what atomicity guarantees exist
- The exact folder tree, naming conventions, and collision resolution
- SQLite schemas for per-file and per-group metadata
- Transactional commit semantics with rollback on partial failure
- Undo log design supporting restore-after-commit and missing-directory cases
- Spotlight exclusion, FSEvents coalescing, and batch-move strategy
- Finder-native communication: custom icons, color tags, sidecar JSON files
- iCloud eviction and locked-volume handling
- What Curator inherits from Git staging, macOS Trash, DEVONthink, and Hazel

**Curator decision (summary):** Physical move with write-ahead log. `rename()` within APFS volume is atomic. The undo log is an append-only SQLite table, never deleted. Commit is transactional: all-or-nothing per group with per-file rollback if any step fails. Restore always goes to `original_path` if the directory exists; Desktop is the fallback. `_Curator Review/` carries a `.metadata_never_index` sentinel file to suppress Spotlight. Every group subfolder contains a `.curator_group.json` sidecar and gets a Finder color tag.

---

## 2. Physical vs Virtual — Why Physical is Correct

### 2.1 The Three Approaches Compared

**Approach A — Physical move:** The file is relocated from its source path to a path inside `_Curator Review/`. The file has one canonical location. Finder shows the file in `_Curator Review/`. The source location is empty.

**Approach B — Alias/symlink:** The file stays in its source location. An alias or symlink is placed in `_Curator Review/` pointing to the source. The file appears to exist in two places.

**Approach C — Hardlink:** A second directory entry is created for the same inode. The file appears in both locations, and both entries are equally "real". Changes to either are immediately reflected in both.

### 2.2 Why Alias/Symlink Fails for Curator

macOS aliases (`.alias` files using `FSCopyObjectSync` with `kFSFileOperationDefaultOptions`) are more robust than POSIX symlinks — they track the target by inode and volume UUID, not just path — but they are still fragile for Curator's use case:

1. **Crash during alias creation**: if Curator crashes between moving the file and creating the alias, the alias does not exist and the original is gone. This is strictly worse than physical move, where a crash leaves the file in a well-defined location (either source or destination, depending on where `rename()` atomically completed).

2. **Aliases confuse applications**: when an application follows an alias, it opens the original path. Finder resolves aliases transparently. This means the "source" folder still appears to contain the file to many apps, defeating the purpose of staging (showing the user what is unresolved).

3. **Alias invalidation**: if the user manually moves the source file before Curator resolves it, the alias target is stale. macOS alias resolution will search the volume for the inode (Finder does this), but command-line tools do not. Any CLI-based restore would fail silently.

4. **Time Machine and iCloud**: both services follow aliases and back up the original. The alias in Review Hub is essentially invisible to backup systems. Curator's staging area would not appear in Time Machine or iCloud Drive at all, making it impossible to recover a staged file from backup.

**Prior art:** Time Machine uses hardlinks (not aliases) for "same file, multiple snapshot appearances" because hardlinks are native inode-level constructs. But Time Machine hardlinks are a special APFS capability (`APFS_FEATURE_HARDLINK_MAP`) that is not available to user-space applications without entitlements.

**Curator decision:** Aliases and symlinks are ruled out. They create split-brain file state: the file appears in two places but is only physically in one. Curator's design principle is that the Review Hub is the single source of truth for staged files.

### 2.3 Why Hardlinks Fail for Curator

POSIX hardlinks cannot cross volume boundaries (`EXDEV` error). On a Mac with an APFS container holding multiple volumes (Macintosh HD + user Data volume), cross-volume hardlinks do not work. On a system with external drives, they are always impossible. Even within a single APFS volume, hardlinks to directories are not supported in user space (only the kernel can create directory hardlinks, for Time Machine purposes). Since Curator stages entire groups (multiple files, possibly into a subfolder), directory hardlinks are the natural tool but they are unavailable.

**Curator decision:** Hardlinks are ruled out due to volume boundary restrictions and the directory hardlink limitation.

### 2.4 Why Physical Move is Correct

Physical move with `rename()` within the same APFS volume is:

- **Atomic**: `rename()` is implemented as a single metadata operation in APFS — it swaps the directory entry atomically. There is no "in-between" state visible to other processes. APFS guarantees this at the volume level (Apple APFS Reference, https://developer.apple.com/documentation/kernel/1387510-rename). This is analogous to `renameat2()` on Linux, but on macOS `rename()` itself provides the guarantee when source and destination are on the same volume.

- **Crash-safe**: if Curator crashes after `rename()` completes, the file is in the destination. If it crashes before, the file is in the source. The write-ahead log (WAL) in Curator's SQLite database records the *intent* before the `rename()` call. On next startup, the WAL is replayed: if the file is not in its intended `staged_path`, it is still at `original_path` and the record is corrected.

- **No Spotlight pollution at source**: once the file is moved, Spotlight drops the source path from its index. The destination is in `_Curator Review/`, which is excluded from Spotlight (see Section 8). Net result: the file disappears from Spotlight for the duration of staging, which is correct behavior — a staged file is "under review" and should not appear in general search.

- **Finder-honest**: the user who opens the source folder sees that the file is gone. The user who opens `_Curator Review/` sees it. There is no ambiguity.

### 2.5 Cross-Volume Move Handling

If a file is on an external drive or a separate APFS volume and `_Curator Review/` is on the system volume, `rename()` will fail with `EXDEV`. In this case, Curator must:

1. Copy the file to the Review Hub destination (`copyfile()` with `COPYFILE_ALL` to preserve xattrs, resource forks, and metadata)
2. Verify the copy (SHA-256 comparison)
3. Delete the source only after verification passes

This is a two-phase operation, not atomic. The WAL must record: `{ state: "cross_volume_copy_pending", source_verified: false }`. If a crash occurs after copy but before source deletion, the recovery path on next startup checks whether the copy hash matches the WAL-recorded hash, and if so, completes the source deletion.

**Curator decision:** Same-volume: use `rename()` — atomic, instantaneous. Cross-volume: use copy-verify-delete with WAL protection. Cross-volume staging is a warning in the UI: "This file is on an external drive. Staging requires a copy, which takes longer and uses space."

### 2.6 What "Limbo" Looks Like and How to Prevent It

Limbo = a file that is neither at its original path nor at its staged path. This can happen if:
- Curator crashes between `rename()` source→temp and `rename()` temp→destination (not applicable if using a single `rename()`)
- Cross-volume copy fails partway

The write-ahead log prevents limbo: the intended `staged_path` is written to the DB before any file operation. On startup, Curator checks all records with `file_state = 'staged_review'` and verifies that the file exists at `staged_path`. Any discrepancy triggers reconciliation (see Section 7.5).

**Curator decision:** Write-ahead log is mandatory. The WAL entry is written and `fsync()`ed before the `rename()` call. This adds ~1ms per file but prevents any limbo state.

---

## 3. Folder Structure Specification

### 3.1 Top-Level Design Philosophy

The Review Hub is context-first, not file-type-first. A user looking at `_Curator Review/` should see categories that reflect *why* files are staged, not what kind of files they are. "Mixed Folders" and "New Contexts" communicate the organizing principle; a flat list of extensions would not.

The name `_Curator Review` begins with `_` so it sorts to the top of any alphabetically sorted folder listing in Finder. The space in "Curator Review" makes it readable as a phrase rather than a technical identifier.

### 3.2 Complete Folder Tree

```
_Curator Review/
│
├── .metadata_never_index          ← sentinel file: suppresses Spotlight on this folder
├── .curator_hub.json              ← hub-level metadata (version, created_at, curator_version)
│
├── Duplicates/
│   ├── Exact/
│   │   └── [Family Name]/         ← e.g. "Thesis Draft - 3 copies"
│   │       ├── .curator_group.json
│   │       └── [files...]
│   └── Near/
│       └── [Family Name]/         ← e.g. "Lab Report Similar"
│           ├── .curator_group.json
│           └── [files...]
│
├── Versions/
│   └── [Family Name]/             ← e.g. "CV - 5 versions"
│       ├── .curator_group.json
│       └── [files...]
│
├── New Contexts/
│   └── [Context Candidate Name]/  ← e.g. "Cyber Security - New?"
│       ├── .curator_group.json
│       └── [files...]
│
├── Mixed Folders/
│   └── [Folder Name] - Incoherent/  ← e.g. "Downloads - Incoherent"
│       ├── .curator_group.json
│       └── [files...]
│
├── Unknown/
│   ├── Unclustered/               ← files that got no group assignment
│   │   ├── .curator_group.json
│   │   └── [files...]
│   └── [Rough Topic]/             ← e.g. "University Material", "Code Projects"
│       ├── .curator_group.json
│       └── [files...]
│
└── Failed to Read/
    ├── Protected/
    │   ├── .curator_group.json
    │   └── [files...]
    ├── Corrupted/
    │   ├── .curator_group.json
    │   └── [files...]
    ├── Too Large/
    │   ├── .curator_group.json
    │   └── [files...]
    └── Unsupported/
        ├── .curator_group.json
        └── [files...]
```

**What is excluded:** A `Locked/` subfolder does not exist. Locked files never enter the state machine (R1 decision). If a file becomes locked after staging, it transitions to `failed_to_read` with `failure_code = 'became_locked'` and moves to `Failed to Read/Protected/`.

### 3.3 Naming Conventions

**Family/Group Auto-Naming**

Duplicate and version family names are generated by:

1. Extract the base name of each file in the family (strip extension, strip trailing numbers/dates)
2. Compute the Longest Common Subsequence (LCS) of the base names
3. If LCS length ≥ 4 characters: use LCS as the family name
4. If LCS < 4 characters: use the most common file extension + count (e.g., "PDF - 7 files")
5. Append " - N copies" for exact duplicates, " - N versions" for version families

Examples:
- Files: `thesis_v1.pdf`, `thesis_v2.pdf`, `thesis_final.pdf` → LCS = "thesis" → name: "thesis - 3 versions"
- Files: `IMG_4521.jpg`, `IMG_4522.jpg` → LCS = "IMG_452" → name: "IMG_452 - 2 copies"
- Files: `a.pdf`, `b.pdf`, `report.pdf` → LCS = "" (too short) → name: "PDF - 3 copies"

For New Context candidates, the name comes from the context graph's auto-label (R4). For Mixed Folders, the name is the source folder's name + " - Incoherent".

**Collision Resolution**

If two groups would produce the same auto-name, append a monotonically increasing suffix: "thesis - 3 versions", "thesis - 3 versions (2)", "thesis - 3 versions (3)". The suffix is stored in the group's `disambiguator` field in the DB so it can be removed if one of the colliding groups is resolved and renamed.

**Curator decision:** LCS-based naming for duplicate/version families. Collision suffixes are numeric in parentheses, not underscores or dashes, to avoid confusion with filename conventions. The `disambiguator` is updated whenever a group is resolved.

### 3.4 Maximum Nesting Depth

Boardman & Sasse (2004, "My Photos are Everywhere", CHI) found that users tolerate at most 3-4 levels of folder nesting before navigation cost increases non-linearly. Their study showed that navigation errors double with each level beyond 3 (Boardman & Sasse, CHI 2004, https://dl.acm.org/doi/10.1145/985692.985766).

The Review Hub tree above has a maximum depth of 3 from `_Curator Review/`:
- Level 1: category (Duplicates, Versions, New Contexts, etc.)
- Level 2: subcategory (Exact, Near for Duplicates; topic for Unknown)
- Level 3: family/group folder

Files live at level 3 (or level 2 for flat categories like Versions, New Contexts, Mixed Folders, Failed to Read). No deeper nesting is permitted.

**Curator decision:** Hard limit of 3 levels below `_Curator Review/`. If a future category requires deeper nesting, the design must be reconsidered, not the depth limit.

---

## 4. Per-File Metadata Schema

### 4.1 Design Rationale

Document management systems (SharePoint, Documentum, Alfresco) universally track: document identifier, source path, disposition, version, and audit trail fields. Documentum's `dm_document` object has 200+ attributes; Curator uses the minimal set required for correct operation plus the fields required by R1's state machine.

The metadata is stored in Curator's SQLite database (not in xattrs, not in `.DS_Store`, not in the sidecar JSON — those are for Finder communication only). The SQLite database is the single source of truth.

### 4.2 SQLite Table: `staged_files`

```sql
CREATE TABLE staged_files (
    -- Identity
    id                  TEXT PRIMARY KEY,       -- UUID v4, never reused
    inode               INTEGER NOT NULL,       -- inode at staging time
    device_id           INTEGER NOT NULL,       -- st_dev at staging time
    sha256              TEXT,                   -- hex SHA-256; NULL if unreadable
    size_bytes          INTEGER NOT NULL,
    mtime_ns            INTEGER NOT NULL,       -- nanosecond mtime at staging time

    -- Paths
    original_path       TEXT NOT NULL,          -- absolute path before staging
    staged_path         TEXT NOT NULL,          -- absolute path in Review Hub
    restore_path        TEXT,                   -- NULL = use original_path; set if user specifies alternate

    -- Staging event
    staged_at           TEXT NOT NULL,          -- ISO 8601 UTC timestamp
    staged_by_run_id    TEXT NOT NULL,          -- FK → pipeline_runs.id

    -- Reason
    reason              TEXT NOT NULL CHECK (reason IN (
                            'duplicate_exact',
                            'duplicate_near',
                            'version_family',
                            'mixed_folder',
                            'new_context_candidate',
                            'failed_to_read',
                            'unknown_unclustered'
                        )),
    confidence          REAL NOT NULL CHECK (confidence BETWEEN 0.0 AND 1.0),
    failure_code        TEXT CHECK (failure_code IN (
                            'protected',
                            'corrupted',
                            'too_large',
                            'unsupported_format',
                            'became_locked',
                            NULL
                        )),

    -- Group membership
    group_id            TEXT NOT NULL,          -- FK → staged_groups.id
    
    -- State machine (from R1)
    file_state          TEXT NOT NULL CHECK (file_state IN (
                            'staged_review',
                            'committed',
                            'ignored',
                            'user_corrected',
                            'failed_to_read'
                        )),

    -- Commit outcome (populated after commit)
    committed_path      TEXT,                   -- where the file actually landed
    committed_at        TEXT,                   -- ISO 8601 UTC

    -- WAL safety flag
    wal_intent          TEXT CHECK (wal_intent IN (
                            'stage_pending',
                            'stage_complete',
                            'commit_pending',
                            'commit_complete',
                            'restore_pending',
                            'restore_complete',
                            NULL
                        )),

    -- Audit
    created_at          TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
    updated_at          TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
);

CREATE INDEX idx_staged_files_group_id ON staged_files(group_id);
CREATE INDEX idx_staged_files_original_path ON staged_files(original_path);
CREATE INDEX idx_staged_files_file_state ON staged_files(file_state);
CREATE INDEX idx_staged_files_sha256 ON staged_files(sha256) WHERE sha256 IS NOT NULL;
CREATE INDEX idx_staged_files_wal ON staged_files(wal_intent) WHERE wal_intent IS NOT NULL;
```

### 4.3 Field Notes

**`id` (UUID v4):** Never derived from the path or inode. Paths change on rename; inodes are reused after deletion. The UUID is the stable cross-session identifier for this staging event.

**`inode` + `device_id`:** Recorded at staging time to detect if the file was externally moved out of the Review Hub. On next startup, Curator can check `stat(staged_path)` and compare `st_ino + st_dev` against the DB. A mismatch means external interference.

**`sha256`:** May be NULL for files in `Failed to Read/` (if the file could not be read at all). For `duplicate_exact` reason, SHA-256 is always present.

**`restore_path`:** Initially NULL, meaning "restore to `original_path`". The user can specify an alternate restore destination via the UI. Once set, this field persists even if the user later changes their mind — the undo log captures the sequence.

**`wal_intent`:** The write-ahead log flag. Before any file operation, this field is set and `fsync()`ed. After the operation completes successfully, the field is cleared (set to NULL). On startup, any non-NULL `wal_intent` triggers recovery.

**`failure_code`:** Only populated when `reason = 'failed_to_read'`. This encodes the specific failure type for the "Failed to Read" subfolder routing.

**Curator decision:** All per-file metadata lives in SQLite, not in xattrs or sidecar files. xattrs are stripped by iCloud sync, email attachment, and many third-party copy tools. The sidecar `.curator_group.json` in each folder is a read-only Finder-communication artifact, regenerated from the DB as needed.

---

## 5. Group Metadata Schema

### 5.1 SQLite Table: `staged_groups`

```sql
CREATE TABLE staged_groups (
    -- Identity
    id                  TEXT PRIMARY KEY,       -- UUID v4
    group_type          TEXT NOT NULL CHECK (group_type IN (
                            'duplicate_exact',
                            'duplicate_near',
                            'version_family',
                            'new_context',
                            'mixed_folder',
                            'failed',
                            'unknown'
                        )),

    -- Display
    group_name          TEXT NOT NULL,          -- auto-generated, user-editable
    disambiguator       INTEGER,                -- NULL or 2,3,... for collision resolution
    hub_path            TEXT NOT NULL,          -- absolute path to group's subfolder in Review Hub

    -- Membership
    member_count        INTEGER NOT NULL DEFAULT 0,
    total_size_bytes    INTEGER NOT NULL DEFAULT 0,

    -- Timing
    created_at          TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
    updated_at          TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),

    -- User action
    user_action         TEXT NOT NULL DEFAULT 'pending' CHECK (user_action IN (
                            'pending',
                            'approved',
                            'rejected',
                            'split',
                            'merged_into',
                            'committed',
                            'restored',
                            'ignored'
                        )),
    user_action_at      TEXT,                   -- ISO 8601 UTC; NULL until acted upon

    -- For split/merge tracking
    parent_group_id     TEXT,                   -- FK → staged_groups.id; set when group is a split product
    merged_into_id      TEXT,                   -- FK → staged_groups.id; set when group was merged into another

    -- Commit destination (populated when user commits)
    commit_destination  TEXT,                   -- user-specified destination folder

    -- Confidence aggregate
    min_confidence      REAL,                   -- minimum confidence across members
    avg_confidence      REAL,                   -- mean confidence across members

    -- Source folder (for mixed_folder type)
    source_folder_path  TEXT,                   -- original folder path that was incoherent

    -- Context graph info (for new_context type)
    context_label       TEXT,                   -- label from context graph
    context_cluster_id  TEXT,                   -- FK → context_clusters.id if applicable

    -- Finder communication
    finder_tag_color    TEXT CHECK (finder_tag_color IN (
                            'red', 'orange', 'yellow', 'green', 'blue', 'purple', 'gray', NULL
                        ))
);

CREATE INDEX idx_staged_groups_user_action ON staged_groups(user_action);
CREATE INDEX idx_staged_groups_group_type ON staged_groups(group_type);
CREATE INDEX idx_staged_groups_parent ON staged_groups(parent_group_id) WHERE parent_group_id IS NOT NULL;
```

### 5.2 Field Notes

**`disambiguator`:** Supports collision-resolution naming (see Section 3.3). When `disambiguator` is NULL, the folder name is `group_name`. When it is 2 or 3, the folder name is `group_name (2)`.

**`user_action` lifecycle:** `pending → approved → committed` (happy path). `pending → rejected → restored` (rejection path). `pending → split` creates two new groups with `parent_group_id` pointing to the original. `pending → merged_into` sets `merged_into_id` and marks the absorbed group as inactive.

**`finder_tag_color`:** Assigned by group type: `red` = failed_to_read, `orange` = duplicate_exact, `yellow` = duplicate_near, `blue` = new_context, `purple` = version_family, `gray` = unknown, `green` = approved/ready-to-commit. This drives the Finder color tag applied to the group subfolder (see Section 9).

**Curator decision:** Groups are first-class entities with their own IDs and full lifecycle tracking. File-level and group-level decisions are recorded independently. A user can commit individual files from a group without committing the group as a whole (partial commit — see Section 6.3).

---

## 6. Commit Semantics

### 6.1 What "Commit" Means

Commit is the user's declaration: "I have decided where these files go." Files move from `_Curator Review/` to their final destination. The group transitions from `approved` to `committed`. The files transition from `staged_review` to `committed`.

Commit is not irreversible — the undo log always allows restoration — but it is intentional and distinct from staging. Staging is automatic (Curator does it); commit is manual (user does it).

### 6.2 Destination Selection

When a user commits a group, they must specify a destination folder. Curator assists with:

1. **Suggestion from context graph**: for `new_context` type, the context graph's recommended folder is pre-populated
2. **Suggestion from similar groups**: if an earlier group of the same type was committed to folder X, and the new group's label similarity to that group exceeds 0.7, suggest X
3. **Free-form entry**: user can type any path or drag a folder from Finder
4. **New folder creation**: user can specify a path that does not yet exist; Curator creates it before moving files

The destination is stored in `staged_groups.commit_destination` when the user selects it, before files are moved. This allows the WAL to reference a complete target even in crash scenarios.

### 6.3 Partial Commit

A user may commit only some files from a group. This is supported:

1. User selects a subset of files in the group's UI view
2. Selected files are committed; their `file_state` transitions to `committed`
3. Remaining files stay in `staged_review` in the same group subfolder
4. The group's `user_action` stays `pending` until all members are either committed or restored
5. `member_count` is not decremented; a separate `committed_count` column tracks progress

This is analogous to `git add -p` (partial staging of hunks) — the user has granular control without losing the group context.

### 6.4 Destination Collision Handling

If a file already exists at the commit destination:

| Scenario | Action |
|---|---|
| Same SHA-256 (identical content) | Skip the move; mark the staged file as `committed` (the destination already has the content) |
| Different SHA-256, user has not been asked | Pause commit for this file; present three options: Rename (auto-suffix), Skip, Overwrite |
| User chose Rename | Append `_curator_N` before the extension; N increments until no collision exists |
| User chose Overwrite | Move staged file to destination, replacing existing; record replaced file's path + hash in undo log |
| User chose Skip | Leave staged file in Review Hub; mark as `ignored` |

**Curator decision:** Never silently overwrite. A collision is a user decision point, not a default. The default behavior (if no user input is possible, e.g., batch commit) is Rename.

### 6.5 Transactional Commit

Committing a group of N files must be transactional: either all succeed or none do. This is critical to avoid partial-commit states that leave the group in an undefined condition.

**Implementation using WAL + SQLite transaction:**

```
BEGIN TRANSACTION;
  UPDATE staged_groups SET user_action = 'committed', user_action_at = now() WHERE id = ?;
  FOR EACH file IN group:
    UPDATE staged_files SET wal_intent = 'commit_pending' WHERE id = ?;
COMMIT;  -- fsync happens here
fsync(db_file);

FOR EACH file IN group:
  rename(staged_path, committed_path);  -- atomic on same volume
  BEGIN TRANSACTION;
    UPDATE staged_files SET
      file_state = 'committed',
      committed_path = ?,
      committed_at = now(),
      wal_intent = NULL
    WHERE id = ?;
  COMMIT;

IF any rename() fails:
  ROLLBACK all completed renames (move back to staged_path);
  UPDATE staged_files SET wal_intent = NULL WHERE group_id = ?;
  UPDATE staged_groups SET user_action = 'approved' WHERE id = ?;  -- revert to pre-commit state
  REPORT failure to user;
```

**Why not a single atomic transaction for all file moves?** File system operations (`rename()`) cannot be wrapped in a SQLite transaction — they are OS-level calls. The WAL pattern is the standard solution: record intent in the DB, execute the OS call, clear intent on success. This is exactly what APFS journaling does at the volume level (Apple APFS Reference, Section "Journal and Metadata Writing").

**Cross-volume commit failure handling:** If the commit destination is on a different volume, the copy-verify-delete pattern is used (see Section 2.5). If the copy succeeds but the verify fails, the copy is deleted and the commit is rolled back.

**Curator decision:** Commit is group-atomic by default. If any file in the group fails to move, all successful moves within that commit attempt are rolled back. The user is shown exactly which file caused the failure and why.

### 6.6 What Can Go Wrong During Commit

Research on macOS file move errors (Apple Technical Note TN2319, POSIX `rename()` errno values):

| Error | errno | Curator response |
|---|---|---|
| Permissions denied | EACCES | Report to user; leave file staged; log in undo log as `commit_failed_permissions` |
| Cross-volume (destination on different drive) | EXDEV | Switch to copy-verify-delete path automatically |
| Disk full at destination | ENOSPC | Pause commit; warn user; retry when space is available |
| File locked (BSD immutable flag) | EPERM | Report; show user how to unlock; retry |
| Destination path too long | ENAMETOOLONG | Offer to shorten filename; never silently truncate |
| Destination directory does not exist | ENOENT | Create directory first (up to 3 levels deep); if deeper, ask user |

**Curator decision:** Curator creates missing destination directories automatically up to 3 levels deep. If the destination requires creating more than 3 new directories, the user is asked to confirm the path, because it likely indicates a typo.

---

## 7. Restore Semantics & Undo Log

### 7.1 What "Restore" Means

Restore is the user's declaration: "This file should not be in the Review Hub." The file moves back to `restore_path` (which defaults to `original_path`). The group transitions to `restored`. The file transitions to `ignored` (it has been explicitly dismissed).

Restore can also be triggered automatically during crash recovery (see Section 7.5).

### 7.2 Restore to Original Path

**Happy path:** `original_path` directory still exists → `rename(staged_path, original_path)` → done.

**Original directory deleted:** The parent directory of `original_path` no longer exists. Options presented to user:
1. Desktop (`~/Desktop/[filename]`)
2. User-specified alternate path
3. Keep in Review Hub with `user_action = 'ignored'` (the file stays but is de-emphasized in the UI)

The "keep in Review Hub" option is important for files the user wants to deal with later but doesn't want Curator to re-stage.

**Curator decision:** When the original directory is missing, the default restore destination is Desktop. This is a well-known, always-accessible location. The UI clearly states: "Original folder no longer exists. File will be restored to Desktop unless you choose a different location."

### 7.3 Restore After Commit

A file has been committed (moved to `committed_path`). The user regrets the decision. Restore-after-commit means: move the file from `committed_path` back to `original_path` (or user-specified path).

This requires that `committed_path` is recorded in `staged_files` and the undo log. Restore-after-commit is indistinguishable mechanically from a regular restore: `rename(committed_path, restore_path)`. The undo log entry captures the full operation chain.

**Cursor decision:** There is no time limit on restore-after-commit. As long as the undo log entry exists and the file is at `committed_path`, the restore can be executed. However, if the user has further modified the file at `committed_path`, Curator warns: "This file has been modified since it was committed. Restoring it will move the modified version, not the original."

### 7.4 Partial Restore

If a group has 10 files, the user can restore 3 and commit the other 7. Each file's restore/commit is tracked independently. The group's `user_action` stays `pending` until all members are resolved.

### 7.5 Undo Log Format

The undo log is an **append-only SQLite table**. Records are never deleted (only soft-archived after 90 days). This is structurally similar to `git reflog` — an immutable audit trail of all operations.

```sql
CREATE TABLE undo_log (
    id              TEXT PRIMARY KEY,           -- UUID v4
    operation       TEXT NOT NULL CHECK (operation IN (
                        'stage',
                        'commit',
                        'restore',
                        'commit_rollback',
                        'restore_rollback',
                        'crash_recovery'
                    )),
    file_id         TEXT NOT NULL,              -- FK → staged_files.id
    group_id        TEXT,                       -- FK → staged_groups.id; NULL for file-only ops
    from_path       TEXT NOT NULL,              -- where the file was before this operation
    to_path         TEXT NOT NULL,              -- where the file went after this operation
    sha256_before   TEXT,                       -- hash before operation (for integrity verification)
    sha256_after    TEXT,                       -- hash after operation; should equal sha256_before
    performed_at    TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now')),
    performed_by    TEXT NOT NULL DEFAULT 'curator',  -- 'curator' or 'user' (for manual moves)
    success         INTEGER NOT NULL DEFAULT 1, -- 0 if operation failed
    failure_reason  TEXT,                       -- populated if success = 0
    undone_at       TEXT,                       -- NULL until this operation is undone
    archived        INTEGER NOT NULL DEFAULT 0  -- set to 1 after 90 days
);

CREATE INDEX idx_undo_log_file_id ON undo_log(file_id);
CREATE INDEX idx_undo_log_group_id ON undo_log(group_id) WHERE group_id IS NOT NULL;
CREATE INDEX idx_undo_log_performed_at ON undo_log(performed_at);
CREATE INDEX idx_undo_log_undone ON undo_log(undone_at) WHERE undone_at IS NOT NULL;
```

**Why append-only?** The undo log is the ground truth for what Curator has done. Deleting records would make it impossible to audit or recover from edge cases. After 90 days, records are `archived = 1` and excluded from the active undo UI, but they remain in the database indefinitely.

**SHA-256 before/after:** Both hashes are recorded. `sha256_after` should always equal `sha256_before` (move operations do not modify content). If they differ, it indicates filesystem corruption or interference. This mismatch is reported as a critical error.

### 7.6 Crash Recovery Protocol

On startup, Curator runs:

```
SELECT * FROM staged_files WHERE wal_intent IS NOT NULL;
```

For each row:
- `wal_intent = 'stage_pending'`: file was being moved TO Review Hub. Check if file is at `staged_path`. If yes, clear `wal_intent`, update `file_state = 'staged_review'`. If no, check `original_path` — file is still at source, clear `wal_intent`, reset `file_state` to `grouped`.
- `wal_intent = 'stage_complete'`: file arrived at `staged_path`. Verify with `stat()`. Clear `wal_intent`.
- `wal_intent = 'commit_pending'`: commit was in progress. Check if file is at `committed_path`. If yes, complete the commit record. If no, file is still at `staged_path`, revert `user_action` to `approved`.
- `wal_intent = 'restore_pending'`: restore was in progress. Check if file is at `restore_path`. If yes, complete the restore. If no, file is still at `staged_path`, mark `wal_intent = NULL` and present to user as "restore interrupted."

**Curator decision:** Crash recovery runs before the main window appears. A progress indicator ("Verifying file integrity...") is shown. If any discrepancy is found, the user sees a non-dismissable alert: "Curator recovered N files after an unexpected quit. Review the recovery log."

---

## 8. Spotlight & FSEvents Management

### 8.1 Spotlight Exclusion

Spotlight maintains a privacy list of folders it does not index (`mdutil -i off [path]`). This can be managed programmatically via:

```swift
// Add to Spotlight privacy list
let url = URL(fileURLWithPath: hubPath)
CSIndexingStatusChanged(url as CFURL, false)
```

However, the most robust method is placing a `.metadata_never_index` file in the root of `_Curator Review/`. This is the same mechanism Apple uses for `/tmp`, Xcode's DerivedData, and other transient directories. The file's presence causes `mds` (the Spotlight daemon) to skip the folder entirely on its next scan.

**Why exclusion matters:** Staging 1,000 files would otherwise trigger 1,000 Spotlight index operations (one per file move into the folder). On a MacBook Air M2, `mds` consumes approximately 40-100ms of CPU per indexed document for content extraction. For 1,000 documents this represents 40-100 CPU-seconds, contending with the main Curator pipeline. Spotlight exclusion reduces this to near-zero.

**Curator decision:** Place `.metadata_never_index` in `_Curator Review/` at hub creation time. Additionally call `mdutil -i off [hubPath]` via `NSTask` at first launch to ensure the privacy list is updated immediately (the sentinel file takes effect on the next `mds` scan, which may be delayed by minutes).

### 8.2 FSEvents During Staging

FSEvents is the macOS kernel-level event notification system. Every `rename()` generates an `kFSEventStreamEventFlagItemRenamed` event. For 1,000 files, this is 2,000 events (one for source removal, one for destination addition).

**FSEvents coalescing:** The FSEvents framework coalesces events that arrive within the same 100ms window (the `latency` parameter of `FSEventStreamCreate`). If Curator moves 100 files within 100ms (approximately 1ms per `rename()` on an M-series Mac's NVMe SSD), all 100 events are delivered as a single batch notification to registered watchers. The coalescing flag `kFSEventStreamCreateFlagNoDefer` disables this (immediate delivery); Curator should NOT use this flag to ensure coalescing happens.

**Curator's own FSEvents watcher:** Curator watches the user's filesystem for external changes. The watcher must ignore events originating from within `_Curator Review/` to avoid processing its own staging operations. This is done by path-prefix filtering: if the event path starts with `hubPath`, discard it.

**External modification detection:** If the user drags a file out of `_Curator Review/` directly in Finder, FSEvents delivers a `kFSEventStreamEventFlagItemRenamed` event with the file's path. Curator detects this and marks the file as `user_corrected` in the DB.

**Curator decision:** Set FSEvents `latency` to 0.5 seconds during bulk staging operations. This maximizes coalescing. Immediately before staging begins, Curator registers an internal "staging in progress" flag; the event handler discards events from `hubPath` while this flag is set. The flag is cleared after staging completes.

### 8.3 Batch Move Strategy

For bulk staging (e.g., 500 files being moved to the Review Hub in one pipeline run):

1. Create all group subfolders first (single `mkdir` calls, no FSEvents noise beyond the folder creation)
2. Move files in batches of 50 using a dispatch queue with `qos: .utility` (below `userInitiated`, above `background`)
3. Between batches, yield the main thread for 10ms to keep the UI responsive
4. Update the DB in a single SQLite transaction per batch (not per file)

This strategy moves 500 files in approximately 10 seconds on typical hardware while consuming <20% CPU.

**`NSFileCoordinator` consideration:** `NSFileCoordinator` is used when multiple processes (or the app and an extension) need coordinated access to a file. For Curator staging files that it fully owns, `NSFileCoordinator` is not needed and adds unnecessary overhead (~5ms per file). However, for iCloud-synced files, `NSFileCoordinator` is mandatory (see Section 10).

**Curator decision:** Use direct `rename()` system calls (via Swift's `FileManager.moveItem(at:to:)`) for non-iCloud files. Use `NSFileCoordinator` only for iCloud-coordinated files. Batch size of 50 files per SQLite transaction.

### 8.4 macOS Sandboxing Implications

If Curator runs in the macOS App Sandbox, it cannot move files to arbitrary locations without user-granted access. Required entitlements:

- `com.apple.security.files.user-selected.read-write` — for files the user has opened or selected
- `com.apple.security.files.downloads.read-write` — for the Downloads folder specifically
- **For arbitrary paths:** `com.apple.security.temporary-exception.files.absolute-path.read-write` with the specific paths, which Apple rarely grants in App Store review

**Practical implication:** Curator must use `NSOpenPanel` to request access to the user's home folder (or the specific watch folders). This access token is stored in a security-scoped bookmark (`URL.bookmarkData(options: .withSecurityScope)`). The bookmark must be accessed with `url.startAccessingSecurityScopedResource()` before any file operation.

**Curator decision:** Curator requests access to the user's home folder via `NSOpenPanel` at first launch. This is a one-time permission. The security-scoped bookmark is stored in `UserDefaults` and renewed annually or when it becomes stale. This is the same pattern used by BBEdit, Codepoint, and other power-user apps that need broad filesystem access.

---

## 9. Finder Integration

### 9.1 Custom Folder Icons

`NSWorkspace.setIcon(_:forFile:options:)` sets a custom icon on any file or folder. The icon is stored in the item's resource fork (specifically, in the `kIconFileCreator` field of the Finder info). This survives `cp -p`, iCloud sync to other Macs with macOS 13+, and Time Machine.

Curator sets icons on group subfolders:
- `duplicate_exact`: red folder icon (signifies urgent/multiple copies)
- `duplicate_near`: orange folder icon
- `version_family`: purple folder icon
- `new_context`: blue folder icon
- `mixed_folder`: yellow folder icon
- `unknown`: gray folder icon
- `failed_to_read`: gray-with-X folder icon
- `approved` (ready to commit): green folder icon

The icons are PNG assets bundled with the Curator app, rendered at 512×512 for Retina displays. The icon is set immediately when the group subfolder is created.

**Limitation:** Custom icons do not propagate via AirDrop and are stripped by some ZIP utilities. Since `_Curator Review/` is local-only (by design), this is acceptable.

**Curator decision:** Set custom folder icons on all group subfolders. Use a consistent color vocabulary (red=urgent, green=ready, blue=new, gray=unknown) that matches the Finder tag colors for redundancy.

### 9.2 Finder Color Tags

Finder tags are stored in the file's extended attribute `com.apple.metadata:_kMDItemUserTags`. Setting them programmatically:

```swift
let tags = ["Curator-Review-Pending"]
try (url as NSURL).setResourceValue(tags, forKey: .tagNamesKey)
```

Predefined Finder colors map to specific tag names: `Red`, `Orange`, `Yellow`, `Green`, `Blue`, `Purple`, `Gray`. Curator uses the same color vocabulary as the custom icons.

Tags are set on the group subfolder, not on individual files inside it. This keeps the visual noise low — the user sees one orange dot on the "thesis - 3 versions" folder, not orange dots on 3 individual files.

**Curator decision:** Apply Finder color tags to group subfolders, not individual files. Update the tag when the group's `user_action` changes (e.g., to Green when approved).

### 9.3 Sidecar `.curator_group.json` Files

Each group subfolder contains a `.curator_group.json` file (hidden because it starts with `.`). This file is the Finder-friendly communication artifact — it allows the user to understand what Curator did even if they're looking at the folder in Terminal or a non-Curator file manager.

```json
{
  "curator_version": "1.0",
  "group_id": "uuid-here",
  "group_name": "thesis - 3 versions",
  "group_type": "version_family",
  "reason": "These 3 files are near-identical versions of the same document.",
  "member_count": 3,
  "created_at": "2026-06-08T14:23:00Z",
  "original_paths": [
    "/Users/user/Documents/thesis_v1.pdf",
    "/Users/user/Desktop/thesis_v2.pdf",
    "/Users/user/Downloads/thesis_final.pdf"
  ],
  "curator_suggestion": "Keep thesis_final.pdf, delete the others.",
  "user_action": "pending"
}
```

The `.curator_group.json` is regenerated from the SQLite DB on every pipeline run and whenever a group's state changes. It is read-only from the filesystem perspective — Curator never reads it as a data source (the DB is the source of truth). If the user edits it manually, the changes are ignored.

**Curator decision:** Generate `.curator_group.json` in every group subfolder. The file is the group's "README" for Finder. It is regenerated, not read, so it cannot become the source of truth.

### 9.4 Hub-Level `.curator_hub.json`

At the root of `_Curator Review/`, a `.curator_hub.json` file provides summary information:

```json
{
  "curator_version": "1.0",
  "hub_created_at": "2026-06-08T10:00:00Z",
  "last_pipeline_run": "2026-06-08T14:00:00Z",
  "total_groups": 47,
  "pending_groups": 31,
  "total_staged_files": 312,
  "total_size_bytes": 2147483648
}
```

This allows a user to quickly inspect the Review Hub state from Terminal without opening the Curator app.

### 9.5 What Hazel and Default Folder X Teach Us

**Hazel** (Noodlesoft) communicates with the user entirely via the filesystem: it moves files and sets Finder labels. It does not modify `.DS_Store`. Hazel's undo is limited — it logs moves to its own database and can undo the last N operations. Curator inherits: filesystem-as-communication, color tags for status.

**Default Folder X** enhances the Open/Save dialog by annotating folders with usage frequency indicators. It stores metadata in `~/Library/Application Support/Default Folder X/`. It does not modify the folders themselves. Curator inherits: metadata lives in `~/Library/Application Support/`, not in the watched folders.

**Curator decision:** Curator stores its database at `~/Library/Application Support/Curator/curator.db`, not inside `_Curator Review/`. The Review Hub contains only staged files and read-only sidecar JSON files.

---

## 10. iCloud & Locked Volume Handling

### 10.1 iCloud-Evicted Files

iCloud Drive can evict local copies of files to save disk space, leaving a stub (a `.icloud` placeholder file, e.g., `.thesis.pdf.icloud`). The original file is retrievable but requires a download.

Curator must detect evicted files before attempting to stage them:

```swift
let resourceValues = try url.resourceValues(forKeys: [.ubiquitousItemIsDownloadedKey, .ubiquitousItemDownloadingStatusKey])
if resourceValues.ubiquitousItemDownloadingStatus == .notDownloaded {
    // Trigger download
    try FileManager.default.startDownloadingUbiquitousItem(at: url)
    // Wait for download via NSMetadataQuery watching NSMetadataUbiquitousItemDownloadingStatusKey
}
```

The download can take seconds to minutes depending on file size and network speed. Curator handles this asynchronously:

1. File enters `staged_review` with `wal_intent = 'icloud_download_pending'`
2. A background task monitors the download via `NSMetadataQuery`
3. When download completes, the staging `rename()` executes
4. `wal_intent` is cleared

If the download fails (no network, file deleted from iCloud), the file transitions to `failed_to_read` with `failure_code = 'icloud_unavailable'`.

**`NSFileCoordinator` for iCloud files:** Apple's documentation requires `NSFileCoordinator` for any read or write to iCloud-managed files. The coordinator serializes access with iCloud's `ubiquityd` daemon, preventing torn reads during sync operations.

**Curator decision:** All iCloud file operations use `NSFileCoordinator`. Non-iCloud files do not. iCloud downloads are triggered asynchronously; the user sees a "Downloading from iCloud..." progress indicator per file.

### 10.2 The Review Hub Should Not Be in iCloud Drive

If the user has placed their `Documents` or `Desktop` folder in iCloud Drive (the "Desktop & Documents Folders" feature), `_Curator Review/` must NOT be created there. Reasons:

1. iCloud would attempt to sync all staged files to iCloud — consuming bandwidth and storage quota
2. iCloud strips certain extended attributes during sync, potentially corrupting Curator's xattr-based metadata
3. iCloud file coordination adds ~10ms latency per file operation, multiplied by 1,000 files = 10 seconds of added latency
4. iCloud can evict staged files back to stubs, breaking the staging guarantee

**Curator decision:** `_Curator Review/` is always created at `~/Library/Application Support/Curator/Review Hub/` — a location guaranteed to be outside iCloud Drive regardless of the user's iCloud settings. A symbolic link at `~/Desktop/_Curator Review` (or the user's chosen shortcut location) points to the real hub for Finder accessibility.

The symlink approach here is the inverse of the general case: the hub lives locally, and a Finder-visible alias points to it. This is safe because:
- The symlink is not a file being staged (it is a navigation shortcut)
- Curator never follows the symlink programmatically (it always uses the real path)
- If the symlink is deleted by the user, the hub is unaffected

### 10.3 Read-Only Volumes

If a file is on a read-only volume (external drive mounted read-only, DMG image, network share without write permission), `rename()` fails with `EACCES` or `EROFS`. Curator cannot stage it.

Behavior:
1. Detect `EROFS` / `EACCES` on attempted move
2. Do NOT copy the file (a read-only file may be intentionally read-only; creating a copy without explicit user consent violates the "no unilateral action" principle)
3. Mark the file as `failed_to_read` with `failure_code = 'read_only_volume'`
4. Report to user: "This file is on a read-only volume and cannot be staged. To review it, mount the volume with write access."

**Curator decision:** Never silently copy a file from a read-only volume. The user must explicitly request the copy. A read-only-volume file is excluded from normal staging and reported in the "Failed to Read / Protected" section.

---

## 11. Prior Art

### 11.1 Git Staging Area

Git's staging area (the index) is physically a file at `.git/index`. When `git add` is run, the file content is copied into `.git/objects/` as a blob object (addressed by SHA-1/SHA-256 of content), and the index is updated to point to that blob. The working tree file is NOT moved — it remains in place. The index is a snapshot of what the next commit will contain.

**What Curator takes from Git:**
- The write-ahead log pattern: Git writes the index atomically (lock file → write → rename over old index)
- SHA-256 for content addressing: Curator uses SHA-256 to detect if a file has been externally modified since staging
- The concept of "staging" as a separate step between "in pipeline" and "committed"

**What Curator does NOT take from Git:**
- Git does not move files — it copies content into objects. Curator moves files physically. The reasons are different: Git needs a snapshot at commit time; Curator needs to physically remove files from their source locations to make the staging visible to the user.

### 11.2 macOS Trash

`~/.Trash/` (per-user) and `/.Trashes/[UID]/` (per-volume root) are the macOS Trash implementation. When a file is trashed:

1. If source and Trash are on the same volume: `rename()` to `~/.Trash/[filename]`. Atomic, fast.
2. If source and Trash are on different volumes: no — macOS creates per-volume Trash at `/.Trashes/[UID]/`. The Trash is always on the same volume as the source. Cross-volume move to Trash is not implemented in the kernel; Finder handles it by moving to the volume-local Trash.

Original path tracking: macOS Trash records original paths in an opaque plist inside the Trash folder (`.Trashes` metadata). This is not public API. Prior to macOS 10.15, Finder used `.DS_Store` within the Trash to store original paths; since 10.15 the mechanism is internal.

**What Curator takes from macOS Trash:**
- Same-volume-Trash pattern: `_Curator Review/` should ideally be on the same volume as the staged files. Since Curator places the hub at `~/Library/Application Support/Curator/Review Hub/`, cross-volume staging is possible for external drives. The hub location should allow per-volume satellite hubs for external drives (see Open Questions).
- The principle of recording `original_path`: Finder has always done this for Trash, even when the API was not public. Curator makes it explicit in the DB.

### 11.3 DEVONthink Inbox

DEVONthink (DEVONtechnologies) has a physical inbox folder at `~/Dropbox/DEVONthink Inbox/` or a user-specified path. Files dropped into the inbox are imported into the DEVONthink database, which moves them to its internal storage (`~/Library/Application Support/DEVONthink 3/`). The original path is preserved in the document's metadata. DEVONthink's inbox is the closest prior art to Curator's Review Hub.

**Key observations:**
- DEVONthink uses SQLite for all metadata (the `.dtBase2` package is a SQLite database)
- DEVONthink preserves `original_path` as a URL in its `ZPATH` column
- DEVONthink does NOT create per-group subfolders — everything in the inbox is flat. Groups are virtual (Smart Groups), not physical.
- DEVONthink does NOT support restore: once imported, the file is in DEVONthink's storage permanently (unless the user exports it)

**What Curator takes from DEVONthink:**
- SQLite as the metadata store
- Preserving original path as a first-class metadata field
- The inbox/staging → classify → commit workflow

**What Curator does differently:**
- Curator creates physical group subfolders (context-first, not flat)
- Curator supports restore (DEVONthink does not)
- Curator is Finder-native (the hub is a regular folder, not a `.dtBase2` package)

### 11.4 Hazel

Hazel (Noodlesoft) watches folders and applies user-defined rules to move or tag files. Its "undo" capability is limited: it logs moves to a proprietary database and can undo the last operation for a rule. It does not have a staging area — files are moved to their final destination immediately.

**What Curator takes from Hazel:**
- Finder color tags as a lightweight status communication mechanism
- The principle that moved files should have their destinations recorded for potential undo
- File watching via FSEvents with coalescing

**What Curator does differently:**
- Curator stages first (Review Hub), then commits. Hazel commits immediately.
- Curator has group-level decisions; Hazel is per-file.
- Curator's undo log is permanent; Hazel's is ephemeral.

### 11.5 Email "Junk" Folder

Mail.app's Junk folder is a staging area for messages the spam filter is confident about. The user can review the Junk folder and restore messages to Inbox (equivalent to Curator restore) or allow permanent deletion (equivalent to Curator "ignored"). Key design elements:
- Staged items are shown with a visual indicator (the yellow exclamation triangle in Mail's Junk)
- The staging period has a time limit (Junk is auto-deleted after 30 days)
- The user can restore individual items without affecting others (partial restore)
- The system can be wrong (false positives) — the staging period protects against irreversible errors

**What Curator takes from Mail.app Junk:**
- A time-bounded staging period is a good design safety net (but Curator should not auto-delete without explicit user configuration)
- Visual indicators of "why this is here" (Curator: the `.curator_group.json` and color tags)
- Individual-item restore from a group

---

## 12. Design Decisions

**DD-1: Physical move, not alias.** Files in the Review Hub are physically present there. `original_path` is recorded in the DB. Aliases and symlinks are ruled out.

**DD-2: `_Curator Review/` lives at `~/Library/Application Support/Curator/Review Hub/`.** A Finder-visible alias is placed at a user-configured location (default: `~/Desktop/_Curator Review`). This ensures the hub is never accidentally placed in iCloud Drive.

**DD-3: `rename()` for same-volume moves; copy-verify-delete for cross-volume.** Same-volume `rename()` is atomic. Cross-volume moves use SHA-256 verification before source deletion. The WAL records intent before any file operation.

**DD-4: Write-ahead log via `wal_intent` column.** Before any file operation, the intended new state is written to the DB and `fsync()`ed. After the operation, `wal_intent` is cleared. On startup, all non-NULL `wal_intent` records trigger recovery.

**DD-5: SQLite is the single source of truth.** xattrs, `.DS_Store`, and sidecar JSON files are read-only Finder communication artifacts. They are regenerated from the DB and never read back.

**DD-6: 3-level maximum folder depth.** `_Curator Review / [category] / [subcategory] / [group folder]`. Files live at level 3 (or level 2 for flat categories). No deeper nesting.

**DD-7: LCS-based family naming with numeric collision disambiguation.** Family names derived from Longest Common Subsequence of member filenames. Collisions get ` (2)`, ` (3)` suffixes.

**DD-8: `.metadata_never_index` + `mdutil -i off` for Spotlight exclusion.** Both mechanisms applied at hub creation time. Eliminates `mds` contention during bulk staging.

**DD-9: FSEvents `latency = 0.5s` during staging; path-prefix filter for hub events.** Maximizes event coalescing. Curator ignores FSEvents from within `hubPath` while staging is in progress.

**DD-10: Commit is group-atomic with per-file rollback.** If any file in a group fails to commit, all completed moves in that commit attempt are rolled back. The user sees exactly which file failed and why.

**DD-11: Destination collision default is Rename.** Silent overwrite is never the default. Collision → pause for user input → options: Rename / Skip / Overwrite. Batch mode default: Rename with `_curator_N` suffix.

**DD-12: Undo log is append-only, retained indefinitely.** Records are soft-archived after 90 days (`archived = 1`) but never deleted. The undo log is the audit trail for all Curator file operations.

**DD-13: iCloud files use `NSFileCoordinator`; non-iCloud files do not.** Detect iCloud status via `.ubiquitousItemDownloadingStatusKey`. Trigger download asynchronously. Stage only after download completes.

**DD-14: Read-only volume files are not silently copied.** They go to `Failed to Read / Protected` with `failure_code = 'read_only_volume'`. The user must explicitly request a copy.

**DD-15: Finder color tags on group subfolders.** Tags reflect group status (orange=duplicate, blue=new context, green=ready to commit, red=failed). Updated when `user_action` changes. Tags are on the folder, not individual files.

**DD-16: `.curator_group.json` in every group subfolder.** Human-readable summary of the group. Generated from DB, never read back as a data source. Hidden (dot-prefixed) to avoid clutter in Finder's default view.

**DD-17: No `Locked/` subfolder.** Locked files never enter the state machine (R1 decision). Locked = excluded before any DB record is created.

**DD-18: Partial commit and partial restore are supported.** File-level decisions are tracked independently of group-level decisions. A group stays `pending` until all members are resolved.

**DD-19: Missing original directory on restore → default to Desktop.** The UI clearly states the original directory is gone. Desktop is the universally accessible fallback. The user can specify an alternative.

**DD-20: Crash recovery runs before the main window appears.** All `wal_intent IS NOT NULL` records are resolved on startup. The user sees a "Verifying file integrity..." indicator. Any discrepancy triggers a non-dismissable recovery report.

---

## 13. Open Questions

**OQ-1: Per-volume satellite hubs for external drives.**
Currently, cross-volume staging requires copy-verify-delete. A more elegant solution would be creating `_Curator Review/` on each mounted volume (as macOS Trash does with `/.Trashes/[UID]/`). This would make all staging same-volume and atomic. Trade-off: the user would need to check multiple Review Hubs, or Curator must aggregate them into a unified view. Is per-volume staging worth the complexity? No decision yet.

**OQ-2: Maximum staging period.**
Should Curator auto-restore files that have been in `staged_review` for more than N days without user action? Mail.app Junk auto-deletes after 30 days. Curator should not auto-delete, but auto-restore (move back to `original_path`) after 60-90 days might prevent files from being "forgotten" in the Review Hub. The risk: the user was intentionally deferring the decision. Needs user research.

**OQ-3: Group name user-editability.**
Should users be able to rename groups in the UI? If yes, the auto-generated name in `group_name` is overwritten. If the user renames and then the group is re-processed (e.g., after a split), does the user's name persist or is it regenerated? The interaction between auto-naming and user-renaming needs a clear policy. Candidate: user edits set a `name_locked = 1` flag; Curator never overwrites a locked name.

**OQ-4: Staging during active file edits.**
If a file is currently open in another application (e.g., a Word document being edited), Curator's `rename()` will succeed (the file is moved), but the editing application now has an open file handle to a path that no longer exists. On macOS, open file handles remain valid after `rename()` (the file's inode is unchanged). The editing app can still save to the old path... which now doesn't exist (the `rename()` removed the directory entry). The app would get `ENOENT` on next save. Solution: detect if a file has open file descriptors before staging (`lsof` equivalent via `proc_pidinfo`). If open, defer staging and show a warning: "File is open in [app]. Staging deferred until the file is closed." No decision yet on the exact detection mechanism.

**OQ-5: Handling very large groups (1,000+ files).**
If a single folder has very low Folder Coherence Score and contains 2,000 files, staging all 2,000 to `Mixed Folders/[Folder Name] - Incoherent/` creates a single flat directory with 2,000 entries. macOS Finder handles large directories, but they are slow to render. Should very large mixed-folder groups be sub-partitioned? If so, by what criterion? No decision yet.

**OQ-6: Curator database backup.**
The SQLite database at `~/Library/Application Support/Curator/curator.db` is the single source of truth. If it is corrupted or deleted, `original_path` information for all staged files is lost (the files are still in Review Hub, but Curator does not know where they came from). Should the database be replicated? Should each staged file carry its `original_path` in an xattr as a fallback, even though xattrs are fragile? Trade-off: xattr redundancy vs. xattr unreliability. No decision yet.
