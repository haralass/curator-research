## ΜΕΡΟΣ 13: UMAP + HDBSCAN PARAMETER TUNING
*(Έρευνα 2026-05-22 — 30 searches, 28 sources)*

### 🔴 ΚΡΙΣΙΜΟ: UMAP είναι ΥΠΟΧΡΕΩΤΙΚΟ πριν HDBSCAN

HDBSCAN FAQ ρητά: "can generally do well up to ~50-100 dimensions, performance sees significant decreases beyond that."
Στα 1024 dims (multilingual-e5/BGE-M3) = **10-20× πάνω από το threshold**. Χωρίς UMAP → mostly noise ή 1 giant cluster.
*(IEEE 2021: UMAP βελτίωσε HDBSCAN κατά 60 percentage points on text. Springer: UMAP+HDBSCAN consistently outperforms HDBSCAN alone.)*

---

### UMAP Parameters (για clustering, όχι visualization)

```python
umap_model = UMAP(
    n_components=10,    # 10 για clustering (BERTopic=5, UMAP docs=10-50)
    n_neighbors=15,     # BERTopic default — balance local/global
    min_dist=0.0,       # ΚΡΙΣΙΜΟ: 0.0 για density clustering (default 0.1 = λάθος)
    metric='cosine',    # text embeddings = directional
    random_state=42,    # reproducibility
    low_memory=False    # faster on Apple Silicon
)
```

**Γιατί min_dist=0.0:** Επιτρέπει στο UMAP να "pack" similar points όσο πιο κοντά γίνεται → δημιουργεί dense islands για το HDBSCAN. Default 0.1 spreads για αισθητική, καταστρέφει clustering.

**Tuning n_components:**
- Αν πολλά clusters χάνονται → αύξησε σε 15-20
- Αν HDBSCAN παράγει μία γιγαντιαία blob → μείωσε σε 5-7
- Ποτέ >50 (ακυρώνει τον λόγο ύπαρξης UMAP)

**Optional speedup:** PCA 1024→100 dims πριν UMAP (μειώνει UMAP time στο μισό):
```python
from sklearn.decomposition import PCA
embeddings_pca = PCA(n_components=100, random_state=42).fit_transform(embeddings)
```

---

### HDBSCAN Parameters

```python
clusterer = hdbscan.HDBSCAN(
    min_cluster_size=10,             # BERTopic default — επιτρέπει μικρούς clusters (hobby folder=8 files)
    min_samples=5,                   # ~half min_cluster_size → λιγότερα noise points
    cluster_selection_method='eom',  # try 'leaf' αν EOM δημιουργεί 1 γιγαντιαίο cluster
    metric='euclidean',              # σωστό ΜΕΤΑ από UMAP reduction
    cluster_selection_epsilon=0.0,   # άφησε 0.0 αρχικά (raise σε 0.1-0.3 αν micro-fragmentation)
    prediction_data=True,            # ΑΠΑΡΑΙΤΗΤΟ για soft clustering + noise handling
    core_dist_n_jobs=-1              # παράλληλο σε όλα τα cores (μόνο hdbscan package, όχι sklearn)
)
```

---

### Noise Handling — Το Τελικό Pattern

```python
# Soft clustering για noise points + GLOSH για antilibrary
soft = hdbscan.all_points_membership_vectors(clusterer)
outlier_scores = clusterer.outlier_scores_
labels = clusterer.labels_  # -1 = noise

ANTILIBRARY_THRESHOLD = 0.7  # tune empirically

final_labels = []
for i, label in enumerate(labels):
    if label == -1:
        if outlier_scores[i] > ANTILIBRARY_THRESHOLD:
            final_labels.append('_Antilibrary')  # genuinely unique file
        else:
            final_labels.append(np.argmax(soft[i]))  # assign to nearest cluster
    else:
        final_labels.append(label)
```

**GLOSH outlier scores** (`clusterer.outlier_scores_`) = **direct feed για antilibrary detection** — files με GLOSH >0.7 είναι genuinely unique, όχι απλώς noise. Αυτό συνδέεται ωραία με το kMDItemUseCount==0 signal.

**Target noise %:** 5-20%. >25% = min_cluster_size πολύ μεγάλο ή UMAP misconfigured.

---

### Performance στα 5000 files (Apple Silicon)

| Step | Time | Memory |
|---|---|---|
| PCA 1024→100 (optional) | ~2s | negligible |
| UMAP 5000×1024→10 | ~30-90s | <1 GB |
| HDBSCAN 5000×10 | ~2-5s | <200 MB |
| **Total** | **~1-2 min** | **<2 GB** |

5000 points = **trivially small** για HDBSCAN. Δεν χρειάζεσαι pynndescent approximate NN σε αυτή την κλίμακα.

---

### Tuning Ladder (ένα parameter τη φορά)

| Πρόβλημα | Fix |
|---|---|
| Πολύ λίγοι clusters (3-5 giant blobs) | Μείωσε min_cluster_size σε 5-7, n_neighbors σε 10 |
| Πάρα πολλοί clusters (100+) | Αύξησε min_cluster_size σε 15-20, n_neighbors σε 30 |
| >30% noise | Μείωσε min_samples σε 2-3, min_cluster_size σε 5 |
| <5% noise (όλα lumped) | Αύξησε min_samples, δοκίμασε cluster_selection_method='leaf' |
| Micro-fragmentation (50+ tiny clusters) | Set cluster_selection_epsilon=0.1-0.3 |
| Semantically mixed clusters | Αύξησε n_components σε 15-20, n_neighbors σε 30 |

---

### HDBSCAN vs K-Means (πότε ποιο)

- **arXiv:2511.19350:** HDBSCAN υστερεί vs K-Means σε **short text** (titles, tweets) — αλλά τα δικά μας αρχεία είναι mixed-length, όχι short-text-only
- **HDBSCAN κερδίζει όταν:** unknown K (η περίπτωσή μας), variable density clusters, noise/outliers επιθυμητοί (antilibrary!)
- **K-Means κερδίζει όταν:** γνωστό K, uniform density, θέλεις κάθε αρχείο assigned

**Verdict:** HDBSCAN είναι η σωστή επιλογή για file organizer — το noise = antilibrary είναι feature, όχι bug.

---
