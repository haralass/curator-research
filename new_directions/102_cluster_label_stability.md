# Cluster Label Stability Across Louvain Re-Clusterings

**File:** 102 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Louvain community detection is non-deterministic: re-running on the same graph can assign different integer IDs to the same semantic community. This breaks Mondrian CP directly — calibration examples labeled with cluster ID `3` from Monday may refer to a completely different community on Tuesday after a nightly re-cluster. The fix is **Hungarian algorithm label matching** after each re-cluster, with a stability check to decide whether re-calibration is needed.

## Key Findings
- **Root cause:** Louvain uses random tie-breaking in node ordering. Same graph, different seed → different community ID assignment, even if community membership is identical.
- **igraph behavior:** `ig.community_louvain(weights=..., resolution=γ)` returns a `VertexClustering` object with `.membership` list. The integer labels in `.membership` are arbitrary and change between runs.
- **Magnitude of drift:** In practice, community *membership* is stable (same files cluster together) but *IDs* are permuted. On a graph with k communities, there are k! possible permutations — for k=20 that's ~2.4×10¹⁸. Naive comparison is useless.
- **Hungarian algorithm:** Also called the Munkres algorithm. Given two clusterings A and B (same nodes), computes the optimal bijective mapping between cluster IDs that maximizes overlap (measured by Jaccard or raw intersection). Runtime: O(k³) — negligible for k≤100.
- **scipy implementation:** `scipy.optimize.linear_sum_assignment` implements the Hungarian algorithm. Requires a cost matrix C where `C[i,j] = -|A_i ∩ B_j|` (negative intersection = cost to minimize).
- **Stability threshold:** If the best-match Jaccard for any cluster drops below 0.7, the community has genuinely split or merged — this is a semantic change, not just a label permutation. Trigger recalibration (see File 98).
- **Persistence strategy:** Store cluster identities as UUIDs in SQLite, not integers. After each re-cluster, run Hungarian matching, update the UUID→new_integer mapping table, and only invalidate calibration sets for clusters with Jaccard < 0.7.

## Relevant Papers / Prior Art
- Kuhn H.W., "The Hungarian Method for the Assignment Problem," *Naval Research Logistics Quarterly* 2(1-2):83–97, 1955. DOI: 10.1002/nav.3800020109. — Original algorithm.
- Lancichinetti A., Fortunato S., "Community detection algorithms: a comparative analysis," *Physical Review E* 80(5):056117, 2009. DOI: 10.1103/PhysRevE.80.056117. — Empirically shows Louvain membership stability vs. label instability; confirms Hungarian matching as the standard remedy.
- Vinh N.X. et al., "Information theoretic measures for clusterings comparison," *JMLR* 11:2837–2854, 2010. — Normalized Mutual Information (NMI) as an alternative stability metric to Jaccard.

## Applicability to Curator

```python
import numpy as np
from scipy.optimize import linear_sum_assignment
import igraph as ig

def match_clusterings(old_membership: list[int], new_membership: list[int], k: int) -> dict[int, int]:
    """
    Returns mapping: new_cluster_id → old_cluster_id (best match).
    Uses Hungarian algorithm on intersection matrix.
    """
    C = np.zeros((k, k), dtype=np.int32)
    for old_id, new_id in zip(old_membership, new_membership):
        C[new_id, old_id] += 1  # intersection count
    
    row_ind, col_ind = linear_sum_assignment(-C)  # maximize intersection
    return {new_id: old_id for new_id, old_id in zip(row_ind, col_ind)}

def jaccard(old_membership, new_membership, new_id, old_id):
    old_set = set(i for i, c in enumerate(old_membership) if c == old_id)
    new_set = set(i for i, c in enumerate(new_membership) if c == new_id)
    inter = len(old_set & new_set)
    union = len(old_set | new_set)
    return inter / union if union > 0 else 0.0

# After nightly re-cluster:
g = build_affinity_graph(files, W_fused)
new_clustering = g.community_louvain(weights='weight', resolution=gamma)
new_membership = new_clustering.membership
k = max(new_membership) + 1

mapping = match_clusterings(old_membership, new_membership, k)

STABILITY_THRESHOLD = 0.7
needs_recalibration = []
for new_id, old_id in mapping.items():
    j = jaccard(old_membership, new_membership, new_id, old_id)
    if j < STABILITY_THRESHOLD:
        needs_recalibration.append(new_id)

# Update SQLite: cluster_uuid table maps stable UUID → current integer ID
# Only invalidate Mondrian CP calibration for clusters in needs_recalibration
```

**SQLite schema addition:**
```sql
CREATE TABLE cluster_identity (
    uuid TEXT PRIMARY KEY,          -- stable identity across re-clusterings
    current_cluster_id INTEGER,     -- updated after each re-cluster
    created_at TEXT,
    last_matched_jaccard REAL       -- track stability over time
);
```

**Decision for Curator:**
- Store cluster UUIDs in `cluster_identity` table.
- After each re-cluster: run Hungarian matching, update `current_cluster_id`, flag clusters with Jaccard < 0.7 for recalibration.
- Mondrian CP calibration sets reference UUID, not integer ID — they survive re-clusterings with Jaccard ≥ 0.7.

## Open Questions
None — this is fully solvable with the above approach.

## Sources
- https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.linear_sum_assignment.html
- https://doi.org/10.1103/PhysRevE.80.056117
- https://igraph.org/python/tutorial/latest/generation.html#community-detection
