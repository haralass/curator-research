## ΜΕΡΟΣ 34: MULTI-SIGNAL ROUTING DECISION
*(Έρευνα 2026-05-23)*

---

### ΘΕΜΑ 1: Πώς κάνουν Signal Fusion τα State-of-the-Art Systems

#### 1.1 Το Dropbox Smart Move Paradigm

Το Dropbox Smart Move (2021-2024) είναι το πιο relevant reference για το Curator γιατί λύνει ακριβώς το ίδιο πρόβλημα: "σε ποιον φάκελο πάει αυτό το αρχείο;". Τα signals που χρησιμοποιεί:

1. **Filename + extension** του αρχείου που μετακινείται (π.χ. `tax_2024.pdf`)
2. **Candidate folder name** (π.χ. `Finance/`)
3. **Sibling files** — τα ονόματα των αρχείων που ήδη υπάρχουν μέσα στον φάκελο

Το κρίσιμο εύρημα: τα siblings δίνουν **περισσότερο context από το folder name**. Ένας φάκελος `Finance/` που περιέχει `w-2.pdf, taxreturn_2023.pdf, charitable.img` είναι πολύ πιο descriptive από μόνο το όνομα `Finance`. Αυτό μεταφράζεται απευθείας στο Curator: ο DenStream micro-cluster centroid είναι essentially ένα weighted average των embeddings όλων των αρχείων — δηλαδή είναι ήδη το "sibling signal" ενσωματωμένο.

**Architecture:** Character-level + GloVE word-level embeddings → similarity matrices (context-to-candidate, context-to-siblings) → deep neural network (< 20 layers) → confidence score. Το τελικό ranking χωρίζεται σε top 20% (high confidence, εμφανίζεται prominently), middle tier, και bottom 80% (κρύβονται).

**Αποτέλεσμα:** 73% offline accuracy, high-confidence suggestions >90% acceptance rate.

**Lesson για Curator:** Το sibling context είναι ήδη ενσωματωμένο στον DenStream centroid — δεν χρειαζόμαστε ξεχωριστό sibling lookup. Αλλά μπορούμε να αυξήσουμε το weight του cosine similarity ακριβώς επειδή ο centroid κωδικοποιεί ήδη το "τι υπάρχει στον cluster".

---

#### 1.2 Early vs Late Fusion για Ετερογενή Signals

Το Curator έχει ακριβώς το πρόβλημα που ορίζει η βιβλιογραφία ως **ετερογενή fusion**: cosine similarity (0-1 float), useCount (0-∞ integer), extension (categorical), FLACON flags (multi-bit boolean), temporal signals (datetime/boolean). Αυτά δεν μπορούν να concatenated naively.

| Fusion Type | Πότε αποδίδει | Αδυναμία |
|-------------|--------------|----------|
| **Early fusion** (concatenate → single model) | Ομοιογενή modalities (π.χ. text + text), μεγάλο training set | Αποτυγχάνει με ετερογενή signals, overfits σε μικρά datasets |
| **Late fusion** (ξεχωριστά models → combine scores) | Ετερογενή modalities, flexibility να προσθέσεις νέο signal | Χάνει cross-modal interactions |
| **Hybrid / tiered** | Κατηγοριοποίηση σε deterministic + probabilistic tiers | Πιο σύνθετη υλοποίηση |

**Συμπέρασμα για Curator:** Χρησιμοποιούμε **tiered hybrid approach**:
- Tier 0: Deterministic rules (extension, creator) — δεν χρειάζεται ML καθόλου
- Tier 1: Strong prior routing (metadata signals → hard priors)
- Tier 2: Late fusion του cosine score με normalized metadata boosters

Αυτό είναι συνεπές με research που δείχνει: για μικρά personal datasets (< 10.000 files), learned weights overfits. Ο χρήστης δεν έχει αρκετά labeled examples για να train neural fusion model. Άρα: **rule-based weighted sum με hand-tuned weights** είναι η πιο robust επιλογή.

---

