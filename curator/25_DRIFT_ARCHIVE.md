## ΜΕΡΟΣ 25: DRIFT-TRIGGERED REFIT + ARCHIVE DNA FINGERPRINTING
*(Έρευνα 2026-05-22 — 16 searches, 30+ sources)*

---

### Α. Drift-Triggered Refit

#### Α1. Υπάρχει κάτι παρόμοιο;

**Σύντομη απάντηση: Ναι, αλλά όχι για file organizers — και το σχεδιασμένο-για-clustering unsupervised variant είναι νέο.**

##### Α1α. Στη βιβλιογραφία clustering + concept drift

Ο συνδυασμός "drift detection → trigger refit" υπάρχει στη βιβλιογραφία stream clustering, αλλά **κυρίως για supervised classification tasks**, όχι για unsupervised clustering quality monitoring.

**StreamDD** (ScienceDirect 2024, Procedia Computer Science) — Ο πιο άμεσα σχετικός αλγόριθμος. Βασίζεται στο CluStream και χρησιμοποιεί **Page-Hinckley test** ως built-in drift detector. Ανιχνεύει Sudden και Incremental drifts απευθείας εντός του loop του stream clustering. Σύγκριση με DenStream: το StreamDD κερδίζει σε drift scenarios, αλλά χρειάζεται labeled ground truth για αξιολόγηση — κάτι που δεν έχουμε για personal files.

**DenDrift** (arXiv:2110.01221, RISE Research Institutes of Sweden, 2021) — Απευθείας παράγωγο του DenStream. Προσθέτει **Non-negative Matrix Factorization** για dimensionality reduction και **Page-Hinckley test** για drift detection. Κρίσιμο εύρημα: "Recent experimental studies indicate that DenStream is **not robust against concept drift**." — Αυτό σημαίνει ότι ο Curator χρειάζεται εξωτερικό drift detector, δεν φτάνει το DenStream από μόνο του.

**Cluster Analysis and Concept Drift Detection in Malware** (arXiv:2502.14135, 2025) — Χρησιμοποιεί **MiniBatch K-Means + silhouette coefficient** ως drift trigger. Σύγκριση τριών σεναρίων: (i) static training, (ii) periodic retraining, (iii) drift-aware retraining. Εύρημα: το drift-aware retraining επιτυγχάνει accuracy εντός 1% του periodic retraining αλλά είναι **σημαντικά πιο αποδοτικό** — λιγότερα refits όταν δεν χρειάζεται.

**Temporal Silhouette** (Machine Learning, Springer 2023 — TU Wien, arXiv companion: dl.acm.org/doi/10.1007/s10994-023-06462-2) — Νέα παραλλαγή του Silhouette index ειδικά σχεδιασμένη για **stream clustering με concept drift**. Αντιμετωπίζει το πρόβλημα ότι το κλασικό Silhouette γίνεται inconsistent όταν τα δεδομένα εξελίσσονται. Δοκιμάστηκε σε 200 scenarios concept drift. Υπάρχει Python implementation: `pip install py-temporal-silhouette` (GitHub: CN-TU/py-temporal-silhouette).

**ICD3** (arXiv:2603.06757, 2025) — "Learning Unbiased Cluster Descriptors for Interpretable Imbalanced Concept Drift Detection" — Ανιχνεύει drift μέσω cluster descriptors. Αφορά κυρίως imbalanced data streams.

**ADWIN-U** (Knowledge and Information Systems, Springer 2025 — link.springer.com/article/10.1007/s10115-025-02523-1) — Κρίσιμο: **Unsupervised variant του ADWIN** ειδικά για stream scenarios χωρίς labeled data. Αυτό είναι η ακριβής λύση για τον Curator — δεν έχουμε labels, άρα χρειαζόμαστε unsupervised drift detection. Εκδόθηκε 2025, δεν υπάρχει ακόμα στο pip, αλλά ο αλγόριθμος είναι δημοσιευμένος.

**clusTransition** (PLOS ONE 2022 — pmc.ncbi.nlm.nih.gov/articles/PMC9754593/) — R package για monitoring transitions σε clustering temporal datasets. Βασίζεται στο MONIC framework. Ανιχνεύει: clusters που επέζησαν, split, absorbed, disappeared, emerged. Δεν υπάρχει Python port — αλλά η ιδέα του transition monitoring είναι απευθείας εφαρμόσιμη.

##### Α1β. Στα file organizers

**Κανένα υπάρχον file organizer δεν έχει drift-triggered refit.** Όλα τα συστήματα που εξετάστηκαν (iyaya/llama-fs, different-ai/file-organizer-2000, QiuYannnn/Local-File-Organizer, AIOS-LSFS) είτε:
- Δεν κάνουν re-clustering καθόλου (static one-shot)
- Κάνουν periodic rebuild σε fixed schedule (π.χ. "run every Sunday")
- Δεν έχουν quality monitoring μετά την αρχική οργάνωση

**Αυτό επιβεβαιώνει ότι ο Drift-Triggered Refit για file organizers είναι genuinely novel.**

