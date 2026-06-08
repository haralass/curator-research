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

### 1.1 TurboVec / TurboQuant (RyanCodrai/turbovec)

**What it is:** Rust implementation of Google Research's TurboQuant algorithm. Compresses float32 embedding vectors to significantly reduce memory usage. Python bindings. ARM and x86 kernels.

**Claimed result:** 10M documents, 384-dim float32 → 31GB. With TurboQuant compression → ~4GB. Compression ratio approximately 7–8×.

**Curator problem it could solve:** Layer 3 compressed vector memory (R11 §1.3). Curator will eventually index embeddings for 30k–1M+ file chunks. Keeping this in manageable RAM is important for a local tool running on a personal Mac.

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes |
| Apple Silicon | Claimed ARM kernels — must verify |
| Stable IDs | Requires verification |
| Incremental insert | Claimed online ingest — must verify |
| Deletions | Requires verification |
| Python bindings | Yes |
| Filtered search | Claimed allowlist support — must verify |
| Maturity | Early / research-stage — not yet production-proven at scale |

**Correct framing:** TurboVec compresses the **vector index**, not the AI model. The 31GB → 4GB claim is about embedding storage, not model weights.

**Verdict:** Research-only pending R11 benchmark. If it passes recall, latency, thermal, and deletion requirements, promote to Phase 2 or MVP.

**Open questions:**
- Does the Rust Python binding block the GIL during search?
- Is the index memory-mapped or RAM-only?
- What is the actual recall@10 at 100k 384-dim vectors?
- Does quantization degrade context boundary detection (R4)?

---

### 1.2 usearch (unum-cloud/usearch)

**What it is:** High-performance HNSW vector search library. ARM/x86 optimized. Memory-mapped indexes. Custom quantization. Python, Swift, C bindings.

**Curator problem it solves:** Layer 3 vector search (R11 §4.2). Already transitively available in Curator via `unisim`.

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes |
| Apple Silicon | Yes — ARM NEON |
| Stable IDs | Yes — external ID support |
| Incremental insert | Yes |
| Deletions | Yes |
| Python bindings | Yes |
| Swift bindings | Yes |
| Filtered search | Yes |
| Maturity | Production-ready |

**Verdict:** Current default candidate for Layer 3. Benchmark to confirm memory/latency profile matches Curator's needs. Low risk.

---

### 1.3 FAISS (facebookresearch/faiss)

**What it is:** Facebook's production vector search library. Widely used, battle-tested. Python bindings via C++ core.

**Curator problem it currently solves:** Used in Curator's existing pipeline for semantic grouping.

**Fit assessment for Layer 3:**

| Criterion | Assessment |
|---|---|
| Stable IDs | No — internal positions only (requires separate mapping table) |
| Incremental insert | Limited — IVF index requires periodic rebuild |
| Deletions | No clean deletion (mark-and-remove) |
| Filtered search | No built-in |
| Apple Silicon | Works, not specifically optimized |

**Verdict:** Keep as fallback. Likely superseded by usearch for Layer 3 in Curator. Migration path: remove direct FAISS usage, route through VectorIndex abstraction (R11 §4.1).

---

### 1.4 sqlite-vec (asg017/sqlite-vec)

**What it is:** SQLite extension for vector search. Maintained by Alex Garcia (also maintains sqlite-utils). Stores vectors inside SQLite. SQL-native filtering.

**Curator problem it could solve:** Small-scale vector search where SQL filtering is more important than ANN speed. Complementary to FTS5.

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes |
| Apple Silicon | Yes |
| Stable IDs | Yes (SQL primary keys) |
| Incremental insert | Yes (SQL INSERT) |
| Deletions | Yes (SQL DELETE) |
| Python bindings | Yes |
| Filtered search | Yes — SQL WHERE |
| Maturity | Active, early production |
| ANN performance | Weaker than HNSW at large scale |

**Verdict:** Phase 2. Interesting for small-scale filtered search (< 50k vectors) or as complement for metadata-heavy queries. Not a replacement for usearch at scale.

---

## Category 2 — Hybrid Local Retrieval

### 2.1 SQLite FTS5 (built-in)

**What it is:** Full-text search extension built into SQLite. Porter stemmer, unicode61 tokenizer, trigram tokenizer, BM25 ranking.

**Curator problem it solves:** Filename search, note content search, course code search, Greek text search (after NFC + accent stripping).

**Status:** Already planned in Layer 0 (TECH_engineering_foundation.md). FTS5 virtual table on `files` with:
- `filename_normalized` (NFC + accent-stripped)
- `filename_greeklish` (transliteration output)
- `text_snippet` (first 512 chars of extracted text)

