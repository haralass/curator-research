# R2 — Duplicate & Version Detection
**Curator Research — New Vision Series**
*Written: 2026-06-08 | Status: Draft for IUI 2027 track*

---

## Overview

Duplicate and version detection is Curator's **mandatory early gate**: it must fire before embedding, OCR, or any expensive downstream operation. At 30,000+ files, a naive O(n²) comparison is computationally infeasible (900M pairs). This document specifies a practical, layered pipeline that is accurate, fast on M1 hardware, and produces group-level output ("Duplicate Families") for Curator's Review Hub.

The pipeline is structured in three tiers of increasing cost, each acting as a pre-filter for the next:

```
Tier 1 — Exact Duplicate Gate       (file size + partial hash + full SHA-256)
Tier 2 — Near-Duplicate Gate         (SimHash for text; pHash/dHash for images)
Tier 3 — Version Family Detection    (RETSim cosine + filename patterns + temporal)
```

Files exit at the first tier that classifies them. Only files that pass through all three gates are forwarded to embedding and clustering.

---

## 1. Method Comparison Tables

### 1.1 Exact Duplicate Detection

| Method | Accuracy | Speed (M1 SSD) | Library / Tool | Threshold | Recommended |
|---|---|---|---|---|---|
| Full SHA-256 | 100% (cryptographic) | ~2.1 GB/s (CryptoKit) | `hashlib` / `CryptoKit` | Identical hash | Yes — final gate |
| Size pre-filter | N/A (pre-filter only) | I/O bound, very fast | `os.stat()` | Exact byte match | Yes — first filter |
| Partial hash (first 4 KB) | ~99.9% (pre-filter) | ~10× faster than full | `hashlib` on slice | Identical partial hash | Yes — second filter |
| Header + Footer hash (fclones model) | ~99.95% (pre-filter) | Fast; good for large files | Custom read | Both ends identical | Yes for files > 1 MB |
| inode / hardlink check | 100% for hardlinks | O(1) per file | `os.stat().st_ino` | Same inode, same dev | Yes — before hashing |
| APFS clone detection | Unreliable | Requires `fcntl` | `apfs-clone-checker` | Clone flag | No — flag unreliable |

**Notes:**
- **fclones** (Rust, 2,800 GitHub stars) processes 1.46M files / 316 GB in 34.59 seconds on SSD using a 6-stage pipeline: size group → inode dedup → header hash → footer hash → full hash. Each stage prunes non-matching groups early.
- **SHA-256 on M1** delivers 2.1 GB/s via Apple CryptoKit. At 30,000 files averaging 500 KB each (15 GB total), full hashing of all files = ~7 seconds. In practice, the size + partial-hash pre-filters eliminate most files, so actual full-hash work is a fraction of this.
- **APFS clonefile**: APFS does not auto-deduplicate files copied from external sources. The clone flag (`CLONE_NOFOLLOW`) records whether a file has ever been cloned, but does not identify the other member of the clone pair and is unreliable after edits. Curator should not rely on it as a signal — use content hashing instead.
- **Zero-byte files**: group separately; all zero-byte files are trivially identical. Never hash them. Present as a "Empty Files" family.

**Curator decision:** Use the fclones 6-stage pipeline model in Python: `stat()` size grouping → inode dedup → partial hash (first 4 KB + last 4 KB) → full SHA-256 within matching groups. Files with identical SHA-256 are exact duplicates. Skip APFS clone detection.

---

### 1.2 Near-Duplicate Text Detection

| Method | Paper / Origin | Accuracy | Speed at 30k docs | Memory | Library | Threshold | Recommended |
|---|---|---|---|---|---|---|---|
| SimHash | Charikar 2002; Google production (Manku et al., WWW 2007) | High for web docs; moderate for personal files | O(n) fingerprint gen + O(n) comparison (or O(n log n) with LSH-like permutation index) | Very low (~8 bytes/doc) | `simhash` (PyPI) | Hamming ≤ 3 (64-bit) | Yes — primary pre-filter |
| MinHash + LSH | Broder 1997; `datasketch` | High for Jaccard similarity | O(n·k) index build; sub-linear query | High (~1 KB/doc for 128 permutations) | `datasketch` | Jaccard ≥ 0.8 | No — memory overhead at scale |
| RETSim | Google, ICLR 2024 (arXiv:2311.17264) | State-of-the-art; multilingual; adversarially robust | Slow (neural inference); GPU helps | ~256 bytes/doc embedding | `pip install unisim` (archived May 2026) | Cosine ≥ 0.9 (near-dup); 0.72–0.86 (version) | Yes — version family gate only |
| BM25 / TF-IDF + cosine | Classical IR | Moderate | Fast | Low | `scikit-learn` | Cosine ≥ 0.85 | No — superseded by SimHash |
| Shingling + exact Jaccard | Classical | High | O(n²) | Low | Custom | Jaccard ≥ 0.8 | No — too slow at scale |

**Key findings:**