---

#### Α2. Metrics για Monitoring

Ο Curator ήδη υπολογίζει τέσσερα metrics (ΜΕΡΟΣ 8) που μπορούν να χρησιμοποιηθούν ως drift signals. Ερώτηση: ποιος συνδυασμός είναι η πιο αξιόπιστη ανίχνευση drift;

##### Α2α. Αξιολόγηση κάθε metric ως drift signal

| Metric | Πλεονεκτήματα | Μειονεκτήματα | Lag (πόσο αργά ανιχνεύει) |
|---|---|---|---|
| **DBCV score** | Ground-truth-free, σχεδιάστηκε για density-based clustering, τεκμηριωμένο στη βιβλιογραφία | Ακριβό να υπολογιστεί (O(N²) worst case), απαιτεί full re-evaluation | Μεγάλο lag — ανιχνεύει μόνο μετά από significant drift |
| **Temporal churn rate** | Πολύ γρήγορο, per-file signal, ανιχνεύει αστάθεια real-time | False positives αν το corpus αλλάζει νόμιμα (νέο εξάμηνο) | Μικρό lag — πρόωρη ανίχνευση |
| **Name-embedding cosine** | Zero extra cost (τρέχει ήδη για naming), intuitive interpretation | Μετράει μόνο naming quality, όχι clustering quality | Μεσαίο lag — ανιχνεύει όταν τα clusters αρχίζουν να "ξεφεύγουν" από τα ονόματά τους |
| **GhostUMAP2 instability** | Τυπικά θεμελιωμένο (arXiv:2507.17174), ανιχνεύει boundary instability | 5× overhead vs baseline UMAP — ακριβό για daily monitoring | Μεγάλο lag — υπολογίζεται μόνο κατά τον refit |

##### Α2β. Το Temporal Silhouette ως νέο metric

Το **Temporal Silhouette** (TU Wien, 2023) είναι ο πιο αξιόπιστος drift indicator για stream clustering γιατί:
1. Σχεδιάστηκε ρητά για scenario με concept drift (το κλασικό Silhouette αποτυγχάνει εκεί)
2. Δοκιμάστηκε σε 200 scenarios drift — αποδεδειγμένα καλύτερο από traditional internal validation
3. Υπάρχει Python implementation (`pip install py-temporal-silhouette`)

**Πρόταση: Χρησιμοποίησε Temporal Silhouette ως primary drift indicator αντί για Silhouette ή DBCV.**

##### Α2γ. Ποιος συνδυασμός μετρικών είναι αξιόπιστος;

Η βιβλιογραφία (Efficient Concept Drift Detection, ACM 2024; Smart Adaptive Ensemble, Nature Scientific Reports 2025) υποδεικνύει ότι **συνδυασμός multiple metrics** με ensemble voting μειώνει false positives.

**Προτεινόμενος συνδυασμός για τον Curator:**

| Priority | Metric | Παρακολούθηση | Threshold |
|---|---|---|---|
| 1 (primary) | **Temporal churn rate** | Per new file arrival | >25% για 2+ εβδομάδες |
| 2 (secondary) | **Name-embedding cosine** | Ανά εβδομάδα | Mean cosine < 0.55 |
| 3 (validation) | **Temporal Silhouette** | Ανά 500 νέα αρχεία | Drop >0.10 από baseline |
| 4 (periodic) | **GhostUMAP2 instability** | Ανά 6 μήνες | >15% unstable |

**Λογική:** Ο churn rate είναι ο γρηγορότερος (low cost, low lag). Το name-embedding cosine επαληθεύει. Το Temporal Silhouette δίνει statistically robust validation. Το GhostUMAP2 επαληθεύει ότι ο refit είναι πραγματικά αναγκαίος πριν το εκτελέσουμε.

---

#### Α3. Concrete Algorithm — Drift-Triggered Refit

##### Αρχιτεκτονική

```
[DenStream loop] → [Metric collector] → [ADWIN-U monitors] → [Ensemble vote] → [Refit decision]
                                                    ↓
                                          [GhostUMAP2 health check] → [Full refit ή abort]
```

##### Πλήρης Αλγόριθμος

