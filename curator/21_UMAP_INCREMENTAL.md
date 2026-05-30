## ΜΕΡΟΣ 21: UMAP INCREMENTAL REFIT — ΠΛΗΡΗΣ ΑΝΑΛΥΣΗ
*(Έρευνα 2026-05-22 — 18 searches, 30+ sources)*

### Γιατί αυτό το θέμα είναι κρίσιμο

Στο Μέρος 15 (DenStream), ορίσαμε ότι το UMAP γίνεται fit μία φορά στο initialization batch (5000 αρχεία), και μετά χρησιμοποιούμε `.transform()` για κάθε νέο αρχείο. Παρόλα αυτά, αφήσαμε ανοιχτό το εξής: τι γίνεται μετά από 1000 νέα αρχεία ή 6-12 μήνες, όταν χρειαστεί **refit του UMAP από μηδέν**; Αυτό το μέρος απαντά εξαντλητικά.

---

### Α. Έχει το UMAP partial_fit / incremental API;

**Σύντομη απάντηση: ΟΧΙ — και δεν πρόκειται να αποκτήσει σύντομα.**

#### GitHub Issue #62 (lmcinnes/umap) — Partial fit request
- Ανοίχτηκε: Απρίλιος 2018 (8 χρόνια πριν)
- Τίτλος: "Partial fit / Incremental UMAP"
- Status: **Ακόμα ανοιχτό, χωρίς merged PR**
- Πρόταση: `partial_fit` method όπως στο scikit-learn `IncrementalPCA`, για lazy loading και online fitting
- Απάντηση maintainer: Θεωρητικά πολύπλοκο — το UMAP βασίζεται σε global nearest-neighbor graph που πρέπει να ξαναχτιστεί κάθε φορά που προστίθενται νέα σημεία

**Δεν υπάρχει `IncrementalUMAP` class πουθενά.** Δεν υπάρχει merged PR. Δεν υπάρχει roadmap.

#### Τι κάνει η BERTopic για streaming (workaround)
Η BERTopic (η πιο γνωστή εφαρμογή του UMAP+HDBSCAN pipeline) στο online mode **αντικαθιστά τελείως το UMAP** με `IncrementalPCA` (scikit-learn) για streaming:
```python
# BERTopic online mode:
from sklearn.decomposition import IncrementalPCA
dim_model = IncrementalPCA(n_components=5)  # αντικαθιστά UMAP
```
Η BERTopic documentation ρητά αναφέρει: "A major benefit of using `merge_models` compared to `partial_fit` is that you can keep using the original UMAP and HDBSCAN models."

**Συμπέρασμα:** Ακόμα και η πιο mature εφαρμογή του UMAP για streaming παραδέχεται ότι δεν λύνεται το incremental πρόβλημα — απλώς τον παρακάμπτει.

#### Paper: Efficient Batch-Incremental Classification Using UMAP (SpringerLink 2020)
- arXiv companion: ResearchGate 340820874
- Πρόταση: Successive embeddings on **disjoint batches** — κάθε batch γίνεται UMAP ανεξάρτητα
- Πρόβλημα: τα embeddings ανά batch είναι incompatible (διαφορετικοί χώροι) → χρειάζεται alignment μεταξύ τους
- **ΔΕΝ λύνει το core πρόβλημα** — απλώς κάνει batch classification, όχι unified manifold

---

### Β. Το Alignment Problem — Γιατί Ο Refit Σπάει Τα Πάντα

#### Τι ακριβώς συμβαίνει κατά τον refit

Έχουμε:
- **Παλιό UMAP (reducer_v1)**: fit σε αρχεία 1-5000 → παράγει συντεταγμένες `X_old` (10-dim)
- **Νέο UMAP (reducer_v2)**: fit σε αρχεία 1-6000 → παράγει συντεταγμένες `X_new` (10-dim)

Τα `X_old` και `X_new` **δεν είναι συγκρίσιμα**. Λόγοι:

1. **Rotation**: Το UMAP αρχικοποιεί τα embeddings τυχαία και τα βελτιστοποιεί. Δύο ανεξάρτητα runs παράγουν χώρους περιστραμμένους αυθαίρετα ως προς αλλήλους (αόριστη rotational symmetry)
2. **Reflection**: Ο χώρος μπορεί να "αντικατοπτριστεί" — ό,τι ήταν αριστερά τώρα είναι δεξιά
3. **Scale**: Διαφορετικά runs χρησιμοποιούν διαφορετικό scale ανάλογα με την πυκνότητα
4. **Translation**: Το κέντρο μάζας μπορεί να μετακινηθεί