**Verdict:** MVP. Already committed.

---

### 2.2 Tantivy (quickwit-oss/tantivy)

**What it is:** Rust full-text search engine (Lucene-like). Python bindings via `tantivy-py`. Significantly faster than FTS5 for large corpora.

**Curator problem it could solve:** At scale (> 500k documents), FTS5 may become slow. Tantivy could replace it.

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes |
| Apple Silicon | Yes |
| Schema flexibility | Yes |
| Greek support | Unicode-aware — must verify accent stripping |
| Incremental | Yes |
| Maturity | Production-ready |

**Verdict:** Phase 2. Not needed for MVP (FTS5 is sufficient at 30k files). Re-evaluate at 200k+ files.

---

## Category 3 — Disk-Backed & Scalable Vector Memory

### 3.1 DiskANN (microsoft/DiskANN)

**What it is:** Graph-based ANN index designed for datasets that exceed RAM. Stores most of the index on SSD, uses RAM for hot layers only.

**Curator problem it could solve:** If Curator's vector memory grows beyond available RAM (e.g., 10M+ chunks from screenshots, PDFs, code files).

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes |
| Apple Silicon | Requires verification — originally x86 optimized |
| Stable IDs | Requires mapping layer |
| Maturity | Research-grade, Microsoft internal production |

**Verdict:** Research-only for now. Revisit if Curator scales past 1M vectors and RAM becomes a constraint.

---

### 3.2 Milvus Lite / LanceDB

**What it is:** Lightweight embedded vector databases. LanceDB is columnar, Apache Arrow-based, designed for local use.

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes (lite/embedded modes) |
| Apple Silicon | Yes |
| Stable IDs | Yes |
| SQL-like filtering | Yes |
| Maturity | Active, early production |
| Dependency weight | Heavier than usearch or sqlite-vec |

**Verdict:** Phase 2 / research. Interesting if Curator needs a more database-like interface to vector memory. May be overkill for MVP.

---

## Category 4 — Visual Document Retrieval

### 4.1 ColPali / ColQwen

**What it is:** Vision-language models that embed document page images directly, without OCR. The model sees the page visually and produces searchable embeddings. Developed at Illuin Technology, published at EMNLP 2024.

**Curator problem it could solve:** Scanned PDFs, visually rich slides, diagrams, screenshots — files that are hard to extract text from but visually meaningful. Retrieval without OCR.

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes — model runs locally (small enough for M-series) |
| Apple Silicon | Yes — tested on M1/M2 |
| OCR dependency | None — visual retrieval |
| File types | PDFs, PNGs, JPGs, screenshots |
| Maturity | Research-grade, active |
| Model size | ~4–7GB — significant |

**Curator-specific concern:** Model size and embedding cost. Generating visual embeddings is expensive. Would be Layer 5 (Temporary High-Precision) or scheduled batch processing during Asleep thermal state only.

**Verdict:** Phase 2 / research. Very interesting for screenshot memory and scanned document retrieval. Not MVP — thermal/compute cost too high for always-on use.

---

### 4.2 Apple Vision Framework (built-in)

**What it is:** Apple's on-device OCR and image understanding. Available in Swift via `VNRecognizeTextRequest`. Free, local, fast, Apple Silicon optimized.

**Curator problem it solves:** OCR for scanned PDFs, screenshots, handwritten notes. Already referenced in R3 (Adaptive Reading Engine) as the preferred OCR backend.

**Status:** Already planned. This is the primary OCR path for Curator.

**Verdict:** MVP. Already committed. Not a new candidate — confirming it is the correct choice.

---

### 4.3 ocrmac (straussmaximilian/ocrmac)

**What it is:** Python wrapper around Apple Vision OCR. Allows calling `VNRecognizeTextRequest` from Python.

**Curator problem it solves:** Same as Apple Vision — allows the Python sidecar to invoke Apple's OCR without requiring a Swift bridge for every OCR call.

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes |
| Apple Silicon | Yes — wraps native framework |
| Quality | Same as Apple Vision (excellent) |
| Dependency weight | Light |
| Maturity | Stable, small |

**Verdict:** MVP candidate for Python-side OCR path. Evaluate alongside the Swift bridge approach. Use whichever is simpler to maintain.

---

### 4.4 MarkItDown (microsoft/markitdown)

**What it is:** Python library that converts various file formats (PDF, DOCX, PPTX, XLSX, images, audio, HTML) to Markdown. Developed by Microsoft. Supports local processing.

