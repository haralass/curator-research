# 58 — DenStream Streaming Clusterer: Confirmed Bugs, Forks, and Safe Alternatives
_Research date: 2026-05-31_

---

## DenStream Algorithm Overview

DenStream (Cao et al., SIAM SDM 2006) clusters continuous data streams without storing raw data. It maintains compact micro-cluster summaries in two categories: **potential micro-clusters** (p-MCs, stable density regions) and **outlier micro-clusters** (o-MCs, sparse or transient regions). An exponential fading function down-weights older observations for concept drift adaptation. Final macro-clusters are extracted on-demand via a DBSCAN pass over p-MC centers.

For Curator's use case — files arriving asynchronously, embeddings clustering incrementally without reprocessing the full corpus — DenStream's `learn_one` interface is architecturally correct. The problem is that every known Python implementation has correctness issues.

---

## River Library Implementation: Confirmed Bugs

Source: `river/river/cluster/denstream.py` (https://github.com/online-ml/river/blob/main/river/cluster/denstream.py) and GitHub Discussion #1555 (https://github.com/online-ml/river/discussions/1555).

River's current version as of research date: **0.24.2** (released April 15, 2026). None of the bugs below are fixed in any River release changelog.

### Bug 1 — Weighted-Sum Decay (Critical)

The `DenStreamMicroCluster` class maintains running `linear_sum` and `squared_sum` accumulators and applies the fading factor to the entire accumulated sum at query time. The paper requires each historical point's contribution to be independently weighted by `f(t - t_i) = 2^(-λ(t - t_i))` before being summed. River's approach assumes all past points arrived at the same time as the most recently edited point — producing wrong center positions, wrong radii, and wrong cluster weights throughout.

### Bug 2 — Shallow Copy in Merge Operation (Critical)

In `_merge()`, when testing whether a new point fits into an existing micro-cluster, River makes a shallow copy of the target micro-cluster, adds the point to the copy, and checks the radius. Because the copy is shallow, the `linear_sum` and `squared_sum` dictionaries are shared objects. Mutating the copy mutates the original — cluster statistics are permanently corrupted even when the point is officially rejected.

### Bug 3 — Dictionary Index Corruption on Deletion

Micro-clusters are stored in a dict keyed by integer indices. When a cluster is deleted (weight drops below threshold), remaining keys are not renumbered. Subsequent insertions can overwrite still-valid clusters at the vacated index.

### Bug 4 — Wrong Initialization Threshold

When the initial offline DBSCAN pass promotes o-MCs to p-MCs, the implementation uses μ as the weight threshold. The paper (Section 4, Definition 4.4) requires **βμ**. Since β ≤ 1, using μ is too permissive — inflating the p-MC count and including noise in the final clustering.

### Bug 5 — BFS Labeling Error in `predict_one`

The BFS used to assign macro-cluster labels visits neighbors and adds them to the queue regardless of whether they are already labeled. The correct algorithm should only enqueue unlabeled neighbors. Already-labeled nodes are revisited, causing incorrect label propagation.

### Bug 6 — Missing ε-Neighborhood Check in Final Assignment

After DBSCAN-style labeling of core micro-clusters, border-point micro-clusters are assigned to the nearest cluster without checking whether they are within ε distance. Points more than ε away from all core clusters should be labeled -1 (noise). River assigns them to the nearest cluster unconditionally.

### Bug 7 — Overlapping Micro-Cluster Deletion Not Implemented

The paper (Section 5) describes merging/deleting micro-clusters when they become spatially identical. River does not implement this — allowing redundant near-duplicate micro-clusters to accumulate indefinitely.

### Bug 8 — Beta Parameter Range Violation

Issue #874: The paper constrains β to (0, 1]. River's **default is `beta=5`** and examples show `beta=1.01`, both violating the constraint. The internal threshold βμ (Equation 4.1) separates potential clusters from outliers; when β > 1, this boundary is semantically invalid.

---

## The Weighted-Sum Decay Bug (Critical) — Detailed

The paper defines the weighted linear sum at time t as:
```
CF1(t) = Σ_{x_i ∈ C} f(t - t_i) · x_i
```
where `f(t - t_i) = 2^(-λ(t - t_i))` and t_i is each individual point's insertion time.

River instead accumulates raw sums and multiplies the entire sum by `2^(-λ(t - last_edit_time))` at query time. This is equivalent to assuming every point was inserted at `last_edit_time`.

**Practical consequences for Curator:**
- Cluster centers drift toward recently-added points far more aggressively than intended
- Cluster radii are wrong → membership tests fail
- Cluster weights are wrong → legitimate clusters pruned prematurely; stale outlier clusters retained
- Net effect: the algorithm fails to form stable clusters on any stream longer than a few dozen points (confirmed in Issue #1466: users report 0 clusters returned on textbook examples)

**The correct implementation:**
```python
# Correct: apply decay at insertion time
decay_at_insert = 2 ** (-lambd * current_time)
self.linear_sum[key] += decay_at_insert * x[key]
```
The `issamemari/DenStream` fork uses exactly this approach.

---

## Safe Forks

### issamemari/DenStream

- **URL:** https://github.com/issamemari/DenStream
- **Stars:** ~42 | **Last commit:** 2020 (core algorithm: 2018)
- **Maintenance:** Effectively unmaintained since 2018. No CI, no releases.

**What it fixes:** The `MicroCluster` class applies decay correctly per-point via `insert_sample(sample, weight)`. The decay factor `2^(-λ)` is applied to existing `sum_of_weights` before adding the new weight — correctly implementing the fading window. Mean and variance update incrementally using weighted formulas.

Does NOT fix Bugs 3, 4, 5, 6, or 7.

scikit-learn-compatible API (`partial_fit`, `fit_predict`, returns -1 for noise).

```python
from DenStream import DenStream
# pip install git+https://github.com/issamemari/DenStream.git

model = DenStream(lambd=1, eps=1, beta=2, mu=2)
model.partial_fit(X_batch)           # incremental
labels = model.fit_predict(X)        # -1 = noise
```

**Verdict:** Better than River for center/radius correctness, but unmaintained for 6+ years and only partially correct. Suitable as a base with additional patches.

---

### waylongo/denstream

- **URL:** https://github.com/waylongo/denstream
- **Stars:** 12 | **Last commit:** Single commit, abandoned

Primarily a Jupyter notebook (76% of codebase). No scikit-learn compatibility, no tests, no packaging. No evidence it fixes any confirmed bugs. **Not suitable for production use.**

---

### MrParosk/pyDenStream ✅ Best Current Option

- **URL:** https://github.com/MrParosk/pyDenStream
- **PyPI:** `pip install pydenstream`
- **Stars:** active | **Last release:** v0.1.4, June 24, 2025 | **CI/CD:** Yes | **Open issues:** 0

The most recently maintained standalone implementation. Has CI/CD, proper packaging, and an active test suite — unlike all other options. Does not explicitly advertise fixing River's bugs, but its active development and test coverage make it the safest choice for production.

---

## DBSTREAM as Alternative

River's `river.cluster.DBSTREAM` (Hahsler & Dunham, 2016) maintains shared-density estimates between micro-clusters rather than fading-window statistics.

**Known bugs in River's DBSTREAM:**

1. **Shallow copy in `_recluster` (Issue #1265):** `_generate_clusters_from_labels` creates macro-clusters as shallow copies of micro-clusters. Subsequent merges corrupt original micro-cluster weights/centers. The number of macro-clusters always equals the number of micro-clusters — merging never works.

2. **Performance degradation (Issue #1086):** Cluster count grows unboundedly on long streams. `learn_one` progressively slows.

3. **KeyError on sparse data (Issue #779):** When a data point is missing a key that exists in an existing micro-cluster center, the center update throws `KeyError`.

4. **`predict_one` reruns `_recluster` on every call:** Even when no new data has been learned, `predict_one` triggers the full offline reclustering pass — O(n_microclusters²) on every query. For Curator with thousands of file embeddings, this is prohibitive.

**DBSTREAM verdict:** Not a safe drop-in alternative given its own shallow-copy reclustering bug and predict-on-every-call performance issue.

---

## Recommendation for Curator

### Primary recommendation: `pydenstream` (MrParosk/pyDenStream)

```bash
pip install pydenstream
```

Active maintenance, CI/CD, proper packaging, 0 open issues. Evaluate for correctness against the issamemari implementation before committing.

### If pydenstream has issues: issamemari + 3 patches

```bash
pip install git+https://github.com/issamemari/DenStream.git
```

Apply these three patches:
1. **Deep copy in merge** (Bug 2): Replace `copy.copy(mc)` with `copy.deepcopy(mc)` in the merge/radius-test path
2. **Beta parameter validation** (Bug 8): Add `assert 0 < beta <= 1` in `__init__`; change default to `beta=0.5, mu=3`
3. **BFS labeling** (Bug 5): Only enqueue unlabeled neighbors in the final cluster extraction BFS

### Usage for Curator (high-dimensional embeddings, 384-dim)

```python
from DenStream import DenStream
import numpy as np

# Parameter guide for Curator:
# eps: ~median pairwise cosine distance in embedding space (try 0.3–0.8 for normalized)
# lambd: lower = longer memory (0.1–0.5 for slow-changing file collections)
# beta: keep ≤ 1 per paper constraint
# mu: minimum points to form a cluster (2–5 for small corpora)
model = DenStream(lambd=0.2, eps=0.5, beta=0.5, mu=2)

# Incremental: called each time a new file embedding arrives
model.partial_fit(embedding.reshape(1, -1))

# At cluster-request time (e.g., sidebar refresh):
all_embeddings = np.vstack(embedding_store)
labels = model.fit_predict(all_embeddings)
# labels[i] == -1 means noise (unclustered file)
```

### Fallback: Incremental HDBSCAN

If streaming clustering proves too unreliable for production, a practical fallback:
- Buffer new files until buffer exceeds threshold (e.g., 50 new files)
- Run full HDBSCAN re-clustering on the complete corpus
- This loses true online behavior but is far more reliable

```python
from hdbscan import HDBSCAN

def recluster(embeddings: np.ndarray, min_cluster_size: int = 3):
    clusterer = HDBSCAN(
        min_cluster_size=min_cluster_size,
        min_samples=1,
        metric="cosine",
    )
    return clusterer.fit_predict(embeddings)
```

---

## Key References

- River DenStream source: https://github.com/online-ml/river/blob/main/river/cluster/denstream.py
- River Discussion #1555 (7 confirmed bugs): https://github.com/online-ml/river/discussions/1555
- River Issue #1466 (0 clusters returned): https://github.com/online-ml/river/issues/1466
- River Issue #874 (beta parameter violation): https://github.com/online-ml/river/issues/874
- River DBSTREAM Issue #1265 (shallow copy): https://github.com/online-ml/river/issues/1265
- River DBSTREAM Issue #1086 (performance): https://github.com/online-ml/river/issues/1086
- issamemari/DenStream: https://github.com/issamemari/DenStream
- MrParosk/pyDenStream: https://github.com/MrParosk/pyDenStream
- waylongo/denstream: https://github.com/waylongo/denstream
- Cao, F. et al. (2006). Density-Based Clustering over an Evolving Data Stream with Noise. SIAM SDM 2006.
- Hahsler, M. & Dunham, M. (2016). Clustering Data Streams Based on Shared Density Between Micro-Clusters. IEEE TKDE.
- River PyPI (v0.24.2): https://pypi.org/project/river/
