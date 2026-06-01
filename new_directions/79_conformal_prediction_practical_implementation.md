# Conformal Prediction: Practical Implementation for File Classification
**File:** 79 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
File 74 established that Doubly Robust Conformal Prediction provides calibrated prediction sets with coverage guarantees. This file documents the practical Python implementation path: MAPIE is the recommended library (sklearn-compatible, actively maintained, supports Mondrian/class-conditional coverage and online partial_fit). Inductive (split) conformal prediction is the right mode for an online system accumulating calibration data. Mondrian conformal prediction gives per-class coverage guarantees with a minimum of ~250 calibration examples per class for reliable behavior.

## Key Findings

### Library Landscape (2025–2026 Status)

**MAPIE (Recommended)**
- Current version: 1.4.x (actively maintained as of 2026)
- sklearn-compatible: follows `.fit()` / `.predict()` API exactly
- Supports classification prediction sets (`MapieClassifier`), regression intervals, and time series
- Key capability: `.partial_fit()` for online/streaming calibration — new calibration points can be incorporated without full refit
- Supports `cv="prefit"` mode: train your classifier separately, pass it to MAPIE, calibrate on a held-out set
- 2026 roadmap includes: exchangeability tests, risk control for complex outputs
- GitHub: https://github.com/scikit-learn-contrib/MAPIE

**crepes (Good for Lightweight Use)**
- Focused, minimal library for conformal regressors and classifiers
- Extended in 2024 to include conformal classifiers (previously only regressors)
- 10x faster than MAPIE for simple tasks with limited data (benchmark from COPA 2024)
- Best for: offline batch evaluation, quick experiments
- Not ideal for production online systems (no partial_fit equivalent)
- GitHub: https://github.com/henrikbostrom/crepes

**nonconformist (Deprecated — Do Not Use)**
- Last meaningful maintenance ~2019; API documentation "severely deprecated"
- No GPU support, no sklearn modern API compatibility
- Issues on GitHub are largely unanswered
- Modern alternatives (MAPIE, crepes) are strictly better in every dimension
- Status: effectively abandoned

**TorchCP (For PyTorch Pipelines)**
- JMLR 2024: designed for PyTorch-native models (neural nets, LLMs)
- Useful if Curator's classifier is a neural net rather than sklearn
- Overkill for an sklearn-based file classifier

### Inductive (Split) vs Transductive Conformal Prediction

**Transductive CP:**
- Recomputes nonconformity scores for all training data on every new test point
- O(n) model retraining per prediction — computationally infeasible for online systems
- Provides tighter prediction sets in theory (uses all data for calibration)

**Inductive (Split) CP — Correct Choice for Curator:**
- Train/fit model once on training set
- Compute nonconformity scores on a held-out calibration set once
- At prediction time: compare new file's nonconformity score against stored calibration quantile
- O(1) per prediction after calibration step
- As calibration set grows (user makes more decisions), simply append new scores and recompute quantile — O(k) where k = new calibration points added
- Validity guarantee: marginal coverage holds as long as calibration + test points are exchangeable

**Online Conformal Prediction with Growing Calibration Data:**
- MAPIE's `.partial_fit()` supports incremental calibration: call after each user decision
- Coverage guarantee is maintained sequentially: even if the calibration set is small initially, the coverage guarantee holds for the running frequency of errors, not just at a fixed snapshot
- Literature (Gibbs & Candès, 2021; Angelopoulos et al., 2024) shows online CP with exchangeable data maintains coverage without distributional assumptions
- Practical implication for Curator: every time the user approves or rejects a suggestion, append the resulting nonconformity score to the calibration buffer. Recompute quantile nightly or after every N=10 new decisions.

### Mondrian Conformal Prediction (Class-Conditional Coverage)

**Why it matters for Curator:** Marginal coverage (90% overall) is insufficient. A folder class with only 5 files may have 70% coverage while a common class has 95%. Mondrian CP gives 90% coverage *per folder class*.

**How it works:**
- Split calibration data by class label y
- For each class y, compute its own quantile q_y from the class-specific calibration subset
- At prediction time, include label y in the prediction set iff nonconformity score ≤ q_y
- Coverage guarantee: P(Y ∈ C(X) | Y = y) ≥ 1 − α for all y

**Minimum calibration set size per class:**
- From Ding et al. (NeurIPS 2023, "Class-Conditional Conformal Prediction with Many Classes"):
  - If |calibration set for class y| < ⌈(1/α) - 1⌉, the quantile is infinite → prediction set always includes that class (conservative fallback)
  - For α = 0.1 (90% coverage): minimum meaningful size = **9 examples per class**
  - For stable, well-behaved quantile estimates: **≥ 250 examples per class** (best performance)
  - Kandinsky CP (2025): 500 samples/class for best group-average coverage deviation
- **Practical implication**: For rare folder classes (< 9 examples), Mondrian CP gracefully degrades to "always include this class" — harmless behavior. Once a class accumulates ≥ 10 examples, it gets a meaningful quantile.

### End-to-End Code Example: Multi-Class File Classifier with MAPIE

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import LabelEncoder
from mapie.classification import MapieClassifier
import numpy as np

# --- Step 1: Train base classifier on labeled examples ---
# X_train: file feature vectors (e.g., TF-IDF + metadata)
# y_train: folder labels (strings)
le = LabelEncoder()
y_train_enc = le.fit_transform(y_train)

base_clf = RandomForestClassifier(n_estimators=100, random_state=42)
base_clf.fit(X_train, y_train_enc)

