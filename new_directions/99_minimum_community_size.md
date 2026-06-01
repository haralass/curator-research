# Minimum Meaningful Community/Cluster Size for Personal File Folders

**File:** 99 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
The minimum meaningful cluster size for a personal file folder is approximately 5–7 files, based on cognitive load theory (Miller's 7±2), information foraging theory's patch profitability threshold, and empirical studies of personal folder organization. Clusters smaller than 5 files should be merged with their nearest neighbor cluster or designated as "misc" rather than proposed as standalone folders. Clusters larger than 50 files should be split (increase Leiden γ) or proposed with sub-folder structure.

## Key Findings
- **Miller's 7±2:** Working memory capacity is 7±2 chunks. A folder with > 9 files requires chunking (scanning) rather than direct recall. Files perceived as a coherent group: 3–7 items. This is a cognitive ceiling for "glanceable" folder contents, not a hard limit — users tolerate larger folders for archive/storage contexts.
- **Bergman et al. (2010) empirical data:** Mean folder size in personal collections: 14.3 files (SD: 22.1). Median: 8 files. Distribution is heavy-tailed (a few archive folders contain hundreds of files). Working folders (frequently accessed) average 5–12 files. Archive folders average 30–200 files.
- **Information foraging: Marginal Value Theorem (Charnov 1976):** A forager leaves a patch when the marginal gain rate equals the average environment rate. Applied to file navigation: a user "leaves" a folder (navigates elsewhere) when the cost of scanning exceeds the benefit of finding the target. For ≤ 7 files, scanning cost is < 1 second (below attention threshold); for > 20 files, scanning cost becomes measurable and users prefer search.
- **Practical minimum (5 files):** A folder with < 5 files is perceived as overhead — the folder itself costs more cognitive effort to remember and navigate to than simply placing the files elsewhere. Users in diary studies report deleting or ignoring folders with 1–3 files.
- **Practical maximum (50 files):** Above 50 files, users report difficulty maintaining folder identity ("I don't know what's in there"). At > 50 files, sub-folder proposals are appropriate.
- **Singleton and doubleton clusters:** Singletons (1 file) and doubletons (2 files) from Louvain/Leiden at high γ are common artifacts. These should be automatically reassigned to the nearest cluster (by affinity edge weight) rather than proposed as folders.
- **Resolution limit interaction:** Louvain/Leiden with modularity can fail to detect clusters smaller than √(2m) nodes due to the resolution limit. With sparse graphs (K=20 edges/node, n=1,000 files, m=20,000 edges): √(40,000) ≈ 200. This means clusters smaller than ~200 files may be merged by modularity. Use CPM (Constant Potts Model) to avoid this limit.

## Relevant Papers / Prior Art
- Miller G.A., "The Magical Number Seven, Plus or Minus Two," *Psychological Review* 63(2):81–97, 1956. DOI: 10.1037/h0043158. — Working memory capacity; 7±2 chunks.
- Pirolli P., Card S.K., "Information Foraging," *Psychological Review* 106(4):643–675, 1999. — Marginal Value Theorem applied to information search; patch-leaving thresholds.
- Bergman O. et al., "The effect of folder structure on personal file navigation," *JASIST* 61(12):2426–2441, 2010. DOI: 10.1002/asi.21415. — Empirical folder size distribution; working vs. archive folders.
- Fortunato S., Barthélemy M., "Resolution limit in community detection," *PNAS* 2007. DOI: 10.1073/pnas.0605965104. — Resolution limit formula √(2m).

## Applicability to Curator

```python
def post_process_clusters(labels: list[int], affinity_matrix: np.ndarray,
                          min_size: int = 5, max_size: int = 50) -> list[int]:
    """
    Post-process Leiden cluster labels:
    - Merge clusters smaller than min_size into nearest neighbor cluster
    - Flag clusters larger than max_size for sub-folder splitting
    """
    from collections import Counter
    import numpy as np
    
    labels = np.array(labels)
    cluster_sizes = Counter(labels)
    
    # Merge small clusters
    for cluster_id, size in cluster_sizes.items():
        if size < min_size:
            # Find members of this small cluster
            members = np.where(labels == cluster_id)[0]
            # For each member, find the nearest node NOT in this cluster
            for member in members:
                row = affinity_matrix[member].copy()
                row[members] = 0  # exclude self-cluster
                best_neighbor = np.argmax(row)
                labels[member] = labels[best_neighbor]  # reassign to neighbor's cluster
    
    # Re-normalize cluster IDs
    unique_ids = sorted(set(labels))
    id_map = {old: new for new, old in enumerate(unique_ids)}
    labels = np.array([id_map[l] for l in labels])
    
    # Flag large clusters for sub-folder split (return metadata only)
    cluster_sizes_new = Counter(labels)
    oversized = [cid for cid, sz in cluster_sizes_new.items() if sz > max_size]
    
    return labels.tolist(), oversized

# Decision for oversized clusters:
# If cluster_size > max_size → re-run Leiden on the subgraph of that cluster at higher γ
def split_oversized_cluster(cluster_members: list[int], affinity_matrix: np.ndarray,
                             gamma: float = 2.0) -> list[int]:
    sub_matrix = affinity_matrix[np.ix_(cluster_members, cluster_members)]
    sub_edges = [(i, j, sub_matrix[i, j]) for i in range(len(cluster_members))
                 for j in range(i+1, len(cluster_members)) if sub_matrix[i, j] > 0]
    sub_labels, _ = cluster_files(sub_edges, len(cluster_members), gamma=gamma)
    return sub_labels
```

**UI implication:** Never propose a folder with < 5 files as a standalone folder. Instead, show these files in a "Miscellaneous" tray within the parent cluster's review card.

## Open Questions
- Is 5 files the right minimum for all file types? Code project files may warrant smaller folders (a 3-file header+source+test trio is a coherent unit). Consider a file-type-aware minimum: `min_size = 3` for code files, `5` for documents/media.

## Sources
- https://www.nngroup.com/articles/spatial-memory/
- https://act-r.psy.cmu.edu/wordpress/wp-content/uploads/2012/12/280uir-1999-05-pirolli.pdf
- https://journals.plos.org/plosone/article?id=10.1371%2Fjournal.pone.0159161
- https://www.nature.com/articles/s41598-019-41695-z
