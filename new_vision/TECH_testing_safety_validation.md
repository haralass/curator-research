# Curator: Testing and Safety Validation Spec

**Status:** Authoritative
**Audience:** Core contributors, anyone touching file movement code
**Last updated:** 2026-06-08
**Based on:** direct read of app.py, pipeline.py, ingest.py, db.py, embeddings.py, and 36 Swift source files

Curator moves real user files. A single bug can silently discard an irreplaceable document. This spec defines the test surface required before any file-moving code ships to a real filesystem.

---

## Section 1: Test Infrastructure

### Synthetic filesystem fixture

```python
# engine/tests/fixtures/make_test_filesystem.py
# Creates a deterministic synthetic personal filesystem for testing.
# No real user files are touched. All files created under base_dir.

from pathlib import Path
from dataclasses import dataclass, field
from typing import NamedTuple

@dataclass
class TestFilesystem:
    base: Path
    all_paths: list[Path] = field(default_factory=list)
    exact_dup_pairs: list[tuple[Path, Path]] = field(default_factory=list)
    near_dup_pairs: list[tuple[Path, Path]] = field(default_factory=list)
    version_families: list[list[Path]] = field(default_factory=list)
    protected_pdfs: list[Path] = field(default_factory=list)
    corrupted_files: list[Path] = field(default_factory=list)
    permission_denied: list[Path] = field(default_factory=list)
    icloud_stubs: list[Path] = field(default_factory=list)
    same_source_files: list[Path] = field(default_factory=list)
    greek_paths: list[Path] = field(default_factory=list)
    course_folders: dict[str, Path] = field(default_factory=dict)
    chaos_folder: Path | None = None


def create_test_filesystem(base_dir: Path) -> TestFilesystem:
    """
    Creates:
    - 20 files with Greek/Greeklish filenames (NFC normalized)
    - 5 exact duplicate pairs (same SHA-256, different paths)
    - 3 near-duplicate pairs (SimHash Hamming distance ≤ 5)
    - 3 version families (v1/v2/final naming patterns)
    - 3 protected PDFs (pypdf encrypted with owner password)
    - 3 corrupted files (truncated PDF, invalid zip header, truncated DOCX)
    - 2 files with permission denied (chmod 000 after creation)
    - 2 iCloud stub simulations (0-byte + com.apple.icloud xattr)
    - 5 files with same kMDItemWhereFroms source URL
    - EPL342/, EPL312/ course folders with realistic lecture/assignment structure
    - chaos/ folder with 50 completely unrelated file types
    """
    ...
```

### Fixture file generators (used by create_test_filesystem)

```python
def create_protected_pdf(path: Path):
    """Create a minimal encrypted PDF using pypdf owner password."""
    # pypdf: PdfWriter + encrypt(user_password="", owner_password="owner123")

def create_corrupted_pdf(path: Path):
    """Write valid PDF header (%PDF-1.4) followed by invalid bytes."""
    # path.write_bytes(b"%PDF-1.4\n" + b"\x00\xff\x00\xff" * 64)

def create_icloud_stub(path: Path):
    """0-byte file + com.apple.icloud xattr to simulate evicted iCloud file."""
    # path.touch()  — 0 bytes
    # subprocess.run(["xattr", "-w", "com.apple.icloud", "placeholder", str(path)])

def create_greeklish_filename(base: Path, greek_name: str, ext: str) -> Path:
    """Write a file with a Greek filename in NFC normalized form."""
    # import unicodedata; name = unicodedata.normalize("NFC", greek_name + ext)
    # return base / name

def create_greek_filenames(base: Path) -> list[Path]:
    """
    Creates files with names including:
    "αρχείο_1.pdf", "Κεφάλαιο 3.docx", "EPL342_εργασία1.pdf",
    "τελική_αναφορά_v2.pdf", "ασκήσεις_διαλέξεων.txt",
    "Greeklish_test_kefalaiο.pdf", plus 14 more variants.
    All names NFC normalized on creation.
    """
    ...
```

### Test database and environment setup

