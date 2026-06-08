# R14 — Spatially Sharded Candidate Graph & Fast Candidate Retrieval Without N²

> **Core principle: "Approximate to find candidates. Exact to decide."**
>
> R14 must make Curator scalable.
>
> Curator must never build a full all-pairs similarity matrix. It must never compare every file with every other file. As the corpus grows to 30k, 100k, or 1M files, the retrieval architecture must scale sub-linearly — while guaranteeing that no important neighbor is systematically missed, and that no approximate search result alone drives a final decision about a user's files.

---

## Why N² Is Unacceptable

A naïve similarity system computes pairwise distances between all files. At 30k files, that is 30,000² / 2 = 450 million comparisons. At 100k files, 5 billion. At 1M files, 500 billion.

Even at 1 microsecond per comparison (optimistic for 384-dim cosine distance), 30k × 30k = 450 seconds of continuous CPU. At 100k files, 83 minutes. This is not a background task — it is a thermal event.

N² is fundamentally unacceptable for a personal, local, thermal-aware tool. Curator must solve the candidate retrieval problem correctly from the start.

---

## 1. The Two-Stage Architecture

The correct architecture for Curator's similarity system is a strict two-stage pipeline, established in the systems literature as the standard pattern for scalable retrieval:

```
Stage 1: Approximate Candidate Retrieval
  Input:  file embedding (query vector)
  Output: top-K candidate neighbors (approximate, fast)
  Method: sharded vector index (HNSW / IVF / hybrid)
  Cost:   O(log n) or O(√n)
  Goal:   recall ≥ 0.90 at K=20

Stage 2: Exact Reranking
  Input:  top-K candidates from Stage 1
  Output: ranked, validated, context-checked list
  Method: exact distance + metadata filters + context boundary signals
  Cost:   O(K) — K is small (20–100), not n
  Goal:   eliminate false positives, enforce context guardrails
```

**The critical rule:** Stage 1 finds candidates. Stage 1 never makes decisions.

A file is never moved, grouped, flagged as duplicate, or assigned to a context based on approximate search alone. Every Stage 1 result passes through Stage 2 validation before any action is taken.

---

## 2. Prior Art

### 2.1 HNSW (Hierarchical Navigable Small Worlds)

**How it works:** Builds a multi-layer proximity graph. Each vector is a node connected to its approximate nearest neighbors. Search proceeds by greedy traversal from the top (coarse) layer down to the base (precise) layer. Time complexity: O(log n) per query. [Source: Michael Brenndoerfer — HNSW Index Architecture]

**Strengths:**
- High recall at low latency — typically 0.95–0.99 recall@10 with fast queries
- No training required (unlike IVF which requires k-means clustering)
- Incremental inserts without full rebuild
- Dominant algorithm in production ANN systems (used by usearch, FAISS, Milvus, Qdrant)

**Weaknesses:**
- Memory-intensive: the graph structure must be stored alongside the vectors. At 100k 384-dim vectors, this is significant.
- **"Unreachable node" problem:** after many deletions, some nodes may become disconnected from the graph, permanently reducing recall. Periodic graph repair is required for deletion-heavy workloads. [Source: Redis HNSW blog]
- Entire index typically loaded in RAM (unless using memory-mapped storage, as in usearch)

**For Curator:** HNSW is the primary candidate for Stage 1. usearch's HNSW implementation uses memory-mapped storage (index can exceed RAM), which mitigates the memory concern.

### 2.2 IVF (Inverted File Index)

**How it works:** Partitions the vector space into K clusters using k-means (or similar). Each vector is assigned to its nearest cluster centroid. At query time, only vectors in the top `nprobe` clusters are searched. Time complexity: O(K + n/K × nprobe). Optimal K ≈ √n. [Source: PingCAP — ANN Search Explained: IVF vs HNSW vs PQ]

