# Concept Drift Detection & Cluster Stability — File Biography
**Compiled:** May 2026 | **Relevance:** Academic grounding για Curator's File Biography system

---

## Κεντρικό Εύρημα

**Το "File Biography" ως per-item longitudinal history είναι genuinely novel.** Υπάρχει academic foundation για τα επιμέρους components, αλλά η σύνθεση σε unified design pattern δεν υπάρχει στη βιβλιογραφία.

---

## Concept Drift — Taxonomy

| Τύπος | Ορισμός | Στον Curator |
|---|---|---|
| **Abrupt** | Instantaneous change | Νέο project ξεκινά |
| **Gradual** | Old fades as new grows | Αλλαγή εργασίας |
| **Incremental** | Slow monotonic shift | Evolving interests |
| **Recurring** | Old concept returns | Ετήσια φορολογικά |

---

## Drift Detection Methods

### ADWIN (Adaptive Windowing) — Foundational
- Variable-size sliding window. Συγκρίνει δύο sub-windows με Hoeffding bound
- Guaranteed false positive + detection delay bounds
- Available: River (Python), scikit-multiflow

**ADWIN-U (2025)** — Unsupervised extension. Λειτουργεί χωρίς labeled data. Νέο metric: BAR (Balanced Accuracy by Amount of Requested Labeled Data) — tradeoff accuracy vs labeling burden. Direct application σε human-in-the-loop review.

### STUDD (Student-Teacher Unsupervised Drift Detection)
Teacher model trained on historical data → student mimics. Αν student mimicking loss ανεβεί → drift detected. **Χωρίς labels**, conservative (low false positive). Suitable για embedding-space drift σε file representations.

### Imbalanced Cluster Drift (arXiv 2603.06757, 2025)
Key insight: **μεγάλα clusters maskάρουν drift σε μικρά clusters**. Αν έχεις πολλά "work" files και λίγα "personal", drift στο personal cluster είναι invisible χωρίς per-cluster analysis. Curator πρέπει να το χειριστεί ρητά.

---

## Cluster Stability Metrics

### Global (Cluster-Level)
| Metric | Χρήση |
|---|---|
| **ARI (Adjusted Rand Index)** | Pairwise agreement between two clusterings. 1 = identical, 0 = random. Rolling stability score |
| **NMI (Normalized Mutual Information)** | Information overlap between clusterings |
| **iXB (incremental Xie-Beni)** | Intra-cluster compactness / inter-cluster separation. O(1) per point. "Distress signals" on degradation |

**iXB with forgetting factor:** `w(t) = w₀ · λ^t` — formal analogue Ebbinghaus σε cluster quality monitoring.

### Evolutionary Clustering Framework (Chi et al., 2007)
```
C_total = α · C_snapshot + (1-α) · C_temporal
```
- **Snapshot Cost (SC)**: Πόσο καλά fits το clustering στα current data
- **Temporal Cost (TC)**: Πόσο deviates από previous clustering

Ο **trade-off parameter α** = stability/plasticity lever. Για τον Curator: high α → react fast to new files; low α → preserve historical structure.

### Cluster Event Taxonomy (E-Stream)
5 canonical events που πρέπει να detect κάθε streaming cluster tracker:
1. **Appearance** — νέο cluster εμφανίζεται
2. **Disappearance** — cluster πεθαίνει
3. **Self-evolution** — cluster shifts χωρίς να split
4. **Merge** — δύο clusters γίνονται ένα
5. **Split** — ένα cluster fork σε δύο

Αυτά είναι τα "life events" του File Biography. Κάθε update = ένα από αυτά τα events.

---

## Per-Item Stability Scores

### CAKE (2025) — arXiv 2602.18435 ← Most Relevant
Evaluates κάθε individual data point:
1. **Assignment stability**: Πόσο consistently lands στο ίδιο cluster across runs
2. **Local geometric fit**: Πόσο καλά fits το local geometry

Combined score [0,1] per item. Items near 0 = ambiguous/unstable → flag for review. Αυτό είναι το πιο κοντινό existing framework στο instability signal του Curator.

### Item Consensus Score
Fraction of runs που item assigned στο modal cluster. 1.0 = perfectly stable; 0.5 = borderline.

---

## DenStream State of the Art (2025/2026)

**Online Density-Based Clustering for Narrative Monitoring (arXiv 2601.20680, 2025):**
Direct comparison DenStream vs DBSTREAM vs others για real-time text/document clustering.
> **DenStream achieved the strongest overall performance.**

**DenStream's fading function:**
```
f(t) = 2^(-λt)
```
= **Formal Ebbinghaus forgetting curve** σε clustering algorithm. Weight ενός micro-cluster decays αν δεν reinforced από νέα data.

### Alternatives
| Variant | Key Feature |
|---|---|
| **DBSTREAM** | Shared-density graph — arbitrary cluster shapes |
| **GeoDenStream** | Entity tracking, "recovers incorrectly labeled noise" |
| **E-Stream** | Real-time feature ranking πριν DenStream |
| **MCMSTStream** | Minimum spanning tree — highest ARI (0.75) σε benchmarks |

### Incremental HDBSCAN
**HDBSCAN = fundamentally batch**. Streaming variants (FISHDBC, Bubble-Tree) exist αλλά ο 2025 benchmark επιβεβαιώνει: **batch HDBSCAN not suitable for real-time streaming**. DenStream/DBSTREAM = recommended.

---

## Ebbinghaus & Clustering Connection

