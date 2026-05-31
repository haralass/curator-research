# GhostUMAP2 × Markov File Biography Cross-Validation Theory

**Do Unstable UMAP Points Correspond to High-Markov-Entropy Files?**

*Research Document — Curator Project*
*Date: 2026-05-31*

---

## 1. Motivation and Core Hypothesis

Curator maintains two independently derived instability signals. The first, **GhostUMAP2** (arXiv:2507.17174), interrogates the geometric stability of a file's position in the UMAP embedding by injecting ghost duplicates and measuring how far they scatter. The second, **Markov File Biography**, observes a file's cluster-assignment history over time and computes the stationary entropy of the transition chain — a measure of how restlessly a file moves between conceptual neighborhoods.

These signals are derived from entirely different sources of evidence: GhostUMAP2 is a snapshot-time geometric probe, while Markov entropy is a longitudinal behavioral trace. If they nonetheless converge on the same files, that convergence is not coincidental — it implies a shared underlying cause: **the file occupies a genuine ambiguity zone in the feature space**, where no cluster can claim it without contestation. Confirming this convergence would give Curator a doubly validated, theoretically grounded concept of "ambiguous file" that is robust to the failure modes of either metric alone.

The central hypothesis is:

> **H₀:** There is no statistically significant rank correlation between a file's GhostUMAP2 instability score and its Markov stationary entropy.
>
> **H₁:** Files with higher GhostUMAP2 instability scores exhibit significantly higher Markov stationary entropy (positive monotonic relationship).

---

## 2. GhostUMAP2 — Formal Definitions and Mechanics

### 2.1 The (r, d)-Stability Definition

GhostUMAP2 formalizes UMAP instability through the concept of *ghost duplicates*. For each data point p with initial UMAP projection position x₀, the algorithm places M ghost duplicates within a circle of radius r centered on x₀ in the initial projection space. After running UMAP optimization to convergence, the final positions of these ghosts are recorded. Point p is defined as **(r, d)-stable** if and only if all M ghost final positions remain confined within a circle of radius d centered on the final embedding of p.

Formally, let G(p) = {g₁, g₂, ..., gₘ} be the ghost set for point p, and let φ(gᵢ) denote the final UMAP projection of ghost gᵢ. Then:

```
p is (r,d)-stable ⟺ max_i ||φ(gᵢ) − φ(p)|| ≤ d
```

A point is **(r, d)-unstable** when at least one ghost escapes the d-ball — meaning the stochastic elements of UMAP (random initialization of negative sample positions, SGD noise) cause meaningfully different outcomes for geometrically nearby initializations. The paper estimates roughly **~1% of points** are (r, d)-unstable across typical datasets, making instability a rare but structurally meaningful signal.

### 2.2 Sources of UMAP Stochasticity

The instability probed by GhostUMAP2 arises from two distinct sources within UMAP's algorithm:

1. **Initial projection positions**: UMAP initializes with spectral embedding or random, and small differences in starting positions can lead to different local minima in the embedding optimization landscape.
2. **Negative sampling**: UMAP's cross-entropy loss is optimized via SGD with stochastic negative samples. Points at density boundaries — where the attractive force from cluster members and the repulsive force from non-members are nearly balanced — are most sensitive to the particular negative samples drawn.

This second mechanism is theoretically crucial: **points at density boundaries have approximately equal attractive pull from two or more clusters**. The gradient signal is ambiguous, and whichever cluster wins the tug-of-war in a given run depends on sampling noise. This is exactly the geometric signature of a file that genuinely belongs to two conceptual neighborhoods.

### 2.3 Adaptive Ghost Dropping

GhostUMAP2 introduces an adaptive dropping scheme that reduces computational overhead by up to 60% compared to exhaustive ghost placement, while maintaining approximately 90% detection accuracy for unstable points. This is achieved by first running a cheap stability pre-screening (few ghosts, coarse d threshold) and only placing the full ghost set M for points that pass the pre-screen. For Curator's 6-month UMAP refit cycle, this means the overhead of ghost injection is tractable even over a large file corpus.

---

## 3. HDBSCAN Soft Clustering and Its Relationship to GhostUMAP2 Instability

HDBSCAN already natively distinguishes core points from border points through its soft clustering membership vectors. The mathematical mechanism is instructive for understanding why these two taxonomies should overlap.

### 3.1 HDBSCAN Membership Probability

HDBSCAN's soft clustering assigns each point p a vector **s(p) ∈ [0,1]^K** where K is the number of clusters, and s(p)_k represents the probability that p belongs to cluster k. This is computed via a Bayesian combination of:

- **Distance-based probability**: Inverse distance from p to the set of exemplar points of cluster k, scaled by lambda (persistence) values from the condensed tree.
- **Outlier probability**: The ratio of the merge height at which p joins cluster k to the maximum lambda value, capturing how peripherally p entered the cluster.

**Core points** — those that persist across a wide range of lambda values, deep within a dense region — receive membership vectors that are heavily concentrated on a single cluster: s(p)_k ≈ 1.0 for one k, near 0 for all others. **Border points** display a desaturated pattern: membership mass spread across two or more clusters, often s(p)_k ∈ [0.2, 0.6] for multiple k simultaneously.

### 3.2 The Overlap with GhostUMAP2 Instability

The theoretical connection is direct. A border point in HDBSCAN has balanced membership across clusters. In the UMAP embedding of that data, such a point sits in the low-density gap between cluster regions — precisely the geometry where UMAP's SGD produces inconsistent gradient signals. GhostUMAP2 would detect this as instability: ghosts initialized on one side of the decision boundary may converge toward Cluster A, while ghosts on the other side converge toward Cluster B, producing high ghost displacement.

The empirical prediction is therefore: **GhostUMAP2-unstable points should heavily overlap with HDBSCAN border points** (those with membership score < 0.7 for their assigned cluster). Curator can verify this overlap cheaply during each UMAP refit by cross-tabulating (r,d)-stability flags against HDBSCAN soft membership scores.

---

## 4. Markov File Biography — Entropy Formalism

### 4.1 The Transition Chain

Each file f in Curator accumulates a cluster-assignment sequence over time: **[c₁, c₂, c₃, ..., cₙ]** where cᵢ ∈ {1, ..., K} is the cluster assigned at the i-th UMAP refit epoch. This sequence is modeled as a first-order Markov chain. The transition matrix **T** has entries:

```
T[j][k] = P(cₜ₊₁ = k | cₜ = j) = count(j → k) / count(j → *)
```

For files with long histories, T is estimated from empirical transition frequencies. For young files with sparse histories, a Dirichlet prior (Bayesian smoothing) can regularize the estimates.

### 4.2 Stationary Distribution and Entropy

For an ergodic Markov chain with transition matrix T, the stationary distribution **π** satisfies πT = π and Σ πᵢ = 1. The **stationary entropy** is:

```
H(π) = −Σ_{k=1}^{K} πₖ log₂ πₖ   [bits]
```

This quantity has a natural interpretation in our context:

- **H(π) ≈ 0**: The chain concentrates almost all mass on one state — the file consistently lands in the same cluster regardless of UMAP refit. The file is a stable, unambiguous cluster member.
- **H(π) = log₂(K)** (maximum): The file is uniformly distributed across all clusters — it has no consistent conceptual home. This is a maximally ambiguous file.

Note that H(π) is distinct from the *entropy rate* of the Markov chain (which also depends on transition probabilities), though both are related. The stationary entropy measures long-run distributional uncertainty; the entropy rate additionally captures transition-by-transition unpredictability.

### 4.3 Relationship Between Entropy and Mixing Time

There is a known mathematical relationship between stationary entropy and mixing time that strengthens the theoretical interpretation. The mixing time τ_mix of a Markov chain is the number of steps required for the chain's distribution to be within ε (in total variation distance) of the stationary distribution π. For chains with high stationary entropy (diffuse π), the chain tends to mix faster — the cluster-assignment distribution converges quickly because no single state is dominant enough to act as an attractor that slows convergence.

For Curator's purposes, high-H(π) files are characterized by fast mixing: starting from any initial cluster assignment, the file's distribution over clusters rapidly approaches the uniform-ish stationary distribution. This is a signature of a file with no strong gravitational pull toward any particular cluster — it is structurally ambiguous. Low-H(π) files have slow mixing, with the chain trapped near one absorbing state, reflecting a file with a clear conceptual identity that only rarely gets mis-assigned.

---

## 5. Theoretical Link — Can the Correspondence Be Formalized?

### 5.1 Informal Argument

The informal causal chain is:

1. File f has features that place it near the boundary of two clusters in high-dimensional space.
2. When UMAP projects these features, f's position in 2D is in the low-density inter-cluster region, making it sensitive to UMAP's stochastic gradient dynamics → **GhostUMAP2 instability**.
3. When HDBSCAN clusters the UMAP embedding, f's boundary position means its cluster assignment is sensitive to local density fluctuations across refit epochs (as the corpus evolves, new files shift the density landscape) → **cluster assignment variability** → **high Markov entropy**.