**Strengths:**
- Memory efficient: overhead is just the centroids (small)
- Natural spatial partitioning — vectors in the same cluster are geometrically nearby
- Easily sharded: each cluster is an independent inverted list
- Recall tunable via `nprobe` (probe more clusters → higher recall, more cost)

**Weaknesses:**
- Requires a training phase (k-means on the corpus before indexing)
- Recall degrades on datasets with complex or uneven distributions — not all clusters are equally dense
- Less effective than HNSW for high-recall requirements (> 0.95) without probing many clusters

**For Curator:** IVF is the basis for **context-level sharding**. Each context group is a natural IVF cluster. Queries within a context search only that cluster's inverted list.

### 2.3 IVF-PQ (Inverted File + Product Quantization)

**How it works:** Combines IVF spatial partitioning with PQ vector compression. Vectors are stored in compressed form (2–8 bits per dimension subspace). Approximate distances computed from compressed codes, then an optional refinement step computes exact distances for top candidates. [Source: Pinecone — Product Quantization]

**Compression math (verified):**
- PQ splits a 384-dim vector into M subspaces, each quantized to 8 bits
- At M=48 subspaces: 48 bytes per vector (from 1,536 bytes float32) = **32x compression**
- At 100k vectors: 1,536 × 100k = ~147MB float32 → ~4.6MB PQ-compressed
- Combined IVF+PQ speedup vs brute force: up to 92x at competitive recall [Source: NVIDIA cuVS IVF-PQ]

**For Curator:** IVF-PQ is the candidate for large-scale (> 500k vectors) or memory-constrained situations. MVP uses HNSW (via usearch); IVF-PQ is a Phase 2 consideration.

### 2.4 LSH (Locality-Sensitive Hashing)

**How it works:** Hashes vectors so that similar vectors collide in the same hash bucket with high probability. Multiple hash tables increase recall at the cost of memory. [Source: Rohan Paul — LSH in Document Retrieval, 2024–2025]

**Comparison vs HNSW:**
- LSH: lower memory per table, but requires 20+ tables for high recall — total memory comparable to HNSW
- LSH: recall is generally lower than HNSW at the same latency budget
- LSH: insertion is O(1) per table — faster than HNSW graph updates
- HNSW: consistently outperforms LSH in recall-latency benchmarks on standard ANN datasets

**For Curator:** LSH is not recommended as the primary candidate retrieval method. HNSW (via usearch) offers better recall-latency trade-offs for Curator's 384-dim, 30k–1M vector range.

### 2.5 Product Quantization (PQ) — Standalone

**How it works:** Splits the vector into M equal subspaces. Each subspace is quantized independently to one of 256 centroids (8-bit). Distance computed as sum of subspace distances via lookup table. [Source: Pinecone — Product Quantization deep dive]

**Recent advance (ICML 2024 — QINCo):** At 12 bytes per vector, QINCo achieves higher recall than prior methods using 16 bytes — pushing PQ compression further without recall loss.

**For Curator:** PQ alone is a compression technique, not a candidate retrieval method. It is used inside IVF-PQ or TurboVec, not standalone. usearch's int8/fp16 quantization serves the same role for Curator's MVP.

---

## 3. Curator-Specific Sharding Strategy

### 3.1 The Natural Shard: Context Group

Curator's data has a natural spatial structure: **context groups**. Files about EPL342 Databases are semantically clustered — they are near each other in vector space almost by definition.

This means Curator does not need to search the entire vector index for every query. Most queries are context-scoped:
- "Find files similar to this PDF within the EPL342 context"
- "Find potential duplicates of this file within its context"
- "Detect context boundary: does this file belong here or in a neighboring group?"

Context-scoped search is already filtered search — and filtered HNSW with allowlists (as supported by usearch) is the correct implementation.

### 3.2 Two-Level Sharding

**Level 1 — Context Shard (metadata filter):**
- Each context group maps to an allowlist of `file_id` values
- Queries within a context pass `allowlist=context_file_ids` to the VectorIndex
- This restricts the HNSW search to only vectors in that context
- Effective cost: O(log |context|), not O(log n)

