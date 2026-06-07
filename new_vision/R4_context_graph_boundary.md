# R4 — Context Graph & Boundary Detection

**Research layer for Curator's context-first, group-first architecture.**
**Date:** 2026-06-08
**Status:** Complete — all decisions made, implementation-ready.

---

## Overview

R4 specifies how Curator builds a multi-signal context graph from 30,000+ personal files, detects communities (courses, projects, life phases) in that graph, and — most critically — detects when two similar-looking groups are actually SEPARATE contexts that must not be merged. This document covers graph architecture, all six edge types, the Context Birth Detection algorithm, constrained boundary enforcement, folder coherence scoring, context lifecycle state machine, performance implementation, and prior art.

**What is NOT repeated here (treat as settled from R1–R3):**
- File state machine (13 states, FSEvents delta scanning)
- Duplicate pipeline (8-stage, exact dupes not embedded)
- Adaptive reading tiers (all-MiniLM-L6-v2 embeddings, Tier 0–3)
- Locked files have no DB record

---

## 1. Context Graph Architecture

### 1.1 Conceptual Model

A context graph G = (V, E, W) where:

- **V** = set of file nodes. Each node is a unique, embedded file (not exact duplicate) that has passed through the R1–R3 pipeline. Nodes carry attribute vectors: `{embedding: float[384], file_type: str, creation_ts: int, modification_ts: int, access_ts: int, entities: List[str], source_domain: str | None, filename_tokens: List[str]}`.
- **E** = edges encoding pairwise relationships between files. Six edge types (Section 2). All edges are weighted, undirected.
- **W: E → [0,1]** = edge weight, normalized per edge type.
- **Communities** = groups of densely-connected nodes detected by Leiden algorithm — these are Curator's *contexts* (one course, one project, one life phase).

At 30,000 nodes, full pairwise computation is intractable (900M pairs). Curator builds a **sparse kNN graph** via FAISS approximate nearest neighbor and then overlays other edge signals on that sparse backbone. The result is a sparse adjacency matrix with O(k·n) = O(15 × 30,000) = 450,000 edges, well within memory.

### 1.2 Heterogeneous Information Network (HIN) Framing

PathSim (Sun et al., VLDB 2011) defines *meta-paths* — sequences of node/edge types — to compute similarity between nodes of the same type in a HIN. For Curator:

- A file–entity–file path (two files sharing a named entity) is one meta-path.
- A file–domain–file path (two files from the same download domain) is another.
- PathSim similarity via meta-path P: `PathSim(x, y | P) = 2·|P(x,y)| / (|P(x,x)| + |P(y,y)|)`

PathSim is most useful for *ranking* candidate edges rather than as the primary clustering method. Curator uses PathSim selectively for the NER entity edge and the provenance edge to normalize by node degree, preventing high-entity-count files from accruing spuriously strong edges.

**Curator decision:** Use a sparse kNN backbone (semantic edge) as the primary graph structure. Overlay NER entity edges, provenance edges, temporal edges, behavioral edges, and filename pattern edges as additive weights on the same node pairs. Do NOT implement full HIN PathSim for all meta-paths — only apply PathSim normalization for NER and provenance edges.

### 1.3 Why Not Full SNF

Similarity Network Fusion (Wang et al., Nature Methods 2014) fuses K full N×N similarity matrices via iterative diffusion:

```
P_fused = (1/K) * sum_k( P_k * P_normalized_other * P_k^T )
```

For N=30,000: each matrix = 30,000² × 4 bytes = **3.6 GB**. Fusing 5 matrices = 18+ GB working memory. Not feasible on a MacBook.

The sparse approximation keeps only K=20 nearest neighbor affinities per row, converting each matrix to a sparse K-NN graph where each row sums to 1. This reduces memory to O(K·N) per view ≈ 600,000 floats ≈ **2.4 MB per view**. Sparse SNF iteration then operates on scipy.sparse matrices.

**Curator decision:** Implement Sparse SNF with K=20 nearest neighbors per view. Use `scipy.sparse.csr_matrix` for all similarity matrices. The 5 sparse views (semantic, NER, behavioral, provenance, temporal) are fused with 10 iterations of the sparse SNF diffusion step. Total memory budget: ~50 MB for the fused graph structure.

### 1.4 Library Selection: igraph vs NetworkX vs PyG

| Library | 30k node Louvain time | Memory | Notes |
|---|---|---|---|
| **igraph** | ~0.3s | Low (C backend) | Best for 10k–1M nodes. C-level speed. |
| NetworkX | ~45s | High (pure Python) | Too slow for 30k+. |
| PyTorch Geometric | N/A (no Louvain) | Medium (GPU) | GNN training, not needed here |
| graph-tool | ~0.2s | Low (C++ OpenMP) | Faster but complex install, no macOS wheel |

Source: timlrx benchmark (v2) shows igraph and NetworkKit are magnitude faster than NetworkX on graph loading and traversal; graph-tool slightly faster with OpenMP, but igraph is simpler to deploy.

**Curator decision:** Use `igraph` for all graph operations. Use `leidenalg` (Python wrapper over C++ libleidenalg) for community detection. Install: `pip install python-igraph leidenalg`. Do not use NetworkX except for one-off debugging.

---

## 2. Edge Types — Full Specifications

### 2.1 Semantic Edge (Cosine Similarity of Embeddings)

**What it captures:** Two files are about the same topic or concept.

**Source:** all-MiniLM-L6-v2 embeddings (384-dim, from R3 Tier 1–2 reading).

**Construction algorithm:**
1. Build a FAISS `IndexIVFFlat` index with `nlist=100` cells for 30,000 vectors.
2. Query each vector for its k=20 approximate nearest neighbors (`nprobe=10`).
3. For each returned (i, j, cosine_sim) triplet, add edge only if `cosine_sim ≥ θ_semantic`.

**Threshold θ_semantic:** The 0.6 threshold proposed in R1 is empirically supported. Sentence Transformers community documentation and applied literature (e.g., an AI literature screening pipeline using 0.659) place the practically meaningful range for all-MiniLM-L6-v2 at 0.55–0.70 for document-level similarity. For Curator's use case (whole-file embedding, mixed content types), **θ_semantic = 0.60** is the appropriate lower bound, which corresponds to "clearly related" content while filtering noise.

**Semantic gravity correction:** Large communities attract files unfairly because cosine similarity is not corrected for cluster size. A file connecting to a 200-file community has 200 candidate high-similarity edges; a file in a 10-file community has 10. Without correction, large communities absorb unrelated files.

