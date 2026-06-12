# R8 — Activity Log & Undo Model

> **Research date:** 2026-06-08
> **Depends on:** R1 (file states), R6 (staging/commit semantics)
> **Core question:** What event schema captures every Curator action with enough detail to show the user exactly what happened and why, undo any action, debug problems, and build trust through transparency?

---

## 0. Guiding Principle

> "If Curator touches a file, the user can see exactly what happened and undo it."

Every section of R8 is subordinate to this principle. The Activity log is not a debugging tool — it is the user's primary trust surface. A user who cannot understand what Curator did will not trust Curator. A user who cannot undo Curator will not use Curator.

---

## 1. Event Schema Design

### 1.1 Prior Art Survey

**Git reflog**
Git records every HEAD change as a reflog entry: timestamp, author, message, previous SHA, new SHA. Key insight: it logs *intent* ("checkout", "commit", "merge"), not just the raw state change. The human-readable message is as important as the SHA pointers. Curator should follow the same pattern: log the semantic action type, not just the file system delta.

Git's approach to non-destructive logging: the reflog never lies, even when you force-push or reset. It is an append-only log that makes "disaster recovery" routine. Curator should match this: the activity log is append-only; undos are recorded as new events, not as deletions of old events.

**macOS FSEvents**
FSEvents records: `kFSEventStreamEventFlagItemCreated`, `ItemRemoved`, `ItemRenamed`, `ItemModified`, `ItemIsFile`, `ItemIsDir`, with path, inode, and `FSEventStreamEventId` (monotonically increasing). FSEvents does NOT record who caused the change or why. This is exactly the gap Curator fills: FSEvents tells you *what* changed; Curator tells you *why*.

FSEvents inode tracking is critical for crash recovery (see §6): Curator can detect that a file moved by comparing inode across paths.

**GDPR audit logs (Article 30, GDPR)**
GDPR requires processors to record: what data was processed, the purpose, categories of data subjects, recipients, retention periods, and security measures. For Curator (a local app processing local files), GDPR does not apply directly, but its discipline is valuable. Key insight: log the *purpose* of every action, not just the action itself. Curator's `reason` field implements this.

GDPR right to erasure (Article 17) is relevant to user-controlled log deletion (§5).

**Database WAL (Write-Ahead Log)**
SQLite WAL mode: before modifying a page, write the new content to the WAL file first. On crash: if WAL contains a complete transaction, replay it; if incomplete, discard it. The page file is never in an inconsistent state.

Curator adaptation: before moving a file, write the event record with `status = 'pending'` to SQLite. After confirming the move succeeded (file exists at destination, original gone or copied), update `status = 'complete'`. On startup: any `status = 'pending'` events are crash-recovery candidates (§6).

**Event sourcing pattern**
Event sourcing: the canonical state is the log of events. Current state is derived by replaying events. Used in financial systems (every transaction is an event; balance = sum of all transactions).

Is this overkill for Curator? Partially. Curator does not need to rebuild its entire file graph by replaying events — that would be slow and complex. But the *undo system* is essentially event sourcing: to undo action A, read event A and reverse it. Curator adopts event sourcing selectively: the activity log is the source of truth for undo, but the file state database (R1) is maintained independently as a materialized view.

**Dropbox file history**
Dropbox records per-file: version ID, modified timestamp, modifier identity, size. Supports point-in-time restore. Key insight: they store the *before* state, not just the *after* state. Curator must record `source_path` (before) for every move to enable undo.

**NSUndoManager (macOS)**
NSUndoManager maintains two stacks: undo stack and redo stack. Each registered action is a closure that performs the inverse operation. Calling `undo()` pops the undo stack, executes the inverse closure, and pushes the inverse onto the redo stack.

NSUndoManager supports grouping: `beginUndoGrouping()` / `endUndoGrouping()` — a group counts as one undo step. This maps directly to Curator's group actions: committing a 50-file group is one undo step.

NSUndoManager does NOT support selective undo (undoing action N without undoing N+1 through current). For selective undo, a custom implementation is needed (§2.3).

---

### 1.2 Event Types

| Event Type | Description | Reversible |
|---|---|---|
| `staging_move` | File moved from original path to `_Curator Review/...` | Yes |
| `commit_move` | File moved from `_Curator Review/...` to final destination | Yes |
| `auto_commit` | File moved directly to destination (high confidence, no staging) | Yes |
| `group_approve` | Group metadata approved (no file move) | Yes (reverts to unapproved) |
| `group_rename` | Group label changed | Yes |
| `group_split` | One group split into two | Yes (re-merge) |
| `group_merge` | Two groups merged | Yes (re-split) |
| `duplicate_keep` | One copy of a duplicate marked as canonical | Yes |
| `duplicate_trash` | Duplicate copy moved to Trash | Conditional* |
| `lock` | File or folder marked as ignored | Yes |
| `unlock` | Lock removed | Yes |
| `rule_created` | Learning rule created from user correction | Yes (delete rule) |
| `rule_applied` | Learned rule applied to a file | Yes (undo rule application) |
| `rule_suspended` | Rule suspended due to repeated undo | Yes (re-enable rule) |
| `restore` | File moved back to original path | Yes |
| `undo` | Reversal of a previous event | Yes (redo) |
| `failed_read` | Curator could not read a file (permission, encoding) | N/A |
| `scan_start` | Scan session started | N/A |
| `scan_complete` | Scan session finished | N/A |
| `batch_start` | A multi-file operation began | N/A |
| `batch_complete` | A multi-file operation finished | N/A |

*`duplicate_trash`: reversible if Trash not yet emptied. Curator checks Trash on undo attempt.

---

### 1.3 SQLite Table Definitions