- **SimHash** (Charikar 2002): produces a 64-bit fingerprint via a weighted bit-voting process over token hashes. Documents with Hamming distance ≤ 3 are ~95%+ similar. Google used SimHash at 8 billion web pages (Manku et al., WWW 2007). At 30,000 personal files, even naive O(n²) Hamming comparison is feasible: 30,000² / 2 = 450M comparisons, each a bitwise XOR + popcount — roughly 0.5 seconds in NumPy on M1.
- **MinHash + LSH** (datasketch): for a 6 GB pre-computed MinHash file, the LSH index consumes 31 GB RAM. At 30,000 documents this is more tractable (~200 MB), but SimHash is simpler and nearly as accurate for personal-file near-duplicate detection.
- **RETSim** (unisim): produces 256-dim metric embeddings. `TextSim().similarity()` returns a score where ≥ 0.9 = near-identical. The package was archived in May 2026 but remains functional. Use it only for the version family gate (comparing candidate pairs already identified by SimHash), not for all-pairs comparison.
- **Personal files vs. web documents**: Web SimHash thresholds (Hamming ≤ 3) were calibrated on HTML pages. Personal files (PDFs, Word docs) are less template-heavy and more structurally consistent. A threshold of Hamming ≤ 5 is appropriate for personal files to catch near-duplicates with minor edits.
- **Text extraction prerequisite**: SimHash and RETSim require text. For PDFs and Word docs, extract text first (pdfminer.six / python-docx). For images of text (scanned PDFs), use OCR (Tesseract) but only after exact-duplicate gate passes — do not OCR known exact duplicates.

**Curator decision:** Run SimHash (Hamming ≤ 5) as the near-duplicate text pre-filter across all text-bearing files. Use RETSim cosine similarity only on candidate pairs to distinguish "near-duplicate" from "version" (cosine ≥ 0.9 = near-dup; 0.72–0.89 = version family candidate).

---

### 1.3 Near-Duplicate Image Detection

| Method | Accuracy | Speed | Library | Threshold | Use Case |
|---|---|---|---|---|---|
| dHash (difference hash) | Good for exact/near-exact | Very fast (~4 ms/image) | `imagehash` | Hamming ≤ 10 | Default — fastest |
| pHash (perceptual hash) | Better than dHash for altered images | Fast (~8 ms/image) | `imagehash` | Hamming ≤ 10 | Photos, resized images |
| aHash (average hash) | Low (many false positives) | Fastest | `imagehash` | — | Not recommended |
| wHash (wavelet hash) | Better than pHash for JPEG artifacts | Moderate | `imagehash` | Hamming ≤ 10 | JPEG photos |
| CNN (MobileNet / EfficientNet) | Best for visually similar but not identical | Slow (GPU preferred) | `imagededup` | Cosine ≥ 0.9 | Near-duplicates with transforms |
| SSIM | Good for screenshots | Moderate (OpenCV) | `skimage.metrics` | SSIM ≥ 0.95 | Screenshot-specific |

**Key findings:**

- **imagededup** (idealo, GitHub): supports PHash, DHash, WHash, AHash, and CNN. CNN is most accurate for near-duplicates with transformations; DHash is fastest for exact duplicates. All methods handle exact duplicates well.
- **Screenshot deduplication**: screenshots are typically pixel-for-pixel or near-pixel-for-pixel captures of UI state. dHash with Hamming ≤ 8 is the most practical approach — it catches screenshots of the same screen with minor cursor movement or timestamp changes. SSIM (≥ 0.95) is more precise but 10× slower. Recommended two-step: dHash pre-filter (Hamming ≤ 8), then SSIM confirmation on candidates.
- **Screenshot vs. photo distinction**: screenshots have uniform noise profiles and sharp edges. Photos have sensor noise and organic content. The two populations should use separate thresholds. Distinguish by EXIF: screenshots lack GPS/lens data and often have `Software: Screenshot` tag.
- **Pre-filter for images**: group by (file size ± 20%) and image dimensions (W×H) before hashing. Images of different dimensions are unlikely to be near-duplicates except for resized versions — run hash comparison only within same-dimension groups, and separately for cross-dimension candidates.
- **2024 research** (MDPI Electronics 15(7):1493, 2025): comparative evaluation of classical perceptual hashes and CNN embedding on UKBench and Amazon Berkeley Objects datasets. Clear trade-off: perceptual hashes are 100× faster; CNN is 20–30% more accurate for near-duplicate pairs. For Curator's use case (30k files, personal Mac), perceptual hashing is the right default with CNN as an optional high-precision mode.

**Curator decision:** Use dHash (Hamming ≤ 10) as the primary image duplicate gate. For candidate pairs identified by dHash, use pHash as a second check (Hamming ≤ 8). CNN (imagededup MobileNet) is a user-activatable "deep scan" mode. For screenshots specifically: dHash ≤ 8, then SSIM ≥ 0.95 confirmation.

---

### 1.4 Version Family Detection

| Signal | Reliability | Implementation | Cost |
|---|---|---|---|
| Filename pattern match | High for explicit versioning | Regex (see §4) | Negligible |
| Temporal proximity | Medium (hours/days apart) | `os.stat().st_mtime` comparison | Negligible |
| Base name similarity | High | Edit distance on stem (Levenshtein ≤ 3) | Negligible |
| Format conversion pair | High for same base name | Extension mapping table | Negligible |
| RETSim cosine 0.72–0.89 | High for text documents | `unisim` TextSim | Medium (inference) |
| TSED AST similarity > 0.23 | High for code files | arXiv:2404.08817 | Medium (parsing) |
| Git log | Perfect for git repos | `git log --follow` | Fast for git repos |

