## ΜΕΡΟΣ 23: UMAP STABILITY — GHOSTUMAP2, ALIGNEDUMAP, ANCHOR POINTS
*(Έρευνα 2026-05-22 — 12 searches, 20+ sources)*

### Α. GhostUMAP2 — Τυπική Μέτρηση Αστάθειας

#### Α1. Τι είναι το GhostUMAP2

Το **GhostUMAP2** (arXiv:2507.17174, Ιούλιος 2025) είναι ένα framework που **ποσοτικοποιεί επίσημα** την αστάθεια των αποτελεσμάτων του UMAP. Δεν προτείνει διορθωτικές λύσεις — μετράει με ακρίβεια πόσο ασταθής είναι κάθε σημείο στο embedding.

**Κεντρική παρατήρηση:** Το UMAP χρησιμοποιεί δύο stochastic elements κατά την βελτιστοποίηση:
1. **Αρχικές θέσεις** (initial projection positions) — τυχαίος θόρυβος για αποφυγή local minima
2. **Negative sampling** — τυχαία δείγματα σε κάθε iteration αντί για όλα τα ζεύγη

Αυτά τα δύο elements σημαίνουν ότι "the projections of data points are determined mostly by chance rather than reflecting neighboring structures" για ένα υποσύνολο σημείων — ιδίως αυτά που βρίσκονται σε cluster boundaries.

#### Α2. Η (r,d)-Stability Metric — Ορισμός

Η βασική ιδέα: εισάγουμε **"ghosts"** — duplicates ενός σημείου `xi` που αρχικοποιούνται εντός κύκλου ακτίνας `r` γύρω από την αρχική θέση `yi`:

```
gik ← SampleCircle(yi, r)   # M ghosts per point, uniform σε κύκλο ακτίνας r
```

Όλα τα ghosts μοιράζονται το ίδιο high-dimensional vector `xi` — διαφέρουν μόνο στο starting position του UMAP optimization. Μετά το optimization:

```
di := max_k || yi' - gik' ||₂
```

Δηλαδή `di` = η μέγιστη απόσταση μεταξύ του τελικού embedding `yi'` και των τελικών θέσεων των ghosts `gik'`.

**Ορισμός (r,d)-stability:** Ένα σημείο είναι **(r,d)-stable** αν `di ≤ d` — δηλαδή αν τα ghosts (που ξεκίνησαν με perturbation `r`) καταλήξουν εντός `d` από το original.

**Τυπικές τιμές (από paper):**
- Default: `r = 0.1`, `d = 0.1`
- Πάνω από 90% των σημείων έχουν `di < 0.01` — δηλαδή είναι πολύ σταθερά
- Με `d = 0.01`: ~90% stable. Με `d = 0.1`: ~99% stable — μόνο ~1% ταξινομείται ως unstable

**Ερμηνεία για τον Curator:** Τα αρχεία με υψηλό `di` (unstable) είναι αυτά που αλλάζουν cluster κατά UMAP refit — ακριβώς αυτά που έχουν χαμηλό HDBSCAN `probabilities_` score. Υπάρχει ισχυρή σχέση μεταξύ (r,d)-instability και cluster boundary proximity.

#### Α3. Αλγόριθμος & Υπολογιστικό Κόστος

**Τριφασικός αλγόριθμος:**
1. **Ghost generation**: παραγωγή M ghosts ανά σημείο (default M=16) σε κύκλο ακτίνας r
2. **Joint layout optimization**: τα originals βελτιστοποιούνται ανεξάρτητα, τα ghosts χωριστά ("ghosts invisible to original points" — δεν επηρεάζουν το κύριο embedding)
3. **Adaptive dropping**: EMA (β=0.2) για εντοπισμό stable ghosts → αφαίρεσή τους νωρίς για επιτάχυνση

