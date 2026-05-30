# Conformal Prediction για File Classification
**Compiled:** May 2026 | **Relevance:** Prediction sets με formal coverage guarantees στον Curator

---

## Τι Είναι Conformal Prediction

Αντί για "αυτό το αρχείο είναι Finance (72%)", λες:
> **"Finance ή Legal — με 95% coverage guarantee"**

Το guarantee είναι formal: αν ζητάς 95% coverage, τουλάχιστον 95% των prediction sets θα περιέχουν τη σωστή κατηγορία. Χωρίς distributional assumptions, για οποιοδήποτε classifier, με finite samples.

### Core Mechanics (Split Conformal Prediction)

1. Train classifier κανονικά
2. Reserve **calibration set** (~300–1000 labeled examples) — ποτέ για training
3. Υπολόγισε **nonconformity score** για κάθε calibration example
4. Πάρε το (1-α)th quantile → `q_hat`
5. Test time: include κάθε κατηγορία με nonconformity score ≤ q_hat

| Property | Details |
|---|---|
| Distribution-free | Καμία assumption |
| Model-agnostic | Wrapper around any classifier |
| Finite-sample valid | Guarantee holds για οποιοδήποτε n |

---

## Nonconformity Scores για LLMs

### LAC (Least Ambiguous Classifier) — Πιο Απλό
```python
nonconformity_score(x, y) = 1 - softmax_prob[y]
```
Trivially implementable από logprobs του Ollama.

### APS (Adaptive Prediction Sets) — Καλύτερο
Αθροίζει softmax probs όλων των κατηγοριών με higher rank από y (inclusive). Sensitive στην full distribution.

### RAPS (Regularized APS) — **Recommended**
```
score_RAPS = score_APS + λ * max(0, rank(y) - k_reg)
```
Αποτρέπει πολλές low-confidence κατηγορίες. **5–10x μικρότερα sets** vs naive, ίδιο coverage. Current practical standard.

### SAPS — Newer, further improvement over RAPS

---

## Key Papers

| Paper | Venue | Key Insight |
|---|---|---|
| **CP for NLP Survey** — arXiv 2405.01976 | TACL Nov 2024 | Definitive survey — classification, QA, generation, MT |
| **"API Is Enough"** — arXiv 2403.01216 | EMNLP 2024 | CP χωρίς logit access — sampling frequency approach |
| **CP Sets Improve Human Decisions** — arXiv 2401.13744 | ICML 2024 | RCT 1000+ participants: CP sets **improve human accuracy** (d=0.4–0.7). Variable set size communicates uncertainty effectively |
| **TECP** | Mathematics 2025 | Token-entropy nonconformity για generative LLMs |
| **Domain-Shift-Aware CP** — arXiv 2510.05566 | 2025 | Coverage maintenance under domain shift |
| **PASC** — arXiv 2605.18812 | 2025 | CP για multi-stage NLP pipelines |
| **RAPS** — arXiv 2009.14193 | ICLR 2021 | Practical standard για classification |

**Critical finding from ICML 2024:** Humans με CP sets είχαν **measurably higher accuracy** από humans χωρίς βοήθεια ή με fixed-size sets. Mechanism: variable set size communicates uncertainty. Single-item set → "confident"; three-item set → "please review." Αυτό validates exactly Curator's design direction.

---

## GitHub Repos