**Curator decision:** Apply version signals in priority order: (1) filename pattern → (2) base name similarity → (3) temporal proximity → (4) format conversion pair → (5) RETSim cosine. Assign version role (see §5) based on combined signal strength.

---

## 2. Detection Pipeline

The Curator duplicate gate runs as a sequential, early-exit pipeline. All stages must complete before any file is forwarded to embedding.

```
INPUT: All files in scope (e.g., ~/Downloads, 30,000 files)

STAGE 0 — Pre-sort and partition
  ├── Exclude zero-byte files → "Empty Files" family (direct to Review Hub)
  ├── Exclude system files (hidden, .DS_Store, etc.)
  └── Partition by broad type: text-bearing | image | binary/archive | code

STAGE 1 — Size Grouping (exact duplicate gate, Tier 1)
  ├── Group all files by exact byte size (os.stat().st_size)
  ├── Singletons (unique size) → EXIT: unique, forward to embedding
  └── Groups of 2+ → proceed to Stage 2

STAGE 2 — Inode / Hardlink Dedup
  ├── Within each size group, check (st_dev, st_ino)
  ├── Files sharing an inode → already the same file (hardlink)
  └── Record hardlink families; mark as `hardlink_duplicate`

STAGE 3 — Partial Hash Pre-filter
  ├── For each size-matched group: read first 4 KB + last 4 KB of each file
  ├── Hash with xxHash (non-cryptographic, ~3× faster than SHA-256 for this use)
  ├── Files with different partial hashes → EXIT: unique, forward to embedding
  └── Files with matching partial hashes → proceed to Stage 4

STAGE 4 — Full SHA-256 Hash
  ├── Hash full file content (SHA-256)
  ├── Files with identical SHA-256 → EXACT DUPLICATE
  │     └── Group into Duplicate Family; mark state: `exact_duplicate`
  │     └── One member designated `candidate_keeper` (most recent mtime or non-backup path)
  │     └── Other members: `duplicate_member` state; DO NOT embed
  └── Different SHA-256 despite matching partial hash → EXIT: unique, forward to embedding

STAGE 5 — Near-Duplicate Text Gate (Tier 2, text-bearing files only)
  ├── Extract text from PDF/DOCX/TXT/MD files (already extracted in Stage 0 partition)
  ├── Compute SimHash fingerprint (64-bit) for each file
  ├── Build in-memory index; compare within extension groups
  ├── Pairs with Hamming distance ≤ 5 → NEAR-DUPLICATE CANDIDATE
  │     └── Run RETSim cosine on candidate pair
  │     └── Cosine ≥ 0.90 → near-duplicate; mark state: `near_duplicate`
  │     └── Cosine 0.72–0.89 → version candidate; proceed to Stage 7
  │     └── Cosine < 0.72 → false positive; EXIT: unique
  └── Hamming distance > 5 → EXIT: unique, forward to embedding

STAGE 6 — Near-Duplicate Image Gate (Tier 2, image files only)
  ├── Compute dHash for each image (imagehash library)
  ├── Group by image dimensions; compare within group
  ├── Pairs with Hamming ≤ 10 → candidate
  │     └── Run pHash check (Hamming ≤ 8)
  │     └── Confirmed → mark state: `near_duplicate_image`
  └── No match → EXIT: unique, forward to embedding

STAGE 7 — Version Family Detection (Tier 3)
  ├── Apply version signals (§4) to all files not yet classified
  ├── Group version candidates by base name cluster
  ├── Assign version role to each member (§5)
  └── Mark state: `version_member`; ALL version members still get embedded
        (version embeddings improve cluster quality downstream)

STAGE 8 — Union-Find Family Grouping (§3)
  ├── Merge all pairwise duplicate/near-duplicate/version relationships
  ├── Output: list of Duplicate Families (exact), Near-Dup Families, Version Chains
  └── Forward non-family files to embedding pipeline

OUTPUT:
  - Exact Duplicate Families → _Curator Review/Duplicates/Exact/
  - Near-Duplicate Families → _Curator Review/Duplicates/Near/
  - Version Chains → _Curator Review/Versions/
  - Unique files → embedding pipeline
```

**Curator decision:** This is the canonical pipeline. Stages 1–4 (exact duplicate gate) must run before any embedding. Stages 5–6 (near-duplicate) run concurrently on their respective partitions. Stage 7 (version) runs after near-duplicate gates. All gates must complete before the embedding worker pool starts.

---

## 3. Family Grouping Algorithm

Individual pairwise relationships (A≈B, B≈C) must be merged into transitive families ({A,B,C}). The correct data structure is **Union-Find (Disjoint Set Union)** with path compression and union by rank.

### 3.1 Union-Find Implementation

```python
class UnionFind:
    def __init__(self, n: int):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x: int) -> int:
        # Path compression
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, x: int, y: int) -> None:
        rx, ry = self.find(x), self.find(y)
        if rx == ry:
            return
        # Union by rank
        if self.rank[rx] < self.rank[ry]:
            rx, ry = ry, rx
        self.parent[ry] = rx
        if self.rank[rx] == self.rank[ry]:
            self.rank[rx] += 1

    def families(self) -> dict[int, list[int]]:
        groups = {}
        for i in range(len(self.parent)):
            root = self.find(i)
            groups.setdefault(root, []).append(i)
        return {k: v for k, v in groups.items() if len(v) > 1}
```