**Κόστος χωρίς βελτιστοποίηση:** ~11.9× overhead vs baseline UMAP (αναλογικά με τα M ghosts)
**Κόστος με adaptive dropping:** ~5.0× overhead (2.4× speedup vs no-reduction)

Για 5000 αρχεία: αν το baseline UMAP παίρνει ~90s, το GhostUMAP2 παίρνει ~450s (7.5 λεπτά) — εφικτό ανά 6-12 μήνες αλλά ΟΧΙ για daily monitoring.

**Python API (package: `ghostumap2`):**
```python
from ghostumap import GhostUMAP2

gmap = GhostUMAP2(
    n_components=10,
    metric='cosine',
    min_dist=0.0
)

# O = original embeddings (ίδιο με UMAP output)
# G = ghost embeddings
# survived = indices ghosts που επέζησαν
O, G, survived = gmap.fit_transform(X, n_ghosts=16, r=0.1)

# Εύρεση unstable σημείων:
unstable_indices = gmap.get_unstable(d=0.1)

# Visualization:
gmap.visualize(label=y, legend=legend)
```

Το GhostUMAP2 έχει **ίδιο API με το standard UMAP** — drop-in replacement.

#### Α4. Μπορεί να χρησιμοποιηθεί ως Refit Trigger;

**Η σύντομη απάντηση: Ναι, αλλά το paper ΔΕΝ το προτείνει ρητά.**

Το paper εστιάζει στη visualization και interpretation. Δεν ορίζει global instability metric και δεν συζητά monitoring scenarios. Παρόλα αυτά, μπορούμε να **φτιάξουμε έναν trigger** ο ίδιοι:

```python
def global_instability_score(X_1024d, r=0.1, d=0.05, n_ghosts=8) -> float:
    """
    Τρέχει GhostUMAP2 και επιστρέφει το ποσοστό unstable σημείων.
    Refit trigger: αν score > threshold → full re-cluster.
    """
    from ghostumap import GhostUMAP2
    import numpy as np

    gmap = GhostUMAP2(n_components=10, metric='cosine', min_dist=0.0)
    O, G, survived = gmap.fit_transform(X_1024d, n_ghosts=n_ghosts, r=r)

    unstable = gmap.get_unstable(d=d)
    instability_score = len(unstable) / len(X_1024d)  # ποσοστό [0, 1]
    return instability_score

# Refit decision:
score = global_instability_score(embeddings, r=0.1, d=0.05)
if score > 0.15:  # >15% σημείων unstable → trigger refit
    trigger_full_refit()
```

**Τι threshold να χρησιμοποιήσεις;**
- Το paper δεν δίνει threshold — δεν ήταν ο στόχος του
- Με `d=0.05`: αναμένουμε ~5-10% unstable σε healthy corpus
- Threshold `15%` είναι conservative εκκίνηση — tune εμπειρικά μετά τον πρώτο χρόνο
- **Σημαντική παρατήρηση:** Αν το corpus αλλάξει θεμελιακά (π.χ. νέο εξάμηνο + 500 νέα αρχεία), η instability θα ανεβεί φυσικά — αυτό είναι το σήμα που θέλουμε

**Εναλλακτικά (χωρίς GhostUMAP2):** Το Trigger C του ΜΕΡΟΣ 21 (>20% αρχείων στο Review για 2+ εβδομάδες) είναι ήδη ένας proxy για embedding drift — και δεν έχει το 5× overhead.

**Verdict GhostUMAP2 ως trigger:** Χρήσιμο αλλά computationally ακριβό. Για daily monitoring: χρησιμοποίησε το Review rate (Trigger C). Για 6-month health check: GhostUMAP2 με `n_ghosts=4` (ελαφρύτερο) ως επιπλέον validation.

---

### Β. AlignedUMAP — Επανεξέταση

#### Β1. Τι κάνει ακριβώς το AlignedUMAP