DenStream decay: `f(t) = 2^(-λt)` = Curator's FLRS: `R(t) = e^(-t/S)` — **και τα δύο είναι exponential decay functions.** DenStream's λ = inverse Ebbinghaus stability S.

**Recommendation systems (2025)** "Incorporating Forgetting Curve and Memory Replay": Fuses Ebbinghaus με interaction frequency + rating density. Items revisited → weight reinforced. Items untouched → decay toward uncertainty.

**Εφαρμογή στο File Biography:**
```
w(t) = w₀ · 2^(-λ · days_since_last_access)
```
Αρχείο που δεν ανοίχτηκε 6 μήνες → cluster assignment uncertain → re-evaluation trigger.

---

## Human-in-the-Loop Review

**ACM WWW 2018 — "A Framework for Human-in-the-Loop Monitoring of Concept-Drift Detection":**
Canonical paper. Key insight: drift detectors generate **candidates, not verdicts**. Humans confirm.

**Design principles:**
1. Drift detection → candidate, όχι αυτόματη ενέργεια
2. Explainability: γιατί detected το drift (SHAP-style)
3. Closed-loop: user decision → update DenStream micro-cluster weights
4. Selective flagging — review fatigue αν πολλά files flagged

**Curator's approach** (instability score < threshold → flag for review) = **valid unsupervised instability signal με no direct precedent** στη βιβλιογραφία.

---

## "File Biography" — Novelty Assessment

| Component | Academic support | Novel? |
|---|---|---|
| Per-file assignment history | CAKE, CLOSE toolkit, trajectory clustering | Partial — existing σε batch; novel σε streaming |
| Exponential decay on confidence | DenStream fading, Ebbinghaus RecSys | Established — DenStream already does this |
| Instability → user review trigger | ACM WWW 2018 HitL framework | Concept exists; per-item file trigger = novel |
| Cluster event taxonomy | E-Stream (2007) | Established |
| **Per-item longitudinal history as first-class object** | **Nowhere** | **Genuinely novel** |

---

## Recommended Architecture για Curator

```python
FileBiography = {
    "file_id": str,
    "assignment_history": [(timestamp, cluster_id, confidence_weight), ...],
    "current_cluster": cluster_id,
    "stability_score": float,         # CAKE-style [0, 1]
    "drift_events": [(timestamp, event_type), ...],  # E-Stream taxonomy
    "last_reinforced": timestamp,      # για Ebbinghaus decay
    "review_flag": bool
}

# Stability computation
def stability(file, N=10):
    history = file.assignment_history[-N:]
    if not history: return 1.0
    modal_cluster = max(set(h[1] for h in history),
                       key=lambda c: sum(1 for h in history if h[1]==c))
    return sum(1 for h in history if h[1]==modal_cluster) / len(history)
    # < 0.6 → flag for review
```

**Layer stack:**
1. **DenStream** — streaming clustering engine (decay λ = function of file activity half-life)
2. **File Biography** — per-item assignment history data structure
3. **Stability Score** — item consensus approach (last N snapshots)
4. **ADWIN-U** — population-level drift detection
5. **HitL Review** — stability < threshold + drift flag → user review με visual biography
6. **Forgetting Curve** — confidence weight decays με days_since_last_access

---

## Open Problems (Relevant to Curator)

1. **Per-item drift attribution** — ποια αρχεία προκαλούν instability; Unsolved
2. **Stability-plasticity calibration** — optimal λ για personal data = user-specific
3. **Masking effect** — small clusters (niche categories) under-detected
4. **Semantic ID stability** — cluster IDs consistent across re-clusterings (ABCDE 2024 begins to address)
5. **Review fatigue** — πόσα files να flag; No principled answer για PIM
6. **Cold-start stability** — new files have no history
7. **Temporal granularity** — per-day vs per-access biography; No principled answer

---

## Sources
- [Online Density-Based Clustering — arXiv 2601.20680](https://arxiv.org/html/2601.20680)
- [CAKE — arXiv 2602.18435](https://arxiv.org/pdf/2602.18435)
- [ADWIN-U — Springer 2025](https://link.springer.com/article/10.1007/s10115-025-02523-1)
- [STUDD — arXiv 2103.00903](https://arxiv.org/abs/2103.00903)
- [Imbalanced Cluster Drift — arXiv 2603.06757](https://arxiv.org/pdf/2603.06757)
- [Evolutionary Spectral Clustering — Microsoft Research](https://www.microsoft.com/en-us/research/publication/evolutionary-spectral-clustering-incorporating-temporal-smoothness/)
- [E-Stream — SpringerLink](https://link.springer.com/chapter/10.1007/978-3-540-73871-8_58)
- [HitL Concept Drift — ACM WWW 2018](https://dl.acm.org/doi/10.1145/3184558.3186343)
- [Online Cluster Validity — arXiv 1801.02937](https://arxiv.org/pdf/1801.02937)
- [Forgetting Curve RecSys — ScienceDirect 2025](https://www.sciencedirect.com/science/article/abs/pii/S0306457325000123)
- [ABCDE — arXiv 2409.18254](https://arxiv.org/pdf/2409.18254)
- [DBSTREAM — River docs](https://riverml.xyz/dev/api/cluster/DBSTREAM/)
- [GeoDenStream — ResearchGate](https://www.researchgate.net/publication/343453280_GeoDenStream_An_improved_DenStream_clustering_method_for_managing_entity_data_within_geographical_data_streams)
- [Outliers on Cluster Evolution — Nature 2024](https://www.nature.com/articles/s41598-024-75928-7)
