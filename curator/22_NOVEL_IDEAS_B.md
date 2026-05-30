## ΜΕΡΟΣ 22: ΠΡΩΤΟΠΟΡΙΑΚΕΣ ΙΔΕΕΣ Α-Ε — ΑΞΙΟΛΟΓΗΣΗ
*(Έρευνα 2026-05-22 — 20+ searches, 35+ sources)*

---

### Ιδέα Α — File Biography

**Ορισμός:** Κάθε αρχείο κρατά ιστορικό: πότε σκαναρίστηκε, σε ποιο cluster ανήκε, πόσες φορές άλλαξε cluster, αν ο χρήστης το απέρριψε. Αρχείο που άλλαξε cluster 3+ φορές = "ασταθές" → Review. Ανίχνευση temporal drift χωρίς επανεκπαίδευση.

#### α) Υπάρχει παρόμοιο;

**Ακαδημαϊκή βιβλιογραφία:**

- **clusTransition (PLoS One 2022, PMC 9754593)**: R package υλοποίηση του MONIC framework για παρακολούθηση transitions σε temporal clustering. Παρακολουθεί split, merge, disappear, emerge events σε επίπεδο **cluster** — ΟΧΙ σε επίπεδο **ατόμου/αρχείου**. Κοντινότερο που βρέθηκε, αλλά κάνει aggregate cluster monitoring, όχι per-file biography.

- **Cluster-based stability evaluation in time series (SpringerLink + PMC 9746592)**: Μετράει stability ανά cluster σύνολο, όχι ανά σημείο.

- **GhostUMAP2 (arXiv:2507.17174)**: Μετράει (r,d)-stability per-point στο UMAP embedding, αλλά δεν κρατά ιστορικό cluster assignments.

- **Assigning Confidence: K-partition Ensembles (arXiv:2602.18435)**: Αναφέρει ότι σημεία σε cluster boundaries αλλάζουν label across runs — αλλά δεν προτείνει history tracking ως feature.

**Εφαρμογές σε file systems:** Κανένα published paper ή εμπορικό εργαλείο δεν κρατά per-file cluster assignment history ως organizational signal.

**macOS metadata API (osxmetadata):**
```python
from osxmetadata import OSXMetaData

meta = OSXMetaData('/path/to/file.pdf')
use_count = meta.kMDItemUseCount      # ή meta['usecount'] — πόσες φορές ανοίχτηκε
used_dates = meta.kMDItemUsedDates    # λίστα ημερομηνιών ανοίγματος
last_used = meta.kMDItemLastUsedDate  # τελευταία φορά — Finder "Last Opened"
```

Σημαντικό: `kMDItemLastUsedDate` ενημερώνεται **μόνο** όταν χρήστης ανοίγει το αρχείο — δεν ενημερώνεται από automated processes. Άρα είναι αξιόπιστος δείκτης ανθρώπινης χρήσης.

#### β) Αξίζει;

**ΝΑΙ — genuinely novel και άμεσα υλοποιήσιμο.**

Το "File Biography" συνδυάζει:
1. **Cluster assignment history** (νέο — από τη SQLite μας)
2. **macOS usage metadata** (υπάρχει ήδη — kMDItemUseCount, kMDItemUsedDates)
3. **User rejection history** (από το approval workflow μας)

Ανίχνευση που δεν υπάρχει πουθενά: *"αυτό το αρχείο έχει αλλάξει cluster 4 φορές — πιθανώς ανήκει στη μεθόριο δύο domains → βάλ' το στο Review αντί να το αποφασίσει αυτόματα το σύστημα."*

Η σύνδεση με GhostUMAP2 είναι ενδιαφέρουσα: τα αρχεία με χαμηλό (r,d)-stability κατά UMAP είναι ακριβώς αυτά που θα αλλάζουν cluster συχνά → το File Biography ποσοτικοποιεί αυτό empirically.

#### γ) Τι χρειάζεται;