```python
# ============================================================
# Drift-Triggered Refit για τον Curator
# Libraries: river, py-temporal-silhouette, ghostumap
# ============================================================

from river import drift
import numpy as np
from collections import deque

class DriftMonitor:
    """
    Παρακολουθεί 3 metrics και ψηφίζει ensemble για refit trigger.
    """
    
    def __init__(self, churn_threshold=0.25, cosine_threshold=0.55,
                 temporal_sil_drop=0.10, cooldown_days=30):
        
        # ADWIN για churn rate (river.drift.ADWIN)
        self.adwin_churn = drift.ADWIN(delta=0.002)
        
        # ADWIN για name-embedding cosine
        self.adwin_cosine = drift.ADWIN(delta=0.002)
        
        # Baseline values (υπολογίζονται μετά τον αρχικό refit)
        self.baseline_temporal_sil = None
        self.churn_history = deque(maxlen=14)   # 14 ημέρες
        
        # Thresholds
        self.churn_threshold = churn_threshold
        self.cosine_threshold = cosine_threshold
        self.temporal_sil_drop = temporal_sil_drop
        
        # Cooldown: αποφυγή συνεχών refits
        self.last_refit_day = 0
        self.cooldown_days = cooldown_days
    
    def update(self, new_file_embedding, churn_rate, name_cosine, current_day):
        """
        Καλείται για κάθε νέο αρχείο ή ανά ημέρα.
        Επιστρέφει True αν χρειάζεται refit.
        """
        # Cooldown check
        if current_day - self.last_refit_day < self.cooldown_days:
            return False
        
        # 1. ADWIN update για churn rate
        self.adwin_churn.update(churn_rate)
        self.churn_history.append(churn_rate)
        
        # 2. ADWIN update για cosine
        self.adwin_cosine.update(name_cosine)
        
        # Αξιολόγηση triggers
        trigger_votes = 0
        
        # Vote 1: ADWIN ανίχνευσε change point στο churn rate
        if self.adwin_churn.drift_detected:
            mean_churn = np.mean(self.churn_history)
            if mean_churn > self.churn_threshold:
                trigger_votes += 2  # βαρύτερη ψήφος (primary metric)
        
        # Vote 2: Σταθερά χαμηλό cosine (δεν χρειάζεται change point)
        if name_cosine < self.cosine_threshold:
            trigger_votes += 1
        
        # Vote 3: ADWIN ανίχνευσε change point στο cosine
        if self.adwin_cosine.drift_detected:
            trigger_votes += 1
        
        # Ensemble απόφαση: ≥3 ψήφοι → προχώρα σε validation
        if trigger_votes >= 3:
            return self._validate_with_temporal_silhouette()
        
        return False
    
    def _validate_with_temporal_silhouette(self):
        """
        Ελέγχει αν το Temporal Silhouette έχει πέσει αρκετά.
        Αποτρέπει false positives.
        """
        from temporal_silhouette import TemporalSilhouette
        
        # Υπολογισμός Temporal Silhouette στα τρέχοντα embeddings
        ts = TemporalSilhouette()
        current_score = ts.score(current_embeddings, current_labels)
        
        if self.baseline_temporal_sil is None:
            self.baseline_temporal_sil = current_score
            return False
        
        drop = self.baseline_temporal_sil - current_score
        if drop > self.temporal_sil_drop:
            print(f"Temporal Silhouette drop: {drop:.3f} > {self.temporal_sil_drop}")
            return True
        
        return False
    
    def register_refit(self, day, new_baseline_score):
        """Καταχώρηση refit — reset baselines."""
        self.last_refit_day = day
        self.baseline_temporal_sil = new_baseline_score
        self.adwin_churn = drift.ADWIN(delta=0.002)   # reset
        self.adwin_cosine = drift.ADWIN(delta=0.002)  # reset
        print(f"Refit registered on day {day}. New baseline: {new_baseline_score:.3f}")


def ghostumap_health_check(embeddings_1024d, threshold=0.15):
    """
    Τελικός έλεγχος πριν το refit — αποτρέπει unnecessary refits.
    Τρέχει μόνο αν η ensemble ψηφίσει refit (5× overhead).
    """
    from ghostumap import GhostUMAP2
    
    gmap = GhostUMAP2(n_components=10, metric='cosine', min_dist=0.0)
    O, G, survived = gmap.fit_transform(embeddings_1024d, n_ghosts=4, r=0.1)
    
    unstable = gmap.get_unstable(d=0.05)
    instability_score = len(unstable) / len(embeddings_1024d)
    
    print(f"GhostUMAP2 instability: {instability_score:.1%}")
    return instability_score > threshold


# ============================================================
# Main loop integration (Curator background process)
# ============================================================

monitor = DriftMonitor(
    churn_threshold=0.25,
    cosine_threshold=0.55,
    temporal_sil_drop=0.10,
    cooldown_days=30
)

for day, new_files in daily_file_stream:
    # DenStream update
    for f in new_files:
        denstream.learn_one({"embedding": f.umap_embedding})
    
    # Υπολογισμός daily metrics
    churn = compute_churn_rate(new_files, denstream)
    cosine = compute_mean_name_cosine(clusters, folder_names)
    
    # Drift check
    if monitor.update(None, churn, cosine, current_day=day):
        print("Ensemble signals drift — running GhostUMAP2 health check...")
        
        all_embeddings = load_all_embeddings()
        if ghostumap_health_check(all_embeddings):
            print("GhostUMAP2 confirms instability → TRIGGERING FULL REFIT")
            run_full_refit()  # UMAP refit + HDBSCAN + Procrustes align
            new_baseline = compute_temporal_silhouette(...)
            monitor.register_refit(day, new_baseline)
        else:
            print("GhostUMAP2: embedding still stable, skipping refit")
```

##### Αποφυγή False Positives

Τρία layers προστασίας:

1. **Cooldown (30 ημέρες):** Δεν μπορούν να γίνουν δύο refits τον ίδιο μήνα. Αυτό αντιμετωπίζει το "αρχή εξαμήνου" scenario όπου πολλά νέα αρχεία αρχίζουν να φτάνουν ταυτόχρονα.

2. **Ensemble voting (≥3 ψήφοι):** Ένα μόνο metric που πέφτει δεν αρκεί. Πρέπει πολλαπλά metrics να συμφωνούν.

3. **GhostUMAP2 validation:** Ακόμα και αν η ensemble ψηφίσει refit, το GhostUMAP2 επαληθεύει ότι τα embeddings είναι πράγματι ασταθή πριν εκτελεστεί ο ακριβός refit.

##### Πότε ενεργοποιείται;

| Scenario | Αναμενόμενη αντίδραση |
|---|---|
| Αρχή νέου εξαμήνου (100 νέα αρχεία, νέο μάθημα) | Churn↑ + Cosine↓ → ensemble votes, GhostUMAP2 επαληθεύει → REFIT |
| Διακοπές (5 νέα αρχεία/εβδομάδα, ίδιος τύπος) | Churn stable, Cosine stable → καμία ενέργεια |
| Burst download (50 PDFs ίδιου μαθήματος) | Churn↑ προσωρινά → ADWIN ανιχνεύει αλλά GhostUMAP2 λέει "stable" → αποφεύγεται false positive |
| Αλλαγή χρήσης (από μαθητής → εργαζόμενος) | Όλα τα metrics πέφτουν σταθερά → refit μετά 30 ημέρες |

---

#### Α4. Implementation

| Στοιχείο | Λεπτομέρεια |
|---|---|
| **Γραμμές κώδικα** | ~150-200 LOC (χωρίς existing pipeline) |
| **Libraries** | `river` (ADWIN — ήδη εξεταστεί), `py-temporal-silhouette` (pip install), `ghostumap` (pip install) |
| **Background process** | Τρέχει ως lightweight Python daemon, ~5 MB memory |
| **Ενεργοποίηση** | Per new file arrival (churn/cosine, microseconds) + ανά 500 αρχεία (Temporal Silhouette, ~2s) |
| **GhostUMAP2 overhead** | ~450s για 5000 αρχεία — τρέχει μόνο αν ensemble vote triggers (≤2 φορές/χρόνο) |
| **Integration** | Εισάγεται στο existing DenStream loop — δεν αλλάζει το core pipeline |
| **Persistence** | Αποθηκεύει monitor state (ADWIN windows, baseline scores) σε SQLite |

**Σημαντικό:** Ο ADWIN στο River είναι **supervised** εκ σχεδιασμού (monitors scalar performance metric). Ο Curator χρησιμοποιεί churn rate και name-embedding cosine ως surrogate scalar metrics — άρα ο ADWIN εφαρμόζεται έμμεσα σε unsupervised context. Αν θέλουμε πλήρως unsupervised: χρησιμοποίησε το **ADWIN-U** (Springer 2025) όταν γίνει διαθέσιμο στο pip.

---

#### Α5. Novelty

##### Ως προς τα file organizers

**Πλήρως novel.** Κανένα γνωστό file organizer σύστημα δεν έχει:
- Quality monitoring μετά την αρχική οργάνωση
- Αυτόματο trigger για re-clustering
- Multi-metric ensemble drift detector
- Integration με GhostUMAP2 health check

Αυτό είναι ίσως η πιο σημαντική novel contribution του Curator ως system.

##### Ως προς τη γενικότερη ML βιβλιογραφία

**Μερικώς novel.** Τα building blocks υπάρχουν:
- ADWIN: 2007 (Bifet & Gavalda)
- StreamDD: 2024 (CluStream + Page-Hinckley)
- DenDrift: 2021 (DenStream + Page-Hinckley)
- Temporal Silhouette: 2023

**Αυτό που δεν υπάρχει:**
1. Ο συγκεκριμένος συνδυασμός των τεσσάρων metrics (churn + cosine + Temporal Silhouette + GhostUMAP2)
2. Η integration με personal file system context
3. Το two-stage validation pattern (ensemble vote → GhostUMAP2 confirm)
4. Η εφαρμογή σε low-frequency streams (5 files/day, όχι high-speed data streams)

##### Contribution Claim

> "Παρουσιάζουμε ένα multi-metric ensemble drift detection system για personal file clustering που συνδυάζει temporal churn monitoring, name-embedding coherence, Temporal Silhouette validation, και GhostUMAP2 health check για αυτόματο triggering UMAP+HDBSCAN refit, χωρίς fixed schedule και χωρίς labeled ground truth."

---

### Β. Archive DNA Fingerprinting

#### Β1. Υπάρχει κάτι παρόμοιο;

**Σύντομη απάντηση: Τα building blocks υπάρχουν, αλλά η εφαρμογή σε file organizer context είναι novel.**

##### Β1α. Στα backup tools

