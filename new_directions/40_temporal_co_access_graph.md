# 40 — Temporal Co-Access Graph: Implicit Project Detection
_Date: 2026-05-31_

## Core Idea

HDBSCAN clusters files by **semantic similarity** — what a file is about. But real work is organized by **projects** — what a file is used for. A tax spreadsheet, a scanned PDF of a receipt, and a Python script that formats financial data are semantically dissimilar but belong to the same project. Semantic clustering cannot detect this. Folder structure often doesn't capture it either.

The Temporal Co-Access Graph (TCAG) detects implicit projects by observing **which files are opened together** within a bounded time window. Files co-accessed frequently and recently form dense subgraphs; community detection on these subgraphs yields project memberships. This is orthogonal to HDBSCAN clustering and captures a dimension of organizational structure that no existing PIM system models.

---

## Prior Art

**CodeScene / Temporal Coupling in Software Engineering:** Adam Tornhill's CodeScene platform (and its academic predecessor, Code Maat) performs temporal coupling analysis on version control commit histories — files committed together frequently are likely to be logically coupled, even if they're in different modules. This is the direct conceptual ancestor of TCAG for source code. No analog exists for general personal file systems.

Tornhill, A. (2015). *Your Code as a Crime Scene*. Pragmatic Bookshelf. — Introduces temporal coupling as a software quality metric.

**Desktop Activity Recognition:** Several systems have inferred "tasks" from application switching patterns (Kaptelinin 1996; Gonzalez & Mark CHI 2004; Iqbal & Bailey CHI 2007). These systems track application context, not individual file co-access, and do not build explicit file-level graphs.

**File Access Pattern Modeling:** USENIX 2001 predictive prefetching work (Griffioen & Appleton) showed that simple last-successor models predict the next file opened with ~72% accuracy, confirming strong temporal structure in file access sequences. But prefetching focuses on I/O latency, not organizational semantics.

**Implicit Task Detection:** Dragunov et al. (UIST 2005) built TaskTracer, which inferred task context from application and window switches. It did not model file co-access at the individual file level and did not build an explicit graph structure.

**No system** in the PIM, UIST, CHI, or IR literature has built a weighted co-access graph over individual files and applied community detection to extract implicit projects.

---

## Graph Construction

### Data Source

macOS Endpoint Security Framework (`ES_EVENT_TYPE_NOTIFY_OPEN`) captures every file open event across all applications, with microsecond timestamps and process identifiers. This is the same data source Curator already uses for FLRS event tracking.

### Session Segmentation

A **work session** is a maximal contiguous period of file activity with no idle gap exceeding τ minutes (recommended τ = 20 min, tunable). Formally:

```
Session s = [t_start, t_end] such that:
  ∀ consecutive events eᵢ, eᵢ₊₁ in s: eᵢ₊₁.timestamp - eᵢ.timestamp < τ
  and the gap before t_start and after t_end exceeds τ
```

Files opened within the same session are candidate co-access pairs.

### Edge Weighting

For files f_i and f_j, the co-access edge weight is:

```
w(fᵢ, fⱼ) = Σ_s [co_accessed(fᵢ, fⱼ, s)] × decay(t_s)
```

Where:
- `co_accessed(fᵢ, fⱼ, s) = 1` if both files were opened in session s, 0 otherwise
- `decay(t_s) = e^(-λ(t_now - t_s))` — recency decay, λ controls half-life (recommended λ = 0.05/day, so sessions from 14 days ago have weight ~0.5)

### Temporal File Coupling Score (TFC)

The TFC score between two files is the normalized edge weight:

```
TFC(fᵢ, fⱼ) = w(fᵢ, fⱼ) / √(degree(fᵢ) × degree(fⱼ))
```

This is structurally identical to the cosine similarity of co-occurrence vectors. TFC(f_i, f_j) ∈ [0, 1] where 1 = always co-accessed, 0 = never co-accessed.

---

## Community Detection: Project Extraction

Given the weighted undirected graph G = (V, E, W) where V = files, E = co-access pairs, W = edge weights, implicit projects are extracted as communities.

**Algorithm choice:** Leiden algorithm (Traag, Waltman & van Eck, 2019) — superior to Louvain for resolution limit and reproducibility. `pip install leidenalg`. For a personal file graph (typically 10K–100K nodes, sparse edges), Leiden runs in seconds.

**Resolution parameter γ:** Controls project granularity. Low γ = few large projects. High γ = many small focused projects. Recommend γ ≈ 1.0 as default, with user-adjustable slider.

**Update schedule:** Rebuild incrementally after each new session. Full recompute weekly or when graph structure changes significantly (measured by normalized cut change > 5%).

---

## Dual-Layer Organization

TCAG introduces a **project layer** orthogonal to the existing **semantic layer** (HDBSCAN):

