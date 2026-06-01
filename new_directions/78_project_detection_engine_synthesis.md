# Project Detection Engine: Synthesizing NER, Implicit Feedback, and Session Boundaries
**File:** 78 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Files 70, 72, and 76 each provide a distinct signal for inferring project membership: NER entity extraction provides semantic co-occurrence edges, FSEvents implicit feedback provides behavioral affinity edges, and conditional entropy H(L|I) provides session boundaries that scope co-access. Combining all three into a unified file affinity graph — where multiple edge types are weighted and fused — produces a system that requires no hardcoded categories and is demonstrably better than any single signal alone. The synthesis approach is well-supported by heterogeneous graph fusion literature and closely mirrors SNF (Similarity Network Fusion) applied to filesystem data.

## Key Findings

### Conceptual Architecture: The Multi-Layer File Affinity Graph
- Build a **weighted multigraph** G = (V, E) where nodes V are files and edges E come from three separate layers:
  - **Layer 1 — NER co-occurrence**: Two files share an edge if they both mention the same named entities (PERSON, ORG, PROJECT_NAME extracted via Apple NaturalLanguage / spaCy). Edge weight = Jaccard similarity of entity sets.
  - **Layer 2 — Behavioral co-access**: Two files share an edge if they appear in the same FSEvents session (bounded by H(L|I) boundary detection from File 76). Edge weight = normalized co-access frequency, dwell-time-adjusted (File 72).
  - **Layer 3 — Semantic embedding similarity**: Two files share an edge if their embedding cosine similarity exceeds a threshold. This layer is optional/auxiliary.
- Fuse the three layers into a single affinity matrix using **Similarity Network Fusion (SNF)** (Wang et al., 2014, Nature Methods). SNF iteratively updates and reinforces consistent structure across all layers while dampening noise unique to one layer.

### Why SNF Is the Right Fusion Algorithm Here
- SNF is **non-parametric and modality-agnostic**: it doesn't assume a shared feature space between NER edges and behavioral edges.
- Operates in O(n²) time — tractable for personal file collections of a few thousand files.
- Critically, SNF **amplifies signals that agree across layers** and **suppresses noise that only appears in one layer** — precisely what is needed when NER produces noisy entity extractions.
- Originally developed for genomics multi-omics integration, now broadly applied (clinical, recommendation systems). The analogy to filesystem signals is direct.

### Early Fusion vs Late Fusion vs Attention-Weighted
- **Early fusion** (concatenate all features before clustering): loses the structural independence of different signal types; NER noise contaminates behavioral signal.
- **Late fusion** (cluster independently on each layer, then merge cluster assignments via voting): discards complementary information when layers disagree — which is the most informative case.
- **SNF / attention-weighted intermediate fusion**: the recommended approach. GRAF (Graph Attention-aware Fusion Networks, PMC 2024) adds learned node-level attention weights so each edge type's contribution is adaptive per-node, not globally fixed.
- For a minimum viable version: **fixed-weight linear combination** of the three normalized adjacency matrices is sufficient to outperform any single signal (confirmed by Heterogeneous Information Crossing on Graphs for Session-Based Systems, ACM TWEB 2023).

### When NER Edges Hurt (Noise Conditions)
- **Generic entities**: Files mentioning "Apple", "Google", or common person names create false edges. Mitigation: filter entities below a per-corpus TF-IDF weight or restrict to low-frequency entities.
- **OCR/PDF extraction noise**: Garbled entity strings. Mitigation: minimum entity mention confidence threshold from NER model.
- **Cross-project homonyms**: Same person name appears in unrelated projects. Mitigation: require entity co-occurrence in ≥2 NER-extracted mentions before creating an edge.
- **Empty entity sets**: Binary/image files have no NER signal. For these nodes, rely entirely on behavioral layer. SNF handles missing modalities gracefully — a zero-row in one adjacency matrix simply contributes no signal from that layer.

### Prior Art: "Behavioral Entity Linking"
- No exact prior art under the name "behavioral entity linking" or "access-pattern-guided entity resolution" exists in the literature — this is genuinely novel framing for Curator.
- The closest work: **cross-document entity clustering** (Bagga & Baldwin, 1998; extended by Finkel & Manning, 2008) which refines entity clusters using document-cluster membership. File 70's NER + this co-clustering creates a "virtuous cycle" (entity refinement → better clusters → better entity resolution).
- **Heterogeneous session graphs** in recommendation systems (HICG, arxiv 2210.12940) use item co-occurrence within sessions + item attribute graphs — directly analogous to behavioral co-access + NER entity graphs.