Time complexity: O(α(n)) per operation — effectively constant. At 30,000 files with 5,000 duplicate pairs, this completes in microseconds.

### 3.2 Separation of Exact vs. Near Duplicates

Exact duplicates and near-duplicates should be in **separate families**. Do not merge an exact duplicate group with a near-duplicate group. Version chains are a third family type. The three family types have different Review Hub presentations and different recommended actions.

### 3.3 Family Naming

Automatic family names should be generated from the shared base name of the members, not from generic labels like "Group 001". Algorithm:

1. Extract the stem of each member's filename (remove extension, version tokens, and date tokens).
2. Find the longest common substring across all stems in the family.
3. Title-case the result and strip numeric suffixes.
4. If the common substring is fewer than 4 characters, fall back to the most common directory name + member count.

Examples:
- `{thesis.docx, thesis_v2.docx, thesis_final.docx}` → stem LCS = "thesis" → family name: **"Thesis"**
- `{IMG_4521.jpg, IMG_4521 (1).jpg, IMG_4521-2.jpg}` → LCS = "IMG_4521" → family name: **"IMG 4521"**
- `{budget2024.xlsx, Budget_2024_final.xlsx}` → LCS = "budget" → family name: **"Budget 2024"** (longest non-trivial token)

**Curator decision:** Use Union-Find for all family grouping. Maintain three separate Union-Find structures: one for exact duplicates, one for near-duplicates, one for versions. Generate family names from stem LCS. Minimum family size for Review Hub presentation: 2 members.

---

## 4. Version Detection: Filename Patterns

The following regex patterns cover the vast majority of informal version naming conventions observed in personal file collections. Apply in order; first match wins.

```python
import re

VERSION_PATTERNS = [
    # Explicit version number: thesis_v2.docx, report_v1.3.pdf
    (r'[_\-\s]v(\d+)(?:[\._](\d+))?(?=[_\-\s.]|$)', 'explicit_version'),

    # Copy suffix: file (1).pdf, file - Copy.docx, file copy.pdf
    (r'[\s\-_]\(?(\d+)\)?(?=[_\-\s.]|$)', 'numbered_copy'),
    (r'[\s\-_][-\s]?copy(?:[\s_]\(?(\d+)\)?)?', 'copy_suffix'),

    # Draft/final/submitted/revised
    (r'[\s_\-](draft|wip|working)(?:[\s_](\d+))?', 'draft'),
    (r'[\s_\-](final|approved|release)(?:[\s_](v?\d+))?', 'final'),
    (r'[\s_\-](submitted?|submission)', 'submitted'),
    (r'[\s_\-](revised?|revision|rev[\s_]?(\d+))', 'revision'),
    (r'[\s_\-](archived?|archive)', 'archived'),
    (r'[\s_\-](backup|bak)', 'backup'),

    # Date stamps: report_2024-03-15.pdf, report_20240315.pdf
    (r'[_\-](\d{4}[-_]\d{2}[-_]\d{2}|\d{8})(?=[_\-\s.]|$)', 'date_version'),

    # FINAL-FINAL, FINAL_FINAL2, etc.
    (r'(final[_\-\s]?){2,}', 'hyperfinal'),

    # Sequential numbered: chapter1.md, chapter2.md
    (r'(\d+)(?=[_\-\s.]|$)', 'sequential_number'),
]

# Regex flags: case-insensitive, apply to stem (filename without extension)
COMPILED = [(re.compile(pat, re.IGNORECASE), role) for pat, role in VERSION_PATTERNS]
```

**Temporal proximity rule:** Two files with the same base stem (Levenshtein distance ≤ 3 on stems) created or modified within 30 days of each other are version candidates, regardless of whether they match a filename pattern. Within 7 days → strong version signal.

**Format conversion pair rule:** Files where one is a PDF and the other is DOCX/ODT/PPTX/XLSX of the same base name (Levenshtein ≤ 2 on stems, ignoring extension) are treated as format-conversion pairs. They are version members, not duplicates. Both are kept; the PDF is tagged `format_export`, the source document is `canonical`.

**Code version rule:** For source code files (.py, .js, .ts, .swift, etc.), check git status first (`git log --follow <path>`). If the file is tracked, it has an authoritative version history — do not apply content-based version detection. For untracked code files, apply TSED (AST edit distance, arXiv:2404.08817) with threshold > 0.23 = functionally similar. TSED implementation: `tree-sitter` parser + custom edit distance.

**Curator decision:** Apply all pattern categories. Temporal proximity (≤ 30 days) is a strong supplementary signal. Format conversion pairs must be detected and tagged, not merged into duplicate families.

---

## 5. Version Role Detection

Version family members should be classified into roles from the following 9-class taxonomy, which maps to the document lifecycle literature (Veeva Vault states; NIH Version Control Guidelines).