```sql
-- =============================================
-- CORE EVENT LOG
-- =============================================
CREATE TABLE curator_events (
    event_id          TEXT PRIMARY KEY,          -- UUID v4
    event_type        TEXT NOT NULL,             -- see event types above
    timestamp         TEXT NOT NULL,             -- ISO 8601 with TZ: '2026-06-08T14:32:11.452+03:00'
    status            TEXT NOT NULL DEFAULT 'complete',  -- 'pending' | 'complete' | 'failed' | 'undone'

    -- File references (nullable for non-file events)
    file_id           TEXT,                      -- FK → files.file_id (R1)
    source_path       TEXT,                      -- absolute path before action
    destination_path  TEXT,                      -- absolute path after action
    source_inode      INTEGER,                   -- inode at source before move (for crash recovery)
    dest_inode        INTEGER,                   -- inode at destination after move

    -- Group context
    group_id          TEXT,                      -- FK → groups.group_id
    batch_id          TEXT,                      -- groups multiple events into one user-visible action

    -- Explanation fields
    reason            TEXT,                      -- human-readable: "Co-accessed with EPL342 files"
    reason_type       TEXT,                      -- 'semantic' | 'ner' | 'provenance' | 'behavioral' | 'duplicate' | 'version' | 'rule' | 'coherence'
    confidence        REAL,                      -- 0.0–1.0
    triggered_by      TEXT,                      -- 'user_action' | 'auto_pipeline' | 'learning_rule'
    rule_id           TEXT,                      -- FK → learning_rules.rule_id (if triggered_by = 'learning_rule')

    -- Undo tracking
    reversible        INTEGER NOT NULL DEFAULT 1, -- 0 or 1 (SQLite boolean)
    irreversible_reason TEXT,                    -- why it cannot be undone (e.g., 'trash_emptied')
    undone_at         TEXT,                      -- ISO 8601 timestamp when undone
    undo_event_id     TEXT,                      -- FK → curator_events.event_id (the undo event)
    redo_event_id     TEXT,                      -- FK → curator_events.event_id (the redo event)

    -- Metadata
    app_version       TEXT,                      -- Curator version that created this event
    session_id        TEXT,                      -- which app session (for grouping in Activity view)

    FOREIGN KEY (undo_event_id) REFERENCES curator_events(event_id),
    FOREIGN KEY (redo_event_id) REFERENCES curator_events(event_id)
);

-- =============================================
-- BATCH FILE EVENTS (for group actions with many files)
-- =============================================
CREATE TABLE curator_event_files (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    batch_id          TEXT NOT NULL,             -- FK → curator_events.batch_id
    file_id           TEXT NOT NULL,
    source_path       TEXT NOT NULL,
    destination_path  TEXT,
    source_inode      INTEGER,
    dest_inode        INTEGER,
    status            TEXT NOT NULL DEFAULT 'complete',  -- per-file status within batch
    error_message     TEXT                        -- if status = 'failed'
);

-- =============================================
-- SCAN SESSIONS
-- =============================================
CREATE TABLE scan_sessions (
    session_id        TEXT PRIMARY KEY,
    scan_type         TEXT NOT NULL,             -- 'initial_cleanup' | 'daily_autopilot' | 'folder_watch'
    started_at        TEXT NOT NULL,
    completed_at      TEXT,
    root_path         TEXT NOT NULL,
    files_scanned     INTEGER DEFAULT 0,
    files_staged      INTEGER DEFAULT 0,
    files_committed   INTEGER DEFAULT 0,
    status            TEXT NOT NULL DEFAULT 'running'  -- 'running' | 'complete' | 'interrupted'
);

-- =============================================
-- INDEXES
-- =============================================
CREATE INDEX idx_events_timestamp    ON curator_events(timestamp DESC);
CREATE INDEX idx_events_type         ON curator_events(event_type);
CREATE INDEX idx_events_file_id      ON curator_events(file_id);
CREATE INDEX idx_events_group_id     ON curator_events(group_id);
CREATE INDEX idx_events_batch_id     ON curator_events(batch_id);
CREATE INDEX idx_events_status       ON curator_events(status);
CREATE INDEX idx_events_rule_id      ON curator_events(rule_id);
CREATE INDEX idx_event_files_batch   ON curator_event_files(batch_id);
CREATE INDEX idx_event_files_file    ON curator_event_files(file_id);
```

**Notes on the schema:**

- `batch_id` links the parent `curator_events` row (the batch_start/batch_complete bookend) to `curator_event_files` rows for each file in a large group operation. This avoids a 50-column array in a single row.
- `status = 'pending'` is the WAL signal: written before the file operation, updated to `'complete'` after verification. On startup, any `'pending'` events trigger crash recovery (§6).
- `source_inode` enables detecting "this file moved elsewhere outside Curator" — even if the path changed, the inode matches.
- `app_version` allows future schema migrations and bug analysis by version.
- `session_id` links events to a scan session, enabling "what did Curator do in today's autopilot run?" queries.

**Curator decision:** Use the schema above verbatim. Enable SQLite WAL mode and journal_mode=WAL for all tables. Enable SQLite STRICT mode to catch type errors early.

---

## 2. Undo Model

### 2.1 Linear Undo (Cmd+Z)

Standard Cmd+Z undoes the most recent reversible action. Implementation:

```
Undo stack: [event_1, event_2, event_3, event_4] ← top (most recent)
Redo stack: []

User presses Cmd+Z:
  1. Pop event_4 from undo stack
  2. Execute inverse of event_4
  3. Record new 'undo' event (event_5) in activity log
  4. Push event_5 onto redo stack
  5. Mark event_4.undone_at = now, event_4.undo_event_id = event_5.event_id
```