The two metrics share a root cause: the file is a genuine boundary member. GhostUMAP2 observes the consequence in projection space at a single epoch; Markov entropy observes the consequence in assignment space across epochs.

### 5.2 Toward Formalization

Let ρ(f) denote the HDBSCAN soft membership entropy for file f at epoch t:

```
ρ(f, t) = −Σ_k s(f,t)_k log₂ s(f,t)_k
```

This is the per-epoch cluster assignment uncertainty. If we assume the cluster assignment cₜ is drawn from the soft membership distribution s(f,t) (i.e., the stochastic assignment rule is the soft membership vector), then the Markov transition matrix T inherits its uncertainty from the temporal fluctuations in s(f,t).

Under the assumption that s(f,t) itself follows a stationary process (the file's relative position in feature space evolves slowly), the expected Markov stationary entropy H(π_f) is bounded below by the expected per-epoch soft entropy:

```
E[H(π_f)] ≥ E_t[ρ(f, t)]   (Jensen-type argument)
```

This bound is not tight in general, but it establishes the directional relationship: files with consistently high per-epoch soft entropy (HDBSCAN border points) will tend to have high Markov stationary entropy. GhostUMAP2 instability, which correlates with HDBSCAN border membership (Section 3.2), therefore transitively correlates with Markov entropy.

A sharper formalization would require characterizing how the density landscape shift between epochs (as the corpus evolves) maps to s(f,t) fluctuations, which depends on the corpus dynamics model. This is left as an open theoretical problem.

---

## 6. Cross-Validation Design

### 6.1 Metrics to Compute per File

For each file f in the corpus, compute:

| Metric | Symbol | Source |
|--------|--------|--------|
| GhostUMAP2 instability score | I(f) | Max ghost displacement, averaged over M ghosts |
| HDBSCAN soft membership entropy | ρ(f) | Per-epoch, averaged over recent K epochs |
| Markov stationary entropy | H(f) | From transition matrix T estimated over full history |
| HDBSCAN membership score | s(f) | Max-cluster soft membership probability |

GhostUMAP2 yields a continuous score (average ghost displacement magnitude), not just a binary stable/unstable flag. Use the raw score for correlation analysis.

### 6.2 Statistical Tests

**Primary test: Spearman rank correlation** between I(f) and H(f) across all files with sufficient history (≥ 5 refit epochs). Spearman is preferred over Pearson because:

- I(f) and H(f) are not expected to be linearly related; the monotonic relationship is the theoretical prediction.
- Both metrics may have heavy-tailed distributions (most files are stable; a minority are highly unstable).

Compute ρ_s(I, H) with a two-tailed hypothesis test against H₀: ρ_s = 0. A sample size of n ≥ 150 files is the practical minimum for reliable Spearman estimation (to detect moderate correlations ρ_s ≈ 0.3 at α = 0.05, 80% power); n ≥ 300 is preferable for stable confidence intervals.

**Secondary tests:**

- Spearman ρ_s(I, ρ_f) and ρ_s(H, ρ_f): Correlate both primary metrics against HDBSCAN per-epoch soft entropy as a common mediator. If both primary metrics correlate strongly with the mediator but moderately with each other, the indirect pathway through HDBSCAN ambiguity is supported.
- **Top-decile overlap rate**: Compute the fraction of files in the top 10% of I(f) that also appear in the top 10% of H(f). Under the null (independence), expected overlap is 1%; under H₁, expect 30–50%+ overlap. This is a more actionable metric for the engineering use case.
- **Point-biserial correlation**: Binarize I(f) using the paper's (r,d)-unstable threshold; correlate binary instability with continuous H(f).

### 6.3 Confound Control

Several confounds must be controlled:

- **File age**: Young files have sparse Markov histories → noisier H(f). Stratify by file age (number of epochs observed) and test within strata.
- **Cluster size**: Files in small clusters may be more frequently re-assigned due to cluster dissolution across epochs, inflating H(f) for reasons unrelated to boundary membership. Include cluster size as a covariate.
- **Feature drift**: Files whose content changes (e.g., living documents being edited) will exhibit high Markov entropy due to genuine conceptual evolution, not boundary ambiguity. Where possible, flag files with substantial content modifications between epochs and exclude or separately analyze them.

---

## 7. Why Convergence Matters — Double Validation of Ambiguity

### 7.1 The Complementary Failure Modes

GhostUMAP2 has characteristic failure modes. It can flag a file as unstable not because the file is ambiguous but because the UMAP projection in its local neighborhood is poorly conditioned — for example, in very sparse regions where few neighbors constrain the embedding. These are false positives in the sense that the file has a clear identity but a poorly supported projection geometry.

Markov entropy has its own failure modes. A file that is genuinely stable but belongs to a cluster that periodically splits and reforms across refit epochs will accumulate artificial transition history, inflating H(f). Similarly, a file in a very small cluster may be repeatedly reassigned when that cluster fails the minimum-size threshold.

**When both metrics agree — when a file is both (r,d)-unstable AND has high H(π) — the probability of both failure modes simultaneously affecting the same file is small.** The double signal is therefore a strong indicator of genuine ambiguity, not metric artifact.

### 7.2 Decision Logic

The practical routing rule for Curator would be:

```
IF I(f) > τ_ghost AND H(f) > τ_markov:
    → CONFIDENTLY AMBIGUOUS → route to human review
ELIF I(f) > τ_ghost XOR H(f) > τ_markov:
    → TENTATIVELY AMBIGUOUS → monitor for one more epoch
ELSE:
    → STABLE → no action
```

Thresholds τ_ghost and τ_markov can be set at the 90th percentile of their respective distributions across the corpus, calibrated to keep the "confidently ambiguous" bin at a manageable fraction of total files (target: 1–3%).

---

## 8. Implementation Plan

### 8.1 GhostUMAP2 Integration (At UMAP Refit Time)

GhostUMAP2 runs during Curator's 6-month UMAP refits. The implementation steps are:

1. After fitting the base UMAP embedding on the full corpus, record all point positions.
2. For each file f, generate M = 20 ghost duplicates with initial positions sampled uniformly within a circle of radius r = 0.05 (in normalized UMAP space) around f's initial position.
3. Run ghost optimization to convergence.
4. Compute I(f) = (1/M) Σᵢ ||φ(gᵢ) − φ(f)||.
5. Flag f as (r,d)-unstable if max_i ||φ(gᵢ) − φ(f)|| > d = 0.15.
6. Store (I(f), binary_flag) in the file metadata database.

Use the adaptive dropping scheme: pre-screen with M = 3 ghosts. Only run full M = 20 ghost set for files that fail the pre-screen. This reduces compute by ~60% with ~90% recall of truly unstable points.

### 8.2 Markov File Biography Tracking (Continuous)

Markov tracking runs continuously as the cluster assignments update:

1. After each UMAP refit, record cₜ(f) for each file f.
2. Maintain the empirical transition matrix T(f) with Bayesian smoothing: T[j][k] += α (Dirichlet prior, α = 0.1) for all (j,k) pairs, plus empirical counts.
3. Compute stationary distribution π(f) by solving πT = π (e.g., via power iteration or left eigenvector of T).
4. Compute H(f) = −Σ_k π(f)_k log₂ π(f)_k.
5. Store H(f) in file metadata, updated after each refit.

Files with fewer than 3 observed epochs use a uniform prior π = (1/K, ..., 1/K), which yields maximum entropy H = log₂(K) — they are treated as maximally uncertain until sufficient history accumulates.

### 8.3 Combined Ambiguity Score

To combine both signals into a single scalar for routing decisions:

```
A(f) = w₁ · rank_normalize(I(f)) + w₂ · rank_normalize(H(f))
```

where rank normalization maps each metric to [0, 1] via its empirical rank among all files. Initial weights w₁ = w₂ = 0.5 treat both signals equally. After the cross-validation study (Section 6), weights can be adjusted to reflect each metric's predictive validity — if Spearman ρ_s(I, ground_truth) > ρ_s(H, ground_truth), upweight I(f).

Ground truth for weight calibration can be obtained via human review: for a sample of files flagged by the combined score, ask the user whether the file is genuinely ambiguous in category. Treat human judgment as the calibration signal.

### 8.4 Logging and Observability

Both scores should be logged in a time-stamped audit table to enable retrospective analysis:

```
file_id | epoch | ghost_score | ghost_unstable | markov_entropy | combined_score
```

This table enables post-hoc computation of the Spearman correlation across any epoch window, analysis of how A(f) evolves as file histories lengthen, and debugging of cases where the two metrics disagree.

---

## 9. Expected Results and Theoretical Predictions

Based on the theoretical analysis, the following quantitative predictions are made:

1. **Spearman ρ_s(I, H) ≈ 0.30–0.50** across files with ≥ 5 epoch histories. A moderate positive correlation is expected; a very high correlation (> 0.7) would be surprising given the different failure modes and time scales of the two metrics.

2. **Top-decile overlap ≈ 25–45%**: Among the top 10% most unstable files by GhostUMAP2, roughly 25–45% should also appear in the top 10% by Markov entropy. This is 2.5–4.5x the null-hypothesis expectation of 10%.

3. **HDBSCAN mediator correlation ρ_s ≈ 0.50–0.70**: The per-epoch HDBSCAN soft membership entropy ρ(f) should correlate more strongly with each primary metric than the primary metrics correlate with each other, supporting the mediation hypothesis.

4. **File age effect**: Correlation should strengthen with file age. For files with ≥ 10 epoch histories, expect ρ_s(I, H) to approach the upper end of the predicted range, as Markov entropy estimates stabilize.

---

## 10. Open Questions and Limitations

**Q1: Does GhostUMAP2 instability persist across refits?** A file that is (r,d)-unstable at epoch t may be stable at epoch t+1 if the corpus evolves to place more files nearby, providing better neighborhood constraints. The degree to which GhostUMAP2 instability is a stable property of a file (vs. a property of the current corpus state) is unknown without longitudinal tracking.

**Q2: Markov chain order.** The first-order Markov assumption (current cluster depends only on previous cluster) may be too simple. If cluster assignments show longer-range autocorrelation (e.g., a file stays in cluster A for 3 epochs, then transitions to B for 3 epochs), a higher-order chain or a hidden Markov model may better characterize the biography. However, with the limited epoch counts available (one refit per 6 months → ~2 epochs per year), higher-order estimation is likely infeasible without pooling across similar files.

**Q3: The causal direction.** The hypothesis assumes ambiguity in feature space causes both GhostUMAP2 instability and high Markov entropy. But high Markov entropy could also be caused by genuine content evolution (the file changes over time), which is not the same as boundary ambiguity. Separating these two causes requires either content-change detection or a within-file analysis that controls for modification timestamps.

**Q4: Corpus size threshold.** With fewer than ~150 files having sufficient epoch history, the Spearman correlation estimate will have wide confidence intervals. In early deployment, treat the cross-validation results as preliminary and delay strong conclusions until sufficient file histories accumulate.

---

## Summary

GhostUMAP2 and Markov File Biography are theoretically linked through a common cause: files near cluster boundaries in feature space are unstable both in their UMAP projection geometry (detected by ghost displacement) and in their cluster assignment history (detected by high stationary entropy). The shared mediator is HDBSCAN soft membership — border points in HDBSCAN should exhibit both elevated GhostUMAP2 scores and elevated Markov entropy.

The empirical cross-validation design uses Spearman rank correlation as the primary test, supplemented by top-decile overlap rates and HDBSCAN mediation analysis. A sample of n ≥ 150 files with ≥ 5 epoch histories is the minimum practical threshold.

If the correspondence is confirmed, Curator gains a doubly validated instability signal: files flagged by both metrics are confidently ambiguous in a way that is robust to each metric's individual failure modes. The combined ambiguity score A(f) = 0.5·rank(I(f)) + 0.5·rank(H(f)) provides a single routing input for human review queues, with weights adjustable based on calibration against human judgments.

---

*Sources consulted:*
- [GhostUMAP2 — arXiv:2507.17174](https://arxiv.org/abs/2507.17174)
- [Stability for UMAP — arXiv:2011.13430](https://arxiv.org/pdf/2011.13430)
- [How Soft Clustering for HDBSCAN Works — hdbscan documentation](https://hdbscan.readthedocs.io/en/latest/soft_clustering_explanation.html)
- [Faster HDBSCAN Soft Clustering with RAPIDS cuML — NVIDIA](https://developer.nvidia.com/blog/faster-hdbscan-soft-clustering-with-rapids-cuml/)
- [Sample size requirements for Spearman correlations — Springer/Psychometrika](https://link.springer.com/article/10.1007/BF02294183)
- [Entropy Rate Estimation for Markov Chains — arXiv:1802.07889](https://arxiv.org/pdf/1802.07889)
- [Maximum Kolmogorov-Sinai entropy vs minimum mixing time — arXiv:1506.08667](https://arxiv.org/pdf/1506.08667)
- [UMAP Is Spectral Clustering on the Fuzzy Nearest-Neighbor Graph — arXiv:2602.11662](https://arxiv.org/pdf/2602.11662)