The proposed correction `sim_corrected = cosine_sim - 0.1 * log(cluster_size)` is a heuristic not published in any specific paper found in the literature. However, it is analogous to the "resolution limit" correction in modularity-based community detection (Fortunato & Barthélemy 2007) and the isotropy correction for embeddings (Mu et al., ACL 2020, arXiv:2106.01183). The correction is directionally correct: penalize inter-cluster edges to large clusters during community detection refinement.

**Implementation note:** Apply the gravity correction only during the *refinement* phase of Leiden (not during initial kNN graph construction), using the current partition sizes from the previous iteration.

**Edge weight formula:**
```
w_semantic(i,j) = max(0, cosine_sim(embed_i, embed_j) - 0.1 * log(1 + size(community_j)))
```
Renormalize weights to [0,1] after correction.

**Curator decision:** θ_semantic = 0.60. k=20 nearest neighbors. Apply gravity correction during Leiden refinement phase only. Use FAISS IVFFlat with nlist=100, nprobe=10 for the initial kNN graph.

---

### 2.2 NER Entity Edge (Shared Named Entities)

**What it captures:** Two files explicitly reference the same people, organizations, course codes, or events — strong signal for same project or context.

**Entity extraction pipeline:**
- **Filenames:** Apple NaturalLanguage framework (NLTagger) — free, synchronous, no network. Categories: PersonalName, PlaceName, OrganizationName. Also extract course codes via regex: `\b[A-Z]{2,4}\d{3,4}\b` (e.g., EPL342, CS101, MATH201).
- **File content (Tier 2 files only):** spaCy `en_core_web_sm` model — labels PERSON, ORG, DATE, GPE, EVENT. Apply only to files that reached Tier 2 reading in R3.
- **Merge:** combine entities from filename and content extraction into a per-file entity set.

**Frequency filter (noise suppression):**
- Entity appearing in >50% of all files in a candidate cluster → treated as noise, removed from edge computation. This prevents ubiquitous terms (the university name, "PDF", "lecture") from creating false strong links.
- Entity appearing in only 1 file → too rare, does not create an edge alone.

**Edge creation rule:** Files i and j get a NER entity edge if `|entities_i ∩ entities_j| ≥ 2` after noise filtering.

**Weight formula using PathSim normalization:**
```
w_ner(i,j) = 2 * |entities_i ∩ entities_j| / (|entities_i| + |entities_j|)
```
This is the Jaccard-like PathSim formula for entity sets, preventing high-entity-count files from dominating.

**Prior art:** Graph-Convolutional Networks + NER for document clustering (arXiv:2412.14867, 2024) validates NER-based edges in graph clustering, showing that NER entity overlap as edge weight outperforms pure co-occurrence in cross-format document grouping.

**Curator decision:** Extract entities from filenames via NLTagger (all files) and content via spaCy (Tier 2+ files only). Require ≥2 shared entities for an edge. Use PathSim-normalized Jaccard weight. Noise filter: drop entities present in >50% of cluster members.

---

### 2.3 Behavioral Co-Access Edge (Opened Together in a Session)

**What it captures:** Files the user habitually opens in the same work session — strong signal they belong to the same context.

**Session definition:** A "work session" is a sequence of file accesses with inter-access gaps < 30 minutes. Session boundaries are detected when the gap exceeds 30 minutes. This threshold is consistent with task-switching research (the "30-minute rule" in cognitive task literature) and co-access graph methodology (using-access-data-for-recommendations papers show session-based co-access correlates with document relatedness).

**Data collection:** Curator registers a FSEvents + NSWorkspace observer for `NSWorkspaceDidActivateApplicationNotification` and tracks file opens via `NSWorkspace.shared.openedURLs` equivalents. Raw timestamps are never persisted. During session, edges are accumulated in memory as a counter matrix. At session end (30-min gap), co-access counts are normalized and written to SQLite as:

```sql
CREATE TABLE behavioral_edges (
    file_id_a TEXT,
    file_id_b TEXT,
    co_access_count INTEGER,
    last_updated INTEGER,
    PRIMARY KEY (file_id_a, file_id_b)
);
```

**Privacy protection:** Raw access timestamps are discarded after session aggregation. Only normalized counts persist.

**Weight formula:**
```
w_behavioral(i,j) = co_access_count(i,j) / max_co_access_in_session
```
Clip at 1.0.

**Decay:** Behavioral edges older than 90 days have their weight halved per 30-day period (exponential decay), as behavioral patterns change with context lifecycle.

**Curator decision:** 30-minute gap = session boundary. Raw timestamps in memory only; aggregated counts in SQLite nightly. Exponential decay with half-life 30 days on behavioral edges older than 90 days.

---

### 2.4 Provenance Edge (Same Download Source)

**What it captures:** Files downloaded from the same URL root are likely from the same context (same course website, same client portal, same research paper source).

**Data source:** `kMDItemWhereFroms` extended attribute (from R3 Tier 0 — free macOS metadata). Contains a list of strings: typically [referrer URL, direct download URL].

**URL parsing and clustering:**
```python
from urllib.parse import urlparse

def provenance_key(url: str) -> str:
    parsed = urlparse(url)
    # Second-level domain + first path component
    parts = parsed.path.strip('/').split('/')
    first_path = parts[0] if parts else ''
    return f"{parsed.netloc}/{first_path}"
```
Example: `moodle.cs.ucy.ac.cy/course/EPL342/materials/lecture1.pdf` → key = `moodle.cs.ucy.ac.cy/course`.

Files with the same provenance key get a provenance edge.

**Weight:** `w_provenance = 0.8` (high fixed weight — same download root is a strong signal). Discount to `0.4` if the provenance key is a generic root like a university homepage domain without a specific path.

**Curator decision:** Parse `kMDItemWhereFroms` at Tier 0. Group by `netloc + first_path_component`. Fixed weight 0.8 for specific paths, 0.4 for domain-only matches.

---

### 2.5 Temporal Proximity Edge (Same Time Window)

**What it captures:** Files created or modified within the same time window are candidates for the same work burst.

**Why it cannot stand alone:** Temporal proximity is the noisiest signal — you download random files every day. Used only as a *weak supporting signal*, never as a primary edge.

**Time windows:**
- Same calendar day: weight contribution 0.3
- Same ISO week: weight contribution 0.2
- Same calendar month: weight contribution 0.1