| Repo | Notes |
|---|---|
| [scikit-learn-contrib/MAPIE](https://github.com/scikit-learn-contrib/MAPIE) | Scikit-learn compatible. MapieClassifier. LAC, RAPS, APS out of the box |
| [ml-stat-Sustech/TorchCP](https://github.com/ml-stat-Sustech/TorchCP) | PyTorch-native. LLM-specific modules. JMLR 2025 |
| [aangelopoulos/conformal-prediction](https://github.com/aangelopoulos/conformal-prediction) | Reference implementation by key researcher |
| [deel-ai/puncc](https://github.com/deel-ai/puncc) | HuggingFace NLP tutorials |

---

## Implementation για Curator

### Calibration Set — Bootstrap από Existing Folders
Ο χρήστης έχει ήδη ένα `Finance/` folder με 500 PDFs → 500 labeled calibration examples δωρεάν. Minimum: 300 files, καλύτερα 500+.

### Code

```python
import numpy as np

# Step 1: Compute calibration scores
cal_scores = []
for file, true_label in calibration_set:
    logprobs = get_ollama_logprobs(file, categories)
    probs = softmax(logprobs)
    label_idx = categories.index(true_label)
    cal_scores.append(1.0 - probs[label_idx])  # LAC score

# Step 2: Compute threshold
alpha = 0.05  # 95% coverage
n = len(cal_scores)
q_level = np.ceil((n + 1) * (1 - alpha)) / n
q_hat = np.quantile(cal_scores, q_level)  # persist to disk

# Step 3: Predict sets at inference
def predict_set(file, q_hat, categories):
    probs = softmax(get_ollama_logprobs(file, categories))
    return {cat: prob for cat, prob in zip(categories, probs)
            if (1.0 - prob) <= q_hat}

# Step 4: Map to HIGH/MEDIUM/LOW
def format_result(pred_set):
    size = len(pred_set)
    cats = sorted(pred_set, key=pred_set.get, reverse=True)
    conf = "HIGH" if size == 1 else "MEDIUM" if size == 2 else "LOW"
    label = " or ".join(cats[:3])
    return f"{label} (95% coverage)", conf
```

**Output examples:**
- `"Finance" (95% coverage)` — HIGH
- `"Finance or Legal" (95% coverage)` — MEDIUM
- `"Finance or Legal or Tax" (95% coverage)` — LOW

---

## Distribution Shift — Νέοι Τύποι Αρχείων

**Το πρόβλημα:** Standard CP requires exchangeability. Όταν εμφανίζεται νέο folder type, coverage μπορεί να degradeάρει.

**Solutions:**
1. **Weighted CP** (Tibshirani 2019) — reweight calibration scores με density ratio
2. **Domain-Shift-Aware CP** (arXiv 2510.05566) — adapts threshold με domain similarity scores
3. **Online/Adaptive CP** — rolling calibration window. User corrections → calibration set update
4. **Empty prediction set = OOD signal** ← **Αυτό είναι feature για τον Curator**

Αν κανένα category δεν περνά threshold → "δεν ξέρω αυτό το αρχείο" → prompt user να ορίσει νέα κατηγορία.

### Incremental Recalibration
```python
# After user correction:
cal_scores.append(new_score)
q_hat = np.quantile(cal_scores, q_level)  # O(n log n), essentially free
```

---

## Computational Overhead

| Phase | Overhead |
|---|---|
| **Calibration** (one-time) | N inference calls + O(n log n) sort. Για n=500: seconds |
| **Inference** (per file) | **Zero overhead** — CP applies threshold to existing logprobs |
| **Recalibration** | < 1ms στην Python |

`q_hat` = ένα floating-point number στο disk. Εντελώς feasible για local-first app.

---

## Open Problems

| Problem | Status |
|---|---|
| Exchangeability violation | Active research |
| Conditional coverage per-class | Marginal guaranteed; per-class requires RAPS |
| Large sets με πολλές κατηγορίες (K>20) | RAPS/SAPS βοηθούν, αλλά partial |
| Calibration set labeling | Bootstrap από existing folders — solved για Curator |
| New categories (open-world) | Empty prediction sets = best current signal |

---

## Implementation Path για Curator

1. **Start**: LAC (1 - softmax_prob) — trivial από Ollama logprobs
2. **Bootstrap calibration**: Από existing user folders (labeled)
3. **α = 0.05** default (95% coverage) — expose ως user setting
4. **Upgrade to RAPS** για μικρότερα sets
5. **Empty sets = "define new category" prompt**
6. **Incremental recalibration** από user corrections

---

## Sources
- [CP for NLP Survey — arXiv 2405.01976](https://arxiv.org/abs/2405.01976)
- [API Is Enough — arXiv 2403.01216](https://arxiv.org/abs/2403.01216)
- [CP Sets Improve Human Decisions — arXiv 2401.13744](https://arxiv.org/pdf/2401.13744)
- [RAPS — ICLR 2021](https://arxiv.org/pdf/2009.14193)
- [Domain-Shift-Aware CP — arXiv 2510.05566](https://arxiv.org/html/2510.05566v1)
- [PASC — arXiv 2605.18812](https://arxiv.org/html/2605.18812)
- [MAPIE](https://github.com/scikit-learn-contrib/MAPIE)
- [TorchCP](https://github.com/ml-stat-Sustech/TorchCP)
- [aangelopoulos/conformal-prediction](https://github.com/aangelopoulos/conformal-prediction)
- [Weighted CP — Tibshirani et al.](https://www.stat.cmu.edu/~ryantibs/papers/weightedcp.pdf)
- [OOD Detection via CP — arXiv 2403.11532](https://arxiv.org/pdf/2403.11532)