The undo stack is not persisted in memory across app sessions — it is reconstructed from the activity log. On app launch: query `curator_events WHERE status != 'undone' ORDER BY timestamp DESC` to rebuild the undo stack.

**NSUndoManager compatibility:** Curator does not use NSUndoManager directly because NSUndoManager's undo stack is in-memory only and does not survive app restarts. Curator's undo stack must survive restarts (a file organized yesterday must be undoable today). Curator implements its own undo stack backed by the SQLite activity log.

Curator does register with NSUndoManager for system Cmd+Z integration, but the actual undo logic is Curator's own. The NSUndoManager registration triggers Curator's internal undo, not a closure.

**Curator decision:** Linear undo via Cmd+Z is the primary undo interface. The undo stack is SQLite-backed, not in-memory. NSUndoManager is used only for menu bar integration.

---

### 2.2 Group Undo

A group action (commit of 50 files) is one undo step. Implementation:

- All 50 file moves share a `batch_id`
- The parent `curator_events` row (type: `batch_complete`) is what sits on the undo stack
- Undoing the parent event triggers reversal of all 50 `curator_event_files` rows

**Atomicity problem:** What if the user deleted 3 of the 50 files after the commit? Curator cannot restore files that no longer exist at the committed path.

Resolution:
1. Before executing group undo, Curator checks each file's current path (using inode if available, path otherwise)
2. Files that cannot be found are listed in a pre-undo confirmation dialog: "3 files cannot be restored (no longer at committed path). Undo the other 47?"
3. User confirms. Undo proceeds for the 47 recoverable files.
4. The undo event records `partial: true` and lists the 3 unrestorable file_ids.

**Cursor decision:** group undo should be atomic where possible, partial with explicit user confirmation where files are missing.

**Curator decision:** Group undo is one Cmd+Z step. Partial undo (when some files are gone) requires explicit user confirmation. The undo event records which files were and were not restored.

---

### 2.3 Selective Undo

**Theory:** Berlage & Genau (1993, "A System for Retrospective Brushing") demonstrated selective undo: undo any past action without undoing actions that happened after it. The core problem is dependency: if action B depended on action A, undoing A without undoing B leaves the system inconsistent.

For Curator's use cases:
- "I committed EPL342 group, then EPL412 group. I want to undo EPL342 without undoing EPL412." — Is this safe? EPL342 and EPL412 files are independent. No dependency. Selective undo is safe.
- "I committed group, then renamed a file in the committed group. I want to undo the commit." — Dependency exists: the rename was applied to the committed path. Undoing the commit would move the file back, but the rename is now on a file at a different path. Conflict.

Dependency detection rule: event B depends on event A if B's `source_path` equals A's `destination_path` (B operated on a file that A put there).

**Is selective undo worth the complexity?**

Myers (1988, "A New Model for Handling Input") surveyed undo models and found users rarely need selective undo in practice — they need it mostly when they regret an action several steps back. For Curator's scale (groups of files, not individual keystrokes), the more common scenario is "I committed the wrong group" which is satisfied by linear undo as long as the user acts reasonably promptly.

The stronger case for selective undo in Curator: rules. If a learned rule organized 200 files over the past week, the user may want to undo the rule's effects without undoing their own manual work from the same period. This is rule-scoped undo, not time-scoped undo (§2.5).

**Curator decision:** Selective undo is NOT implemented in v1. Linear undo covers 95% of use cases. Rule-scoped undo (undo all effects of a specific rule) IS implemented as a special case, because it has a clean dependency boundary.

---

### 2.4 Undo After Commit

A file was committed to `~/Documents/EPL342/Assignments/hw1.pdf`. The user later decides this was wrong.

Option A: Move file back to original path (`~/Downloads/hw1.pdf`).
Option B: Move file back to `_Curator Review/` for re-staging.

Analysis:
- Option A is semantically clean: it fully reverses the action. The original path was recorded at `staging_move` time.
- Option A has a problem: the original path may no longer make sense. If the original was `~/Desktop`, returning there recreates the chaos.
- Option B puts the file back into the Review Hub for the user to re-decide. This is safer but may be confusing ("why is it back in Review?").

Research on mental models: Dropbox's "restore to previous version" puts the file back in place (Option A equivalent). Time Machine restores to original location. Both use Option A. The mental model is clear: undo = reverse.

Resolution: Curator offers both, with Option A as the default and a "Re-stage instead" option in the undo confirmation. The confirmation shows: "Return `hw1.pdf` to `~/Downloads/` (original location) or move to Review Hub for re-staging?"

**Cursor decision:** Undo-after-commit defaults to restoring original path. "Re-stage" is an explicit alternative. The original path is always available in `source_path` of the `staging_move` event, which chains back through `staging_move` → `commit_move`.

**Curator decision:** Undo after commit defaults to original path restore. Destination path for the undo is read from the original `staging_move` event's `source_path` field (the pre-staging location), accessed by following the `batch_id` chain.

---

### 2.5 Undo After Learning (Rule-Scoped Undo)

A rule was created: "Files from moodle.cs.ucy.ac.cy/EPL342 → EPL342/Assignments". It ran 47 times over 2 weeks.

User realizes the rule was wrong (some files were lecture slides, not assignments). The user wants to undo the rule AND all files it organized.

Implementation:
1. User opens Activity view, finds the rule, clicks "Undo all rule applications"
2. Curator queries: `SELECT * FROM curator_events WHERE rule_id = ? AND event_type = 'rule_applied' AND status != 'undone'`
3. Curator checks each file: does it still exist at the destination? (It may have been further moved or deleted)
4. Confirmation: "47 files were organized by this rule. 44 can be restored. 3 cannot be found. Proceed?"
5. User confirms. Curator moves 44 files back to their original paths.
6. Rule is suspended (not deleted — its history is preserved).
7. A single `undo` event with `batch_id` grouping all 44 reversals is recorded.