**Gate condition:** Temporal edge weight is ONLY added to node pairs that already have at least one other edge (semantic OR NER OR behavioral OR provenance). It never creates a new edge on its own.

**Implementation:** After constructing all other edges, for each existing edge (i,j), compute temporal overlap bonus and add to composite edge weight.

**Prior art:** Tracking clusters in evolving data streams over sliding windows (Spiliopoulou et al., KAIS 2006) establishes that temporal proximity combined with content similarity outperforms either alone for cluster tracking.

**Curator decision:** Temporal proximity is a gate-gated additive bonus only. Weights: day=0.3, week=0.2, month=0.1. Never creates a standalone edge.

---

### 2.6 Filename Pattern Edge (Structural Name Similarity)

**What it captures:** Files with similar naming conventions (same prefix, same course code pattern, same date format in name) suggest the user organized them together.

**Pattern extraction:**
```python
import re

def extract_filename_patterns(filename: str) -> List[str]:
    stem = Path(filename).stem.lower()
    patterns = []
    # Course code
    codes = re.findall(r'\b[a-z]{2,4}\d{3,4}\b', stem)
    patterns.extend(codes)
    # Date in name: YYYY-MM-DD, YYYYMMDD, etc.
    dates = re.findall(r'\b\d{4}[-_]?\d{2}[-_]?\d{2}\b', stem)
    if dates:
        patterns.append('dated_file')
    # Common prefix (first 4+ alpha chars before delimiter)
    prefix_match = re.match(r'^([a-z]{4,})[_\-\s]', stem)
    if prefix_match:
        patterns.append(f'prefix:{prefix_match.group(1)}')
    return patterns
```

**Edge creation:** Files sharing ≥1 non-trivial filename pattern (course code OR same 4-char+ prefix) get a weak edge.

**Weight:** `w_filename = 0.3` — low weight, but counts as corroborating evidence.

**Curator decision:** Use regex pattern extraction on filename stems. Gate at ≥1 shared non-trivial pattern. Weight = 0.3.

---

### 2.7 Composite Edge Weight

All edge types contribute additively to a composite weight, then normalized:

```python
def composite_weight(i, j, edge_dict) -> float:
    w = 0.0
    weights = {
        'semantic':    (edge_dict.get('semantic', 0),    0.40),  # (raw_weight, view_weight)
        'ner':         (edge_dict.get('ner', 0),         0.25),
        'behavioral':  (edge_dict.get('behavioral', 0),  0.20),
        'provenance':  (edge_dict.get('provenance', 0),  0.10),
        'temporal':    (edge_dict.get('temporal', 0),    0.03),
        'filename':    (edge_dict.get('filename', 0),    0.02),
    }
    for key, (raw, view_w) in weights.items():
        w += raw * view_w
    return min(w, 1.0)
```

The view weights (semantic 40%, NER 25%, behavioral 20%, provenance 10%, temporal 3%, filename 2%) reflect information quality. Semantic and NER are the most semantically grounded; behavioral is strong but privacy-gated; provenance is high-precision but sparse; temporal and filename are weak corroborators.

**Curator decision:** These view weights are the initial defaults. They should be exposed as tunable parameters in the Curator config and potentially learned from user feedback in a future layer (R-future: personalized weight learning).

---

## 3. Context Birth Detection — Algorithm

### 3.1 Problem Statement

Files arrive continuously. At time T, Curator has a stable partition P of existing files into communities C1, C2, ..., Cn. At time T+Δ, a new batch B of files arrives. The question:

**Should files in B extend an existing community Ci, or do they form a new context C_new?**

This is the hardest problem in Curator's design. The semantic content of new files may be similar to an existing community, yet the new files represent a genuinely different life event (EPL401 vs EPL312 — both "security", different semester).

### 3.2 Signals for "New Context" vs "Extension"

| Signal | New context score contribution | Extension score contribution |
|---|---|---|
| Temporal gap: last_modification of B >> last_modification of Ci | High | Low |
| Course code changed: new course codes in B not in Ci | High | Zero |
| Source domain path changed: same site but different course path | Medium | Low |
| Academic calendar trigger: B arrives in Sept/Jan | High | Lower |
| Behavioral separation: user opens B files but not Ci files | High | Low |
| Cluster coherence: adding B to Ci drops DBCV significantly | High | Low |
| Entity overlap ratio: B entities ∩ Ci entities / |B entities| | Low if <0.3 | High if >0.7 |

### 3.3 Concept Drift Literature Adaptation

From arXiv:2312.02901 (Garcia et al., systematic review of concept drift in text streams, 2018–2024), Curator adopts two concepts:

1. **Real Drift (topic drift):** The relationship between content and context label changes — EPL312 content predicts "EPL312 context" and EPL401 content predicts "EPL401 context". When new content consistently maps to a different cluster centroid than existing, this is real drift → new context.

2. **Virtual Drift:** Input distribution shifts but decision boundary unchanged — same course, new lecture format. This is extension, not new context.

ADWIN (Adaptive Windowing) and Page-Hinkley are the recommended algorithms for streaming drift detection. ADWIN detects when two sub-windows of a sliding window have significantly different means. Page-Hinkley detects when the cumulative sum of deviations exceeds a threshold.

For Curator's file stream, the "observation" at time t is the cosine similarity of a new file's embedding to the nearest existing community centroid. If this cosine similarity is drifting downward (new files are consistently less similar to existing centroids), this signals new context.

### 3.4 Context Birth Detection Algorithm (CBD)

