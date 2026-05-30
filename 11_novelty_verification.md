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
- arXiv:2405.13268 — Stochastic Online CP with Semi-Bandit Feedback (ICML 2025) — nearly identical feedback model, in document retrieval
- arXiv:2410.08852 — Conformalized Interactive Imitation Learning — online CP with intermittent user feedback
- arXiv:2503.10345 — Mirror Online CP with Intermittent Feedback (2025)
**Action:** Reframe as domain novelty (file routing). Cite as related work, not prior art that invalidates.

## Markov File Biography — Novel
No paper models per-file cluster assignment history as a Markov chain with per-file stationary entropy for routing to human review. All existing drift detection is population-level. The term "file biography" is absent from the literature.
Closest: arXiv:2005.08267 — Model-Based Longitudinal Clustering (HMM-based, population-level, different purpose)

## Active Learning — Prior Art
Centroid-representative cold-start has ~20 years of prior art:
- Springer 2004: "Using Cluster-Based Sampling to Select Initial Training Set for Active Learning"
- ACM IDA 2021: "A two-stage clustering-based cold-start method"
**Action:** Frame as novel *application* to personal filesystem + empirical benchmark (90% accuracy, ~8 confirmations/cluster).

## H(location|intent) — Novel
No prior work defines this as a formal metric for PIM quality. PIM papers (Bergman, Jones) use behavioral proxies only. No information-theoretic formalization of organization quality found anywhere.
**Strongest claim in the paper.**

## Weighted CP+FLRS — Novel
FLACON published Oct 2025 (MDPI Entropy, doi:10.3390/e27111133 — confirmed real). The CP+FLACON hybrid has no precedent. CP provides coverage guarantees; FLACON provides metadata-aware distance.
Closest: arXiv:2407.10230 — Weighted aggregation of conformity scores (no FLACON component)

## 🚪 New Doors Opened
1. **Semi-Bandit CP for file routing** (arXiv:2405.13268) — the feedback model is almost identical to ours. Could we adopt their regret minimization framework to *prove* convergence of our recalibration? This would turn "partial prior art" into a theoretical contribution with formal guarantees.
2. **FLACON is real and recent (Oct 2025)** — the authors are active. Is there an opportunity for collaboration or citation exchange?
3. **"File biography" as terminology is unclaimed** — opportunity to define/own this term in the literature formally.