**SQLite schema (νέος πίνακας):**
```sql
CREATE TABLE file_biography (
    id            INTEGER PRIMARY KEY,
    file_id       INTEGER NOT NULL REFERENCES files(id),
    event_ts      TEXT NOT NULL,          -- ISO timestamp
    event_type    TEXT NOT NULL,          -- 'scan', 'cluster_assign', 'user_reject', 'user_approve', 'cluster_change'
    cluster_id    INTEGER,               -- cluster assigned at this event
    cluster_name  TEXT,                  -- human-readable
    confidence    REAL,                  -- HDBSCAN probability at this event
    notes         TEXT                   -- π.χ. 'refit v1→v2'
);

CREATE INDEX idx_bio_file ON file_biography(file_id);
```

**Stability score (απλό υπολογιστικό μοντέλο):**
```python
def file_stability_score(file_id: int, db) -> float:
    """
    Returns 0.0 (totally unstable) to 1.0 (perfectly stable).
    Αρχείο που ποτέ δεν άλλαξε cluster → 1.0
    Αρχείο που άλλαξε 5 φορές σε 10 scans → 0.0
    """
    events = db.query(
        "SELECT event_type FROM file_biography WHERE file_id=? ORDER BY event_ts",
        (file_id,)
    )
    cluster_changes = sum(1 for e in events if e['event_type'] == 'cluster_change')
    total_scans = sum(1 for e in events if e['event_type'] == 'scan')

    if total_scans <= 1:
        return 1.0  # Αρκετά δεδομένα δεν υπάρχουν ακόμα

    return max(0.0, 1.0 - (cluster_changes / (total_scans - 1)))


# Routing rule:
# stability_score < 0.5 AND user_reject_count > 0 → Route to Review (bypass DenStream)
```

**Πολυπλοκότητα:** Πολύ χαμηλή. Ένας SQLite πίνακας + ένα write event κάθε φορά που αλλάζει cluster assignment. Δεν χρειάζεται ML.

---

### Ιδέα Β — Semantic Spotlight

**Ορισμός:** Όταν ο χρήστης κάνει Spotlight search για κάτι που ήδη οργανώσαμε, καταγράφουμε το event ως implicit signal: "ο χρήστης δεν βρήκε το αρχείο εκεί που το βάλαμε." Implicit feedback χωρίς explicit rating.

#### α) Υπάρχει παρόμοιο;

**Ακαδημαϊκή βιβλιογραφία:**
- **Bergman 2008 (ACM TOIS)** — "Search-after-organization rate" ως quality metric: *αν user κάνει Spotlight για αρχείο που οργανώσαμε = failed placement.* Ορίστηκε ως metric, ποτέ δεν υλοποιήθηκε αυτόματα.
- **Bergman 2010 (JASIST)** — Navigation success rate: χρησιμοποιεί task-based retrieval studies, όχι automatic monitoring.

**Εμπορικά εργαλεία:** Κανένα εμπορικό file organizer (DEVONthink, Dropbox, VaultSort) δεν παρακολουθεί Spotlight searches ως implicit feedback.

#### β) Αξίζει; — ΠΡΟΒΛΗΜΑΤΙΚΗ ΙΔΕΑ

**Τεχνικό πρόβλημα #1: Δεν μπορείς να παρακολουθείς τι ψάχνει ο χρήστης στο Spotlight.**

Από πολλαπλές πηγές:
- NSMetadataQuery / MDQuery: επιτρέπει στην **εφαρμογή** να κάνει queries στο Spotlight index — ΔΕΝ επιτρέπει παρακολούθηση queries άλλων εφαρμογών ή του χρήστη
- Apple Developer Forums: "Callers of MDQuery / NSMetadataQuery could not only handle such results, but they have also come to expect these, yet these remain unavailable to third-party applications."
- Objective-See security research: Τρίτες εφαρμογές ΔΕΝ μπορούν να intercept user Spotlight queries χωρίς exploitation — ακόμα και με SIP disabled, δεν υπάρχει documented API
- **Χωρίς SIP disable**: Αδύνατο να παρακολουθήσεις user Spotlight queries

**Τεχνικό πρόβλημα #2: Ακόμα κι αν μπορούσες να ξέρεις ότι ο χρήστης άνοιξε ένα αρχείο, δεν ξέρεις ΠΩΣ το βρήκε.**