```python
# engine/tests/conftest.py
import pytest
from pathlib import Path
import tempfile

@pytest.fixture(scope="session")
def test_filesystem(tmp_path_factory):
    base = tmp_path_factory.mktemp("curator_test_fs")
    return create_test_filesystem(base)

@pytest.fixture(autouse=True)
def isolated_db(tmp_path, monkeypatch):
    """Each test gets a fresh SQLite DB and FAISS index."""
    db_path = tmp_path / "test.db"
    index_path = tmp_path / "test.index"
    monkeypatch.setenv("CURATOR_DB_PATH", str(db_path))
    monkeypatch.setenv("CURATOR_INDEX_PATH", str(index_path))
    from curator_sidecar import db, embeddings
    db.init_db(db_path)
    yield
    # teardown: restore chmod on permission_denied files so tmp cleanup works
```

---

## Section 2: Unit Test Specs

### File Identity Tests

**Test: safe-save detection**
- Input: file at path `/tmp/thesis.pdf` with inode 1001. Safe-save: new inode 1002 appears at same path, same SHA-256.
- Expected: `db.upsert_file` returns same file_id for inode 1002 as was assigned to inode 1001.
- Assertion: `file_id_before == file_id_after`; `changed == True`; `is_new == False`.
- Rationale: Current `upsert_file` falls through to the `path=?` lookup when inode changes. Verify that path matches existing record and returns its id.

**Test: cross-volume identity**
- Input: file A on volume 101 (inode 500, SHA-256 "abc123"), file B on volume 102 (inode 600, SHA-256 "abc123").
- Expected: `content_hash` matches across volumes; detected as "same content" by duplicate gate even though inode/device differ.
- Assertion: `duplicate_gate(file_a_id, file_b_id) == True`.

**Test: iCloud stub detection**
- Input: 0-byte file with `com.apple.icloud` xattr at path `/Users/x/Documents/thesis.pdf`.
- Expected: `ingest.detect_icloud_stub(path) == True`; file_state set to `locked`, not `queued`.
- Assertion: file never enters embedding queue; `file_state == "locked"`.

**Test: Finder move same volume**
- Input: FSEvents rename event — old path `/Downloads/report.pdf` → new path `/Documents/report.pdf`, same inode.
- Expected: `db.upsert_file` updates `path` column; file_id unchanged.
- Assertion: `SELECT path FROM files WHERE id=?` returns new path; old path gone.

---

### Duplicate Detection Tests

**Test: exact duplicate → same Union-Find family**
- Input: two files with identical SHA-256 at different paths.
- Expected: both assigned to same `duplicate_family_id` in files table.
- Assertion: `SELECT duplicate_family_id FROM files WHERE id IN (?, ?)` — both return same non-null value.

**Test: near-duplicate (SimHash Hamming ≤ 5) → same family**
- Input: two text files differing by 10 words out of 500; SimHash distance computed as ≤ 5.
- Expected: near-duplicate pair assigned to same family.
- Assertion: `duplicate_family_id` matches; `similarity_type == "near_duplicate"`.

**Test: non-duplicate (Hamming > 5) → different families**
- Input: two semantically similar but textually different files (lecture notes from different weeks).
- Expected: different `duplicate_family_id` or NULL.
- Assertion: family IDs differ; files remain as separate groups.

**Test: version family → version family, NOT duplicate family**
- Input: `thesis_v1.pdf`, `thesis_v2.pdf`, `thesis_final.pdf` — different content, same base name pattern.
- Expected: detected as version family (`version_family_id` set); `duplicate_family_id` NULL for all three.
- Assertion: `SELECT version_family_id FROM files WHERE path LIKE '%thesis%'` — all same; duplicate_family_id all NULL.

**Test: format twin not shown as duplicate**
- Input: `thesis.docx` and `thesis.pdf` — same document, different format.
- Expected: format twin flag set; NOT shown in duplicate UI card.
- Assertion: `is_format_twin == True`; both files excluded from duplicate gate results.

---

### Move Safety Tests

**Test: same-volume atomic rename**
- Input: file at `/Downloads/report.pdf`. Stage to hub on same volume.
- Expected: `os.rename()` used (atomic); source path gone; destination present; inode unchanged.
- Assertion: `src_stat_before.st_ino == dst_stat_after.st_ino`; `os.path.exists(src) == False`.

**Test: cross-volume copy-verify-delete sequence**
- Input: file on volume A, hub folder on volume B.
- Expected: copy → verify SHA-256 match → delete source. If SHA mismatch after copy: abort, leave source intact.
- Assertion: call order recorded: copy, verify, delete. Verify that `send2trash` is called for delete (never `os.unlink`).

