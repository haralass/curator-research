# Doubly Robust Conformal Prediction Bound
**File:** 74 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Doubly Robust Conformal Prediction (DRCP) provides statistically valid prediction set bounds under covariate shift — meaning Curator can generate calibrated confidence intervals for its file classification suggestions even when the test distribution (user's current files) differs from the training distribution (the model's training corpus). Instead of a raw softmax probability, output a conformal prediction set: "this file belongs to one of {Finance, Projects, Archive} with 90% coverage."

## Key Findings
- **Yang et al. 2024 (JRSS-B 86(4):943–965):** DRCP recasts covariate shift as a missing data problem, applies semiparametric efficiency theory. Valid coverage guarantees even when importance weights are misspecified — "doubly robust" because it's correct if either the propensity model OR the outcome model is correct.
- **Ai & Ren 2024 (arxiv 2402.13042):** Fine-grained framework distinguishing covariate shift from conditional relationship shift — reweights training samples for the identifiable part, uses worst-case bounds for the unidentifiable part. Produces valid and efficient prediction intervals.
- **Lévy-Prokhorov conformal prediction (arxiv 2502.14105):** Distributionally robust extension handling both local and global perturbations — strongest theoretical guarantees but most complex to implement.
- **Standard split conformal prediction** is the simplest entry point — no distributional assumptions, works with any model, ~10 lines of code.
- The key user-facing insight: softmax probability "87% Finance" is uncalibrated and misleading; conformal prediction set {"Finance", "Projects"} with 90% coverage is statistically honest.

## Relevant Papers / Prior Art
- "Doubly Robust Calibration of Prediction Sets under Covariate Shift" — Yang et al., JRSS-B 2024 (arxiv 2203.01761, PMC 11398884)
- "Not All Distributional Shifts Are Equal: Fine-Grained Robust Conformal Inference" — Ai & Ren, arxiv 2402.13042
- "Conformal Prediction under Lévy-Prokhorov Distribution Shifts" — arxiv 2502.14105
- "Conformal Prediction: A Data Perspective" — survey, arxiv 2410.06494

## Applicability to Curator
Medium. Immediate win: implement standard split conformal prediction in Python sidecar to replace raw softmax confidence scores. Calibration set = user's confirmed accept decisions (natural ground truth available from day one).

```python
# Split conformal prediction (nonconformity = 1 - softmax_prob)
def calibrate(model, cal_files, cal_labels, alpha=0.1):
    scores = [1 - model.predict_proba(f)[label] 
              for f, label in zip(cal_files, cal_labels)]
    q = np.quantile(scores, 1 - alpha)  # (1-alpha) quantile
    return q  # threshold

def predict_set(model, file, threshold):
    probs = model.predict_proba(file)
    return [label for label, p in probs.items() if 1 - p <= threshold]
```

Minimum calibration set for reliable coverage: ~200 confirmed placements.

## Open Questions
- What calibration set to use? User's confirmed placements (accept decisions) are the natural choice — available after ~2 weeks of use.
- Should Curator show the full prediction set ("Finance or Projects") or just top-1 with calibrated confidence?
- Minimum calibration set size for reliable coverage: ~200 confirmed placements — is that reachable in early usage?

## Sources
- https://arxiv.org/pdf/2203.01761
- https://arxiv.org/pdf/2402.13042
- https://arxiv.org/html/2502.14105v2
- https://arxiv.org/pdf/2410.06494