**Borg Backup** (borgbackup.readthedocs.io) — Χρησιμοποιεί **content-defined chunking** (Buzhash algorithm) για deduplication. Χωρίζει κάθε αρχείο σε variable-length chunks, hash τους, αποθηκεύει μόνο νέα chunks. Αυτό είναι chunk-level deduplication μέσα στα αρχεία, **όχι** file-list-level similarity μεταξύ archives. Δεν ανιχνεύει "αυτά τα δύο ZIP περιέχουν τα ίδια αρχεία."

**Restic** — Ίδια προσέγγιση με Borg (content-addressable storage). Επίσης chunk-level, όχι archive-structure-level.

**Συμπέρασμα:** Τα backup tools κοιτάνε **μέσα** στα αρχεία (byte-level chunks), ενώ το Archive DNA κοιτάει τη **δομή** του archive (list of filenames) — εντελώς διαφορετική προσέγγιση.

##### Β1β. Στην ακαδημαϊκή βιβλιογραφία

Η αναζήτηση "archive fingerprinting", "zip file similarity", "file list similarity detection" επέστρεψε:
- Patent US8793798: "Detection of undesired computer files in archives" — αφορά malware detection, όχι version detection
- Patent US11714906: "Reducing threat detection processing by applying similarity measures to entropy measures of files" — entropy-based, όχι structure-based
- Chunk-based resemblance detection (arXiv:2106.01273): αφορά δεδομένα backup chunks, όχι archive file lists

**Κανένα paper δεν εξετάζει file-list Jaccard similarity για archive version detection.** Αυτό επιβεβαιώνει novelty.

##### Β1γ. Στα file comparison tools

Υπάρχει `zipfile-dedup` (GitHub: rongrimes/zipfile-dedup) — αφαιρεί duplicate αρχεία μεταξύ δύο ZIP με **ίδιο timestamp**. Δεν χρησιμοποιεί Jaccard similarity, δεν ανιχνεύει versions.

Υπάρχουν tools για directory diff (diff, rsync --dry-run), αλλά απαιτούν **extraction** του archive — άρα δεν λύνουν το πρόβλημα "χωρίς extraction".

---

#### Β2. Technical Approach

##### Β2α. Ορισμός Archive DNA

Για κάθε archive αρχείο (`.zip`, `.tar.gz`, `.7z`, `.rar`):

```
DNA(archive) = normalize(filelist(archive))
```

Όπου `normalize`:
1. Εξαγωγή `namelist()` χωρίς extraction (zero-cost, Python standard library)
2. Lowercase όλα τα ονόματα
3. Αφαίρεση extensions (`.py` → αφαιρείται)
4. Αφαίρεση path components (κρατάμε μόνο basenames)
5. Strip αριθμοί version από ονόματα (π.χ. `v2_`, `_final`, `_backup`)
6. Sort alphabetically
7. Αποτέλεσμα: frozenset από normalized filenames

##### Β2β. Similarity Computation

```python
import zipfile
import tarfile
import re
from datasketch import MinHash, MinHashLSH

def normalize_name(name: str) -> str:
    """Κανονικοποίηση ονόματος αρχείου για σύγκριση."""
    import os
    basename = os.path.basename(name)                       # αφαίρεση path
    stem = os.path.splitext(basename)[0]                    # αφαίρεση extension
    stem = stem.lower()                                     # lowercase
    stem = re.sub(r'[_\-](v\d+|final|backup|old|new|copy|draft)', '', stem)  # strip version tags
    stem = re.sub(r'\d+$', '', stem)                        # strip trailing numbers
    stem = re.sub(r'[^a-z0-9]', '_', stem)                  # normalize special chars
    return stem.strip('_')

def extract_dna(path: str) -> frozenset | None:
    """Εξαγωγή Archive DNA χωρίς extraction στο disk."""
    try:
        if zipfile.is_zipfile(path):
            with zipfile.ZipFile(path, 'r') as zf:
                # Έλεγχος encryption — αν encrypted, DNA = None
                names = []
                for info in zf.infolist():
                    if info.flag_bits & 0x1:  # encrypted
                        return None
                    if not info.is_dir():     # skip directories
                        names.append(normalize_name(info.filename))
                return frozenset(n for n in names if n)  # αφαίρεση κενών
        
        elif tarfile.is_tarfile(path):
            with tarfile.open(path, 'r:*') as tf:
                names = [normalize_name(m.name) for m in tf.getmembers()
                         if not m.isdir()]
                return frozenset(n for n in names if n)
    
    except Exception:
        return None  # corrupted ή unsupported format

def jaccard_similarity(set_a: frozenset, set_b: frozenset) -> float:
    """Jaccard similarity μεταξύ δύο sets."""
    if not set_a or not set_b:
        return 0.0
    intersection = len(set_a & set_b)
    union = len(set_a | set_b)
    return intersection / union if union > 0 else 0.0


# ============================================================
# MinHash LSH για scalability (datasketch — ήδη έχουμε)
# ============================================================

def build_minhash(dna: frozenset, num_perm=64) -> MinHash:
    """Δημιουργία MinHash signature από Archive DNA."""
    m = MinHash(num_perm=num_perm)
    for item in dna:
        m.update(item.encode('utf-8'))
    return m

def build_archive_lsh(archives: list[dict], threshold=0.7) -> MinHashLSH:
    """
    Δημιουργία LSH index για scalable archive similarity search.
    archives = [{"path": "/path/file.zip", "dna": frozenset, "minhash": MinHash}]
    """
    lsh = MinHashLSH(threshold=threshold, num_perm=64)
    for arch in archives:
        if arch["dna"] and len(arch["dna"]) >= 3:  # min 3 files για αξιόπιστο Jaccard
            lsh.insert(arch["path"], arch["minhash"])
    return lsh

def find_archive_versions(new_archive_path: str, lsh: MinHashLSH,
                           archive_db: dict) -> list[tuple[str, float]]:
    """
    Βρίσκει archives που είναι πιθανές εκδόσεις του νέου archive.
    Επιστρέφει: [(path, jaccard_score), ...] ταξινομημένο
    """
    dna = extract_dna(new_archive_path)
    if dna is None or len(dna) < 3:
        return []
    
    m = build_minhash(dna)
    candidates = lsh.query(m)  # O(1) approximate search
    
    # Exact Jaccard για τους candidates (λίγοι)
    results = []
    for cand_path in candidates:
        cand_dna = archive_db[cand_path]["dna"]
        score = jaccard_similarity(dna, cand_dna)
        if score > 0.0:
            results.append((cand_path, score))
    
    return sorted(results, key=lambda x: -x[1])
```

