# R12 — External Research & Open-Source Acceleration Scan

> **Core principle: "Benchmark-backed adoption only."**
>
> Curator must not collect libraries. It must collect evidence.
>
> Every external tool evaluated here must pass the Curator-fit test before it is adopted. A tool that looks impressive is not a tool that helps Curator's users.

---

## Purpose

This document is a living scan of public systems research and open-source projects relevant to Curator's architecture. It exists to:

1. Prevent re-inventing things that are already solved
2. Identify tools that could make Curator faster, lighter, safer, or more local
3. Maintain a clear record of what was considered, what was adopted, and why

This document does **not** make adoption decisions. It surfaces candidates, records their fit assessment, and assigns an evaluation status. Final adoption is governed by benchmarks (R11) and the build plan (FINAL_mvp_build_plan.md).

---

## Curator-Fit Test

Before any tool is evaluated in detail, it must pass this gate:

| Question | Required answer |
|---|---|
| What Curator problem does it solve? | Must name a specific, real problem |
| Is it local-first? | Yes — no network calls during normal operation |
| Does it work on Apple Silicon (arm64)? | Yes, or a clear workaround exists |
| Does it reduce RAM, CPU, heat, or latency? | At least one of these must improve |
| Does it support stable IDs (not just positions)? | Yes, or the mapping overhead is acceptable |
| Does it support incremental updates and deletes? | Yes — Curator ingests continuously |
| Does it have Python or Swift integration? | Yes, or C/Rust with FFI |
| Is it mature enough for a personal tool? | Usable, maintained, not vaporware |
| Does it have a safe fallback if it fails? | Curator must degrade gracefully |
| Verdict | MVP / Phase 2 / Research-only / Reject |

---

## Category 1 — Compressed Vector Search

### 1.1 TurboVec (RyanCodrai/turbovec)