**Test: WAL intent written BEFORE file operation**
- Input: any stage or commit action.
- Expected: `wal_intents` row with `status='pending'` exists in DB before any filesystem call.
- Assertion: mock filesystem call; confirm `wal_intents` row committed before mock is invoked.

**Test: WAL complete marked AFTER file operation**
- Input: successful stage operation.
- Expected: `wal_intents.status` updated to `'complete'` only after filesystem call returns without exception.
- Assertion: if filesystem call raises OSError mid-way, `status` remains `'pending'`.

**Test: destination collision → _curator_N suffix**
- Input: stage file `report.pdf` to hub folder that already contains `report.pdf`.
- Expected: file staged as `report_curator_2.pdf`; no overwrite.
- Assertion: both files exist; original `report.pdf` unmodified.

**Test: permission denied → error class, no crash**
- Input: file with `chmod 000`.
- Expected: `PermissionError` caught; `file_state` set to `'failed_read'`; `error_class` set to `'permission_denied'`; file routed to `Review Hub/Failed to Read/Permission/`.
- Assertion: no uncaught exception; API returns 200 with error detail in response body.

---

### WAL Recovery Tests (4-case matrix from new architecture spec)

These four cases must all be tested in isolation. Each test:
1. Creates a WAL intent with `status='pending'`.
2. Restarts the pipeline (simulates crash recovery).
3. Asserts the correct recovery action.

**Case 1: file only at source (move never started)**
- Condition: WAL pending, src_path exists, dst_path does not exist.
- Recovery action: restart move operation from src → dst.
- Assertion: move attempted on recovery start.

**Case 2: file only at destination, SHA matches (move completed, WAL not marked)**
- Condition: WAL pending, src_path gone, dst_path exists, SHA-256 matches original.
- Recovery action: mark WAL `status='complete'`; update `files.path` to dst_path.
- Assertion: no filesystem operation; WAL marked complete.

**Case 3: file at both src and dst, SHA matches (copy succeeded, delete failed)**
- Condition: WAL pending, both paths exist, SHA-256 of dst matches original.
- Recovery action: delete source (via send2trash); mark WAL complete.
- Assertion: src deleted; WAL complete.

**Case 4: file at both src and dst, SHA mismatch (copy corrupted)**
- Condition: WAL pending, both paths exist, SHA-256 of dst does NOT match original.
- Recovery action: delete dst (corrupted copy); restart move.
- Assertion: dst removed; move retried.

**Case 5: neither src nor dst exists**
- Condition: WAL pending, both paths gone.
- Recovery action: discard WAL intent; log warning.
- Assertion: WAL row deleted or status='failed'; warning in error log.

**Case 6: missing Review Hub folder**
- Condition: hub folder path recorded in DB does not exist on disk.
- Recovery action: re-create hub folder; re-stage any groups that reference it.
- Assertion: hub folder exists after recovery; group status still 'staged'.

**Case 7: orphan file in hub (no DB record)**
- Condition: file physically present in hub folder but no `wal_intents` row and no `files` row matches its path.
- Recovery action: detected on startup scan; either matched by SHA-256 to existing file record (update path) or moved to `Review Hub/Unknown/` for user review.
- Assertion: no silent discard; file accessible in UI.

---

### State Machine Tests

**Valid transitions to test (must all succeed):**
- `queued` → `hashing`
- `hashing` → `duplicate_gate`
- `duplicate_gate` → `embedding`
- `duplicate_gate` → `staged` (exact duplicate shortcut)
- `embedding` → `staged`
- `staged` → `committed`
- `staged` → `restored` → original path, state reset
- `hashing` → `failed_read` (PermissionError)
- `embedding` → `failed_read` (corrupted file)

**Invalid transitions to test (must raise error):**
- `committed` → `embedding` (already committed)
- `locked` → `staged` (locked files must never be staged)
- `failed_read` → `committed` (cannot commit unreadable file)

**Watchdog test:**
- Set `_pipeline_running = True` and `_pipeline_start_time = time.time() - 3700` (>60 minutes ago).
- Assert watchdog fires, sets `_pipeline_running = False`, logs error.

**locked vs ignored distinction:**
- `locked`: iCloud stub, permission denied, actively open file. Must never get DB record beyond metadata.
- `ignored`: user explicitly dismissed. Gets DB record with `file_state='ignored'`. Must not reappear in hub.
- Assertion: `locked` files never appear in `review_groups`; `ignored` files survive restart without re-appearing.

