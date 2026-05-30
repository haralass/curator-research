# 23 — OCP-Unlock+ + Revealed Preference Open Problem
_Date: 2026-05-30_

## OCP-Unlock+ (arXiv:2604.17984, April 2026)

Algorithm: EXP3.P-style exponential weights over K discretized threshold levels. "Unlocking" mechanism: when prediction set covers true label, propagates signal to ALL thresholds via monotonicity.

**vs SPS:**
| | SPS | OCP-Unlock+ |
|---|---|---|
| Setting | Stochastic IID | Adversarial / non-IID |
| Regret | O(√T log T) | O(√(K ln K / T)) |
| Under distribution shift | Breaks — MC(t) consistently decreases | Designed for it |
| Implementation complexity | Simple, hyperparameter-free | Complex (K, T must be known, 3 parameters) |

**For Curator:** OCP-Unlock+ is better justified for personal file streams (non-IID, user behavior drifts). SPS breaks under shift, experiments confirm this.

**Implementation cost:** Significantly more complex than SPS. Parameters β, γ, η depend on K and T.

## 🔴 Revealed Preference Under Constrained Choice — GENUINELY OPEN PROBLEM

**The problem:** User picks folder from {f1, f2, f3} shown. That's their best choice FROM THE SHOWN SET — not the global best. If f4 was optimal but not shown, we never know.

**What exists:**
- Economics: McFadden (1978) sampling correction, consideration set models (Barseghyan et al. AER) — well-studied but **no conformal prediction connection**
- ML: Partial Feedback Online Learning (arXiv:2601.21462), Multiple Correct Answers (arXiv:2602.09402) — similar setup but no constraint-endogeneity
- CP: OCP-Unlock+, SPS — assume exogenous true label, never model endogenous choice

**No paper treats:** Conformal prediction where the label is endogenously chosen from the shown prediction set. The label is a constrained best-response, not a noisy signal of fixed ground truth.

**Practical consequence:** Feedback loop / algorithmic monoculture — model never learns to route to folders that never appear in prediction sets. "Rich-get-richer" bias in feedback collection.

**This is a NeurIPS/ICML-level theoretical problem** if formalized properly. Adapting McFadden sampling correction + Heckman selection to CP feedback collection = novel contribution that no existing paper touches.

## 🚪 New Doors
1. **Endogenous label noise in conformal prediction** — completely open. First paper to formalize this + propose correction (IPS estimator? Heckman?) = major theoretical contribution.
2. **OCP-Unlock+ needs K and T in advance** — for a personal file system with unknown horizon, this is a problem. Any paper on parameter-free adversarial CP?
3. **arXiv:2602.09402 (Feb 2026) "Learning with Multiple Correct Answers"** — the trichotomy of regret bounds under different feedback models. Read — might partially address our problem.