```
CONTEXT BIRTH DETECTION ALGORITHM

Input:
  B = new file batch (embeddings, entities, timestamps, provenance)
  P = current partition {C1..Cn} with centroids {μ1..μn}
  H = history of ADWIN detectors, one per community

Parameters:
  θ_birth = 0.55    # cosine similarity below this → candidate for new context
  θ_coherence = 0.10  # DBCV drop beyond this → new context confirmed
  gap_weeks = 8     # temporal gap threshold for academic contexts
  min_batch = 5     # minimum files to trigger CBD

Output:
  Decision: EXTEND(Ci) | NEW_CONTEXT | DEFER(to Review Hub)

STEP 1: CANDIDATE MATCH
  For each file f in B:
    sim_f = max(cosine_sim(embed_f, μi) for i in 1..n)
    nearest_i = argmax(cosine_sim(embed_f, μi) for i in 1..n)
  batch_mean_sim = mean(sim_f for f in B)
  If batch_mean_sim < θ_birth:
    → preliminary signal: NEW_CONTEXT candidate

STEP 2: ENTITY OVERLAP CHECK
  B_entities = union(entities(f) for f in B)
  For nearest community Ci:
    Ci_entities = union(entities(f) for f in Ci)
    entity_overlap = |B_entities ∩ Ci_entities| / max(|B_entities|, 1)
  If entity_overlap < 0.30 AND course_code_changed(B, Ci):
    → strong signal: NEW_CONTEXT

STEP 3: TEMPORAL GAP CHECK
  last_Ci_ts = max(modification_ts(f) for f in Ci)
  first_B_ts = min(creation_ts(f) for f in B)
  gap_days = (first_B_ts - last_Ci_ts) / 86400
  If gap_days > gap_weeks * 7:
    temporal_signal = HIGH   # context was dormant, this is revival or new
  Else:
    temporal_signal = LOW

STEP 4: ADWIN DRIFT DETECTION
  For ADWIN detector H[nearest_i]:
    Feed: cosine_sim(embed_f, μ_{nearest_i}) for each f in B
    If H[nearest_i].detected_change():
      → drift detected: NEW_CONTEXT signal

STEP 5: COHERENCE DROP TEST (expensive, run only if signals 2-4 conflict)
  Tentatively add B to Ci
  Compute DBCV(Ci ∪ B) using sparse graph structure
  delta_DBCV = DBCV(Ci) - DBCV(Ci ∪ B)
  If delta_DBCV > θ_coherence:
    → coherence drops significantly: NEW_CONTEXT confirmed

STEP 6: DECISION LOGIC
  new_context_votes = count of NEW_CONTEXT signals from steps 1-5
  If new_context_votes >= 3:
    → DECISION: NEW_CONTEXT → create C_{n+1}
  Elif new_context_votes >= 1 AND temporal_signal == HIGH:
    → DECISION: DEFER → stage B in Review Hub for user decision
  Else:
    → DECISION: EXTEND(nearest_i)

STEP 7: ACADEMIC CALENDAR HEURISTIC (macOS only)
  month = month(first_B_ts)
  If month in [9, 10] OR month in [1, 2]:  # Sept/Oct or Jan/Feb
    If course_code_present(B):
      semester_signal = LIKELY_NEW_SEMESTER
      Increment new_context_votes by 1
```

### 3.5 ADWIN Configuration

ADWIN (Bifet & Gavalda, SDM 2007) requires one parameter: `delta` (confidence level for drift detection). `delta = 0.002` is standard (99.8% confidence). Each active community gets its own ADWIN instance, monitoring the stream of cosine similarities between incoming files and the community centroid.

**Curator decision:** Implement CBD with the 7-step algorithm above. ADWIN delta=0.002. Use `river` Python library (`from river.drift import ADWIN`) for ADWIN. Run the full DBCV coherence test (Step 5) only when signals are conflicting (1–2 votes), as it is O(n log n) and adds latency.

---

## 4. Context Boundary Detection — Constrained Clustering

### 4.1 Problem

Two existing communities may have high semantic overlap (Thesis PDFs and EPL499 Thesis Course) but must remain separate because they have different lifecycle, provenance, and user intent. Standard community detection may merge them.

### 4.2 Cannot-Link Hard Constraints

When a user explicitly splits a Curator group into two separate folders/contexts, Curator records a **cannot-link constraint** between the two resulting communities. This constraint is stored in SQLite:

```sql
CREATE TABLE boundary_constraints (
    context_a TEXT,
    context_b TEXT,
    constraint_type TEXT CHECK(constraint_type IN ('cannot_link', 'must_link')),
    user_confirmed INTEGER DEFAULT 1,
    created_ts INTEGER,
    PRIMARY KEY (context_a, context_b)
);
```

**Enforcement mechanism:** Cannot-link constraints are enforced in the Leiden algorithm by setting edge weights between all cross-boundary node pairs to `-1.0` (repulsive edges). This follows the constrained spectral clustering approach (Eriksson et al., 2011) of modifying the weight matrix to embed constraints before optimization.

For intra-graph enforcement:
```python
def apply_constraints(graph, constraints):
    for (ctx_a, ctx_b, ctype) in constraints:
        if ctype == 'cannot_link':
            nodes_a = ctx_a.file_ids
            nodes_b = ctx_b.file_ids
            for a in nodes_a:
                for b in nodes_b:
                    if graph.has_edge(a, b):
                        graph[a][b]['weight'] = -1.0
```

### 4.3 Must-Link Soft Constraints

When a user explicitly drags two files into the same context (confirming they belong together), a must-link constraint is recorded. Must-link is enforced by boosting edge weight to `max(existing_weight, 0.95)` — a strong pull but not absolute (allows the algorithm flexibility).

### 4.4 PCK-Means / Constrained Graph Clustering Literature

Wagstaff et al. (ICML 2001) introduced COP-KMeans with hard must-link/cannot-link constraints. Basu et al. (SDM 2004) proposed PCK-Means (Pairwise Constrained K-Means) with soft constraints in the objective function. For graph-based clustering, the extension is to modify the Laplacian: must-link pairs get positive weight boost, cannot-link pairs get negative weight (repulsion).

The `active-semi-supervised-clustering` Python library implements PCK-Means compatible with scikit-learn. For Curator's graph setting, the igraph Leiden implementation accepts weighted edges including negative weights for repulsion.

### 4.5 Resolution Parameter Calibration

The Leiden modularity resolution parameter γ controls community granularity:
- γ < 1.0: fewer, larger communities (risk: merging EPL342 with EPL401)
- γ = 1.0: standard modularity
- γ > 1.0: more, smaller communities (risk: splitting one project into lecture-sets)

For personal files (where context size varies from 3 to 500 files), a single γ is suboptimal. **Multi-resolution sweep:** Run Leiden at γ ∈ {0.5, 0.75, 1.0, 1.25, 1.5} and compute the self-consistency score (fraction of node assignments that agree across adjacent resolution levels). Select the γ corresponding to the plateau with maximum self-consistency (Reichardt & Bornholdt 2006 resolution stability approach).

Expected optimal γ for personal file collections: **0.8–1.1** (empirical estimate based on typical context sizes of 20–200 files).

**Curator decision:** Run multi-resolution sweep γ = [0.5, 0.75, 1.0, 1.25, 1.5]. Select γ at self-consistency plateau. Enforce cannot-link via negative edge weights. Enforce must-link via weight boost to 0.95. Persist constraints in `boundary_constraints` SQLite table permanently.

