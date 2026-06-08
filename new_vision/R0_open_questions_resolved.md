# R0 — Open Questions Resolved

> All blocking (B1–B5) and non-blocking (NB1–NB10) questions from R0_synthesis §7 are closed here.
> Research questions (RQ1–RQ5) are intentionally left open — they are IUI 2027 hypotheses, not pre-conditions.

---

## BLOCKING

### B1 — Cross-volume staging: network drop between copy and delete

**Question:** Source on network volume. Copy succeeds. Network drops before delete. What is recovery?

**Answer:** Use the WAL record + SHA-256 to determine which step completed. The 4-case recovery algorithm on startup:

```python
# For every WAL record with status != 'complete' on startup:
src_exists = path_exists(intent.src_path)
dst_exists = path_exists(intent.dest_path)

if src_exists and not dst_exists:
    # Copy never started, or failed mid-copy (partial dest was cleaned).
    # → Re-run full pipeline: copy → verify → delete
    action = RESTART_FULL_PIPELINE

elif not src_exists and dst_exists:
    # Case (b): delete succeeded but WAL update crashed.
    # Verify dest to rule out corrupt partial copy:
    if sha256(intent.dest_path) == intent.expected_sha256:
        action = MARK_COMPLETE          # ← normal case
    else:
        action = QUARANTINE_DEST_ALERT  # corrupt dest, source gone

elif src_exists and dst_exists:
    if sha256(intent.dest_path) == intent.expected_sha256:
        # Case (c): copy+verify done, delete not attempted.
        # → Delete source, mark complete.
        action = DELETE_SOURCE_THEN_MARK_COMPLETE
    else:
        # Case (d): partial/corrupt copy. Source intact.
        # → Delete corrupt dest, restart from step 1.
        action = DELETE_DEST_THEN_RESTART

else:  # neither exists
    # Stale WAL (both deleted externally after WAL written).
    action = DISCARD_INTENT_LOG_WARNING
```

**Key implementation detail:** Store `expected_sha256` in the WAL record at step 2 (after verification succeeds), before step 3 (delete). Never before.

**Why not atomic rename?** macOS `rename()` returns `EXDEV` (errno 18) across mount points — same as Linux. `renameat2` doesn't exist on macOS. Cross-volume atomic move is impossible; copy-then-delete is the only correct approach everywhere.

**Curator decision:** WAL pattern is correct as designed. Add `expected_sha256` field to `staged_files.wal_intent` JSON payload. Run 4-case recovery check on every startup before accepting new work.

---

### B2 — NSFileCoordinator timeout for iCloud-evicted files

**Question:** `NSFileCoordinator` has no built-in timeout. How long to wait, and what is the fallback?

**Answer:** Timeout = **30 seconds** via external `DispatchWorkItem` watchdog + `coordinator.cancel()`. Fallback state = `evicted_skip` (not `failed_to_read` — file is not failed, just unavailable now).

```swift
func stageEvictedFile(at url: URL, timeout: TimeInterval = 30) async throws -> URL {
    let coordinator = NSFileCoordinator(filePresenter: nil)

    // Trigger iCloud download request before coordinating
    try FileManager.default.startDownloadingUbiquitousItem(at: url)

    return try await withCheckedThrowingContinuation { continuation in
        let queue = DispatchQueue(label: "curator.coordinator", qos: .utility)
        var didResume = false

        let watchdog = DispatchWorkItem {
            guard !didResume else { return }
            didResume = true
            coordinator.cancel()
            continuation.resume(throwing: CuratorError.iCloudDownloadTimeout(url: url))
        }
        queue.asyncAfter(deadline: .now() + timeout, execute: watchdog)

        var coordError: NSError?
        coordinator.coordinate(readingItemAt: url, options: [], error: &coordError) { resolvedURL in
            watchdog.cancel()
            guard !didResume else { return }
            didResume = true
            continuation.resume(returning: resolvedURL)
        }

        if let error = coordError, !didResume {
            watchdog.cancel()
            didResume = true
            continuation.resume(throwing: error)
        }
    }
}
```

**Critical pitfall:** Never call `NSFileCoordinator` from inside an eviction call, and never nest coordinators — `trashItem` internally acquires its own coordination lock (deadlock).

**Fallback state:** On timeout or `NSUserCancelledError`:
- Set file status to `evicted_skip` (new status, distinct from `failed_to_read`)
- Record `skipped_reason = "icloud_timeout"` in `staged_files`
- Add to retry queue with exponential backoff (1h, 4h, 24h)
- Never mark as permanently failed

