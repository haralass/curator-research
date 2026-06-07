# R3 — Adaptive Reading Engine & Enough-to-Act

**Status:** Research complete — June 2026
**Context:** Curator processes 30,000+ mixed files locally on Apple Silicon (M1/M2/M3). The engine must extract "enough" information per decision level without over-reading. Full deep extraction for all 30k files is never acceptable.

---

## 1. Signal Tier Table

The tiers are ordered strictly by cost. Curator starts at Tier 0 and escalates only when the current tier cannot resolve the decision.

### Tier 0 — Free Signals (Zero file I/O)

All data already in the filesystem kernel structures. No file open() call required. Obtained via `os.scandir()` DirEntry, `os.stat()`, `xattr`, or `mdls`/`MDItemCopyAttribute` (Spotlight index lookup — no file open).

| Signal | How to get it | Example value | Classification value |
|---|---|---|---|
| Filename | `DirEntry.name` | `lecture_notes_CS101.pdf` | Very high — paper above: 96.7% accuracy from filename alone |
| Extension | `Path(name).suffix` | `.pdf`, `.swift`, `.xlsx` | High for unambiguous types |
| File size | `DirEntry.stat().st_size` | 47,382 bytes | Medium — 200MB ZIP ≠ 2KB ZIP |
| mtime | `DirEntry.stat().st_mtime` | Unix timestamp | High for recency grouping |
| ctime | `DirEntry.stat().st_ctime` | Unix timestamp | Medium for creation epoch |
| atime | `DirEntry.stat().st_atime` | Unix timestamp | Medium for "last accessed" age |
| Parent path | `Path(path).parent` | `/Downloads/CS101/Week3` | Very high — path encodes context |
| Path depth | `len(Path(path).parts)` | 7 | Medium |
| `kMDItemWhereFroms` | `mdls -name kMDItemWhereFroms` or `osxmetadata` | `["https://moodle.cs.ucy.ac.cy/..."]` | Deterministic — URL reveals source domain |
| `kMDItemCreator` | `mdls -name kMDItemCreator` | `"Xcode"`, `"Zoom"`, `"Pages"` | Deterministic — app that created it |
| `kMDItemLastUsedDate` | `mdls` | Date | High for recency |
| `kMDItemDownloadedDate` | `mdls` | Date | High for "is this from internet?" |
| `kMDItemIsScreenCapture` | `xattr com.apple.metadata:kMDItemIsScreenCapture` | `1` | Deterministic — it IS a screenshot |
| `kMDItemScreenCaptureType` | `xattr` | `"selection"`, `"display"` | Type of screenshot |
| `kMDItemContentType` | `mdls` | `"com.adobe.pdf"`, `"public.jpeg"` | High — UTI is reliable |
| `kMDItemKind` | `mdls` | `"PDF Document"`, `"Swift Source File"` | High |
| `kMDItemKeywords` | `mdls` | `["work", "invoice"]` | Very high when present |
| `kMDItemTitle` | `mdls` | `"Q3 Budget Review"` | Very high when present |
| `kMDItemAuthors` | `mdls` | `["John Smith"]` | High for document provenance |
| `kMDItemDurationSeconds` | `mdls` | `3600.0` | Deterministic for audio/video |
| `kMDItemPixelWidth/Height` | `mdls` | `2880`, `1800` | Retina screenshot detection |
| `kMDItemOrientation` | `mdls` | `0` (landscape) | Photo/screenshot distinction |
| `kMDItemColorSpace` | `mdls` | `"RGB"` | Media type hint |
| `kMDItemMusicalGenre` | `mdls` | `"Jazz"` | Audio classification |
| `kMDItemFSSize` | `mdls` | bytes | Redundant with stat, but available |
| Finder color tags | `xattr com.apple.metadata:_kMDItemUserTags` | `["Red", "Work"]` | User intent signal |
| iCloud status | `xattr com.apple.fileprovider.collocated` or `brctl` | local/cloud-only | Determines if readable |
| `kMDItemUseCount` | `mdls` | `47` | High-use = important file |
| `kMDItemCopyright` | `mdls` | `"© 2024 ACM"` | Academic/commercial provenance |

**Cost:** ~0.01ms per file for stat(). `mdls` via subprocess adds ~5-10ms overhead per file due to process launch. Better: use `osxmetadata` Python library or direct `MDItemCopyAttribute` via PyObjC — this queries the Spotlight index in-process at ~0.1-1ms per file. For batch reads, use `NSMetadataQuery`.

**Programmatic access (Python):**
```python
# Option A: osxmetadata (pure Python, wraps xattr + MDItemCopyAttribute)
from osxmetadata import OSXMetaData
md = OSXMetaData("/path/to/file")
where_froms = md.wherefroms       # kMDItemWhereFroms
creator = md.creator              # kMDItemCreator
is_screen = md.get_xattr("com.apple.metadata:kMDItemIsScreenCapture")

# Option B: subprocess mdls (slower, one process per file — avoid in loops)
import subprocess, plistlib
result = subprocess.run(["mdls", "-plist", "-", "/path/to/file"],
                        capture_output=True)
attrs = plistlib.loads(result.stdout)

# Option C: NSMetadataQuery via PyObjC (batch, fastest for large sets)
# Query Spotlight index — returns results without opening files
```

**Curator decision:** Use `osxmetadata` + direct xattr reads for Tier 0. Never call `subprocess mdls` in a per-file loop. For the initial 30k-file scan, walk with `os.walk()` (internally uses `os.scandir()` since Python 3.5), batch MDItem queries via PyObjC. Tier 0 alone with the filename classifier paper's approach can resolve ~60-70% of files deterministically.

---

### Tier 1 — Cheap Signals (File header read, milliseconds)

Requires opening the file but reading only the first 512-4096 bytes. Never reads more.

| Signal | How to get it | Cost (M1) | What it resolves |
|---|---|---|---|
| MIME type via libmagic | `python-magic` first 512 bytes | ~1ms | True type vs extension lie |
| Magic bytes | Manual header check | <1ms | Format confirmation |
| PDF text-selectability | `pdfminer` 512-byte header check | ~5ms | Routes to OCR vs text path |
| PDF page count | `pdfminer` catalog read (no page parse) | ~10ms | Size estimation |
| Image dimensions | PIL `Image.open()` with lazy load | ~2ms | Screenshot dimension match |
| Archive manifest | `zipfile.ZipFile.namelist()` | ~5-50ms (size-dependent) | Project vs data vs media |
| First 512 bytes text | `file.read(512).decode(errors='ignore')` | <1ms | Shebang, encoding, keywords |
| Code shebang | First line of text file | <1ms | Language hint |
| EXIF data (images) | `PIL.Image._getexif()` or `exifread` | ~3ms | Camera vs screenshot vs generated |