Το `AlignedUMAP` (umap-learn) είναι η επίσημη λύση για **alignment μεταξύ διαφορετικών timesteps** του ίδιου corpus. Δέχεται μια λίστα datasets + ένα `relations` dictionary που αντιστοιχίζει indices μεταξύ consecutive snapshots.

```python
from umap import AlignedUMAP

reducer = AlignedUMAP(
    n_neighbors=15,
    n_components=10,
    min_dist=0.0,
    metric='cosine',
    alignment_regularisation=0.01,  # "σκληρότητα" alignment constraint
    alignment_window_size=3          # πόσα προηγούμενα snapshots λαμβάνει υπόψη
)

# relations: list of dicts — κάθε dict αντιστοιχεί index στο t με index στο t+1
# Π.χ. {0: 0, 1: 1, 3: 2} = αρχείο με index 0 στο snapshot t → index 0 στο snapshot t+1
relations = [{old_idx: new_idx for old_idx, new_idx in pairs}, ...]

reducer.fit(list_of_embedding_arrays, relations=relations)
```

**Τεχνική λεπτομέρεια:** Εσωτερικά το AlignedUMAP:
1. Κάνει αρχική ευθυγράμμιση με Procrustes για κάθε ζεύγος consecutive snapshots
2. Προσθέτει regularizer στην UMAP objective: "related points πρέπει να είναι κοντά στα N embeddings"
3. Βελτιστοποιεί ταυτόχρονα όλα τα N embeddings (όχι διαδοχικά)

**Τι κάνει καλύτερα από naive Procrustes:**
- Procrustes = global linear rigid transformation — ολόκληρο το cloud κινείται ομοιόμορφα
- AlignedUMAP = τοπικός regularizer — επιτρέπει διαφορετική alignment σε διαφορετικές περιοχές
- Paper (Application of Aligned-UMAP, PMC 10318357): AlignedUMAP υπερτερεί naive Procrustes σε longitudinal biomedical data

#### Β2. Κρίσιμη Αποκάλυψη: Memory Grows με τον Χρόνο

**Εδώ αλλάζει η αξιολόγησή μας του ΜΕΡΟΣ 21:**

Από GitHub Issue #658 (Best Practices for AlignedUMAP on large datasets) και από την επίσημη documentation:

> **"The current AlignedUMAP model keeps all the previous data, so it will only really work in a batch streaming approach where occasionally a fresh model is trained, dropping some of the historical data before continuing with updates."**

Δηλαδή:
- AlignedUMAP ΔΕΝ είναι streaming — κρατά **ΟΛΑ** τα historical snapshots στη μνήμη
- Μετά από 3 refits (6, 12, 18 μήνες): κρατά 3 × 5000+ embeddings × 10 dims = ~450K floats per snapshot
- Υπολογιστικό κόστος: ~57s για 10 embeddings του pendigits dataset (μικρό) → για 5000 αρχεία + πολλά snapshots: σημαντικό overhead
- Χρήστης στο issue με 5M+ samples ανέφερε ότι "the AlignedUMAP approach is quite time consuming"

**Workaround που προτείνεται:** Periodic fresh model training (παρόμοιο με την "nuclear option" του ΜΕΡΟΣ 21) + `update()` για incremental batches.

#### Β3. Χρειάζεται ΟΛΑ τα historical data;

**ΝΑΙ**, αν θέλεις το full AlignedUMAP behavior. Αν χρησιμοποιείς `update()`:
- Μπορεί να τρέξει σε batches — δεν χρειάζεται *ταυτόχρονα* όλα τα δεδομένα
- Αλλά: τα αποτελέσματα batch-wise differ σημαντικά από full-dataset UMAP (επιβεβαιωμένο στο issue)

#### Β4. Είναι Πραγματικά Overkill για τον Curator;

**Αναθεωρημένη Απάντηση: ΝΑΙ, είναι overkill — και για καλύτερους λόγους από ό,τι νομίζαμε.**

Στο ΜΕΡΟΣ 21 είπαμε "overkill" λόγω πολυπλοκότητας. Τώρα έχουμε επιπλέον:

| Λόγος | Λεπτομέρεια |
|---|---|
| **Memory grows without bound** | Κρατά ΟΛΑ τα historical snapshots — αντίθετο με τον "100% local" minimal-footprint στόχο |
| **Full refit ούτως ή άλλως** | AlignedUMAP βελτιστοποιεί ΟΛΑ τα embeddings ταυτόχρονα — δεν είναι πιο γρήγορο από full refit |
| **Batch precision loss** | Το `update()` σε batches δίνει διαφορετικά (χειρότερα) αποτελέσματα |
| **Refit interval 6-12 μήνες** | Τόσο σπάνια που το alignment continuity δεν αξίζει το overhead |
| **Curator δεν χρειάζεται visual continuity** | Δεν κάνουμε interactive visualization — δεν πρέπει ένα scatter plot να φαίνεται "ίδιο" |

**Τελικό verdict:** Η απόρριψη του ΜΕΡΟΣ 21 ήταν σωστή. AlignedUMAP = επαληθευμένα overkill για τον Curator.

---

### Γ. Anchor Points — Σχεδιασμός για τον Curator

#### Γ1. Υπάρχει αυτό στη Βιβλιογραφία;

**Ναι, υπάρχουν παρόμοιες ιδέες — αλλά η ακριβής μας εφαρμογή είναι novel.**

**Landmark-based dimensionality reduction (γενικό):**
- "Online landmark replacement for out-of-sample dimensionality reduction" (arXiv:2311.12646, 2024): Προτείνει δυναμική αντικατάσταση landmarks για non-stationary data. Χρησιμοποιεί geometric graphs + minimal dominating set. Δεν αφορά alignment μεταξύ refits.
- "Landmark MDS" (Balasubramanian et al.): Κλασική μέθοδος — embed ένα μικρό subset (landmarks) και τοποθέτησε τα υπόλοιπα σχετικά προς αυτά. Δεν αφορά alignment.
- "Scalability and robustness of spectral embedding: landmark diffusion" (arXiv:2001.00801): ROSELAND — spectral embedding μέσω landmarks. Δεν αφορά Procrustes alignment.

**Procrustes + anchor points για embedding alignment:**
- "Manifold Alignment using Procrustes Analysis" (ICML 2008): Χρησιμοποιεί landmark points για alignment δύο low-dimensional embeddings. Κεντρική ιδέα: αν έχεις M "anchor" σημεία με γνωστή αντιστοιχία, εφάρμοσε Procrustes σε αυτά και μεταφέρου τον μετασχηματισμό.
- "When Embedding Models Meet: Procrustes Bounds and Applications" (arXiv:2510.13406, 2025): Δείχνει ότι αν τα pairwise dot products διατηρούνται κατά προσέγγιση, υπάρχει isometry που align-άρει τα δύο embedding sets. Practical recipe: Procrustes post-processing με anchor links.
- Latent space alignment με cluster centroids ως landmarks (ResearchGate, 2022): Χρησιμοποιεί centroids clusters ως landmarks για Procrustes — analogous αλλά όχι same ακριβώς.

**Η δική μας ιδέα (HDBSCAN high-confidence members ως anchors):** Δεν βρέθηκε σε paper. Genuinely novel application — συνδυάζει HDBSCAN `probabilities_` ως κριτήριο anchor selection + File Biography ως stability check + Procrustes για alignment.

#### Γ2. Πώς να Επιλέξεις τα Anchor Points

**Κριτήρια καλού anchor:**