**Timeout by pass type:** 30s for initial scan passes; 15s for background delta passes (lower priority, don't block other files).

**Curator decision:** 30s timeout. `send2trash` and `NSFileCoordinator` must never be nested. Add `evicted_skip` status to the file state machine (transient, not in the 13 canonical states — it's a sub-state of `new`/`metadata_only` indicating deferred-by-icloud).

---

### B3 — NSFileManager.trashItem from Python sidecar (no UI)

**Question:** Does `trashItem` work from a background process? Sandbox entitlements?

**Answer:** **Yes**, from a LaunchAgent (per-user session). **No**, from a LaunchDaemon (system session, root, no user context). Curator's sidecar is a LaunchAgent → it works.

**Python implementation:** Use **`send2trash`** (wraps `FSMoveObjectToTrashSync` via ctypes, no compilation, no PyObjC required):

```python
# requirements.txt: send2trash>=1.8.3
from send2trash import send2trash, TrashPermissionError

def safe_trash(path: str) -> None:
    """Move file to user Trash. Works from LaunchAgent. Raises OSError on failure."""
    try:
        send2trash(path)
    except TrashPermissionError as e:
        raise PermissionError(f"Cannot trash {path}: {e}") from e
```

**Do NOT:** Call `send2trash` or `trashItem` inside an `NSFileCoordinator` block. Call them bare.

**Sandbox entitlements:**

| Scenario | Entitlement |
|---|---|
| Unsandboxed LaunchAgent (Curator's case) | None needed |
| Sandboxed LaunchAgent | `com.apple.security.files.user-selected.read-write` + scoped bookmarks |
| LaunchDaemon (system) | `trashItem` fails — use POSIX `~/.Trash/` fallback |

**Curator decision:** Add `send2trash>=1.8.3` to `requirements.txt`. Use it exclusively for all "trash" operations in the Python sidecar. The undo log records the resulting Trash path via `NSFileManager.trashItemAtURL:resultingItemURL:` (Swift side, for recovery).

---

### B4 — Mondrian CP before 20 calibration examples per context

**Question:** New contexts start with 0 examples. What confidence is assigned before 20 are collected?

**Answer:** Use **global CP** (non-Mondrian, all contexts pooled) as cold-start fallback until a context reaches 20 calibration examples. Then switch to Mondrian CP for that context.

```python
def get_confidence_set(file_id, context_id, alpha=0.15):
    n_cal = get_calibration_count(context_id)
    if n_cal >= 20:
        # Mondrian CP: context-specific calibration
        return mondrian_cp.predict(file_id, context_id, alpha=alpha)
    else:
        # Global CP fallback: all contexts pooled
        # Wider prediction sets — acceptable for cold start
        return global_cp.predict(file_id, alpha=alpha)
```

**Confidence display:** During cold-start (< 20 examples), the 4-dot scale shows max 3 dots (never 4) regardless of global CP score — UI signal to user that this context is still "learning". 4 dots = Mondrian CP confidence only.

**Curator decision:** Global CP fallback with α=0.15. 4-dot maximum during cold-start capped at 3 dots. Mondrian CP activates per-context at N=20. No special handling needed in DB schema — `communities.calibration_count` field tracks this.

---

### B5 — RETSim (unisim) archived on GitHub — is it safe to use?

**Question:** The `google/unisim` GitHub repo was archived May 2026. Does the pip package still build on Apple Silicon?

**Answer:** **Safe to use.** Confirmed:

- PyPI: `unisim 1.0.1` available (last updated 2024-08-08)
- Wheel type: `unisim-1.0.1-py3-none-any.whl` — **pure Python, no compiled extensions**
- Backend: **ONNX + onnxruntime** (NOT TensorFlow — TF is dev-only for model conversion)
- `onnxruntime` has full Apple Silicon / MPS support
- No architecture-specific wheel needed → installs on macOS arm64 cleanly

**Dependencies (runtime only):**
```
numpy, tabulate, tqdm, jaxtyping, onnx, onnxruntime, pandas, usearch>=2.6.0
```
All available for Apple Silicon.

**Risk:** GitHub repo archived → no future updates or bug fixes. Not a concern for Curator since we lock the version.

**Fallback (if unisim stops working):** `all-MiniLM-L6-v2` cosine similarity with adjusted thresholds (already in the pipeline for embeddings):
- Near-duplicate: cosine ≥ 0.93 (higher than RETSim's 0.90 because all-MiniLM is noisier)
- Version candidate: cosine 0.78–0.92 (shifted up accordingly)

**Curator decision:** Use `unisim==1.0.1`, pin the version. Add `unisim==1.0.1` to `requirements.txt`. Implement `all-MiniLM-L6-v2` cosine as fallback path activated when `unisim` fails to import. One `try: import unisim; RETSIM_BACKEND = 'unisim'` / `except: RETSIM_BACKEND = 'minilm'` guard at module level.

---

## NON-BLOCKING

### NB1 — Undo window: 90 days uniform, or longer for committed files?

**Decision:** **Two-tier retention:**
- `staging_move` events: 90 days active undo
- `commit_move` events: **180 days** active undo (user consciously approved the commit — longer regret window is justified)
- Both: indefinitely archived (status `archived`, soft-deleted from undo UI after window)

Implementation: `curator_events.undo_expires_at` = NULL (indefinite) for archived; computed field based on event_type for UI display.

---

### NB2 — Crash recovery UX: silent or surfaced?

**Decision:** **Surface briefly, non-modally.**

On startup after recovery:
- If 0 operations recovered → silent (no notification)
- If 1–5 operations recovered → transient status bar message: "3 interrupted operations recovered" (auto-dismiss 4s)
- If 6+ operations recovered → persistent Home screen card: "Recovery needed — X operations were interrupted. [Review]"

Never: modal dialogs, blocking alerts. The user may not care; the information is in Activity log if they do.

---

### NB3 — Per-file undo within a group commit: accommodate in v1 data model?

**Decision:** **Yes — the data model already supports it.** `curator_event_files` has per-file `status` and `error_message` fields. In v2, querying `WHERE batch_id = ? AND file_id = ?` would enable per-file undo within a group. No schema migration needed — just implement the UI in v2.

In v1: group undo only (whole `batch_id`). The capability is latent in the schema.

---

### NB4 — reason_details: JSON array vs. normalized signals table?

**Decision:** **JSON for v1.** Store as `TEXT` column containing a JSON array of signal objects:

```json
[
  {"type": "semantic", "score": 0.87, "compared_to": "file_uuid_123"},
  {"type": "ner", "entities": ["EPL342", "Assignment 2"], "jaccard": 0.67},
  {"type": "provenance", "domain": "moodle.cs.ucy.ac.cy", "weight": 0.8}
]
```

Add SQLite FTS5 virtual table over `reason` (plain-text summary field, separate from `reason_details`). If analytics over signals are needed in v2, migrate to normalized `curator_event_signals` table then.

---

### NB5 — .curator_log.html in _Curator Review?

**Decision:** **No.**

Reasons:
1. Security: unencrypted file listing all staged file paths, accessible to any process with Finder access.
2. Conceptual pollution: Review Hub is for files awaiting decisions, not for app metadata.
3. Maintenance: keeping an HTML file in sync with SQLite is brittle.

Activity log is app-only (SQLCipher-encrypted SQLite). If user wants to export activity, add an explicit "Export log as CSV" function in the app.

---

### NB6 — OBI component weights (uncalibrated)

**Decision:** Start with **literature-derived weights**; calibrate empirically in Phase 2.

Starting weights:
```
OBI = 0.30·H̃(L|I) + 0.20·FDE + 0.20·OR + 0.15·NDD + 0.15·(1-FLRS̄)
```

Phase 2 calibration: Pearson correlation of each component against PSSUQ Q8 ("organizational burden" self-report). Weight by correlation magnitude. Publish calibrated weights in IUI 2027 paper.

---

### NB7 — DCR baseline for chaotic vs. organized filesystems

**Decision:** **Cannot pre-commit; measure in Phase 1.** 

Procedure: Phase 1 diary study participants provide baseline decision count log (how many file decisions they made manually in 2 weeks before Curator). DCR = (baseline decisions) / (Curator review sessions × avg groups/session). This gives a real denominator.

Placeholder in evaluation protocol: Phase 1 baseline measurement of "file decisions per week" as a structured diary question.

---

### NB8 — ProxAnn gate at 0.70: empirical basis?

**Decision:** Start at **0.65** (more permissive), calibrate upward from Phase 0.

Rationale: 0.70 is ProxAnn's default for academic cluster evaluation (Hoyle et al., ACL 2025). Personal file clusters are shorter and noisier than topic model clusters — a lower threshold is appropriate until calibrated. 

Phase 0 calibration: Run ProxAnn on HippoCamp benchmark. Find the threshold that maximizes (GCR improvement / groups flagged) curve. Expected range: 0.60–0.75.

---

### NB9 — Undo <60s threshold: configurable?

**Decision:** **Not configurable in v1.**

60 seconds is calibrated from "accidental vs. deliberate" action research (analogous to Fitts's Law for temporal commitment — actions taken and reversed within 60s are cognitively equivalent to "not made"). Adding configurability adds UI surface and cognitive load for a marginal benefit. Re-evaluate in v2 if user feedback shows the threshold is wrong.

If a user reverses an action between 30–60s, the learning system doesn't penalize them — this is the intended behavior.

---

### NB10 — Trash reversibility: evaluate on render or on undo click?

**Decision:** **Lazy — evaluate on undo click only.**

Rendering the Activity view should never trigger disk access. Checking `~/.Trash/` for 100+ events on every render would be:
- Slow (FSEvents doesn't watch Trash in real-time)
- Racy (Trash contents change outside Curator)
- Battery-hostile

On undo click: check dynamically. If `NSFileManager.trashItem()` resultingItemURL is no longer in Trash → show: "This file was permanently deleted from Trash. Cannot restore." → mark event `irreversible = 1`.

Display in Activity view: show "Restore" button for all `duplicate_trash` events within the 90-day window. Let the undo click fail gracefully with a clear message if Trash was emptied.

---

## RESEARCH QUESTIONS (intentionally open — IUI 2027 hypotheses)

| ID | Hypothesis | Testable in Phase |
|---|---|---|
| RQ1 | ADHD OAP differs significantly from neurotypical OAP (ANOVA on abandonment session length) | Phase 2 |
| RQ2 | Coherence Verification pre-filtering improves GCR enough to justify API cost | Phase 1 (with/without) |
| RQ3 | OBI correlates with subjective organizational stress r ≥ 0.5 (Pearson, PSSUQ) | Phase 2 |
| RQ4 | HippoCamp QA accuracy maps to human re-finding success (dual measurement) | Phase 2 |
| RQ5 | FSEvents + MDQuery can intercept Spotlight queries resolving to Curator-organized files | Phase 3 |

These are open by design. They are the paper's contribution.

---

## Summary: All Doors Closed

| ID | Status | Decision |
|---|---|---|
| B1 | ✅ Closed | 4-case WAL recovery; store expected_sha256 before delete |
| B2 | ✅ Closed | 30s timeout; DispatchWorkItem watchdog; evicted_skip state |
| B3 | ✅ Closed | send2trash>=1.8.3; LaunchAgent context; not inside coordinator |
| B4 | ✅ Closed | Global CP fallback until N=20; 3-dot max in cold-start UI |
| B5 | ✅ Closed | unisim 1.0.1 safe (ONNX, pure Python); pin version; all-MiniLM fallback |
| NB1 | ✅ Closed | 90 days staging; 180 days commit_move |
| NB2 | ✅ Closed | Surface briefly (4s transient) if 1-5 recovered; card if 6+ |
| NB3 | ✅ Closed | Data model already supports per-file undo; implement in v2 |
| NB4 | ✅ Closed | JSON for v1; FTS5 on plain-text reason field |
| NB5 | ✅ Closed | No HTML file in Review Hub; export function in app instead |
| NB6 | ✅ Closed | Weights: 0.30·H̃+0.20·FDE+0.20·OR+0.15·NDD+0.15·(1-FLRS̄); calibrate Phase 2 |
| NB7 | ✅ Closed | Measure in Phase 1 diary study; no pre-commit value |
| NB8 | ✅ Closed | Start ProxAnn gate at 0.65; calibrate from Phase 0 HippoCamp |
| NB9 | ✅ Closed | 60s threshold fixed in v1; not configurable |
| NB10 | ✅ Closed | Lazy evaluation on undo click; never on Activity view render |
| RQ1–RQ5 | 🔬 Open (by design) | IUI 2027 research hypotheses |

*Document written: 2026-06-08. Sources: B1-B3 macOS API research; B5 PyPI verification (unisim 1.0.1, py3-none-any, onnxruntime backend).*