**Level 2 — Cross-Context Candidate Retrieval (boundary detection):**
- For context boundary detection (R4), Curator searches across context boundaries
- Restricted to contexts that are temporally or spatially nearby (same parent folder, same time window, overlapping course codes)
- This is a wider but still bounded search — not global

**Global search is only used for:**
- New file with no context signal (no course code, unknown folder, no metadata)
- Explicit user search ("find everything related to X")
- In both cases, the search returns candidates only — no automatic decisions

### 3.3 Exclusion Blocklist

Every vector query must apply a **blocklist** of excluded file IDs:
- Locked files (never entered Curator — but if somehow present, blocklisted)
- Ignored files (user explicitly excluded)
- `failed_unreadable` files (no valid vector exists)
- Files in `trashed` or `permanently_deleted` state

The blocklist is small (typically < 500 entries) and applied as an in-memory set filter before returning results.

---

## 4. Sparse Candidate Graph

### 4.1 What It Is

The **Sparse Candidate Graph** is the data structure that Curator builds and maintains for similarity-based operations (grouping, duplicate detection, context boundary detection).

It is NOT a full all-pairs graph. It is an **adjacency list** of (file_id, neighbor_file_id, similarity_score) triples, containing only the top-K neighbors for each file.

```sql
CREATE TABLE candidate_graph (
    file_id         INTEGER NOT NULL REFERENCES files(file_id),
    neighbor_id     INTEGER NOT NULL REFERENCES files(file_id),
    similarity      REAL NOT NULL,
    edge_type       TEXT NOT NULL, -- 'same_context' / 'cross_context' / 'near_duplicate'
    computed_at     INTEGER NOT NULL,
    PRIMARY KEY (file_id, neighbor_id)
);
CREATE INDEX idx_candidate_graph_neighbor ON candidate_graph (neighbor_id);
```

**Why SQLite, not in-memory?**
- The graph must survive restarts
- The graph can be queried by file_id (adjacency lookup) or by neighbor_id (reverse lookup)
- SQLite WAL mode keeps it consistent with the `files` table
- At 100k files × K=10 edges = 1M rows, this is a small table for SQLite

### 4.2 Graph Population

The graph is populated incrementally:
1. New file is embedded (Layer 3)
2. Stage 1: top-K candidates retrieved from VectorIndex (HNSW)
3. Stage 2: candidates validated against context signals and exact distance
4. Surviving edges written to `candidate_graph`

The graph is **not rebuilt from scratch**. New files are inserted incrementally. When a file is trashed or removed, its edges are deleted.

### 4.3 Top-K Size

K is not a fixed number. It depends on context:

| Situation | K | Reason |
|---|---|---|
| Within active context, small (< 100 files) | 20 | Dense neighborhood useful |
| Within active context, large (> 1k files) | 10 | Fewer edges needed for clustering |
| Cross-context boundary detection | 5 | Only strong cross-context edges matter |
| Near-duplicate detection | 3 | High similarity threshold, few candidates |

### 4.4 Edge Decay

Edges are not permanent. A neighbor that was relevant 6 months ago may no longer be relevant (different context, file moved, content changed). Edges with `computed_at` older than N days are candidates for refresh during Idle Completion Mode.

---

## 5. Recall: Measuring Missed Neighbors

### 5.1 The Recall Problem in Curator's Context

Approximate search misses some true neighbors — this is the fundamental trade-off. The question for Curator is: **which missed neighbors matter?**

A missed neighbor is harmful if:
- It would have caused a near-duplicate to be flagged (preventing the user from having two copies of the same file)
- It would have prevented a context boundary error (file grouped in the wrong context)

A missed neighbor is harmless if:
- It would have been the 11th most similar file, and Curator already found the top 10
- It belongs to a completely different context (cross-context miss for a within-context query)