**Curator decision:** Rule-scoped undo is a first-class operation accessible from the Rules section of Activity view. It uses the same batch undo mechanism as group undo, filtered by `rule_id`.

---

### 2.6 Non-Undoable Actions

| Scenario | Detection | Curator response |
|---|---|---|
| File moved to Trash, Trash emptied | `NSFileManager.trashItem()` returns a URL to the Trash location. On undo: check if that URL still exists. If not, Trash was emptied. | Show: "This file was moved to Trash and cannot be recovered. The Trash was emptied." Mark `reversible = 0`, `irreversible_reason = 'trash_emptied'`. |
| File deleted outside Curator | FSEvents `ItemRemoved` event on the committed path. Curator detects this and updates the file state to `externally_deleted`. | On undo attempt: "This file was deleted outside Curator and cannot be restored." |
| File moved outside Curator | FSEvents `ItemRenamed`/`ItemCreated` on a new path (same inode). Curator detects the new path. | Undo can still work: move from the new path back to original. Curator asks: "This file is now at `~/Desktop/hw1.pdf` (moved outside Curator). Return it to original location anyway?" |
| Duplicate deletion, Trash emptied | Same as Trash scenario above. | Same response. |

**NSFileManager.trashItem()** — yes, it supports recovery. The method returns `resultingItemURL` pointing to the Trash location. Files in Trash are recoverable until `NSWorkspace.shared.performSelector(for: .emptyTrash)` is called. Curator stores the `resultingItemURL` as `destination_path` for `duplicate_trash` events so undo can restore from Trash.

**Curator decision:** Before offering undo for any deletion action, Curator checks if the file exists at the recorded `destination_path`. If not, the undo button is grayed out with a tooltip explaining why. Non-undoable status is computed dynamically (not stored) because Trash state can change after the event was logged.

---

## 3. Activity View Design

### 3.1 Information Architecture

```
Activity
├── Today
│   ├── [Session: Daily Autopilot, 14:32] — 23 files organized
│   │   ├── Staged: 5 files → EPL342 context
│   │   ├── Auto-committed: 12 files (high confidence)
│   │   ├── Duplicates: 6 files → Trash
│   │   └── [Undo this session]
│   └── [Session: Manual, 16:01] — 2 groups committed
│       ├── Committed: EPL342 Materials (47 files)
│       └── Committed: Thesis Draft (12 files)
├── Yesterday
│   └── ...
├── This Week
│   └── ...
└── Older
    └── ...

Filter bar: [All] [Staging] [Commits] [Auto-commits] [Duplicates] [Learning] [Failed]
```

**Grouping hierarchy:**
1. Day (primary grouping — matches mental model of "what did Curator do today?")
2. Session within day (each app launch or autopilot run is a session)
3. Batch within session (group commit, duplicate scan, etc.)

**Filtering:** The filter bar is persistent. Selecting "Learning" shows only rule events across all time. Useful for auditing what Curator has learned.

**Search:** Full-text search over `reason`, `source_path`, `destination_path`, file name. SQLite FTS5.

### 3.2 Per-Event Display

```
┌─────────────────────────────────────────────────────────────────┐
│  [📄 pdf icon]  hw1.pdf                                         │
│  Staged to Review Hub                                           │
│  → _Curator Review/New Contexts/EPL342/                         │
│  "Co-accessed with EPL342 files on 3 occasions (87% confidence)"│
│  14:32:11                                      [Undo] [Details] │
└─────────────────────────────────────────────────────────────────┘
```

Fields shown:
- File icon (using macOS Quick Look icon, matching file type)
- File name (not full path — too noisy)
- Action verb: "Staged", "Committed", "Auto-committed", "Moved to Trash", "Locked"
- Destination (short, showing folder name, not full path)
- Reason (truncated to 80 chars, expandable)
- Timestamp (relative: "2 minutes ago", absolute on hover)
- Undo button (present if `reversible = 1` and `undone_at IS NULL`)
- Details disclosure (expands to show full `source_path`, `confidence`, `triggered_by`, `rule_id`)

### 3.3 Group Action Display

```
┌─────────────────────────────────────────────────────────────────┐
│  [📁]  EPL342 Materials                                         │
│  Committed → ~/Documents/University/EPL342/                     │
│  47 files moved · 2 skipped (already at destination)           │
│  15:01:44                                [Undo Group] [Details] │
└─────────────────────────────────────────────────────────────────┘
  ▼ Show 47 files
    [📄] lecture_01.pdf  ~/Downloads/lecture_01.pdf →…
    [📄] hw1.pdf         ~/Desktop/hw1.pdf →…
    ...
```

"Show N files" is collapsed by default. Expanding reveals per-file rows (same format as per-event display, but without individual undo buttons — group undo is atomic).

Skipped files show: "2 files skipped (already at destination)" — these appear in the expanded list with a gray "Already there" badge.

### 3.4 Learning Event Display

```
┌─────────────────────────────────────────────────────────────────┐
│  [🧠]  New rule learned                                         │
│  "Files from moodle.cs.ucy.ac.cy/EPL342 → EPL342/Assignments"  │
│  Applied to 3 files immediately                                 │
│  14:33:02              [Suspend Rule] [Undo All Applications]   │
└─────────────────────────────────────────────────────────────────┘
```

Rule suspension shows:
```
┌─────────────────────────────────────────────────────────────────┐
│  [⚠️]  Rule suspended                                           │
│  "Moodle EPL342 files → EPL342/Assignments"                     │
│  Reason: 2 undo actions triggered on rule-organized files       │
│  16:22:09                      [Re-enable Rule] [Delete Rule]   │
└─────────────────────────────────────────────────────────────────┘
```