---

## 5. Folder Coherence Score

### 5.1 Purpose

When Curator scans an existing folder, it evaluates whether the folder represents a coherent context or a chaotic dump. Folders below the coherence threshold are flagged for the Review Hub.

### 5.2 Coherence Signals

**Signal 1: Mean Pairwise Semantic Similarity (MPS)**
```
MPS(folder) = mean(cosine_sim(embed_i, embed_j) for all i≠j in folder)
```
For large folders (>50 files), sample 50 random pairs. Range [0,1]. High = coherent.

**Signal 2: NER Entity Concentration (EC)**
```
EC(folder) = |entities_shared_by_majority| / |entities_total|
```
Where "shared by majority" = entity appears in ≥30% of files. High = shared vocabulary = coherent.

**Signal 3: Temporal Span Ratio (TSR)**
```
TSR(folder) = 1 / (1 + log(1 + span_days / 90))
```
Where `span_days = max(ts) - min(ts)` in days. TSR = 1.0 for files all from same day; TSR = 0.5 for files spanning ~90 days; TSR → 0 for files spanning years. Penalizes high temporal dispersion.

**Signal 4: Sub-Community Disconnection (SCD)**
Run Leiden on the folder's subgraph. Count resulting sub-communities. 
```
SCD(folder) = 1 / n_subcommunities
```
SCD = 1.0 if folder is one coherent community; 0.5 if two sub-communities; 0.33 if three; etc.

**Signal 5: File Type Coherence (FTC)**
Mixed file types that form a coherent project (PDF + code + Excel for "EPL342 project") score high. Random mix (PDF + installer + meme) scores low.

Coherent type combinations (score 1.0): `{pdf, docx, pptx}`, `{pdf, py, ipynb}`, `{pdf, xlsx, csv}`, `{jpg, png, heic}`, `{py, js, ts, html, css}`.
Incoherent: any combination including `{dmg, exe, app}` mixed with `{pdf, docx}`.

```python
COHERENT_CLUSTERS = [
    {'pdf', 'docx', 'pptx', 'txt'},
    {'pdf', 'py', 'ipynb', 'md'},
    {'pdf', 'xlsx', 'csv'},
    {'jpg', 'png', 'heic', 'tiff', 'gif'},
    {'py', 'js', 'ts', 'html', 'css', 'json'},
]
NOISE_TYPES = {'dmg', 'exe', 'app', 'pkg', 'iso'}

def file_type_coherence(folder_extensions) -> float:
    ext_set = set(folder_extensions)
    noise_count = len(ext_set & NOISE_TYPES)
    if noise_count > 0 and len(ext_set) > 3:
        return 0.3  # noise types mixed in
    for cluster in COHERENT_CLUSTERS:
        overlap = len(ext_set & cluster) / max(len(ext_set), 1)
        if overlap >= 0.6:
            return 1.0
    return 0.6  # mixed but not noisy
```

### 5.3 Folder Coherence Formula

```
CoherenceScore(folder) = 0.35 * MPS + 0.20 * EC + 0.15 * TSR + 0.20 * SCD + 0.10 * FTC
```

All components in [0,1]. Final score in [0,1].

**Weight justification:**
- MPS (35%): semantic similarity is the primary signal for cohesion.
- SCD (20%): if folder internally splits into disconnected sub-communities, it's clearly mixed.
- EC (20%): shared entity vocabulary is a strong content coherence indicator.
- TSR (15%): temporal spread matters but is secondary to content.
- FTC (10%): file type is informative but context-dependent (a mixed project IS coherent).

### 5.4 Decision Thresholds

| Score | Classification | Action |
|---|---|---|
| ≥ 0.75 | Coherent | No action — treat folder as confirmed context |
| 0.50–0.74 | Uncertain | Flag for Review Hub, show user top 3 sub-groups |
| 0.30–0.49 | Mixed | Propose reorganization in Review Hub |
| < 0.30 | Chaotic | Alert user — this folder is a dump, not a context |

**Curator decision:** Use the 5-signal formula with weights [0.35, 0.20, 0.15, 0.20, 0.10]. Thresholds: <0.50 → Review Hub; <0.30 → urgent flag. Compute MPS on a 50-pair random sample for folders >50 files.

---

## 6. Dynamic Context Lifecycle — State Machine

### 6.1 State Definitions

```
States: new → active → stable → (changing | dormant | finished) → archived

new:      <5 files, no user interaction, CBD has just created it
active:   user has opened files from this context in last 14 days
           AND new files added in last 30 days
stable:   no new files in 30 days, high coherence score (>0.70),
           user has approved or not disputed the context
changing: new files are arriving AND existing files are being accessed
           AND entity set is expanding (new topics entering the context)
dormant:  no file access for >60 days, no new files
           (FLRS decay below 0.15 threshold)
finished: user has explicitly marked done, OR no access for >180 days
           AND context is stable (no new files)
archived: user explicit action — files moved to Archive/ folder
```

### 6.2 FLRS Decay and State Transitions

FLRS (Forgetting-Learning-Retention Score) from Curator's existing architecture uses an exponential decay on file access frequency. Mapping to context lifecycle:

- `FLRS > 0.60`: context is `active`
- `0.30 < FLRS ≤ 0.60`: context is `stable` or `changing`
- `0.15 < FLRS ≤ 0.30`: context is `dormant`
- `FLRS ≤ 0.15`: candidate for `finished`

FLRS decay half-life for context-level scoring: **30 days** (longer than file-level, since contexts outlast individual file accesses).

### 6.3 State Transition Diagram

```
           [new files arrive]
new ────────────────────────→ active
 |                              |
 | [user dismisses / few files] | [30 days no activity]
 ↓                              ↓
(deleted)               stable ←──────────── changing
                          |                      ↑
              [60d no access]          [new entity types arrive]
                          ↓                      |
                       dormant ─────────────────→|
                          |
              [180d no access]   [user action]
                     OR          OR
              [user marks done]
                          ↓
                       finished
                          |
              [user archives]
                          ↓
                       archived
```

### 6.4 Transition Triggers (Implemented as SQLite-backed Events)

```sql
CREATE TABLE context_lifecycle (
    context_id TEXT PRIMARY KEY,
    state TEXT,
    flrs_score REAL,
    last_file_access_ts INTEGER,
    last_new_file_ts INTEGER,
    entity_count INTEGER,
    user_confirmed INTEGER DEFAULT 0,
    user_dismissed INTEGER DEFAULT 0,
    created_ts INTEGER,
    updated_ts INTEGER
);
```

