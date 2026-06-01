# Mondrian CP Cold Start: <9 Calibration Examples Per Class

**File:** 85 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Mondrian conformal prediction (CP) requires per-class calibration sets to produce valid, class-conditional coverage guarantees. With fewer than ~9–10 examples per class the empirical quantile estimate is unreliable (coverage error can exceed ε by > 5%). The recommended strategy for Curator is a three-tier fallback: (1) standard (non-Mondrian) CP when any class has <10 calibration examples; (2) a weighted hybrid that partially borrows from the marginal calibration set; (3) full Mondrian CP once each class accumulates ≥20 examples. The `crepes` library (Henrik Boström, KTH) implements all three modes and is the correct dependency.

## Key Findings
- **Coverage validity:** Mondrian CP gives class-conditional 1−ε coverage if the calibration set for class k has sufficient examples. With n_k calibration examples, the worst-case coverage error is bounded by 1/n_k. For ε=0.10 and target worst-case error ≤0.05, you need n_k ≥ 20.
- **Minimum safe threshold:** 9–10 examples per class is an absolute floor (gives ≥1−ε coverage with ≈50% confidence only); 20 is the recommended production threshold.
- **crepes library:** `pip install crepes` (Henrik Boström, MLJAR/KTH, COPA 2024). Implements `ConformalClassifier` with a `mc` (MondrianCategorizer) argument. When `mc=None`, falls back to standard CP automatically.
- **MondrianCategorizer API:** pass a function `f(y_hat) → category` as the `mc` argument to `calibrate()`. For Curator, the category is the predicted folder cluster ID.
- **Fallback design:** crepes does not automatically detect cold-start; the Curator code must check `min(class_counts) < threshold` and set `mc=None` accordingly.
- **Prediction set size at cold start:** standard CP tends to produce larger prediction sets (more conservative), which is acceptable — it means showing more candidate folders to the user rather than fewer.
- **MAPIE compatibility:** MAPIE also supports Mondrian CP via `cv="prefit"` + `groups` argument, but crepes is more ergonomic for the categorical Mondrian use case.

## Relevant Papers / Prior Art
- Boström H., "crepes: a Python Package for Generating Conformal Regressors and Predictive Systems," *PMLR* 179, 2022. — Canonical reference for the library.
- Boström H., Johansson U., "Mondrian conformal regressors," *Proceedings of COPA*, 2020. Semantic Scholar: 5ef5c232... — Defines Mondrian taxonomy and coverage proofs.
- Vovk V., Gammerman A., Shafer G., *Algorithmic Learning in a Random World*, Springer, 2005. — Foundational CP theory including Mondrian CP validity proofs.

## Applicability to Curator

```python
from crepes import ConformalClassifier
from crepes.extras import MondrianCategorizer
import numpy as np

# After Louvain: cluster_labels[i] = cluster ID for file i
# Split: 70% train model, 20% calibrate, 10% test (held out)

clf = train_model(X_train, y_train)  # any sklearn-compatible classifier

cc = ConformalClassifier()

# Count calibration examples per predicted class
cal_preds = clf.predict(X_cal)
class_counts = np.bincount(cal_preds)
MIN_CAL_THRESHOLD = 20

if class_counts.min() >= MIN_CAL_THRESHOLD:
    # Full Mondrian CP
    mc = MondrianCategorizer()
    mc.fit(cal_preds)  # category = predicted class
    cc.fit(clf, X_cal, y_cal)
    cc.calibrate(X_cal, y_cal, mc=mc)
    mode = "mondrian"
else:
    # Standard CP fallback
    cc.fit(clf, X_cal, y_cal)
    cc.calibrate(X_cal, y_cal)  # mc=None → standard CP
    mode = "standard"

# Prediction
epsilon = 0.10
prediction_sets = cc.predict_set(X_test, significance=epsilon)
# prediction_sets[i] = list of candidate cluster labels for file i
```

**Switch logic for Curator:**
- New install (0 files): standard CP, prediction sets will be large (show all clusters as candidates).
- After ingesting ~200 files across ≥10 clusters: check `class_counts.min() >= 20`; switch to Mondrian CP.
- Re-check after each recalibration event (see File 98).

## Open Questions
- Does cluster ID assignment remain stable across re-clusterings? If not, the calibration set labels become stale and Mondrian CP is invalid regardless of count. This is the stronger constraint than sample count (see File 98 on drift detection).

## Sources
- https://github.com/henrikbostrom/crepes
- https://pypi.org/project/crepes/
- https://proceedings.mlr.press/v179/bostrom22a.html
- https://cml.rhul.ac.uk/copa2024/presentations/COPA_2024_09_11%20Henrik%20Bostrom.pdf
- https://crepes.readthedocs.io/en/latest/crepes.html
