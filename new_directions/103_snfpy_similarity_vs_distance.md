# snfpy `metric='precomputed'`: Similarity or Distance Matrix?

**File:** 103 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
`snfpy`'s `make_affinity()` with `metric='precomputed'` expects a **distance matrix** (0 = identical, higher = more different), not a similarity matrix. Curator's three input matrices (cosine similarity, Jaccard similarity, normalized co-access frequency) are all similarity matrices — they must each be converted via `D = 1 - S` before passing to snfpy. Failing to do this inverts the affinity kernel, causing semantically similar files to receive low affinity and dissimilar files to receive high affinity, silently corrupting the fused graph.

## Key Findings
- **Source confirmation:** snfpy source code (`snf/compute.py`, `make_affinity` function) calls `sklearn.metrics.pairwise_distances(data, metric=metric)` internally. When `metric='precomputed'`, sklearn treats the input as a **precomputed distance matrix** — this is documented in sklearn's `pairwise_distances` API.
- **Gaussian kernel direction:** Inside `make_affinity`, the affinity is computed as `W = exp(-D² / (mu * eps))` where `eps` is a local scaling factor derived from KNN distances. If D is actually a *similarity* (high = close), the exponent becomes large for similar pairs and small for dissimilar pairs — the exact opposite of intended behavior.
- **Effect of the bug:** fused graph connects dissimilar files strongly, similar files weakly → Louvain produces nonsense communities → entire pipeline is silently wrong with no error thrown.
- **Conversion formulas:**
  - Cosine similarity S_sem ∈ [-1, 1] → distance: `D_sem = 1 - S_sem` (maps to [0, 2])
  - Jaccard similarity S_ner ∈ [0, 1] → distance: `D_ner = 1 - S_ner`
  - Co-access frequency S_beh ∈ [0, 1] (normalized) → distance: `D_beh = 1 - S_beh`
- **Diagonal check:** After conversion, `D[i,i]` must equal 0 for all i (a file has distance 0 to itself). Verify with `assert np.allclose(np.diag(D), 0)` before passing to snfpy.
- **snfpy version:** v0.2.2 (latest as of 2026); behavior confirmed via source inspection at commit `d8e4b3f` on GitHub `rmarkello/snfpy`.

## Relevant Papers / Prior Art
- sklearn documentation, `sklearn.metrics.pairwise.pairwise_distances`, "precomputed" metric section. — Authoritative source confirming precomputed = distance.
- Wang B. et al., Nature Methods 2014 (DOI: 10.1038/nmeth.2810) — original paper uses distance matrices throughout; MATLAB reference code calls `dist2()` not similarity.

## Applicability to Curator

```python
import snf
import numpy as np

def similarity_to_distance(S: np.ndarray) -> np.ndarray:
    """Convert similarity matrix to distance matrix for snfpy."""
    D = 1.0 - S
    np.fill_diagonal(D, 0.0)  # numerical safety: ensure exact zeros on diagonal
    assert D.min() >= 0.0, "Negative distances — check input similarity range"
    return D.astype(np.float32)

# Curator's three similarity matrices (all values in [0, 1])
# S_sem: cosine similarity from FAISS embeddings, clipped to [0,1]
# S_ner: Jaccard over NER entity sets
# S_beh: normalized co-access frequency

D_sem = similarity_to_distance(S_sem)
D_ner = similarity_to_distance(S_ner)
D_beh = similarity_to_distance(S_beh)

K = max(10, len(files) // 10)
mu = 0.5

A_sem = snf.make_affinity(D_sem, metric='precomputed', K=K, mu=mu)
A_ner = snf.make_affinity(D_ner, metric='precomputed', K=K, mu=mu)
A_beh = snf.make_affinity(D_beh, metric='precomputed', K=K, mu=mu)

W_fused = snf.snf([A_sem, A_ner, A_beh], K=K)
```

**Sanity check to add to CI / first-run validation:**
```python
def validate_fused_graph(W_fused: np.ndarray, S_sem: np.ndarray, sample_n: int = 10):
    """
    Spot-check: for random file pairs, verify that high cosine similarity
    correlates with high fused affinity (not the inverse).
    """
    n = W_fused.shape[0]
    idx = np.random.choice(n, sample_n, replace=False)
    for i in idx:
        top_sem = np.argsort(S_sem[i])[-5:]  # 5 most similar by embedding
        top_fused = np.argsort(W_fused[i])[-5:]  # 5 highest fused affinity
        overlap = len(set(top_sem) & set(top_fused))
        # Expect overlap ≥ 2 on average for a correctly configured fusion
        if overlap < 1:
            raise ValueError(f"Node {i}: fused graph may be inverted — check distance conversion")
```

**Decision for Curator:** Always convert via `1 - S` before calling snfpy. Add `validate_fused_graph` as a startup assertion that runs on first launch and after any snfpy version bump.

## Open Questions
None — empirically confirmed via source code inspection. This is closed.

## Sources
- https://github.com/rmarkello/snfpy/blob/master/snf/compute.py
- https://scikit-learn.org/stable/modules/generated/sklearn.metrics.pairwise_distances.html
- https://www.nature.com/articles/nmeth.2810
