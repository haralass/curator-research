# Curator: Local AI File Intelligence System for macOS
_Product Vision & Research Direction — Last updated: 2026-05-31_

---

## 1. The Core Idea

Curator is a **local AI personal file organizer for macOS**. It organizes the user's personal files locally on their Mac — no cloud, no external servers, no hardcoded rules like "if PDF → put in Documents."

The core insight: files are not just names, extensions and dates. Every file has meaning, history, provenance, usage patterns, relationships to other files, and connections to a course, project, trip, invoice, research topic, client, or life phase. Curator understands files as personal objects, not as bytes on disk.

**Core positioning:**
> Curator doesn't just organize folders. It organizes the user's personal digital archive based on meaning, usage, provenance, and memory.

---

## 2. The Real Problem

The surface problem is a full Downloads folder. The deeper problem is **lost context**:

After days or weeks, users can't remember:
- Why they downloaded a PDF
- Which course or project it belonged to
- Whether they ever used it
- Whether a newer version exists
- Whether it's a duplicate
- Whether it relates to other files
- Whether it should stay active or be archived

Finder shows names, sizes, dates. Spotlight searches words. Hazel applies user-written rules. None of these understand what a file *means in the user's life*. Curator fills this gap.

---

## 3. Why This Has a Future

Three converging trends make this the right moment:

1. **File volume is growing** — PDFs, screenshots, lecture notes, AI-generated files, exports, code projects, invoices, zip files, drafts. The chaos will increase, not decrease.

2. **Privacy demand is growing** — many users won't upload their entire filesystem to a cloud AI. "Everything runs locally on your Mac" is a strong differentiator.

3. **Local AI is now feasible** — Apple Silicon, local embeddings (Qwen3-Embedding-0.6B), local LLMs (Ollama), OCR, VLMs, and tools like ColPali make a fully on-device AI system practical in 2026.

---

## 4. How to Position It

**Not this:**
> "An AI that puts files in folders."

**This:**
> "A privacy-first macOS app that uses local AI, semantic clustering, behavioral signals, provenance metadata, and conformal prediction to organize, explain, and maintain the user's personal digital archive — with formal uncertainty quantification."

The difference: this framing shows architecture, research depth, a trust layer, and commercial positioning. It cannot be dismissed as a thin LLM wrapper.

---

## 5. What Curator Is and Is Not

**Curator IS:**
- Local AI file organizer
- Semantic file clustering system
- TextTwin / AI-readable document layer
- Duplicate and version detector
- Safe file mover with manifest review
- Semantic search engine with local RAG
- File Biography system (Markov-based)
- Personal archive memory layer with forgetting curves (FLRS)
- AI-readable filesystem builder
- Conformal prediction routing system
- Personal Knowledge Graph with 6 typed edges

**Curator IS NOT (in Phase 1):**
- AI tutor or study planner
- Note-taking app
- Mental health app
- Full autonomous agent
- Cloud storage app
- ChatGPT replacement
- Notion replacement

Do one thing perfectly first: understand and organize files safely.

---

## 6. Core Operation Flow

1. User grants Curator access to folders (Downloads, Desktop, Documents, project folders)
2. Curator **scans without moving anything**
3. Builds a local SQLite catalog with file metadata, provenance, and macOS extended attributes
4. **Level Router** decides analysis depth per file (see Section 8)
5. Content extraction → TextTwin → embeddings → clustering → naming → routing
6. Curator presents a **manifest** (proposed organization plan) — never acts blindly
7. User approves, corrects, or rejects each proposal
8. Every action becomes feedback for the system to learn

---

## 7. Full Architecture

```
Scan → Catalog → Privacy Filter → Level Router → Content Extraction
→ TextTwin → Embeddings → UMAP → HDBSCAN → Toponymy Naming
→ FLACON Routing → Conformal Prediction → Confidence Tier
→ Review Manifest → Safe Execution → Learning
```

### Key Components

**Scan:** Find all files in user-selected folders via FSEvents/Endpoint Security Framework.

**Catalog:** Store metadata in SQLite: path, filename, extension, size, dates, SHA-256 hash, source URL (kMDItemWhereFroms), quarantine metadata, iCloud status, macOS extended attributes.

**Privacy Filter:** Flag sensitive files (`.env`, private keys, bank statements, medical docs, legal contracts) for protected-mode analysis — no deep content read without explicit user consent.

**Level Router:** Tiered analysis (see Section 8) — the most important architectural decision.

**Content Extraction:** Text from PDFs (text layer + OCR), DOCX/PPTX/XLSX, source code, zip archives, images.