**Curator problem it could solve:** Tier 1 / Tier 2 text extraction for office documents, replacing custom parsers.

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes |
| Apple Silicon | Yes |
| Format coverage | Wide — PDF, DOCX, PPTX, XLSX, HTML, images |
| OCR | Via Azure or local model — must use local path only |
| Maturity | Active, Microsoft-maintained |
| Curator concern | Default Azure OCR must be disabled — local-only path required |

**Verdict:** Phase 2 candidate for Tier 1 extraction fallback. Must verify local-only operation (no network calls). If confirmed, could simplify the extraction layer.

---

## Category 5 — Code Understanding

### 5.1 Tree-sitter

**What it is:** Incremental parsing library. Supports 100+ languages. Produces concrete syntax trees. Used in Neovim, Helix, GitHub Copilot, Zed.

**Curator problem it could solve:** Understanding code files. Extracting imports, class names, function signatures, module structure. Better than regex for code sketch memory (Layer 2).

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes |
| Apple Silicon | Yes |
| Python bindings | Yes (`tree-sitter` + language grammars) |
| Supported languages | Python, Swift, JS/TS, C/C++, Rust, Java, Go, etc. |
| Incremental | Yes — reparsing only changed sections |
| Maturity | Production-ready |

**Curator use case:** For code projects in Curator (e.g., CS coursework), tree-sitter could produce a better Layer 2 sketch:
- Language detected
- Import statements extracted
- Top-level class/function names
- Module structure

