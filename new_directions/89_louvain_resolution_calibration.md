# Louvain/Leiden Resolution Parameter Calibration for Personal Files

**File:** 89 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
The resolution parameter γ in Louvain/Leiden controls cluster granularity: γ < 1 produces fewer, larger clusters; γ > 1 produces more, smaller clusters; γ = 1 is the standard modularity default. For personal file collections, the target cluster size is typically 10–50 files per folder (based on cognitive load research), which suggests γ in the range 1.0–1.5 for most collections. The Leiden algorithm (which fixes Louvain's disconnected-community bug) is preferred and is available via `python-igraph` ≥ 0.10.

## Key Findings
- **Louvain vs. Leiden:** Leiden (Traag et al., *Scientific Reports* 2019) guarantees well-connected communities; Louvain can produce internally disconnected communities. Use Leiden for Curator. Available in python-igraph via `graph.community_leiden()`.
- **Resolution parameter effect:**
  - γ = 0.5: large, coarse clusters (few folders, many files each)
  - γ = 1.0: standard modularity; moderate cluster sizes
  - γ = 1.5: finer granularity; more, smaller clusters
  - γ = 2.0: very fine; risk of singleton clusters
- **Quality function options:** Leiden supports (1) modularity (default) and (2) Constant Potts Model (CPM). CPM with resolution γ finds communities where internal edge density > γ, making γ interpretable as a minimum density threshold. For Curator, CPM is recommended because it avoids the resolution limit of modularity.
- **Automatic resolution selection:** No fully automatic method exists. Standard practice: run Leiden at γ ∈ {0.5, 0.75, 1.0, 1.25, 1.5, 2.0}, select γ that minimizes |cluster_size - target_size|. For Curator's target of 10–30 files/folder, grid search at install time (< 1 second total) and store γ in user settings.
- **Multi-resolution hierarchy:** Run Leiden at multiple γ values to get a hierarchy: γ=0.5 → top-level folders, γ=1.5 → sub-folders. This enables a two-level folder hierarchy proposal (main folder → sub-folder) at low computational cost.
- **Natural cluster sizes for personal files:** Bergman et al. (2010) found users maintain 5–20 top-level folders and 10–50 files per folder. This matches γ ≈ 1.0–1.25 for typical collections.
- **Resolution limit of modularity:** Modularity cannot detect communities smaller than √(2m) nodes where m = number of edges. For sparse graphs (K=20 neighbors), this limit is small; CPM has no resolution limit.
- **Stability across runs:** Leiden is non-deterministic. Use `random_state` seed for reproducibility; compare two runs with Normalized Mutual Information (NMI) to detect instability.

## Relevant Papers / Prior Art
- Traag V.A., Waltman L., van Eck N.J., "From Louvain to Leiden: guaranteeing well-connected communities," *Scientific Reports* 9:5233, 2019. DOI: 10.1038/s41598-019-41695-z. — Defines Leiden algorithm and CPM quality function.
- Blondel V.D. et al., "Fast unfolding of communities in large networks," *J. Statistical Mechanics* 2008. — Original Louvain; 15-year retrospective at arXiv:2311.06047.
- Bergman O. et al., "The effect of folder structure on personal file navigation," *JASIST* 2010. DOI: 10.1002/asi.21415. — Empirical cluster size data for personal collections.
- Resolution limit: Fortunato S., Barthélemy M., "Resolution limit in community detection," *PNAS* 104(1):36–41, 2007. DOI: 10.1073/pnas.0605965104.

## Applicability to Curator

```python
import igraph as ig
import numpy as np

def calibrate_resolution(graph: ig.Graph, target_min=10, target_max=30) -> float:
    """Grid search for γ that produces clusters in [target_min, target_max] files."""
    best_gamma = 1.0
    best_score = float('inf')
    
    for gamma in [0.5, 0.75, 1.0, 1.25, 1.5, 2.0]:
        partition = graph.community_leiden(
            objective_function='CPM',
            resolution_parameter=gamma,
            n_iterations=10,
            weights='weight'
        )
        sizes = partition.sizes()
        median_size = np.median(sizes)
        # Penalize deviation from target range midpoint
        target_mid = (target_min + target_max) / 2
        score = abs(median_size - target_mid)
        if score < best_score:
            best_score = score
            best_gamma = gamma
    
    return best_gamma

def cluster_files(edges, n_files, gamma=None) -> list[int]:
    """Build igraph, run Leiden, return cluster label per file."""
    g = ig.Graph(n=n_files, directed=False)
    g.add_edges([(e[0], e[1]) for e in edges])
    g.es['weight'] = [e[2] for e in edges]
    
    if gamma is None:
        gamma = calibrate_resolution(g)
    
    partition = g.community_leiden(
        objective_function='CPM',
        resolution_parameter=gamma,
        n_iterations=10,
        weights='weight'
    )
    
    labels = [0] * n_files
    for cluster_id, members in enumerate(partition):
        for node in members:
            labels[node] = cluster_id
    return labels, gamma

# Two-level hierarchy
def hierarchical_clusters(edges, n_files):
    coarse_labels, _ = cluster_files(edges, n_files, gamma=0.5)   # top folders
    fine_labels, _   = cluster_files(edges, n_files, gamma=1.5)   # sub-folders
    return coarse_labels, fine_labels
```

**Store calibrated γ** in GRDB user_settings table. Re-calibrate when collection grows by >20% or when user explicitly adjusts folder granularity preference.

## Open Questions
- Is CPM's γ interpretable enough to expose to the user as a "folder size" slider? (CPM γ = minimum internal density → not intuitive; consider mapping it to a "few large folders ↔ many small folders" UI control instead.)

## Sources
- https://www.nature.com/articles/s41598-019-41695-z
- https://networkx.org/documentation/stable/reference/algorithms/generated/networkx.algorithms.community.louvain.louvain_communities.html
- https://arxiv.org/pdf/2311.06047
- https://igraph.org/r/html/1.3.4/cluster_leiden.html