---

### Error Classification Tests

**Protected PDF → correct routing**
- Input: PDF encrypted with owner password.
- Expected: `_classify_extraction_error` returns `("PROTECTED_PDF", "PDF is password-protected")`.
- File physically routed to: `Review Hub/Failed to Read/Protected/`.
- Assertion: `file_state == 'failed_read'`; hub subpath contains "Protected".

**Corrupted file → correct routing**
- Input: file with valid extension, truncated/invalid bytes.
- Expected: `("CORRUPT_FILE", "File appears corrupt")`.
- Routed to: `Review Hub/Failed to Read/Corrupted/`.

**OCR timeout → correct routing**
- Input: scanned PDF that exceeds `_EXTRACT_TIMEOUT` (8 seconds).
- Expected: `("TIMEOUT", "Extraction timed out")`.
- Routed to: `Review Hub/Failed to Read/Timeout/`.

**Permission denied → correct routing**
- Input: `chmod 000` file.
- Expected: `("PERMISSION_DENIED", "File cannot be read")`.
- Routed to: `Review Hub/Failed to Read/Permission/`.

---

## Section 3: Integration Tests

### Scan Pipeline Integration

**Test: 100-file synthetic scan**
- Setup: create_test_filesystem with 100 files. Run POST /scan/start. Poll GET /scan/status until complete.
- Expected: all files have a non-null `file_state`; duplicate pairs detected; failure cases routed to hub error subfolders.
- Assertions:
  - `SELECT COUNT(*) FROM files WHERE file_state IS NULL` → 0
  - `SELECT COUNT(*) FROM files WHERE file_state = 'failed_read'` → at least 8 (3 protected + 3 corrupted + 2 permission denied)
  - All 5 exact duplicate pairs → `duplicate_family_id` not null
  - Scan completes in < 10 minutes on M1

**Test: Greek filenames survive full scan**
- Setup: 20 files with Greek/Greeklish names. Run full scan.
- Expected: no UnicodeDecodeError; all files reach state `embedded` or `staged`; filenames stored as NFC in DB.
- Assertion: `SELECT path FROM files WHERE path LIKE '%αρχ%'` returns results; no error log entries with "UnicodeDecodeError".

### Review Hub Integration

**Test: stage a group**
- Setup: run scan to produce review groups. Call `POST /review_hub/group/{id}/approve`.
- Expected: files physically present in hub folder path; `review_groups.status == 'staged'`; `wal_intents.status == 'complete'` for all members.
- Assertions: `os.path.exists(hub_path)` for each file; `SELECT status FROM review_groups WHERE id=?` → 'staged'.

**Test: commit a staged group**
- Setup: group in staged state. Call `POST /review_hub/group/{id}/commit`.
- Expected: files at final destination path; `review_groups.status == 'committed'`; activity log entry created.
- Assertions: source hub path gone; final path exists; `activity_log` has `event_type='committed'` row.

**Test: restore a staged group**
- Setup: group in staged state. Call `POST /review_hub/group/{id}/restore`.
- Expected: files at original path (pre-staging); `review_groups.status == 'restored'`; files available at original location.
- Assertions: original paths exist; hub paths gone.

**Test: split a group**
- Setup: group with 6 files. Call `POST /review_hub/group/{id}/split` with 3 file_ids.
- Expected: original group has 3 remaining files; new group has 3 files; both groups in `review_groups`; `split_from` points to original.
- Assertions: `SELECT COUNT(*) FROM review_group_members WHERE group_id=?` → 3 for each group.

### Crash Recovery Integration

**Test: WAL pending at startup**
- Setup: create pending WAL intent for Case 1 (file at source only). Simulate restart by calling recovery function directly.
- Expected: move completes; WAL marked complete.

**Test: missing hub folder at startup**
- Setup: create a `review_groups` record with a `hub_dir` path that does not exist. Call startup recovery.
- Expected: hub_dir re-created; files re-staged into it.

**Test: orphan .curator_group.json in hub**
- Setup: create a JSON sidecar file in hub folder with no matching `review_groups` DB record. Call startup scan.
- Expected: file detected; either re-linked to DB or moved to `Review Hub/Unknown/`; no crash.

---

## Section 4: Greek-Specific Tests

All Greek tests must run with the test filesystem containing files created with Greek/Greeklish names.

