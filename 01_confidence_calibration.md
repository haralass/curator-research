# AI Confidence Calibration — Deep Research Report
**Compiled:** May 2026 | **Relevance:** Curator HIGH/MEDIUM/LOW confidence system

---

## Τι είναι Calibration

Ένας classifier είναι **well-calibrated** όταν η stated confidence αντικατοπτρίζει την πραγματική ακρίβεια:
- Αν λέει 80% confidence σε 100 αποφάσεις → ~80 πρέπει να είναι σωστές
- Τα modern LLMs και instruction-tuned models είναι **συστηματικά over-confident** out of the box
- Μετά από RLHF fine-tuning η κατάσταση χειροτερεύει — ακριβώς τα models που τρέχει το Ollama

### Δύο Τύποι Uncertainty

| Τύπος | Ορισμός | Μειώσιμο; | Για τον Curator |
|---|---|---|---|
| **Aleatoric** | Inherent noise — αμφίσημα filenames, παρόμοια αρχεία | Όχι | Flag ως "αυτό το αρχείο είναι αμφίσημο" |
| **Epistemic** | Το model δεν ξέρει — OOD file type, σπάνια κατηγορία | Ναι | Flag ως "δεν έχω δει τέτοιο αρχείο πριν" |

---

## State of the Art — Μέθοδοι

### Post-hoc (χωρίς retraining — πιο πρακτικό για Curator)

| Μέθοδος | Πώς λειτουργεί | ECE Μείωση |
|---|---|---|
| **Temperature Scaling** | Διαιρεί logits με scalar T πριν softmax | ~50–82% |
| **Adaptive Temperature Scaling (ATS)** | Προβλέπει per-token T από features | 10–50% over standard TS |
| **JUCAL** | Calibrates aleatoric + epistemic ξεχωριστά | NLL ↓15%, prediction set ↓20% |
| **CCPS** | Perturbs hidden states, trains probe | **55% ECE μείωση** (χρειάζεται white-box) |
| **Isotonic Regression** | Non-parametric monotonic fit | ~77% |

### Verbalized vs Token-Probability Confidence
- **Token-probability**: από softmax/logit — γρήγορο αλλά unreliable μετά RLHF
- **Verbalized**: ρωτάς "πόσο σίγουρος είσαι;" — για RLHF models **καλύτερα calibrated** από token-probs
- → Το Curator που ρωτά το LLM να πει HIGH/MEDIUM/LOW είναι **σωστή προσέγγιση** βάσει έρευνας

---

## Key Papers

| Paper | Venue | Key Insight |
|---|---|---|
| **Survey of Confidence Estimation in LLMs** | ResearchGate 2024 | Taxonomy verbalized vs token-prob; ECE comparisons |
| **Adaptive Temperature Scaling (ATS)** — arXiv 2409.19817 | ICLR 2025 | Per-token adaptive calibration για RLHF models |
| **CCPS** — arXiv 2505.21772 | EMNLP 2025 | 55% ECE reduction με hidden state perturbation |
| **JUCAL** — arXiv 2602.20153 | OpenReview 2026 | Aleatoric + epistemic jointly; 15–20% improvement |
| **LLM4Tag** — arXiv 2502.13481 | KDD 2025 | **Πιο κοντινό σύστημα στον Curator** — tagging με calibrated confidence |
| **Taming Overconfidence in RLHF** — arXiv 2410.09724 | TMLR 2025 | Γιατί τα Ollama models είναι over-confident |
| **SmoothECE** | ICLR 2024 | Καλύτερο calibration metric, χωρίς binning artifacts |
| **Uncertainty-Based Abstention** — arXiv 2404.10960 | 2024 | Entropy thresholds → refuse to answer; = Curator LOW |
| **Are You Really Sure?** (CHI 2024) | CHI 2024 | Human trust calibration ≠ model calibration — HCI gap |

---

## GitHub Repos