```python
def is_good_anchor(file_id: str, db) -> bool:
    """
    Ελέγχει αν ένα αρχείο είναι κατάλληλο anchor για Procrustes alignment.
    """
    prob = db.get_hdbscan_prob(file_id)       # HDBSCAN probabilities_[i]
    glosh = db.get_glosh_score(file_id)       # HDBSCAN outlier_scores_[i]
    history = db.get_cluster_history(file_id) # File Biography: list of (date, cluster_id)
    cluster_id = db.get_current_cluster(file_id)

    # Κριτήριο 1: Υψηλό soft membership
    if prob < 0.90:
        return False

    # Κριτήριο 2: Χαμηλό outlier score (όχι boundary)
    if glosh > 0.15:
        return False

    # Κριτήριο 3: Stable history — ίδιο cluster σε 2+ consecutive refits
    # (απαιτεί File Biography από Ιδέα Α, ΜΕΡΟΣ 22)
    if len(history) >= 2:
        recent_clusters = [h[1] for h in history[-2:]]
        if len(set(recent_clusters)) > 1:  # αλλαγή cluster → αποκλεισμός
            return False

    return True
```

**Πόσα anchors ανά cluster;**
- Η βιβλιογραφία (ICML 2008, Manifold Alignment) δεν δίνει ακριβή αριθμό — λέει "enough to define the transformation"
- Για Procrustes σε 10-dimensional space: χρειαζόμαστε τουλάχιστον 10+1=11 anchor points για να ορίσουμε μοναδικό ορθογώνιο μετασχηματισμό
- **Πρακτική σύσταση:** 5-10 anchors ανά cluster (50-100 total για 10-15 clusters) — αρκετό headroom πάνω από το θεωρητικό minimum
- Αν ένα cluster έχει < 5 κατάλληλα anchors → skip Procrustes για αυτό, re-init DenStream centroid από scratch

**Σημαντική παρατήρηση:** Η ελάχιστη απαίτηση για stable Procrustes σε d-dimensional space είναι να έχεις M ≥ d+1 anchor points (για να ορίζεται μοναδική λύση). Με d=10 και M=50-100 anchors: εξαιρετικά over-determined system → numerical stability εγγυημένη.

#### Γ3. Ο Πλήρης Αλγόριθμος

```
ANCHOR-POINT PROCRUSTES ALIGNMENT PROCEDURE

Πριν τον refit:
1. Εντόπισε anchor files: is_good_anchor() → A = {a₁, a₂, ..., aₙ}
2. Αποθήκευσε: old_anchors = X_old_10d[anchor_indices]  # (N, 10)

Κατά τον refit:
3. Fit νέο UMAP: reducer_new.fit_transform(all_embeddings_1024d)
4. Αποθήκευσε: new_anchors = X_new_10d[anchor_indices]  # (N, 10)

Post-refit alignment:
5. Υπολόγισε Procrustes rotation:
   from scipy.linalg import orthogonal_procrustes
   Q, scale = orthogonal_procrustes(old_anchors, new_anchors)
   # Q = (10, 10) ορθογώνιος πίνακας rotation

6. Εφάρμοσε Q στα DenStream centroids:
   denstream_centroids_new = denstream_centroids_old @ Q
   # Τα centroids τώρα βρίσκονται στον νέο UMAP space

7. Update DenStream: αντικατέστησε centroids με τα rotated
   (βάρη, radii, decay factors παραμένουν αναλλοίωτα)
```

**Μαθηματική εγκυρότητα:** Ορθογώνιος μετασχηματισμός (rotation + reflection) διατηρεί:
- Αποστάσεις (isometry)
- Εσωτερικές γωνίες
- Το σχήμα των micro-clusters

Άρα: αν εφαρμόσουμε Q σε DenStream centroid, το centroid μετακινείται στη σωστή περιοχή του νέου embedding space. Τα βάρη (weights), τα radii, και τα decay factors δεν αφορούν position → **δεν χρειάζονται μετασχηματισμό**.

