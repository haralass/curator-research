## ΜΕΡΟΣ 37: EXECUTION LAYER — FILE MOVES, SUBFOLDERS, MANIFEST
*(Έρευνα 2026-05-23)*

---

### ΘΕΜΑ 1: Atomic File Moves σε Rust/macOS

#### 1.1 Είναι πραγματικά atomic το `std::fs::rename`;

**Σύντομη απάντηση: Ναι, αλλά μόνο εντός του ίδιου APFS volume.**

Το APFS (Apple File System) εισήγαγε το **Atomic Safe-Save primitive**: το rename εκτελείται σε single transaction — είτε ολοκληρώνεται πλήρως είτε δεν συμβαίνει καθόλου. Δεν υπάρχει intermediate state. Αυτό στηρίζεται στο copy-on-write metadata scheme του APFS που παρέχει crash protection χωρίς παραδοσιακό journaling overhead.

**Πότε αποτυγχάνει το `std::fs::rename`:**
- **Cross-volume**: Αν `src` και `dst` είναι σε διαφορετικά mount points (π.χ. external drive, network share, ή διαφορετικό APFS container), το OS επιστρέφει `EXDEV` error. Η `std::fs::rename` στη Rust επιστρέφει `ErrorKind::CrossesDevices`.
- **Permissions**: Αν ο χρήστης δεν έχει write permission στον destination directory.
- **Destination exists + is directory**: Αν στον dst υπάρχει directory (όχι file), το rename αποτυγχάνει.

**Σημαντική λεπτομέρεια για Curator:** Τα αρχεία στο `~/Downloads` και οι οργανωμένοι φάκελοι `~/Downloads/Curator/` είναι συνήθως στο **ίδιο APFS volume** — άρα το `std::fs::rename` είναι atomic και ταχύτατο (δεν αντιγράφεται δεδομένο, απλώς αλλάζει directory entry).

#### 1.2 Cross-volume fallback: copy → verify → delete

Αν το rename αποτύχει με `EXDEV`, η σωστή υλοποίηση:
1. `std::fs::copy(src, dst_tmp)` — αντίγραφο σε temporary path (`.curator_tmp_<uuid>`)
2. Verify SHA-256 hash: src == dst_tmp
3. Atomic rename `dst_tmp → dst` (εντός ίδιου volume τώρα)
4. `std::fs::remove_file(src)` — διαγραφή original μόνο μετά από επιτυχή verify

Ο SHA-256 υπολογισμός γίνεται με το crate `sha2` + `BufReader` σε 8KB chunks.

#### 1.3 Extended Attributes (xattrs) κατά τη μετακίνηση

Το macOS `mv` command διατηρεί xattrs αυτόματα (same-volume rename). Αλλά με cross-volume copy, τα xattrs **χάνονται** αν δεν τα χειριστείς manually. Χρήση του crate `xattr` (Stebalien/xattr):

- `xattr::list(path)` — λίστα όλων των attribute names
- `xattr::get(path, name)` — τιμή attribute
- `xattr::set(path, name, value)` — εγγραφή attribute στο destination

Κρίσιμα macOS xattrs που πρέπει να διατηρηθούν:
- `com.apple.quarantine` — "downloaded from internet" flag (Gatekeeper)
- `com.apple.metadata:kMDItemWhereFroms` — Spotlight origin URL
- `com.apple.FinderInfo` — Finder color tags

#### 1.4 Conflict Resolution

Αν στον destination υπάρχει ήδη αρχείο με το ίδιο όνομα:
- Έλεγχος αν είναι identical (SHA-256 match) → skip, log as duplicate
- Αν διαφορετικό → rename suffix: `filename_2.ext`, `filename_3.ext`, etc.
- Max retries: 99 (μετά → error)

#### 1.5 Υλοποίηση: `safe_move()`