| Repo | Χρήση |
|---|---|
| [ljyflores/calibrated-confidence-for-nlg](https://github.com/ljyflores/calibrated-confidence-for-nlg) | ACL 2025 — text generation calibration |
| [rlacombe/LLM-Calibration](https://github.com/rlacombe/LLM-Calibration) | ICML 2025 — CoT impairs calibration (προσοχή!) |
| **netcal** (Python library) | Production-grade: temperature scaling, Beta calibration, ECE computation |
| **sklearn CalibratedClassifierCV** | Platt scaling + isotonic regression built-in |

---

## Metrics

| Metric | Δυνατά | Αδύνατα |
|---|---|---|
| **ECE** | Intuitive, universal | Sensitive to bin count |
| **SmoothECE** (ICLR 2024) | No binning artifacts, continuous | — |
| **Brier Score** | Proper scoring rule | Conflates calibration + sharpness |
| **Reliability Diagram** | Visual, intuitive | Needs many samples |

**Σύσταση για Curator:** SmoothECE + Brier Score + Reliability Diagram με 200–500 labelled samples.

---

## Open Gaps (2025–2026)

1. **Post-RLHF calibration degradation** — instruction-tuned models (Ollama) χωρίς universally agreed fix
2. **Calibration under distribution shift** — νέα domains degrading calibration
3. **Verbalized confidence misalignment** — model λέει 90% και δεν σημαίνει τίποτα consistent
4. **Many-classes calibration** — 50–200+ κατηγορίες = λίγα samples per bin
5. **Calibration in agentic pipelines** — multi-step compounds errors
6. **PIM / personal file classification — σχεδόν ανεξερεύνητο** ← το μεγαλύτερο gap
7. **Threshold selection για LOW/MEDIUM/HIGH** — δεν υπάρχει principled method
8. **Human trust calibration** — ακόμα και well-calibrated AI communicates poorly to users

---

## Εφαρμογή στον Curator

| Εύρημα | Πράξη στον Curator |
|---|---|
| Raw token-probs unreliable post-RLHF | Μην χρησιμοποιείς logprobs απευθείας — verbalized confidence |
| Verbalized confidence καλύτερο για RLHF | Ήδη το κάνεις — αλλά calibrate the mapping |
| Temperature Scaling = cheapest fix | 200–500 labelled examples → fit temperature scalar T |
| Aleatoric vs epistemic distinction | "Αυτό το αρχείο είναι αμφίσημο" vs "δεν έχω δει τέτοιο" |
| LOW confidence → ask user | Σωστό design pattern = "selective prediction with deferral" |
| LLM4Tag architecture | 3 modules: candidate recall → generation → calibration |
| **User corrections = calibration signal** | Online calibration από user accept/reject — **δεν υπάρχει στη βιβλιογραφία για PIM** |

### Implementation Path

1. **Immediate**: Log κάθε απόφαση (model, confidence, category, outcome)
2. **Short-term** (50–200 labels): Temperature scaling σε validation set
3. **Medium-term**: Reliability diagram + SmoothECE tracking
4. **Advanced**: Conformal prediction → prediction sets αντί για single category
5. **Research**: Online calibration από user corrections — **genuinely novel**

---

## Sources
- [Survey of Confidence Estimation in LLMs](https://www.researchgate.net/publication/382633391_A_Survey_of_Confidence_Estimation_and_Calibration_in_Large_Language_Models)
- [ATS — arXiv 2409.19817](https://arxiv.org/abs/2409.19817)
- [CCPS — arXiv 2505.21772](https://arxiv.org/abs/2505.21772)
- [JUCAL — arXiv 2602.20153](https://arxiv.org/abs/2602.20153)
- [LLM4Tag — arXiv 2502.13481](https://arxiv.org/abs/2502.13481)
- [Taming Overconfidence — arXiv 2410.09724](https://arxiv.org/abs/2410.09724)
- [Uncertainty-Based Abstention — arXiv 2404.10960](https://arxiv.org/abs/2404.10960)
- [SmoothECE — ICLR 2024](https://proceedings.iclr.cc/paper_files/paper/2024/file/06cf4bae7ccb6ea37b968a394edc2e33-Paper-Conference.pdf)
- [KDD 2025 Survey](https://dl.acm.org/doi/10.1145/3711896.3736569)
- [CHI 2024 — Are You Really Sure?](https://dl.acm.org/doi/10.1145/3613904.3642671)