**Πρακτική συνέπεια για τον Curator:**
- DenStream έχει micro-clusters με centroids σε `X_old` space
- Μετά refit: τα centroids είναι invalid, γιατί αντιπροσωπεύουν θέσεις στον παλιό χώρο
- Τα νέα αρχεία γίνονται embed σε `X_new` space → **misrouting στη routing φάση**
- Τα cluster assignments παλιών αρχείων (stored στη SQLite) αντιστοιχούν σε παλιό clustering → invalid για re-check

#### Το random_state=42 δεν σώζει

Από τα GitHub issues lmcinnes/umap:

**Issue #1080 (Setting a random state still leads to stochastic results):**
- Αιτία: παραλληλισμός — race conditions μεταξύ threads στη βελτιστοποίηση Numba
- Setting `random_state` **disables parallelism** για να διασφαλιστεί reproducibility
- Αλλά: αποτελέσματα εξαρτώνται από **version του Numba** εγκατεστημένο
- Cross-platform: `random_state=42` σε macOS ≠ `random_state=42` σε Linux

**Issue #153 (Cross-OS non-determinism):**
- Διαφορετικά OS → διαφορετικά αποτελέσματα ακόμα και με ίδιο `random_state`

**Issue #525 (Reproducibility):**
- Υποσχόμενη reproducibility: **μόνο στο ίδιο μηχάνημα, ίδια Numba version, ίδια δεδομένα, ίδια σειρά**
- Αν αλλάξεις έστω ένα αρχείο → partial non-determinism

**Συμπέρασμα:** `random_state=42` **ΔΕΝ** εγγυάται ίδιες συντεταγμένες μεταξύ refits. Ακόμα και αν η Numba version δεν αλλάξει, τα νέα αρχεία στο corpus αλλάζουν τον nearest-neighbor graph → διαφορετικό manifold.

#### GhostUMAP2 — Επίσημη Ποσοτικοποίηση Instability
- **arXiv:2507.17174** — "GhostUMAP2: Measuring and Analyzing (r,d)-Stability of UMAP"
- Ορισμός: ένα σημείο είναι **(r,d)-stable** αν "ghost" duplicates (με perturbation r στο initial position) παραμένουν εντός d στο τελικό embedding
- Αποτέλεσμα: Σημαντικό ποσοστό σημείων **δεν είναι stable** — ιδίως αυτά σε cluster boundaries
- **Σχέση με Curator**: τα αρχεία που αλλάζουν cluster κατά UMAP refit είναι ακριβώς αυτά με χαμηλό (r,d)-stability → σχέση με Ιδέα Α (File Biography)

---

### Γ. Procrustes Alignment — Η Κλασική Λύση

#### Τι είναι το Orthogonal Procrustes Problem

Δοθέντων δύο embeddings `X1` (παλιό) και `X2` (νέο) για τα **ίδια σημεία** (τα αρχεία που υπάρχουν και στα δύο runs):

```
Q* = argmin_{Q ∈ O(n)} || X1 @ Q - X2 ||_F
```

Λύση: `Q* = U @ V^T` όπου `U, S, V^T = SVD(X1^T @ X2)`

Αυτό βρίσκει τον βέλτιστο **ορθογώνιο μετασχηματισμό** (rotation + reflection) που ευθυγραμμίζει τον παλιό χώρο με τον νέο.

#### scipy.spatial.procrustes vs scipy.linalg.orthogonal_procrustes

```python
from scipy.linalg import orthogonal_procrustes
from scipy.spatial import procrustes

# Επιλογή A: scipy.linalg.orthogonal_procrustes (ΠΡΟΤΕΙΝΕΤΑΙ)
R, scale = orthogonal_procrustes(X_old, X_new)
# R = rotation matrix (n_components × n_components)
# X_old_aligned = X_old @ R  ← τώρα aligned με X_new space

# Επιλογή B: scipy.spatial.procrustes (κάνει επιπλέον standardization)
X_old_std, X_new_std, disparity = procrustes(X_old, X_new)
# ΠΡΟΣΟΧΗ: αυτό αλλάζει scale — χρήσιμο για σύγκριση, λιγότερο για alignment
```