#### 1.3 Κανονικοποίηση Signals Διαφορετικών Κλιμάκων

Πριν οποιαδήποτε fusion, κάθε signal πρέπει να φτάσει στο `[0, 1]`:

```
cosine_similarity:     ήδη [0, 1] — κρατάμε as-is

use_count:             sigmoid(useCount / 10.0) → πλησιάζει 1 για > 30 uses
                       Εναλλακτικά: min(useCount / max_useCount, 1.0)

recency_score:         exp(-days_since_use / 30)  → 1 αν χρησιμοποιήθηκε σήμερα,
                       0.5 αν πριν 21 μέρες, ~0 αν πριν 3+ μήνες

boolean flags:         0.0 ή 1.0 απευθείας

categorical match:     1.0 αν match, 0.0 αν no match, 0.5 αν partial/unknown
```

---

#### 1.4 Τι Γίνεται Όταν Signals Contradicting

Παράδειγμα conflict: ένα αρχείο `assignment_EPL326.pdf` κατέβηκε από email sender `professor@ucy.ac.cy` (→ academic) αλλά το embedding ταιριάζει εξίσου καλά σε cluster "Personal Notes" και cluster "EPL326".

Η βιβλιογραφία προτείνει **source credibility hierarchy**: χρησιμοποίησε τον signal που είναι πιο reliable. Στο Curator αυτό μεταφράζεται σε:

1. Αν deterministic rule fires → ακολούθησε αδιαμφισβήτητα (extension/creator δεν negotiated)
2. Αν strong prior (provenance) contradicts embedding → **strong prior νικάει** αλλά flag για review
3. Αν δύο soft signals contradict → χαμήλωσε confidence score, πήγαινε σε "requires_review"

---

### ΘΕΜΑ 2: Confidence Score Design — Η Φόρμουλα

#### 2.1 Ορισμός του Routing Score

```
routing_score(file f, cluster c) =
    w_sem  * cosine(embed(f), centroid(c))
  + w_prov * provenance_match(f, c)
  + w_ext  * extension_match(f, c)
  + w_beh  * behavioral_affinity(f, c)
  + w_flac * flacon_alignment(f, c)
  + w_temp * temporal_cohesion(f, c)
```

**Προτεινόμενα weights (hand-tuned, personal system):**

| Signal Component | Weight | Λογική |
|-----------------|--------|--------|
| `w_sem` (semantic/cosine) | 0.35 | Core signal, αλλά μόνο όταν tier 0/1 δεν fires |
| `w_prov` (provenance: WhereFroms, Quarantine, Creator) | 0.25 | Πολύ reliable, direct evidence |
| `w_ext` (extension match profile) | 0.15 | Deterministic αν tier 0 — αλλιώς soft boost |
| `w_beh` (useCount + recency) | 0.10 | Behavioral history, σπάνια available για νέα files |
| `w_flac` (FLACON flags alignment) | 0.10 | Structural category match |
| `w_temp` (temporal cohesion) | 0.05 | Weak signal, tie-breaker μόνο |

Σύνολο weights = 1.00

**Confidence thresholds:**

```
routing_score >= 0.80  →  AUTO_ROUTE     (το σύστημα κινείται μόνο)
routing_score  0.50-0.79  →  SUGGEST    (ρωτάει τον χρήστη, παρουσιάζει top-2)
routing_score < 0.50   →  NEW_CLUSTER    (candidate για νέο cluster ή manual)
```

---

#### 2.2 Component Definitions

**`provenance_match(f, c)`** — συνδυάζει 3 provenance sub-signals:
```python
def provenance_match(file_signals, cluster_profile):
    score = 0.0
    # kMDItemWhereFroms domain match
    if file_signals.get('domain') in cluster_profile.get('known_domains', []):
        score += 0.5
    # kMDItemCreator match
    if file_signals.get('creator') in cluster_profile.get('known_creators', []):
        score += 0.3
    # LSQuarantineAgentName match
    if file_signals.get('quarantine_agent') in cluster_profile.get('known_agents', []):
        score += 0.2
    return min(score, 1.0)
```