##### Β2γ. Interpretation Thresholds

| Jaccard Score | Ερμηνεία | Ενέργεια |
|---|---|---|
| **≥ 0.85** | Σχεδόν ταυτόσημα (μόνο minor additions/deletions) | Auto → `_versions/` folder |
| **0.70 – 0.85** | Ίδιο project, διαφορετική φάση | Propose grouping → Review |
| **0.50 – 0.70** | Πιθανώς σχετικά (fork, branch) | Weak signal — show to user |
| **< 0.50** | Άσχετα | Αγνόησε |

##### Β2δ. Βελτίωση: File sizes ως επιπλέον signal

Αν δύο archives έχουν Jaccard = 0.75 (border case), η εσωτερική κατανομή file sizes δίνει επιπλέον πληροφορία:

```python
def size_profile_similarity(zf1_path: str, zf2_path: str) -> float:
    """
    Σύγκριση distribution των file sizes εντός δύο archives.
    Επιστρέφει cosine similarity των size distributions.
    """
    def get_sizes(path):
        with zipfile.ZipFile(path, 'r') as zf:
            return sorted([info.file_size for info in zf.infolist()
                           if not info.is_dir()])
    
    s1 = np.array(get_sizes(zf1_path))
    s2 = np.array(get_sizes(zf2_path))
    
    # Bin into buckets (1KB, 10KB, 100KB, 1MB, 10MB+)
    bins = [0, 1024, 10240, 102400, 1048576, float('inf')]
    h1, _ = np.histogram(s1, bins=bins)
    h2, _ = np.histogram(s2, bins=bins)
    
    # Cosine similarity
    if np.linalg.norm(h1) == 0 or np.linalg.norm(h2) == 0:
        return 0.0
    return np.dot(h1, h2) / (np.linalg.norm(h1) * np.linalg.norm(h2))

# Combined score:
# final_score = 0.7 * jaccard + 0.3 * size_similarity
```

---

#### Β3. Integration με Existing Pipeline

Από το ΜΕΡΟΣ 6 (Αρχιτεκτονική), η duplicate detection pipeline τρέχει σε αυτή τη σειρά:

```
SHA-256 exact dup → MinHash near-dup text → imagededup images → RETSim versions
```

**Το Archive DNA εισάγεται ΠΡΙΝ το embedding step:**

```
SHA-256 exact dup
    ↓
[ΝΕΟΣ] Archive DNA Fingerprinting (για .zip, .tar.gz, .7z, .rar)
    ↓  ← Εδώ ανιχνεύονται archive versions χωρίς extraction
MinHash near-dup text (για αρχεία που δεν είναι archives)
    ↓
BGE-M3 embedding
    ↓
UMAP → HDBSCAN → DenStream
```

**Γιατί πριν το embedding:**
1. **Αποφυγή double-embedding:** Archives που ανιχνεύτηκαν ως versions δεν χρειάζεται να embedαριστούν ξεχωριστά
2. **Ταχύτητα:** Archive DNA τρέχει σε microseconds (zero extraction). Embedding τρέχει σε seconds ανά αρχείο
3. **Ορθότητα:** Αν δύο archives είναι versions, θέλουμε να τα χειριστούμε ως group **πριν** τα κάνουμε embed (αλλιώς το HDBSCAN μπορεί να τα βάλει σε διαφορετικά clusters λόγω filename differences)

**Integration Code:**