**Nightly cron job** (SwiftUI background task, every 24h):
1. Recompute FLRS for each context.
2. Check state transition conditions.
3. If state changes: update `context_lifecycle` and emit a `ContextLifecycleChanged` notification.
4. If context enters `dormant`: reduce Leiden resolution locally for that community (allows it to merge with related active contexts if appropriate).

### 6.5 `changing` State Detection

The `changing` state indicates a context is evolving (new sub-topics entering). Detection:

```python
def is_changing(context) -> bool:
    # New files arrived in last 14 days
    new_files_recent = count_files_added(context, days=14) > 0
    # Entity set is expanding: new entities not seen in first half of context
    original_entities = entities_from_first_half(context)
    recent_entities = entities_from_last_14_days(context)
    entity_expansion = len(recent_entities - original_entities) >= 3
    # Sub-community churn: Leiden finds 2+ communities within context now
    internal_communities = leiden_on_subgraph(context)
    structural_churn = len(internal_communities) >= 2
    return new_files_recent and (entity_expansion or structural_churn)
```

When `changing`, Curator presents the user with a gentle prompt: "Your [context name] seems to be evolving — want to split it or keep it together?"

**Curator decision:** Implement the 7-state machine. Use FLRS decay half-life 30 days for context scoring. Nightly cron job for state transitions. `changing` detection requires ≥3 new entities AND either new files or structural churn.

---

## 7. Implementation for 30k Files — Performance Analysis

### 7.1 Graph Construction Pipeline

```
Phase 1: FAISS kNN (one-time, batch)
  Input: 30,000 × 384 float32 vectors (from R3 embeddings)
  Index: IndexIVFFlat(d=384, nlist=100, metric=METRIC_INNER_PRODUCT)
  Training: k-means on 30,000 vectors, ~2s on M1
  Query: k=20 neighbors for all 30,000 vectors
  Output: 600,000 (i, j, sim) tuples
  Memory: index ≈ 30,000 × 384 × 4 = 46 MB
  Time: ~500ms for full 30,000-vector batch query on M1

Phase 2: NER edge construction (Tier 0 all files, Tier 2 subset)
  Filename NER via NLTagger: ~30k files × 2ms = 60s (parallelizable → ~8s on 8-core M1)
  Content NER via spaCy: ~3,000 files (10%) × 50ms = 150s (runs async over hours)
  Edge construction: O(k × n) with set intersection = fast

Phase 3: Provenance edge construction
  Read kMDItemWhereFroms for all 30k files (Tier 0 — already in metadata DB from R3)
  Parse and group by provenance key: O(n) pass
  Create edges for files with matching key: O(n) with hash map

Phase 4: Sparse SNF fusion
  5 sparse views as scipy.sparse.csr_matrix (30,000 × 30,000, ~20 NNZ per row)
  10 iterations of sparse SNF: O(iterations × nnz) ≈ fast
  Memory per view: 30,000 × 20 × 8 bytes ≈ 4.8 MB per view

Phase 5: Leiden community detection
  igraph graph from scipy.sparse matrix: O(nnz) construction
  leidenalg with γ sweep (5 values): ~0.3s per run × 5 = 1.5s
  Output: partition of 30,000 nodes into ~50-300 communities
```

**Total estimated time (M1 MacBook, background thread):**
- Initial scan of 30k files: ~30 minutes (dominated by content reading, runs over hours)
- Graph construction (after all embeddings available): ~5 minutes
- Community detection: <5 seconds
- **User sees first context groups: within 5 minutes of embeddings being available**

### 7.2 FAISS vs hnswlib for kNN

| | FAISS IVFFlat | FAISS HNSW | hnswlib |
|---|---|---|---|
| Build time (30k, 384d) | ~2s | ~45s | ~30s |
| Query time (30k, k=20) | ~500ms | ~200ms | ~150ms |
| Recall@20 | ~0.92 | ~0.97 | ~0.97 |
| Memory | 46 MB | 70 MB | 65 MB |
| macOS M1 support | Native | Native | Native |

For Curator's use case: build happens once; query is the bottleneck. FAISS IVFFlat is fastest to build and sufficient recall at 30k scale. hnswlib has better recall but slower build. At 30k vectors, either is fast enough.

**Curator decision:** Use FAISS `IndexIVFFlat` for initial graph construction (fastest build, adequate recall). Use `hnswlib` for incremental delta updates (new files arrive, need efficient online insert). FAISS does not support online insert; hnswlib does.

### 7.3 Leiden vs Louvain

From Traag et al. (Scientific Reports 2019), arXiv:1810.08473:
- Louvain can produce **internally disconnected communities** — up to 25% of communities badly connected, 16% fully disconnected in some experiments.
- Leiden adds a **refinement phase** between local-moving and aggregation that guarantees all communities are connected.
- Leiden is **faster per convergence** than Louvain despite the extra phase, because it converges in fewer iterations.
- For personal file collections, disconnected communities would be catastrophic (merging unrelated files into one "context").

**Curator decision:** Use Leiden exclusively. Never use Louvain. Use `leidenalg.find_partition` with `ModularityVertexPartition` and the resolution parameter γ from the multi-resolution sweep.

### 7.4 Scalability Roadmap

| File Count | Strategy |
|---|---|
| < 5,000 | Full pairwise semantic matrix feasible; skip FAISS |
| 5,000 – 50,000 | Current architecture (FAISS kNN + Sparse SNF + Leiden) |
| 50,000 – 200,000 | Switch to hnswlib incremental index; use sketched modularity |
| > 200,000 | Consider distributed igraph or GVE-Leiden (403M edges/s on server) |

For Curator's current scope (30k files), the current architecture is correct. Design with the scalability roadmap in mind.

---

## 8. Prior Art — Papers, Repos, and What Curator Does Differently

### 8.1 Personal File Organization

**Indaleko: The Unified Personal Index** (Mason, UBC PhD thesis, arXiv:2602.20507, 2025)
- Graph database (ArangoDB) integrating temporal + spatial + activity metadata for 31M files across 8 storage platforms.
- Enables NL queries like "photos near conference venue last spring."
- DOI: not yet assigned (arXiv preprint)
- **Difference from Curator:** Indaleko is a retrieval system — it finds files on demand. Curator is an *organization* system — it groups files proactively and stages decisions. Indaleko requires explicit queries; Curator surfaces context groups without user prompting.