**Verdict:** Phase 2. Useful for CS student users (Curator's primary persona). Not MVP — file type detection + extension heuristics are sufficient for Layer 2 initially.

---

## Category 6 — OCR & Document Parsing

### 6.1 pdfminer.six / pdfplumber

**What it is:** Pure-Python PDF text extraction. pdfplumber wraps pdfminer with a better API, supports table extraction.

**Status:** Already referenced in Curator's existing pipeline. pdfplumber is the Tier 1 PDF text extractor.

**Verdict:** MVP. Already committed.

---

### 6.2 pymupdf (fitz)

**What it is:** Python bindings for MuPDF. Significantly faster than pdfminer for large PDFs. Can render pages for visual inspection. Extracts text, images, annotations.

**Curator problem it could solve:** Faster Tier 1 PDF extraction, especially for large lecture slides (100+ pages).

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes |
| Apple Silicon | Yes |
| Speed vs pdfplumber | 5–10× faster on large PDFs |
| Table extraction | Basic |
| License | AGPL (free for open source / personal use) |

**Verdict:** Phase 2 candidate. Replace pdfplumber for large PDFs if extraction becomes a thermal bottleneck. Benchmark first.

---

### 6.3 docling (DS4SD/docling)

**What it is:** IBM Research document understanding library. Converts PDFs, DOCX, PPTX, images to structured Markdown/JSON with layout understanding. Used in RAG pipelines.

**Curator concern:** Heavy dependency, potentially network-calling models.

**Verdict:** Research-only. Too heavy for Curator's Tier 1. May be relevant for Tier 2 deep extraction on specific file types in Phase 2.

---

## Category 7 — Thermal & Energy-Aware Scheduling

### 7.1 macOS NSProcessInfo.thermalState (Apple built-in)

**What it is:** Apple API reporting current thermal pressure level: `nominal`, `fair`, `serious`, `critical`.

**Status:** Core input for R11 Thermal Governor (§3). Already committed as the primary thermal signal.

**Verdict:** MVP. Already committed.

---

### 7.2 IOKit Power Source API (Apple built-in)

**What it is:** Apple API for battery/AC state, charge level, discharge rate.

**Status:** Secondary input for Thermal Governor. Used to distinguish battery vs. AC-powered modes.

**Verdict:** MVP. Already committed.

---

### 7.3 psutil

**What it is:** Cross-platform Python library for CPU, memory, disk, network, and process monitoring.

**Curator problem it could solve:** CPU load monitoring for Thermal Governor. Battery info complement to IOKit.

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes |
| Apple Silicon | Yes |
| CPU load | Yes — per-core and aggregate |
| Battery | Yes — complements IOKit |
| Maturity | Production-ready, widely used |

**Verdict:** MVP candidate for Thermal Governor CPU monitoring in Python sidecar.

---

### 7.4 Python `nice` / `ionice` (POSIX)

**What it is:** POSIX process priority. `os.nice(15)` reduces CPU scheduling priority. `ionice -c 3` sets idle I/O class.

**Status:** Should be applied to all Curator sidecar workers at startup.

**Verdict:** MVP. One-line implementation. Apply immediately in Phase 1.

---

## Category 8 — Filesystem Safety & Recovery

### 8.1 send2trash (arsenetar/send2trash)

**What it is:** Cross-platform Python library that moves files to the system Trash instead of deleting them.

**Status:** Already committed in Curator's NON-NEGOTIABLE rules. `os.unlink` is forbidden — only `send2trash` or `NSFileManager.trashItem`.

**Verdict:** MVP. Already committed.

---

### 8.2 atomicwrites (untitaker/atomicwrites)

**What it is:** Python library for atomic file writes (write to temp, then rename). Prevents partial writes from corrupting files.

**Curator problem it could solve:** Any time Curator writes to a file (metadata, cache, index), the write should be atomic.

**Fit assessment:**

| Criterion | Assessment |
|---|---|
| Local-first | Yes |
| Apple Silicon | Yes |
| Safety | Atomic (temp + rename) |
| Maturity | Stable, small |

**Note:** Python's `os.replace()` is atomic on POSIX and Windows. `atomicwrites` adds cross-platform edge case handling. For Curator's index writes, `os.replace()` may be sufficient.

**Verdict:** Phase 2. Evaluate for index and cache writes. `os.replace()` may be sufficient for MVP.

---

### 8.3 iCloud edge cases — fsevents + POSIX

**What it is:** Not a library — a known class of problems.

iCloud Drive's on-demand sync (`.icloud` placeholder files) creates edge cases for Curator:
- Files may be in `new` state but not locally available
- `stat()` succeeds but `open()` fails with ENOENT after eviction
- `os.rename()` across iCloud boundaries may behave unexpectedly
- File identity (inode) may change after iCloud sync

**Status:** Referenced in TECH_engineering_foundation.md. Must be handled in ingest.py.

**Research needed:** Systematic mapping of iCloud + POSIX edge cases specifically on macOS 14+. Consider filing as a TECH document (TECH_icloud_edge_cases.md) if complexity warrants it.

**Verdict:** MVP concern. Document and handle in Layer 1 (Ingestion). Not a new library — a constraint.

---

## Adoption Decision Registry

This registry records the current status of all candidates evaluated in this document. Updated as benchmarks complete and build phases progress.

| Tool | Category | Curator Verdict | Phase | Notes |
|---|---|---|---|---|
| usearch | Vector search | Default candidate | MVP | Pending R11 benchmark confirmation |
| FAISS | Vector search | Fallback / deprecated | Phase 1 | Migrate to VectorIndex abstraction |
| TurboVec | Compressed vectors | Research-only | Phase 1.5+ | Pending R11 benchmark |
| sqlite-vec | Vector search | Phase 2 | Phase 2 | Complement to FTS5 for small-scale |
| DiskANN | Disk-backed vectors | Research-only | Phase 3+ | Only if > 1M vectors |
| LanceDB | Vector DB | Research-only | Phase 2+ | Evaluate if VectorIndex abstraction needs more |
| FTS5 | Full-text search | Committed | MVP | Already in engineering foundation |
| Tantivy | Full-text search | Phase 2 | Phase 2 | If FTS5 becomes bottleneck at scale |
| Apple Vision | OCR | Committed | MVP | Primary OCR path |
| ocrmac | OCR (Python) | MVP candidate | MVP | Evaluate alongside Swift bridge |
| ColPali | Visual retrieval | Phase 2 | Phase 2 | High cost — Asleep state only |
| MarkItDown | Document extraction | Phase 2 | Phase 2 | Verify local-only path |
| Tree-sitter | Code parsing | Phase 2 | Phase 2 | CS student use case |
| pdfplumber | PDF extraction | Committed | MVP | Tier 1 extractor |
| pymupdf | PDF extraction | Phase 2 | Phase 2 | Benchmark for large PDF speedup |
| docling | Document understanding | Research-only | Phase 3+ | Too heavy for Tier 1 |
| NSProcessInfo | Thermal | Committed | MVP | Thermal Governor primary signal |
| IOKit | Battery | Committed | MVP | Thermal Governor secondary |
| psutil | CPU monitoring | MVP candidate | MVP | Thermal Governor in Python sidecar |
| os.nice / ionice | Worker priority | Committed | MVP | Apply at sidecar startup |
| send2trash | Safe delete | Committed | MVP | NON-NEGOTIABLE rule |
| atomicwrites | Atomic writes | Phase 2 | Phase 2 | Evaluate vs os.replace() |

---

## Maintenance

This document should be updated when:
- A new candidate is evaluated
- A benchmark result changes a candidate's verdict
- A tool is adopted or rejected after Phase testing
- A new problem area emerges that requires external tooling

Do not add tools to this document without completing the Curator-fit test (§ "Curator-Fit Test"). Do not mark a tool as "MVP" without benchmark evidence or a strong prior art justification.
