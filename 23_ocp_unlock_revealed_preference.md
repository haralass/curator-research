# 23 — OCP-Unlock+ + Revealed Preference Open Problem
_Date: 2026-05-30_

## OCP-Unlock+ (arXiv:2604.17984, April 2026)

Algorithm: EXP3.P-style exponential weights over K discretized threshold levels [Auer et al. 2002]. "Unlocking" mechanism: when prediction set covers true label, propagates signal to ALL thresholds via monotonicity.

**vs SPS [Angelopoulos et al. 2025]:**
| | SPS | OCP-Unlock+ |
|---|---|---|
| Setting | Stochastic IID | Adversarial / non-IID |
| Regret | O(√T log T) | O(√(K ln K / T)) |
| Under distribution shift | Breaks — MC(t) consistently decreases | Designed for it |
| Implementation complexity | Simple, hyperparameter-free | Complex (K, T must be known, 3 parameters) |

**For Curator:** OCP-Unlock+ is better justified for personal file streams (non-IID, user behavior drifts). SPS breaks under shift, experiments confirm this.

**Implementation cost:** Significantly more complex than SPS. Parameters β, γ, η depend on K and T.

## Revealed Preference Under Constrained Choice — GENUINELY OPEN PROBLEM

**The problem:** User picks folder from {f1, f2, f3} shown. That's their best choice FROM THE SHOWN SET — not the global best. If f4 was optimal but not shown, we never know.

**What exists:**
- Economics: McFadden (1978) sampling correction, consideration set models [Barseghyan et al. AER] — well-studied but **no conformal prediction connection** [McFadden 1978]
- ML: Partial Feedback Online Learning (arXiv:2601.21462), Multiple Correct Answers (arXiv:2602.09402) — similar setup but no constraint-endogeneity
- CP: OCP-Unlock+ [arXiv:2604.17984], SPS [arXiv:2405.13268] — assume exogenous true label, never model endogenous choice

**No paper treats:** Conformal prediction where the label is endogenously chosen from the shown prediction set. The label is a constrained best-response, not a noisy signal of fixed ground truth.

**Practical consequence:** Feedback loop / algorithmic monoculture — model never learns to route to folders that never appear in prediction sets. "Rich-get-richer" bias in feedback collection.

**This is a NeurIPS/ICML-level theoretical problem** if formalized properly. Adapting McFadden sampling correction + Heckman selection to CP feedback collection = novel contribution that no existing paper touches.

## References

- OCP-Unlock+ Authors. (2026). "Online Conformal Prediction with Adversarial Semi-Bandit Feedback via Regret Minimization." arXiv:2604.17984.
- Angelopoulos, A. N., et al. (2025). "Stochastic Online Conformal Prediction with Semi-Bandit Feedback." *Proceedings of ICML 2025*. arXiv:2405.13268.
- Auer, P., Cesa-Bianchi, N., Freund, Y., & Schapire, R. E. (2002). "The Nonstochastic Multiarmed Bandit Problem." *SIAM Journal on Computing*, 32(1), 48–77. DOI: 10.1137/S0097539701398375. (EXP3.P algorithm.)
- McFadden, D. (1978). "Modeling the choice of residential location." *Transportation Research Record*, 673, 72–77. (Sampling correction and consideration sets in discrete choice.)
- Barseghyan, L., Molinari, F., O'Donoghue, T., & Teitelbaum, J. C. (2021). "Discrete Choice under Risk with Limited Consideration." *American Economic Review*, 111(6), 1972–2006. DOI: 10.1257/aer.20190116.
- Heckman, J. (1979). "Sample Selection Bias as a Specification Error." *Econometrica*, 47(1), 153–161. DOI: 10.2307/1912352.
- Partial Feedback Online Learning. (2026). arXiv:2601.21462.
- Multiple Correct Answers. (2026). "Learning with Multiple Correct Answers." arXiv:2602.09402.