**`extension_match(f, c)`** — ελέγχει αν η extension ανήκει στο extension profile του cluster:
```python
def extension_match(file_signals, cluster_profile):
    ext = file_signals.get('extension', '')
    ext_counts = cluster_profile.get('extension_histogram', {})
    if not ext_counts:
        return 0.5  # unknown cluster profile → neutral
    total = sum(ext_counts.values())
    return ext_counts.get(ext, 0) / total  # fraction of cluster with same ext
```

**`behavioral_affinity(f, c)`** — χρησιμοποιεί use history:
```python
def behavioral_affinity(file_signals, cluster_profile):
    use_count = file_signals.get('use_count', 0)
    days_since = file_signals.get('days_since_last_use', 999)
    if use_count == 0:
        return 0.0  # νέο αρχείο, χωρίς behavioral data
    recency = math.exp(-days_since / 30.0)
    usage = min(use_count / 20.0, 1.0)  # saturates at 20 uses
    return 0.6 * recency + 0.4 * usage
```

**`flacon_alignment(f, c)`** — πόσα FLACON bits ταιριάζουν:
```python
def flacon_alignment(file_signals, cluster_profile):
    file_type = file_signals.get('flacon_type')
    cluster_dominant_type = cluster_profile.get('dominant_flacon_type')
    if file_type and cluster_dominant_type:
        return 1.0 if file_type == cluster_dominant_type else 0.0
    return 0.5  # unknown
```

**`temporal_cohesion(f, c)`** — co-modification με files του cluster:
```python
def temporal_cohesion(file_signals, cluster_profile):
    # Αρχεία που τροποποιήθηκαν την ίδια ώρα (±10 λεπτά)
    comod_files = file_signals.get('co_modified_files', [])
    cluster_members = set(cluster_profile.get('member_paths', []))
    overlap = len(set(comod_files) & cluster_members)
    if not comod_files:
        return 0.0
    return min(overlap / len(comod_files), 1.0)
```

---

### ΘΕΜΑ 3: 3-Tier Decision Tree

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TIER 0 — DETERMINISTIC RULES (no ML, no embedding needed)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

IF extension IN {.dmg, .pkg, .mpkg}
    → route: ~/Applications/ (or ~/Downloads/Installers/)
    → confidence: 1.0, reason: "installer package"

IF extension IN {.app}
    → route: /Applications/
    → confidence: 1.0, reason: "application bundle"

IF kMDItemCreator == "Xcode" OR extension IN {.xcodeproj, .xcworkspace, .swift}
    → route: ~/Developer/
    → confidence: 1.0, reason: "Xcode artifact"

IF extension == ".ics"
    → route: ~/Documents/Calendar/
    → confidence: 1.0, reason: "calendar file"

IF extension IN {.vcf}
    → route: ~/Documents/Contacts/
    → confidence: 1.0, reason: "contact card"

IF LSQuarantineTypeNumber == 6 (sandboxed/restricted)
    → flag: sandboxed_origin = True
    → πρόσθεσε tag "sandboxed" αλλά ΜΗΝ override routing
    → reason: sandboxed δεν σημαίνει specific folder, μόνο provenance info

IF extension IN {.torrent} AND LSQuarantineAgentName contains "torrent"
    → route: ~/Downloads/Torrents/
    → confidence: 0.95, reason: "torrent file from torrent client"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TIER 1 — STRONG PRIOR ROUTING (confidence > 0.85)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

IF kMDItemWhereFroms domain == "mail.google.com" OR "mail.ucy.ac.cy"
   AND kMDItemCreator IN {"Mail", "Outlook", "Spark"}
   AND cosine(embed(f), work_cluster_centroid) > 0.45
    → route: work/email cluster
    → confidence: 0.90, reason: "email attachment from work domain"

IF kMDItemWhereFroms domain IN KNOWN_ACADEMIC_DOMAINS
   (π.χ. "ucy.ac.cy", "canvas.instructure.com", "elearning.ucy.ac.cy")
    → route: best matching university course cluster
    → confidence: 0.87, reason: "downloaded from university system"