| Role | Code | Detection Signals | Priority |
|---|---|---|---|
| Draft | `draft` | Filename pattern `draft`, `wip`, `working`; earliest mtime in family; RETSim cosine 0.72–0.85 vs. Final | Low |
| Revision | `revision` | Filename pattern `rev`, `revised`, `v2+`; intermediate mtime; cosine 0.85–0.92 vs. prior | Medium |
| Final | `final` | Filename pattern `final`, `approved`, `release`, `v1.0`; no version token + latest mtime | High |
| Submitted | `submitted` | Filename pattern `submitted`, `submission`; often PDF; mtime matches grant/deadline context | High |
| Archived | `archived` | Filename pattern `archived`, `backup`, `bak`; often in subdirectory `archive/` or `old/` | Low |
| Superseded | `superseded` | Earlier version member when a later Final/Submitted exists | Low |
| Branch | `branch` | Same base, diverged content (RETSim cosine 0.65–0.79 vs. any other member); different stem suffix indicating purpose | Medium |
| Format Export | `format_export` | PDF/PNG/HTML conversion of a source document; detected by format conversion pair rule | Low |
| Reference Copy | `reference_copy` | Filename pattern `ref`, `reference`, `for_reference`; typically read-only | Low |

**Role assignment algorithm:**

```
For each member in a version family:
1. Apply filename pattern regex → candidate role list
2. Sort family members by mtime (oldest first)
3. Oldest member with 'draft' signal → role: Draft
4. Intermediate members with version numbers → role: Revision
5. Member with 'final' or 'approved' signal → role: Final
6. Member with 'submitted' signal → role: Submitted
7. Member with 'backup'/'archive' signal → role: Archived
8. Earlier member where a later Final/Submitted exists → role: Superseded
9. PDF of same stem as DOCX → role: Format Export
10. Default (no signal) → role: Revision (ambiguous)
```

**Keeper selection for version families:** The keeper is not automatically selected — it is surfaced to the user. However, Curator's suggested keeper is the member with role `Final` or `Submitted` (latest mtime wins in ties). If no Final/Submitted exists, the latest Revision is suggested.

**Curator decision:** Implement the 9-class taxonomy. Version role is stored in the file's metadata record. The Review Hub displays the role badge next to each family member. Keeper suggestion is shown but not auto-applied.

---

## 6. Performance Analysis at 30,000 Files

### 6.1 File Profile Assumptions (30,000 files in ~/Downloads)
- Average file size: 500 KB (wide distribution: screenshots 200 KB, PDFs 2 MB, code <50 KB)
- Total data: ~15 GB
- File type distribution (estimated): images 40% (12k), PDFs 20% (6k), Office docs 10% (3k), code 10% (3k), archives 10% (3k), misc 10% (3k)
- Expected duplicate rate: 15–25% (screenshots especially duplicate heavily)

### 6.2 Cost Per Stage

| Stage | Operation | Cost at 30k files | Bottleneck |
|---|---|---|---|
| Stage 0 — Partition | `os.walk` + `stat()` | < 1 second | I/O metadata |
| Stage 1 — Size grouping | In-memory dict | < 0.1 second | CPU |
| Stage 2 — Inode check | `stat()` already cached | < 0.1 second | CPU |
| Stage 3 — Partial hash | Read 8 KB/file × 30k = 240 MB | ~0.5 seconds (SSD) | I/O |
| Stage 4 — Full SHA-256 | Only for partial-hash matches; assume 20% = 6k files × 500 KB = 3 GB | ~1.5 seconds (2.1 GB/s M1) | I/O + CPU |
| Stage 5 — SimHash text | Text extraction + SimHash for 9k text files | ~5–10 seconds | CPU (text extraction) |
| Stage 5b — RETSim | Inference on ~500 candidate pairs (unisim) | ~10–30 seconds CPU; ~2–5 sec GPU | Neural inference |
| Stage 6 — dHash images | 12k images × 4 ms = 48 seconds | ~48 seconds | I/O + CPU |
| Stage 6b — pHash confirm | ~1k candidates × 8 ms | ~8 seconds | CPU |
| Stage 7 — Version detection | Regex + Levenshtein on 30k stems | ~1–2 seconds | CPU |
| Stage 8 — Union-Find | 30k nodes, ~5k pairs | < 0.01 second | CPU |

**Total estimated time: 70–100 seconds on M1 Mac (cold, SSD), 15–25 seconds with warm filesystem cache.**

This is acceptable for a background task. Curator should run the duplicate gate as a low-priority background process with progress reporting to the UI.

### 6.3 O(n²) Avoidance Strategy

The critical insight: **never compare all pairs**. The pipeline's layered filtering ensures that:

- Stage 1 (size grouping) reduces candidate pairs from 450M to ~10k–50k (files of the same size).
- Stage 3 (partial hash) eliminates ~95% of false size-matches, leaving ~500–2,500 pairs.
- Stage 4 (full hash) is run only on these survivors.

For near-duplicate text (Stage 5): SimHash at 30,000 documents produces 30,000 64-bit integers. A vectorized NumPy all-pairs Hamming distance computation takes ~0.5 seconds. This is feasible. For very large corpora (>100k files), switch to LSH-based approximate search using `datasketch` or a custom band-and-hash scheme.

For images (Stage 6): group by image dimensions first, then compare within dimension groups. Most screenshots share common resolutions (e.g., 2560×1600 for Retina MacBook). This reduces comparison count by ~10×.

**Curator decision:** The proposed pipeline avoids O(n²) through size-bucket blocking. No LSH index is needed at 30,000 files for text (SimHash all-pairs is fast enough). For image comparison, group by dimensions. Add a config option to enable `datasketch` LSH for corpora > 100k files.