```rust
use std::path::{Path, PathBuf};
use std::io::{self, BufReader, Read};
use sha2::{Sha256, Digest};
use chrono::Utc;
use serde_json::json;

#[derive(Debug, serde::Serialize)]
pub struct UndoEntry {
    pub op: String,          // "move"
    pub src: PathBuf,
    pub dst: PathBuf,
    pub timestamp: String,
    pub xattrs_preserved: bool,
}

pub fn sha256_of_file(path: &Path) -> io::Result<String> {
    let file = std::fs::File::open(path)?;
    let mut reader = BufReader::with_capacity(8192, file);
    let mut hasher = Sha256::new();
    let mut buf = [0u8; 8192];
    loop {
        let n = reader.read(&mut buf)?;
        if n == 0 { break; }
        hasher.update(&buf[..n]);
    }
    Ok(format!("{:x}", hasher.finalize()))
}

fn copy_xattrs(src: &Path, dst: &Path) -> io::Result<()> {
    for attr_name in xattr::list(src)? {
        if let Some(value) = xattr::get(src, &attr_name)? {
            let _ = xattr::set(dst, &attr_name, &value);
            // Best-effort: δεν fail αν dst FS δεν υποστηρίζει xattrs
        }
    }
    Ok(())
}

fn resolve_conflict(dst: &Path) -> PathBuf {
    if !dst.exists() { return dst.to_path_buf(); }
    let stem = dst.file_stem().unwrap_or_default().to_string_lossy();
    let ext  = dst.extension().map(|e| format!(".{}", e.to_string_lossy()))
                              .unwrap_or_default();
    let parent = dst.parent().unwrap_or(Path::new("."));
    for i in 2..=99 {
        let candidate = parent.join(format!("{}_{}{}", stem, i, ext));
        if !candidate.exists() { return candidate; }
    }
    // Fallback: timestamp suffix
    parent.join(format!("{}_{}{}", stem, Utc::now().timestamp(), ext))
}

pub fn safe_move(
    src: &Path,
    dst_dir: &Path,
    undo_log: &mut Vec<UndoEntry>,
) -> io::Result<PathBuf> {
    // 1. Δημιούργησε destination directory αν δεν υπάρχει
    std::fs::create_dir_all(dst_dir)?;

    let filename = src.file_name().ok_or_else(|| {
        io::Error::new(io::ErrorKind::InvalidInput, "no filename")
    })?;
    let raw_dst = dst_dir.join(filename);

    // 2. Conflict resolution
    let dst = resolve_conflict(&raw_dst);

    // 3. Ελέγξτε αν source και destination είναι στο ίδιο volume
    match std::fs::rename(src, &dst) {
        Ok(_) => {
            // Same-volume atomic rename: xattrs διατηρούνται αυτόματα
            undo_log.push(UndoEntry {
                op: "move".into(),
                src: src.to_path_buf(),
                dst: dst.clone(),
                timestamp: Utc::now().to_rfc3339(),
                xattrs_preserved: true,
            });
            Ok(dst)
        }
        Err(e) if e.kind() == io::ErrorKind::CrossesDevices => {
            // Cross-volume fallback: copy → verify → preserve xattrs → delete
            let tmp_dst = dst_dir.join(format!(
                ".curator_tmp_{}", uuid::Uuid::new_v4()
            ));
            std::fs::copy(src, &tmp_dst)?;

            // Verify integrity
            let src_hash = sha256_of_file(src)?;
            let dst_hash = sha256_of_file(&tmp_dst)?;
            if src_hash != dst_hash {
                let _ = std::fs::remove_file(&tmp_dst);
                return Err(io::Error::new(
                    io::ErrorKind::Other,
                    "SHA-256 mismatch after cross-volume copy",
                ));
            }

            // Preserve xattrs manually
            let xattrs_ok = copy_xattrs(src, &tmp_dst).is_ok();

            // Atomic rename tmp → final dst (same volume now)
            std::fs::rename(&tmp_dst, &dst)?;

            // Διέγραψε source μόνο μετά επιτυχή verify + rename
            std::fs::remove_file(src)?;

            undo_log.push(UndoEntry {
                op: "move".into(),
                src: src.to_path_buf(),
                dst: dst.clone(),
                timestamp: Utc::now().to_rfc3339(),
                xattrs_preserved: xattrs_ok,
            });
            Ok(dst)
        }
        Err(e) => Err(e),
    }
}
```

**Dependencies στο `Cargo.toml`:**
```toml
sha2    = "0.10"
xattr   = "1.3"
uuid    = { version = "1", features = ["v4"] }
chrono  = { version = "0.4", features = ["serde"] }
```

---

### ΘΕΜΑ 2: Undo Log Design

#### 2.1 Πού αποθηκεύουμε το undo log;

**Απόφαση: JSON files στο `~/.curator/undo_logs/`**, όχι SQLite.