**TextTwin:** AI-readable document twin: `.clean.txt` + `.ai.md` + `.map.json` with page anchors, table extraction, source linking.

**Embeddings:** **Qwen3-Embedding-0.6B** (52.33 MMTEB vs BGE-M3's 40.88 — ~30% better). MPS-native on Apple Silicon. For visual/scanned PDFs: **ColPali/ColQwen2-v1.0** (87.3 NDCG@5 on ViDoRe, no OCR required).

**UMAP:** Dimensionality reduction for visualization and clustering preparation.

**HDBSCAN:** Dynamic clustering without predefined categories. Streaming updates via **DenStream** in Ongoing Mode.

**Toponymy Naming:** From the UMAP/HDBSCAN authors. Hierarchical cluster naming with `OllamaNamer` backend for local LLM. Three-level hierarchy (broad → specific → instance). `pip install toponymy`.

**FLACON Routing:** 6-dimensional flag distance (α=0.4 semantic, β=0.4 ARISE-embedded flags, γ=0.2 temporal). ARISE (ICLR 2025) generates semantic descriptions for macOS metadata flag values (kMDItemContentType categories) instead of one-hot encoding.

**Conformal Prediction:** Split CP with LAC/RAPS nonconformity scores. Provides prediction sets `C(x)` with formal coverage guarantee `P(true_folder ∈ C(x)) ≥ 1-α`. For non-IID/drifting behavior: OCP-Unlock+ (arXiv:2604.17984, April 2026).

**Confidence Tiers:**
- High: semantic + filename + source + cluster all agree → auto-approve queue
- Medium: two plausible folders, ambiguous content → review queue
- Low: outlier, sensitive, OCR-failed → no auto-move

**Review Manifest:** User sees proposed organization before any file moves.

**Safe Execution:** Dry-run, conflict detection, no overwrite, atomic moves, undo log, open-file skip, iCloud placeholder handling.

**Learning:** Every accept/reject/correction/undo updates the routing model.

---

## 8. Level Router

The Level Router is one of Curator's most important and research-novel components. It decides how much analysis each file needs — preventing both over-analysis (slow, compute-heavy) and under-analysis (poor routing).

### Levels

**Level 0 — Metadata only**
Extension, filename, size, dates, hash, source URL, quarantine metadata.
For: installers (.dmg, .pkg), exact duplicates (same SHA-256), large archives, obvious temp files.

**Level 1 — Filename + learned patterns**
Course codes, project names, date patterns, keywords from filename.
For: `EPL342_Lecture_10_Normalization.pdf` → routable without opening content.

**Level 2 — Content extraction**
Open file, extract text or summary.
For: files with uninformative filenames (`document.pdf`, `scan001.pdf`).

**Level 3 — Embeddings + cluster comparison**
Compare Qwen3 embedding against existing cluster centroids, HDBSCAN membership probabilities, HDBSCAN outlier score.
For: ambiguous content that doesn't clearly match Level 1 patterns.

**Level 4 — LLM reasoning**
Local LLM (via Ollama, llama3.2:3b) for: ambiguous cases, smart rename generation, cluster naming explanation, files that fit two plausible destinations.

**Level 5 — OCR / VLM / ColPali**
For: screenshots (llava:7b for description), scanned PDFs without text layer (ColQwen2-v1.0 for visual embedding), diagrams, UI mockups, tables where layout matters.

### Research Contribution
The Level Router is a research contribution: formal evaluation of what fraction of files resolve at each level, accuracy gain per level, compute cost per level, and optimal stopping rules. This has not been studied in the PIM literature.

---

## 9. Privacy Filter

Before deep analysis, Curator detects potentially sensitive files and enters **protected mode**:

- `.env`, private keys, certificates, password exports
- Wallet files, bank statements, tax documents
- Medical records, legal documents, identity documents
- Contracts, credentials

Protected mode: no deep content read, no LLM call, no summarization without explicit user consent. Shown in Privacy Dashboard. User can override per-file.

---

## 10. Content Extraction Engine

| File Type | Method |
|---|---|
| PDF (text layer) | pdfminer/pymupdf text extraction |
| PDF (scanned) | OCR (Tesseract) or ColPali (if text layer < 100 chars/page) |
| DOCX/PPTX/XLSX | python-docx, python-pptx, openpyxl |
| Images/Screenshots | llava:7b description + OCR |
| Source code | Language detection, imports, function signatures, README relation |
| ZIP archives | List internal files, detect project type, README extraction (no full decompress needed) |

---

## 11. TextTwin Engine

TextTwin creates an AI-readable twin document for every PDF and structured document. Not just PDF-to-text — a **source-linked, page-anchored, AI-readable document layer**.

Three outputs per document:
1. **`.clean.txt`** — Clean plain text, correct reading order
2. **`.ai.md`** — Markdown with headings, bullets, tables, structure preserved
3. **`.map.json`** — Machine-readable source map: page anchors, confidence per section, OCR warnings, detected tables/figures

The TextTwin preserves: page numbers, title hierarchy, tables, footnotes, captions, figure references, OCR confidence scores, warnings about likely extraction errors.

**Why this matters:** PDFs are designed for display, not AI. TextTwin makes the filesystem AI-readable — enabling semantic search, chunked RAG, citation-level references, and LLM reasoning over document content without hallucination about page location.

---

## 12. Embedding Engine

**Primary:** Qwen3-Embedding-0.6B (52.33 MMTEB, MPS-native on Apple Silicon, replaces BGE-M3 at 40.88). Fix: `attn_implementation="eager"` for macOS SDPA bug.

**Visual/scanned PDFs:** ColQwen2-v1.0 (Apache 2.0, 87.3 NDCG@5 on ViDoRe benchmark, ~4-6 GB on MPS, ~1-3 sec/page on M2 Pro). Routing heuristic: if extractable text < 100 chars/page → use ColPali, else use Qwen3.

**Storage:** ChromaDB for vector storage. ColPali embeddings: two-stage (ANN retrieval via pooled vectors + exact MaxSim re-ranking from saved `.pt` files).

Embedding granularity:
- **File-level:** for routing/clustering
- **Chunk-level:** for semantic search and RAG

---

## 13. UMAP + HDBSCAN Clustering

No hardcoded categories. Files reveal their own natural groups.

**Pipeline:** Qwen3 embeddings → UMAP (2D for visualization, higher-dim for HDBSCAN) → HDBSCAN (min_cluster_size tuned per user, noise points = orphans)

**Streaming updates (Ongoing Mode):** DenStream for incremental clustering when new files arrive. No full HDBSCAN refit for every new file.

**Stability validation:** GhostUMAP2 (`pip install ghostumap2`) runs during 6-month refits. Identifies (r,d)-unstable points → route to human review. Correlated with Markov File Biography entropy (both capture the same underlying instability — see File 37).

Example discovered clusters (not hardcoded):
- EPL342 Database Lectures
- OIK398 HR Exam Material
- Curator Research Papers
- Old App Installers
- Screenshots for UI Design
- Bank Documents
- GitHub Project Archives
- Unread Economics Papers

---

## 14. Cluster Naming

**Toponymy** (from the UMAP/HDBSCAN authors at TutteInstitute). Three-level recursive naming: `pip install toponymy`. Uses OllamaNamer for local LLM backend. No cloud dependency.

Input: top filenames, keywords, metadata, representative files, source domains, user folder context.

Output: human-readable, hierarchical cluster names like "EPL342 Database Lectures and Assignments."

---

## 15. Routing Engine — Conformal Prediction

Curator's routing uses **Conformal Prediction (CP)** for formally guaranteed folder suggestions — a novel contribution in the PIM space.

**How it works:**
- Nonconformity score: how surprising is it that file `x` belongs to folder `f`? (LAC or RAPS score)
- Calibration set: user's historical file placements
- Prediction set `C(x)`: all folders whose nonconformity score is below the `(1-α)` quantile
- Formal guarantee: `P(true_folder ∈ C(x)) ≥ 1-α` (e.g., 90% coverage)

**For non-IID/drifting behavior (user behavior changes over time):**
OCP-Unlock+ (arXiv:2604.17984, April 2026) — adversarial semi-bandit CP with sublinear regret. Handles distribution shift unlike SPS (IID only).

**Feedback loop / Algorithmic monoculture correction:**
IPS (Inverse Propensity Score) correction prevents the rich-get-richer bias where folders never shown in prediction sets never receive positive feedback (see File 31). Novel research contribution.

**Signal combination:**
```
Routing score = α·semantic_similarity + β·ARISE_flag_distance + γ·temporal_proximity
             + δ·provenance_signal + ε·FLRS_prior + ζ·co_access_history
```

Where provenance = `kMDItemWhereFroms` xattr (index 0 = download URL, index 1 = referring page). Note: `com.apple.provenance` xattr is NOT the download URL — it's an app attribution pointer to ExecPolicy. The correct signal is `kMDItemWhereFroms`.

---

## 16. Confidence System

Confidence is computed from signals, not from LLM self-report:

| Tier | Condition | Action |
|---|---|---|
| High | Semantic + filename + source + cluster all agree | Auto-approve queue |
| Medium | Two plausible folders, weak filename, overlap | Review queue |
| Low | Outlier, sensitive, OCR-failed, ambiguous | No auto-move, flag for manual |

---

## 17. Review Manifest

Curator never moves files without showing a manifest first. Per-file the manifest shows:
- Current location → proposed location
- Proposed rename (if any)
- Reason (in plain language)
- Confidence tier
- Similar files already in destination (sibling preview)
- Duplicate/version status
- Whether review is required

User actions: approve / reject / change folder / rename / archive / ignore / mark sensitive / undo later.

The **sibling preview** is critical for trust: showing 12 similar files already in the proposed folder makes the reason self-evident.

---

## 18. Review UI — Summary Screen

After scan, the main screen shows:

```
Found 842 files
612 can be organized
94 high-confidence moves (auto-approve available)
38 need review
21 duplicates detected
17 probable older versions
56 PDFs processed through TextTwin
9 PDFs had OCR warnings
14 files excluded (privacy filter)
```

Grouped view by proposed folder. Per-file: filename, destination, reason, confidence, similar files, preview, action buttons. User always sees what will happen and why.

---

## 19. Safe Execution Engine

Only approved moves execute. Safety requirements:
- Dry-run before execution
- Conflict detection (no silent overwrite)
- No overwrite without explicit permission
- Atomic moves where possible
- Rename conflict handling
- Full undo log (restore to original path)
- Skip open/locked files
- Handle iCloud placeholder files
- Handle symlinks

If Curator loses user trust, the product is dead. Safety is non-negotiable.

---

## 20. Duplicate and Version Detection

**Exact duplicates:** SHA-256 hash comparison.

**Near-duplicates:** Cosine similarity > 0.95 on Qwen3 embeddings. Detected via LSH (approximate, not exhaustive pairwise).

**Versions:** Filename pattern matching + semantic similarity.
Example: `essay_draft.docx`, `essay_final.docx`, `essay_final2.docx`, `essay_submitted.pdf` → same document, different phases.

**Archive DNA Fingerprinting:** Jaccard similarity on normalized filename sets within ZIP archives — groups archive versions without full decompression.

**Image duplicates:** Perceptual hashing (pHash) for screenshots and images.

---

## 21. Smart Rename

Preview-based rename suggestions (not automatic):

```
document.pdf → OIK398_Hybrid_Work_HR_Lecture.pdf
IMG_4920.png → Curator_UI_Settings_Screenshot.png
final_final.docx → Economics_Crisis_Essay_Final_Draft.docx
scan001.pdf → EPL342_Normalization_Notes.pdf
```

Based on: content, folder context, source metadata, project, date, cluster membership.

---

## 22. Folder README Generator

For each organized folder, Curator can generate a short human-readable summary:

> This folder contains material for EPL342 Databases. It includes 12 lecture PDFs, 4 assignments, 3 SQL files, and 6 diagrams. The most-accessed files are Lecture 8, Lecture 10, and Assignment 3. There are 2 probable older versions and 1 unread PDF that matches the course topic.

This is especially valuable when returning to a folder after months away.

---

## 23. File Biography (Markov-Based)

Every file has a recorded history — its biography. Tracked events:
- When it appeared (with source)
- When analyzed and at what level
- Which cluster it was assigned to
- When moved and where
- When reopened
- When archived
- When it changed cluster (project phase change)

**Markov File Biography:** Each file's cluster assignment history forms a Markov chain `[c₁, c₂, c₃, ...]`. The stationary entropy H(π) of this chain measures file instability — how often the file changes clusters. High entropy = ambiguous/boundary file → candidate for human review.

**Cross-validation with GhostUMAP2:** GhostUMAP2-unstable points and high-Markov-entropy files should identify the same ambiguous files. Combined ambiguity score: `A(f) = 0.5·rank(I_ghostumap) + 0.5·rank(H_markov)` (see File 37).

---

## 24. FLRS — File Location Retention Score

FLRS models how well a user remembers a file's location over time. Grounded in cognitive neuroscience of spatial memory binding.

**Formula:**
```
FLRS(f, t) = R(t, f) × A(f) × B(f, t)
```

Where:
- `R(t, f) = e^(-t/S)` — Ebbinghaus forgetting curve (replicated: Murre & Dros 2015, PLOS ONE). S₀ ≈ 1.0 day for initial stability (associative binding fragility: Pertzov et al. 2012, Talamini & Gorree 2012).
- `A(f)` — Accessibility penalty (folder depth, organizational clarity)
- `B(f, t)` — Retrieval-strengthening bonus, implemented via **FSRS-4.5** (not SM-2). FSRS fitted on 500M+ Anki reviews: `B = R(t, S_f)` where S_f updated by FSRS after each retrieval event. Direct navigation = "Good" grade; search-assisted = "Again" grade.

**FLRS decay crossover:** FLRS = 0.5 at ~17 hours after single access (S₀ = 1.0 day).
**After 1st retrieval:** S ≈ 3–5 days. After 3rd retrieval: S ≈ 30–50 days ("feels permanent").

**FLRS-triggered actions:**
- FLRS approaching 0.5 → resurface file (remind user where it is)
- FLRS < 0.2 → candidate for archive suggestion
- Very low FLRS + no recent access → antilibrary candidate

**FLRS novelty:** First formalization of file *location* memory decay (vs. content importance). ForgetIT (EU project, 2013-2018) modeled content importance for deletion — different problem. FLRS is the first computational implementation of "Design for Forgetting" (Sas & Whittaker CHI 2013) for location memory specifically.

---

## 25. Antilibrary / Unread but Important

Files downloaded but never opened. Curator finds these and adds semantic context:

> "You downloaded 8 papers on Cyprus banking crisis. 5 were never opened. Of these, 2 are highly relevant to your current research project."

Not just "unread." Ranked by relevance to active clusters and projects.

---

## 26. Semantic Search / Local RAG

With TextTwin chunks and Qwen3 embeddings, Curator supports semantic search:

- "Find the PDF about macroprudential policy"
- "Where are the screenshots for the Curator UI?"
- "Files related to EPL342 normalization"
- "Why do I have this file?"

Hybrid retrieval: dense (Qwen3) + sparse (BM25) with Reciprocal Rank Fusion (RRF) + reranker. For visual PDFs: ColQwen2 two-stage retrieval.

---

## 27. "Why Do I Have This?" Feature

User clicks any file and asks. Curator responds:

> "You downloaded this from Moodle on May 12. It matches Lecture 8 of OIK398. It was opened alongside Lecture 7 and Lecture 9. It hasn't been opened since. It's suggested to stay in OIK398 / Lectures."

Makes the system feel genuinely intelligent.

---

## 28. Temporal Co-Access Graph — Implicit Project Detection

Files opened together within a 30-minute session form co-access edges. Weighted graph built over time. Dense subgraphs → implicit projects detected via Leiden community detection.

**Why this matters:** A tax spreadsheet, a scanned PDF, and a Python script that are always opened together form a project — even if semantically dissimilar. Folder structure won't capture this. HDBSCAN won't capture this.

**Temporal File Coupling (TFC) score:** `TFC(fᵢ, fⱼ) = w(fᵢ, fⱼ) / √(degree(fᵢ) × degree(fⱼ))`

Dual-layer organization: **semantic layer** (HDBSCAN topics) + **project layer** (TCAG projects). Complementary views.

Research contribution: first application of temporal coupling analysis (established in software engineering via CodeScene) to personal file systems. See File 40.

---

## 29. Save-Point Interception

Curator intercepts the native macOS Save dialog (`NSSavePanel`) via Finder Sync Extension + XPC + Accessibility API. Before the file is written to disk, Curator displays a ranked folder suggestion as an accessory view inside the Save panel.

Features local Qwen3 embedding of the typed filename + app context + time-of-day → CP prediction set → top suggestion in <50ms.

**Zero-Displacement Rate (ZDR):** fraction of files that never need to move after first save. Formally: `ZDR = |{f : loc₀(f) = loc₃₀ₐₐᵧₛ(f)}| / |total files saved|`

No existing AI file organizer does this. Default Folder X uses recency heuristics only. See File 39.

---

## 30. Organizational Burden Index (OBI)

A single scalar metric for the cognitive cost of a user's filesystem:

```
OBI = 0.30·H̃(L|I) + 0.20·FDE + 0.20·OR + 0.15·NDD + 0.15·(1 - FLRS̄)
```

Where:
- `H̃(L|I)` = normalized H(location|intent) — how uncertain is location given intent (File 29)
- `FDE` = Folder Depth Entropy — unpredictability of folder hierarchy depth
- `OR` = Orphan Ratio — fraction of files in no HDBSCAN cluster
- `NDD` = Near-Duplicate Density — fraction of file pairs with cosine similarity > 0.95
- `1 - FLRS̄` = mean forgotten-file burden

First operationalization of PIM Burden (Dinneen & Julien JASIST 2022) as a computable metric. Displayed as a health dashboard with scalar gauge, time series, and decomposition view. See File 41.

---

## 31. H(location|intent) — Universal Quality Metric

**H(L|I) = -Σ p(l,i) log p(l|i)**

Zero = knowing intent perfectly determines file location (perfect organization). Maximum = file location is random given intent.

**Novelty:** Zero prior art in PIM literature for this metric (exhaustive search of CHI, IUI, SIGIR, JASIST 2000-2026). The closest relative is Theil's Uncertainty Coefficient — never applied to PIM.

**CP connection (provable):** Angelopoulos et al. NeurIPS 2024 proved `H(Y|X) ≤ φ(α, E[|S|])` — the CP prediction set size provides a computable upper bound on H(L|I). Curator's router provides a formal guarantee language. See File 29.

**Intent proxies on macOS:** Spotlight query text (cleanest, ~35-50% coverage), application launch context, session-windowed file access sequences.

---

## 32. Personal Knowledge Graph

6-typed-edge heterogeneous graph — no prior system combines all 6:
1. `semantic_similarity` — embedding cosine similarity
2. `NER_co_occurrence` — shared named entities (people, organizations, concepts)
3. `temporal_proximity` — creation/access time proximity
4. `source_domain` — same download source
5. `citation_link` — explicit citation in document
6. `workflow_sequence` — temporal co-access (from TCAG)

Prior art has at most 2 edge types. First claim: no prior personal file organization system constructs a heterogeneous graph combining all 6 dimensions.

---

## 33. Visual File Map

DataMapPlot (TutteInstitute — same authors as UMAP + HDBSCAN): interactive HTML visualization via DeckGL, embedded in Curator's Tauri webview as iframe. Shows semantic map of the entire filesystem. "Nobody has built this as a consumer feature" (File 27).

---

## 34. Agent Mode (Phase 7)

Natural language instructions → Curator prepares plan for user review:
- "Organize my Downloads but don't move anything without review"
- "Find all duplicates"
- "Collect everything related to Curator research"
- "Create archive for OIK398"

Agent mode only after trust layer is established. The user must trust the system before granting it broader agency.

---

## 35. Privacy Dashboard

Dedicated screen showing:
- Which folders are monitored
- Which files were analyzed (and at what level)
- Which files were excluded (privacy filter)
- Which files have TextTwin / embeddings
- Confirmation that nothing left the Mac
- Controls: delete local index, export data, reset learning

---

## 36. Two Core Modes

**Deep Reorganize:** Full scan + HDBSCAN clustering + manifest for hundreds/thousands of files. For users with accumulated chaos.

**Ongoing Mode:** When a new file appears in Downloads, Curator uses DenStream + existing cluster centroids to route it without full HDBSCAN refit. Instant routing suggestion within seconds of file appearance.

---

## 37. Database Schema

### Files
`file_id, current_path, original_path, filename, extension, mime_type, size_bytes, created_at, modified_at, last_opened_at, hash_sha256, is_duplicate_candidate, is_sensitive_candidate, scan_status, analysis_level_used`

### Metadata
`file_id, spotlight_kind, where_from_url, where_from_referrer, creator_app, quarantine_source, content_type, tags, icloud_status, author, title_metadata`

### ExtractedContent
`content_id, file_id, plain_text, markdown_text, extraction_method, ocr_used, extraction_confidence, page_count, language_detected, warnings`

### TextTwin
`texttwin_id, file_id, clean_txt_path, markdown_path, json_map_path, has_page_anchors, has_tables, ocr_pages, low_confidence_pages`

### Chunks
`chunk_id, file_id, content_id, page_number, chunk_text, start_offset, end_offset, source_anchor, chunk_type`

### Embeddings
`embedding_id, chunk_id or file_id, model_name, vector_path or vector_id, dimension, created_at`

### Clusters
`cluster_id, cluster_name, cluster_description, parent_cluster_id, cluster_confidence, created_by_method`

### FileCluster
`file_id, cluster_id, membership_probability, similarity_score, routing_reason, is_primary_cluster`

### MovePlan
`move_id, file_id, old_path, proposed_new_path, proposed_folder, proposed_filename, confidence_level, reason, status`
Status: `pending | approved | rejected | changed | executed | undone`

### BiographyEvents
`event_id, file_id, event_type, old_value, new_value, timestamp, source`
Event types: `scanned | extracted | embedded | clustered | renamed | moved | opened | archived | duplicate_detected | reviewed | undo`

### CoAccessEdges
`edge_id, file_id_a, file_id_b, session_id, co_access_count, last_co_access, weight`

### FLRSState
`file_id, stability_S, difficulty_D, last_access, last_retrieval_grade, flrs_score, fsrs_state_json`

### UserFeedback
`feedback_id, file_id, action_type, original_suggestion, user_choice, timestamp`
Action types: `accepted | rejected | changed_destination | changed_name | ignored | undo`

---

## 38. MVP — What to Build First

The MVP must prove Curator can take real files and produce logical, useful, safe suggestions.

**MVP includes:**
- Local scan + FSEvents
- SQLite catalog
- Privacy exclusions
- Level Router (Levels 0-3)
- Basic content extraction (PDFs, DOCX, images)
- TextTwin basic
- Qwen3-Embedding-0.6B embeddings
- UMAP + HDBSCAN
- Toponymy cluster naming
- Folder suggestions
- Manifest review UI
- Confidence tiers (High/Medium/Low)
- Safe execution + undo log
- Basic duplicate detection (hash + embedding similarity)
- Basic smart rename

If this works for 500–5000 files in a trustworthy way, the product exists.

---

## 39. Advanced Features (Post-MVP)

In priority order:
1. ColPali/ColQwen2 for visual PDFs
2. File Biography + Markov entropy
3. FLRS + FSRS-4.5
4. Antilibrary / unread-but-important detection
5. DenStream streaming (Ongoing Mode)
6. GhostUMAP2 stability + 6-month refit
7. Personal Knowledge Graph (6 typed edges)
8. Semantic search + local RAG
9. Temporal Co-Access Graph + project detection
10. Save-Point Interception (NSSavePanel)
11. Organizational Burden Index (OBI)
12. H(location|intent) quality metric
13. DataMapPlot visual file map
14. Conformal Prediction + IPS correction
15. Agent Mode

---

## 40. Research Direction

The research framing is not "I built an AI file organizer." It is:

> **Local AI for Personal File Organization with Calibrated Routing, File Biography, and Memory-Aware Retrieval**

### Core Research Question
How can a local AI system organize personal files in a way that is useful, safe, explainable, and adapted to how users remember and re-find their files?

### Novel Contributions
1. **H(location|intent)** as a universal PIM quality metric — zero prior art
2. **FLRS** (File Location Retention Score) — first formalization of file location memory decay using Ebbinghaus curves + FSRS
3. **Markov File Biography** — per-file cluster assignment history as Markov chain, stationary entropy for instability
4. **Conformal Prediction for file routing** — formal coverage guarantees in PIM
5. **IPS-corrected CP calibration** — correcting algorithmic monoculture feedback loop (NeurIPS-level open problem)
6. **Level Router** — formal evaluation of tiered analysis depth
7. **Temporal Co-Access Graph** — first application of temporal coupling to personal files
8. **Save-Point Interception** — pre-save folder prediction, Zero-Displacement Rate metric
9. **Organizational Burden Index (OBI)** — first computable operationalization of PIM Burden (JASIST 2022)

### Research Axes

**Level Router:** What fraction of files resolve at each level? What is the accuracy/compute tradeoff? When is it optimal to stop and request review?

**Calibrated confidence:** When to auto-suggest? When to show 2-3 options? When to not touch the file?

**File Biography:** Can we model a file's life as a Markov chain and detect instability from stationary entropy?

**FLRS:** Can we predict when users forget file locations? Does FSRS outperform SM-2 for file access patterns?

**Review fatigue:** How many review decisions can a file organizer request before the user disengages? What is the optimal batch size? (Open research question — not yet covered in literature.)

**Mixed file clustering:** How do you evaluate cluster quality when members are mixed types (PDFs, screenshots, code, Excel)?

**macOS provenance:** How much routing accuracy does `kMDItemWhereFroms` add beyond semantic embeddings alone?

**FileNav Benchmark:** Measure time-to-find with: Finder only, Spotlight only, Curator semantic search, Curator organized folders.

---

## 41. Evaluation Metrics

**Standard metrics:**
- Acceptance rate (% of proposals user accepts)
- Correction rate (% changed)
- Undo rate (% reversed after execution)
- Review fatigue (files reviewed before disengagement)
- Time saved (organization speed vs. manual)
- Findability (time-to-find after Curator vs. baseline)
- Cluster quality (do co-clustered files semantically belong together?)

**Novel metrics (Curator-specific):**
- **H(location|intent)** reduction pre/post Curator
- **OBI** reduction over time
- **Zero-Displacement Rate (ZDR)** with Save-Point Interception
- **PFAS** (Project-Folder Alignment Score) — gap between user folders and TCAG projects
- **CP coverage validation** — empirical verification that P(true_folder ∈ C(x)) ≥ 1-α

---

## 42. Business Model

**Free version:** Basic scan, limited suggestions, manual review only.

**Pro (one-time, ~€49):**
- Unlimited organization
- Duplicate + version detection
- Smart rename
- Visual file map
- Safe execution + undo
- Local semantic clustering
- TextTwin

**Plus (subscription, ~€7/month):**
- Semantic search + local RAG
- Personal Knowledge Graph
- Agent Mode
- FLRS + continuous learning
- Advanced File Biography
- Archive DNA

**Professional (~€99+/user or annual):**
- For researchers, freelancers, legal/accounting professionals
- Project archives, exportable manifests, advanced privacy controls

Start with one-time Pro purchase (easier trust for Mac utility). Subscription later, only for genuinely ongoing-value features.

---

## 43. Target Users (Phase 1)

1. **Students** — lecture PDFs, screenshots, assignments, exam material, zip files, papers
2. **Developers** — GitHub zips, code snippets, error screenshots, docs, project fragments
3. **Researchers** — papers, notes, figures, datasets, duplicates, unread PDFs
4. **Freelancers/Professionals** — invoices, contracts, briefs, drafts, final versions, client files
5. **Mac power users** — already pay for productivity utilities, early adopters

---

## 44. Roadmap

| Phase | Features |
|---|---|
| 1 — Core | Scan, catalog, Level Router, content extraction, SQLite |
| 2 — Semantic | Qwen3, UMAP, HDBSCAN, Toponymy, manifest review |
| 3 — Trust | Confidence tiers, safe execution, undo, manifest UI |
| 4 — Intelligence | TextTwin, ColPali, duplicate/version detection, smart rename |
| 5 — Memory | FLRS + FSRS, File Biography, Antilibrary |
| 6 — Research | H(location|intent), OBI, CP routing, IPS correction |
| 7 — Interaction | Semantic search, "Why do I have this?", local RAG, Agent Mode |
| 8 — Advanced | Save-Point Interception, TCAG project detection, Visual file map |

---

## 45. Key Risks

| Risk | Mitigation |
|---|---|
| Bad clustering | Tuning + GhostUMAP2 stability + user feedback loop |
| Trust failure from bad move | Manifest + undo + dry-run — always |
| Speed (heavy AI on all files) | Level Router is mandatory |
| Privacy fear | Privacy dashboard + local-only messaging |
| Feature overload | Strict phase separation |
| Feedback loop / monoculture | IPS correction in CP calibration |

---

## 46. The Moat

Not one feature — the combination:
- Local-first privacy
- Level Router (novel research)
- TextTwin (AI-readable filesystem)
- File Biography + FLRS (memory layer)
- Conformal Prediction (formal guarantees)
- TCAG (project detection)
- Save-Point Interception (proactive routing)
- macOS provenance metadata integration
- Semantic + behavioral + temporal signals combined
- OBI (measurable organizational health)

> Curator is not an AI folder sorter. It is a personal file intelligence system.

---

## 47. Ecosystem (Long-Term)

The Curator engine — understanding files — is the foundation:

- **Curator:** Where is my material? (file organization, FLRS, biography)
- **MarginOS/Trace:** What am I keeping from what I read? (highlights, annotations, source-linked notes)
- **MindCurator/Compass:** How am I learning it? (study map, active recall, exam prep)
- **SessionOS:** Where did I leave off? (context capsules, tab/window memory, resume sessions)

Build the brain that understands files first. Then every other app uses that brain.

---

## 48. One Paragraph Summary

Curator is a privacy-first local AI app for macOS that organizes personal files based on content, usage, provenance, time, and inter-file relationships. Instead of hardcoded folders or simple rules, it uses an adaptive Level Router that decides how much analysis each file needs — from basic metadata to embeddings, clustering, LLM reasoning, OCR, and visual PDF analysis. With Qwen3-Embedding-0.6B, ColQwen2 for visual PDFs, UMAP, HDBSCAN, Toponymy naming, conformal prediction routing with formal coverage guarantees, FLRS forgetting curves (FSRS-4.5 based), manifest review, and safe undo, Curator proposes natural folder organization without endangering files. Longer-term, it evolves into a full personal file intelligence layer with File Biography, Markov-based instability detection, Temporal Co-Access Graph for implicit project detection, Save-Point Interception, Organizational Burden Index, and an H(location|intent) quality metric — the first computable, information-theoretic measure of personal file organization quality.