---

## 7. Review Hub Design for Duplicates

### 7.1 Folder Structure in _Curator Review

```
_Curator Review/
├── Duplicates/
│   ├── Exact/
│   │   ├── Thesis/                    ← family named from stem LCS
│   │   │   ├── thesis.docx            ← suggested keeper (marked ★)
│   │   │   ├── thesis - Copy.docx
│   │   │   └── .curator_family.json   ← machine-readable metadata
│   │   ├── IMG 4521/
│   │   │   ├── IMG_4521.jpg           ← keeper
│   │   │   ├── IMG_4521 (1).jpg
│   │   │   └── IMG_4521 (2).jpg
│   │   └── ...
│   └── Near/
│       ├── Budget Report/
│       │   ├── budget_report_jan.xlsx  ← keeper (suggested)
│       │   ├── budget_report_jan_v2.xlsx
│       │   └── .curator_family.json
│       └── ...
└── Versions/
    ├── Thesis/
    │   ├── thesis_draft.docx          ← role: Draft
    │   ├── thesis_v2.docx             ← role: Revision
    │   ├── thesis_final.docx          ← role: Final ★ (suggested keeper)
    │   ├── thesis_submitted.pdf       ← role: Submitted
    │   └── .curator_family.json
    └── ...
```

### 7.2 `.curator_family.json` Metadata Schema

```json
{
  "family_id": "sha256-prefix-of-keeper",
  "family_name": "Thesis",
  "family_type": "exact_duplicate | near_duplicate | version_chain",
  "created_at": "2026-06-08T14:23:00Z",
  "members": [
    {
      "path": "/Users/.../thesis.docx",
      "role": "final",
      "suggested_keeper": true,
      "similarity_score": 1.0,
      "size_bytes": 245760,
      "mtime": "2025-11-15T09:00:00Z",
      "sha256": "abc123..."
    },
    {
      "path": "/Users/.../thesis - Copy.docx",
      "role": "duplicate_member",
      "suggested_keeper": false,
      "similarity_score": 1.0,
      "size_bytes": 245760,
      "mtime": "2025-11-14T20:30:00Z",
      "sha256": "abc123..."
    }
  ],
  "user_decision": null,
  "user_decided_at": null
}
```

### 7.3 User-Facing Information Per Family

For each Duplicate Family card in the Review Hub, show:
- **Family name** (auto-generated, editable)
- **Family type badge**: "Exact Duplicate" / "Near-Duplicate (N% similar)" / "Version Chain"
- **Member count**: "3 files, 24 MB total"
- **Suggested keeper**: highlighted with ★, showing: filename, size, date, location
- **Other members**: filename, size, date, location, similarity score (for near-dups)
- **Quick preview**: thumbnail (images) or first 200 chars of text (documents)
- **Similarity score**: shown as percentage for near-duplicates; hidden for exact duplicates (redundant)
- **Location context**: if members span multiple directories, show directory tree

### 7.4 User Actions Per Family

| Action | Description | Result |
|---|---|---|
| **Keep suggested, trash rest** | Default action | Keeper stays; others moved to Trash |
| **Keep all as versions** | Reclassify as version chain | Family moved from Exact/Near to Versions |
| **Choose different keeper** | Select a different member | User selects; others trashed |
| **Mark as not duplicate** | False positive | Family dismissed; all files forwarded to embedding |
| **Keep all, do nothing** | No action | Family stays in Review Hub |
| **Trash all** | All are unwanted | All moved to Trash |

**Curator decision:** These six actions cover all real-world user needs. "Keep all as versions" is the escape hatch for false positives in exact duplicate detection (e.g., two files with same SHA-256 but different provenance intent). "Mark as not duplicate" dismisses the family entirely and teaches Curator's false-positive filter.

---

## 8. Prior Art

### Papers

| Paper | Authors | Venue | DOI / arXiv | Relevance |
|---|---|---|---|---|
| "Detecting Near-Duplicates for Web Crawling" | Manku, Jain, Das Sarma | WWW 2007 | https://dl.acm.org/doi/10.1145/1242572.1242592 | SimHash at 8B pages; Hamming distance ≤ 3 threshold validated |
| "Min-Wise Independent Permutations" (MinHash) | Broder et al. | STOC 1998 | https://dl.acm.org/doi/10.1145/276698.276781 | MinHash / Jaccard foundation |
| "In Defense of MinHash Over SimHash" | Shrivastava, Li | arXiv:1407.4416 | https://arxiv.org/abs/1407.4416 | MinHash outperforms SimHash in high-similarity regimes |
| "RETSim: Resilient and Efficient Text Similarity" | Zhang et al. (Google) | ICLR 2024 | https://arxiv.org/abs/2311.17264 | State-of-the-art near-dup text; 256-dim metric embeddings |
| "Revisiting Code Similarity Evaluation with AST Edit Distance" | (TSED) | arXiv:2404.08817 | https://arxiv.org/abs/2404.08817 | TSED > 0.23 = functionally similar code |
| "Comparative Evaluation of Perceptual Hashing and Deep Embedding for Image Deduplication" | — | MDPI Electronics 15(7):1493, 2025 | https://www.mdpi.com/2079-9292/15/7/1493 | pHash vs CNN trade-offs |
| "Near-Duplicate Detection in Web App Model Inference" | Yandrapally et al. | ICSE 2020 | PDF: https://tsigalko18.github.io/assets/pdf/2020-Yandrapally-ICSE.pdf | UI state near-dup detection |

