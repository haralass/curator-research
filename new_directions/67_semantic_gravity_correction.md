# Semantic Gravity Correction
**File:** 67 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Embedding models accumulate representational drift when file content changes, preprocessing pipelines shift, or model versions update — making previously-indexed embeddings incompatible with new ones. Curator should treat embedding vectors as versioned artifacts and detect drift by comparing cosine distance distributions week-over-week, triggering selective re-embedding rather than full re-indexing. For folders, "gravity correction" means periodically re-anchoring folder centroid vectors to the current file set so that suggested destinations don't degrade over time.

## Key Findings
- Mixing embeddings from two model/pipeline versions creates chaotic geometry — never blend without re-indexing
- Drift detection: compare embedding centroid shift per folder; threshold ~0.15 cosine distance indicates stale index
- Topic-enriched embeddings (TF-IDF + LSA + LDA layered over sentence embeddings) resist drift better than plain sentence embeddings because topic structure is more stable
- RAG literature treats this as a production reliability problem: metadata must track embedding model version, preprocessing hash, text checksum, chunking config
- Query drift compensation (arxiv 2506.00037) maintains compatibility with old corpus vectors using adapter layers — applicable to Curator's folder index without full rebuild

## Relevant Papers / Prior Art
- "Query Drift Compensation: Enabling Compatibility in Continual Learning of Retrieval Embedding Models" — arxiv 2506.00037
- "Still Fresh? Evaluating Temporal Drift in Retrieval Benchmarks" — arxiv 2603.04532
- "Topic-Enriched Embeddings to Improve Retrieval Precision in RAG Systems" — arxiv 2601.00891
- "Embedding Drift: The Quiet Killer of Retrieval Quality in RAG Systems" — Medium/DEV Community practitioner writeup

## Applicability to Curator
High. As users add/rename/delete files, folder centroid embeddings drift. Curator should track per-folder embedding freshness and queue folders for re-centroid computation when >20% of their contents have changed. The adapter-layer approach means Curator can update folder vectors cheaply without re-embedding every file. Schema: `embeddings` table should include `model_version`, `pipeline_hash`, `embedded_at` columns.

## Open Questions
- What drift threshold triggers a re-centroid vs. a full folder re-embed? (0.15 cosine distance is a starting point)
- Should Curator expose embedding version provenance to the user or keep it entirely internal?
- How to handle the cold-start case where a folder has <3 files — centroid is unstable by definition

## Sources
- https://arxiv.org/pdf/2506.00037
- https://arxiv.org/pdf/2603.04532
- https://arxiv.org/html/2601.00891v1
- https://medium.com/@anindyasinghobi/embedding-drift-the-quiet-killer-of-retrieval-quality-in-rag-systems-b5d46bee3bba