Λόγοι:
- Το Curator είναι Tauri app — το JSON είναι άμεσα readable από frontend
- Δεν υπάρχει ανάγκη για complex queries
- Human-readable → ο χρήστης μπορεί να δει τι έγινε
- Εύκολο backup/export

**Μορφή filename:** `session_YYYYMMDD_HHMMSS_<uuid8>.json`

**Πόσο ιστορικό κρατάμε:** 30 sessions τελευταία (rotate παλιότερα). Δεν αξίζει να κρατάς >1 μήνα.

#### 2.2 JSON Schema για Undo Log

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "CuratorUndoLog",
  "type": "object",
  "required": ["version", "session_id", "created_at", "status", "operations"],
  "properties": {
    "version":    { "type": "string", "const": "1.0" },
    "session_id": { "type": "string", "description": "UUID v4" },
    "created_at": { "type": "string", "format": "date-time" },
    "status": {
      "type": "string",
      "enum": ["complete", "partial", "undone", "undone_partial"]
    },
    "manifest_id": {
      "type": "string",
      "description": "Pointer στο manifest που δημιούργησε αυτό το session"
    },
    "operations": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "op", "src", "dst", "timestamp", "status"],
        "properties": {
          "id":        { "type": "integer" },
          "op":        { "type": "string", "enum": ["move", "mkdir", "delete"] },
          "src":       { "type": "string", "description": "Absolute path" },
          "dst":       { "type": "string", "description": "Absolute path (null για delete/mkdir)" },
          "timestamp": { "type": "string", "format": "date-time" },
          "status": {
            "type": "string",
            "enum": ["done", "failed", "undone", "skipped"]
          },
          "src_hash":  { "type": "string", "description": "SHA-256 before move" },
          "dst_hash":  { "type": "string", "description": "SHA-256 after move (verify)" },
          "xattrs_preserved": { "type": "boolean" },
          "error":     { "type": "string", "description": "Error message αν failed" }
        }
      }
    },
    "stats": {
      "type": "object",
      "properties": {
        "total":   { "type": "integer" },
        "success": { "type": "integer" },
        "failed":  { "type": "integer" },
        "skipped": { "type": "integer" }
      }
    }
  }
}
```

#### 2.3 Rust Struct

```rust
use serde::{Deserialize, Serialize};
use std::path::PathBuf;

#[derive(Debug, Serialize, Deserialize, Clone, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum OpType { Move, Mkdir, Delete }

#[derive(Debug, Serialize, Deserialize, Clone, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum OpStatus { Done, Failed, Undone, Skipped }