### Libraries / Tools

| Tool | Language | GitHub Stars | Use in Curator |
|---|---|---|---|
| `fclones` | Rust | ~2,800 | Architectural reference for 6-stage exact dup pipeline |
| `imagededup` (idealo) | Python | ~5,000 | Image near-dup detection (dHash, pHash, CNN) |
| `imagehash` | Python | ~3,000 | Perceptual hashing (pHash, dHash, aHash, wHash) |
| `datasketch` | Python | ~2,300 | MinHash + LSH (for future > 100k file scale) |
| `simhash` | Python | ~800 | SimHash fingerprint generation |
| `unisim` (google/unisim) | Python | ~1,200 | RETSim text similarity (archived May 2026, still functional) |
| `dupeGuru` | Python | ~5,200 | UX reference: duplicate review UI |
| `python-Levenshtein` | Python/C | ~1,400 | Fast Levenshtein distance for stem comparison |
| `tree-sitter` | Python/C | ~16,000 | AST parsing for TSED code similarity |

### Commercial Implementations

| Product | Approach | UX Insight |
|---|---|---|
| **Gemini 2 (MacPaw)** | Proprietary; tests ~12 parameters; near-dup awareness | Red Dot award–winning UI; side-by-side preview; family grouping; checkbox selection; respects folder structure |
| **dupeGuru** | Open source; music/image/file modes; results in groups | Results tab shows duplicate sets; Actions tab for batch operations; preview before action |
| **diskDedupe** | APFS-aware; clone detection since v2.0; skips already-cloned files | Shows size savings; APFS clone creation option |
| **Dropbox block dedup** | 4 MB chunk SHA-256; content-addressed object store | Block-level granularity enables cross-file dedup; Merkle tree for file integrity |

---

## 9. Design Decisions

**DD-01: Exact duplicate gate is mandatory before embedding.**
No file that is an exact duplicate of another file in the same ingestion batch shall be embedded independently. The `exact_duplicate` members get a `duplicate_member` state and are excluded from the embedding worker pool. Only the `candidate_keeper` is embedded.

**DD-02: Near-duplicate files are also excluded from embedding.**
Files with `near_duplicate` state (SimHash Hamming ≤ 5, confirmed by RETSim ≥ 0.90) are excluded from embedding. They appear only in the Review Hub.

**DD-03: Version members ARE embedded.**
Files with `version_member` state are embedded, because their content differs meaningfully and contributes to cluster quality. Version chains are surfaced in the Review Hub but the files also appear in their contextual clusters.

**DD-04: The partial hash reads first 4 KB and last 4 KB.**
This catches files that differ only at the end (e.g., truncated copies), which a head-only partial hash would miss. The 4 KB size matches the macOS default page size and is consistent with what `jdupes` uses internally.

**DD-05: Non-cryptographic hash (xxHash) for partial hash; SHA-256 for full hash.**
xxHash is ~3× faster than SHA-256 for short reads and has negligible collision probability at 30,000 files. SHA-256 is used for the full hash because it is the standard for content-addressed deduplication and is acceptable to present as proof of identity to the user.

**DD-06: Image comparison groups by dimensions first.**
Before running dHash, files are grouped by (width, height). Cross-dimension comparison runs only when a filename pattern or temporal signal suggests a resize relationship.

**DD-07: SimHash threshold is Hamming ≤ 5 for personal files, not ≤ 3.**
Google's Hamming ≤ 3 threshold was calibrated on HTML pages with heavy template boilerplate. Personal files (PDFs, DOCX) have less structural noise. A threshold of ≤ 5 is appropriate to catch documents with minor paragraph edits without excessive false positives.

**DD-08: RETSim (unisim) is used only for candidate confirmation, not all-pairs.**
Running RETSim inference on 30,000 documents all-pairs is computationally infeasible without GPU. It is applied only to pairs already identified as candidates by SimHash, typically ≤ 1,000 pairs in a 30k-file corpus.

**DD-09: APFS clonefile is not used as a detection signal.**
The APFS clone flag is unreliable (does not persist after edits; no cross-clone identification). Content hashing is the authoritative signal.

**DD-10: Zero-byte files are always a Duplicate Family.**
All zero-byte files are trivially identical. They are grouped into a single "Empty Files" family without hashing. The user is asked whether to trash all or keep one.

**DD-11: Format conversion pairs (e.g., thesis.docx + thesis.pdf) are version members, not duplicates.**
Converting a DOCX to PDF produces a near-identical but functionally distinct file. Presenting these as "duplicates" and suggesting the user trash one would cause data loss. They are always tagged as `format_export` / `canonical` pairs within a version chain.

**DD-12: Version detection runs on ALL files, not just those that failed near-dup detection.**
A file can pass the near-dup gate (Hamming > 5) but still be a version of another file based on filename patterns and temporal proximity. The version gate is independent of the near-dup gate.

**DD-13: Minimum family size for Review Hub is 2 members.**
Single-member "families" are not shown. Every family that appears in the Review Hub has at least 2 members.