**Source:** [GitHub — RyanCodrai/turbovec](https://github.com/RyanCodrai/turbovec) | [PyPI](https://pypi.org/project/turbovec/) | Created March 2026

**What it actually is:**  
A Rust vector index with Python bindings, implementing the TurboQuant quantization algorithm. TurboQuant was published as a Google Research paper at ICLR 2026 — its original scope is **LLM KV cache compression** (compressing the attention key-value cache of transformer models during inference). TurboVec extends the quantization math to general vector search indexes. These are related but distinct use cases.

**What TurboQuant does:**  
Combines PolarQuant (rotation-based coordinate transform) with a 1-bit QJL residual correction. The method is data-oblivious, requires no training, and operates within a factor of ≈2.7 of the information-theoretic limit. At 3-bit: ~6x memory reduction on KV caches with near-zero quality loss on LLM benchmarks.

**Correct compression math for vector indexes (verified):**

| Dimension | float32 size / vector | At 2-bit (16x) | At 4-bit (8x) |
|---|---|---|---|
| 384-dim | 1,536 bytes | 96 bytes | 192 bytes |
| 768-dim | 3,072 bytes | 192 bytes | 384 bytes |
| 1536-dim | 6,144 bytes | 384 bytes | 768 bytes |

For 10M vectors at 1536-dim: float32 = ~61GB (not 31GB as stated in the README). The README's 31GB baseline has been independently flagged as inaccurate. The 4GB compressed target is approximately correct for 2-bit 1536-dim. The 31GB number does not correspond to any standard embedding dimension. [Source: independent analysis at blog.pebblous.ai]

**Relevant for Curator because:** Curator uses 384-dim embeddings. At this dimension, float32 for 10M vectors = ~15.4GB. At 2-bit compression = ~0.96GB. This would be significant RAM savings at large scale — but the 384-dim performance characteristics are not published; available claims are for 1536-dim.

**Verified features:**
- ARM NEON and AVX-512BW kernels — 12–20% faster than FAISS IndexPQFastScan on ARM [Source: turbovec README]
- Training-free (no codebook, no calibration data, no separate train phase)
- Available on PyPI and crates.io

**Unverified / open questions:**
- Stable external ID support (Curator requires `file_id` lookup — not confirmed)
- True deletion support (not confirmed in available docs)
- Filtered search (allowlist/blocklist) — claimed, not independently verified
- Index persistence (save/load) behavior — not confirmed
- Recall@10 at 384-dim specifically — not published
- "10M scale is not yet reproduced" by independent analysts [Source: blog.pebblous.ai]

**Curator verdict:** Research-only until benchmark. The compression math is sound (TurboQuant is peer-reviewed ICLR 2026). The implementation maturity and 384-dim behavior require independent verification before adoption.

---

### 1.2 usearch (unum-cloud/usearch)

**Source:** [GitHub — unum-cloud/usearch](https://github.com/unum-cloud/usearch) | [Benchmarks](https://github.com/unum-cloud/usearch/blob/main/BENCHMARKS.md)

**What it is:** HNSW-based vector search library. ARM NEON + AVX-512 optimized. Memory-mapped storage. Custom quantization (int8, fp16, fp32, binary). Python and Swift bindings. Billion-scale benchmarks published.

**Why it is the current default candidate:**
- External ID support (stable `file_id` lookups, not internal positions)
- Memory-mapped index (can exceed RAM without full load)
- Filtered search
- Incremental inserts and deletions
- Apple Silicon ARM NEON optimization (confirmed)
- Swift bindings for future native layer integration
- Already transitively available in Curator via `unisim`
- Active, commercial-grade development

**Curator verdict:** Default candidate for Layer 3. Pending R11 benchmark at 384-dim, 30k–100k vector scale.

---

### 1.3 FAISS (facebookresearch/faiss)

**Source:** [GitHub — facebookresearch/faiss](https://github.com/facebookresearch/faiss)

**Current role:** Used in Curator's existing pipeline (sparse_snf.py context).

**Weaknesses for Layer 3:**
- No stable external IDs — uses internal integer positions (requires separate mapping table in SQLite)
- No built-in filtered search
- IVF index type requires periodic rebuild — not truly incremental
- No clean deletion (mark-and-rebuild approach)

**Migration path:** Keep as fallback. Route all vector operations through the VectorIndex abstraction (R11 §4.1). Replace with usearch for Layer 3 in MVP.

**Curator verdict:** Fallback / deprecated for new code. Migrate away.

---

### 1.4 sqlite-vec (asg017/sqlite-vec)

**Source:** [GitHub — asg017/sqlite-vec](https://github.com/asg017/sqlite-vec) | [Site](https://alexgarcia.xyz/sqlite-vec/) | MIT/Apache-2.0 dual license

**What it is:** SQLite extension for vector search, written in C with no external dependencies. KNN search, multiple distance metrics, SIMD-accelerated. Runs on macOS, iOS, Linux, Windows, WASM. Maintained by Alex Garcia (also: sqlite-utils, datasette).

**Why it's interesting for Curator:**
- Vectors live inside SQLite — no separate index file, no separate process
- SQL-native filtering (`WHERE context_id = X`) — natural complement to FTS5
- Stable IDs via SQL primary keys
- Incremental inserts/deletes via standard SQL
- Very light dependency (C, no Rust, no heavy runtime)
- Transactional (ACID with WAL)

**Weakness:** No HNSW algorithm — brute-force or basic quantized scan. Performance degrades significantly above ~50k vectors. Not suitable as the primary semantic search backend at Curator's projected scale.

**Curator verdict:** Phase 2. Evaluate as complement to FTS5 for small-corpus filtered search. Not a replacement for usearch.

---

### 1.5 DiskANN (microsoft/DiskANN)

**Source:** [GitHub — microsoft/DiskANN](https://github.com/microsoft/DiskANN)

**What it is:** Graph-based ANN index designed for datasets exceeding RAM. Stores most of the graph on SSD, keeps hot layers in RAM.

**Apple Silicon status:** No confirmed arm64 / macOS support found in available documentation or GitHub issues. Originally built for x86 Linux/Windows.

**Curator verdict:** Research-only. No confirmed Apple Silicon support. Revisit only if Curator scales past 1M vectors and RAM becomes a hard constraint that usearch memory-mapped storage cannot handle.

---

## Category 2 — Hybrid Local Retrieval

### 2.1 SQLite FTS5 (built-in)

**Source:** [SQLite FTS5 Documentation](https://www.sqlite.org/fts5.html)

**What it is:** Full-text search extension built into SQLite. BM25 ranking, Porter stemmer, unicode61 tokenizer, trigram tokenizer. Zero additional dependency.

**Status:** Already committed in Layer 0 (TECH_engineering_foundation.md). FTS5 virtual table on `files` with:
- `filename_normalized` (NFC + accent-stripped)
- `filename_greeklish` (transliteration output)
- `text_snippet` (first 512 chars of extracted text)

**Curator verdict:** **MVP. Already committed.**

---

### 2.2 Tantivy (quickwit-oss/tantivy)

**Source:** [GitHub — quickwit-oss/tantivy](https://github.com/quickwit-oss/tantivy) | [Python bindings](https://github.com/quickwit-oss/tantivy-py)

**What it is:** Apache Lucene-inspired full-text search engine written in Rust. Approximately 2x faster than Lucene in published benchmarks. Powers Quickwit and ParadeDB in production. Python bindings available via `tantivy-py` (requires Rust to build from source if no wheel available for the target platform).

**Why it matters for Curator:** FTS5 has known limitations at large document counts. If Curator scales to 500k+ files with deep text extraction, FTS5 query latency may degrade.

**Fit assessment:**
- Greek support: unicode-aware tokenization — accent stripping must be applied in Python before indexing (same as FTS5)
- Incremental index updates: yes
- Python bindings: yes (wheel availability on arm64 must be verified)
- Maturity: production-ready (Quickwit uses it in production)

**Curator verdict:** Phase 2. Not needed for MVP (FTS5 is sufficient at 30k files). Re-evaluate at 200k+ files.

---

## Category 3 — Disk-Backed & Scalable Vector Memory

### 3.1 LanceDB

**Source:** [GitHub — lancedb/lancedb](https://github.com/lancedb/lancedb)

**What it is:** Embedded vector database using the Lance columnar format (Apache Arrow-based). Designed for local-first use. SQL-like filtering, versioning, incremental updates.

**Fit assessment:**
- Local-first embedded mode: yes
- Apple Silicon: yes
- Stable IDs and filtering: yes
- Dependency weight: moderate (heavier than usearch or sqlite-vec)
- Maturity: active, early production

**Curator verdict:** Research-only / Phase 2. More database-like than Curator needs for Layer 3. Evaluate if the VectorIndex abstraction requires richer query capabilities than usearch provides.

---

## Category 4 — Visual Document Retrieval

### 4.1 ColPali / ColQwen (illuin-tech/colpali)

**Source:** [arXiv:2407.01449](https://arxiv.org/abs/2407.01449) | [ICLR 2025 paper](https://proceedings.iclr.cc/paper_files/paper/2025/file/99e9e141aafc314f76b0ca3dd66898b3-Paper-Conference.pdf) | [GitHub — illuin-tech/colpali](https://github.com/illuin-tech/colpali)

**What it is:** Vision-language model that produces multi-vector embeddings directly from document page images. No OCR required. Retrieval via late interaction (similar to ColBERT but for images). Evaluated on the ViDoRe benchmark (Visual Document Retrieval Benchmark), covering multiple domains and languages.

**Why it matters for Curator:** Scanned PDFs, slides with diagrams, screenshots, handwritten notes — files that standard text extraction handles poorly. ColPali retrieves these visually, without needing OCR to succeed first.

**Fit assessment:**
- Local-first: yes (model runs locally)
- Apple Silicon: tested on M-series (confirmed in community usage)
- OCR dependency: none — this is the point
- File types: PDFs (page images), PNG, JPG, screenshots
- Model size: ColQwen2-2B is ~4GB on disk — significant for a background tool
- Embedding cost: generating visual embeddings is expensive (seconds per page)
- Maturity: ICLR 2025 published, active development

**Curator-specific constraint:** The model size and per-page embedding cost make this unsuitable for always-on background processing. It should be treated as Layer 5 (Temporary High-Precision) or scheduled for Asleep thermal state only.

**Curator verdict:** Phase 2. Very relevant for screenshot memory and scanned document retrieval. Not MVP — thermal/compute cost requires thermal-aware scheduling before adoption.

---

## Category 5 — OCR & Document Parsing

### 5.1 Apple Vision Framework (built-in)

**Source:** [Apple Developer Documentation — VNRecognizeTextRequest](https://developer.apple.com/documentation/vision/vnrecognizetextrequest)

**What it is:** Apple's on-device OCR engine, available via Swift `VNRecognizeTextRequest`. Free, local, Apple Silicon optimized. On macOS Sonoma+, also supports LiveText (on-device advanced recognition).

**Status:** Already committed as the primary OCR backend for Curator's Swift layer. Not a new candidate — confirming it is the correct choice.

**Curator verdict:** **MVP. Already committed.**

---

### 5.2 ocrmac (straussmaximilian/ocrmac)

**Source:** [GitHub — straussmaximilian/ocrmac](https://github.com/straussmaximilian/ocrmac) | [PyPI](https://pypi.org/project/ocrmac/)

**What it is:** Python wrapper around Apple's Vision framework OCR via `pyobjc-framework-Vision`. Accepts file paths or PIL Image objects. Returns text, confidence scores, and bounding boxes. Supports both the standard Vision OCR and the newer LiveText framework (macOS Sonoma+). macOS 10.15+ required.

**Why it matters for Curator:** Allows the Python sidecar to invoke Apple's OCR without a full Swift bridge round-trip for every file. Same OCR quality as the native Swift path (same underlying `VNRecognizeTextRequest`).

**Fit assessment:**
- Local-first: yes (wraps native framework, no network)
- Apple Silicon: yes (native framework)
- Quality: identical to Swift Vision path
- LiveText support: yes (macOS Sonoma+)
- Dependency: pyobjc-framework-Vision (macOS-only, reasonable)
- Maturity: stable, actively maintained

**Open question:** For Curator's architecture, is the Python OCR path (ocrmac) or a Swift XPC call to the native layer the better design? The answer depends on inter-process communication overhead vs. the cost of adding an XPC endpoint.

**Curator verdict:** MVP candidate for Python-side OCR in sidecar. Evaluate alongside the Swift bridge approach. Use whichever is simpler to maintain and has lower IPC overhead.

---

### 5.3 pdfplumber

**Source:** [GitHub — jsvine/pdfplumber](https://github.com/jsvine/pdfplumber) | MIT license

**Status:** Already committed as Tier 1 PDF text extractor.

**Performance reference:** ~18 pages/sec on text extraction benchmarks. Strong table detection (returns rows/columns as Python lists). [Source: pdfmux benchmark 2026]

**Curator verdict:** **MVP. Already committed.**

---

### 5.4 PyMuPDF (fitz)

**Source:** [GitHub — pymupdf/PyMuPDF](https://github.com/pymupdf/PyMuPDF) | [Docs](https://pymupdf.readthedocs.io/) | AGPL-3.0 license

**What it is:** Python bindings for MuPDF. Significantly faster than pdfplumber for pure text extraction.

**Concrete benchmark (pdfmux 2026, confirmed by multiple independent sources):**
- PyMuPDF: ~180 pages/sec text extraction
- pdfplumber: ~18 pages/sec text extraction
- Delta: **8–12x faster** on plain text

**Why the difference matters for Curator:** A 200-page lecture PDF takes pdfplumber ~11 seconds. PyMuPDF takes ~1.1 seconds. At 30k PDFs, the thermal and time difference is material.

**License concern:** AGPL-3.0. For a personal local tool distributed to users, this requires attention. PyMuPDF also offers a commercial license. For Curator's personal-use distribution model, AGPL compliance requires that Curator itself be open-sourced under AGPL, or a commercial license is obtained.

**When to prefer pdfplumber:** Table-heavy financial or structured documents — pdfplumber's table detection is more reliable than PyMuPDF's.

**Curator verdict:** Phase 2. Benchmark on Curator's actual PDF corpus (lecture slides, lab reports, CS assignments). If the speed difference creates a thermal advantage on long extraction queues, adopt for Tier 1 non-table PDFs. Resolve license before shipping.

---

### 5.5 MarkItDown (microsoft/markitdown)

**Source:** [GitHub — microsoft/markitdown](https://github.com/microsoft/markitdown) | [PyPI](https://pypi.org/project/markitdown/)

**What it is:** Python library converting PDF, DOCX, PPTX, XLSX, HTML, images, and audio to Markdown. Designed to feed LLM pipelines. Processes files in memory without creating temporary files.

**Local-only operation (confirmed):** Core converters (PDF, DOCX, PPTX, XLSX, HTML) work entirely offline. Optional cloud features (Azure Content Understanding, LLM integration) require network — these must be explicitly disabled. Install with `markitdown[pdf,docx]` to get offline format support without cloud dependencies.

**Python version requirement:** 3.10+. Curator uses 3.11.15 — compatible.

**Why it's interesting for Curator:** Single library for multi-format Tier 1 extraction, potentially simplifying the extraction layer. Markdown output is well-suited for FTS5 indexing.

**Curator concern:** MarkItDown is optimized for LLM input, not necessarily for structured metadata extraction (e.g., Excel headers as sketch signals, not full Markdown tables). May produce more text than Curator needs for Layer 2 sketch signals.

**Curator verdict:** Phase 2. Evaluate as optional Tier 1 fallback for formats that the existing extraction chain handles poorly. Do not replace the primary extraction pipeline with it until benchmarked on Curator's actual file corpus.

---

## Category 6 — Code Understanding

### 6.1 Tree-sitter (tree-sitter/py-tree-sitter)

**Source:** [GitHub — tree-sitter/py-tree-sitter](https://github.com/tree-sitter/py-tree-sitter) | [PyPI](https://pypi.org/project/tree-sitter/)

**What it is:** Incremental parsing library with Python bindings. Supports 100+ languages (Python, Swift, JS/TS, C, Rust, Java, Go, etc.). Produces concrete syntax trees. Used in Neovim, Helix, GitHub Copilot, Zed. Pre-compiled wheels available for major platforms via `tree-sitter-languages`.

**What it can do for Curator's Layer 2 sketch:** For code files (CS coursework, lab assignments):
- Language detection (confirmed, not just extension-based)
- Import statements extracted (exact module names)
- Top-level class and function names
- Module structure (files vs. packages)
- Incremental re-parse on file change (only changed regions)

**Apple Silicon status:** Pre-compiled wheels available. `tree-sitter-languages` provides binary wheels for arm64.

**Curator use case:** CS student persona (primary Curator target) has many code files: Python labs, Swift projects, Java assignments, C exercises. Better Layer 2 sketch for these than filename + folder context alone.

**Curator verdict:** Phase 2. Not needed for MVP (extension heuristics + first-line detection are sufficient for initial sketch). Add in Phase 2 when CS course code becomes a first-class context type.

---

## Category 7 — Thermal & Energy-Aware Scheduling

### 7.1 macOS NSProcessInfo.thermalState (Apple built-in)

**Source:** [Apple Developer Documentation](https://developer.apple.com/documentation/foundation/nsprocessinfo/1417480-thermalstate)

**Four states:** `nominal`, `fair`, `serious`, `critical`.

**Important Apple Silicon caveat:** On M-series chips, the `fair` state covers both moderate and heavy thermal pressure. The granularity is lower than on Intel Macs. A Mac that is actively throttling under load may still report `fair` rather than `serious`. Curator must supplement thermalState with CPU load monitoring to detect actual throttling earlier. [Source: Stan's blog — Building a macOS app to know when my Mac is thermal throttling, Dec 2025]

**Status:** Core signal for the Thermal Governor (R11 §3). Already committed.

**Curator verdict:** **MVP. Already committed. Use with CPU load supplement.**

---

### 7.2 IOKit Power Source API (Apple built-in)

**Source:** [Apple Developer Documentation — IOKit](https://developer.apple.com/documentation/iokit)

**What it provides:** Battery vs. AC detection, charge level, discharge rate (mA). Available from Swift via `IOPSCopyPowerSourcesInfo()`.

**Status:** Secondary signal for Thermal Governor. On battery, Curator defaults to Attentive state regardless of thermalState.

**Curator verdict:** **MVP. Already committed.**

---

### 7.3 psutil

**Source:** [GitHub — giampaolo/psutil](https://github.com/giampaolo/psutil) | [PyPI](https://pypi.org/project/psutil/) | [Docs](https://psutil.readthedocs.io/)

**What it is:** Cross-platform Python library for CPU, memory, disk, network, and process metrics.

**macOS / Apple Silicon behavior (researched):**
- `psutil.cpu_percent(interval=1)` — works on Apple Silicon, provides aggregate and per-core CPU %
- `psutil.sensors_battery()` — provides battery percentage and charging state on macOS
- `psutil.sensors_temperatures()` — limited on macOS; does not reliably report CPU temperature on Apple Silicon. For temperature, `powermetrics` subprocess is more accurate but requires `sudo`.
- For Curator's needs (CPU load + battery state), psutil is sufficient without privilege escalation.

**Curator verdict:** **MVP candidate for Python-side Thermal Governor inputs.** Use `cpu_percent` for load, `sensors_battery` for charging state. Do not rely on `sensors_temperatures` on Apple Silicon.

---

### 7.4 Python `os.nice()` (POSIX built-in)

**What it is:** POSIX process priority. `os.nice(15)` lowers CPU scheduling priority so sidecar workers yield to user applications.

**Cost:** Zero. One line. No dependency.

**Curator verdict:** **MVP. Apply at sidecar startup immediately.**

---

## Category 8 — Filesystem Safety & Recovery

### 8.1 send2trash (arsenetar/send2trash)

**Source:** [GitHub — arsenetar/send2trash](https://github.com/arsenetar/send2trash) | [PyPI](https://pypi.org/project/send2trash/)

**Status:** Already committed. Curator's NON-NEGOTIABLE rule: `os.unlink` is forbidden. Only `send2trash` (Python) or `NSFileManager.trashItem` (Swift) are allowed for file removal.

**Curator verdict:** **MVP. Already committed.**

---

### 8.2 Atomic writes — `os.replace()` (Python built-in)

**What it is:** `os.replace(src, dst)` is atomic on POSIX systems (rename syscall). Write to temp file → `os.replace()` → destination. If the process crashes mid-write, the original is intact.

**When to use:** Any time Curator writes to a file (index file, cache, snapshot). This is the correct pattern for all Curator write operations.

**vs. atomicwrites library:** The `atomicwrites` third-party library adds cross-platform edge case handling (Windows). On macOS (POSIX), `os.replace()` alone is sufficient.

**Curator verdict:** **MVP. Use `os.replace()` for all file writes. No additional library needed.**

---

### 8.3 iCloud Drive edge cases

**What it is:** Not a library — a class of real filesystem behavior that Curator must handle.

iCloud Drive's on-demand download (eviction) creates edge cases:
- Files may appear to `stat()` correctly but `open()` fails (placeholder `.icloud` file, content evicted)
- `inode` may change after iCloud re-download
- `os.rename()` across iCloud sync boundaries may trigger unexpected behavior
- Large files may be evicted while Curator is mid-read

**Required handling in ingest.py:**
- Detect `.icloud` placeholder files (name starts with `.`, ends with `.icloud`)
- Do not attempt extraction on unevicted placeholders
- After iCloud re-download, verify inode hasn't changed before updating identity record
- Catch `FileNotFoundError` on `open()` even after `stat()` succeeds

**Curator verdict:** MVP concern. Document and handle in Layer 1 (Ingestion). Not a library — a set of defensive coding patterns.

---

## Adoption Decision Registry

Current status of all candidates. Updated as benchmarks complete.

| Tool | Category | Verdict | Phase | Key condition |
|---|---|---|---|---|
| usearch | Vector search | Default candidate | MVP | Pending R11 benchmark |
| FAISS | Vector search | Fallback / migrate away | Phase 1 | Route through VectorIndex abstraction |
| TurboVec | Compressed vectors | Research-only | Phase 1.5+ | Pending R11 benchmark; verify IDs/deletions/384-dim |
| sqlite-vec | Vector search | Phase 2 | Phase 2 | Complement to FTS5, not replacement at scale |
| DiskANN | Disk-backed vectors | Research-only | Phase 3+ | No confirmed Apple Silicon support |
| LanceDB | Vector DB | Research-only | Phase 2+ | Only if VectorIndex abstraction needs richer queries |
| FTS5 | Full-text search | **Committed** | MVP | Already in engineering foundation |
| Tantivy | Full-text search | Phase 2 | Phase 2 | Re-evaluate at 200k+ files |
| Apple Vision | OCR | **Committed** | MVP | Primary OCR path |
| ocrmac | OCR (Python) | MVP candidate | MVP | Evaluate vs. Swift XPC bridge |
| ColPali | Visual retrieval | Phase 2 | Phase 2 | High cost — Asleep state only, after thermal scheduling |
| MarkItDown | Document extraction | Phase 2 | Phase 2 | Verify no network calls; benchmark on Curator corpus |
| Tree-sitter | Code parsing | Phase 2 | Phase 2 | CS student use case, extension-heuristics sufficient for MVP |
| pdfplumber | PDF extraction | **Committed** | MVP | Tier 1 extractor |
| PyMuPDF | PDF extraction | Phase 2 | Phase 2 | 8–12x faster than pdfplumber; resolve AGPL license first |
| NSProcessInfo | Thermal | **Committed** | MVP | Supplement with CPU load on Apple Silicon |
| IOKit | Battery | **Committed** | MVP | Thermal Governor secondary signal |
| psutil | CPU monitoring | MVP candidate | MVP | Use cpu_percent + sensors_battery; skip sensors_temperatures |
| os.nice() | Worker priority | **Committed** | MVP | Apply at sidecar startup |
| send2trash | Safe delete | **Committed** | MVP | NON-NEGOTIABLE rule |
| os.replace() | Atomic writes | **Committed** | MVP | No additional library needed |

---

## Maintenance

This document should be updated when:
- A new candidate is evaluated
- A benchmark result changes a candidate's verdict
- A tool is adopted or rejected after Phase testing
- A new problem area emerges that requires external tooling

Do not add tools to this document without completing the Curator-fit test. Do not mark a tool as MVP without benchmark evidence or a confirmed prior-art justification.

---

## Sources

- [turbovec GitHub (RyanCodrai)](https://github.com/RyanCodrai/turbovec)
- [turbovec PyPI](https://pypi.org/project/turbovec/)
- [TurboQuant — ICLR 2026 (OpenReview)](https://openreview.net/pdf?id=tO3ASKZlok)
- [TurboQuant — Google Research Blog](https://research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/)
- [turbovec & TurboQuant Analysis 2026 — Pebblous (independent)](https://blog.pebblous.ai/report/turbovec-2026/en/)
- [TurboVec: Fit 10 Million Vectors in 4 GB — AI Monks (Medium)](https://medium.com/aimonks/turbovec-fit-10-million-vectors-in-4-gb-the-rag-index-that-changes-everything-3885a0e3c4ab)
- [usearch GitHub](https://github.com/unum-cloud/usearch)
- [usearch Benchmarks](https://github.com/unum-cloud/usearch/blob/main/BENCHMARKS.md)
- [sqlite-vec GitHub](https://github.com/asg017/sqlite-vec)
- [sqlite-vec site (Alex Garcia)](https://alexgarcia.xyz/sqlite-vec/)
- [ColPali — arXiv:2407.01449](https://arxiv.org/abs/2407.01449)
- [ColPali — ICLR 2025 paper](https://proceedings.iclr.cc/paper_files/paper/2025/file/99e9e141aafc314f76b0ca3dd66898b3-Paper-Conference.pdf)
- [ColPali GitHub (illuin-tech)](https://github.com/illuin-tech/colpali)
- [ocrmac GitHub](https://github.com/straussmaximilian/ocrmac)
- [MarkItDown GitHub (Microsoft)](https://github.com/microsoft/markitdown)
- [PyMuPDF vs pdfplumber benchmark 2026 (pdfmux)](https://pdfmux.com/blog/pymupdf-vs-pdfplumber/)
- [py-benchmarks PDF libraries (py-pdf)](https://github.com/py-pdf/benchmarks)
- [tantivy-py GitHub](https://github.com/quickwit-oss/tantivy-py)
- [py-tree-sitter GitHub](https://github.com/tree-sitter/py-tree-sitter)
- [NSProcessInfo thermalState — Apple Docs](https://developer.apple.com/documentation/foundation/nsprocessinfo/1417480-thermalstate)
- [Building a macOS thermal throttling app (Stan's blog, Dec 2025)](https://stanislas.blog/2025/12/macos-thermal-throttling-app/)
- [psutil documentation](https://psutil.readthedocs.io/)
- [psutil GitHub](https://github.com/giampaolo/psutil)