### 3.5 Design Inspirations

**Time Machine:** Date-grouped, visual timeline scrubber, "restore" as the primary action. Curator adopts date grouping and the "undo" button as the primary action per event.

**Dropbox file history:** Per-file history with restore button. Clean, minimal. Curator adopts the per-event detail drawer (source path, destination path, timestamp).

**Git clients (Tower, SourceTree):** Commit list with message, file diff disclosure, branch context. Curator adopts the batch disclosure pattern (expand a group commit to see individual file moves).

**iOS Screen Time:** Session-grouped summaries with time and counts. Curator adopts the session summary row ("Daily Autopilot — 14:32 — 23 files organized") as the top-level item.

**Curator decision:** Activity view uses day + session grouping as the default. Chronological order within each session. Filter bar persists across app sessions. Full-text search via SQLite FTS5 over paths and reasons.

---

## 4. "Why" Explanation Templates

Every Curator action must explain why it happened. The `reason` field stores the human-readable string; `reason_type` stores the machine-readable category for filtering.

### 4.1 Semantic Similarity
**Template:** "Content is [X]% similar to other [context] files"
**Example:** "Content is 87% similar to other EPL342 lecture notes (cosine similarity on FAISS embeddings)"
**reason_type:** `semantic`
**When used:** FAISS embedding similarity above threshold; primary signal for context graph placement.

### 4.2 Named Entity Recognition (NER)
**Template:** "Mentions [entity list] found in [context] files"
**Example:** "Mentions 'EPL342', 'Prof. Andreou', 'Assignment 2' — same entities as existing EPL342 files"
**reason_type:** `ner`
**When used:** NER extraction finds course codes, professor names, project names that match an existing context.

### 4.3 Provenance (Download Source)
**Template:** "Downloaded from [domain] ([path context])"
**Example:** "Downloaded from moodle.cs.ucy.ac.cy/course/EPL342 on 2026-05-14"
**reason_type:** `provenance`
**When used:** Extended attribute `com.apple.metadata:kMDItemWhereFroms` contains a recognizable URL pattern.

### 4.4 Behavioral (Co-access)
**Template:** "Opened together with [context] files on [N] occasions"
**Example:** "Opened together with EPL342 files on 3 occasions (last: 2026-06-01)"
**reason_type:** `behavioral`
**When used:** FSEvents or usage logs show this file was opened in close temporal proximity to a known-context file.

### 4.5 Exact Duplicate
**Template:** "Identical to [filename] (exact content match)"
**Example:** "Identical to `hw1_final.pdf` in EPL342/Assignments/ (SHA-256 match)"
**reason_type:** `duplicate`
**When used:** SHA-256 hash collision detected (R2).

### 4.6 Near-Duplicate / Version
**Template:** "[X]% similar to [filename] — possible earlier version"
**Example:** "94% similar to `thesis_v3.docx` — this appears to be an earlier draft"
**reason_type:** `version`
**When used:** RETSim similarity above version threshold but below duplicate threshold (R2).

### 4.7 Learned Rule
**Template:** "Learned rule applied: [rule description]"
**Example:** "Learned rule applied: 'Files from moodle.cs.ucy.ac.cy/EPL342 → EPL342/Assignments' (rule #14, applied 23 times)"
**reason_type:** `rule`
**When used:** A validated learning rule (R5) was triggered.

### 4.8 Folder Coherence
**Template:** "Folder [name] contained [N] unrelated contexts (coherence: [score])"
**Example:** "Folder 'New Folder 2' contained 6 unrelated contexts (coherence score: 0.23) — contents separated"
**reason_type:** `coherence`
**When used:** Context graph boundary detection (R4) determines that a folder is incoherent and should be split.

### 4.9 Composite Reason
When multiple signals agree, the reason combines them:
**Example:** "Content similar to EPL342 files (84%) + downloaded from moodle.cs.ucy.ac.cy/EPL342 + mentions 'EPL342'"
**reason_type:** `composite`
Implementation: store all contributing signals in a JSON field `reason_details` (array of {type, score, description}), derive the human-readable `reason` string from the top 1–3 signals.

### 4.10 Explainability Research Notes

**LIME / SHAP applicability:** LIME (Local Interpretable Model-agnostic Explanations) generates local linear approximations of complex model decisions. SHAP (SHapley Additive exPlanations) attributes each feature's contribution to a prediction. Both are designed for classification models with numerical features.

Curator's decisions are not purely numerical — they combine hash matching, NER, URL parsing, and FAISS embeddings. SHAP is applicable to the FAISS embedding component (which features in the embedding drove similarity?). However, for Curator's explanations, a simpler approach suffices: report the top contributing signal by type (provenance > NER > behavioral > semantic) with its score. Full SHAP attribution is over-engineering for v1.

**"Trust in Transparency" (arXiv:2510.04968):** The paper finds that showing reasoning increases user acceptance of AI decisions, but that overwhelming detail creates explanation fatigue. The sweet spot: one clear sentence with the strongest signal, optional expansion for details. Curator's default explanation is one sentence; the Details disclosure shows all signals.

**Spam filter analogy:** SpamAssassin shows a score breakdown: "BAYES_99 +3.5, FROM_SUSPICIOUS +1.2, RCVD_IN_XBL +0.3". Curator can adopt a similar (but less technical) breakdown in the Details drawer: "Provenance: moodle.cs.ucy.ac.cy (strong), Content similarity: 84%, Entity match: EPL342".

**Dropbox Smart Move:** Shows "Other files like this are in Finance/". Simple, contextual, no score. Curator's default one-liner follows this pattern.