IF AirDrop sender device IN user's known_devices
   AND co_modified_files overlap with known cluster > 0
    → route: cluster with highest co-modification overlap
    → confidence: 0.85, reason: "AirDrop from own device, temporal cohesion"

IF NER entities contain ["Prof.", "Dr.", "Assignment", "Deadline", "Grade"]
   AND kMDItemCreator IN {"Pages", "Microsoft Word", "Google Docs"}
    → route: most relevant course cluster (by cosine)
    → confidence: 0.85, reason: "academic document NER match"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TIER 2 — EMBEDDING-BASED ROUTING (standard case)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Υπολόγισε routing_score(f, c) για κάθε cluster c:
    score = weighted fusion formula (§2.1)

IF max_score >= 0.80:
    → AUTO_ROUTE to argmax cluster
    → log decision + all signal values

IF 0.50 <= max_score < 0.80:
    → SUGGEST top-2 clusters με siblings preview
    → Ρώτα τον user: "Πού να βάλω αυτό;"

IF max_score < 0.50:
    → NEW_CLUSTER candidate
    → Πρότεινε νέο φάκελο ή "Uncategorized"
    → Trigger DenStream για νέο micro-cluster
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### ΘΕΜΑ 4: Tie-Breaking & Ambiguous Files

#### 4.1 Ορισμός "Ambiguous"

Ένα αρχείο είναι ambiguous όταν:
```
|score(cluster_1) - score(cluster_2)| < 0.10
AND score(cluster_1) < 0.80
```

Δηλαδή: δύο clusters είναι σχεδόν ισόβαθμοι ΚΑΙ κανένας δεν φτάνει το auto-route threshold.

Παράδειγμα: `EPL326_notes.pdf` → cosine 0.68 για EPL326 cluster, cosine 0.63 για Thesis cluster. Διαφορά 0.05, κανένας > 0.80 → ambiguous.

#### 4.2 Tie-Breaking Hierarchy

Όταν βρεθεί tie (διαφορά < 0.10):

1. **Temporal cohesion ως tie-breaker** — ποιος cluster έχει περισσότερα members που τροποποιήθηκαν την ίδια ώρα; Αυτός νικάει.

2. **Provenance signal ως tie-breaker** — αν το αρχείο κατέβηκε από URL που ανήκει σε έναν cluster (π.χ. UCY eCampus → EPL326), αυτός νικάει.

3. **Recency bias** — αν ο χρήστης έχει χρησιμοποιήσει αρχεία από έναν cluster πιο πρόσφατα (last 7 days), ελαφρά προτίμηση σε αυτόν.

4. **Αν ακόμα tie** → SUGGEST mode με siblings preview (βλ. §4.3).

#### 4.3 Bridge Node Integration (από ΜΕΡΟΣ 3)

Το ΜΕΡΟΣ 3 ανέφερε betweenness centrality για να εντοπίσουμε bridge nodes — αρχεία που συνδέουν πολλά clusters. Αυτό integrάρεται ως εξής:

```python
if is_bridge_node_candidate(file_signals, clusters):
    # Αρχείο που ανήκει σε δύο κόσμους (π.χ. thesis + EPL326)
    # → dual-tag: βάλε symlink ή tag και στους δύο clusters
    # → ΜΗΝ force ένα folder, πρόσθεσε metadata tag
    return RoutingDecision(
        destination=primary_cluster,
        tags=[primary_cluster.name, secondary_cluster.name],
        is_bridge=True,
        confidence=max_score
    )
```

Η λογική: ένα bridge node δεν "ανήκει" σε έναν φάκελο — ανήκει σε δύο contexts. Η λύση για ένα macOS system είναι να το βάλεις σε έναν φάκελο αλλά να του δώσεις tags και για τους δύο.

#### 4.4 Siblings Preview για User Review

Όταν το σύστημα ρωτά τον χρήστη, δείχνει:

```
┌─────────────────────────────────────────────────────┐
│  Πού να βάλω: assignment_epl326_final.pdf           │
├─────────────────────────────────────────────────────┤
│  Option A: EPL326/ (score: 0.68)                    │
│  └── epl326_slides_week5.pdf                        │
│  └── lab3_solution.py                               │
│  └── midterm_notes.md                               │
├─────────────────────────────────────────────────────┤
│  Option B: Thesis/ (score: 0.63)                    │
│  └── thesis_outline_v3.docx                         │
│  └── related_work_notes.pdf                         │
│  └── bibliography.bib                               │
└─────────────────────────────────────────────────────┘
```

Αυτό το pattern (top-2 options + 3 siblings each) είναι αρκετό για τον χρήστη να αποφασίσει σε < 3 δευτερόλεπτα. Η απόφαση του χρήστη → feedback που upweights τον νικητή cluster για αυτό το pattern.

---

### ΘΕΜΑ 5: Signal Reliability Hierarchy

| Signal | Reliability (1-5) | Latency | Requires content read | Available for iCloud files |
|--------|------------------|---------|-----------------------|--------------------------|
| File extension | 5 | 0ms | No | Yes |
| `kMDItemCreator` | 4 | ~1ms | No | No (iCloud placeholder) |
| `LSQuarantineAgentName` | 4 | ~2ms (SQLite) | No | No |
| `kMDItemWhereFroms` | 4 | ~1ms | No | Partially |
| FLACON flags | 3 | ~5ms (computed) | Partial (filename only) | Yes |
| BGE-M3 cosine similarity | 3 | ~50-200ms | Yes (embedding) | No (need download) |
| NER entities | 3 | ~500ms+ | Yes (full text) | No |
| `kMDItemUseCount` | 3 | ~1ms | No | No (local only) |
| Temporal co-modification | 2 | ~10ms (fs scan) | No | Partial |
| `kMDItemUsedDates` | 2 | ~1ms | No | No |
| `LSQuarantineTypeNumber` | 2 | ~2ms (SQLite) | No | No |

**Σημειώσεις:**
- **iCloud placeholder files** (.icloud extension): μόνο extension, FLACON partial, και filename-based signals είναι διαθέσιμα. Το σύστημα πρέπει να το γνωρίζει και να αποφεύγει να κατεβάσει το file μόνο για routing.
- **Latency priority**: για real-time routing, χρησιμοποίησε πρώτα τα < 10ms signals. Embedding είναι batch operation — τρέχει async.
- **Reliability 5**: extension είναι ο μόνος 100% reliable signal. Ένα `.py` αρχείο είναι ΠΑΝΤΑ Python code, ανεξάρτητα από τι λέει το embedding.

---

### ΘΕΜΑ 6: Python Routing Engine — Code Sketch