### 5.2 Recall Target

Curator targets **recall@10 ≥ 0.90** for within-context queries. This means: of the 10 true nearest neighbors, Curator finds at least 9.

For near-duplicate detection (where missing a true duplicate is most harmful), the target is higher: **recall@5 ≥ 0.95**.

These targets are consistent with production ANN systems. At 0.90–0.99 recall, HNSW is orders of magnitude faster than brute force. [Source: Milvus ANN recall documentation]

### 5.3 Measuring Recall in Production

Curator cannot compute ground-truth recall at runtime (that would require brute force). Instead, it uses proxy signals:

**Recall proxy — neighbor count stability:**
If a file's top-K neighbors change significantly between two successive index queries (without the file or its neighbors changing), this indicates index degradation (e.g., the "unreachable node" problem in HNSW after many deletions).

**Recall proxy — duplicate detection consistency:**
If the R2 duplicate gate flags a file as near-duplicate of X, but X's K-neighbors don't include the first file, the graph is inconsistent. This indicates recall loss.

**Response:** Trigger an index health check. If degradation is confirmed, rebuild the HNSW index for the affected context shard. (Full index rebuild is acceptable per-context; global rebuild is not.)

---

## 6. Exact Reranking: The Stage 2 Gate

### 6.1 What Stage 2 Does

Stage 2 receives the top-K candidates from Stage 1 and applies:

1. **Exact cosine distance** — recomputed from full float32 vectors (not compressed)
2. **Metadata filters** — file type compatibility, date proximity, size ratio
3. **Context boundary signals** — does the candidate share the same course code, parent folder, or inferred context? (R4)
4. **State filters** — exclude `failed_unreadable`, `trashed`, `permanently_deleted`
5. **Confidence scoring** — combine distance + metadata + context into a final confidence score

### 6.2 Stage 2 Decision Rule

Stage 2 produces a ranked list of validated candidates. But Stage 2 itself does not make decisions. It produces candidates for:
- The duplicate detection gate (R2) — which has its own decision logic
- The context boundary algorithm (R4) — which has its own decision logic
- The Review Hub (R6) — if confidence is below threshold

The pipeline is:
```
VectorIndex (Stage 1) → validated candidates → R2 / R4 / R6 → decision
```

No step in this pipeline acts autonomously on user files. The final decision always goes through the appropriate research module's logic, and uncertain decisions always surface to the user via Review Hub.

### 6.3 Cross-Encoder Reranking (Phase 2)

For high-stakes decisions (context boundary detection, ambiguous near-duplicate flagging), Stage 2 can optionally use a cross-encoder model that jointly scores a (query, candidate) pair.

Cross-encoders are not suitable for Stage 1 (cost is O(n) per query — they cannot be the candidate generator). They are appropriate for Stage 2 because K is small (20–100 candidates). [Source: Digital Applied — Hybrid Search: BM25, Vector & Reranking 2026]

**Cross-encoder for Curator:**
- Only for Layer 5 (Temporary High-Precision) decisions
- Only during Calm or Asleep thermal state
- Candidate count: ≤ 50
- Model: small cross-encoder (< 100M params) — MiniLM-class

This is Phase 2. MVP Stage 2 uses exact cosine + metadata filters only.

---

## 7. Context Boundary Guardrails

### 7.1 The Missed Merge and Wrong Split Problem

The two failure modes for candidate-graph-based grouping:

**Missed merge:** Two files that should be in the same group are in different groups because their embedding similarity is below the threshold (recall loss or high-threshold filter).

**Wrong split / wrong merge:** Two files that should be in different groups are merged because their embeddings are similar but they belong to different contexts (e.g., EPL341 and EPL342 — similar content, different course).

Approximate search makes **missed merge** more likely. A high-recall system (recall@10 ≥ 0.90) limits missed merges to rare edge cases.