**NFC normalization round-trip**
- Input: filename with NFD decomposition (e.g., "α" + combining accent as two codepoints).
- Expected: stored in DB as NFC single codepoint.
- Assertion: `SELECT path FROM files WHERE path = ?` matches NFC string; NFD query returns no results.

**Accent stripping for search index**
- Input: "κεφάλαιο" (with acute accent on α).
- Expected: FTS5 `greeklish_text` column contains "κεφαλαιο" (stripped).
- Assertion: `SELECT * FROM files_fts WHERE greeklish_text MATCH 'κεφαλαιο'` returns the "κεφάλαιο" file.

**Greeklish transliteration for search**
- Input: "κεφάλαιο".
- Expected: `greeklish_text` column contains "kefalaiο" (ISO 843 or phonetic).
- Assertion: `SELECT * FROM files_fts WHERE greeklish_text MATCH 'kefalaiο'` returns the original file.

**Mixed EPL342 + Greek filename**
- Input: file "EPL342_εργασία1.pdf".
- Expected: course code "EPL342" extracted via `_COURSE_RE`; filename stored NFC; cluster label contains "EPL342".
- Assertion: `SELECT label FROM cluster_identity WHERE uuid=?` contains "EPL342".

**FTS5 accented search**
- Input: search query "κεφαλαιο" (no accent).
- Expected: file "Κεφάλαιο 3.docx" appears in results.
- Assertion: `GET /search?q=κεφαλαιο` returns file_id of accented file.

**FTS5 Greeklish search**
- Input: search query "kefalaiο".
- Expected: same "Κεφάλαιο 3.docx" appears in results.
- Assertion: `GET /search?q=kefalaiο` returns same file_id.

---

## Section 5: Performance Safety Tests

**1000-file scan completes within 10 minutes on M1**
- Setup: 1000-file synthetic filesystem. Run full scan (hash + embed + group).
- Assertion: `time.time() - scan_start < 600` seconds.
- Failure action: log per-phase timing breakdown; identify bottleneck phase.

**Idempotent scan**
- Setup: run full scan twice on same filesystem without modifying any files between runs.
- Expected: second scan produces identical group assignments (same group labels, same file counts).
- Assertion: diff of `review_groups` before and after second scan → no changes.

**No memory leak after 1000-file embed**
- Setup: embed 1000 files sequentially. Measure RSS before and after.
- Assertion: `process.memory_info().rss < 2 * 1024 * 1024 * 1024` (2 GB).
- Method: `psutil.Process().memory_info().rss` sampled at start and end.

**No SQLite timeout under thread contention**
- Setup: 16 I/O threads + 4 extraction threads + 2 embedding threads all writing to DB simultaneously.
- Assertion: zero `sqlite3.OperationalError: database is locked` exceptions in 5-minute run.
- Note: WAL mode (already enabled in db.py) should handle this; test confirms it does.

---

## Section 6: Test File Generation

Implementations for the fixture file generators listed in Section 1:

```python
def create_protected_pdf(path: Path) -> None:
    """Minimal encrypted PDF via pypdf."""
    from pypdf import PdfWriter
    writer = PdfWriter()
    writer.add_blank_page(width=595, height=842)
    writer.encrypt(user_password="", owner_password="owner_secret_123")
    with path.open("wb") as f:
        writer.write(f)


def create_corrupted_pdf(path: Path) -> None:
    """Valid PDF header + invalid body bytes."""
    path.write_bytes(b"%PDF-1.4\n1 0 obj\n" + b"\x00\xff\xfe\xfd" * 128)


def create_invalid_zip(path: Path) -> None:
    """File with .zip extension but not a valid zip."""
    path.write_bytes(b"PK\x03\x04" + b"\xff" * 256)  # valid magic, corrupt body


def create_truncated_docx(path: Path) -> None:
    """DOCX is a zip; truncate after zip header."""
    import zipfile, io
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w") as zf:
        zf.writestr("word/document.xml", "<truncated>")
    path.write_bytes(buf.getvalue()[:64])  # truncate


def create_icloud_stub(path: Path) -> None:
    """0-byte file + com.apple.icloud xattr."""
    import subprocess
    path.touch()
    subprocess.run(
        ["xattr", "-w", "com.apple.icloud", "1", str(path)],
        check=False,
    )


def create_greek_filenames(base: Path) -> list[Path]:
    """20 files with Greek/Greeklish names, all NFC normalized."""
    import unicodedata
    names = [
        "αρχείο_1.pdf",
        "Κεφάλαιο 3.docx",
        "EPL342_εργασία1.pdf",
        "τελική_αναφορά_v2.pdf",
        "ασκήσεις_διαλέξεων.txt",
        "προγραμματισμός.pdf",
        "δίκτυα_υπολογιστών.docx",
        "βάσεις_δεδομένων_v1.pdf",
        "βάσεις_δεδομένων_v2.pdf",
        "βάσεις_δεδομένων_final.pdf",
        "EPL312_homework1.pdf",
        "EPL312_homework2.pdf",
        "Greeklish_kefalaiο.pdf",
        "mathimata_2025.txt",
        "notes_protΑ_semester.docx",
        "ανάλυση_αλγορίθμων.pdf",
        "αλγεβρα_ασκηση3.pdf",
        "διπλωματική_εργασία.docx",
        "βιβλιογραφία.txt",
        "Εργαστήριο_4_Report.pdf",
    ]
    paths = []
    for name in names:
        nfc_name = unicodedata.normalize("NFC", name)
        p = base / nfc_name
        p.write_text(f"Test content for {nfc_name}\n" * 20, encoding="utf-8")
        paths.append(p)
    return paths
```