```python
from dataclasses import dataclass, field
from typing import Optional
import math


@dataclass
class RoutingDecision:
    destination: str              # cluster name ή folder path
    confidence: float             # 0.0 - 1.0
    requires_review: bool         # True → ρώτα τον user
    reason: str                   # human-readable explanation
    tier: int                     # 0, 1, ή 2
    top_alternatives: list        # top-2 candidates για review mode
    is_bridge: bool = False       # True → dual-tag candidate
    suggested_tags: list = field(default_factory=list)
    signal_breakdown: dict = field(default_factory=dict)


# ─── Deterministic rule tables ───────────────────────────────────────────────

INSTALLER_EXTENSIONS = {".dmg", ".pkg", ".mpkg"}
APP_EXTENSIONS = {".app"}
DEV_EXTENSIONS = {".xcodeproj", ".xcworkspace", ".swift", ".xcassets"}
DEV_CREATORS   = {"Xcode", "Xcode 15", "Xcode 16"}
CALENDAR_EXTENSIONS = {".ics"}
CONTACT_EXTENSIONS  = {".vcf"}

ACADEMIC_DOMAINS = {
    "ucy.ac.cy", "canvas.instructure.com", "elearning.ucy.ac.cy",
    "moodle.ucy.ac.cy", "courses.cs.ucy.ac.cy"
}

ACADEMIC_NER_KEYWORDS = {"assignment", "deadline", "grade", "professor", "lecture"}


class RoutingEngine:

    def __init__(self, cluster_profiles: dict, user_config: dict):
        """
        cluster_profiles: {cluster_name → ClusterProfile} με:
            - centroid: np.array (BGE-M3 embedding)
            - known_domains: set
            - known_creators: set
            - known_agents: set
            - extension_histogram: dict
            - dominant_flacon_type: str
            - member_paths: list
            - dominant_ner_entities: set
        user_config: personalisation (weights, thresholds)
        """
        self.clusters = cluster_profiles
        self.cfg = user_config
        self.weights = {
            'sem':  user_config.get('w_sem',  0.35),
            'prov': user_config.get('w_prov', 0.25),
            'ext':  user_config.get('w_ext',  0.15),
            'beh':  user_config.get('w_beh',  0.10),
            'flac': user_config.get('w_flac', 0.10),
            'temp': user_config.get('w_temp', 0.05),
        }
        self.AUTO_THRESHOLD   = user_config.get('auto_threshold', 0.80)
        self.SUGGEST_THRESHOLD = user_config.get('suggest_threshold', 0.50)

    # ── PUBLIC ENTRY POINT ────────────────────────────────────────────────────

    def route(self, file_signals: dict, file_embedding=None) -> RoutingDecision:
        """
        file_signals: {
            'extension': str,
            'creator': str,
            'quarantine_agent': str,
            'quarantine_type': int,
            'domain': str,              # kMDItemWhereFroms domain
            'use_count': int,
            'days_since_last_use': float,
            'flacon_type': str,
            'co_modified_files': list,
            'ner_entities': set,
            'is_airdrop': bool,
            'airdrop_device': str,
        }
        file_embedding: np.array (BGE-M3) — может быть None для tier 0/1
        """

        # ── TIER 0: Deterministic rules ───────────────────────────────────────
        tier0 = self._tier0_rules(file_signals)
        if tier0:
            return tier0

        # ── TIER 1: Strong prior routing ─────────────────────────────────────
        tier1 = self._tier1_priors(file_signals, file_embedding)
        if tier1:
            return tier1

        # ── TIER 2: Embedding-based fusion ───────────────────────────────────
        return self._tier2_fusion(file_signals, file_embedding)

    # ── TIER 0 ────────────────────────────────────────────────────────────────

    def _tier0_rules(self, s: dict) -> Optional[RoutingDecision]:
        ext = s.get('extension', '').lower()
        creator = s.get('creator', '')

        if ext in INSTALLER_EXTENSIONS:
            return RoutingDecision("Installers", 1.0, False,
                                   f"installer package ({ext})", 0, [])
        if ext in APP_EXTENSIONS:
            return RoutingDecision("/Applications", 1.0, False,
                                   "application bundle", 0, [])
        if ext in DEV_EXTENSIONS or creator in DEV_CREATORS:
            return RoutingDecision("Development", 1.0, False,
                                   f"Xcode artifact (ext={ext}, creator={creator})", 0, [])
        if ext in CALENDAR_EXTENSIONS:
            return RoutingDecision("Calendar", 1.0, False, "calendar file", 0, [])
        if ext in CONTACT_EXTENSIONS:
            return RoutingDecision("Contacts", 1.0, False, "contact card", 0, [])

        return None

    # ── TIER 1 ────────────────────────────────────────────────────────────────

    def _tier1_priors(self, s: dict, emb) -> Optional[RoutingDecision]:
        domain  = s.get('domain', '')
        creator = s.get('creator', '')
        ner     = s.get('ner_entities', set())

        # Academic domain download
        if domain in ACADEMIC_DOMAINS:
            best_cluster, best_score = self._best_semantic_match(emb)
            if best_score > 0.40:
                return RoutingDecision(best_cluster, 0.87, False,
                                       f"academic domain ({domain})", 1, [])

        # Email attachment
        mail_creators = {"Mail", "Outlook", "Spark", "Airmail"}
        if creator in mail_creators or "mail" in domain:
            best_cluster, best_score = self._best_semantic_match(emb)
            if best_score > 0.45:
                return RoutingDecision(best_cluster, 0.88, False,
                                       f"email attachment via {creator or domain}", 1, [])

        # NER academic keywords
        if len(ner & ACADEMIC_NER_KEYWORDS) >= 2:
            best_cluster, best_score = self._best_semantic_match(emb)
            if best_score > 0.50:
                return RoutingDecision(best_cluster, 0.85, False,
                                       f"NER match: {ner & ACADEMIC_NER_KEYWORDS}", 1, [])

        return None

    # ── TIER 2 ────────────────────────────────────────────────────────────────

    def _tier2_fusion(self, s: dict, emb) -> RoutingDecision:
        scores = {}
        breakdown = {}

        for name, profile in self.clusters.items():
            sem  = self._cosine(emb, profile.get('centroid')) if emb is not None else 0.5
            prov = self._provenance_match(s, profile)
            ext  = self._extension_match(s, profile)
            beh  = self._behavioral_affinity(s)
            flac = self._flacon_alignment(s, profile)
            temp = self._temporal_cohesion(s, profile)

            w = self.weights
            total = (w['sem']  * sem +
                     w['prov'] * prov +
                     w['ext']  * ext +
                     w['beh']  * beh +
                     w['flac'] * flac +
                     w['temp'] * temp)

            scores[name] = total
            breakdown[name] = {
                'sem': sem, 'prov': prov, 'ext': ext,
                'beh': beh, 'flac': flac, 'temp': temp
            }

        ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        if not ranked:
            return RoutingDecision("Uncategorized", 0.0, True,
                                   "no clusters available", 2, [])

        best_name, best_score = ranked[0]
        alts = [r[0] for r in ranked[1:3]]  # top-2 alternatives

        # Bridge node detection: gap < 0.10 between top-2
        is_bridge = (len(ranked) >= 2 and
                     abs(ranked[0][1] - ranked[1][1]) < 0.10 and
                     best_score >= self.SUGGEST_THRESHOLD)

        if best_score >= self.AUTO_THRESHOLD:
            return RoutingDecision(
                best_name, best_score, False,
                f"fusion score {best_score:.2f}", 2, alts,
                is_bridge=False,
                signal_breakdown=breakdown.get(best_name, {})
            )
        elif best_score >= self.SUGGEST_THRESHOLD:
            return RoutingDecision(
                best_name, best_score, True,
                f"ambiguous (score {best_score:.2f})", 2, alts,
                is_bridge=is_bridge,
                signal_breakdown=breakdown.get(best_name, {})
            )
        else:
            return RoutingDecision(
                "New Cluster", best_score, True,
                f"low confidence ({best_score:.2f}) — new cluster candidate", 2, alts,
                signal_breakdown=breakdown.get(best_name, {})
            )

    # ── HELPER METHODS ────────────────────────────────────────────────────────

    def _cosine(self, a, b) -> float:
        if a is None or b is None:
            return 0.5
        import numpy as np
        denom = (np.linalg.norm(a) * np.linalg.norm(b))
        return float(np.dot(a, b) / denom) if denom > 0 else 0.0

    def _best_semantic_match(self, emb):
        if emb is None:
            return list(self.clusters.keys())[0], 0.0
        scored = {n: self._cosine(emb, p.get('centroid'))
                  for n, p in self.clusters.items()}
        best = max(scored, key=scored.get)
        return best, scored[best]

    def _provenance_match(self, s, profile) -> float:
        score = 0.0
        if s.get('domain') in profile.get('known_domains', set()):
            score += 0.5
        if s.get('creator') in profile.get('known_creators', set()):
            score += 0.3
        if s.get('quarantine_agent') in profile.get('known_agents', set()):
            score += 0.2
        return min(score, 1.0)

    def _extension_match(self, s, profile) -> float:
        ext = s.get('extension', '')
        hist = profile.get('extension_histogram', {})
        if not hist:
            return 0.5
        total = sum(hist.values())
        return hist.get(ext, 0) / total

    def _behavioral_affinity(self, s) -> float:
        use_count  = s.get('use_count', 0)
        days_since = s.get('days_since_last_use', 999)
        if use_count == 0:
            return 0.0
        recency = math.exp(-days_since / 30.0)
        usage   = min(use_count / 20.0, 1.0)
        return 0.6 * recency + 0.4 * usage

    def _flacon_alignment(self, s, profile) -> float:
        file_type    = s.get('flacon_type')
        cluster_type = profile.get('dominant_flacon_type')
        if file_type and cluster_type:
            return 1.0 if file_type == cluster_type else 0.0
        return 0.5

    def _temporal_cohesion(self, s, profile) -> float:
        comod   = set(s.get('co_modified_files', []))
        members = set(profile.get('member_paths', []))
        if not comod:
            return 0.0
        return min(len(comod & members) / len(comod), 1.0)
```

