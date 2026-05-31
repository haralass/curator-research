# 16 — Semi-Bandit CP (arXiv:2405.13268) — Deep Analysis
_Date: 2026-05-30_

## Feedback Model Match: Near-Perfect ✅
- User picks folder from set → observe s_t (equivalent to y*_t ∈ C_t)
- User rejects all → binary indicator only (y*_t ∉ C_t)
- Matches SPS paper's model almost exactly

One nuance: user "acceptance" is revealed preference, not ground truth. If the true best folder wasn't in the set, we'd never know. Paper has same structural issue, doesn't address it.

## Algorithm (SPS) — Adoptable Directly
```
τ_1 ← −∞
for each file:
    if user_rejects: substitute s_t ← τ_t  (score substitution trick)
    Compute truncated empirical CDF G̅_t with DKW correction ε_t = √(log(2T²)/2t)
    τ_t = max(sup{τ | G̅_t(τ) ≤ 1−α}, τ_{t-1})  # monotone non-decreasing
```
**No hyperparameters.** Parameter-free. Simple to implement.

## Guarantees
- **Regret:** R_T ≤ K(2 log T + 4√(T log T) + 1) + 4φ_max → sublinear, O(√(T log T))
- **Safety:** τ_t ≤ τ* always w.p. ≥ 1−2/T → sets never chronically too small from day 1

## Critical Problem: IID Assumption
Paper assumes data i.i.d. from fixed distribution D [Angelopoulos et al. 2025]. Personal file systems are emphatically non-IID:
- Tax season → many tax docs
- New semester → new course files
- Distribution shifts over weeks/months

At 5–100 files/week, meaningful convergence (T=100–1000) takes **months to years**. The paper only experiments at T=10,000.

## Practical Verdict
- Implement SPS — correct for our feedback structure, coverage safety from day 1
- Do NOT claim formal regret convergence in the paper for personal use scale
- **The coverage guarantee is the practical claim**

## Open Theoretical Contributions
Three open problems that would make genuine theoretical contributions:
1. **Non-IID / distribution shift version of SPS** — blending with Gibbs & Candes (2021) adversarial techniques [Gibbs & Candes 2021], but those require full label observation → combination is OPEN
2. **Small-T finite-sample bounds** non-vacuous at T=50–200 — not in literature
3. **Revealed preference formalization** — when user picks from constrained set, that's not ground truth — modeling this rigorously is new

Any one of these would upgrade Temporal CP from "application paper" to "theoretical contribution."

## References

- Angelopoulos, A. N., et al. (2025). "Stochastic Online Conformal Prediction with Semi-Bandit Feedback." *Proceedings of ICML 2025*. arXiv:2405.13268.
- Gibbs, I., & Candès, E. J. (2021). "Adaptive Conformal Inference Under Distribution Shift." *Advances in Neural Information Processing Systems 34 (NeurIPS 2021)*. arXiv:2106.00170. https://proceedings.neurips.cc/paper_files/paper/2021/hash/0d441de75945e5acbc865406fc9a2559-Abstract.html.
- Massart, P. (1990). "The Tight Constant in the Dvoretzky-Kiefer-Wolfowitz Inequality." *The Annals of Probability*, 18(3), 1269–1283. DOI: 10.1214/aop/1176990746. (DKW inequality used in SPS bound.)
- Vovk, V., Gammerman, A., & Shafer, G. (2005). *Algorithmic Learning in a Random World*. Springer. (Foundations of conformal prediction.)