**Curator decision:** Default explanation is one sentence (strongest signal). Details drawer shows all signals with scores. `reason_type` enables filtering by explanation type. `reason_details` (JSON array) stores machine-readable signal breakdown for debugging and future UI features.

---

## 5. Privacy Design

### 5.1 What the Activity Log Contains

The activity log is sensitive: it reveals what files exist, what they're about (NER entities in reasons), where they came from (provenance URLs), and behavioral patterns (co-access). For a local app, the threat model is primarily: other processes on the same machine, backups exposed to third parties, and iCloud sync.

### 5.2 Encryption

SQLCipher is a SQLite extension that encrypts the database with AES-256-CBC. The database is decrypted page-by-page in memory; the on-disk file is entirely encrypted.

Key management: the encryption key is stored in the macOS Keychain (`kSecClassGenericPassword`, `kSecAttrService: "com.curator.app"`, `kSecAttrAccessible: kSecAttrAccessibleWhenUnlockedThisDeviceOnly`). The `ThisDeviceOnly` flag prevents the key from being backed up to iCloud or migrated to a new device without explicit user consent.

**Should Curator encrypt the activity log?** Yes, for the same reason as the main database (R6 decision). The activity log contains even more sensitive information than the file state database — it includes NER entities extracted from file contents, download URLs, and behavioral patterns.

**Curator decision:** Activity log is stored in the same SQLCipher database as the main Curator database (same file, different table). Key stored in macOS Keychain with `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`.

### 5.3 Retention Policy

**How long to keep events?**

Options:
- 30 days: too short — user may want to undo a commit from last month
- 90 days: reasonable for most use cases
- 1 year: covers academic year / project cycles
- Forever: simplest, but log grows indefinitely

At the estimated event rate (§7), the activity log is ~15MB after initial scan and grows ~18MB/year. Storage is not a concern. The constraint is relevance: very old events are rarely needed.

**Proposed policy:**
- Full event detail: 1 year
- Compacted summary: forever (aggregated monthly: "March 2026: 1,247 files organized, 3 groups split, 2 rules created")
- Undo window: 90 days for staging_move events, 180 days for commit_move events (events older than their window are marked `reversible = 0` automatically but are never hard-deleted)

**Compaction:** A background job runs monthly. Events older than 1 year are summarized into a `curator_event_summaries` table and deleted from `curator_events`. Summaries are not reversible — they are historical records only.

```sql
CREATE TABLE curator_event_summaries (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    period_start  TEXT NOT NULL,   -- '2026-03-01'
    period_end    TEXT NOT NULL,   -- '2026-03-31'
    files_staged  INTEGER,
    files_committed INTEGER,
    files_auto_committed INTEGER,
    duplicates_trashed INTEGER,
    rules_created INTEGER,
    rules_applied INTEGER,
    total_events  INTEGER
);
```

### 5.4 User-Controlled Deletion

Activity settings panel:
- "Clear activity before [date picker]" — deletes all events before the selected date
- "Clear all activity" — with confirmation: "This will permanently delete all activity history. Undo history will be lost."
- "Export activity log" — exports a JSON or CSV of all events (privacy transparency feature)

GDPR right to erasure analogy: while GDPR doesn't apply to local apps, Curator should respect the same principle — the user has the right to erase their activity history at any time without consequence to the app's core function. Curator continues to work after a full log clear; it loses undo history but gains nothing from keeping logs the user wants gone.

**Curator decision:** Full event detail retained for 1 year. Undo window is 90 days. Monthly compaction summarizes old events. User can delete any portion of the log from Settings. Export to JSON/CSV available.

---

## 6. Crash Recovery

### 6.1 The Problem

Curator crashes mid-staging of 500 files. At the crash point: 200 files are in `_Curator Review`, 300 are still at their original locations, and the source originals for the 200 may or may not have been deleted. The state is inconsistent.

### 6.2 Write-Ahead Log Pattern for File Operations

Before every file operation, Curator writes a `status = 'pending'` event to SQLite. After confirming the operation succeeded, it updates `status = 'complete'`.

```
[BEFORE staging hw1.pdf]
INSERT INTO curator_events (event_id, event_type, status, source_path, destination_path, ...)
VALUES ('uuid-1', 'staging_move', 'pending', '/Users/h/Downloads/hw1.pdf', '/_Curator Review/.../hw1.pdf', ...);

[EXECUTE: copy file to _Curator Review]
[VERIFY: file exists at destination, has same size/hash]
[EXECUTE: delete original]
[VERIFY: original no longer exists at source]

UPDATE curator_events SET status = 'complete', source_inode = ..., dest_inode = ... WHERE event_id = 'uuid-1';
```

On crash at any point:
- If crash before copy: event is `pending`, original still at source. Recovery: delete the `pending` event (no file was moved).
- If crash after copy, before delete: event is `pending`, file exists at BOTH source and destination. Recovery: complete the operation (delete source) or roll back (delete destination copy). Default: complete, because the copy succeeded.
- If crash after delete, before marking `complete`: event is `pending`, file only at destination. Recovery: mark `complete` (operation succeeded, just the log update failed).

### 6.3 Two-Phase Staging

Curator NEVER uses `rename()` (atomic move) for cross-filesystem moves, and NEVER uses a single-step move. Two-phase protocol:

**Phase 1 — Copy:**
```swift
try FileManager.default.copyItem(at: sourcePath, to: reviewHubPath)
// Verify: checksum match, size match, inode recorded
```

**Phase 2 — Delete original:**
```swift
try FileManager.default.removeItem(at: sourcePath)
// Verify: source no longer exists
```

Same-filesystem moves (within same volume): APFS rename is atomic. Curator can use `moveItem(at:to:)` which maps to `renameat2()` — truly atomic, no intermediate state. The WAL record is still written before the move, but crash recovery is simpler: either the rename happened or it didn't.