**Failure modes:**
1. **Πολύ λίγα anchors (N < 20):** Ο Procrustes λύνει over-determined system — με λίγα anchors η λύση είναι unstable. Αντιμετώπιση: αν N < 20 → skip Procrustes → full DenStream re-init.
2. **Anchors clustered σε μία περιοχή:** Αν τα anchors είναι γεωγραφικά concentrated, ο Procrustes περιγράφει καλά μόνο εκείνη την περιοχή. Αντιμετώπιση: διασφάλισε representation από κάθε cluster.
3. **Νέος cluster χωρίς anchors:** Νέος cluster που δημιουργήθηκε μεταξύ refits → δεν έχει anchor history. Αντιμετώπιση: DenStream re-init από HDBSCAN centroid (ήδη στον νέο space).
4. **Μεγάλη structural change:** Αν το νέο UMAP είναι τόσο διαφορετικό που ο Procrustes δίνει μεγάλο residual → αναγνώρισε ότι η alignment απέτυχε → full re-init. Έλεγχος: `scale` από `orthogonal_procrustes()` μακριά από 1.0 → red flag.

#### Γ4. Integration με DenStream — Είναι Έγκυρο;

**Ναι, με επιφυλάξεις.**

Ο DenStream διατηρεί ανά micro-cluster:
- **center** (centroid): 10-dim vector → **αυτό αλλάζει με Q**
- **weight**: scalar (αθροιστικό decay) → **δεν αλλάζει** (δεν αφορά position)
- **radius**: scalar → **δεν αλλάζει** (isometry διατηρεί αποστάσεις)
- **λ (decay factor)**: scalar → **δεν αλλάζει**

Άρα το Procrustes rotation εφαρμόζεται **μόνο στα centers** — όλα τα άλλα παραμένουν αναλλοίωτα. Αυτό είναι mathematically sound.

**Πρακτικό implementation:**

```python
import numpy as np
from scipy.linalg import orthogonal_procrustes

def align_denstream_to_new_umap(
    denstream,
    old_anchor_coords: np.ndarray,  # (N, 10) — παλιά UMAP coords των anchors
    new_anchor_coords: np.ndarray,  # (N, 10) — νέα UMAP coords των ίδιων αρχείων
    min_anchors: int = 20           # minimum για αξιόπιστο alignment
) -> bool:
    """
    Align DenStream centroids από old UMAP space → new UMAP space.
    Returns True αν alignment επιτυχής, False αν χρειάζεται full re-init.
    """
    if len(old_anchor_coords) < min_anchors:
        return False  # Πολύ λίγα anchors → full re-init

    # Υπολόγισε Procrustes rotation
    Q, scale = orthogonal_procrustes(old_anchor_coords, new_anchor_coords)

    # Sanity check: scale κοντά στο 1.0 σημαίνει καλή alignment
    if abs(scale - 1.0) > 0.3:  # >30% scale difference → structural change
        return False

    # Εφάρμοσε rotation σε όλα τα DenStream micro-cluster centers
    for mc in denstream.micro_clusters:
        old_center = np.array(list(mc.center.values()))  # 10-dim
        new_center = old_center @ Q
        mc.center = {i: float(v) for i, v in enumerate(new_center)}

    return True

# Χρήση:
success = align_denstream_to_new_umap(
    denstream,
    old_anchor_coords=old_10d[anchor_indices],
    new_anchor_coords=new_10d[anchor_indices]
)

if not success:
    # Fallback: full DenStream re-initialization (ΜΕΡΟΣ 21 nuclear option)
    denstream = reinitialize_denstream_from_scratch(new_10d)
```

#### Γ5. Procrustes vs AlignedUMAP — Πότε Αρκεί το Ένα;

| Κριτήριο | Procrustes + Anchors | AlignedUMAP |
|---|---|---|
| **Τύπος μετασχηματισμού** | Global linear (rotation + reflection) | Τοπικός non-linear regularizer |
| **Ακρίβεια** | Καλή αν corpus δεν αλλάζει δραστικά | Καλύτερη αν local structure αλλάζει |
| **Απαίτηση δεδομένων** | Μόνο anchor points (50-100) | ΟΛΑ τα historical snapshots |
| **Υπολογιστικό κόστος** | O(N²) SVD — milliseconds | O(T × N) όπου T = timesteps — λεπτά |
| **Memory** | Ελάχιστη | Αυξάνει με τον χρόνο |
| **Για Curator (5000 files, 6-12μ refit)** | ✅ Αρκετό | ⚠️ Overkill |
| **Για visualization continuity** | ⚠️ Approximate | ✅ Καλύτερο |