---

### Σύνοψη Ευρημάτων

**Η βασική αρχή** που προκύπτει από τη βιβλιογραφία και από το Dropbox Smart Move: για ένα personal system με μικρό dataset, **hand-tuned tiered rules με normalized weighted sum** νικάνε οποιαδήποτε learned fusion approach. Ο λόγος: δεν υπάρχουν αρκετά labeled examples για να train ένα fusion model χωρίς overfitting.

**Τα 3 key design decisions:**

1. **Tier 0 πρώτα, πάντα** — extension/creator rules είναι 100% accurate και free. Ποτέ μην ξοδεύεις embedding compute για ένα .dmg αρχείο.

2. **DenStream centroid ≈ Dropbox siblings** — ο centroid ήδη κωδικοποιεί το "τι υπάρχει σε αυτόν τον cluster". Το cosine similarity είναι ήδη "context-aware" χωρίς ξεχωριστό sibling lookup.

3. **Ambiguous = opportunity, όχι failure** — όταν score 0.50-0.80, το σύστημα ζητάει user feedback. Αυτή η feedback loop είναι ο μόνος τρόπος να calibrated αυτοί οι weights χωρίς external training data.

**Confidence threshold calibration:**
- `0.80` για auto-route βασίζεται στο Dropbox benchmark: high-confidence suggestions έχουν >90% acceptance. Θα πρέπει να calibrated με τα πρώτα 50-100 αρχεία.
- `0.50` για suggest/new-cluster split είναι conservative — αν βλέπεις πολλές "new cluster" decisions, κατέβασέ το σε `0.40`.

---

*Πηγές:*
- [Dropbox Smart Move: ML-powered file organization](https://dropbox.tech/machine-learning/smart-move-ml-ai-file-organization-automation)
- [On Comparing Early and Late Fusion Methods — ResearchGate](https://www.researchgate.net/publication/374301002_On_Comparing_Early_and_Late_Fusion_Methods)
- [Early vs Late Fusion in Multimodal Data Processing — GeeksforGeeks](https://www.geeksforgeeks.org/deep-learning/early-fusion-vs-late-fusion-in-multimodal-data-processing/)
- [Multimodal Data Fusion overview — ScienceDirect](https://www.sciencedirect.com/topics/computer-science/multimodal-data-fusion)
- [Classification Under Ambiguity: When Is Average-K Better Than Top-K?](https://arxiv.org/pdf/2112.08851)
- [A Framework for Optimizing Human-Machine Interaction in Classification Systems](https://arxiv.org/pdf/2601.05974)
- [Systems for ML-based classification of digital computer files using file metadata — USPTO](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/12287761)
- [Normalizing Cosine Similarity for Scoring — Medium](https://medium.com/@kswastik29/normalizing-cosine-similarity-for-scoring-a-practical-approach-8df5e3d41876)