# 41 — Organizational Burden Index (OBI): Measuring the Cognitive Cost of a Filesystem
_Date: 2026-05-31_

## Core Idea

How disorganized is a user's filesystem — right now, as a single number? No existing PIM system can answer this question. There are subjective surveys ("how hard is it to find your files?"), there are task-based studies (measure retrieval time in the lab), and there are component-level metrics (folder count, depth). But there is no **computable, continuous scalar metric** that quantifies the total cognitive burden imposed by a file system's organization on its user.

The Organizational Burden Index (OBI) is that metric. It synthesizes five information-theoretic and behavioral components into a single score, operationalizing the theoretical construct of "PIM Burden" first named by Dinneen & Julien (JASIST 2022).

---

## Prior Art

**Dinneen & Julien (JASIST 2022)** — "Personal Information Management Burden: A Concept Analysis" (doi:10.1002/asi.24692). This is the defining paper naming PIM Burden as a theoretical construct. They define it as the cognitive and temporal cost of maintaining, retrieving, and managing personal information. However, they explicitly do not operationalize it as a computable metric — the paper calls for future work to do so. OBI is that future work.

**Bergman et al. (2010)** — Measured retrieval failure rates across organizational strategies. Closest empirical work, but requires controlled lab conditions and task-based measurement. Not a passive, always-on metric.

**H(location|intent) — File 29** — Measures one component of OBI: how much uncertainty remains about file location given user intent. OBI extends this single metric into a composite.

**DBCV (Moulavi et al., 2014)** — Density-Based Cluster Validation for HDBSCAN internal quality. Measures cohesion and separation of clusters, but not cognitive accessibility for the user.

**Folders & Files Complexity Metrics (Boardman & Sasse, CHI 2004)** — Introduced folder tree depth and breadth as complexity measures. These are components of OBI's folder depth entropy term, but were never combined into a composite cognitive burden metric.

No published work combines entropy-theoretic, clustering-theoretic, and behavioral signals into a single computable organizational quality score.

---

## Formal Definition

OBI is a weighted linear combination of five component scores, each normalized to [0, 1] where 0 = no burden and 1 = maximum burden:

```
OBI = w₁·H̃(L|I) + w₂·FDE + w₃·OR + w₄·NDD + w₅·(1 - FLRS̄)
```

### Component 1: Normalized Conditional Entropy H̃(L|I)

H(location|intent) from File 29, normalized by H_max = log₂(|folders|):

```
H̃(L|I) = H(L|I) / log₂(|F|)    ∈ [0, 1]
```

where |F| = total number of folders. H̃(L|I) = 0 means knowing intent perfectly determines location; H̃(L|I) = 1 means location is independent of intent.

**Estimation:** From Spotlight query logs + subsequent navigation events (as described in File 29).

### Component 2: Folder Depth Entropy (FDE)

Measures whether files are distributed across a pathologically deep or unbalanced folder hierarchy:

```
FDE = H(depth distribution of all files) / log₂(max_depth)
```

Where depth(f) = number of folder levels from root to file f's containing folder. A flat filesystem (all files at depth 1) has FDE = 0. A filesystem with files at unpredictable depths has FDE close to 1.

**Rationale:** Deeper hierarchies impose higher navigation cost (Boardman & Sasse CHI 2004 showed navigation time scales linearly with folder depth). High entropy of the depth distribution means the user cannot predict how deep to navigate.

### Component 3: Orphan Ratio (OR)

Files not assigned to any meaningful HDBSCAN cluster (noise points) represent organizational ambiguity — they don't belong to any recognizable topic:

```
OR = |{f : cluster(f) = NOISE}| / |total files|    ∈ [0, 1]
```

**Rationale:** Orphan files impose disproportionate search burden because they have no semantic neighbors to help the user locate them.

### Component 4: Near-Duplicate Density (NDD)

Files that are near-identical impose dual burden: they consume space and they create retrieval ambiguity ("which version was the final one?"):

```
NDD = |{(fᵢ, fⱼ) : cosine_sim(embed(fᵢ), embed(fⱼ)) > 0.95}| / (|N| choose 2)
```

Approximated efficiently using LSH (locality-sensitive hashing) rather than exhaustive pairwise comparison.

**Rationale:** Near-duplicates directly cause navigation errors in user studies (Shen et al. 2006 on version confusion).

### Component 5: Mean FLRS Decay (1 - FLRS̄)

Files with low FLRS scores are files whose locations the user has likely forgotten — they impose retrieval burden even if they're correctly organized:

```
FLRS̄ = (1/N) Σ_f FLRS(f, t_now)    ∈ [0, 1]
OBI component = 1 - FLRS̄
```

**Rationale:** A filesystem full of files the user has forgotten how to find imposes high burden regardless of how well-organized it structurally is.

### Default Weights

Initial weights before user-study calibration:

| Component | Weight | Rationale |
|---|---|---|
| H̃(L\|I) | w₁ = 0.30 | Primary theoretical contribution |
| FDE | w₂ = 0.20 | Navigation depth directly costs time |
| OR | w₃ = 0.20 | Orphans are high-retrieval-cost files |
| NDD | w₄ = 0.15 | Duplicates cause version confusion |
| 1 - FLRS̄ | w₅ = 0.15 | Forgotten files = high retrieval burden |