**FileGram** (arXiv:2604.04901, April 2026)
- Grounds AI agent personalization in filesystem behavioral traces.
- Three memory channels: procedural, semantic, episodic — learned from atomic file operations.
- **Difference from Curator:** FileGram personalizes AI agent behavior from file traces; it does not reorganize files or detect contexts. No community detection. Curator uses behavioral traces as one of six edge signals in a clustering architecture.

**AIOS-LSFS** (ICLR 2025, arXiv:2410.11843)
- LLM-based semantic file system. CRUD operations via natural language. Vector database index for semantic retrieval.
- Uses all-MiniLM-L6-v2 embeddings (same as Curator R3).
- Achieves 15% retrieval accuracy improvement + 2.1× speed over traditional FS.
- No graph, no community detection, no context grouping.
- **Difference from Curator:** LSFS is a semantic layer on top of the filesystem for LLM agent use. Curator is a human-facing organizer that builds a context graph and makes group-level decisions for the user.

**HippoCamp** (arXiv:2604.01221, April 2026)
- Benchmark for memory-augmented agents on personal computers (2K+ real files, 581 QA pairs).
- Evaluates context-aware reasoning over multimodal personal file systems.
- Best commercial models achieve only 48.3% on user profiling tasks.
- **Difference from Curator:** HippoCamp is a benchmark/evaluation dataset. It does not provide a file organization system. Its low scores validate Curator's approach: brute-force LLM retrieval fails; structured context graphs are needed.

### 8.2 Graph Clustering and Community Detection

**PathSim** (Sun et al., VLDB 2011, DOI: 10.14778/3402707.3402736)
- Meta-path-based similarity in Heterogeneous Information Networks.
- Curator uses PathSim normalization for NER entity edges and provenance edges.

**SNF** (Wang et al., Nature Methods 2014, DOI: 10.1038/nmeth.2810)
- Similarity Network Fusion via iterative diffusion for multi-modal data.
- GitHub: github.com/rmarkello/snfpy (Python implementation)
- Curator uses Sparse SNF (K=20 NN per view) rather than full N×N SNF.

**From Louvain to Leiden** (Traag et al., Scientific Reports 2019, arXiv:1810.08473)
- Guarantees well-connected communities; refinement phase fixes Louvain disconnection bug.
- GitHub: github.com/vtraag/leidenalg
- Curator uses Leiden exclusively.

**Constrained Clustering — COP-KMeans** (Wagstaff et al., ICML 2001)
- Must-link/cannot-link constraints as hard constraints in k-means.
- Curator extends to graph clustering via negative edge weights for cannot-link.

**PCK-Means** (Basu et al., SDM 2004)
- Soft pairwise constraints in k-means objective.
- GitHub: github.com/datamole-ai/active-semi-supervised-clustering
- Curator adapts the soft-constraint philosophy for Leiden via weight modification.

**DBCV** (Moulavi et al., SDM 2014, DOI: 10.1137/1.9781611973440.96)
- Density-based clustering validation index, range [-1, 1].
- GitHub: github.com/christopherjenness/DBCV and github.com/Kaufman-Lab-Columbia/k-DBCV
- Curator uses DBCV to measure coherence drop in Context Birth Detection Step 5.

### 8.3 Concept Drift

**Concept Drift Adaptation in Text Stream Mining** (Garcia et al., 2024, arXiv:2312.02901)
- Systematic review of 48 papers (2018–2024).
- Drift taxonomy: feature drift, real drift (topic drift), semantic shift, virtual drift.
- ADWIN and Page-Hinkley are the most widely validated algorithms for streaming detection.
- Curator uses ADWIN for context birth detection (detecting when incoming files drift away from existing community centroids).

**ADWIN** (Bifet & Gavalda, SDM 2007)
- Adaptive windowing algorithm. Detects drift when two sub-windows have significantly different means.
- Python: `from river.drift import ADWIN`

### 8.4 Folder Classification

**DRAGON** (arXiv:2602.09071, Feb 2026)
- Robust classification for large software repository collections using only file/directory names and README.
- Achieves 60.8% F1@5 on repository type classification from structure alone.
- Demonstrates that filename and directory structure are powerful signals — even without reading content.
- **Relevance to Curator:** Validates using filename pattern edges (Signal 2.6) as a meaningful signal. However, DRAGON classifies repository TYPE (library, app, tool, etc.), while Curator identifies personal CONTEXT (course, project, phase) — different classification targets.

### 8.5 Topic Segmentation

**GraphSeg** (Glavas et al., 2016)
- Graph-based topic segmentation. Nodes = sentences, edges = semantic relatedness. Maximal cliques = coherent segments.
- Curator borrows the insight: within-context sub-community detection (changing state) is analogous to topic segmentation.

---

## 9. Design Decisions — Numbered and Actionable

**D1:** Graph architecture is a sparse kNN backbone (k=20, FAISS IVFFlat) with 5 overlay edge types. Use `scipy.sparse.csr_matrix`. Community detection via Leiden (leidenalg).

**D2:** Semantic edge threshold θ = 0.60 for all-MiniLM-L6-v2 embeddings. Gravity correction applied during Leiden refinement only.

**D3:** NER edge requires ≥2 shared entities after noise filtering (remove entities in >50% of cluster files). Weight = PathSim Jaccard.

**D4:** Behavioral co-access session boundary = 30-minute gap. Raw timestamps in memory only; aggregated counts to SQLite nightly. Half-life decay 30 days after 90-day age.

**D5:** Provenance edge from `kMDItemWhereFroms` parsed to `netloc + first_path_component`. Fixed weight 0.8 (specific path) or 0.4 (domain only).

**D6:** Temporal proximity is additive bonus only; never creates standalone edges. Gate: edge must already exist from another signal.

**D7:** Filename pattern edge via regex (course codes + 4-char prefix). Weight = 0.3.

**D8:** Composite weight = weighted sum: semantic 40%, NER 25%, behavioral 20%, provenance 10%, temporal 3%, filename 2%.

**D9:** Sparse SNF with K=20, 10 iterations. Memory: ~50 MB total. Use `snfpy` library adapted for sparse matrices.

**D10:** Context Birth Detection: 7-step algorithm with ADWIN (delta=0.002, `river` library). DBCV coherence test only when signals conflict. Decision threshold: ≥3 votes → new context; 1–2 votes + high temporal gap → Review Hub; 0 votes → extend.

**D11:** Cannot-link constraints stored in `boundary_constraints` SQLite table, enforced via negative edge weights (-1.0) in igraph before Leiden run. Never deleted without explicit user action.