**Rule of thumb:** Αν ο corpus αλλάζει < 30% μεταξύ refits (ρεαλιστικό για personal files), ο Procrustes alignment με καλά-επιλεγμένα anchors είναι **αρκετός**. Αν ο corpus αλλάζει δραστικά (π.χ. νέος χρήστης με εντελώς διαφορετικά αρχεία), κανένα alignment δεν βοηθά — full re-init είναι η σωστή απάντηση.

---

### 🏆 Τελική Σύσταση UMAP Refit Strategy

#### Τι αλλάζει από το ΜΕΡΟΣ 21

**Το ΜΕΡΟΣ 21 ήταν ουσιαστικά σωστό.** Η "nuclear option" (full re-cluster από μηδέν) παραμένει η βέλτιστη επιλογή. Αλλά έχουμε πλέον **τρεις σημαντικές προσθήκες:**

**1. GhostUMAP2 ως έξτρα refit trigger (νέο):**
- Τρέχει ανά 6 μήνες ως health check (μαζί με τον refit)
- `global_instability_score() > 0.15` = επιπλέον επιβεβαίωση ότι ο refit ήταν αναγκαίος
- ΟΧΙ για daily monitoring — 5× overhead
- Χρησιμοποίησε `n_ghosts=4` για ελαφρύτερο run

**2. AlignedUMAP — επαληθευμένα overkill (επιβεβαίωση ΜΕΡΟΣ 21):**
- Memory grows without bound: ακατάλληλο για personal desktop app
- Batch precision loss: το `update()` δίνει χειρότερα αποτελέσματα
- Απόφαση ΜΕΡΟΣ 21 **επαληθεύεται** με νέα στοιχεία

**3. Anchor Points + Procrustes για DenStream migration (νέο, υψηλή αξία):**
- Αντί full DenStream re-init, εφάρμοσε Q rotation στα centroids
- Κόστος: ~milliseconds vs ~minutes για full re-init
- Requirement: File Biography (Ιδέα Α, ΜΕΡΟΣ 22) για stable anchor selection
- Fallback: full re-init αν N < 20 anchors ή scale > 1.3

#### Ενημερωμένο Refit Decision Tree

```
REFIT TRIGGERS (ΜΕΡΟΣ 21 + νέα):
├── Trigger A (χρόνος): >12 μήνες από τελευταίο refit
├── Trigger B (αριθμός): >1000 νέα αρχεία μέσω DenStream
├── Trigger C (ποιότητα): >20% αρχείων → Review για 2+ εβδομάδες  [ΜΕΡΟΣ 21]
└── Trigger D (GhostUMAP2): instability_score > 0.15 [ΝΕΟ — 6μ health check]

REFIT PROCEDURE (ενημερωμένο):
1. Snapshot current state (ΜΕΡΟΣ 21 Βήμα 1)
2. Re-embed changed files (ΜΕΡΟΣ 21 Βήμα 2)
3. Fit νέο UMAP + HDBSCAN (ΜΕΡΟΣ 21 Βήματα 3-4)
4. *** ΝΕΟ ***: Προσπάθεια Procrustes alignment για DenStream:
   a. Βρες anchor files (prob > 0.90, glosh < 0.15, stable File Biography)
   b. Αν N_anchors ≥ 20: εφάρμοσε Q rotation στα DenStream centroids
   c. Αν N_anchors < 20: full DenStream re-init (ΜΕΡΟΣ 21 Βήμα 6)
5. Re-name clusters με HERCULES αν structure άλλαξε >20% (ΜΕΡΟΣ 21 Βήμα 5)
6. Compute diff + approval UI (ΜΕΡΟΣ 21 Βήματα 7-9)
```