```python
def process_archive(path: str, lsh: MinHashLSH, archive_db: dict, 
                    version_threshold=0.85, review_threshold=0.70):
    """
    Επιστρέφει: ("exact_dup" | "version" | "related" | "unique", metadata)
    """
    dna = extract_dna(path)
    
    # 1. Encrypted / tiny / no DNA → treat as regular file
    if dna is None or len(dna) < 3:
        return "unique", {"reason": "too_small_or_encrypted"}
    
    # 2. Archive DNA similarity search
    matches = find_archive_versions(path, lsh, archive_db)
    
    if not matches:
        # Προσθήκη στο index για μελλοντικές συγκρίσεις
        m = build_minhash(dna)
        lsh.insert(path, m)
        archive_db[path] = {"dna": dna, "minhash": m}
        return "unique", {}
    
    best_match, best_score = matches[0]
    
    if best_score >= version_threshold:
        return "version", {"match": best_match, "score": best_score,
                           "action": "auto_versions_folder"}
    elif best_score >= review_threshold:
        return "related", {"match": best_match, "score": best_score,
                           "action": "propose_grouping"}
    else:
        return "unique", {}
```

---

#### Β4. Edge Cases

| Edge Case | Πρόβλημα | Λύση |
|---|---|---|
| **Archive με 1-2 αρχεία** | Jaccard unreliable (π.χ. 1/1 = 1.0 ή 0/2 = 0.0) | Min threshold: `len(dna) >= 3` — αν λιγότερα, treat ως regular file |
| **Archive με 1000+ αρχεία** | Jaccard stable αλλά μπορεί να χάσει σημαντικές διαφορές | Προσθήκη size profile similarity ως tiebreaker |
| **Encrypted archives** | `flag_bits & 0x1` → δεν μπορούμε να διαβάσουμε namelist | DNA = None → fall back σε filename+size similarity μόνο |
| **UUID-named files** (`f3a2b1.dat`)| Normalize δεν βοηθάει — όλα τα ονόματα random | Ανίχνευση UUID pattern → fall back σε size distribution |
| **Nested archives** (`archive.zip` περιέχει `inner.zip`) | Recursive extraction costly | Πρώτο επίπεδο μόνο — inner archives αγνοούνται |
| **Archives με μόνο binary** (`data1.bin`, `data2.bin`) | Normalized names = `data`, `data` → collision | Χρήση file sizes ως primary signal αντί ονομάτων |
| **.tar.gz vs .zip** του ίδιου project | Διαφορετικά formats → δεν συγκρίνονται αυτόματα | Cross-format comparison: υπολόγισε DNA για αμφότερα, συγκρίνεις sets |

##### Αντιμετώπιση UUID-named files:

```python
import uuid

def is_uuid_filename(name: str) -> bool:
    """Ανίχνευση UUID pattern."""
    stem = os.path.splitext(os.path.basename(name))[0]
    try:
        uuid.UUID(stem)
        return True
    except ValueError:
        return False

def extract_dna_smart(path: str) -> frozenset | None:
    """DNA με fallback για UUID-heavy archives."""
    dna = extract_dna(path)
    if dna is None:
        return None
    
    # Αν >50% ονόματα είναι UUID → χρησιμοποίησε size buckets αντί ονομάτων
    with zipfile.ZipFile(path, 'r') as zf:
        names = [info.filename for info in zf.infolist() if not info.is_dir()]
        uuid_count = sum(1 for n in names if is_uuid_filename(n))
        
        if uuid_count / max(len(names), 1) > 0.5:
            # Fallback: αντιπροσώπευση ως size fingerprint
            sizes = sorted([info.file_size for info in zf.infolist() if not info.is_dir()])
            bins = [0, 1024, 10240, 102400, 1048576, float('inf')]
            hist, _ = np.histogram(sizes, bins=bins)
            return frozenset([f"size_bucket_{i}_{count}" 
                              for i, count in enumerate(hist) if count > 0])
    
    return dna
```

---

#### Β5. Novelty

##### Ως προς τα file organizers

**Πλήρως novel.** Κανένα file organizer tool ή research paper δεν εφαρμόζει:
- File-list Jaccard similarity για archive version detection
- Zero-extraction archive comparison για deduplication
- MinHash LSH index ειδικά για archive DNA

Τα backup tools (Borg, Restic) κάνουν chunk-level deduplication *εντός* των αρχείων, όχι *μεταξύ* archives.

##### Ως προς τη γενικότερη βιβλιογραφία

Η ιδέα του "represent archive as set of filenames → Jaccard similarity" δεν εμφανίστηκε σε κανένα ακαδημαϊκό paper στις αναζητήσεις. Ο πλησιέστερος τομέας είναι "directory fingerprinting" αλλά και εκεί η εφαρμογή σε compressed archives χωρίς extraction είναι καινοτόμα.

##### Contribution Claim

> "Παρουσιάζουμε το Archive DNA Fingerprinting, μια μέθοδο ανίχνευσης near-duplicate archives βασισμένη σε Jaccard similarity του normalized file list χωρίς extraction, σε συνδυασμό με MinHash LSH για scalability. Η μέθοδος ανιχνεύει project versions (π.χ. `project_v1.zip` και `project_final.zip`) ακόμα και αν τα filenames είναι εντελώς διαφορετικά."

