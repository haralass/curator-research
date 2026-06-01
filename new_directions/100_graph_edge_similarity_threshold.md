# Graph Edge Similarity Threshold: Sensitivity Analysis and Adaptive θ

**File:** 100 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
The similarity threshold θ for including an edge in Curator's affinity graph critically determines cluster quality. Hard thresholding at a single global θ produces poorly connected nodes (isolated singletons) in sparse regions of the file collection. The evidence-based solution is **rank-based thresholding**: keep the top-K edges per row rather than applying a global cosine similarity cutoff. This guarantees connectivity (every node has at least K neighbors) and is adaptive to local density. K=15–20 is the recommended starting point, calibrated against the minimum community size constraint (File 99).

## Key Findings
- **Hard global threshold problems:** If θ=0.5 cosine similarity, nodes with no neighbors above θ become isolated singletons. In personal file collections with heterogeneous content (photos mixed with documents), many files will have cosine similarity < 0.5 to everything else. A global threshold always under-connects sparse regions and over-connects dense regions.
- **Percolation threshold:** Graph connectivity undergoes a phase transition at the percolation threshold. Below this threshold, the graph is fragmented (many disconnected components); above it, a giant connected component emerges. For Erdős-Rényi-style random graphs, the percolation threshold is at mean degree ≈ 1. For Curator's structured graphs, the practical threshold for useful community detection is mean degree ≥ 10.
- **Rank-based thresholding (recommended):** For each row i of the affinity matrix, keep only the top-K entries (K-nearest neighbors in similarity space). The resulting graph has exactly K edges per node (plus reciprocal edges, so degree ≈ K–2K). This approach is used in SNF itself (the KNN masking step) and in the PMC study on molecular similarity networks.
  - PMC:4812625 (Impact of similarity threshold on molecular similarity networks): "Rank-based thresholding with small d consistently produces connected networks even at low global similarity." K=15 produces stable community detection in heterogeneous networks.
- **Adaptive θ per file type:** Files of different types have systematically different embedding distributions:
  - Code files: high pairwise cosine (same vocabulary) → θ=0.6 may still leave many edges
  - Photos (CLIP embeddings): lower pairwise cosine for diverse content → θ=0.3 needed
  - PDFs (text embeddings): intermediate
  - **Solution:** Use rank-based K uniformly across types rather than per-type θ, since K adapts to local density automatically.
- **Sensitivity analysis:** PMC:4812625 shows that community structure is stable for K in [10, 30] for networks of 1,000–5,000 nodes. Below K=10, many singletons appear; above K=30, communities merge inappropriately.
- **After SNF:** SNF already applies KNN masking internally (the K parameter). After SNF outputs W_fused, a second sparsification step is needed to reduce the dense W_fused to a sparse igraph-compatible edge list. Use top-K per row again, K=20.
- **Explosive percolation:** arXiv:1504.00050 shows that in thresholded networks, the transition from disconnected to connected is abrupt (first-order phase transition). This means a small decrease in K or θ can suddenly fragment the graph. Keep K ≥ 15 as a hard floor.

## Relevant Papers / Prior Art
- PMC:4812625, "Impact of similarity threshold on the topology of molecular similarity networks and clustering outcomes," 2016. — Best empirical study of threshold sensitivity; recommends rank-based thresholding, K=15–20.
- arXiv:1504.00050, "Explosive percolation in thresholded networks," 2015. — Phase transition behavior; percolation threshold in similarity networks.
- Wang et al. (2014), *Nature Methods*, DOI: 10.1038/nmeth.2810. — SNF KNN masking (the internal K parameter in make_affinity).

## Applicability to Curator

```python
import numpy as np

def rank_based_threshold(W: np.ndarray, K: int = 20) -> np.ndarray:
    """
    Apply rank-based (KNN) thresholding to similarity matrix W.
    Keeps top-K entries per row; zeros out the rest.
    Produces a sparse, symmetric matrix for igraph input.
    
    Args:
        W: np.ndarray of shape (n, n), similarity values in [0, 1]
        K: number of neighbors to keep per node
    Returns:
        W_sparse: sparse affinity matrix (same shape, most entries 0)
    """
    n = W.shape[0]
    W_sparse = np.zeros_like(W)
    
    for i in range(n):
        row = W[i].copy()
        row[i] = 0  # exclude self-similarity
        top_k_idx = np.argpartition(row, -K)[-K:]
        W_sparse[i, top_k_idx] = row[top_k_idx]
    
    # Symmetrize: keep edge if either direction includes it
    W_sparse = np.maximum(W_sparse, W_sparse.T)
    return W_sparse

def compute_mean_degree(W_sparse: np.ndarray) -> float:
    """Diagnostic: mean node degree in the sparse graph."""
    return (W_sparse > 0).sum(axis=1).mean()

# Usage after SNF:
W_fused = snf.snf([A_sem, A_ner, A_beh], K=20)
W_sparse = rank_based_threshold(W_fused, K=20)

mean_deg = compute_mean_degree(W_sparse)
if mean_deg < 10:
    # Warning: graph may be too sparse for reliable community detection
    # Increase K or reduce θ
    W_sparse = rank_based_threshold(W_fused, K=30)

# Convert to igraph edge list
edges = []
for i in range(len(W_sparse)):
    for j in range(i+1, len(W_sparse)):
        if W_sparse[i, j] > 0:
            edges.append((i, j, float(W_sparse[i, j])))
```

**Calibration procedure (run once at install, re-run after major collection growth):**
```python
for K in [10, 15, 20, 25, 30]:
    W_s = rank_based_threshold(W_fused, K=K)
    # Run Leiden, check cluster size distribution
    labels, _ = cluster_files(to_edge_list(W_s), n_files, gamma=1.0)
    sizes = Counter(labels).values()
    singleton_rate = sum(1 for s in sizes if s < 5) / len(sizes)
    if singleton_rate < 0.05:  # < 5% singletons
        optimal_K = K
        break
```

## Open Questions
- Should the SNF internal K and the post-SNF sparsification K be the same value? (They don't have to be — SNF K controls the diffusion neighborhood; post-SNF K controls igraph connectivity. They can be tuned independently.)

## Sources
- https://pmc.ncbi.nlm.nih.gov/articles/PMC4812625/
- https://arxiv.org/pdf/1504.00050
- https://www.nature.com/articles/nmeth.2810
- https://www.emergentmind.com/topics/cosine-similarity-threshold
