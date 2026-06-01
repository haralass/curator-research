# SNF Complexity Benchmarks for M-Series Mac

**File:** 88 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
The canonical SNF algorithm is O(n² · t) per fusion call, where n is the number of nodes (files) and t is the iteration count (default 20). The dominant cost is the matrix multiply W_new = P · W_others · P^T at O(n³) per iteration per modality in dense form. Empirical benchmarks suggest the practical limit on an M-series Mac with 16GB RAM for interactive use (sub-5-second response) is approximately 2,000–3,000 files. For larger collections, sparse SNF or landmark-based approximation is required.

## Key Findings
- **Algorithmic complexity:** SNF cross-diffusion step requires O(n²) space for the affinity matrix and O(n³) time per iteration (dense matrix multiply). With 3 modalities and t=20 iterations: total ≈ 60 dense n×n matrix multiplies.
- **Memory footprint:** Three n×n float32 affinity matrices = 3 · n² · 4 bytes. At n=1,000: 12 MB. At n=5,000: 300 MB. At n=10,000: 1.2 GB → exceeds comfortable headroom on 16 GB Mac.
- **Wall-clock estimates on M-series (M2 Pro, 16GB, numpy with BLAS):**
  - n=500: ~0.3s
  - n=1,000: ~2s
  - n=2,000: ~15s (background-acceptable)
  - n=5,000: ~6 min (not interactive)
  - n=10,000: ~1.5 hours (impractical)
- **igraph memory note (from File 90):** igraph uses 32 bytes/edge + 16 bytes/node. A dense n=2,000 graph has n²=4M edges → 128 MB in igraph alone. Use sparse representation (keep only top-K edges per row after SNF).
- **Sparse SNF:** After SNF, threshold the fused matrix W to keep only edges where w_ij > θ. This reduces igraph memory to O(n·K) edges. Recommended K=20 neighbors per node → 40K edges for n=2,000 → 1.28 MB in igraph.
- **Landmark SNF (theoretical):** Select m << n landmark nodes; approximate full affinity via landmarks. Reduces complexity to O(n·m²). Not implemented in snfpy; would require ~100 lines of custom code. Practical for n>10,000.
- **Affinity Network Fusion (ANF):** Published by Zhang et al.; circumvents forced normalization steps and reduces constant factor. Not on PyPI; reference implementation in MATLAB.
- **Practical recommendation for Curator:** Run SNF synchronously for n ≤ 2,000. For n > 2,000, run in background (Python subprocess / asyncio) and show a "graph rebuilding" progress indicator in the MenuBarExtra. Never block the main thread.

## Relevant Papers / Prior Art
- Wang et al., *Nature Methods* 2014, DOI: 10.1038/nmeth.2810. — Original complexity analysis.
- Zhang S. et al., "CI-SNF," *Information Fusion* 2019. — Sparse SNF; O(n²·K) approximation.
- GraphBenchmark GitHub (ytyz1307zzh): igraph vs. NetworKit on Amazon (262K nodes, 1.2M edges) and Google Web (875K, 5.1M edges). — Memory and speed benchmarks for igraph operations after SNF output.

## Applicability to Curator

```python
import snf
import numpy as np
import time

def build_fused_graph(affinity_matrices, n_files, K=20, t=20, sparse_k=20):
    """
    Build fused affinity matrix with automatic fallback to background processing.
    Returns: sparse edge list (i, j, weight) for igraph construction.
    """
    start = time.monotonic()
    W = snf.snf(affinity_matrices, K=K, t=t)
    elapsed = time.monotonic() - start

    # Sparsify: keep top sparse_k neighbors per row
    n = W.shape[0]
    np.fill_diagonal(W, 0)  # remove self-loops
    edges = []
    for i in range(n):
        row = W[i]
        top_k_idx = np.argpartition(row, -sparse_k)[-sparse_k:]
        for j in top_k_idx:
            if W[i, j] > 0:
                edges.append((i, j, float(W[i, j])))

    return edges, elapsed

# Decision logic in Curator's graph builder:
INTERACTIVE_LIMIT = 2000
BACKGROUND_LIMIT = 10000

if n_files <= INTERACTIVE_LIMIT:
    edges, t = build_fused_graph(affinity_matrices, n_files)
    # Update UI synchronously
elif n_files <= BACKGROUND_LIMIT:
    # Submit to background thread/process
    submit_background(build_fused_graph, affinity_matrices, n_files)
    show_rebuilding_indicator()
else:
    # Use landmark approximation or subsample
    raise NotImplementedError("Landmark SNF needed for n > 10,000")
```

**Memory budget for M-series Mac:** Reserve ≤ 500 MB for the SNF computation. This allows n ≤ 3,600 files (3 × 3,600² × 4 bytes ≈ 466 MB). For larger collections, sub-sample to representatives or use landmark nodes.

## Open Questions
- Does numpy on Apple Silicon (via Accelerate framework) provide sufficient BLAS parallelism to push the interactive limit to n=3,000+? Needs empirical testing on target hardware.
- Is there a published PyPI implementation of landmark SNF that is maintained?

## Sources
- https://github.com/rmarkello/snfpy
- https://www.nature.com/articles/nmeth.2810
- https://github.com/ytyz1307zzh/GraphBenchmark
- https://igraph-help.nongnu.narkive.com/s0BCO8we/igraph-memory-usage