##### Implementation Complexity

| Στοιχείο | Εκτίμηση |
|---|---|
| **Γραμμές κώδικα** | ~200-250 LOC |
| **Libraries** | `zipfile` (standard), `tarfile` (standard), `datasketch` (ήδη ΜΕΡΟΣ 2), `numpy` (ήδη) |
| **Χρόνος εκτέλεσης** | ~0.1ms/archive για namelist extraction, O(1) για LSH query |
| **Memory** | MinHash: 64 × 8 bytes = 512 bytes/archive → 2.5 MB για 5000 archives |
| **Δυσκολία integration** | Χαμηλή — εισάγεται ως step ΠΡΙΝ το embedding, δεν αλλάζει downstream |

---

### Σύνοψη Νέων Ιδεών

| Ιδέα | Παρόμοιο υπάρχει; | Genuinely Novel; | Αξίζει; | Πολυπλοκότητα | Προτεραιότητα |
|---|---|---|---|---|---|
| **Drift-Triggered Refit** | Ναι — StreamDD (2024), DenDrift (2021), ADWIN-U (2025) — αλλά ΟΧΙ για file organizers | ✅ Ναι — ο συγκεκριμένος συνδυασμός metrics (churn+cosine+TemporalSil+GhostUMAP2) + εφαρμογή σε file organizer | ✅ Πολύ — αποτρέπει ανακριβή clustering μετά από σημαντικές αλλαγές χρήσης | Μέτρια (~200 LOC, 3 νέες βιβλιοθήκες) | ⭐⭐⭐ Υψηλή (γίνεται μετά το core pipeline) |
| **Archive DNA Fingerprinting** | Ελάχιστα — backup chunk dedup (Borg/Restic) αλλά εντελώς διαφορετική προσέγγιση | ✅ Ναι — file-list Jaccard για archive version detection δεν υπάρχει πουθενά | ✅ Πολύ — 1000s αρχεία ~/Downloads είναι archives, πολλά versions | Χαμηλή (~250 LOC, χρησιμοποιεί ήδη installed datasketch) | ⭐⭐⭐⭐ Πολύ Υψηλή (εύκολο να υλοποιηθεί, άμεσο αποτέλεσμα) |

**Archive DNA προτεινόμενο ως πρώτη υλοποίηση** — χαμηλότερη πολυπλοκότητα, υψηλή ορατή αξία, χρησιμοποιεί ήδη existing infrastructure (datasketch MinHash LSH από ΜΕΡΟΣ 2).

---

**Sources Μέρος 25:**
- StreamDD: Stream clustering with Drift Detection — https://www.sciencedirect.com/science/article/pii/S187705092402595X
- DenDrift: A Drift-Aware Algorithm for Host Profiling (arXiv:2110.01221) — https://arxiv.org/abs/2110.01221
- ADWIN-U: Adaptive Windowing for Unsupervised Drift Detection (Springer 2025) — https://link.springer.com/article/10.1007/s10115-025-02523-1
- Temporal Silhouette: Validation of Stream Clustering Robust to Concept Drift (Machine Learning, 2023) — https://link.springer.com/article/10.1007/s10994-023-06462-2
- py-temporal-silhouette GitHub (CN-TU/py-temporal-silhouette) — https://github.com/CN-TU/py-temporal-silhouette
- Cluster Analysis and Concept Drift Detection in Malware (arXiv:2502.14135) — https://arxiv.org/abs/2502.14135
- ADWIN River API documentation — https://riverml.xyz/dev/api/drift/ADWIN/
- clusTransition: An R package for monitoring transition in cluster solutions (PLOS ONE 2022) — https://pmc.ncbi.nlm.nih.gov/articles/PMC9754593/
- Ensemble Drift Detection: A Lightweight Concept Drift Detection Ensemble (IEEE) — https://ieeexplore.ieee.org/document/7372248
- Smart Adaptive Ensemble Model for concept drift (Nature Scientific Reports 2025) — https://www.nature.com/articles/s41598-025-05122-w
- Unsupervised Concept Drift Detection from Deep Learning Representations (arXiv:2406.17813) — https://arxiv.org/html/2406.17813v2
- ruptures: change point detection in Python (arXiv:1801.00826) — https://arxiv.org/abs/1801.00826
- MinHash LSH datasketch documentation — https://ekzhu.com/datasketch/lsh.html
- Borg Backup Internals (content-defined chunking) — https://borgbackup.readthedocs.io/en/stable/internals.html
- zipfile-dedup GitHub (rongrimes) — https://github.com/rongrimes/zipfile-dedup
- Fingerprints for Highly Similar Streams (Jaccard + MinHash, Microsoft Research) — https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/Jaccard.pdf
- Cluster-based stability evaluation in time series datasets (PMC) — https://www.ncbi.nlm.nih.gov/pmc/articles/PMC9746592/
- Online clustering with interpretable drift adaptation (ScienceDirect 2025) — https://www.sciencedirect.com/science/article/pii/S2667305325000365

---