**Σημαντικό:** Το `scipy.spatial.procrustes` εφαρμόζει standardization (mean centering + scale normalization) **πριν** τον μετασχηματισμό — αυτό αλλάζει τις πραγματικές συντεταγμένες. Χρήσιμο για benchmark μέτρηση ομοιότητας μεταξύ δύο runs, λιγότερο κατάλληλο για alignment που θα εφαρμοστεί στα data. Χρησιμοποίησε `scipy.linalg.orthogonal_procrustes` για actual alignment.

#### AlignedUMAP — Το Built-in UMAP Solution

Το umap-learn έχει ήδη `AlignedUMAP` class — αυτό είναι η **επίσημη λύση** για alignment μεταξύ refits:

```python
from umap import AlignedUMAP

# AlignedUMAP δέχεται λίστα datasets + relations (ποιο σημείο αντιστοιχεί σε ποιο)
reducer = AlignedUMAP(
    n_neighbors=15,
    n_components=10,
    min_dist=0.0,
    metric='cosine',
    alignment_regularisation=0.01,  # πόσο "σκληρό" είναι το alignment constraint
    alignment_window_size=3         # πόσα προηγούμενα snapshots λαμβάνει υπόψη
)

# Πώς δουλεύει:
# - Βελτιστοποιεί ταυτόχρονα N embeddings
# - Προσθέτει regularizer: related points πρέπει να είναι κοντά στα N embeddings
# - Εσωτερικά χρησιμοποιεί Procrustes για αρχική ευθυγράμμιση + MAPPER algorithm για refinement
```

**Τι κάνει καλύτερα από naive Procrustes:**
- Procrustes = linear transformation — εφαρμόζεται σε ολόκληρο το cloud σαν ενιαίο rigid body
- AlignedUMAP = τοπικός regularizer — επιτρέπει διαφορετική alignment σε διαφορετικές περιοχές
- Paper (Application of Aligned-UMAP, PMC 10318357, 2023): AlignedUMAP υπερτερεί naive Procrustes σε longitudinal biomedical data

**Περιορισμός:** AlignedUMAP χρειάζεται να ξέρει ποιο index στο old corpus αντιστοιχεί σε ποιο index στο new corpus → απαιτεί `relations` dict. Αυτό είναι feasible γιατί τα files έχουν stable IDs στη SQLite μας.

#### Wasserstein Procrustes (Advanced)
- arXiv:1805.11222 — "Unsupervised Alignment of Embeddings with Wasserstein Procrustes"
- Χρησιμοποιεί Wasserstein distance (optimal transport) αντί L2 → πιο robust σε outliers
- **Δεν χρειάζεται anchor points** — unsupervised
- Αλλά: πολύ πιο βαρύ υπολογιστικά. Για 5000 σημεία + O(n^2) Wasserstein → υπερβολικό

---

### Δ. Εναλλακτικές Λύσεις

#### Δ1. ParametricUMAP — Νευρωνικό Δίκτυο ως Embedding Function

- arXiv:2009.12981, Neural Computation MIT Press 2021
- Αντί optimization-based UMAP, εκπαιδεύει ένα neural network `f(x) → z`
- Το δίκτυο μαθαίνει **παραμετρικό mapping** → για νέα δεδομένα: απλώς forward pass
- **Fine-tuning support**: `transform_landmarked_pumap` — fit σε landmarks, transform υπόλοιπα

**Για incremental/refit:**
```python
from umap.parametric_umap import ParametricUMAP

# Αρχική εκπαίδευση:
pumap = ParametricUMAP(n_components=10)
pumap.fit(X_init)

# Fine-tuning με νέα δεδομένα (χωρίς full refit):
pumap.fit(X_new_batch)  # συνεχίζει από εκεί που έμεινε

# Πρόβλημα: UMAP loss invariant σε rotation/translation
# → embedding space μπορεί να drifts ακόμα και με fine-tuning
```

**Γιατί ΔΕΝ επιλύει εντελώς το alignment πρόβλημα:**
- Από UMAP docs ("Transforming New Data with Parametric UMAP"): "fine-tuning will start from where you left off, but the embedding space's structure may drift because the UMAP loss function is invariant to translation and rotation"
- Λύση: `landmarks` option — anchors some points → μειώνει drift αλλά δεν εξαλείφει
- **Υπολογιστικό κόστος**: Χρειάζεται TensorFlow/PyTorch — extra dependencies, slower inference vs. non-parametric UMAP