# --- Step 2: Calibrate with held-out labeled examples ---
# cv="prefit" means: base_clf is already trained, just calibrate
mapie_clf = MapieClassifier(
    estimator=base_clf,
    cv="prefit",
    method="score"  # uses softmax score as nonconformity measure
)
mapie_clf.fit(X_calib, le.transform(y_calib))

# --- Step 3: Mondrian (class-conditional) prediction sets ---
# alpha=0.1 → 90% coverage guarantee
y_pred, y_pred_sets = mapie_clf.predict(
    X_new,
    alpha=0.1,
    include_last_label=True  # ensures validity
)

# y_pred_sets shape: (n_samples, n_classes, n_alpha_levels)
# y_pred_sets[i, :, 0] is a boolean mask of included classes for sample i
included_labels = le.inverse_transform(
    np.where(y_pred_sets[0, :, 0])[0]
)
print(f"Prediction set for new file: {included_labels}")
# e.g., ["Work/ProjectX", "Work/ProjectY"]  ← show both to user

# --- Step 4: Online update after user decision ---
# When user confirms file goes to "Work/ProjectX":
new_score = ...  # compute nonconformity score for this calibration point
# MAPIE supports partial_fit for some configurations:
# mapie_clf.partial_fit(X_new_confirmed, y_new_confirmed)
# Or: manually append to calibration buffer and refit calibration step only
```

### Key Behavior Properties

| Scenario | MAPIE Behavior |
|---|---|
| New file, well-represented class | Tight prediction set, often singleton |
| New file, ambiguous features | Wide prediction set (2–3 folders shown) |
| New file, class never seen | Falls back to conservative set (all classes) |
| Class with < 9 calibration examples | Always includes that class (harmless) |
| User rejects suggestion | Nonconformity score logged, calibration improves |

### Nonconformity Measure Choice

- **Score method** (default): nonconformity = 1 − softmax probability of true class. Simple, works well.
- **RAPS** (Regularized Adaptive Prediction Sets, Angelopoulos et al. 2021): adjusts for softmax overconfidence. Better for neural classifiers.
- For sklearn-based file classifiers, the score method is sufficient.

## Relevant Papers / Prior Art

- Angelopoulos, A.N. & Bates, S. (2022). "A Gentle Introduction to Conformal Prediction and Distribution-Free Uncertainty Quantification." https://arxiv.org/abs/2107.07511
- Ding, T. et al. (2023). "Class-Conditional Conformal Prediction with Many Classes." *NeurIPS 2023*. https://proceedings.neurips.cc/paper_files/paper/2023/file/cb931eddd563f8d473c355518ce8601c-Paper-Conference.pdf
- Boström, H. (2024). "Conformal Prediction in Python with crepes." *COPA 2024*. https://proceedings.mlr.press/v230/bostrom24a.html
- Cordier, T. et al. (2022). "MAPIE: An Open-Source Library for Distribution-Free Uncertainty Quantification." *arXiv:2207.12274*. https://arxiv.org/abs/2207.12274
- Gibbs, I. & Candès, E. (2021). "Adaptive Conformal Inference Under Distribution Shift." *NeurIPS 2021*. (Online CP with coverage guarantees)
- Venn Prediction / Mondrian framework: Vovk, V., Gammerman, A., Shafer, G. (2005). *Algorithmic Learning in a Random World.* Springer.

## Applicability to Curator

1. **Use MAPIE with `cv="prefit"` and `method="score"`** for the production classifier. Train the base model (e.g., LogisticRegression or RandomForest on file features), calibrate on confirmed user decisions.
2. **Target α = 0.1** (90% marginal coverage) as the default. Expose this as a user-facing "confidence" slider in settings.
3. **Mondrian per-class coverage**: implement by grouping calibration scores by folder class. Once a class has ≥ 10 labeled examples, switch from global quantile to per-class quantile for that class.
4. **Show multi-element prediction sets to the user**: when the prediction set has size > 1, surface all candidates (e.g., "This might be: Work/CoolProject or Archive/2025"). This is the key UX unlock — calibrated uncertainty expressed as explicit alternatives rather than a single low-confidence guess.
5. **Cold start**: before any calibration data (0 labeled examples), default to global softmax top-1 suggestion with an explicit "I'm not confident yet" indicator. Switch to conformal sets after first 20 labeled examples.
6. **nonconformist**: do not use. It is deprecated. MAPIE is the production-ready replacement.

## Open Questions

- **Covariate shift**: If user's file-naming conventions or project domains change significantly over time, the exchangeability assumption weakens. How to detect and handle distribution shift in the calibration buffer? (EnbPI or weighted CP may help.)
- **Feature drift**: If the embedding model is updated, all calibration scores are invalid and must be recomputed. Need a version-tagged calibration buffer.
- **Multi-label files**: A file may belong to multiple folders (e.g., a shared document). Standard conformal prediction for multi-label classification is more complex — MAPIE 2026 roadmap item.
- **Optimal calibration set refresh policy**: How often to recompute quantiles? Every decision vs. batched nightly?

## Sources
- https://mapie.readthedocs.io/en/latest/
- https://github.com/scikit-learn-contrib/MAPIE
- https://github.com/henrikbostrom/crepes
- https://github.com/donlnz/nonconformist
- https://proceedings.mlr.press/v230/bostrom24a.html
- https://proceedings.neurips.cc/paper_files/paper/2023/file/cb931eddd563f8d473c355518ce8601c-Paper-Conference.pdf
- https://arxiv.org/pdf/2406.06818
- https://arxiv.org/pdf/2305.12616
- https://www.jmlr.org/papers/volume26/24-2141/24-2141.pdf