Weights are calibrated via regression against self-reported difficulty and NASA-TLX scores in the validation study.

---

## OBI Dashboard

Curator exposes OBI as a persistent health indicator with three views:

**1. Scalar gauge:** Single OBI score (0–100 scale) displayed as a circular gauge with color coding: green (0–30), yellow (31–60), red (61–100).

**2. Time series:** OBI plotted over weeks and months. Curator's interventions (cluster restructuring, FLRS-triggered resurfacing, duplicate resolution) appear as visible drops in the OBI curve.

**3. Decomposition bar:** Shows the five components' contributions to the current OBI. "Your main burden is forgotten files (FLRS component = 0.42)" or "Your folder hierarchy is too deep (FDE component = 0.38)."

This decomposition view enables actionable intervention: if FDE is high, Curator suggests flattening the folder hierarchy. If OR is high, Curator proposes re-clustering orphan files. If NDD is high, Curator flags near-duplicates for user review.

---

## Validation Study Design

**RQ1:** Does OBI correlate with self-reported retrieval difficulty?
- Instrument: 5-item ESM (Experience Sampling Method) survey, sent 3×/day for 2 weeks: "How hard was it to find a file in the last 2 hours?"
- Analysis: Spearman correlation between daily OBI and mean ESM score

**RQ2:** Does OBI predict task-based retrieval time?
- Instrument: Lab session, 20 known-item retrieval tasks (10 own files, 10 unfamiliar files in a provided filesystem)
- Analysis: Linear regression of time-to-find on OBI and its components

**RQ3:** Does OBI decrease as Curator's interventions take effect?
- Design: Longitudinal within-subjects, OBI measured weekly over 8 weeks with Curator active
- Analysis: Linear mixed model, OBI ~ time + (1|participant)

**Sample size:** N = 24 participants for ESM + lab session; N = 20 for longitudinal study. Powered at 80% to detect r = 0.40 correlation (RQ1).

---

## Connection to H(location|intent)

OBI generalizes H(location|intent) in two ways:

1. **H(L|I) is OBI's largest component (w₁ = 0.30)**, so improvements in CP routing quality (which reduce H(L|I)) directly reduce OBI. This creates a formal link between Curator's routing algorithm and the user-facing health metric.

2. **OBI adds behavioral and structural dimensions** that H(L|I) alone cannot capture: forgotten files (FLRS component), structural complexity (FDE), orphans (OR), and version clutter (NDD).

If Curator's CP router achieves H(L|I) → 0, OBI's theoretical minimum is:
```
OBI_min = w₂·FDE_min + w₃·OR_min + w₄·NDD_min + w₅·(1 - FLRS̄_max)
```
This is a tractable lower bound, and it shows that even a perfect router cannot eliminate all burden — the user still needs to maintain their filesystem (deduplicate, recall locations, navigate hierarchy). OBI captures this residual burden.

---

## Research Contribution

**Primary:** First computable operationalization of PIM Burden (Dinneen & Julien JASIST 2022) as a continuous, always-on scalar metric. If validated against retrieval difficulty and NASA-TLX, OBI becomes a standard outcome measure for PIM research — all future file organizer papers can report OBI improvement as a comparable metric.

**Secondary:** The decomposition model reveals which organizational pathology dominates for different user types. Filers (organized folder users) may have low FDE but high NDD. Pilers (searchers) may have high OR and high H̃(L|I). OBI decomposition enables targeted interventions.

**Target:** IUI 2027 (as Curator's unified evaluation framework) or JASIST as a standalone measurement paper.

---

## References

- Dinneen, J. D., & Julien, C.-A. (2022). Personal information management burden: A concept analysis. *Journal of the American Society for Information Science and Technology, 73*(8), 1084–1101. https://doi.org/10.1002/asi.24692
- Boardman, R., & Sasse, M. A. (2004). "Stuff goes into the computer and doesn't come out": A cross-tool study of personal information management. *Proceedings of CHI 2004*, 583–590. https://doi.org/10.1145/985692.985766
- Bergman, O., Whittaker, S., Sanderson, M., Nachmias, R., & Ramamoorthy, A. (2010). The effect of folder structure on personal file navigation. *Journal of the American Society for Information Science and Technology, 61*(12), 2426–2441. https://doi.org/10.1002/asi.21415
- Moulavi, D., Jaskowiak, P. A., Campello, R. J. G. B., Zimek, A., & Sander, J. (2014). Density-based clustering validation. *Proceedings of SDM 2014*, 839–847. https://doi.org/10.1137/1.9781611973440.96
- Elsweiler, D., Ruthven, I., & Jones, C. (2007). Towards memory supporting personal information management tools. *Journal of the American Society for Information Science and Technology, 58*(7), 924–946. https://doi.org/10.1002/asi.20575
- Hart, S. G., & Staveland, L. E. (1988). Development of NASA-TLX (Task Load Index): Results of empirical and theoretical research. *Advances in Psychology, 52*, 139–183. https://doi.org/10.1016/S0166-4115(08)62386-9
- Angelopoulos, A. N., Bates, S., Candès, E. J., Jordan, M. I., & Lei, L. (2024). Conformal risk control. *arXiv:2208.02814*. https://arxiv.org/abs/2208.02814