**Verdict για Curator:** ParametricUMAP είναι ενδιαφέρον αλλά προσθέτει complexity χωρίς να εγγυάται alignment. Ο Curator δεν χρειάζεται real-time embedding inference — η `.transform()` με 50-200ms είναι αποδεκτή.

#### Δ2. Re-cluster από Μηδέν (Nuclear Option)

Αντί να προσπαθήσουμε να ευθυγραμμίσουμε τους χώρους, **πετάμε τα παλιά cluster assignments** και ξαναχτίζουμε τα πάντα:

```
Refit procedure (nuclear option):
1. Re-embed όλα τα αρχεία (BGE-M3) — ΟΧΙ αν δεν έχουν αλλάξει
2. Fit νέο UMAP στο full corpus (5000-6000 αρχεία)
3. Fit νέο HDBSCAN
4. Re-name clusters με HERCULES
5. Re-initialize DenStream με νέα centroids
6. Παρουσίασε στον χρήστη τη "νέα οργάνωση" ως diff από την παλιά
```

**Εφικτό για 5000-6000 αρχεία:**
- UMAP (6000 × 1024 → 10): ~60-120s
- HDBSCAN (6000 × 10): ~3-5s
- Re-naming (HERCULES): ~2-5 min (LLM calls)
- **Συνολικά: ~5-8 λεπτά** — αποδεκτό για μία φορά ανά 6-12 μήνες

**Γιατί αυτό είναι η ΣΩΣΤΗ λύση για τον Curator:**
- Ο Curator τρέχει Deep Reorganize ήδη ως "batch operation"
- Ο χρήστης ήδη αναμένει το approval workflow μετά από refit
- Δεν υπάρχει requirement για real-time continuity — τα clusters αλλάζουν ούτως ή άλλως με νέα αρχεία
- Ο Procrustes alignment προσθέτει πολυπλοκότητα για να "σώσει" DenStream state — αλλά ο DenStream πρέπει να re-initialized ούτως ή άλλως μετά από refit

---

### Ε. Ποια Μέθοδος Είναι Κατάλληλη για τον Curator;

| Μέθοδος | Alignment | Πολυπλοκότητα | DenStream | Verdict |
|---|---|---|---|---|
| `random_state=42` μόνο | ❌ Δεν εγγυάται | Μηδαμινή | Invalid | ❌ Δεν αρκεί |
| naive Procrustes (scipy) | Μερική | Χαμηλή | Approximate | ⚠️ Acceptable workaround |
| AlignedUMAP | Καλή | Μέτρια | Approximate | ✅ Καλή αλλά overkill |
| ParametricUMAP | Μερική (drift) | Υψηλή | Approximate | ⚠️ Πολλές dependencies |
| Re-cluster από μηδέν | N/A (νέος χώρος) | Χαμηλή | Re-initialized | ✅ ΒΕΛΤΙΣΤΟ για Curator |

**Επιλογή για Curator: Full re-cluster + DenStream re-init** — απλούστερο, καθαρότερο, ήδη συμβατό με το approval workflow.

---

### 🏆 Πρακτική Πρόταση: Βήμα-Βήμα Refit Procedure

#### Πότε γίνεται refit
- **Trigger A (χρόνος):** 6-12 μήνες από τον τελευταίο refit
- **Trigger B (αριθμός):** >1000 νέα αρχεία έχουν γίνει routed μέσω DenStream
- **Trigger C (ποιότητα):** Το DenStream routing quality πέφτει (>20% αρχείων πηγαίνουν Review για 2+ εβδομάδες)
- **Manual:** Χρήστης επιλέγει "Deep Reorganize" από το UI

#### Βήμα 1: Snapshot της τρέχουσας κατάστασης
```python
# Αποθήκευσε το current state ΠΡΙΝ κάνεις οτιδήποτε
snapshot = {
    'timestamp': datetime.now().isoformat(),
    'cluster_assignments': dict(zip(file_ids, cluster_labels)),  # file_id → cluster_name
    'cluster_names': cluster_names_dict,  # cluster_id → "EPL326 Notes"
    'umap_version': 'v1',
}
save_json('/curator/snapshots/pre_refit_snapshot.json', snapshot)
```

#### Βήμα 2: Re-embed μόνο αρχεία που άλλαξαν
```python
# Cache check: αν mtime + size ίδια → χρήσιμοποίησε cached embedding
# Μόνο νέα ή τροποποιημένα αρχεία χρειάζονται re-embedding
changed_files = [f for f in all_files
                 if db.get_mtime(f) != os.path.getmtime(f)
                 or db.get_size(f) != os.path.getsize(f)]
# Για τα περισσότερα αρχεία (π.χ. παλιά PDFs): cached embedding, zero cost
```

