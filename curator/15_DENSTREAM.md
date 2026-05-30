## ΜΕΡΟΣ 15: DENSTREAM PARAMETER TUNING
*(Έρευνα 2026-05-22 — 21 searches + 7 fetches, 20 sources)*

### 🔴 ΚΡΙΣΙΜΟ: River DenStream έχει 7 confirmed bugs

Συγκεκριμένα ο πιο κρίσιμος: **weighted sum decay bug** — το decay εφαρμόζεται στο total accumulated sum αντί για individual point weights, κάνει τα `mc.center`, `mc.radius`, `mc.weight` λάθος.

**Χρησιμοποίησε αντί river:**
- `pip install` από: https://github.com/issamemari/DenStream ή https://github.com/waylongo/denstream

Για slow streams (file organizer = 5 files/day), το impact είναι tolerable. Αλλά γνωρίζουμε ότι υπάρχει.

---

### Parameters — Τι σημαίνουν

| Parameter | River name | Default | Χρήση |
|---|---|---|---|
| Decay factor | `decaying_factor` | 0.25 | Πόσο γρήγορα ξεθωριάζουν παλιά clusters |
| Outlier multiplier | `beta` | 0.75 | Διαχωρισμός outlier vs core |
| Weight threshold | `mu` | 2 | Πόσα files χρειάζεται ένα cluster για να "solidify" |
| Neighborhood radius | `epsilon` | 0.02 | Max radius micro-cluster |
| Init buffer | `n_samples_init` | 1000 | Files πριν ξεκινήσει η online φάση |
| Arrival rate | `stream_speed` | 100 | Files ανά logical time unit |

---

### epsilon — k-NN Distance Plot Method

```python
from sklearn.neighbors import NearestNeighbors
from kneed import KneeLocator  # pip install kneed
import numpy as np

k = 3  # = target mu value
nn = NearestNeighbors(n_neighbors=k, metric='euclidean')
nn.fit(X_umap_10d)  # το 5000-file initialization batch μετά UMAP
dists, _ = nn.kneighbors(X_umap_10d)
k_dists_sorted = np.sort(dists[:, -1])

kneedle = KneeLocator(
    np.arange(len(k_dists_sorted)), k_dists_sorted,
    S=1.0, curve='convex', direction='increasing'  # S=1.5 για μεγαλύτερο epsilon
)
epsilon = k_dists_sorted[kneedle.knee]
print(f"Recommended epsilon: {epsilon:.4f}")
# Expected range για 10-dim UMAP: 0.3–1.5
```

---

### decaying_factor (lambda) — Μαθηματική σχέση

```
half-life_in_files = 1 / lambda    (με stream_speed=1)
```

| Επιθυμητό fade | lambda |
|---|---|
| 3 μήνες (5 files/day = ~450 files) | 0.0022 |
| **6 μήνες (~900 files)** | **0.0011** ← recommended |
| 1 χρόνο (~1825 files) | 0.00055 |
| 2 χρόνια (~3650 files) | 0.00027 |

**Πρακτικά:** `lambda = 1 / (daily_files * days)`

---

### Συνιστώμενη Configuration

```python
from river import cluster  # ή issamemari/DenStream για bugfix version

# Calculate epsilon from k-NN plot first (see above)
# Then:
denstream = cluster.DenStream(
    decaying_factor=0.0011,  # 6-month half-life at 5 files/day
    beta=0.5,
    mu=3,                    # 3 files minimum για real cluster
    epsilon=epsilon,          # από k-NN plot
    n_samples_init=500,      # seed με πρώτα 500 files
    stream_speed=1           # 1 file = 1 time unit
)
```

---

### UMAP transform() για νέα αρχεία

```python
# Fit μία φορά στο initialization batch:
reducer = umap.UMAP(n_components=10, n_neighbors=15, min_dist=0.0,
                    metric='cosine', random_state=42,
                    transform_queue_size=8.0)  # 8.0 αντί default 4.0 για accuracy
reducer.fit(X_init_1024d)

# Για κάθε νέο αρχείο:
vec_10d = reducer.transform(new_embedding.reshape(1, -1))[0]
# ~50-200ms ανά αρχείο μετά JIT warmup — αποδεκτό
```

**Πρόγραμμα refit:** Κάθε 6-12 μήνες ή μετά ~1000 νέα αρχεία. Μετά από refit → re-project όλα τα embeddings + re-init DenStream.

---

### Online Routing Pattern

```python
def route_new_file(embedding_1024d, cosine_fallback_threshold=0.82):
    vec_10d = reducer.transform(embedding_1024d.reshape(1, -1))[0]
    d = {i: float(v) for i, v in enumerate(vec_10d)}
    
    denstream.learn_one(d)
    cluster_id = denstream.predict_one(d)
    
    # River bug #4: predict_one επιστρέφει cluster ακόμα και για distant points
    # → ελέγχουμε cosine similarity με centroid ως fallback
    if cluster_id != -1:
        centroid = np.array(list(denstream.p_micro_clusters[cluster_id].center.values()))
        cosine_sim = np.dot(vec_10d, centroid) / (np.linalg.norm(vec_10d) * np.linalg.norm(centroid))
        if cosine_sim < cosine_fallback_threshold:
            cluster_id = -1  # treat as new/unmatched
    
    return cluster_id  # -1 = new micro-cluster / goes to Review
```

---

### Alternatives αν DenStream bugs είναι dealbreaker

**Cosine centroid routing (simpler fallback):**
```python
def route_simple(vec_10d, centroids, threshold=0.82):
    if not centroids: return None
    sims = [np.dot(vec_10d, c)/(np.linalg.norm(vec_10d)*np.linalg.norm(c)) for c in centroids]
    best = np.argmax(sims)
    return best if sims[best] >= threshold else None  # None = new cluster
```
Χωρίς temporal decay, χωρίς automatic cluster merging. Πρέπει να τα διαχειριστείς manually.

---