**Wrong merge** is not caused by recall loss — it is caused by threshold miscalibration. A file that is similar to files in a different context should be caught by Stage 2's context boundary signals (course code mismatch, folder mismatch, time mismatch).

### 7.2 Guardrails

For every candidate edge produced by Stage 2, a set of guardrails is checked before the edge is accepted:

| Guardrail | Check | On failure |
|---|---|---|
| Course code mismatch | If both files have course codes and they differ → reject edge | Discard edge |
| Folder distance | If file paths are more than N levels apart without common ancestor → reduce confidence | Lower confidence score |
| Temporal gap | If file creation dates differ by > 2 academic semesters → reduce confidence | Lower confidence score |
| File type incompatibility | PDF vs executable → reject edge | Discard edge |
| State mismatch | One file is `committed`, other is `new` → defer edge | Hold until both are stable |

A rejected edge is not deleted permanently. It is kept with `similarity = 0` as a negative example for future calibration.

### 7.3 Approximate Search Does Not Make Final Decisions

This is the inviolable rule:

> A file is never moved, grouped, flagged as duplicate, or assigned to a context based on approximate search results alone.

Stage 1 produces a ranked list. Stage 2 validates and filters. The downstream module (R2, R4, R6) makes the decision. If confidence is below threshold, the item goes to Review Hub for user resolution.

---

## 8. Scalability Path

| Scale | Vector index approach | Sharding | Notes |
|---|---|---|---|
| 0–30k files | HNSW (usearch, in-memory or mmap) | Context allowlists | MVP |
| 30k–200k files | HNSW (mmap) | Context allowlists + folder-level pre-filter | Phase 2 |
| 200k–1M files | IVF-PQ or HNSW-PQ | Context shards + cross-shard bridge | Phase 2–3 |
| 1M+ files | DiskANN or HNSW with disk tiering | Full spatial sharding | Phase 3+ |

At each scale transition, the VectorIndex abstraction (R11 §4.1) allows the backend to be swapped without changing the candidate graph or Stage 2 logic.

---

## 9. Implementation Checklist

- [ ] `candidate_graph` table: file_id, neighbor_id, similarity, edge_type, computed_at
- [ ] Stage 1: VectorIndex.search() with allowlist (context) and blocklist (excluded)
- [ ] Stage 2: exact cosine recomputation + metadata filters + context guardrails
- [ ] K selection logic per context size and query type
- [ ] Recall proxy monitoring: neighbor count stability + duplicate consistency
- [ ] Index health check trigger on detected degradation
- [ ] Context shard allowlist management (updated when files move contexts)
- [ ] Edge decay: scheduled refresh of edges older than N days during Idle Completion

---

## 10. Relationship to Other Documents

| Document | Relationship |
|---|---|
| R2_duplicate_version_detection.md | Stage 1+2 produces near-duplicate candidates for R2's decision logic |
| R4_context_graph_boundary.md | Stage 2 guardrails implement R4's boundary signals |
| R6_review_hub_staging.md | Low-confidence Stage 2 outputs route to Review Hub |
| R11_calm_vector_memory_thermal_indexing.md | VectorIndex abstraction is Stage 1's backend; thermal state governs when Stage 1 runs |
| R13_progressive_completeness_thermal_safe_scan.md | Graph population is part of Deep Understanding (Layer 3) — governed by completeness queue |
| TECH_engineering_foundation.md | `candidate_graph` table lives in the same SQLite DB |

---

## 11. Open Questions

| # | Question | Status |
|---|---|---|
| Q1 | Should `candidate_graph` be stored in SQLite or in the VectorIndex itself? | Prefer SQLite for ACID + WAL. Confirm no performance issue. |
| Q2 | What is the correct K for Leiden clustering input? (Does more edges = better clusters?) | Open — benchmark on Curator's corpus |
| Q3 | How often should edge decay refresh run? Daily? Weekly? | Open — depends on corpus churn rate |
| Q4 | Is HNSW "unreachable node" problem a real concern at Curator's deletion rate? | Open — estimate deletion rate from FSEvents logs |
| Q5 | Does usearch's HNSW implementation handle the unreachable node problem via periodic compaction? | Open — check usearch documentation |
| Q6 | For cross-context boundary detection, what is the correct nprobe / beam width? | Open — needs calibration on real corpus |