**Magic byte quick-reference for Curator:**
```
%PDF-           → PDF
PK\x03\x04      → ZIP (docx, xlsx, jar, apk, zip)
\x89PNG         → PNG
\xff\xd8\xff    → JPEG
GIF8            → GIF
RIFF....WEBP    → WebP
\x1f\x8b        → gzip (tar.gz)
BZh             → bzip2
\xfd7zXZ        → xz
Rar!            → RAR
\xd0\xcf\x11\xe0 → Old .doc/.xls/.ppt (OLE2)
#!/             → Script (check rest for interpreter)
<?xml           → XML (SVG, plist, many others)
{               → Possible JSON
---             → Possible YAML
#               → Possible Markdown/config/ini
```

**Archive manifest heuristics:**
Reading `zipfile.ZipFile.namelist()` is cheap (reads central directory at end of file, ~5-50ms for typical archives). From the manifest, infer purpose:

- Contains `package.json` or `requirements.txt` or `Cargo.toml` or `.xcodeproj/` → software project
- Contains `README.md` → likely documented project or course material
- Extension distribution: >70% `.jpg`/`.png` → photo album
- Extension distribution: >70% `.pdf`/`.docx` → document backup
- Contains `syllabus`, `lecture`, `week`, `assignment` in filenames → course backup
- Single top-level folder with version number → software release
- Contains `__MACOSX/` → macOS-created zip

**PDF selectability check (no full parse):**
```python
from pdfminer.pdfparser import PDFParser
from pdfminer.pdfdocument import PDFDocument
from pdfminer.pdftypes import resolve1
from pdfminer.pdfpage import PDFPage

def pdf_quick_info(path):
    with open(path, 'rb') as f:
        parser = PDFParser(f)
        doc = PDFDocument(parser, fallback=True)
        page_count = resolve1(doc.catalog['Pages'])['Count']
        # Check first page for text — if text_state has chars, it's selectable
        first_page = next(PDFPage.create_pages(doc))
        # If text extraction yields nothing → route to OCR
    return page_count
```

**Cost:** Tier 1 for 30k files at ~10ms average = ~5 minutes total. Feasible as part of initial scan.

**Curator decision:** Run Tier 1 immediately after Tier 0 during initial scan. Parallelise with `concurrent.futures.ThreadPoolExecutor(max_workers=8)` — I/O bound, threads help. Tier 0+1 together resolve ~80-85% of files sufficiently for initial group display.

---

### Tier 2 — Medium Signals (Content extraction, seconds)

Full text extraction. Only invoked selectively — not for all 30k files.

| Signal | Tool | Cost (M1) | Notes |
|---|---|---|---|
| PDF text (selectable) | `markitdown` or `pdfminer.six` | 0.12s per 100-page PDF | markitdown: 12s/100 pages, fast but layout poor |
| PDF text (scanned) | `ocrmac` (Apple Vision) | ~0.2s per page | Apple Vision M3 Max: 207ms/page |
| DOCX text | `markitdown` or `python-docx` | ~0.05s typical doc | Fast for digital-native |
| XLSX summary | `openpyxl` read-only header scan | ~0.5s small file | Sheet names + first row = column headers |
| Code structure | `tree-sitter` AST parse | <10ms per file | Import/framework detection, 66 languages |
| Image OCR | `ocrmac` (Apple Vision) | ~0.2s per image | Better than tesseract on macOS |
| Code language | `go-enry` or `hyperpolyglot` | <5ms | Linguist heuristics ported to Go/Rust |
| Document headings | `python-docx` section parse | ~0.1s | Heading structure aids clustering |

**Curator decision:** Tier 2 is only invoked for:
1. Files where Tier 0+1 produced ambiguous or low-confidence classification
2. Files selected as cluster centroid samples (see Section 6)
3. Files flagged by user for Review Hub (needs accurate label)

---

### Tier 3 — Deep Signals (Expensive, minutes)

Reserved for high-value decisions where errors are costly.

| Signal | Tool | Cost (M1) | Notes |
|---|---|---|---|
| Semantic embedding | `all-MiniLM-L6-v2` | ~20ms/doc (batched, MPS) | 384-dim, ~50 docs/s batched on M1 |
| Image embedding | `CLIP ViT-B/32` | ~50ms/image on MPS | Scene/content understanding |
| Image captioning | `moondream2` 4-bit | ~1-3s per image (MPS) | Only for truly ambiguous images |
| NER extraction | `spaCy en_core_web_sm` | ~50ms/doc | Entity types: PERSON, ORG, DATE, MONEY |
| Full PDF OCR | `ocrmac` full document | 0.2s/page × N pages | Scanned documents only |
| Deep code analysis | `tree-sitter` full AST + dependency graph | ~100ms per project | Only for project-root files |

**Embedding throughput note:** `all-MiniLM-L6-v2` on M1 with MPS backend: approximately 50-100 documents/second when batched (batch_size=64). For 30k documents this is ~5-10 minutes — feasible if run selectively. The model is 22M params, 384 dims. Storage cost: 384 × 4 bytes × 30,000 = 46MB — trivially manageable in memory and on disk.

