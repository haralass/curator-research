# Betweenness Centrality for Review UI Ordering
**File:** 69 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Betweenness centrality (BC) measures how often a node lies on the shortest path between all other node pairs in a graph — nodes with high BC are structural bridges. Applied to Curator's review queue, files that connect many semantic clusters (cross-project references, shared assets) should be reviewed first because their placement decision has downstream cascade effects on other suggestions. This gives a principled, graph-derived ordering for the approval UI.

## Key Findings
- BC formula: for each node v, `BC(v) = Σ(σ(s,t|v) / σ(s,t))` across all pairs — O(V·E) with Brandes' algorithm
- In Curator's context: nodes = files, edges = semantic similarity above threshold → high-BC files are "hubs" that, once placed, resolve many ambiguous suggestions for nearby files
- Approximation algorithms (Brandes 2001, divide-and-conquer arxiv 1406.4173) scale to tens of thousands of files efficiently
- Betweenness-ordering heuristic (arxiv 1409.6470): can rank by BC without computing exact values — good for real-time queue updates
- Neo4j and TigerGraph expose BC as a built-in GDS function — demonstrates production viability at scale

## Relevant Papers / Prior Art
- "An Efficient Heuristic for Betweenness-Ordering" — arxiv 1409.6470
- "Exact and Approximate Algorithms for Computing Betweenness Centrality in Directed Graphs" — arxiv 1708.08739
- "A Divide-and-Conquer Algorithm for Betweenness Centrality" — arxiv 1406.4173
- Neo4j Graph Data Science — betweenness centrality documentation

## Applicability to Curator
Medium-High. Build a file similarity graph in-memory (edges where cosine sim > 0.6), compute approximate BC in Python sidecar using NetworkX (`nx.betweenness_centrality(G, normalized=True)`), sort review queue by descending BC. Expected result: users resolve ambiguous clusters faster because anchor files get processed first. NetworkX handles 10k-node graphs in <2s on modern hardware.

## Open Questions
- What similarity threshold defines a meaningful edge for this graph? Too low = everything connected, BC loses signal.
- Should BC be recomputed after each approval (expensive) or only when the queue is first generated?
- Does high BC correlate with user-perceived "important to decide" or just semantic bridginess?

## Sources
- https://arxiv.org/pdf/1409.6470
- https://arxiv.org/pdf/1406.4173
- https://arxiv.org/pdf/1708.08739
- https://neo4j.com/docs/graph-data-science/current/algorithms/betweenness-centrality/