Cross-filesystem moves (e.g., source on network volume, Review Hub on boot SSD): must use copy+delete. The WAL pattern above applies.

APFS copy-on-write: APFS uses copy-on-write (COW) semantics — cloning a file within the same APFS volume is near-instant and space-efficient (the clone shares blocks until modified). Curator can use `FileManager.default.copyItem` on APFS and get COW clones for free. This makes Phase 1 fast even for large files on the same volume.

### 6.4 Recovery Procedure on Startup

```
On Curator launch:
1. Open SQLite database
2. Query: SELECT * FROM curator_events WHERE status = 'pending'
3. For each pending event:
   a. Check if source_path still exists (using inode match if available)
   b. Check if destination_path exists
   c. Determine state:
      - Source exists, destination does NOT: copy failed. Roll back: delete pending event, no file change needed.
      - Source exists, destination EXISTS: copy succeeded, delete failed. Complete: delete source, mark event complete.
      - Source NOT exists, destination EXISTS: normal completion (log update failed). Mark event complete.
      - Neither exists: something external happened. Mark event as 'failed', log an error event.
4. For each 'running' scan_session:
   - Update status to 'interrupted'
   - Show user: "A scan was interrupted. Resume from where it left off?" [Resume] [Discard]
5. Present recovery summary if any pending events were found: "Curator recovered X file operations from an interrupted session."
```

**SQLite WAL mode:** Enable `PRAGMA journal_mode=WAL` and `PRAGMA synchronous=NORMAL`. WAL mode allows concurrent reads during writes, and `synchronous=NORMAL` gives a good safety/performance balance (data is safe after a WAL frame is written; only an OS crash during the WAL write itself can cause loss, which is extremely rare on APFS with battery-backed storage).

**Curator decision:** Two-phase staging (copy → verify → delete) for all cross-filesystem moves. APFS same-volume moves use `moveItem` (atomic rename). WAL event written before every operation, updated after. Recovery on startup checks all `pending` events and resolves them.

---

## 7. Storage & Retention Analysis

### 7.1 Event Count Estimates

**Initial cleanup (30,000 files):**
- `scan_start` + `scan_complete`: 2 events
- Per-file staging decisions: ~20,000 staging_move events (assume 2/3 of files are staged)
- Auto-commit events: ~5,000 (high-confidence files go direct)
- Duplicate events: ~1,000 (R2 estimates ~3-5% duplicate rate)
- Failed reads: ~100
- Total initial scan: ~26,100 events

**Daily autopilot (steady state):**
- ~50–100 new files/day entering Downloads
- ~40–80 staging_move events
- ~10–20 auto_commit events
- ~5 learning events
- Total: ~60–110 events/day → ~25,000–40,000 events/year

**After 1 year:** ~66,000 events in the full log. With compaction (events > 1 year summarized), steady state is ~40,000 active events at any time.

### 7.2 Storage Size Estimates

**Per-event size (average):**
- Fixed fields (UUIDs, timestamps, booleans): ~200 bytes
- Path fields (source + destination): ~150 bytes average
- Reason field: ~100 bytes
- Other text fields: ~50 bytes
- Total: ~500 bytes per event

**SQLite overhead:** ~30% overhead for indexes, B-tree structure, page headers. Effective per-event size: ~650 bytes.

| Scenario | Events | Raw size | With overhead |
|---|---|---|---|
| After initial scan | 26,000 | 13 MB | 17 MB |
| After 1 year | 66,000 | 33 MB | 43 MB |
| After 5 years (with compaction) | 40,000 | 20 MB | 26 MB |

These are all trivially small. A modern macOS machine has hundreds of GB of storage; 26–43 MB is negligible. **Storage is not a constraint for Curator's activity log.**

SQLite with default settings handles millions of rows without performance degradation. 66,000 rows is far below any performance concern. The indexes defined in §1.3 ensure all common queries (by timestamp, file_id, group_id, event_type) run in milliseconds.

### 7.3 Index Strategy

Queries most likely to be slow without indexes:
1. "Show all events for file X" → index on `file_id`
2. "Show all events today" → index on `timestamp DESC`
3. "Show all events for group Y" → index on `group_id`
4. "Show all pending events (crash recovery)" → index on `status`
5. "Show all events for rule Z" → index on `rule_id`

All five are covered by the indexes in §1.3. No additional indexes needed at Curator's scale.

**Curator decision:** SQLite is sufficient for all activity log storage and query needs at Curator's projected scale. No external database, no time-series database, no log aggregation service needed. The activity log is one table in the main Curator SQLite database.

---

## 8. Prior Art Summary

| System | What Curator learns |
|---|---|
| **Git reflog** | Log intent + state change. Append-only. Every undo is a new event, not a deletion. Recover from anything. |
| **macOS FSEvents** | Inode-based file tracking. Use inodes for crash recovery (detect moved files by inode). |
| **GDPR audit logs** | Log purpose of every action. User has right to erase. |
| **SQLite WAL** | Write intent before action. Mark complete after. On crash: resolve pending entries. |
| **Event sourcing** | Undo = read event + reverse. Activity log is source of truth for undo. File state DB is materialized view. |
| **NSUndoManager** | Group undo as one step. Linear stack. Does not survive restarts → Curator implements its own SQLite-backed stack. |
| **Berlage & Genau (1993)** | Selective undo theory. Dependency detection required. Too complex for v1; rule-scoped undo as practical compromise. |
| **Dropbox history** | Per-file before/after paths, restore button, clean UI. Source path always recorded. |
| **Time Machine** | Date-grouped, visual, restore = primary action. Session grouping in Activity view. |
| **Git clients (Tower)** | Batch disclosure pattern. Expand commit to see file list. Curator adopts for group commits. |
| **iOS Screen Time** | Session summary rows. Curator adopts for daily autopilot summary. |
| **APFS COW** | Same-volume file copy is fast and space-efficient. Curator uses `copyItem` for staging on APFS. |

