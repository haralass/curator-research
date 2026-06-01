# Heterogeneous File Affinity Graph: Design, Fusion & Maintenance
**File:** 82 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
A heterogeneous information network (HIN) treating files as nodes and semantic/entity/behavioral signals as typed edges outperforms pure embedding similarity for file retrieval and grouping. The foundational framework is PathSim (Sun et al., VLDB 2011) with extensions in HAN (Wang et al., 2019) and HERec (Shi et al., 2018). For Curator's scale (10k–500k file nodes), igraph is the practical default. The minimum viable HIN that beats pure cosine similarity requires only two edge types: semantic edges + NER entity co-occurrence.

## Key Findings

### HIN node and edge types for Curator
- **Node types:** File, Folder, Person, Project, Tag, Date
- **Edge types:** SemanticSim, EntityCoOccur, BehavioralCoAccess, ContainedIn, TaggedWith

### PathSim meta-paths
```
PathSim(x, y; P) = (2 × |p_{x→y}|) / (|p_{x→x}| + |p_{y→y}|)
```
Useful meta-paths:
- **F–Person–F**: files sharing a named person entity → client/project affinity
- **F–Session–F**: files co-opened in the same session → behavioral affinity
- **F–Folder–F**: files in the same folder → organizational proximity

### Three edge types

**Semantic edges:** Cosine sim > 0.75, k-NN graph (k=10–20), LSH for approximate k-NN at scale. Update trigger: content hash delta.

**NER entity co-occurrence edges:** Inverted index entity→[file_ids]. Weight = Jaccard similarity of shared entity sets. Subtypes: Person, Project/Org, Location edges. Pruning: ignore high-frequency entities (user's own name, "Apple", "PDF").

**Behavioral co-access edges:** FSEvents → session window (30 min) → co-access increment. Temporal decay:
```
w(e, t) = Σ_i  exp(−λ · (t_now − t_i))
```
Half-life ~140 days (λ=0.005). Prune when `w < ε=0.01`. Store as timestamped event log, recompute nightly.

### Edge weight fusion
**MVP (Option A) — weighted linear sum:**
```
affinity(u, v) = α·w_sem(u,v) + β·w_ner(u,v) + γ·w_beh(u,v)
```
Default: α=0.5, β=0.3, γ=0.2 (downweight behavioral until sufficient data). Transition to HAN-style learned attention (Option B) after ~500 labeled pairs from user confirmations.

### Graph maintenance under moves/renames/deletes
Node identity strategy (layered):
1. **Primary:** inode + device_id — stable across moves within same filesystem
2. **Secondary:** SHA-256 content hash — handles cross-filesystem moves
3. **Tertiary:** filename + size + mtime — fallback

- **Move/rename:** FSEvents `kFSEventStreamEventFlagItemRenamed` → update path attribute only. All edges preserved. O(1).
- **Delete:** Mark `status=deleted`, preserve edges for 7-day grace period, then purge.
- **Content change:** Recompute semantic + NER edges for that file only. Behavioral edges unaffected.
- **New file:** Embed → k-NN semantic edges → NER entity edges → behavioral edges accumulate over time.

### Library comparison at scale
| Library | 50k nodes | 500k nodes | Install | Notes |
|---|---|---|---|---|
| NetworkX | OK | Impractical | Trivial | Good for prototyping only |
| igraph | Fast | Manageable | Easy (pip) | C core; **recommended default** |
| graph-tool | Fastest | Fast | Hard | OpenMP parallel; best for betweenness at 1M+ |
| NetworKit | Fastest | Fastest | Moderate | Best for 500k+ traversals |

**Recommendation:** NetworkX for MVP, igraph for production.

### Existing PIM tools using graph-based relationships
| Tool | Approach | Gap |
|---|---|---|
| TheBrain | Explicit user-defined thought links | No automatic inference |
| TagSpaces 6.2 | Tags Graph — tag↔file↔folder visualization | User-defined tags only |
| DEVONthink | Cosine TF-IDF "See Also"; no explicit graph | Homogeneous; no HIN |
| Obsidian | Bidirectional links | Markdown only |

None implement heterogeneous automatic edge inference across semantic + NER + behavioral. This is Curator's differentiator.

### Minimum viable HIN
Two node types (File, Folder) + two edge types (SemanticSim + EntityCoOccur):
- Meta-paths: F–F (direct semantic), F–Entity–F (NER bridge), F–Folder–F (co-location)
- Fusion: α=0.6 semantic + β=0.4 NER

Sufficient to outperform pure cosine similarity because NER co-occurrence captures topical affinity that semantic similarity misses (e.g., two files about the same client written in different styles: low cosine sim, high NER co-occurrence).

## Relevant Papers / Prior Art
- PathSim — Sun et al., VLDB 2011 — foundational HIN similarity
- HAN — arXiv 1903.07293 — hierarchical attention over meta-paths
- HERec — arXiv 1711.10730 — meta-path random walk embeddings
- TGN Framework — arXiv 2403.16066 — temporal graph networks for dynamic edge weighting
- DeBaTeR — arXiv 2411.09181 — bipartite temporal graph denoising
- edge2vec — arXiv 1809.02269 — edge-type transition matrices
- Tim Lrx benchmarks — NetworkX/igraph/graph-tool/NetworKit performance comparison

## Applicability to Curator
1. **Cold-start graph:** Embed all files → k-NN semantic edges → NER entity edges. Usable immediately, behavioral edges accumulate.
2. **Similarity query:** PathSim over F–Entity–F and F–F meta-paths, weighted-sum fused with behavioral.
3. **Folder biography feedback:** Folder completeness scores (File 81) become Folder node features → feed meta-path weights.
4. **Explainability:** Edge types explain *why* files are related: "Both mention Project Alpha (NER)" vs. "Opened together Tuesday (behavioral)."
5. **Betweenness centrality (File 69):** Computed on this graph → high-BC files reviewed first in approval UI.

## Open Questions
- NER disambiguation: "Apple" company vs. user contact "Apple Chen" — resolve without full knowledge graph?
- Session definition sensitivity: 30-min window vs. 2-hour produces very different behavioral edge structures.
- Privacy: should behavioral edges be in-memory only, never persisted?
- At 500k files × 20 semantic edges + NER + behavioral = potentially 50M edges. Is igraph's memory footprint acceptable on 8GB MacBook?
- Meta-path selection automation: can Curator auto-discover useful meta-paths (WWW 2015 paper) vs. hard-coding them?

## Sources
- https://www.researchgate.net/publication/220538331_PathSim
- https://arxiv.org/pdf/1903.07293
- https://arxiv.org/pdf/1711.10730
- https://arxiv.org/html/2403.16066v2
- https://arxiv.org/pdf/2411.09181
- https://arxiv.org/pdf/1809.02269
- https://www.timlrx.com/blog/benchmark-of-popular-graph-network-packages-v2
- https://arxiv.org/pdf/2311.10260
- https://www.pkrc.net/detect-inode-moves.html
- https://dl.acm.org/doi/10.1145/2736277.2741123
