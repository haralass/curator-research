# 14 — RAG for Personal Files
_Date: 2026-05-30_

## Assessment: Engineering vs Research
Mostly engineering — EXCEPT two novel angles:
1. **macOS Spotlight-native RAG**: Using kMDItemLastUsedDate, kMDItemWhereFroms, kMDItemKeywords as first-class retrieval signals. No published system does this.
2. **HDBSCAN soft-membership query expansion**: Using probabilities_ (soft membership) as continuous prior — files with high cluster confidence get boost when query maps near centroid. Not published.

## Recommended Architecture for Curator
```
Query
  → Ollama query rewriter (temporal extraction, file-type hints) [Llama 3.1 8B]
  → ChromaDB (BGE-M3 + cluster_id filter) || BM25 (rank_bm25, filename + text)
  → RRF fusion
  → bge-reranker-v2-m3
  → Ranked file list (paths + snippet + cluster)
```

## Chunking by File Type
- PDFs text-heavy: Docling → 512-token recursive
- PDFs image-heavy: ColPali (embed page images, NO OCR) — arXiv:2407.01449
- Images: moondream2 caption → embed caption + EXIF
- Code: AST chunking (function/class boundaries)
- Spreadsheets: Docling xlsx→md → row-group chunks
- ZIP: manifest doc + recursive

## Key Papers
- **ColPali** arXiv:2407.01449 (ICLR 2025) — visual PDF retrieval, bypass OCR entirely
- **ClusterRAG** arXiv:2605.18769 (2026) — HDBSCAN cluster-guided RAG, closest to Curator
- **CAR** arXiv:2511.14769 — adaptive cutoff via cluster structure, -60% tokens, +10% accuracy
- **PBR** arXiv:2510.08935 — query expansion with user history
- **Temporal RAG / TMRL** arXiv:2601.05549 — Matryoshka temporal embeddings

## Apple Silicon Stack
Ollama 0.19+ (MLX backend): prefill +57%, decode +93% vs llama.cpp.
Full stack (Llama 3.1 8B + BGE-M3 + moondream2 + bge-reranker) fits in 16GB unified memory at Q4_K_M.

## Privacy
Fully local → main risk is local privilege escalation and embedding inversion.
Mitigations: ChromaDB in ~/Library/Application Support/Curator/ with 700 permissions. Never log query text. No network endpoint.

## 🚪 New Doors Opened
1. **ColPali (ICLR 2025)** — embeds PDF *page images* directly, no OCR. For a file organizer handling academic PDFs and scanned docs, this could be a major quality improvement AND a research contribution (first application of ColPali to personal file retrieval).
2. **Rewind AI was acquired by Meta (Dec 2025) and shut down** — the most privacy-conscious screen/file indexer is GONE. This leaves a vacuum. Curator could position itself as the privacy-first successor with explicit research backing.
3. **ClusterRAG (arXiv:2605.18769, 2026)** — published AFTER we designed our architecture. They cluster users, we cluster files — same algorithmic idea, different axis. We can frame our approach as a novel application of their insight to the file domain.
4. **No benchmark for personal file retrieval exists** — HippoCamp (arXiv:2604.01221) covers agent tasks but not retrieval precision. Creating a personal file retrieval benchmark (diverse types, vague NL queries, ground truth file pointers) would be a dataset paper contribution for SIGIR/ECIR.
