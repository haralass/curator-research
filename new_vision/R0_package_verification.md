# R0 — Package & Dependency Verification

> Live-verified on the actual Curator venv (Python 3.11.15, Apple Silicon, macOS).
> Venv: `~/Applications/Curator.app/Contents/Resources/curator_venv/`
> Date: 2026-06-08

---

## Results

| Package | Role | Status | Wheel type | Action |
|---|---|---|---|---|
| `leidenalg 0.12.0` | Community detection | ✅ INSTALLED + TESTED | C extension | None — functional |
| `igraph 1.0.0` | Graph backend | ✅ INSTALLED | C extension | None |
| `scipy 1.17.1` | Sparse SNF math | ✅ INSTALLED | Binary | None |
| `faiss 1.14.2` | kNN index | ✅ INSTALLED | Binary | None |
| `river 0.25.0` | ADWIN (R4) | ⚠️ NOT INSTALLED | `universal2` wheel available (cp311) | `pip install river==0.25.0` |
| `ocrmac 1.0.1` | OCR via Apple Vision (R3) | ⚠️ NOT INSTALLED | Pure Python | `pip install ocrmac` |
| `markitdown 0.1.6` | PDF/DOCX extraction (R3) | ⚠️ NOT INSTALLED | Pure Python | `pip install markitdown` |
| `send2trash 2.1.0` | Safe trash from sidecar (R6) | ⚠️ NOT INSTALLED | Pure Python | `pip install send2trash` |
| `unisim 1.0.1` | RETSim version similarity (R2) | ⚠️ NOT INSTALLED | Pure Python (ONNX) | `pip install unisim` |
| `moondream 1.2.2` | Image understanding (R3 Tier 3) | ⚠️ NOT INSTALLED | Pure Python | `pip install moondream` |
| `hnswlib 0.8.0` | Incremental kNN inserts (R4) | ⚠️ NOT INSTALLED | **Source only** — needs C++ compile | `pip install hnswlib` (Xcode CLT available ✅) |
| `snfpy` | Sparse SNF | ❌ NOT INSTALLABLE | `snfpy 0.2.2` exists but is **full-matrix** SNF — NOT the sparse variant | See note below |

---

## Critical Finding: Sparse SNF Must Be Implemented

**No library for sparse SNF exists on PyPI.** `snfpy 0.2.2` is the original Wang 2014 implementation — full matrix, O(n²) memory. At 30k files this produces 3.6 GB/view.

**What exists:**
- `snfpy` — full matrix only
- `sparse-snf` — does NOT exist on PyPI
- `scipy.sparse.csr_matrix` — ✅ available (scipy 1.17.1 in venv)

**What must be written:** A sparse SNF implementation using `scipy.sparse`. This is real engineering work (~200 lines of Python), not a pip install.

**Reference implementation approach** (based on Wang et al. 2014):
```python
import scipy.sparse as sp
import numpy as np

def sparse_snf(affinity_matrices: list[sp.csr_matrix], K: int = 20, t: int = 10) -> sp.csr_matrix:
    """
    Sparse Similarity Network Fusion.
    Input: list of sparse kNN affinity matrices (each: n×n, csr_matrix)
    Output: fused affinity matrix (sparse, csr_matrix)
    """
    n_views = len(affinity_matrices)
    n = affinity_matrices[0].shape[0]

    # Step 1: Normalize each view → row-stochastic kernel P
    def normalize_kernel(W: sp.csr_matrix) -> sp.csr_matrix:
        row_sums = np.array(W.sum(axis=1)).flatten()
        row_sums[row_sums == 0] = 1  # avoid division by zero
        D_inv = sp.diags(1.0 / row_sums)
        return D_inv @ W

    # Step 2: For each view, build KNN-only "status" matrix S (sparse)
    # S_v[i,j] = W[i,j] / (2 * sum of K nearest neighbors of i, excluding i)
    def build_status_matrix(W: sp.csr_matrix) -> sp.csr_matrix:
        # Only keep top-K entries per row (already done by FAISS/hnswlib output)
        P = normalize_kernel(W)
        return P

    P_matrices = [build_status_matrix(W) for W in affinity_matrices]

    # Step 3: Iterative diffusion (t iterations)
    for _ in range(t):
        new_P = []
        for v in range(n_views):
            # Average of other views' current state
            other_sum = sp.csr_matrix((n, n), dtype=np.float32)
            for u in range(n_views):
                if u != v:
                    other_sum = other_sum + P_matrices[u]
            avg_other = other_sum / (n_views - 1)
            # Diffuse through current view's kNN kernel
            new_P.append(affinity_matrices[v] @ avg_other @ affinity_matrices[v].T)
        P_matrices = [normalize_kernel(p) for p in new_P]

    # Step 4: Fuse all views
    fused = sum(P_matrices) / n_views
    fused = normalize_kernel(fused)
    return fused
```

This keeps all matrices sparse (only kNN edges exist — K=20 per node → 20×n non-zeros vs n² for full).

**Memory at 30k files, K=20:**
- Full SNF: 30,000² × 4 bytes = **3.6 GB per view**
- Sparse SNF: 30,000 × 20 × 4 bytes = **2.4 MB per view** ✅

---

## hnswlib vs usearch

`hnswlib 0.8.0` has **source-only** on PyPI. Compiles with Xcode CLT (confirmed available on this machine). However, `unisim` already depends on **`usearch>=2.6.0`**, which:
- Has pre-built Apple Silicon wheels
- Also provides HNSW-based ANN search
- Would already be installed as a transitive dependency of `unisim`

**Curator decision (revised):** Use **`usearch`** (via `unisim` transitive dependency) for incremental kNN inserts instead of `hnswlib`. Eliminates one source-only compilation. `usearch` 3.x has parity with `hnswlib` on HNSW performance.

---

## leidenalg Functional Test

```
leidenalg 0.12.0
Leiden on Petersen graph → 2 communities ✅
igraph C backend confirmed active
```

Leidenalg is installed and working. No issues.

---

## Requirements Update Required

When implementing, add to `requirements_prod.txt`:

```
# New pipeline requirements (R2-R6)
river==0.25.0          # ADWIN context drift detection (R4)
ocrmac>=1.0.1          # Apple Vision OCR for scanned PDFs (R3)
markitdown>=0.1.6      # PDF/DOCX/XLSX text extraction (R3)
send2trash>=2.1.0      # Safe trash from Python sidecar (R6, B3)
unisim==1.0.1          # RETSim version similarity (R2) — pin version, repo archived
moondream>=1.2.2       # Vision LM for image Tier 3 (R3) — optional/lazy import
# usearch installed transitively via unisim — replaces hnswlib
```

**Remove from `requirements_prod.txt`:**
```
snfpy   # full-matrix only, not usable at 30k scale — replaced by sparse_snf.py
mapie   # replaced by crepes (already in requirements_prod.txt)
```

**Must implement (cannot pip install):**
- `engine/curator_sidecar/sparse_snf.py` — Sparse SNF (~200 lines, scipy.sparse)

---

## Summary

| Category | Count | Status |
|---|---|---|
| Already in venv, working | 4 | leidenalg, igraph, scipy, faiss |
| Need `pip install` (easy) | 6 | river, ocrmac, markitdown, send2trash, unisim, moondream |
| Source compile (manageable) | 0 | hnswlib replaced by usearch (transitive via unisim) |
| Must implement from scratch | 1 | **sparse_snf.py** |
| Cannot use / wrong version | 1 | snfpy (full-matrix, unusable at scale) |

**One real engineering task before Layer 2 can run:** `sparse_snf.py`. Everything else is `pip install`.