#### Βήμα 3: Fit νέο UMAP από μηδέν
```python
import umap
import numpy as np

all_embeddings = load_all_embeddings_from_db()  # (N, 1024) — includes νέα αρχεία

reducer_new = umap.UMAP(
    n_components=10,
    n_neighbors=15,
    min_dist=0.0,
    metric='cosine',
    random_state=42,           # consistency within same run
    low_memory=False,
    transform_queue_size=8.0   # better accuracy για transform()
)
X_new_10d = reducer_new.fit_transform(all_embeddings)

# Αποθήκευσε τον νέο reducer
import joblib
joblib.dump(reducer_new, '/curator/models/umap_reducer_v2.pkl')
```

#### Βήμα 4: Fit νέο HDBSCAN
```python
import hdbscan

clusterer_new = hdbscan.HDBSCAN(
    min_cluster_size=10,
    min_samples=5,
    cluster_selection_method='eom',
    metric='euclidean',
    prediction_data=True,
    core_dist_n_jobs=-1
)
clusterer_new.fit(X_new_10d)

new_labels = clusterer_new.labels_  # -1 = noise/antilibrary
new_probs = clusterer_new.probabilities_  # soft membership
new_outlier = clusterer_new.outlier_scores_  # GLOSH → antilibrary
```

#### Βήμα 5: Re-name clusters με HERCULES (μόνο αν δομή άλλαξε)
```python
# Σύγκριση παλιών vs νέων clusters:
# Αν cluster_count αλλαχθεί >20% ή silhouette βελτιώθηκε >0.05 → re-name
# Αλλιώς: διατήρησε τα παλιά ονόματα (less disruption)

old_cluster_count = len(set(old_labels)) - 1  # εξαιρεί -1
new_cluster_count = len(set(new_labels)) - 1

if abs(new_cluster_count - old_cluster_count) / old_cluster_count > 0.2:
    # Significant structural change → full re-naming via HERCULES
    new_cluster_names = hercules_name_all_clusters(X_new_10d, new_labels)
else:
    # Minor change → try to match old names to new clusters
    new_cluster_names = match_names_to_new_clusters(old_cluster_names, X_new_10d, new_labels)
```

#### Βήμα 6: Re-initialize DenStream
```python
# Ο DenStream πρέπει να re-initialized — δεν υπάρχει migration path
# Χρησιμοποίησε τα νέα 10d coordinates ως seed

from epsilon_calculation import calculate_epsilon_from_knn
epsilon = calculate_epsilon_from_knn(X_new_10d, k=3)

denstream_new = DenStream(
    decaying_factor=0.0011,   # 6-month half-life
    beta=0.5,
    mu=3,
    epsilon=epsilon,
    n_samples_init=min(500, len(X_new_10d))
)

# Seed με τα υπάρχοντα αρχεία (σιωπηλή initialization)
for i, vec in enumerate(X_new_10d):
    d = {j: float(v) for j, v in enumerate(vec)}
    denstream_new.learn_one(d)

# Αποθήκευσε
joblib.dump(denstream_new, '/curator/models/denstream_v2.pkl')
```

#### Βήμα 7: Compute diff και παρουσίασε στον χρήστη
```python
# Κάθε αρχείο: συγκρίνουμε old cluster assignment με new
diff = []
for file_id in all_file_ids:
    old_cluster = snapshot['cluster_assignments'].get(file_id)
    new_cluster = new_cluster_names.get(new_labels[file_id_to_idx[file_id]])

    if old_cluster != new_cluster:
        diff.append({
            'file': file_id,
            'old_folder': old_cluster,
            'new_folder': new_cluster,
            'confidence': float(new_probs[file_id_to_idx[file_id]]),
            'reason': 'Cluster structure updated after periodic refit'
        })

# Diff: μόνο τα αρχεία που αλλάζουν folder → approval UI
# Αρχεία που μένουν στο ίδιο folder: silent, no action needed
```

#### Βήμα 8: Execute μετά approval
```python
# Ο χρήστης approves/rejects τις αλλαγές
# Execute: όπως το standard Deep Reorganize (dependency-ordered moves + undo log)
```

