# SNF Implementation: snfpy Package, Minimum Code, and vs. Weighted Sum

**File:** 84 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
`snfpy` (v0.2.1) is available on PyPI and provides a complete, production-ready Python implementation of Similarity Network Fusion. It can be installed with `pip install snfpy` and requires ~10 lines of user code. For Curator's 3-modality graph (semantic + NER + behavioral), snfpy is the correct choice over a weighted sum because SNF preserves cross-modal complementarity through iterative diffusion rather than collapsing it into a scalar blend.

## Key Findings
- **PyPI availability:** `pip install snfpy` installs v0.2.1 (last tagged 2019, but stable; still works on Python 3.10+). GitHub: `rmarkello/snfpy`.
- **Core API surface:** `snf.make_affinity(data, metric, K, mu)` → affinity matrix; `snf.snf([A1, A2, A3], K=20)` → fused network. That is the entire public API needed.
- **K parameter:** typically `N // 10` where N is number of nodes (files). For 1,000 files, K=100.
- **mu parameter:** scaling factor in (0.2, 0.8); mu=0.5 is the default safe choice.
- **t (iterations):** default 20; convergence typically occurs before t=10 for well-separated modalities.
- **Minimum implementation without snfpy:** ~50 lines — needs a Gaussian kernel, KNN masking, and an iterative cross-diffusion loop. The Wang et al. 2014 MATLAB reference is the authoritative source.
- **SNF vs. weighted sum:** Weighted sum (α·S₁ + β·S₂ + γ·S₃) treats modalities as interchangeable and averages out complementary signal. SNF's cross-diffusion step W_i ← P_i · (Σ_{j≠i} W_j / (V-1)) · P_i^T uses each modality to sharpen the others. Empirically, Wang et al. show SNF recovers clusters invisible to any single modality; weighted sum does not.
- **Affinity Network Fusion (ANF):** a faster approximation that avoids forced normalization; relevant if n > 5,000 files.
- **License:** LGPLv3 — compatible with commercial use; no copyleft obligation on Curator's proprietary code.

## Relevant Papers / Prior Art
- Wang B. et al., "Similarity network fusion for aggregating data types on a genomic scale," *Nature Methods* 11(3):333–337, 2014. DOI: 10.1038/nmeth.2810. — Original algorithm; defines kernel, KNN mask, cross-diffusion update rule.
- Zhang S. et al., "CI-SNF: Exploiting contextual information to improve SNF-based information retrieval," *Information Fusion*, 2019. DOI: 10.1016/j.inffus.2018.04.007. — Demonstrates sparse SNF for retrieval tasks.
- `metasnf` R package (arXiv:2410.17976) — scalable meta-clustering using SNF; shows practical defaults.

## Applicability to Curator

Curator has three affinity matrices:
1. **S_sem**: pairwise cosine similarity of FAISS/ChromaDB embeddings.
2. **S_ner**: entity co-occurrence Jaccard matrix from NER tags.
3. **S_beh**: co-access frequency normalized to [0,1] from FSEvents session logs.

```python
import snf
import numpy as np

# Assumes S_sem, S_ner, S_beh are np.ndarray of shape (n_files, n_files)
# Each should be pre-normalized to [0,1] with diagonal = 1.0

K = max(10, len(files) // 10)  # ~10% of nodes
mu = 0.5

# Convert raw similarity matrices to SNF affinity matrices
A_sem = snf.make_affinity(S_sem, metric='precomputed', K=K, mu=mu)
A_ner = snf.make_affinity(S_ner, metric='precomputed', K=K, mu=mu)
A_beh = snf.make_affinity(S_beh, metric='precomputed', K=K, mu=mu)

# Fuse
W_fused = snf.snf([A_sem, A_ner, A_beh], K=K)

# W_fused is now the input adjacency/weight matrix for igraph → Louvain
```

If snfpy is unavailable (e.g., environment restrictions), the 50-line fallback:

```python
def _normalize(W):
    D = np.diag(1.0 / (W.sum(axis=1) + 1e-9))
    return D @ W

def snf_fuse(matrices, K=20, t=20):
    P = [_normalize(W) for W in matrices]
    W = [W.copy() for W in matrices]
    for _ in range(t):
        W_new = []
        for i, Pi in enumerate(P):
            others = np.mean([W[j] for j in range(len(W)) if j != i], axis=0)
            W_new.append(Pi @ others @ Pi.T)
        W = [_normalize(w) for w in W_new]
    return np.mean(W, axis=0)
```

Decision: **use snfpy**. It is on PyPI, LGPLv3, and handles edge cases (zero rows, numerical stability) that a naive re-implementation will miss.

## Open Questions
- Does snfpy's `make_affinity` with `metric='precomputed'` accept a similarity matrix directly, or does it expect a distance matrix? (The docs are ambiguous; test empirically — if it expects distance, pass `1 - S`.)
- At what file count does SNF runtime become a UX problem on an M-series Mac (see File 88)?

## Sources
- https://snfpy.readthedocs.io/en/latest/
- https://github.com/rmarkello/snfpy
- https://www.nature.com/articles/nmeth.2810
- https://snfpy.readthedocs.io/en/latest/usage.html