#### Σύνοψη Αποφάσεων

| Θέμα | Απόφαση | Λόγος |
|---|---|---|
| **Full re-cluster vs. incremental** | Full re-cluster ✅ | Δεν υπάρχει partial_fit · DenStream re-init σχεδόν πάντα αναγκαίο |
| **AlignedUMAP** | Reject ✅ | Memory grows · batch precision loss · overkill για 6-12μ intervals |
| **Procrustes (naive, χωρίς anchors)** | Acceptable workaround ⚠️ | Χρησιμοποιεί ΟΛΑ τα αρχεία — over-determined, αλλά computationally εντάξει |
| **Procrustes + anchor points** | Υιοθετείται ✅ (νέο) | Γρήγορο · mathematically valid · αποτρέπει full DenStream re-init |
| **GhostUMAP2 ως trigger** | Υιοθετείται ✅ (νέο) | Επίσημη μέτρηση instability · χρήση ανά 6μ ως validation |
| **GhostUMAP2 για daily monitoring** | Reject | 5× overhead — πολύ ακριβό για daily use |
| **ParametricUMAP** | Reject ✅ | Extra dependencies · drift εξακολουθεί να υπάρχει |

---

**Sources Μέρος 23:**
- GhostUMAP2 arXiv:2507.17174 — https://arxiv.org/abs/2507.17174
- GhostUMAP2 HTML full paper — https://arxiv.org/html/2507.17174
- ghostumap2 PyPI — https://pypi.org/project/ghostumap2/
- GhostUMAP GitHub (jjmmwon/ghostumap) — https://github.com/jjmmwon/ghostumap
- GhostUMAP IEEE Xplore (original v1) — https://ieeexplore.ieee.org/document/10771096/
- Stability for UMAP arXiv:2011.13430 — https://arxiv.org/abs/2011.13430
- AlignedUMAP basic usage docs — https://umap-learn.readthedocs.io/en/latest/aligned_umap_basic_usage.html
- AlignedUMAP time-varying demo — https://umap-learn.readthedocs.io/en/latest/aligned_umap_politics_demo.html
- AlignedUMAP best practices Issue #658 — https://github.com/lmcinnes/umap/issues/658
- Application of Aligned-UMAP to longitudinal biomedical studies (PMC 10318357) — https://pmc.ncbi.nlm.nih.gov/articles/PMC10318357/
- NIH-CARD AlignedUMAP-BiomedicalData GitHub — https://github.com/NIH-CARD/AlignedUMAP-BiomedicalData
- Online landmark replacement arXiv:2311.12646 — https://arxiv.org/abs/2311.12646
- Manifold Alignment using Procrustes Analysis ICML 2008 — https://icml.cc/Conferences/2008/papers/229.pdf
- When Embedding Models Meet: Procrustes Bounds arXiv:2510.13406 — https://arxiv.org/html/2510.13406v1
- Procrustes Alignment emergentmind overview — https://www.emergentmind.com/topics/orthogonal-procrustes-alignment
- scipy.linalg.orthogonal_procrustes docs — https://docs.scipy.org/doc/scipy/reference/generated/scipy.linalg.orthogonal_procrustes.html
- DenDrift: Drift-Aware DenStream extension arXiv:2110.01221 — https://arxiv.org/pdf/2110.01221
- Embedding drift monitoring (APXML) — https://apxml.com/courses/monitoring-managing-ml-models-production/chapter-2-advanced-drift-detection/embedding-drift-monitoring
- kMDItemLastUsedDate + kMDItemUsedDates forensic context — https://vulntech.com/tutorial/tutorial/learn-digital-forensics/macos-spotlight-fsevents-knowledgec-analysis/

---
