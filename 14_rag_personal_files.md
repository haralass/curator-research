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
- PDFs image-heavy: ColPali (embed page images, NO OCR) — arXiv:2407.01449 [Faysse et al. 2025]
- Images: moondream2 caption → embed caption + EXIF
- Code: AST chunking (function/class boundaries)
- Spreadsheets: Docling xlsx→md → row-group chunks
- ZIP: manifest doc + recursive

## Key Papers
- **ColPali** arXiv:2407.01449 (ICLR 2025) — visual PDF retrieval, bypass OCR entirely [Faysse et al. 2025]
- **ClusterRAG** arXiv:2605.18769 (2026) — HDBSCAN cluster-guided RAG, closest to Curator [ClusterRAG 2026]
- **CAR** arXiv:2511.14769 — adaptive cutoff via cluster structure, -60% tokens, +10% accuracy [CAR 2025]
- **PBR** arXiv:2510.08935 — query expansion with user history [PBR 2025]
- **Temporal RAG / TMRL** arXiv:2601.05549 — Matryoshka temporal embeddings [TMRL 2026]

## Apple Silicon Stack
Ollama 0.19+ (MLX backend): prefill +57%, decode +93% vs llama.cpp.
Full stack (Llama 3.1 8B + BGE-M3 + moondream2 + bge-reranker) fits in 16GB unified memory at Q4_K_M.

## Privacy
Fully local → main risk is local privilege escalation and embedding inversion.
Mitigations: ChromaDB in ~/Library/Application Support/Curator/ with 700 permissions. Never log query text. No network endpoint.

## References

- Faysse, M., Sibille, H., Wu, T., Omrani, B., Viaud, G., Hudelot, C., & Colombo, P. (2025). "ColPali: Efficient Document Retrieval with Vision Language Models." *Proceedings of ICLR 2025*. arXiv:2407.01449.
- ClusterRAG. (2026). "HDBSCAN Cluster-Guided Retrieval-Augmented Generation." arXiv:2605.18769.
- CAR. (2025). "Context-Adaptive Retrieval via Cluster Structure." arXiv:2511.14769.
- PBR. (2025). "Personalized Beyond Retrieval: Query Expansion with User History." arXiv:2510.08935.
- TMRL. (2026). "Temporal RAG with Matryoshka Retrieval Learning." arXiv:2601.05549.
- Campello, R. J. G. B., Moulavi, D., & Sander, J. (2013). "Density-Based Clustering Based on Hierarchical Density Estimates." *PAKDD 2013*, pp. 160–172. DOI: 10.1007/978-3-642-37456-2_14.
- Chen, J., Xiao, S., Zhang, P., Luo, K., Lian, D., & Liu, Z. (2024). "BGE M3-Embedding: Multi-Lingual, Multi-Functionality, Multi-Granularity Text Embeddings Through Self-Knowledge Distillation." arXiv:2402.03216.