**D12:** Louvain resolution γ: multi-resolution sweep at [0.5, 0.75, 1.0, 1.25, 1.5]; select self-consistency plateau. Expected optimal: 0.8–1.1 for personal file collections.

**D13:** Folder coherence = 0.35·MPS + 0.20·EC + 0.15·TSR + 0.20·SCD + 0.10·FTC. Thresholds: ≥0.75 coherent; 0.50–0.74 uncertain (Review Hub); <0.50 mixed; <0.30 chaotic.

**D14:** Context lifecycle: 7 states (new, active, stable, changing, dormant, finished, archived). FLRS decay half-life 30 days for context-level score. Nightly cron for transitions.

**D15:** Leiden exclusively (never Louvain). Hierarchical sub-community detection for `changing` state: run Leiden on community subgraph with γ=1.5 to expose sub-topics.

**D16:** For incremental updates (delta files from R1 FSEvents), use hnswlib for online insert. Rebuild full FAISS index weekly in background.

**D17:** ADWIN per community instance. Each community that has been `active` or `stable` for >7 days gets its own ADWIN detector, seeded with the cosine similarity distribution of its existing files relative to their centroid.

**D18:** `changing` state detection requires: (a) new files in last 14 days, AND (b) ≥3 new entities not in original entity set OR ≥2 internal Leiden sub-communities.

**D19:** Academic calendar heuristic: if incoming files contain course codes AND current month is Sept/Oct or Jan/Feb, increment new-context vote by 1 in CBD algorithm.

**D20:** View weights in composite edge formula (D8) are configurable parameters. Future R layer will learn optimal weights from user feedback on group assignments.

---

## 10. Open Questions — Genuinely Unresolved

**OQ1: Optimal gravity correction formula.** The `sim - 0.1·log(cluster_size)` formula is heuristic. No paper validates this specific functional form for file clustering. Should the coefficient be 0.05? 0.15? Depends on the distribution of community sizes in Curator's target collections. Requires empirical tuning on a held-out dataset of labeled personal file collections.

**OQ2: Optimal ADWIN delta for file streams.** The standard delta=0.002 (99.8% confidence) is calibrated for high-frequency data streams (sensor data, web traffic). File arrival streams are low-frequency (10–100 files/day). Lower frequency means ADWIN's sliding window may take days to accumulate enough samples for reliable drift detection. Alternative: use Page-Hinkley with tunable threshold, or a batch-based test (KS-test on embedding distributions) for low-frequency streams.

**OQ3: Sub-context hierarchy.** Section 1 defines single-level communities. But EPL342 naturally has Lectures, Assignments, Projects as sub-contexts. How deep should the hierarchy go? Two levels (context / sub-context) may be sufficient. Three levels (context / sub-context / sub-sub-context) may create cognitive overhead. No PIM paper has studied user tolerance for hierarchical context depth.

**OQ4: Behavioral edges across devices.** If the user has files on both a MacBook and an iPhone (iCloud), co-access patterns are device-specific. Files opened on iPhone are not captured by macOS FSEvents. Cross-device behavioral edges are currently impossible without an iOS companion app. This limits behavioral edge coverage for users with multi-device workflows.

**OQ5: Provenance key collisions.** `kMDItemWhereFroms` is empty for files transferred by AirDrop, USB, or email attachment. ~30% of personal files may have no provenance URL. Provenance edges will be sparse. Is the 10% view weight in D8 still appropriate when 70% of files lack provenance, or should it be downweighted to 5%?

**OQ6: DBCV scalability in Step 5 of CBD.** DBCV requires a mutual reachability distance graph (similar to HDBSCAN's core distances). For a community of 200 files, this is manageable (~200² = 40k pairs). For a community of 2,000 files (a large long-running project), computing DBCV is expensive. The `k-DBCV` implementation (Kaufman Lab) is faster but still O(n²) in the worst case. An approximate DBCV using sampled pairs may be needed for large communities.

**OQ7: NER for non-English files.** Many students have course materials in Greek or other languages. spaCy `en_core_web_sm` performs poorly on non-English text. Apple NaturalLanguage supports Greek (NLLanguage.greek) for basic NER but has lower accuracy. Consider `spaCy el_core_news_sm` for Greek content — adds a 12 MB model download.

**OQ8: Graph persistence between runs.** The context graph G is recomputed from scratch on first run. Incremental delta updates (D16) handle new files. But if the user renames or moves a large number of files without Curator being active (e.g., manual reorganization during Curator downtime), reconciling the graph state with the new filesystem state is non-trivial. A "graph reconciliation" algorithm for mass moves is needed.

**OQ9: Cannot-link constraint cascade.** If context A and context B are cannot-linked, and a new context C is born that shares significant content with both A and B, should the A-B cannot-link constraint prevent C from being linked to both? Or should C be evaluated independently? This requires a constraint propagation rule that does not currently exist in the design.

**OQ10: User tolerance for false context births.** The CBD algorithm (D10) may over-trigger, creating too many granular contexts (e.g., separating "EPL342 Week 1" from "EPL342 Week 2" as different contexts). The Review Hub is the safety valve, but if users get too many "is this a new context?" prompts, they will dismiss them without reading. Maximum 1 "new context?" prompt per week per domain is a tentative UX constraint that needs validation.

---

## Implementation Checklist (R4 Deliverables for Engineering)

- [ ] `ContextGraph` class: igraph wrapper + sparse matrix management
- [ ] `FAISSIndex` class: IVFFlat for batch + hnswlib for incremental
- [ ] `EdgeBuilder` class: 6 edge type constructors + composite weight calculator
- [ ] `SparseSnf` class: scipy.sparse SNF implementation (5 views, 10 iterations)
- [ ] `LeidenRunner` class: multi-resolution sweep, constraint injection
- [ ] `ContextBirthDetector` class: 7-step CBD algorithm + ADWIN per community
- [ ] `FolderCoherenceScorer` class: 5-signal formula
- [ ] `ContextLifecycleManager` class: state machine + nightly cron
- [ ] `BoundaryConstraintStore` class: SQLite persistence for cannot/must-link
- [ ] `ReviewHubQueue` class: interface for flagged contexts/folders

---

*This document is the canonical R4 specification for Curator. All Layer 1 decisions (R1–R3) are treated as finalized inputs. R4 decisions take precedence over any prior Curator implementation that uses the old SNF+Louvain+individual-proposal architecture.*