---

## Summary

- Curator never computes all-pairs similarity. N² is rejected architecturally.
- The two-stage pipeline — **approximate candidate retrieval → exact reranking** — is the standard pattern from production retrieval systems.
- **HNSW** (via usearch) is the primary Stage 1 backend: O(log n), high recall, incremental inserts, memory-mapped.
- **Context groups are natural shards**: most queries are context-scoped (allowlist-filtered HNSW), not global.
- The **Sparse Candidate Graph** (SQLite-backed adjacency list) persists top-K validated edges per file.
- **Stage 1 never makes decisions.** All decisions require Stage 2 validation + downstream module logic.
- Context boundary **guardrails** (course code, folder distance, temporal gap, file type) prevent wrong-merge errors that recall optimization alone cannot prevent.
- The system scales from 30k to 1M+ files via the VectorIndex abstraction, without changing the candidate graph or Stage 2 logic.
- Approximate to find candidates. Exact to decide.

---

## Sources

- [Michael Brenndoerfer — HNSW Index: Architecture for Fast Vector Search](https://mbrenndoerfer.com/writing/hnsw-index-vector-search-architecture)
- [Michael Brenndoerfer — IVF Index: Clustering-Based Vector Search & Partitioning](https://mbrenndoerfer.com/writing/ivf-index-clustering-vector-search-partitioning)
- [PingCAP — Approximate Nearest Neighbor (ANN) Search Explained: IVF vs HNSW vs PQ](https://www.pingcap.com/article/approximate-nearest-neighbor-ann-search-explained-ivf-vs-hnsw-vs-pq/)
- [Redis — How HNSW Algorithms Boost Search Performance](https://redis.io/blog/how-hnsw-algorithms-can-improve-search/)
- [Pinecone — Product Quantization: Compressing high-dimensional vectors by 97%](https://www.pinecone.io/learn/series/faiss/product-quantization/)
- [NVIDIA Technical Blog — Accelerating Vector Search: cuVS IVF-PQ Deep Dive](https://developer.nvidia.com/blog/accelerating-vector-search-nvidia-cuvs-ivf-pq-deep-dive-part-1/)
- [Rohan Paul — Locality-Sensitive Hashing in Document Retrieval: 2024–2025 Review](https://www.rohan-paul.com/p/locality-sensitive-hashing-in-document)
- [Milvus — Why are high recall values important in ANN search?](https://milvus.io/ai-quick-reference/why-are-high-recall-values-important-when-benchmarking-approximate-nearest-neighbor-searches-and-how-do-vector-databases-typically-beat-off-recall-for-speed)
- [Milvus — How do false positives and false negatives manifest in ANN search?](https://milvus.io/ai-quick-reference/how-do-false-positives-and-false-negatives-manifest-in-ann-search-results-and-how-do-they-relate-to-the-concepts-of-precision-and-recall-respectively-in-a-vector-search-evaluation)
- [Digital Applied — Hybrid Search: BM25, Vector & Reranking 2026](https://www.digitalapplied.com/blog/hybrid-search-bm25-vector-reranking-reference-2026)
- [arXiv — Unleashing Graph Partitioning for Large-Scale Nearest Neighbor Search](https://arxiv.org/pdf/2403.01797)
- [arXiv — Bang for the Buck: Vector Search on Cloud CPUs (2025)](https://arxiv.org/pdf/2505.07621)
- [QINCo — ICML 2024 (referenced via Pinecone PQ article)](https://www.pinecone.io/learn/series/faiss/product-quantization/)
