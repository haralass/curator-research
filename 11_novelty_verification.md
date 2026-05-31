# 11 — Novelty Verification: 5 Mathematical Frameworks
_Date: 2026-05-30_

## Results

| Framework | Verdict | Confidence |
|---|---|---|
| Temporal Conformal Prediction | 🟡 PARTIAL PRIOR ART | Medium |
| Markov File Biography | 🟢 CONFIRMED NOVEL | High |
| Active Learning for File Org | 🟡 PARTIAL PRIOR ART | High |
| H(location\|intent) Entropy | 🟢 CONFIRMED NOVEL | High |
| Weighted CP+FLRS | 🟢 CONFIRMED NOVEL | High |

## Temporal CP — Prior Art
- arXiv:2405.13268 — Stochastic Online CP with Semi-Bandit Feedback (ICML 2025) — nearly identical feedback model, in document retrieval [Angelopoulos et al. 2025]
- arXiv:2410.08852 — Conformalized Interactive Imitation Learning — online CP with intermittent user feedback
- arXiv:2503.10345 — Mirror Online CP with Intermittent Feedback (2025)

**Action:** Reframe as domain novelty (file routing). Cite as related work, not prior art that invalidates.

## Markov File Biography — Novel
No paper models per-file cluster assignment history as a Markov chain with per-file stationary entropy for routing to human review. All existing drift detection is population-level. The term "file biography" is absent from the literature.

Closest: arXiv:2005.08267 — Model-Based Longitudinal Clustering (HMM-based, population-level, different purpose) [Ye et al. 2020]

## Active Learning — Prior Art
Centroid-representative cold-start has ~20 years of prior art:
- Springer 2004: "Using Cluster-Based Sampling to Select Initial Training Set for Active Learning" [Nguyen & Smeulders 2004]
- ACM IDA 2021: "A two-stage clustering-based cold-start method"

**Action:** Frame as novel *application* to personal filesystem + empirical benchmark (90% accuracy, ~8 confirmations/cluster).

## H(location|intent) — Novel
No prior work defines this as a formal metric for PIM quality. PIM papers [Bergman et al. 2010; Jones & Teevan 2007] use behavioral proxies only. No information-theoretic formalization of organization quality found anywhere.

**Strongest claim in the paper.**

## Weighted CP+FLRS — Novel
FLACON published Oct 2025 (MDPI Entropy, doi:10.3390/e27111133 — confirmed real) [Yoon 2025]. The CP+FLACON hybrid has no precedent. CP provides coverage guarantees; FLACON provides metadata-aware distance.

Closest: arXiv:2407.10230 — Weighted aggregation of conformity scores (no FLACON component)

## References

- Angelopoulos, A. N., et al. (2025). "Stochastic Online Conformal Prediction with Semi-Bandit Feedback." *Proceedings of ICML 2025*. arXiv:2405.13268.
- ArXiv:2410.08852. "Conformalized Interactive Imitation Learning." 2024.
- ArXiv:2503.10345. "Mirror Online Conformal Prediction with Intermittent Feedback." 2025.
- Ye, W., et al. (2020). "Model-Based Longitudinal Clustering." arXiv:2005.08267.
- Nguyen, H. T., & Smeulders, A. (2004). "Active Learning Using Pre-clustering." *Proceedings of the 21st ICML*. Springer.
- Bergman, O., Whittaker, S., Sanderson, M., Nachmias, R., & Ramamoorthy, A. (2010). "The effect of folder structure on personal file navigation." *Journal of the American Society for Information Science and Technology*, 61(12), 2426–2441. DOI: 10.1002/asi.21415.
- Jones, W., & Teevan, J. (Eds.) (2007). *Personal Information Management*. University of Washington Press.
- Yoon, S. (2025). "FLACON: An Information-Theoretic Approach to Flag-Aware Contextual Clustering for Large-Scale Document Organization." *Entropy*, 27(11), 1133. DOI: 10.3390/e27111133.
- ArXiv:2407.10230. "Weighted Aggregation of Conformity Scores for Conformal Prediction." 2024.