- `kMDItemLastUsedDate` αλλάζει **όταν ο χρήστης ανοίγει το αρχείο**, αλλά δεν σου λέει αν το βρήκε μέσω Spotlight, Finder navigation, recent items, ή Alfred
- FSEvents καταγράφει filesystem events αλλά όχι "application source" του opening
- Correlation "Spotlight search → αρχείο ανοίχτηκε μέσα σε 30 seconds" είναι heuristic, όχι confirmed

**Τεχνικό πρόβλημα #3: Privacy**

Ακόμα κι αν ήταν τεχνικά feasible, monitoring των Spotlight queries χωρίς explicit user consent είναι privacy violation. Το macOS δεν παρέχει API για αυτό ακριβώς γι' αυτό τον λόγο.

#### γ) Τι χρειάζεται;

**Η ιδέα ΔΕΝ είναι υλοποιήσιμη** όπως περιγράφεται στο macOS χωρίς:
1. SIP disable (μη αποδεκτό)
2. Exploitation of Spotlight plugin vulnerability (δεν κάνουμε αυτό)
3. System Extension που εγκαθίσταται στο OS level (ενάντια στο Curator's philosophy)

**Εναλλακτική που ΔΟΥΛΕΥΕΙ:**
Αντί να παρακολουθούμε τι ψάχνει ο χρήστης, παρακολουθούμε τι **ανοίγει**:
```python
# Παρακολούθηση με FSEvents:
from fsevents import Observer, Stream

def file_opened_handler(event):
    # Καταγράφει αρχεία που ανοίγονται από ~/Downloads ή Organized folders
    # Ενημερώνει kMDItemLastUsedDate implicitly (το κάνει το macOS)
    # Μπορούμε να εντοπίσουμε: "αρχείο οργανώθηκε αλλά user επέστρεψε να το ψάξει"
    pass
```

**Verdict:** Ιδέα Β ως "Spotlight interception" = **μη υλοποιήσιμη** χωρίς serious privacy issues. Η υποστηρικτική ιδέα (track file usage behavior after organization) είναι ήδη καλυμμένη από το `kMDItemLastUsedDate` + File Biography (Ιδέα Α).

---

### Ιδέα Γ — Cold Start Bootstrap via Existing Folder Structure

**Ορισμός:** Αντί να ξεκινάμε με άδεια clusters, διαβάζουμε την υπάρχουσα δομή φακέλων (~/Documents, ~/Desktop) ως weak supervision. Φάκελος "EPL326" → χρησιμοποίησε τα αρχεία μέσα του ως seed embedding για initial cluster. Μην υιοθετείς τη δομή, χρήσιμοποίησε την ως prior.

#### α) Υπάρχει παρόμοιο;

**Seeded K-means (academia):**
- Seeded K-Means (Basu et al., 2002): Χρησιμοποιεί labeled data για initialization — initial centroid για cluster i = mean of seed points with label i. Seeds χρησιμοποιούνται μόνο για initialization, όχι σε επόμενα βήματα.
- Αυτό είναι ακριβώς αυτό που θέλουμε! Τα αρχεία σε folder "EPL326" = seed points για cluster "EPL326".

**Semi-supervised clustering με must-link/cannot-link (πολλά papers):**
- "A Framework for Deep Constrained Clustering" (arXiv:2101.02792)
- "Semi-Supervised Clustering via Structural Entropy" (arXiv:2312.10917)
- Must-link: αρχεία στον ίδιο φάκελο → ίδιο cluster (soft constraint)
- Cannot-link: αρχεία σε πολύ διαφορετικούς φακέλους → διαφορετικά clusters (soft constraint)

**ClusterLLM (arXiv:2305.14871):** Δέχεται few-shot annotations ως prior, αλλά δεν έχει explicit "folder structure" initialization. Μπορεί να χρησιμοποιηθεί: folder name = "perspective" instruction.

**k-LLMmeans (arXiv:2502.09667):** Δεν αναφέρει folder structure initialization. Αλλά: ο k-LLMmeans ξεκινά με k-means++ seeding — μπορεί να αντικατασταθεί με seeded initialization από υπάρχοντες φακέλους.

**Εμπορικά εργαλεία:** Κανένα δεν κάνει αυτό αυτόματα.

#### β) Αξίζει;

**ΝΑΙ — genuinely useful, ιδίως για cold start.**

Πλεονεκτήματα:
1. **Λύνει το cold start problem**: Αντί ο HDBSCAN να ξεκινά από μηδέν, έχει ήδη seeds που αντικατοπτρίζουν την τρέχουσα user mental model
2. **Μηδενικό κόστος**: Οι φάκελοι υπάρχουν ήδη — δεν χρειάζεται extra user effort
3. **Soft prior**: Δεν αναγκάζει τη δομή — αν ο HDBSCAN/k-means++ βρει καλύτερη ομαδοποίηση, ελεύθερος να την ακολουθήσει

Ρίσκα:
1. **Αναπαραγωγή bad organization**: Αν ο χρήστης έχει κακώς οργανωμένους φακέλους → οι seeds θα ενισχύσουν την κακή δομή
2. **Μερική κάλυψη**: Τα αρχεία στο ~/Downloads πιθανώς δεν έχουν αντίστοιχο folder structure → seeds από ~/Documents δεν καλύπτουν όλα τα cluster

**Σύγκρουση seed vs. embedding:** Τι γίνεται αν αρχείο στο folder "EPL326" κάνει embed κοντά στο cluster "Personal Projects"; Η seed initialization δίνει ένα starting point — η βελτιστοποίηση μπορεί να αποφασίσει αλλιώς. Αυτό είναι expected behavior (soft prior, όχι hard constraint).

#### γ) Τι χρειάζεται;

**Seeded HDBSCAN (δεν υπάρχει native) → Εναλλακτική: seeded k-means για initial centroids:**

```python
import numpy as np
from sklearn.cluster import KMeans

def bootstrap_from_folder_structure(root_dirs: list[str], embeddings_db) -> np.ndarray:
    """
    Reads existing folder structure and returns initial centroids.
    root_dirs: π.χ. ['~/Documents', '~/Desktop']
    """
    seeds = {}  # folder_name → list of embeddings

    for root in root_dirs:
        for folder in os.scandir(os.path.expanduser(root)):
            if not folder.is_dir():
                continue
            folder_name = folder.name
            folder_embeddings = []

            for f in os.scandir(folder.path):
                if f.is_file():
                    emb = embeddings_db.get(f.path)  # None αν δεν έχει embed
                    if emb is not None:
                        folder_embeddings.append(emb)

            if len(folder_embeddings) >= 3:  # min 3 αρχεία για αξιόπιστο seed
                seeds[folder_name] = np.mean(folder_embeddings, axis=0)

    if not seeds:
        return None  # fallback σε k-means++

    return np.array(list(seeds.values()))  # (K, 1024) — initial centroids


# Χρήση στο pipeline:
seed_centroids = bootstrap_from_folder_structure(['~/Documents', '~/Desktop'], db)

if seed_centroids is not None:
    # Seeded k-means initialization:
    # Ξεκίνα UMAP → μετά k-means++ με seeded centroids ως init
    # ΟΧΙ στο HDBSCAN (δεν υποστηρίζει seeding)
    # HDBSCAN παραμένει unsupervised — τα seeds βοηθούν μόνο στη naming phase
    seed_centroids_10d = reducer.transform(seed_centroids)  # project στο UMAP space
    # Χρήσιμοποίησε seed_centroids_10d ως initial_centroids για k-LLMmeans
    # ή ως cluster name seeds για HERCULES
```

**Πρακτικό approach (πιο απλό):**
1. Διάβασε folder names από ~/Documents + ~/Desktop
2. Χρησιμοποίησε τα folder names ως **candidate cluster name seeds** στο HERCULES prompt
3. Αν νέο cluster έχει cosine similarity >0.7 με seed centroid → προτείνει το υπάρχον folder name

**Πολυπλοκότητα:** Χαμηλή-μέτρια. Το seeded initialization προσθέτει ~20 γραμμές κώδικα. Η integration με HERCULES απαιτεί τροποποίηση του prompt.

---

### Ιδέα Δ — Confidence Decay over Time

**Ορισμός:** Το cluster assignment confidence ενός αρχείου μειώνεται αν δεν το έχει ανοίξει ο χρήστης εδώ και X μήνες. Χαμηλό confidence + χαμηλή χρήση = υποψήφιο για Antilibrary. Κανένα σύστημα δεν κάνει αυτό.

#### α) Υπάρχει παρόμοιο;

**DenStream fading factor (λ):**
- Ο DenStream έχει ήδη exponential decay για **cluster weights** (βάρος micro-cluster): `weight(t) = weight(0) * e^(-λ*t)`
- Αυτό αφορά τη **βαρύτητα του cluster** — ξεθωριάζουν clusters που δεν λαμβάνουν νέα αρχεία
- ΔΕΝ αφορά το confidence ατόμου/αρχείου

**"Clustering Data Streams with Adaptive Forgetting" (IEEE 2029365):**
- Adaptive forgetting factor per cluster — κάθε cluster έχει δικό του λ βάσει data content
- Ακόμα σε cluster level, όχι per-item

**Forward Decay (DIMACS 2002, fwddecay.pdf):**
- Time decay model για streaming systems: `weight(n) = e^(-λ*n)` όπου n = time units
- Γενικό framework, εφαρμόστηκε σε itemsets — ΟΧΙ σε cluster assignment confidence

**Collaborative Filtering με temporal decay (SPJ 2024):**
- Decay function για ratings: `w(t) = e^(-α*(T-t))` όπου T = current time, t = rating time
- Εφαρμογή σε user-item ratings, όχι cluster assignments

**HDBSCAN soft membership (probabilities_):**
- Δίνει αρχικό confidence `p(file ∈ cluster_k)` κατά το clustering
- ΔΕΝ ενημερώνεται με τον χρόνο — static snapshot

**Συμπέρασμα:** Κανένα paper ή σύστημα δεν συνδυάζει **HDBSCAN soft membership + temporal decay based on user access** για per-file cluster assignment confidence. Genuinely novel.

#### β) Αξίζει;

**ΝΑΙ — και είναι άμεσα υλοποιήσιμο.**

Η βασική ιδέα είναι απλή και elegant:

```python
import math
from datetime import datetime, timedelta

def current_confidence(
    initial_prob: float,    # HDBSCAN probabilities_[i] — αρχικό confidence
    last_used_date,         # kMDItemLastUsedDate — None αν δεν ανοίχτηκε ποτέ
    use_count: int,         # kMDItemUseCount
    glosh_score: float,     # outlier score από HDBSCAN
    lambda_decay: float = 0.005  # ρυθμός decay (0.005 ≈ half-life 6 μήνες)
) -> float:
    """
    Computes current cluster assignment confidence incorporating time decay.
    Formula: confidence(t) = initial_prob * e^(-λ * days_since_last_use)
    """
    if use_count == 0 or last_used_date is None:
        # Αρχείο δεν ανοίχτηκε ποτέ — decay από creation date
        days_since_use = (datetime.now() - file_creation_date).days
    else:
        days_since_use = (datetime.now() - last_used_date).days

    # Exponential decay
    time_factor = math.exp(-lambda_decay * days_since_use)

    # Συνδυασμός με GLOSH: υψηλό outlier score επιταχύνει decay
    glosh_penalty = 1.0 - (glosh_score * 0.3)  # max 30% penalty από GLOSH

    return initial_prob * time_factor * max(0.0, glosh_penalty)


# Routing rules:
# current_confidence(file) < 0.3 AND use_count == 0 → Antilibrary candidate
# current_confidence(file) < 0.5 AND use_count > 0 → Review (user has used it, but confidence low)
# current_confidence(file) >= 0.7 → Stable, no action
```

**Σχέση με DenStream:** Ο DenStream ήδη χρησιμοποιεί λ για cluster-level decay. Η Ιδέα Δ **συμπληρώνει** αυτό με per-file decay βασισμένο σε actual user access — δύο ξεχωριστά αλλά συμβατά mechanisms.

**Σχέση με Antilibrary:** Ο Curator ήδη χρησιμοποιεί `kMDItemUseCount == 0` για Antilibrary. Η Ιδέα Δ προσθέτει nuance: αρχεία με `use_count > 0` αλλά `last_used > 18 μήνες` και `initial_prob < 0.5` → soft antilibrary candidate (δεν διαγράφεται αμέσως, αλλά flagged).

**Σχέση με GLOSH:** Ο GLOSH outlier score ήδη υπολογίζεται. Συνδυασμός GLOSH + time decay = πιο robust antilibrary detection.

#### γ) Τι χρειάζεται;

- Αποθήκευσε στη SQLite: `initial_hdbscan_prob` (float), `glosh_score` (float) — ήδη υπολογιζόμενα
- Διάβασε `kMDItemLastUsedDate` + `kMDItemUseCount` μέσω osxmetadata (ήδη έχουμε το library)
- Ο τύπος `e^(-λ*t)` υπολογίζεται on-the-fly κατά το review — δεν χρειάζεται αποθήκευση
- **Tuning λ**: Ξεκίνα με λ=0.005 (half-life ~140 μέρες) και tune εμπειρικά

**Πολυπλοκότητα:** Πολύ χαμηλή. Ένας μαθηματικός τύπος πάνω σε existing data. ~15 γραμμές κώδικα.

---

### Ιδέα Ε — Explainable Cluster Names

**Ορισμός:** Αντί μόνο folder name ("Networking Labs"), δείξε γιατί: "Θέλω να βάλω αυτά τα 12 αρχεία στο 'Networking Labs' γιατί μοιράζονται θέματα: TCP/IP, socket programming, Wireshark." Το "γιατί" προκύπτει ήδη από τον LLM που γεννά τα ονόματα — απλά το εμφανίζουμε στο UI.

#### α) Υπάρχει παρόμοιο;

**Ακαδημαϊκή βιβλιογραφία:**

- **"Explainable clustering: Methods, challenges, and future opportunities" (J. Intelligent Systems 2025, de Gruyter):** Survey paper — καλύπτει explainability μεθόδους αλλά δεν αφορά file management UIs.
- **"Interpretable Clustering: A Survey" (arXiv:2409.00743, 2024):** Algorithmic interpretability — decision tree explanations, rule-based clustering. ΟΧΙ για LLM-generated natural language explanations.
- **"Trust in Transparency" (arXiv:2510.04968, Μέρος 3 της έρευνάς μας):** Explaining AI reasoning αυξάνει αποδοχή — validated.
- **McKinsey AI trust survey 2024:** 40% CIOs αναφέρουν explainability ως top AI adoption barrier.

**HERCULES (pyhercules):** Γεννά `title` (max 5-7 words) + `description` (1-2 sentences) per cluster. Η `description` είναι essentially ο rationale — αλλά δεν είναι formatted ως "γιατί" explanation. Από README: "The algorithm supports two main representation modes: 'direct' mode and 'description' mode."

**k-LLMmeans (arXiv:2502.09667):** Τα LLM-generated summaries ως centroids είναι ήδη "explainable" centroids — αλλά δεν formatάρονται ως "because they share X, Y, Z" explanations.

**TnT-LLM (arXiv:2403.12173):** Zero-shot taxonomy generation — δεν γεννά explicit "because" rationale στο output.

**Εμπορικά εργαλεία:** Κανένα δεν δείχνει "because" explanation στο file organization UI. DEVONthink, Dropbox, VaultSort: μόνο folder name, ποτέ rationale.

**HCI papers:** "Trust in Transparency" (Μέρος 3) επιβεβαιώνει ότι showing reasoning αυξάνει acceptance. Συγκεκριμένα για file organization UIs δεν υπάρχει validated study — **genuinely novel application**.

#### β) Αξίζει;

**ΝΑΙ — και η υλοποίηση είναι εξαιρετικά απλή** γιατί έχουμε ήδη τον LLM call για naming.

**Evidence ότι αξίζει:**
1. "Trust in Transparency" (arXiv:2510.04968): Explanation αυξάνει αποδοχή AI recommendations
2. McKinsey 2024: Explainability = #1 barrier για AI adoption
3. Dropbox Smart Move: "Potential siblings" preview αυξάνει acceptance rate — analogous mechanism
4. "Why Would You Suggest That?" (arXiv:2406.02018): Adding explanation αυξάνει self-reported trust

**Τι δεν γνωρίζουμε (gap):** Πόσο αυξάνεται συγκεκριμένα το acceptance rate για file organization cluster assignments όταν δείχνεις explanation. Αυτή η gap = novel research contribution αν μετρηθεί.

#### γ) Τι χρειάζεται;

**Επέκταση του HERCULES/naming prompt:**

```python
# Τωρινό prompt (απλοποιημένο):
prompt = f"""
Given these files: {filenames}
Key terms: {tfidf_terms}
Generate a folder name (2-4 words):
"""

# Νέο prompt με explanation:
prompt = f"""
Given these files: {filenames}
Key terms: {tfidf_terms}
Content snippets: {representative_snippets}

Generate a JSON with:
1. "name": folder name (2-4 words, specific)
2. "reason": one sentence starting with "These files share" explaining why they belong together
3. "key_topics": list of 3-5 main topics/keywords

Example output:
{{
  "name": "Networking Labs",
  "reason": "These files share topics in computer networking, including socket programming, TCP/IP protocols, and packet analysis with Wireshark.",
  "key_topics": ["TCP/IP", "socket programming", "Wireshark", "networking", "protocols"]
}}
"""

result = ollama.generate(model='qwen2.5:7b', prompt=prompt)
import json
cluster_info = json.loads(result['response'])
```

**UI display (React/Tauri):**
```tsx
// Στο approval panel:
<ClusterCard>
  <FolderIcon />
  <FolderName>{cluster.name}</FolderName>
  <ExplanationBadge>
    💡 {cluster.reason}
  </ExplanationBadge>
  <KeyTopics>
    {cluster.key_topics.map(t => <Tag>{t}</Tag>)}
  </KeyTopics>
  <FileList>{/* αρχεία που προτείνεται να μπουν */}</FileList>
</ClusterCard>
```

**Αποθήκευση στη SQLite:**
```sql
ALTER TABLE clusters ADD COLUMN reason TEXT;  -- "These files share..."
ALTER TABLE clusters ADD COLUMN key_topics TEXT;  -- JSON array
```

**Κόστος:** Ελάχιστο. Ήδη κάνουμε LLM call για naming — απλά επεκτείνουμε το response format σε JSON. Ένα extra field στο SQLite. Μικρή αλλαγή στο React UI.

**Πολυπλοκότητα:** Εξαιρετικά χαμηλή — 20-30 γραμμές αλλαγές συνολικά. Αυτό είναι η πιο "free lunch" ιδέα στη λίστα.

---

### Σύνοψη Ιδεών (πίνακας)

| Ιδέα | Παρόμοιο υπάρχει; | Genuinely Novel; | Αξίζει; | Πολυπλοκότητα | Προτεραιότητα |
|---|---|---|---|---|---|
| **Α — File Biography** | clusTransition (cluster level, R) — ΟΧΙ per-file | ✅ ΝΑΙ — per-file cluster history ως organizational signal | ✅ ΝΑΙ | Χαμηλή (SQLite πίνακας) | **P1 — υλοποίησε πρώτο** |
| **Β — Semantic Spotlight** | Bergman 2008 (metric only, not automated) | ⚠️ Ορισμός novel, υλοποίηση αδύνατη | ❌ ΟΧΙ — τεχνικό impasse | N/A | **P5 — abandon** |
| **Γ — Cold Start Bootstrap** | Seeded K-means (Basu 2002), semi-supervised lit | ✅ ΝΑΙ — εφαρμογή σε personal files + folder structure | ✅ ΝΑΙ | Μέτρια (seed generation + HERCULES integration) | **P3** |
| **Δ — Confidence Decay** | DenStream λ (cluster level) — ΟΧΙ per-file | ✅ ΝΑΙ — HDBSCAN prob + time decay + GLOSH | ✅ ΝΑΙ | Χαμηλή (~15 γραμμές) | **P2 — trivial to add** |
| **Ε — Explainable Cluster Names** | HERCULES description (partial) — ΟΧΙ "because" UI | ✅ ΝΑΙ — "because" explanation στο file organization UI | ✅ ΝΑΙ | Ελάχιστη (~30 γραμμές) | **P1 — free lunch** |

---

### Σύσταση Υλοποίησης

**Sprint 3 (νέα προσθήκη):**
- Ιδέα Ε (Explainable Names): Extend LLM naming prompt → JSON με `reason` + `key_topics`. Αλλαγή ~30 γραμμές. Υψηλό value / χαμηλό κόστος.
- Ιδέα Δ (Confidence Decay): Πρόσθεσε `initial_hdbscan_prob` + `glosh_score` στη SQLite. Υπολόγισε decay on-the-fly. ~15 γραμμές.

**Sprint 4:**
- Ιδέα Α (File Biography): Νέος SQLite πίνακας `file_biography`. Write events κατά scan + refit. Stability score function. ~100 γραμμές.
- Ιδέα Γ (Cold Start Bootstrap): Folder structure reader. Seeded centroids για HERCULES initialization. ~80 γραμμές.

**Abandon:**
- Ιδέα Β (Semantic Spotlight): Τεχνικά αδύνατο χωρίς privacy violations. Αντικαταστάθηκε από combination Ιδέα Α + `kMDItemLastUsedDate` tracking.

---

**Sources Μέρος 22:**
- clusTransition R package (PLoS One 2022, PMC 9754593) — https://pmc.ncbi.nlm.nih.gov/articles/PMC9754593/
- GhostUMAP2 (r,d)-stability arXiv:2507.17174 — https://arxiv.org/abs/2507.17174
- osxmetadata PyPI — https://pypi.org/project/osxmetadata/
- osxmetadata GitHub (RhetTbull) — https://github.com/RhetTbull/osxmetadata
- macOS Forensics: kMDItemLastUsedDate — https://forensic4cast.com/2016/10/macos-timestamps-from-extended-attributes-and-spotlight/
- NSMetadataQuery Apple docs — https://developer.apple.com/documentation/foundation/nsmetadataquery
- Objective-See Spotlight to Apple Intelligence (0day) — https://objective-see.org/blog/blog_0x81.html
- MDQuery interception blocked for third-party apps — https://cedowens.medium.com/querying-spotlight-apis-with-jxa-3ae4bb9af3b4
- Seeded K-Means (Basu 2002) — https://rdrr.io/cran/SSLR/man/seeded_kmeans.html
- Semi-supervised Clustering via Structural Entropy arXiv:2312.10917 — https://arxiv.org/pdf/2312.10917
- Framework for Deep Constrained Clustering arXiv:2101.02792 — https://arxiv.org/pdf/2101.02792
- ClusterLLM arXiv:2305.14871 — https://arxiv.org/abs/2305.14871
- k-LLMmeans arXiv:2502.09667 — https://arxiv.org/abs/2502.09667
- Clustering Data Streams with Adaptive Forgetting IEEE — https://ieeexplore.ieee.org/document/8029365/
- Forward Decay streaming model (DIMACS) — https://dimacs.rutgers.edu/~graham/pubs/papers/fwddecay.pdf
- HDBSCAN soft clustering docs — https://hdbscan.readthedocs.io/en/latest/soft_clustering.html
- HDBSCAN soft clustering explanation — https://hdbscan.readthedocs.io/en/latest/soft_clustering_explanation.html
- Assigning Confidence: K-partition Ensembles arXiv:2602.18435 — https://arxiv.org/pdf/2602.18435
- Explainable clustering Methods and Challenges (J. Intelligent Systems 2025) — https://www.degruyterbrill.com/document/doi/10.1515/jisys-2024-0477/html
- Interpretable Clustering Survey arXiv:2409.00743 — https://arxiv.org/html/2409.00743v1
- pyhercules GitHub — https://github.com/bandeerun/pyhercules
- HERCULES arXiv:2506.19992 — https://arxiv.org/abs/2506.19992
- TnT-LLM arXiv:2403.12173 — https://arxiv.org/abs/2403.12173
- McKinsey AI Trust Explainability 2024 — https://www.mckinsey.com/capabilities/quantumblack/our-insights/building-ai-trust-the-key-role-of-explainability
- Is Trust Correlated with Explainability arXiv:2504.12529 — https://arxiv.org/pdf/2504.12529
- Why Would You Suggest That? arXiv:2406.02018 — https://arxiv.org/pdf/2406.02018
- FSEvents macOS forensics — https://www.hexordia.com/blog/mac-forensics-analysis

---