### Clustering Step After Graph Construction
- Once the fused affinity matrix is built, apply **spectral clustering** with automatic k selection via eigengap heuristic, or **community detection** (Louvain algorithm) which is parameter-free and scales better.
- Louvain is preferred for Curator: no need to specify k, handles variable project counts per user naturally, runs in near-linear time.

### Existing macOS/Filesystem Tools Doing Something Similar
- **DEVONthink** (commercial): uses "AI classification" combining content similarity and user behavior (See Also & Classify feature). Proprietary, no published algorithm.
- **Tempi / Alfred / Recents**: purely behavioral, no NER.
- **Spotlight**: purely content/metadata, no behavioral signal.
- **iCloud Drive Optimize Storage**: automated file eviction based on access recency — a degenerate single-signal behavioral system that frequently causes files to disappear from Spotlight (documented user complaint: files evicted to iCloud not findable via local Spotlight).
- No known macOS tool combines NER entity graphs with FSEvents behavioral graphs. This is a genuine gap.

### Minimum Viable Combination
The simplest combination better than any single signal:
1. Build two adjacency matrices: A_ner (entity Jaccard) and A_behav (session co-access frequency, normalized).
2. Fuse: A_fused = 0.5 * A_ner + 0.5 * A_behav (uniform weights, no learning needed to start).
3. Run Louvain community detection on A_fused.
4. Label communities with their dominant entities from Layer 1.

This already beats pure content clustering because behavioral co-access captures context-free relationships (e.g., a PDF and a PNG opened together during a design session have no NER overlap but belong to the same project). It beats pure behavioral clustering because NER edges bridge files that were never co-accessed but semantically belong together.

## Relevant Papers / Prior Art

- Wang, B. et al. (2014). "Similarity network fusion for aggregating data types on a genomic scale." *Nature Methods*. https://www.nature.com/articles/nmeth.2810
- Agarwal, S. et al. (2024). "Graph-Convolutional Networks: Named Entity Recognition and LLM Embedding in Document Clustering." *arXiv:2412.14867*. https://arxiv.org/abs/2412.14867
- Zhang et al. (2023). "Heterogeneous Information Crossing on Graphs for Session-Based Recommender Systems." *ACM Transactions on the Web*. https://dl.acm.org/doi/10.1145/3572407
- Fan et al. (2024). "Fusing multiplex heterogeneous networks using graph attention-aware fusion networks (GRAF)." *PMC*. https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11586420/
- Ji et al. (2023). "Entity Co-occurrence Graph-Based Clustering for Twitter Event Detection." *ResearchGate*. https://www.researchgate.net/publication/385863128

## Applicability to Curator

1. **Implementation order**: Start with Layer 2 (behavioral co-access, already available from FSEvents + File 76 session boundaries) as the primary clustering signal. Add Layer 1 (NER) as a second layer once entity extraction is stable. Fuse with fixed weights initially.
2. **Cold start**: For new files with no behavioral history, rely entirely on NER edges. For binary files (images, binaries), rely entirely on behavioral edges. SNF handles the asymmetry.
3. **Community labels**: Each detected community is labeled by its top-N named entities (from Layer 1). This is how Curator surfaces "Project: CoolStartup pitch" without a hardcoded category taxonomy.
4. **Edge noise budget**: Set a minimum co-access count of 2 and a minimum entity Jaccard of 0.15 before adding an edge. Sparse graphs cluster more cleanly than dense noisy ones.
5. **Update cadence**: Rerun Louvain daily (or on-demand). SNF fusion is cheap once adjacency matrices are incrementally maintained.

## Open Questions

- How to handle the **temporal decay** of behavioral edges? A file pair co-accessed 6 months ago should contribute less than one co-accessed last week. Exponential decay weighting on behavioral edges needs empirical calibration.
- **Cross-project files** (e.g., a shared template used in all projects): these will have high degree in the behavioral graph and may absorb unrelated communities. Need a "hub node" detection and exclusion step.
- What is the right **minimum community size** before Curator surfaces a cluster as a candidate project? Likely 3–5 files as a practical floor.
- Does adding **Layer 3 (semantic embeddings)** meaningfully improve community quality over Layers 1+2 alone? Needs empirical evaluation on real user file collections.

## Sources
- https://arxiv.org/abs/2412.14867
- https://www.nature.com/articles/nmeth.2810
- https://dl.acm.org/doi/10.1145/3572407
- https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11586420/
- https://www.researchgate.net/publication/385863128_Entity_Co-occurrence_Graph-Based_Clustering_for_Twitter_Event_Detection
- https://www.emergentmind.com/topics/similarity-network-fusion-snf
- https://arxiv.org/pdf/2210.12940