---

## Section 7: Pre-Release Manual Checklist

This checklist must be completed by a human on a real Mac before shipping any build that touches file movement. Use the test filesystem (base_dir must NOT overlap with any real user data).

- [ ] Full synthetic scan completes without error — verify GET /scan/status shows all files in terminal states
- [ ] No real files touched — verify by checking mtimes of files outside base_dir before and after scan
- [ ] Stage 5 files manually via POST /review_hub/group/{id}/approve — verify they are physically in Review Hub folder
- [ ] Restore 5 staged files via POST /review_hub/group/{id}/restore — verify at original paths; hub paths gone
- [ ] Commit 5 staged files via POST /review_hub/group/{id}/commit — verify at final paths; hub paths gone
- [ ] Undo last committed group via POST /undo — verify files back in Review Hub staging
- [ ] Crash mid-staging: kill -9 sidecar while staging is in progress; restart; verify WAL recovery runs; verify correct 4-case recovery
- [ ] Greek filename round-trip: stage "Κεφάλαιο 3.docx" and restore; verify filename unchanged (NFC preserved)
- [ ] Protected PDF routing: verify "encrypted_test.pdf" appears in Review Hub/Failed to Read/Protected/ not in staging queue
- [ ] 0-byte iCloud stub: verify stub file does not enter staging; `file_state == 'locked'` in DB
- [ ] Permission denied file (chmod 000): verify it appears in Review Hub/Failed to Read/Permission/; chmod 755 it, re-scan; verify it now processes normally
- [ ] Split group: create group with 6 files; split into 3+3; verify two groups in hub, both stageable independently
- [ ] Rename group: rename via POST /review_hub/group/{id}/rename; verify new label used as destination folder name on commit
- [ ] Activity log: verify all above actions appear in GET /activity in correct order with correct event_types
- [ ] FTS5 search: query "κεφαλαιο" (no accent); verify accented file appears; query "kefalaiο"; verify same file appears

---

## Section 8: New Module Test Specs (thermal.py / passports.py / vector_index.py)

Added June 2026. These modules did not exist when §2–§3 were written.

### thermal.py — ThermalGovernor Tests