**DD-14: The suggested keeper is the file with the latest mtime that is not in a backup directory.**
Heuristic: the most recently modified file is the most current. Files in paths containing `backup`, `archive`, `old`, `bak`, `.trash` are deprioritized for keeper selection regardless of mtime.

**DD-15: The duplicate gate caches results across runs.**
After first ingestion, each file's (path, mtime, size, inode) tuple is stored in Curator's local SQLite state database alongside its SHA-256 and SimHash. On subsequent runs, files with unchanged (path, mtime, size, inode) tuples skip re-hashing. This reduces incremental scan time from ~100 seconds to < 5 seconds for a stable ~/Downloads.

---

## 10. Open Questions

**OQ-01: SimHash threshold calibration for personal files.**
The threshold of Hamming ≤ 5 is reasoned from first principles but not empirically validated on a representative personal file corpus. Curator should collect opt-in anonymized false-positive/false-negative reports to tune this threshold. A user-adjustable slider in settings is a reasonable interim solution.

**OQ-02: RETSim archival and replacement.**
The `unisim` package was archived in May 2026. Google has not announced a replacement. As of the writing of this document, the archived package remains installable and functional. If it becomes unavailable, the fallback is: (a) a fine-tuned sentence-transformer model (`all-MiniLM-L6-v2` with cosine similarity), or (b) MinHash + Jaccard for the version confirmation step. This should be monitored quarterly.

**OQ-03: Version detection accuracy for informal naming conventions.**
The regex patterns in §4 cover explicit versioning conventions. Real-world personal files often use idiosyncratic names ("thesis_REAL_FINAL_OMG.docx"). The temporal proximity heuristic partially covers this, but accuracy for highly informal naming is unknown. A user feedback loop ("was this correctly identified as a version?") is needed to improve detection over time.

**OQ-04: Handling archives (ZIP, RAR, tar.gz) for exact duplicates.**
Archive files that are exact duplicates by SHA-256 are handled correctly. Archives with the same content but different compression levels or metadata (creation date in ZIP header) will have different SHA-256 values and will not be caught by the exact gate. Content-level archive deduplication (extract + compare contents) is expensive and out of scope for v1 — mark as future work.

**OQ-05: Cross-file-type near-duplicate detection.**
A screenshot of a PDF and the PDF itself are semantically the same document but will not be caught by either the text near-dup gate (different types) or the image gate (no text). Multi-modal embedding (CLIP-style) could detect these pairs but is expensive. Mark as future work.

**OQ-06: The "hyperfinal" problem.**
Files named `thesis_final_FINAL_FINAL2.docx` represent a genuine user frustration pattern. The current pipeline correctly detects these as version family members, but role assignment is ambiguous (all match `final` pattern). A special `hyperfinal` role could be surfaced to the user with a gentle nudge ("You have 4 'final' versions — which one is truly final?").

**OQ-07: Performance on network drives / NAS.**
The pipeline assumes SSD I/O speeds (~2 GB/s). On network drives (NFS, SMB), partial hash I/O speed may drop to 50–200 MB/s, increasing Stage 3 time from 0.5 to 5–10 seconds. The caching strategy (DD-15) mitigates this on subsequent runs but not on first ingestion. Curator should detect network drives via `os.stat().st_dev` and show a warning.

**OQ-08: Privacy of RETSim / unisim inference.**
RETSim runs locally (the model weights are bundled with the `unisim` package). No file content leaves the device. This is a hard requirement for Curator. If the fallback sentence-transformer is used (OQ-02), it must also run locally via `sentence-transformers` with a cached model — no API calls.

---

## References

- Manku, G.S., Jain, A., Das Sarma, A. (2007). "Detecting Near-Duplicates for Web Crawling." WWW 2007. https://dl.acm.org/doi/10.1145/1242572.1242592
- Charikar, M. (2002). "Similarity Estimation Techniques from Rounding Algorithms." STOC 2002.
- Broder, A. et al. (1998). "Min-wise independent permutations." STOC 1998. https://dl.acm.org/doi/10.1145/276698.276781
- Shrivastava, A., Li, P. (2014). "In Defense of MinHash Over SimHash." arXiv:1407.4416. https://arxiv.org/abs/1407.4416
- Zhang, M. et al. (2024). "RETSim: Resilient and Efficient Text Similarity." ICLR 2024. arXiv:2311.17264. https://arxiv.org/abs/2311.17264
- Shi, E. et al. (2024). "Revisiting Code Similarity Evaluation with Abstract Syntax Tree Edit Distance." arXiv:2404.08817. https://arxiv.org/abs/2404.08817
- MDPI Electronics (2025). "Comparative Evaluation of Perceptual Hashing and Deep Embedding Methods for Robust and Efficient Image Deduplication." Vol. 15(7):1493. https://www.mdpi.com/2079-9292/15/7/1493
- Kolaczkowski, P. fclones: Efficient Duplicate File Finder. https://github.com/pkolaczk/fclones
- idealo. imagededup: Finding duplicate images made easy. https://github.com/idealo/imagededup
- ekzhu. datasketch: MinHash, LSH, HyperLogLog. https://github.com/ekzhu/datasketch
- google/unisim (archived). RETSim implementation. https://github.com/google/unisim

---

*End of R2 — Duplicate & Version Detection Research*
*Next: R3 — File Role and Intent Classification*