#[derive(Debug, Serialize, Deserialize, Clone, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum SessionStatus { Complete, Partial, Undone, UndonePartial }

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct FileOperation {
    pub id:                u32,
    pub op:                OpType,
    pub src:               PathBuf,
    pub dst:               Option<PathBuf>,
    pub timestamp:         String,       // RFC3339
    pub status:            OpStatus,
    pub src_hash:          Option<String>,
    pub dst_hash:          Option<String>,
    pub xattrs_preserved:  bool,
    pub error:             Option<String>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct UndoSession {
    pub version:     String,             // "1.0"
    pub session_id:  String,             // UUID v4
    pub created_at:  String,             // RFC3339
    pub status:      SessionStatus,
    pub manifest_id: Option<String>,
    pub operations:  Vec<FileOperation>,
}
```

#### 2.4 Undo Logic — τι γίνεται αν το αρχείο τροποποιήθηκε;

Αλγόριθμος undo για κάθε operation:
1. Ελέγξτε αν το `dst` υπάρχει ακόμα
2. Υπολόγισε SHA-256 του current `dst`
3. Αν `current_hash != original dst_hash` → **το αρχείο τροποποιήθηκε μετά το move**
   - Προειδοποίηση στον χρήστη: "Αυτό το αρχείο έχει αλλάξει. Θέλεις να το μετακινήσεις πάλι;"
   - Default: skip (μην κάνεις overwrite modified file)
4. Αν `current_hash == original dst_hash` → ασφαλές undo: `safe_move(dst, src_dir)`
5. Undo σε αντίστροφη σειρά operations (τελευταίο πρώτο)

```rust
pub fn undo_session(session: &UndoSession) -> Vec<String> {
    let mut warnings = Vec::new();
    let ops_reversed: Vec<_> = session.operations.iter().rev()
        .filter(|op| op.status == OpStatus::Done)
        .collect();

    for op in ops_reversed {
        match op.op {
            OpType::Move => {
                let dst = match &op.dst { Some(d) => d, None => continue };
                if !dst.exists() {
                    warnings.push(format!("SKIP: {} no longer exists", dst.display()));
                    continue;
                }
                // Hash check
                if let (Some(original_hash), Ok(current_hash)) = (
                    &op.dst_hash,
                    sha256_of_file(dst)
                ) {
                    if current_hash != *original_hash {
                        warnings.push(format!(
                            "WARN: {} was modified after move — skipping undo",
                            dst.display()
                        ));
                        continue;
                    }
                }
                let src_dir = op.src.parent().unwrap_or(std::path::Path::new("."));
                let mut dummy_log = Vec::new();
                let _ = safe_move(dst, src_dir, &mut dummy_log);
            }
            OpType::Mkdir => {
                // Μόνο αν κενό directory
                if let Some(dst) = &op.dst {
                    if dst.exists() {
                        if std::fs::read_dir(dst).map(|mut d| d.next().is_none()).unwrap_or(false) {
                            let _ = std::fs::remove_dir(dst);
                        } else {
                            warnings.push(format!("SKIP mkdir undo: {} not empty", dst.display()));
                        }
                    }
                }
            }
            OpType::Delete => {
                warnings.push("Cannot undo delete without backup".into());
            }
        }
    }
    warnings
}
```

---

### ΘΕΜΑ 3: Subfolder Creation Logic

#### 3.1 Πότε δημιουργούμε subfolders; — Ερευνητική βάση

Από UX research (Sortio, uxdesign.cc) και file organizer projects:
- **< 10 αρχεία**: Δεν αξίζει subfolder — μικρός cognitive load
- **10–30 αρχεία**: Gray zone — μόνο αν τα αρχεία έχουν ξεκάθαρα sub-θέματα
- **30–50 αρχεία**: Κατώφλι για subfolder creation
- **> 50 αρχεία**: Πάντα subfolder

**Επιλεγμένο threshold για Curator: N = 25 αρχεία**

Λογική: ~25 είναι η μέση τιμή μεταξύ "too few" (< 10) και "clearly too many" (> 50). Αντανακλά το cognitive sweet spot.

#### 3.2 Cosine Distance Guard

Αν τα αρχεία ενός cluster είναι θεματικά πολύ κοντά (cosine distance < 0.25 μεταξύ sub-centroids), δεν αξίζει να τα χωρίσεις — η διαφορά δεν είναι αρκετά ξεκάθαρη για τον χρήστη.

Από HERCULES paper (arXiv:2506.19992): η αναδρομική ιεραρχία σταματά όταν ο optimal k = 1 (δηλαδή δεν υπάρχει λόγος διαχωρισμού).

#### 3.3 Max Depth Enforcement

Max depth = 3 levels (αποφασισμένο). Enforce μέσω depth counter που περνά recursive.

```python
from sklearn.cluster import AgglomerativeClustering
from sklearn.metrics.pairwise import cosine_distances
import numpy as np
from typing import Optional

MAX_DEPTH = 3
MIN_FILES_FOR_SUBFOLDER = 25
MIN_COSINE_DISTANCE = 0.25    # κάτω από αυτό → δεν αξίζει να χωρίσεις


def should_create_subfolders(
    embeddings: np.ndarray,
    n_files: int,
    current_depth: int,
) -> bool:
    """
    Αποφασίζει αν πρέπει να δημιουργηθούν subfolders για ένα cluster.

    Args:
        embeddings:     (N, D) array με τα embeddings των αρχείων
        n_files:        Πλήθος αρχείων στο cluster
        current_depth:  Τρέχον βάθος (0 = root, max = MAX_DEPTH-1)

    Returns:
        True αν αξίζει να δημιουργηθούν subfolders
    """
    # 1. Depth limit
    if current_depth >= MAX_DEPTH - 1:
        return False

    # 2. Minimum size threshold
    if n_files < MIN_FILES_FOR_SUBFOLDER:
        return False

    # 3. Cosine distance guard: έλεγξε αν υπάρχει πραγματική διαφοροποίηση
    if len(embeddings) < 2:
        return False

    # Δοκίμασε split σε 2 clusters και μέτρησε απόσταση centroids
    clusterer = AgglomerativeClustering(
        n_clusters=2,
        metric="cosine",
        linkage="average",
    )
    labels = clusterer.fit_predict(embeddings)
    mask_0 = labels == 0
    mask_1 = labels == 1

    if mask_0.sum() == 0 or mask_1.sum() == 0:
        return False

    centroid_0 = embeddings[mask_0].mean(axis=0, keepdims=True)
    centroid_1 = embeddings[mask_1].mean(axis=0, keepdims=True)
    dist = cosine_distances(centroid_0, centroid_1)[0, 0]

    # Αν centroids είναι πολύ κοντά → δεν αξίζει διαχωρισμός
    return dist >= MIN_COSINE_DISTANCE


def create_subfolders(
    files: list[dict],           # [{"path": str, "embedding": np.ndarray, "name": str}]
    parent_folder_name: str,
    current_depth: int,
    get_cluster_label: callable, # fn(centroid_embedding) -> str  (LLM call)
) -> dict:
    """
    Δημιουργεί subfolder structure για ένα cluster αρχείων.

    Returns:
        {
          "subfolders": [
            {
              "name": "Invoices",
              "files": [...],
              "depth": 1,
              "subfolders": [...]   # αναδρομικά, αν χρειαστεί
            }
          ]
        }
    """
    embeddings = np.array([f["embedding"] for f in files])
    n_files = len(files)

    if not should_create_subfolders(embeddings, n_files, current_depth):
        return {"subfolders": [], "files": files}

    # Αυτόματη επιλογή k: sqrt(N) ως ευρετικό, clamp [2, 8]
    k = int(np.clip(np.sqrt(n_files), 2, 8))

    clusterer = AgglomerativeClustering(
        n_clusters=k,
        metric="cosine",
        linkage="average",
    )
    labels = clusterer.fit_predict(embeddings)

    subfolders = []
    for cluster_id in range(k):
        mask = labels == cluster_id
        cluster_files = [f for f, m in zip(files, mask) if m]
        if not cluster_files:
            continue

        # Centroid για LLM label generation
        cluster_embeddings = embeddings[mask]
        centroid = cluster_embeddings.mean(axis=0)
        subfolder_name = get_cluster_label(centroid)

        # Αναδρομικά sub-subfolders
        nested = create_subfolders(
            cluster_files,
            subfolder_name,
            current_depth + 1,
            get_cluster_label,
        )

        subfolders.append({
            "name":       subfolder_name,
            "depth":      current_depth + 1,
            "files":      cluster_files if not nested["subfolders"] else [],
            "subfolders": nested["subfolders"],
        })

    return {"subfolders": subfolders}
```

**Σύνοψη παραμέτρων:**

| Παράμετρος | Τιμή | Λόγος |
|---|---|---|
| `MIN_FILES_FOR_SUBFOLDER` | 25 | UX research sweet spot |
| `MIN_COSINE_DISTANCE` | 0.25 | Ξεκάθαρη θεματική διαφορά |
| `MAX_DEPTH` | 3 | Cognitive load limit |
| k selection | √N, clamp [2,8] | Balanced granularity |

---

### ΘΕΜΑ 4: JSON Manifest Design

#### 4.1 Inspiration: Terraform Plan

Το Terraform `plan` output παρέχει εξαιρετικό UX pattern:
- Summary line: `Plan: 5 to add, 3 to change, 2 to destroy`
- Κάθε resource: action type + reason + before/after state
- Machine-readable JSON + human-readable text

Για Curator: `Plan: 47 moves, 5 new folders, 3 duplicates found`

#### 4.2 Πλήρης JSON Manifest Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "CuratorManifest",
  "type": "object",
  "required": ["version", "manifest_id", "created_at", "summary", "execution_plan"],
  "properties": {

    "version":     { "type": "string", "const": "1.0" },
    "manifest_id": { "type": "string", "description": "UUID v4" },
    "created_at":  { "type": "string", "format": "date-time" },
    "dry_run":     { "type": "boolean", "default": true },

    "summary": {
      "type": "object",
      "description": "Human-readable plan line: '47 to move, 5 new folders, 3 duplicates'",
      "properties": {
        "total_files_scanned": { "type": "integer" },
        "total_moves":         { "type": "integer" },
        "auto_approve_count":  { "type": "integer",
          "description": "confidence >= 0.85, tier=auto" },
        "needs_review_count":  { "type": "integer",
          "description": "confidence 0.60-0.84, tier=review" },
        "new_folders_count":   { "type": "integer" },
        "new_subfolders_count":{ "type": "integer" },
        "duplicates_found":    { "type": "integer" },
        "antilibrary_count":   { "type": "integer",
          "description": "Αρχεία που ο χρήστης δεν έχει ανοίξει ποτέ" },
        "skipped_count":       { "type": "integer",
          "description": "confidence < 0.60, δεν μετακινούμε" },
        "estimated_bytes_moved": { "type": "integer" }
      }
    },

    "execution_plan": {
      "type": "object",
      "description": "Ordered steps για εκτέλεση — πρώτα mkdirs, μετά moves",
      "properties": {

        "step_1_create_folders": {
          "type": "array",
          "description": "Πρέπει να εκτελεστεί ΠΡΙΝ τα moves",
          "items": {
            "type": "object",
            "required": ["folder_id", "path", "parent_cluster", "depth"],
            "properties": {
              "folder_id":      { "type": "string" },
              "path":           { "type": "string", "description": "Absolute path" },
              "parent_cluster": { "type": "string", "description": "Cluster ID" },
              "depth":          { "type": "integer", "minimum": 1, "maximum": 3 },
              "file_count":     { "type": "integer" },
              "label_reason":   { "type": "string",
                "description": "Γιατί αυτό το label; (LLM explanation)" }
            }
          }
        },

        "step_2_moves": {
          "type": "array",
          "description": "Αλφαβητικά ταξινομημένα ανά src path για determinism",
          "items": {
            "type": "object",
            "required": ["move_id", "src", "dst", "confidence", "tier", "reason"],
            "properties": {
              "move_id":     { "type": "integer" },
              "src":         { "type": "string", "description": "Absolute source path" },
              "dst":         { "type": "string", "description": "Absolute destination path" },
              "cluster_id":  { "type": "string" },
              "folder_id":   { "type": "string",
                "description": "Pointer στο step_1 folder_id — dependency" },
              "confidence":  { "type": "number", "minimum": 0, "maximum": 1 },
              "tier": {
                "type": "string",
                "enum": ["auto", "review", "skip"],
                "description": "auto: >=0.85 | review: 0.60-0.84 | skip: <0.60"
              },
              "reason":      { "type": "string",
                "description": "One-line explanation: 'Tax document 2024 → Finance/Tax'" },
              "file_size_bytes": { "type": "integer" },
              "last_accessed":   { "type": "string", "format": "date-time" },
              "conflict_detected": {
                "type": "boolean",
                "description": "True αν υπάρχει ήδη αρχείο στον dst" },
              "conflict_resolution": {
                "type": "string",
                "enum": ["rename_suffix", "skip_duplicate", "overwrite", "pending"],
                "description": "pending = ζήτα από χρήστη" },
              "status": {
                "type": "string",
                "enum": ["pending", "executed", "skipped", "failed"],
                "default": "pending"
              }
            }
          }
        }
      }
    },

    "duplicates": {
      "type": "array",
      "description": "Groups αρχείων με identical SHA-256",
      "items": {
        "type": "object",
        "properties": {
          "hash":          { "type": "string" },
          "size_bytes":    { "type": "integer" },
          "files":         { "type": "array", "items": { "type": "string" } },
          "keep":          { "type": "string",
            "description": "Absolute path του αρχείου που κρατάμε (newest mtime)" },
          "action":        { "type": "string", "enum": ["trash", "skip", "pending"] }
        }
      }
    },

    "antilibrary": {
      "type": "array",
      "description": "Αρχεία που ο χρήστης δεν έχει ανοίξει ποτέ (kMDItemLastUsedDate = null)",
      "items": {
        "type": "object",
        "properties": {
          "path":           { "type": "string" },
          "size_bytes":     { "type": "integer" },
          "created_at":     { "type": "string", "format": "date-time" },
          "suggested_action": { "type": "string",
            "enum": ["archive", "review", "keep"] }
        }
      }
    },

    "conflicts": {
      "type": "array",
      "description": "Αντικρουόμενα moves — δύο αρχεία θέλουν το ίδιο dst",
      "items": {
        "type": "object",
        "properties": {
          "dst":           { "type": "string" },
          "competing_moves": { "type": "array", "items": { "type": "integer" },
            "description": "move_ids που διεκδικούν το ίδιο dst" },
          "resolution":    { "type": "string", "enum": ["rename_suffix", "pending"] }
        }
      }
    },

    "execution_state": {
      "type": "object",
      "description": "Tracking partial execution — αν κοπεί η εκτέλεση στη μέση",
      "properties": {
        "folders_created":  { "type": "integer" },
        "moves_executed":   { "type": "integer" },
        "moves_failed":     { "type": "integer" },
        "last_executed_id": { "type": "integer" },
        "completed":        { "type": "boolean" },
        "undo_session_id":  { "type": "string",
          "description": "Pointer στο UndoSession που δημιουργήθηκε" }
      }
    }

  }
}
```

#### 4.3 Dependency Ordering — Βασική Αρχή

Η εκτέλεση του manifest ακολουθεί αυστηρή σειρά:

```
STEP 1: Create all folders (step_1_create_folders)
  └─ Topological sort: parent folders πριν child folders
  └─ depth=1 πρώτα, μετά depth=2, μετά depth=3

STEP 2: Execute moves (step_2_moves)
  └─ Κάθε move έχει folder_id pointer
  └─ Βεβαιώσου ότι το folder_id exists πριν το move
  └─ Σε case partial execution: resume από last_executed_id + 1
```

#### 4.4 Conflict Detection μεταξύ Moves

Πριν εκτέλεση, scan για conflicts:
```python
def detect_manifest_conflicts(moves: list[dict]) -> list[dict]:
    """Βρες moves που διεκδικούν το ίδιο dst path."""
    dst_to_moves = {}
    for move in moves:
        dst = move["dst"]
        dst_to_moves.setdefault(dst, []).append(move["move_id"])

    conflicts = []
    for dst, move_ids in dst_to_moves.items():
        if len(move_ids) > 1:
            conflicts.append({
                "dst": dst,
                "competing_moves": move_ids,
                "resolution": "pending"
            })
    return conflicts
```

#### 4.5 Summary Line Generation (Terraform-inspired)

```rust
pub fn format_plan_summary(manifest: &CuratorManifest) -> String {
    let s = &manifest.summary;
    format!(
        "Plan: {} to move ({} auto / {} review), {} new folders, {} duplicates, {} antilibrary items",
        s.total_moves,
        s.auto_approve_count,
        s.needs_review_count,
        s.new_folders_count + s.new_subfolders_count,
        s.duplicates_found,
        s.antilibrary_count,
    )
}
// Output: "Plan: 47 to move (38 auto / 9 review), 6 new folders, 3 duplicates, 12 antilibrary items"
```

---

### ΣΥΝΟΨΗ ΜΕΡΟΥΣ 37

| Θέμα | Κύριο εύρημα |
|---|---|
| **Atomic moves** | `std::fs::rename` είναι atomic σε APFS same-volume. Cross-volume: copy→verify(SHA-256)→rename\_tmp→delete\_src |
| **xattrs** | Χρήση `xattr` crate. Same-volume: αυτόματο. Cross-volume: manual copy με `xattr::list` + `xattr::set` |
| **Conflict** | resolve\_conflict() με `_2`, `_3` suffix. Hash check για duplicates |
| **Undo log** | JSON στο `~/.curator/undo_logs/`, 30 sessions max, SHA-256 hash check πριν undo |
| **Subfolders** | Threshold N=25, cosine\_distance >= 0.25, max\_depth=3, k=√N clamp [2,8] |
| **Manifest** | Terraform-inspired: step\_1 (mkdirs) → step\_2 (moves), folder\_id pointers, partial execution tracking, conflict detection |

**Επόμενα βήματα:**
- Ενσωμάτωση `safe_move()` στο Tauri command handler
- Tauri event streaming για progress UI (`emit("move_progress", {current, total})`)
- Frontend dry-run preview με manifest JSON rendering
- Test: cross-volume move με external USB drive

---

*Πηγές: Rust std::fs docs, APFS Guide (Apple Developer), xattr crate (Stebalien), HERCULES paper arXiv:2506.19992, Sortio UX research, Terraform JSON format docs, llama-fs (iyaja), scikit-learn AgglomerativeClustering docs*