---

## 9. Design Decisions

1. **Append-only log.** Events are never deleted or modified (except `status` updates during the same operation). Undos are new events that reference the original event. This guarantees the log is always consistent and auditable.

2. **`status = 'pending'` as WAL signal.** Written before every file operation. Updated to `'complete'` after verification. Recovery on startup resolves all `'pending'` entries. No separate WAL file needed — SQLite's own WAL mode handles atomicity of the log entry itself.

3. **Two-phase staging for cross-filesystem moves.** Copy → verify → delete. Never atomic delete-and-move. APFS same-volume moves use atomic `moveItem`. This ensures no data loss on crash.

4. **Inode recording.** `source_inode` and `dest_inode` stored for every file move. Enables crash recovery and detection of "file moved outside Curator" scenarios via FSEvents correlation.

5. **Linear undo for v1; rule-scoped undo as special case.** Selective undo (Berlage & Genau) is deferred. Rule-scoped undo is implemented because it has a clean dependency boundary (`rule_id` filter) and is the most common "undo many things at once" use case.

6. **Group undo is one Cmd+Z step.** A 50-file commit is one undo step. Partial undo (when some files are gone) requires explicit user confirmation. The undo event records which files were and were not restored.

7. **Two-tier undo window: 90 days for staging_move events, 180 days for commit_move events (R0_open_questions_resolved NB1). Events older than their window are marked `reversible = 0` automatically but are never hard-deleted.** This is a UX decision: "undo" should feel immediate and reliable, not archaeological. Users wanting to move a file back after their window should use drag-and-drop, not undo.

8. **Default explanation is one sentence.** The strongest signal is shown inline. Details drawer shows all signals. This matches the "Trust in Transparency" research finding: one clear reason increases trust; too many reasons create fatigue.

9. **Activity view uses day + session grouping.** The top-level item is a session summary row ("Daily Autopilot — 14:32 — 23 files organized"), not individual events. Individual events are revealed on expansion. This prevents the Activity view from being an overwhelming list.

10. **SQLCipher encryption for the activity log.** Same key as the main database, stored in macOS Keychain with `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`. The activity log contains NER entities and provenance URLs extracted from file content — it is sensitive.

11. **1-year full retention, monthly compaction after that.** At ~650 bytes/event and ~40,000 events/year steady state, storage is ~26 MB — negligible. The 1-year limit is a relevance decision, not a storage decision.

12. **`duplicate_trash` undo via Trash inspection.** `NSFileManager.trashItem()` returns the Trash URL. On undo, Curator checks if the file is still in Trash. If yes, restores it. If no (Trash emptied), marks `reversible = 0` and shows an explanatory message. Reversibility is computed dynamically, not stored.

13. **Batch table for large group operations.** Group commits with many files use `curator_event_files` table, not a single `curator_events` row with an array. This is more queryable, indexable, and avoids SQLite row size limits.

14. **FTS5 full-text search over `reason` and paths.** The Activity view search searches over file names, paths, and reason text. SQLite FTS5 handles this efficiently at Curator's scale.

---

## 10. Open Questions

1. **Undo window for committed files: 90 days or longer?** The 90-day window is chosen arbitrarily. Academic users may want to undo a commit from last semester (4+ months ago). Should the undo window be longer for commit events specifically, or user-configurable? There is a UX risk: offering undo on a file committed 6 months ago, where the file has since been further organized, creates complex dependency situations.

2. **Recovery from "file moved outside Curator after commit": offer to track the move or ignore it?** When FSEvents detects that a committed file was moved by another app (Finder, another organizer), Curator can update its database to reflect the new path. But should it do so silently, or ask the user? Silent tracking keeps the database accurate; asking the user is more transparent but adds noise.

3. **Per-file undo within a group commit?** Currently, group undo is atomic (all or nothing, with partial undo for missing files). Should users be able to undo a single file from a committed group without undoing the whole group? This requires breaking the batch into individual undo steps, which complicates the undo stack. Deferred for v1.

4. **Activity view density: how many events per day is "too many"?** On heavy initial cleanup days, there may be 1,000+ staging events in one session. The session summary row hides this, but users who expand the session see a very long list. Should individual file events be paginated within a session expansion? Virtual list rendering (SwiftUI `LazyVStack`) handles this technically, but the UX implications of "500 staging moves in one session" are not fully resolved.

5. **`reason_details` JSON schema vs. normalized table?** Storing signal details as a JSON array in `reason_details` is flexible but not queryable beyond full-text. A normalized `curator_event_signals` table (one row per signal per event) would enable "which events were triggered by provenance signals?" queries. Is this query needed? Unknown in v1.

6. **Should the Activity log be visible from `_Curator Review` UI or only from the main app?** The Review Hub is a Finder folder. Users browsing it in Finder cannot see why files are there. A `.curator_log.html` file in `_Curator Review` showing a human-readable summary would make the log accessible without opening the app. Security concern: this file is unencrypted. Worth it?

7. **Log export format for privacy transparency.** JSON is machine-readable; CSV is user-readable in Excel. Should Curator export both? Should the export include NER entities extracted from file content (which may be sensitive), or only paths and action types?

8. **Crash recovery UX: silent or surfaced?** If Curator resolves 3 pending events on startup (completing interrupted moves), should it show the user a notification ("3 file operations were recovered after an unexpected quit") or handle it silently? Silent recovery is cleaner UX but reduces transparency. Surfaced recovery builds trust but may alarm users who don't understand what "interrupted operation" means.