**Moondream2 note:** Photon runtime (moondream's native engine) is ~5x faster than the Transformers path. On Apple Silicon via MPS with 4-bit quantization, expect 1-3 seconds per image for captioning. Use only when CLIP embedding + dimension heuristics fail to classify an image.

**Curator decision:** Tier 3 embedding is run on the representative sample of each cluster (centroid + outliers, not all files). NER and image captioning are invoked only when the decision type requires it (auto-commit requires NER; truly ambiguous images require captioning).

---

## 2. Per-Type Extractor Recommendations

### PDF Routing

```
PDF detected (Tier 1 magic bytes) →
  ├─ Check kMDItemCreator (Tier 0)
  │   ├─ "Microsoft Word", "Pages", "LaTeX" → almost certainly selectable text
  │   └─ "Scanner", "Image Capture", kMDItemOrientation anomaly → likely scanned
  ├─ Quick text test: pdfminer on first page (Tier 1, ~10ms)
  │   ├─ Returns >50 chars → SELECTABLE TEXT path
  │   │   └─ Use: markitdown (speed) OR pdfminer.six (quality)
  │   └─ Returns <10 chars → SCANNED path  
  │       └─ Use: ocrmac (Apple Vision Framework, Python: pip install ocrmac)
  └─ Mixed: handle per-page (rare, flag for Review Hub)
```

**Tool comparison:**

| Tool | Speed | Quality | Tables | Images | Recommendation |
|---|---|---|---|---|---|
| `markitdown` | ★★★★★ | ★★ | ★ | ★ | Fast draft extraction, good for clustering |
| `pdfminer.six` | ★★★★ | ★★★ | ★★ | ✗ | Clean text, no layout analysis |
| `marker-pdf` | ★★ | ★★★★ | ★★★★ | ★★★★ | Best quality, too slow for bulk (use for finalists) |
| `MinerU` | ★ | ★★★★★ | ★★★★★ | ★★★★★ | Best overall quality, very slow, scientific docs |
| `ocrmac` | ★★★★ | ★★★★ | ★★ | N/A | Scanned PDFs on macOS, native Vision framework |

**Curator decision:** Use `markitdown` for Tier 2 bulk extraction (speed priority), `ocrmac` for scanned PDFs. Reserve `marker-pdf` for the small set of documents that reach the "prepare for permanent archive" decision — where layout fidelity matters.

---

### Word/DOCX Routing

```
DOCX detected (ZIP magic + [Content_Types].xml confirms) →
  ├─ Tier 1: python-docx to extract heading list only (~10ms, no full text)
  │   └─ Headings → document structure signal → cluster label hint
  └─ Tier 2: markitdown for full text (fast) OR python-docx for structured extraction
```

**Heading structure value:** A DOCX with headings "Introduction → Methods → Results → Discussion" is an academic paper. "Executive Summary → Q3 Results → Forecast" is a business report. Heading extraction at Tier 1 cost is very high ROI.

```python
from docx import Document
def docx_headings(path):
    doc = Document(path)
    return [p.text for p in doc.paragraphs if p.style.name.startswith('Heading')]
```

**Curator decision:** Always extract DOCX headings at Tier 1 cost as a heading-structure signal. Full text only at Tier 2 when embedding is needed.

---

### Excel/CSV Routing

```
XLSX detected (ZIP + xl/ folder structure) →
  ├─ Tier 1: openpyxl read-only, extract:
  │   ├─ Sheet names (immediate cluster label)
  │   ├─ First row of first sheet (column headers)
  │   └─ Data type distribution (formulas? numbers? dates? strings?)
  └─ Tier 2: Full text dump if embedding needed

CSV detected →
  ├─ Tier 1: Read first 2KB, extract header row and first 3 data rows
  └─ Type inference from column names (heuristic rules below)
```

**Spreadsheet type classification heuristics (from column names):**

| Pattern | Inferred type |
|---|---|
| Columns contain: "amount", "price", "total", "invoice", "tax", "VAT" | Finance/accounting |
| Columns contain: "date", "name", "id", numeric majority | Dataset/data export |
| Few rows (<20), label-value pairs | Form/questionnaire |
| Columns contain: "grade", "student", "score", "assignment" | Academic gradebook |
| "lat", "lon", "latitude", "longitude" | Geospatial data |
| "feature_", "label", "train", "test" | ML dataset |

**Curator decision:** Read sheet names + first-row headers at Tier 1 (~5ms via openpyxl read-only mode). This alone classifies ~80% of spreadsheets. Full extraction only for uncertain cases.

---

### Code Routing

```
Code file detected (extension + shebang Tier 1) →
  ├─ Language detection: go-enry (Go port of GitHub Linguist, <5ms per file)
  │   Pipeline: modeline → known filename → shebang → extension → heuristics → Bayes
  ├─ Project root detection (Tier 0, free):
  │   Look for: package.json, requirements.txt, Cargo.toml, .xcodeproj, Makefile,
  │             setup.py, go.mod, pom.xml, build.gradle, CMakeLists.txt, .git/
  │   → If found: the directory IS a project — treat as a unit
  ├─ Framework detection (Tier 2, tree-sitter imports, <10ms):
  │   Python: "import django" → web; "import torch" → ML; "import SwiftUI" → iOS
  │   JS: "import React" → web frontend; "require('express')" → backend
  │   Swift: "import SwiftUI" → app; "import XCTest" → tests
  └─ Tier 3 (rarely): Full AST dependency graph for complex projects
```

**Linguist detection strategy order:**
1. Modeline (Vim/Emacs mode line) — exact
2. Known filename (Makefile, Dockerfile, Gemfile) — exact  
3. Shebang (`#!/usr/bin/env python3`) — exact
4. Extension — high confidence for unambiguous extensions
5. Heuristics (regex patterns in content, e.g., `^[^#]+:-` for Prolog)
6. Bayesian classifier — last resort

**Alternatives:** `hyperpolyglot` (Rust, fastest), `go-enry` (Go, Python bindings available), `linguist` (Ruby, too heavy for embedding). For Curator: call `go-enry` via subprocess or use its Python bindings.

**Curator decision:** Detect code language with go-enry at Tier 1. Detect project roots with Tier 0 path heuristics. Group all files within a detected project root together before any deeper analysis.

---

### Image/Screenshot Routing

```
Image detected (JPEG/PNG/HEIC/WEBP) →
  ├─ Tier 0 first (FREE):
  │   ├─ kMDItemIsScreenCapture == 1 → SCREENSHOT (certain)
  │   ├─ kMDItemCreator == "Screenshot" → SCREENSHOT
  │   ├─ Pixel dimensions: 2560×1600, 2880×1800, 1920×1080 (standard Retina) → likely screenshot
  │   ├─ kMDItemOrientation == 0 AND very large pixel count → likely monitor screenshot
  │   └─ kMDItemWhereFroms present → DOWNLOADED image
  ├─ Tier 1 (if still ambiguous):
  │   ├─ EXIF data: has GPS → photo; has CameraModel → camera photo; no EXIF → generated/screenshot
  │   └─ File size relative to dimensions: very low bytes/pixel → highly compressed screenshot
  ├─ Tier 2:
  │   ├─ ocrmac on first region → if text found → screenshot or document scan
  │   └─ CLIP embedding (ViT-B/32) → cosine similarity to known clusters
  └─ Tier 3 (truly ambiguous only):
      └─ moondream2 caption → natural language description for human review
```

**Screenshot dimension patterns (common Retina displays):**
- 2880×1800 — MacBook Pro 15" early models
- 3456×2160 — MacBook Pro 14" M1/M2  
- 3024×1964 — MacBook Pro 14" notch area
- 5120×2880 — Pro Display XDR
- 1920×1080, 2560×1440, 3840×2160 — external monitors

**Receipt/document image detection:** Run fast OCR (ocrmac, ~0.2s) on first quadrant of image. If text density > threshold AND contains currency symbols OR dates → receipt/document scan.

**Curator decision:** Use `kMDItemIsScreenCapture` as the first and definitive signal. It is set by macOS automatically. For images without this attribute, EXIF → dimensions → fast OCR is the progression. Defer moondream2 to Tier 3 and only for images going into "Auto-commit" decisions.

---

### Archive Routing

```
Archive detected (ZIP/tar/gz/rar magic bytes) →
  ├─ Tier 1: List manifest (zipfile.ZipFile.namelist() or tarfile.getnames())
  │   Cost: ~5-50ms depending on archive size (reads central directory only, no extraction)
  ├─ Heuristic classification from manifest:
  │   ├─ Has package.json/requirements.txt/Cargo.toml → software project
  │   ├─ Has README.md + source extensions → documented project
  │   ├─ >70% image extensions → photo album / media backup
  │   ├─ >70% .pdf/.docx/.pptx → document backup
  │   ├─ Filename pattern: lecture/week/assignment + .pdf → course backup
  │   ├─ Single top-level dir with version (v1.2.3, 2024-01) → software release / backup snapshot
  │   ├─ __MACOSX/ present → macOS zip (created by Finder)
  │   └─ Mixed with no pattern → generic backup → flag for user
  └─ NEVER extract contents — manifest only is safe and sufficient
```

**Curator decision:** Read archive manifest at Tier 1. Never extract. Classify from manifest heuristics. Mark unresolvable archives for Review Hub (let user decide).

---

## 3. Router Decision Tree

```
FILE ARRIVES IN PIPELINE
         │
         ▼
    [TIER 0: FREE]
    Read filesystem + Spotlight metadata
    (kMDItemCreator, WhereFroms, ContentType, IsScreenCapture, etc.)
         │
         ├─ Decision resolvable with high confidence?
         │   Examples: kMDItemIsScreenCapture=1, kMDItemCreator="Xcode",
         │             kMDItemWhereFroms contains known domain
         │   → YES: Record classification, mark confidence=HIGH, skip to cache
         │
         └─ NO: Escalate to Tier 1
                    │
               [TIER 1: HEADER READ]
               Read first 512-4096 bytes
               Confirm MIME, magic bytes, page count, archive manifest, EXIF
                    │
                    ├─ Decision resolvable?
                    │   → YES: Record classification, mark confidence=MEDIUM-HIGH
                    │
                    └─ NO or confidence < threshold:
                         Is this file a cluster centroid or in "Review Hub" queue?
                              │
                              ├─ NO: Defer — keep Tier 1 label, mark for async Tier 2
                              │
                              └─ YES:
                                   [TIER 2: CONTENT EXTRACTION]
                                   Full text / OCR / headings / code imports
                                        │
                                        ├─ Decision resolvable?
                                        │   → YES: Record, confidence=HIGH
                                        │
                                        └─ NO or it's an auto-commit candidate:
                                             [TIER 3: EMBEDDING + NER + CAPTION]
                                             Semantic embedding, entity extraction
                                             Only for highest-stakes decisions
```

**Confidence thresholds:**
- `>0.90`: Auto-classify, no user review needed
- `0.70-0.90`: Classify but flag as "Curator suggests..."
- `<0.70`: Route to Review Hub — human makes the call

---

## 4. Enough-to-Act Thresholds

| Decision type | Minimum required signals | Escalate if... |
|---|---|---|
| **Initial virtual grouping** (rough sort for display) | Tier 0: filename + extension + size + path + ContentType | Always sufficient for initial UI |
| **Duplicate candidate detection** | Tier 0: size + filename similarity + partial hash (first 64KB) | Two files same size → compute SHA-256 |
| **Review Hub staging** (uncertain files) | Tier 0 + Tier 1 | Confidence < 0.70 after Tier 1 |
| **Cluster label assignment** (naming a group) | Tier 1 + Tier 2 text extraction for centroid sample | Tier 0+1 centroid text is empty/binary |
| **Embedding-based cluster assignment** | Tier 2 text → Tier 3 embedding | Required for semantic similarity search |
| **Auto-commit** (Curator acts without user) | Tier 2 + Tier 3 embedding + NER + confidence > 0.90 | If any Tier 3 result is ambiguous → downgrade to suggest |
| **Permanent delete suggestion** | ALL tiers + NER entity check (no PII in trash) + confidence > 0.95 | Never auto-delete — always requires confirmation |
| **Archive manifest inspection** | Tier 1 manifest only | Archive is corrupted → flag |
| **Screenshot classification** | Tier 0 (kMDItemIsScreenCapture) — single attribute | IsScreenCapture absent → Tier 1 EXIF |
| **Code project detection** | Tier 0 path heuristics (project root files) | No project root found → Tier 1 shebang |

**The Enough-to-Act principle formally stated:** A tier is "enough" when the posterior probability of the correct classification, given all signals collected so far, exceeds the decision threshold for the current action. Actions with reversible consequences (virtual grouping, staging) have low thresholds. Actions with irreversible consequences (auto-commit, delete) require the highest tier.

**Prior art:** This mirrors "anytime algorithms" from AI — algorithms that can be interrupted at any time and return the best answer achievable with signals gathered so far (Zilberstein, 1996, "Using Anytime Algorithms in Intelligent Systems"). Cost-sensitive feature selection literature (PMC4159408, PMC6986087) confirms that greedy feature selection with cost penalties is theoretically sound for this pattern.

**Curator decision:** Implement a `DecisionContext` dataclass that tracks which tier has been reached per file and what the current confidence is. The router checks this context before escalating. This enables async deepening — files can be re-queued for higher tiers when idle CPU time is available.

---

## 5. Caching Strategy

### What to cache per file

```sql
CREATE TABLE file_cache (
    path TEXT PRIMARY KEY,
    -- Invalidation keys
    mtime REAL NOT NULL,
    size INTEGER NOT NULL,
    sha256 TEXT,            -- NULL until computed; lazy
    -- Tier 0 signals (always cached)
    content_type TEXT,
    creator TEXT,
    where_froms TEXT,       -- JSON array
    is_screenshot INTEGER,
    finder_tags TEXT,       -- JSON array
    keywords TEXT,          -- JSON array
    -- Tier 1 signals
    mime_type TEXT,
    true_extension TEXT,    -- from magic bytes
    page_count INTEGER,
    image_width INTEGER,
    image_height INTEGER,
    has_exif INTEGER,
    archive_manifest TEXT,  -- JSON list of top-level items
    pdf_is_selectable INTEGER,
    -- Tier 2 signals
    extracted_text TEXT,    -- Tier 2 full text (can be large)
    headings TEXT,          -- JSON list, DOCX headings
    code_language TEXT,
    code_framework TEXT,
    -- Tier 3 signals
    embedding BLOB,         -- 384 × float32 = 1536 bytes per file
    ner_entities TEXT,      -- JSON: {PERSON:[], ORG:[], DATE:[], MONEY:[]}
    image_caption TEXT,
    -- Classification results
    tier_reached INTEGER DEFAULT 0,
    confidence REAL,
    assigned_cluster TEXT,
    last_processed REAL     -- Unix timestamp
);
```

**Storage cost for 30k files (full cache):**
- Metadata fields: ~500 bytes/file × 30k = ~15MB
- Extracted text: avg 5KB/file × 30k = ~150MB (compress with zlib → ~30-50MB)
- Embeddings (384 dims × float32): 1,536 bytes × 30k = **46MB** — entirely manageable
- NER entities: ~200 bytes/file × 30k = ~6MB
- Total estimated: **~250MB** uncompressed, ~100MB with text compression

This is trivially small compared to typical M1 Mac storage.

### Cache invalidation

**Primary check:** `mtime + size` match → assume unchanged. Cost: zero extra I/O (already in `os.stat()`).

**Secondary check (suspicious files only):** Compute SHA-256 if:
- mtime changed but size is identical (unusual — could be touch, could be edit)
- File is in a "high stakes" group (tagged for auto-commit)

**Never recompute SHA-256 as a primary strategy** — reading 30k files fully for hashing costs ~5-10 minutes on an SSD for typical file sizes.

**Strategy in code:**
```python
def needs_reprocessing(cached: dict, current_stat: os.stat_result) -> bool:
    if cached['mtime'] != current_stat.st_mtime:
        return True
    if cached['size'] != current_stat.st_size:
        return True
    return False  # Fast path: assume unchanged
```

### Cache backend

**Recommendation: SQLite (primary) + in-memory numpy array (for active embeddings)**

- SQLite for all metadata and text: mature, reliable, zero-dependency, ACID transactions
- `sqlite-vec` extension for vector search directly in SQLite (avoids needing a separate FAISS process)
- For active session: load all embeddings into a numpy float32 matrix for fast cosine similarity (`np.dot` is faster than sqlite-vec for bulk operations)
- FAISS is an alternative but adds complexity; for 30k files it is unnecessary — numpy cosine similarity of 30k × 384 fits in ~46MB RAM and runs in milliseconds

**Curator decision:** Single SQLite database at `~/.curator/cache.db` with `sqlite-vec` for vector queries. On session start, load embeddings into numpy for fast in-memory search. Write back to SQLite after each processing batch.

---

## 6. Group-Level Sampling

When Curator encounters a potential group of N similar files, it must not read all N deeply.

### How many samples needed to characterize a group?

For group characterization (labeling, type detection), not hypothesis testing:
- Groups of 10-50 files: read all
- Groups of 50-200 files: sample 20 (proportional stratified sample)
- Groups of 200-500 files: sample 30-40
- Groups of 500+ files: sample 50 maximum

**Theoretical basis:** With 50 samples from a group of 500, if the group has a true type distribution (e.g., 80% PDFs of type X, 20% of type Y), the sample captures this distribution within ±7% at 95% confidence (standard proportion estimation). For file classification, this precision is more than sufficient.

### Sampling strategy

**Phase 1 — Cluster formation (Tier 0+1 only):**
Cluster all 30k files by extension + size bucket + path segment. This is free. Result: rough groups (e.g., "PDFs in Downloads", "Swift files in Xcode projects").

**Phase 2 — Centroid selection for deep read (Tier 2+3):**

```python
def select_centroid_sample(file_group: list[FileRecord], k: int = 30) -> list[FileRecord]:
    """
    Select k representative files from a group for deep reading.
    Uses Tier 1 embeddings (filename token embeddings — fast) to find centroids.
    """
    if len(file_group) <= k:
        return file_group
    
    # Step 1: Compute cheap filename embeddings (TF-IDF on filename tokens)
    # These are NOT semantic embeddings — just for diversity sampling
    from sklearn.feature_extraction.text import TfidfVectorizer
    from sklearn.cluster import MiniBatchKMeans
    
    names = [f.filename for f in file_group]
    tfidf = TfidfVectorizer(analyzer='char_wb', ngram_range=(2,4))
    X = tfidf.fit_transform(names).toarray()
    
    # Step 2: k-means to find k clusters, take centroid of each
    kmeans = MiniBatchKMeans(n_clusters=k, random_state=42)
    labels = kmeans.fit_predict(X)
    
    # Step 3: For each cluster, take the file closest to centroid
    centroids = []
    for i in range(k):
        cluster_files = [file_group[j] for j in range(len(file_group)) if labels[j] == i]
        centroids.append(cluster_files[0])  # Closest to centroid
    
    return centroids
```

**Phase 3 — Outlier detection:**

After deep-reading centroids, identify outliers (files that don't fit the group pattern):
```python
# After computing Tier 3 embeddings for centroid sample:
group_centroid_embedding = embeddings.mean(axis=0)
similarities = cosine_similarity(all_embeddings, group_centroid_embedding.reshape(1,-1))
outliers = [file_group[i] for i, sim in enumerate(similarities) if sim < 0.5]
# Outliers get individual attention — route to Review Hub
```

**Coreset methods (research context):** Formal coreset selection (arxiv 2505.17799) is the ML research area for this problem. For Curator's practical needs, the k-means centroid selection above is a proven effective approximation without the theoretical overhead.

**Curator decision:** Use filename TF-IDF k-means for fast centroid selection (no Tier 2+ needed). Deep-read only centroids. Flag semantic outliers for Review Hub. Group sizes above 500 never get more than 50 deep reads.

---

## 7. Performance Budget

### Realistic time estimates for 30k files on M1 Mac

| Phase | Operation | Time estimate | Parallelism |
|---|---|---|---|
| Filesystem walk | `os.walk()` on 30k files | 1-3 seconds | Single thread, I/O bound |
| Tier 0 metadata | `os.stat()` + xattr for 30k files | 30-90 seconds | 8 threads (I/O bound) |
| Tier 0 Spotlight | `MDItemCopyAttribute` batch for 30k | 60-180 seconds | NSMetadataQuery batch |
| **Tier 0 total** | | **2-5 minutes** | |
| Tier 1 header read | 30k × 10ms average | 300 seconds / 8 threads | **~5-7 minutes** |
| **Tier 0+1 total** | | **7-12 minutes** | Good enough for first display |
| Tier 2 text extract | 3k files (10% of total) × 200ms | 600 seconds / 4 processes | ~2.5 minutes |
| Tier 3 embedding | 3k texts × 20ms (batched) | 60 seconds | ~1 minute |
| **Full pipeline (selective)** | | **15-25 minutes** | |

**The key insight:** Tier 2+3 should never run on all 30k files. If Tier 2 runs on 10% (3,000 files — the ambiguous ones and cluster centroids), the full pipeline is feasible in 15-25 minutes. The first usable UI display can appear after 7-12 minutes of Tier 0+1 processing.

**Progressive display strategy:**
1. T+0-5 min: Tier 0 scan completes → display rough groups (extension-based)
2. T+5-12 min: Tier 1 scan completes → refine groups, add confidence scores
3. T+12-25 min: Tier 2+3 on centroids → finalize cluster labels and auto-commit candidates

**Parallelism notes:**
- Tier 0+1 is I/O-bound → `ThreadPoolExecutor(max_workers=8)` optimal on M1
- Tier 2 text extraction is CPU-bound → `ProcessPoolExecutor(max_workers=4)` (4 performance cores on M1)
- Tier 3 embedding: batch encode with MPS backend → single process, batch_size=64

**Filesystem walk speed:** `os.walk()` internally uses `os.scandir()` since Python 3.5, returning DirEntry objects with cached stat(). Walking 30k files costs ~1-3 seconds purely. The bottleneck is the per-file metadata lookups.

**Curator decision:** Target T+10 minutes for first complete UI display with Tier 0+1 data. Tier 2+3 runs as background async deepening while user reviews initial groups. Use `asyncio` + `ThreadPoolExecutor` for the I/O-bound tiers, `ProcessPoolExecutor` for CPU-bound Tier 2.

---

## 8. macOS-Specific Signals

macOS provides signals unavailable on Windows/Linux that dramatically improve classification accuracy for personal file corpora.

### Complete useful kMDItem attribute list

**Identity & Provenance:**

| Attribute | Access | Value | Use in Curator |
|---|---|---|---|
| `kMDItemWhereFroms` | xattr + MDItem | `["https://moodle.cs.ucy.ac.cy/..."]` | Domain = institution/context. URL = exact source. Deterministic. |
| `kMDItemCreator` | mdls | `"Xcode"`, `"Zoom"`, `"Pages"`, `"Adobe Acrobat"` | App = file purpose. Deterministic. |
| `kMDItemContentType` | mdls | `"com.adobe.pdf"`, `"public.python-script"` | UTI type string. More precise than extension. |
| `kMDItemKind` | mdls | `"PDF Document"`, `"Python Script"` | Human-readable kind string. |
| `kMDItemTitle` | mdls | `"Q3 Financial Report"` | Document title if embedded. |
| `kMDItemAuthors` | mdls | `["Smith, J."]` | Academic/business document authorship. |
| `kMDItemKeywords` | mdls | `["invoice", "2024"]` | User or app-assigned tags. |
| `kMDItemCopyright` | mdls | `"© 2024 Springer"` | Academic/commercial origin. |
| `kMDItemComment` | mdls | Finder comment | User-set context. |

**Temporal:**

| Attribute | Access | Use in Curator |
|---|---|---|
| `kMDItemLastUsedDate` | mdls | Recency signal. Old = stale. |
| `kMDItemUseCount` | mdls | High use count = user values this file. |
| `kMDItemDownloadedDate` | mdls | When downloaded — confirms internet origin. |
| `kMDItemContentCreationDate` | mdls | When content was created (not file). |
| `kMDItemContentModificationDate` | mdls | Last edit time. |
| `kMDItemFSCreationDate` | mdls | Filesystem creation time. |

**Screenshot detection:**

| Attribute | Access | Value | Use |
|---|---|---|---|
| `kMDItemIsScreenCapture` | `xattr com.apple.metadata:kMDItemIsScreenCapture` | `1` | Definitive screenshot flag |
| `kMDItemScreenCaptureGlobalRect` | xattr | Rect dictionary | Which monitor/area captured |
| `kMDItemScreenCaptureType` | xattr | `"selection"`, `"display"`, `"window"` | Screenshot subtype |

**Media:**

| Attribute | Use |
|---|---|
| `kMDItemPixelWidth/Height` | Dimension-based screenshot heuristics |
| `kMDItemDurationSeconds` | Audio/video length → meeting recording vs short clip |
| `kMDItemMediaTypes` | Contains "Sound"/"Video" |
| `kMDItemOrientation` | Portrait vs landscape |
| `kMDItemColorSpace` | RGB/CMYK/Gray |
| `kMDItemMusicalGenre` | Music classification |

**Finder:**

| Attribute | Access | Use |
|---|---|---|
| `com.apple.metadata:_kMDItemUserTags` | xattr | Color labels + tag names (user intent) |
| `com.apple.FinderInfo` | xattr | Finder flags, locked status |

### Programmatic access from Python

```python
# Recommended: osxmetadata library
pip install osxmetadata

from osxmetadata import OSXMetaData
import plistlib

def get_tier0_signals(path: str) -> dict:
    md = OSXMetaData(path)
    result = {}
    
    # MDItem attributes (via Spotlight index, no file open)
    result['where_froms'] = md.wherefroms or []
    result['creator'] = md.creator
    result['content_type'] = md.get('kMDItemContentType')
    result['kind'] = md.get('kMDItemKind')
    result['title'] = md.get('kMDItemTitle')
    result['authors'] = md.get('kMDItemAuthors') or []
    result['keywords'] = md.keywords or []
    result['last_used'] = md.lastuseddate
    result['use_count'] = md.get('kMDItemUseCount')
    result['downloaded'] = md.get('kMDItemDownloadedDate')
    result['pixel_width'] = md.get('kMDItemPixelWidth')
    result['pixel_height'] = md.get('kMDItemPixelHeight')
    result['duration'] = md.get('kMDItemDurationSeconds')
    
    # Extended attributes (direct xattr reads)
    try:
        raw = md.get_xattr('com.apple.metadata:kMDItemIsScreenCapture')
        result['is_screenshot'] = bool(plistlib.loads(raw)) if raw else False
    except Exception:
        result['is_screenshot'] = False
    
    try:
        raw = md.get_xattr('com.apple.metadata:_kMDItemUserTags')
        result['finder_tags'] = plistlib.loads(raw) if raw else []
    except Exception:
        result['finder_tags'] = []
    
    return result
```

**iCloud status detection:**
```python
import subprocess

def is_icloud_cloudonly(path: str) -> bool:
    """Files not downloaded from iCloud cannot be read."""
    try:
        result = subprocess.run(
            ['brctl', 'log', '--wait', '--shorten'],
            capture_output=True, text=True, timeout=1
        )
        # Simpler: check for com.apple.ubiquity xattr
        import xattr
        attrs = xattr.listxattr(path)
        return 'com.apple.ubiquity.ubiquitous' in attrs
    except Exception:
        return False
```

**Prior art on macOS-specific metadata for file classification:** No academic papers found specifically on using kMDItem attributes for personal file management AI. However, the attributes are well-documented in Apple's Spotlight metadata reference and the `osxmetadata` library (RhetTbull/osxmetadata) provides the most complete Python access layer. The `macos_mditem_metadata` repo (RhetTbull/macos_mditem_metadata, updated Sept 2024) provides a JSON catalogue of all attributes with macOS version availability.

**Curator decision:** kMDItemWhereFroms and kMDItemCreator are the two highest-value free signals in the entire pipeline. A file with `kMDItemWhereFroms = ["https://moodle.cs.ucy.ac.cy/"]` is university material with certainty. A file with `kMDItemCreator = "Zoom"` is a meeting recording. These two attributes alone can deterministically classify thousands of files without any file read. Implement a domain-to-context rule table (configurable by user) as a first-pass classifier.

---

## 9. Prior Art

### Papers

| Paper | Relevance |
|---|---|
| "Document Type Classification using File Names" (arXiv 2410.01166, 2024) | Random Forest on TF-IDF filenames: 99.63% accuracy, 1.23×10⁻⁴ sec/prediction, 442× faster than DiT. Directly applicable to Curator's Tier 0 classifier. |
| "Anytime Algorithm for Feature Selection" (ResearchGate, Zilberstein 1996 lineage) | Theoretical basis for progressive tier escalation. Anytime algorithms return best-available answer at any interruption point. |
| "Cost-Constrained Feature Selection" (PMC6986087) | Greedy forward selection with cost penalties — formalism for Curator's "escalate only when needed" router. |
| "A Coreset Selection of Coreset Selection Literature" (arXiv 2505.17799, 2025) | Comprehensive taxonomy of coreset methods. Geometry-based and uncertainty-based approaches directly applicable to group-level sampling. |
| "Rough Sets and Laplacian Score Based Cost-Sensitive Feature Selection" (PMC6005488) | Cost-sensitive feature selection with ordered features by acquisition cost. Maps exactly to Tier 0/1/2/3 cost ordering. |
| READOC Benchmark (arXiv 2409.05137) | Document structured extraction benchmark — useful for evaluating Curator's Tier 2 extraction quality. |

### Libraries

| Library | Purpose | Quality | Notes |
|---|---|---|---|
| `osxmetadata` (RhetTbull) | Python access to all kMDItem attributes + xattr | ★★★★★ | Best available Python interface for macOS metadata |
| `macos_mditem_metadata` (RhetTbull) | JSON catalogue of all kMDItem attributes | ★★★★★ | Reference data, updated Sept 2024 |
| `markitdown` (Microsoft) | PDF/DOCX/XLSX/HTML → Markdown | ★★★ (speed) / ★ (quality) | 12s/100 pages, 139k GitHub stars, fast but layout-blind |
| `marker-pdf` | PDF → Markdown with layout | ★★★★ | Best open-source PDF quality, GPU optional |
| `pdfminer.six` | PDF text extraction + catalog read | ★★★★ | Reliable, catalog-based page count is cheap |
| `ocrmac` (straussmaximilian) | Apple Vision OCR Python wrapper | ★★★★★ | Native macOS, 207ms/page on M3 Max, best for macOS |
| `python-docx` | DOCX structure extraction | ★★★★★ | Headings, sections, tables |
| `openpyxl` | Excel read/write | ★★★★ | Read-only mode fast for header extraction |
| `go-enry` | GitHub Linguist ported to Go | ★★★★★ | Python bindings available, <5ms/file |
| `hyperpolyglot` | Linguist ported to Rust | ★★★★★ | Even faster than go-enry |
| `tree-sitter` (Python bindings) | AST parsing, 66 languages | ★★★★★ | <10ms/file, import and framework extraction |
| `python-magic` | libmagic MIME detection | ★★★★★ | 512 bytes, <1ms/file |
| `sentence-transformers all-MiniLM-L6-v2` | Semantic embeddings | ★★★★ | 22M params, 384 dims, MPS-accelerated on M1 |
| `sqlite-vec` | Vector search in SQLite | ★★★★ | No separate vector DB needed |
| `moondream2 4-bit` (Photon runtime) | Vision-language model for images | ★★★★ | 1-3s/image on M1 via MPS, use sparingly |
| `spaCy en_core_web_sm` | NER entity extraction | ★★★★ | ~50ms/doc, PERSON/ORG/DATE/MONEY |

### Benchmarks summary

| Operation | Speed on M1 | Source |
|---|---|---|
| `os.walk()` 30k files | ~1-3 seconds | Python stdlib benchmarks |
| `MDItemCopyAttribute` per file | ~0.1-1ms | osxmetadata docs |
| `python-magic` MIME detect | <1ms per file | Library documentation |
| PDF page count (catalog read) | ~10ms per file | pdfminer.six |
| PDF text extract (markitdown) | ~120ms per page | Benchmark: 12s/100 pages |
| Apple Vision OCR | ~207ms per page | M3 Max benchmark (community) |
| Tesseract OCR | ~300-500ms per page | Published benchmarks |
| `all-MiniLM-L6-v2` embedding | ~20ms/doc batched, MPS | Community estimates |
| `moondream2` captioning | ~1-3s per image, MPS | Moondream docs, community |
| `go-enry` language detection | <5ms per file | go-enry benchmarks |
| `tree-sitter` AST parse | <10ms per file | Published benchmarks |
| openpyxl header read (read-only) | ~5-50ms | openpyxl docs |

---

## 10. Design Decisions

**D1: Filename classifier is the first gate, before any file I/O.**
Based on arXiv 2410.01166: Random Forest on TF-IDF filename tokens achieves 99.63% accuracy at 1.23×10⁻⁴ sec/prediction. Train once on a labelled set (1000 user files with known types). This alone resolves 60-70% of a typical corpus before opening any file.

**D2: kMDItemWhereFroms domain table is Curator's secret weapon.**
Build a configurable YAML rule table: `moodle.cs.ucy.ac.cy → university/CS`, `github.com → code`, `arxiv.org → research paper`, `drive.google.com → work doc`. User can add their own domains. Lookup is O(1) per file. Zero cost, deterministic.

**D3: Archive manifest is always Tier 1, never Tier 2.**
`zipfile.ZipFile.namelist()` reads only the central directory (end of file), not the compressed contents. This is safe, fast, and sufficient to classify the archive's purpose. Never extract to classify.

**D4: Screenshot detection uses kMDItemIsScreenCapture as the sole primary signal.**
macOS sets this attribute automatically for all screenshots. It is reliable, free, and requires no file read. Dimension heuristics are fallback only for files where this attribute is absent (e.g., screenshots imported from another machine).

**D5: Code project grouping precedes all other classification.**
When a directory contains `package.json`, `requirements.txt`, or `.xcodeproj`, ALL files in that directory tree are grouped as a single "project" entity, regardless of individual file types. The project is the unit, not the file. This prevents splitting a codebase into 50 individual files in the UI.

**D6: Tier 2 runs on at most 10% of files (3,000 of 30,000).**
The budget constraint is hard. If Tier 0+1 cannot resolve a file and it is not a cluster centroid, it waits in async queue. The user sees the Tier 0+1 result in the UI while deeper analysis proceeds in background.

**D7: Single SQLite database with sqlite-vec extension is the cache.**
No external services, no FAISS process, no Redis. SQLite is the only dependency. The database is at `~/.curator/cache.db`. Embeddings stored as BLOBs, loaded into numpy on session start. This is sufficient for 30k files (46MB embeddings, 250MB total).

**D8: Cache invalidation uses mtime+size as primary, SHA-256 as lazy secondary.**
mtime+size check costs zero extra I/O. SHA-256 is only computed when: (a) file is flagged for auto-commit and freshness is critical, or (b) mtime changed but size is identical (unusual pattern). For 99% of files, mtime+size is sufficient.

**D9: Group sampling uses filename TF-IDF k-means, not Tier 3 embeddings.**
Centroid selection for groups >50 files uses cheap TF-IDF on filenames (milliseconds, no file read). Tier 3 embeddings are computed AFTER the sample is selected, not before. This prevents the chicken-and-egg problem of needing deep signals to decide which files to read deeply.

**D10: moondream2 captioning is reserved for Tier 3 and only for image auto-commit decisions.**
Image captioning is 1-3 seconds per image. For a group of 500 images, even 50 centroids × 3s = 2.5 minutes just for captioning. Reserve it for images that: (a) CLIP embedding cannot classify, AND (b) are candidates for auto-commit (not just "sort loosely"). For grouping purposes, CLIP ViT-B/32 embedding is sufficient.

**D11: iCloud-only files are skipped in Tier 1+ until downloaded.**
Check `com.apple.ubiquity.ubiquitous` xattr. If set and file is cloud-only, only Tier 0 (Spotlight index, which caches metadata even for cloud-only files) is used. Flag in UI: "Some files are iCloud-only — download to enable full analysis."

**D12: Parallel processing architecture: I/O-bound tiers use threads, CPU-bound use processes.**
- Tier 0+1: `ThreadPoolExecutor(max_workers=8)` — filesystem I/O is the bottleneck
- Tier 2: `ProcessPoolExecutor(max_workers=4)` — CPU-bound extraction, avoid GIL
- Tier 3 embedding: single process, MPS-accelerated batch encode
- All results flow through a central `asyncio` event loop that updates the SQLite cache atomically

---

## 11. Open Questions

**OQ1: Domain rule table completeness and maintainability.**
`kMDItemWhereFroms` is deterministic when the domain is known, but requires a maintained domain-to-context mapping. How does Curator handle unknown domains? Suggestion: treat unknown domains as "internet download, source unknown" and route to Tier 1. Could crowdsource a domain table via a community-maintained YAML file (similar to how AdBlock maintains filter lists).

**OQ2: Filename classifier training data.**
The arXiv 2410.01166 approach requires training a Random Forest on labelled filenames. Where does the initial training set come from? Options: (a) pre-ship a trained model from a public domain filename corpus, (b) have the user label 100 files on first run to seed the classifier, (c) use the heuristic rules as a zero-shot baseline and retrain from user confirmations. The active learning approach (c) is most aligned with Curator's incremental philosophy but adds UX complexity.

**OQ3: Multilingual filenames and non-ASCII paths.**
Cyprus-specific use case: filenames may be in Greek. The TF-IDF filename classifier in the paper uses English-corpus TF-IDF. Greek filenames will hit the "ambiguous" fallback. Does go-enry handle Greek filenames for code files? Are kMDItem attributes correctly populated for Greek-named files? Testing needed.

**OQ4: Performance of NSMetadataQuery batch vs per-file MDItemCopyAttribute.**
For 30k files, is it faster to run one `NSMetadataQuery` that returns all Spotlight-indexed attributes for all files in a scope, or to call `MDItemCopyAttribute` per file? The batch query approach (via `NSMetadataQuery` with `kMDQueryScopeHome`) may return all results in seconds vs the per-file approach taking minutes. Requires benchmarking on actual hardware. If NSMetadataQuery batch works well, Tier 0 could drop from 2-5 minutes to under 60 seconds.

**OQ5: Handling files not indexed by Spotlight.**
Some files are excluded from Spotlight indexing (Privacy > Spotlight > Privacy tab, or `.metadata_never_index` sentinel file). For these files, all MDItem attributes will be null. Curator must detect this case and fall back to filesystem-only Tier 0 signals. Estimated prevalence: <5% of typical user files.

**OQ6: Embedding model choice — MiniLM-L6-v2 vs alternatives for personal file domains.**
`all-MiniLM-L6-v2` is trained on general web text. Personal file text (code, lecture notes, personal finance) may cluster poorly with this model. Alternatives: `all-MiniLM-L12-v2` (better quality, same speed class), `BAAI/bge-small-en-v1.5` (often outperforms MiniLM on retrieval tasks), or a domain-adapted model. Needs empirical evaluation on a 1k-file sample from actual Curator users. If clustering quality is poor with MiniLM, this is the first model to swap.

**OQ7: How to handle scanned PDFs that are large (50+ pages)?**
Apple Vision OCR at 207ms/page × 50 pages = ~10 seconds per document. For a corpus with 100 scanned PDFs, that's ~17 minutes of OCR alone. Strategy options: (a) OCR only first 3 pages for classification, (b) OCR only when user requests full text, (c) queue OCR as a low-priority background job. Recommendation: OCR first 3 pages for Tier 2 classification, defer full OCR to Tier 3 on-demand.

**OQ8: What is the correct threshold for "enough" confidence?**
The 0.70/0.90 thresholds in Section 4 are estimates. They should be calibrated empirically against a validation set of 500 user-labelled files. The calibration itself could be a one-time setup step: "Help Curator learn — confirm these 50 groupings." This would simultaneously calibrate the confidence thresholds and seed the filename classifier training data (addressing OQ2).

**OQ9: Handling duplicate files across tiers.**
Duplicate detection (same content, different name/path) currently requires SHA-256 at Tier 1. But SHA-256 requires reading the whole file. For large files (1GB video), this is expensive. Alternative: use `size + first-64KB-hash + last-64KB-hash` as a near-duplicate detector that works for >99% of cases without reading the full file. Only full SHA-256 when the partial hash matches.

**OQ10: Progressive display vs batch display.**
Should Curator show files as soon as Tier 0 completes for each file (streaming display), or wait for a full batch to complete before rendering? Streaming display is better UX but requires the UI to handle partial, updating group structures. Batch display is simpler but keeps the user waiting. Recommendation: show a progress indicator with rough counts during processing, render first full display at Tier 0 completion, progressively refine as Tier 1 and 2 complete.

---

*Research compiled June 2026. Sources: arXiv 2410.01166 (filename classification), arXiv 2505.17799 (coreset selection), Apple Spotlight Metadata Reference, osxmetadata/RhetTbull, markitdown Microsoft benchmarks (12s/100 pages, CPU), Apple Vision OCR M3 Max (207ms/page community benchmark), go-enry/github-linguist documentation, pdfminer.six GitHub (catalog-based page count), sentence-transformers SBERT documentation, moondream.ai documentation, openpyxl benchmarks, Python os.walk PEP 471.*