#### Βήμα 9: Atomic swap
```python
# Μόνο αν ο χρήστης approve:
os.rename('/curator/models/umap_reducer_v2.pkl', '/curator/models/umap_reducer_current.pkl')
os.rename('/curator/models/denstream_v2.pkl', '/curator/models/denstream_current.pkl')

# Διατήρησε τα παλιά αρχεία ως backup:
os.rename('/curator/models/umap_reducer_v1.pkl', '/curator/models/umap_reducer_v1_backup.pkl')
```

---

### Τι Γίνεται με Τα "Παλιά" Coordinates στη SQLite

**Δεν χρειάζεται να σώσεις τα παλιά 10d coordinates στη SQLite.** Η SQLite αποθηκεύει τα **1024d raw embeddings** (από το BGE-M3). Τα 10d UMAP coordinates είναι παραγόμενα/ephemeral — υπολογίζονται live από τον reducer. Μετά από refit, απλώς ξαναϋπολογίζεις τα 10d για όλα τα αρχεία χρησιμοποιώντας τον νέο reducer — αυτό είναι ήδη το Βήμα 3 παραπάνω.

**Schema SQLite (Δεν αλλάζει):**
```sql
-- Αποθηκεύεις:
-- file_path, embedding_1024d (BLOB), cluster_id, cluster_name, confidence, last_seen_mtime, last_seen_size
-- ΟΧΙ: umap_coord_10d — παράγεται on-the-fly
```

---

### Ανοιχτά Ζητήματα

| Ζήτημα | Status |
|---|---|
| Incremental UMAP (partial_fit) | Ανύπαρκτο — δεν πρόκειται να αλλάξει σύντομα |
| random_state determinism | Μόνο on same machine+OS+Numba version |
| AlignedUMAP για periodic snapshots | Feasible αλλά overkill για personal use |
| DenStream migration μεταξύ refits | Δεν υπάρχει migration — πάντα re-init |
| Approval UI για refit diff | Ίδιο με Deep Reorganize — ήδη σχεδιασμένο |

---

**Sources Μέρος 21:**
- GitHub Issue #62 lmcinnes/umap (Partial fit request, 2018) — https://github.com/lmcinnes/umap/issues/62
- GitHub Issue #1080 (random_state stochastic) — https://github.com/lmcinnes/umap/issues/1080
- GitHub Issue #153 (cross-OS non-determinism) — https://github.com/lmcinnes/umap/issues/153
- GitHub Issue #525 (reproducibility) — https://github.com/lmcinnes/umap/issues/525
- GitHub Issue #1108 (semi-deterministic despite random_state) — https://github.com/lmcinnes/umap/issues/1108
- UMAP Reproducibility docs — https://umap-learn.readthedocs.io/en/latest/reproducibility.html
- AlignedUMAP basic usage — https://umap-learn.readthedocs.io/en/latest/aligned_umap_basic_usage.html
- AlignedUMAP time varying demo — https://umap-learn.readthedocs.io/en/latest/aligned_umap_politics_demo.html
- Application of Aligned-UMAP to longitudinal biomedical studies (PMC 10318357, 2023) — https://pmc.ncbi.nlm.nih.gov/articles/PMC10318357/
- GhostUMAP2: Measuring (r,d)-Stability of UMAP — https://arxiv.org/abs/2507.17174
- Efficient Batch-Incremental Classification Using UMAP (SpringerLink 2020) — https://link.springer.com/chapter/10.1007/978-3-030-44584-3_4
- BERTopic Online Topic Modeling docs — https://maartengr.github.io/BERTopic/getting_started/online/online.html
- ParametricUMAP paper arXiv:2009.12981 — https://arxiv.org/pdf/2009.12981
- ParametricUMAP transform docs — https://umap-learn.readthedocs.io/en/latest/transform_landmarked_pumap.html
- scipy.linalg.orthogonal_procrustes — https://www.tutorialspoint.com/scipy/scipy_orthogonal_procrustes_function.htm
- When Embedding Models Meet: Procrustes Bounds — https://arxiv.org/html/2510.13406v1
- Unsupervised Alignment with Wasserstein Procrustes arXiv:1805.11222 — https://arxiv.org/pdf/1805.11222
- BERTopic GitHub Issue #2098 (partial_fit warnings) — https://github.com/MaartenGr/BERTopic/issues/2098
- Stop Your UMAP From Moving (DEV Community) — https://dev.to/afuji/stop-your-umap-from-moving-the-pipeline-randomness-youre-missing-gje

---