| Layer | Method | What it captures |
|---|---|---|
| Semantic | HDBSCAN over embeddings | What files are about |
| Project | TCAG + Leiden | What files are used for together |

A file can belong to semantic cluster "Machine Learning Papers" AND project "Grant Application 2026." These are not competing categorizations — they are complementary views. The project layer answers "what am I working on right now?" The semantic layer answers "what is this file?"

Curator's UI can expose both views simultaneously: a "Projects" panel (TCAG-derived) and a "Topics" panel (HDBSCAN-derived).

---

## Key Research Finding Opportunity

**Hypothesis:** User-defined folder structures systematically misrepresent actual project structure. Files that belong to the same implicit project (TCAG community) are typically scattered across 3–5 different folders.

This can be measured empirically:

```
Project-Folder Alignment Score (PFAS) = 
  mean over all TCAG projects P:
    max_f [fraction of P's files in folder f]
```

PFAS = 1.0 means every project's files are in one folder. PFAS = 0.2 means files are scattered across 5 folders on average. We predict PFAS ≈ 0.3–0.4 for typical users — a quantification of how badly folder structures represent actual work.

This would be the first empirical measurement of project-folder misalignment in personal file systems. A study with N=20 participants, logging 4 weeks of file access data, would yield this result with high confidence.

---

## Connection to Other Curator Components

**vs. Markov File Biography (file 06):** Markov biography tracks per-file cluster assignment history. TCAG tracks inter-file co-access. These are orthogonal signals. A file with high Markov entropy (bouncing between clusters) that has high TFC with a stable project group is likely a shared resource (e.g., a style guide used across multiple projects).

**vs. HDBSCAN (semantic clusters):** Project detection can rescue files that HDBSCAN marks as noise (outliers). A file that doesn't fit any semantic cluster may be a project-critical artifact (e.g., a raw data export with no recognizable content type) that TCAG correctly places in a project context.

**vs. FLRS:** Files accessed together in a project context likely have correlated FLRS decay — if you forget where one project file is, you've probably forgotten where the others are too. A "project FLRS" aggregate score could trigger resurfacing of all project files together, not just individual ones.

---

## Research Contribution

1. **First application of temporal coupling analysis to personal file systems** — adapts a software engineering technique (CodeScene) to the PIM domain.

2. **Temporal File Coupling (TFC) score** — a novel relational metric between files, measuring project co-membership without requiring semantic similarity or explicit user annotation.

3. **Project-Folder Alignment Score (PFAS)** — first empirical measurement of the gap between user-defined folder structure and actual project-based work organization.

4. **Dual-layer organization model** — semantic layer (HDBSCAN) + project layer (TCAG) as complementary views of the same filesystem. No PIM system has proposed this architecture.

**Target venue:** IUI 2027 (as part of the main Curator paper) or UIST 2027 as a standalone systems contribution.

---

## References

- Gonzalez, V. M., & Mark, G. (2004). "Constant, constant, multi-tasking craziness": Managing multiple working spheres. *Proceedings of CHI 2004*, 113–120. https://doi.org/10.1145/985692.985707
- Griffioen, J., & Appleton, R. (1994). Reducing file system latency using a predictive approach. *USENIX Annual Technical Conference*, 197–207. https://www.usenix.org/conference/usenix-summer-1994/reducing-file-system-latency-using-predictive-approach
- Iqbal, S. T., & Bailey, B. P. (2007). Leveraging characteristics of task structure to predict the cost of interruption. *Proceedings of CHI 2007*, 741–750. https://doi.org/10.1145/1240624.1240741
- Kaptelinin, V. (1996). Activity theory: Implications for human-computer interaction. In B. Nardi (Ed.), *Context and Consciousness*. MIT Press.
- Dragunov, A. N., Dietterich, T. G., Johnsrude, K., McLaughlin, M., Li, L., & Herlocker, J. L. (2005). TaskTracer: A desktop environment to support multi-tasking knowledge workers. *Proceedings of IUI 2005*, 75–82. https://doi.org/10.1145/1040830.1040855
- Tornhill, A. (2015). *Your Code as a Crime Scene: Use Forensic Techniques to Arrest Defects, Bottlenecks, and Bad Design in Your Programs*. Pragmatic Bookshelf.
- Traag, V. A., Waltman, L., & van Eck, N. J. (2019). From Louvain to Leiden: Guaranteeing well-connected communities. *Scientific Reports, 9*, 5233. https://doi.org/10.1038/s41598-019-41695-z
- Blondel, V. D., Guillaume, J.-L., Lambiotte, R., & Lefebvre, E. (2008). Fast unfolding of communities in large networks. *Journal of Statistical Mechanics: Theory and Experiment, 2008*(10), P10008. https://doi.org/10.1088/1742-5468/2008/10/P10008