```python
# All NSProcessInfo and psutil calls mocked — no real hardware required

def test_nominal_ac_idle_returns_calm():
    gov = ThermalGovernor(thermal=NSProcessInfoThermalState.nominal,
                         cpu_pct=10, on_battery=False, user_idle=True)
    assert gov.current_state() == ThermalState.CALM

def test_fair_low_cpu_returns_attentive():
    gov = ThermalGovernor(thermal=NSProcessInfoThermalState.fair,
                         cpu_pct=30, on_battery=False, user_idle=False)
    assert gov.current_state() == ThermalState.ATTENTIVE

def test_fair_high_cpu_returns_reduced():
    # Apple Silicon caveat: fair + cpu > 60% treated as reduced (R11 §3.2)
    gov = ThermalGovernor(thermal=NSProcessInfoThermalState.fair,
                         cpu_pct=75, on_battery=False, user_idle=False)
    assert gov.current_state() == ThermalState.REDUCED

def test_battery_alone_returns_attentive():
    gov = ThermalGovernor(thermal=NSProcessInfoThermalState.nominal,
                         cpu_pct=10, on_battery=True, user_idle=False)
    assert gov.current_state() == ThermalState.ATTENTIVE

def test_critical_returns_paused():
    gov = ThermalGovernor(thermal=NSProcessInfoThermalState.critical,
                         cpu_pct=90, on_battery=False, user_idle=False)
    assert gov.current_state() == ThermalState.PAUSED

def test_allowed_workers_scales_with_state():
    assert ThermalGovernor(ThermalState.CALM).allowed_workers(tier=1) == 8
    assert ThermalGovernor(ThermalState.ATTENTIVE).allowed_workers(tier=1) == 4
    assert ThermalGovernor(ThermalState.REDUCED).allowed_workers(tier=1) == 0
    assert ThermalGovernor(ThermalState.PAUSED).allowed_workers(tier=1) == 0

def test_callback_fires_on_state_change():
    fired = []
    gov = ThermalGovernor(ThermalState.CALM)
    gov.register_callback(lambda s: fired.append(s))
    gov._set_state(ThermalState.REDUCED)
    assert fired == [ThermalState.REDUCED]
```

### passports.py — PassportGate Tests

```python
def test_all_registry_passports_are_valid():
    for op_type, passport in PASSPORT_REGISTRY.items():
        assert passport.is_valid(), f"Passport for '{op_type}' failed is_valid()"

def test_gate_allows_cheap_op_on_battery():
    passport = PASSPORT_REGISTRY["tier0_scan"]  # battery_allowed=True, trivial
    snapshot = SystemSnapshot(on_battery=True, user_idle=False, available_ram_mb=8000,
                               thermal_state=ThermalState.ATTENTIVE, user_is_active=False)
    decision = passport_gate(passport, mock_governor(ThermalState.ATTENTIVE), snapshot)
    assert decision == GateDecision.ALLOW

def test_gate_defers_expensive_op_on_battery():
    passport = PASSPORT_REGISTRY["ocr_full_document"]  # battery_allowed=False
    snapshot = SystemSnapshot(on_battery=True, ...)
    decision = passport_gate(passport, mock_governor(ThermalState.CALM), snapshot)
    assert decision == GateDecision.DEFER

def test_gate_defers_idle_op_when_user_active():
    passport = PASSPORT_REGISTRY["embed_batch_idle"]  # requires_idle=True
    snapshot = SystemSnapshot(user_idle=False, on_battery=False, ...)
    decision = passport_gate(passport, mock_governor(ThermalState.CALM), snapshot)
    assert decision == GateDecision.DEFER

def test_gate_defers_when_thermal_floor_not_met():
    passport = PASSPORT_REGISTRY["ocr_single_page"]  # thermal_floor=CALM
    decision = passport_gate(passport, mock_governor(ThermalState.ATTENTIVE), ...)
    assert decision == GateDecision.DEFER

def test_expensive_battery_allowed_raises_at_construction():
    with pytest.raises(ValueError, match="battery_allowed"):
        OperationPassport(
            operation_type="bad_op",
            cost_tier=CostTier.EXPENSIVE,
            battery_allowed=True,  # invalid combination
            # ... other fields
        ).is_valid()

def test_unknown_operation_type_raises_at_enqueue():
    with pytest.raises(UnknownOperationType):
        enqueue_job(file_id=1, operation_type="nonexistent_op")
```

### vector_index.py — VectorIndex Protocol Tests

Both `UsearchVectorIndex` and `NumpyFallbackIndex` must pass the same suite:

```python
@pytest.fixture(params=["usearch", "numpy"])
def vector_index(request, tmp_path):
    return get_vector_index({"backend": request.param, "path": tmp_path / "idx"})

def test_add_then_search_finds_it(vector_index):
    vec = np.random.rand(384).astype(np.float32)
    vector_index.add(file_id=42, vector=vec)
    results = vector_index.search(query=vec, k=1)
    assert results[0][0] == 42

def test_delete_removes_from_search(vector_index):
    vec = np.random.rand(384).astype(np.float32)
    vector_index.add(file_id=99, vector=vec)
    vector_index.delete(file_id=99)
    results = vector_index.search(query=vec, k=5)
    assert all(fid != 99 for fid, _ in results)

def test_allowlist_restricts_results(vector_index):
    vecs = {i: np.random.rand(384).astype(np.float32) for i in range(10)}
    for fid, vec in vecs.items():
        vector_index.add(file_id=fid, vector=vec)
    query = vecs[3]
    results = vector_index.search(query=query, k=5, allowlist=[3, 4, 5])
    assert all(fid in {3, 4, 5} for fid, _ in results)

def test_save_load_roundtrip(vector_index, tmp_path):
    vec = np.random.rand(384).astype(np.float32)
    vector_index.add(file_id=7, vector=vec)
    vector_index.save(tmp_path / "saved_idx")
    loaded = get_vector_index({"backend": ..., "path": tmp_path / "saved_idx"})
    loaded.load(tmp_path / "saved_idx")
    results = loaded.search(query=vec, k=1)
    assert results[0][0] == 7

def test_count_reflects_adds_and_deletes(vector_index):
    for i in range(5):
        vector_index.add(file_id=i, vector=np.random.rand(384).astype(np.float32))
    assert vector_index.count() == 5
    vector_index.delete(file_id=2)
    assert vector_index.count() == 4
```

---

## Design Decisions

1. **WAL written before every filesystem call.** The only way to guarantee recovery is to have a persistent record of intent before touching the disk. This adds one DB write per file operation but makes all crash scenarios deterministic.

2. **send2trash for all deletes.** Source files that have been successfully copied and verified are sent to Trash, never permanently deleted with `os.unlink`. A bug that causes a false "copy verified" result leaves the user with a recoverable file in Trash rather than permanent data loss.

3. **Hamming distance threshold of 5 for near-dedup.** At SimHash Hamming ≤ 5 on 64-bit hashes, the expected false positive rate for genuinely different 500-word documents is < 0.1%. The threshold is intentionally conservative — we prefer missing a near-duplicate over wrongly grouping distinct files.

4. **NFC normalization on write, not on read.** All paths stored in the DB are NFC normalized at ingest time (`unicodedata.normalize("NFC", path)`). This avoids per-query normalization overhead and ensures that macOS (which uses NFD internally for HFS+) and Linux test environments agree.

5. **FTS5 stores three representations per file.** The `files_fts` table stores `filename` (original), `nfc_text` (accent-preserved NFC), and `greeklish_text` (Greeklish transliteration + accent-stripped). Queries match against all three columns so users can search in any romanization style.

6. **iCloud stubs detected by 0-byte + xattr, not by size alone.** A 0-byte file could be an intentionally empty placeholder. The `com.apple.icloud` xattr is the definitive signal. Files that are 0-byte without the xattr are treated normally and will produce an empty embedding (filename-only embed text).

7. **locked state is never stored for iCloud stubs beyond a path record.** The rationale: iCloud files evict and restore unpredictably. Recording them as `locked` provides UI visibility without blocking re-processing when they become available.

---

## Open Questions

1. **WAL recovery on very large scans (30k+ files).** Current WAL table has no partition. At 30k pending intents after a crash, recovery scan may be slow. Consider batching recovery in chunks of 500.

2. **Orphan detection scope.** Should orphan detection scan the entire hub folder on every startup, or only check WAL-pending paths? Full scan catches files moved by Finder into the hub accidentally; WAL-only check is faster but may miss them.

3. **SimHash for non-text files.** The current near-dedup spec assumes text content. For binary files (PDFs with only images, etc.) SimHash of filename+folder metadata is used as fallback. Should the threshold be tighter for metadata-only hashes? (Proposed: Hamming ≤ 2 for metadata-only.)

4. **Greek OCR via ocrmac.** ocrmac uses the macOS Vision framework which has good Greek support. However, it is macOS-only. The test suite must either mock ocrmac on Linux CI or gate Greek OCR tests with `@pytest.mark.skipif(platform.system() != "Darwin", reason="ocrmac macOS only")`.

5. **moondream image captioning latency in scan.** The spec places moondream behind `CURATOR_VISION=1`. If enabled, the scan throughput target (1000 files < 10 minutes) may need to be relaxed. Open question: what is the acceptable latency budget per image file with moondream on M1?

6. **Commit destination path resolution.** When a group is committed, the destination folder is derived from the group label. If the label is Greek (e.g., "Κεφάλαια"), the folder path will contain Greek characters. This is valid on macOS HFS+ but should be tested explicitly with `os.makedirs` + `shutil.move` on a volume with Greek label.
