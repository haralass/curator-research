## ΜΕΡΟΣ 35: SELF-TUNING ΓΙΑ PERSONAL USE
*(Έρευνα 2026-05-23)*

---

### Εισαγωγή: Γιατί το Personal Use αλλάζει τα πάντα

Το Curator δεν είναι SaaS. Δεν έχει πολλούς χρήστες. Δεν υπάρχει "average user".
Αυτό σημαίνει ότι **ό,τι θεωρείται anti-pattern σε ML γενικά, εδώ είναι feature**:

- Overfitting = εξαιρετικό (θέλουμε να "overfit" στον έναν χρήστη)
- Labeled data = κάθε user action (confirmation/rejection/manual move)
- Privacy concerns = μηδέν (είναι τα δικά του αρχεία)
- False positives = διορθώσιμα με undo, δεν χρειάζεται conservatism

Το αποτέλεσμα: ένα σύστημα που **μαθαίνει επιθετικά** από έναν και μόνο άνθρωπο.

---

## ΘΕΜΑ 1: Implicit vs Explicit Feedback — Τι μαθαίνουμε χωρίς να ρωτάμε

### 1.1 Το Πρόβλημα με το Explicit Feedback

Το κλασικό "thumbs up/thumbs down" δεν δουλεύει για file organization: ο χρήστης δεν
θα πατά "approve" σε κάθε αρχείο που το Curator τοποθέτησε σωστά. Ψάχνουμε signals
που υπάρχουν **ήδη** ως side-effects της φυσικής χρήσης.

### 1.2 Implicit Signals στο macOS — Catalog

**Signal 1: Manual move μετά την οργάνωση (FSEvents)**

```python
# Παρακολούθηση μέσω FSEvents (watchdog library ή native FSEventStream)
# Αν αρχείο X μετακινηθεί από το path που έβαλε το Curator
# μέσα σε window 48 ωρών → negative training example

class CuratorFSWatcher:
    def on_moved(self, event):
        src = event.src_path
        dst = event.dest_path
        
        if self.was_placed_by_curator(src):
            curator_decision = self.get_curator_decision(src)
            
            # Αυτό είναι gold-standard NEGATIVE label
            self.log_correction(
                file_id=curator_decision.file_id,
                original_cluster=curator_decision.cluster,
                corrected_path=dst,
                signal_type="manual_move",
                confidence=-1.0  # Σίγουρο negative
            )
```

Το FSEvents API (Apple Developer Documentation) παρέχει ακριβώς αυτή τη δυνατότητα:
παρακολούθηση filesystem operations σε real-time. Το `fswatch` (emcrisostomo/fswatch)
είναι cross-platform wrapper που χειρίζεται αυτόματα το FSEventStream.

**Signal 2: kMDItemUseCount + kMDItemUsedDates (Spotlight metadata)**

```bash
# Query: πόσες φορές ανοίχτηκε το αρχείο + πότε
mdls -name kMDItemUseCount \
     -name kMDItemLastUsedDate \
     -name kMDItemUsedDates \
     /path/to/file.pdf
```

Από forensic research (Forensic4cast, 2016): `kMDItemUsedDates` είναι array που
κρατά **κάθε ημέρα που ανοίχτηκε** το αρχείο, χωρίς να μπορεί να το αλλάξει
automated process — μόνο ο ίδιος ο χρήστης το ενεργοποιεί ανοίγοντας το αρχείο.

Interpretation:
- `kMDItemUseCount` αυξάνεται → **correct placement** (το βρήκε και το χρησιμοποίησε)
- `kMDItemLastUsedDate` > placement_date → **placement was findable**
- `kMDItemUsedDates` = empty μετά από 30 μέρες → ίσως **antilibrary candidate**

**Signal 3: Spotlight search ακολουθούμενο από file open**

Αν ο χρήστης κάνει Spotlight search για αρχείο που έχει ήδη οργανώσει το Curator,
αυτό σημαίνει: **"δεν το βρήκα στο expected location"** → wrong placement signal.

```python
# Indirect detection: συγκρίνουμε timestamps
# Αν (Spotlight index hit) + (file opened within 5 min of search-like pattern)
# και το αρχείο ΔΕΝ ανοίχτηκε από τον folder του → ο user το "βρήκε μέσω search"

def infer_spotlight_search(file_path, open_timestamp):
    """
    Heuristic: αν το kMDItemLastUsedDate ΔΕΝ αλλάζει μέσω browse
    αλλά μέσω mds (Spotlight daemon) process → ο user έψαξε
    
    Ελέγχουμε: ήταν το parent folder open στο Finder εκείνη την ώρα;
    Αν ΟΧΙ → πιθανόν Spotlight → negative placement signal
    """
    parent_was_open = self.finder_history.was_open(
        os.path.dirname(file_path), open_timestamp
    )
    if not parent_was_open:
        return PlacementSignal.NEGATIVE_SPOTLIGHT_SEARCH
```

**Signal 4: File moved to Trash μέσα σε X ημέρες**

```python
TRASH_SIGNAL_WINDOW_DAYS = 14

def check_trash_signal(self, file_id):
    file_meta = self.db.get(file_id)
    if file_meta.is_in_trash:
        days_since_placement = (now() - file_meta.curator_placement_date).days
        if days_since_placement <= TRASH_SIGNAL_WINDOW_DAYS:
            # Ο user το πέταξε αμέσως → should have been in antilibrary
            self.update_antilibrary_score(file_meta.content_cluster, +0.1)
```

### 1.3 LlamaFS Watch Mode — Τι Μαθαίνει

Από το iyaja/llama-fs (GitHub): στο watch mode, το LlamaFS εκκινεί ένα daemon
που παρακολουθεί το directory και **intercepts filesystem operations**, χρησιμοποιώντας
τα πιο πρόσφατα manual edits για να βελτιώσει τις προβλέψεις.

Practical example: αν ο χρήστης φτιάξει φάκελο "2023 Tax Docs" και αρχίσει να
βάζει 1-3 αρχεία μέσα, το LlamaFS αυτόματα συνεχίζει το pattern για τα υπόλοιπα.
**Αυτό είναι few-shot learning από παρατήρηση**, όχι explicit labeling.

### 1.4 hyperfield/ai-file-sorter — Learned Behavior DB

Από GitHub (hyperfield/ai-file-sorter): το σύστημα κρατά **δύο ξεχωριστά stores**:

1. **Categorization cache** — για faster reruns και consistency hints
2. **Learned-behavior database** (SQLite) — category decisions που ο user approved
   στο Review dialog, χρησιμοποιούνται ως hints για future runs

Αυτό είναι ακριβώς το **separation of concerns** που χρειάζεται το Curator:
- Cache = speed
- Learned DB = personalization

---

## ΘΕΜΑ 2: Few-Shot Personal Calibration

### 2.1 SetFit — Η Βάση Θεωρίας

Το SetFit (arXiv:2209.11055, Tunstall et al.) είναι η πιο relevant ακαδημαϊκή
δουλειά για τον χρήση μας:

- **Few-shot fine-tuning** Sentence Transformers με contrastive pairs
- **Χωρίς prompts ή verbalizers** — δουλεύει με embedding space
- **Competitive accuracy** με ελάχιστα labeled examples
- Multilingual υποστήριξη (σημαντικό: Ελληνικά filenames!)

Για το Curator: αντί να fine-tune ολόκληρο μοντέλο, χρησιμοποιούμε SetFit-style
approach στο **projection layer** που maps embeddings → cluster centroids.

### 2.2 Πόσα Examples Χρειάζονται;

Από human-in-the-loop research (Springer AI Review, 2022) και SetFit paper:

| Examples ανά category | Expected accuracy improvement |
|----------------------|-------------------------------|
| 3-5 pairs | +15-20% vs cold start |
| 8-10 pairs | +30-40% |
| 20+ pairs | +50-60%, diminishing returns |

**Conclusion για Curator**: με **5-8 confirmations ανά cluster**, το routing accuracy
βελτιώνεται σημαντικά. Δεν χρειαζόμαστε εκατοντάδες labels.

### 2.3 Concrete Confirmation Loop Algorithm

```python
class ConfirmationLoop:
    """
    Κάθε user confirmation είναι gold-standard labeled pair.
    Ενημερώνει DenStream centroid + projection layer + threshold.
    """
    
    RETRAIN_BUFFER_SIZE = 10  # Retrain projection layer κάθε 10 confirmations
    
    def on_user_confirms(self, file: FileRecord, cluster_id: str):
        """
        User λέει: "ναι, αυτό το αρχείο ανήκει στον folder C"
        """
        # 1. Update DenStream centroid για το cluster C
        #    Incremental centroid update: μ_new = (n * μ_old + x) / (n + 1)
        cluster = self.denstream.get_cluster(cluster_id)
        cluster.update_centroid(file.embedding)  # Fading factor εφαρμόζεται εσωτερικά
        
        # 2. Add (file_embedding, cluster_id) στο projection layer buffer
        self.projection_buffer.append(
            TrainingPair(embedding=file.embedding, label=cluster_id)
        )
        
        # 3. Αν buffer >= threshold → retrain projection layer (SetFit-style)
        if len(self.projection_buffer) >= self.RETRAIN_BUFFER_SIZE:
            self._retrain_projection_layer()
            self.projection_buffer.clear()
        
        # 4. Update routing confidence threshold για cluster C
        #    (βλ. Θέμα 3 για τον αλγόριθμο)
        self.threshold_manager.record_accept(cluster_id, confidence=file.routing_confidence)
        
        # 5. Log για future "auto-approve" decision
        self.approval_log.append(ApprovalEvent(
            file_id=file.id,
            cluster=cluster_id,
            confidence_at_decision=file.routing_confidence,
            timestamp=now()
        ))
        
        # 6. Check αν αυτό το pattern μπορεί να γίνει auto-approve
        self._evaluate_auto_approve_upgrade(cluster_id)
    
    def on_user_rejects(self, file: FileRecord, suggested_cluster: str, 
                         actual_cluster: str):
        """
        User λέει: "όχι, αυτό πάει εκεί" — negative + positive signal μαζί
        """
        # Negative example για suggested_cluster
        self.projection_buffer.append(
            TrainingPair(embedding=file.embedding, label=suggested_cluster, weight=-1.0)
        )
        # Positive example για actual_cluster
        self.projection_buffer.append(
            TrainingPair(embedding=file.embedding, label=actual_cluster, weight=+1.0)
        )
        self.threshold_manager.record_reject(suggested_cluster, 
                                              confidence=file.routing_confidence)
    
    def _retrain_projection_layer(self):
        """
        Contrastive fine-tuning του projection layer.
        Δημιουργούμε pairs: (same_cluster_A, same_cluster_B) → similar
                            (cluster_A, cluster_B) → dissimilar
        SetFit-style Siamese training.
        """
        contrastive_pairs = self._build_contrastive_pairs(self.projection_buffer)
        self.projection_layer.fine_tune(contrastive_pairs, epochs=3, lr=1e-4)
    
    def _evaluate_auto_approve_upgrade(self, cluster_id: str):
        """
        Αν τα τελευταία 15 confirmations για cluster C είχαν confidence > X
        → αυτόματη αναβάθμιση σε auto-approve για confidence > X
        """
        recent = self.approval_log.recent(cluster_id, n=15)
        if len(recent) >= 15:
            min_confidence = min(e.confidence_at_decision for e in recent)
            # Conservative: auto-approve μόνο αν min confidence > 0.70
            if min_confidence > 0.70:
                self.threshold_manager.upgrade_auto_approve(cluster_id, min_confidence)
```

### 2.4 Catastrophic Forgetting — Αντιμετώπιση

Από continual learning research (arXiv:2409.11329): το κύριο πρόβλημα είναι ότι
νέα training data "ξεχνά" παλιά clusters.

**Λύση για Curator**: Experience Replay micro-buffer

```python
class ExperienceReplayBuffer:
    """
    Κρατά ένα representative sample από παλιές αποφάσεις.
    Κάθε retrain του projection layer περιλαμβάνει ΠΑΝΤΑ αυτά τα παλιά pairs.
    """
    MAX_PER_CLUSTER = 20  # 20 παλιά examples ανά cluster
    
    def get_replay_pairs(self) -> List[TrainingPair]:
        """
        Επιστρέφει balanced sample: ίσο ποσοστό από κάθε cluster.
        Prioritizes: πιο παλιά examples (για να μην ξεχαστούν).
        """
        pairs = []
        for cluster_id, examples in self.cluster_examples.items():
            # Κράτα τα πιο παλιά (FIFO με priority)
            selected = examples[:self.MAX_PER_CLUSTER]
            pairs.extend(selected)
        return pairs
```

---

## ΘΕΜΑ 3: Adaptive Thresholds — Thresholds που Αλλάζουν

### 3.1 Γιατί τα Hardcoded Thresholds Αποτυγχάνουν

Σήμερα: `auto_route > 0.80`, `suggest 0.50-0.80`, `new_cluster < 0.50`.

Αυτά είναι αυθαίρετα. Ο Haralampos μπορεί να είναι "trusting user" (αποδέχεται
0.65+ suggestions πάντα) ή "cautious user" (θέλει 0.90+ για auto-route).

**Bayesian framing**: θεωρούμε το threshold ως prior, το user behavior ως evidence,
και κάνουμε posterior update.

Από speech recognition research (USPTO patent 8239203): adaptive confidence thresholds
για speech recognition χρησιμοποιούν ακριβώς αυτή τη λογική — αν τα results με
confidence > X γίνονται accepted χωρίς correction, το X κατεβαίνει.

### 3.2 Concrete Formula

```python
def update_threshold(
    current_threshold: float,
    recent_accept_rate: float,
    window: int = 20,
    step: float = 0.02,
    min_threshold: float = 0.45,
    max_threshold: float = 0.90
) -> float:
    """
    Adaptive threshold based on user acceptance rate over sliding window.
    
    Args:
        current_threshold: τρέχον threshold (π.χ. 0.80 για auto-route)
        recent_accept_rate: ποσοστό αποδοχής στα τελευταία `window` suggestions
        window: sliding window μεγέθους (default: 20 decisions)
        step: βήμα αλλαγής threshold (default: 0.02)
        min_threshold: floor για aggressive automation (δεν πέφτει κάτω)
        max_threshold: ceiling για conservative mode
    
    Returns:
        Νέο threshold
    
    Λογική:
        accept_rate > 0.90 → ο user εμπιστεύεται → κατέβασε threshold (πιο aggressive)
        accept_rate < 0.70 → ο user διορθώνει συχνά → ανέβασε threshold (πιο conservative)
        0.70 <= accept_rate <= 0.90 → ισορροπία → μη αλλάξεις τίποτα
    """
    if recent_accept_rate > 0.90:
        new_threshold = current_threshold - step
    elif recent_accept_rate < 0.70:
        new_threshold = current_threshold + step
    else:
        return current_threshold  # No change needed
    
    return max(min_threshold, min(max_threshold, new_threshold))


class AdaptiveThresholdManager:
    """
    Ξεχωριστά thresholds ανά cluster — κάθε cluster αναπτύσσει διαφορετική "εμπιστοσύνη"
    """
    
    def __init__(self):
        # Global defaults
        self.auto_route_threshold = 0.80
        self.suggest_threshold = 0.50
        
        # Per-cluster overrides (αναπτύσσονται από experience)
        self.cluster_thresholds: Dict[str, float] = {}
        
        # Sliding window: τελευταίες N αποφάσεις ανά cluster
        self.decision_windows: Dict[str, deque] = defaultdict(lambda: deque(maxlen=20))
    
    def record_accept(self, cluster_id: str, confidence: float):
        self.decision_windows[cluster_id].append(("accept", confidence))
        self._maybe_update_threshold(cluster_id)
    
    def record_reject(self, cluster_id: str, confidence: float):
        self.decision_windows[cluster_id].append(("reject", confidence))
        self._maybe_update_threshold(cluster_id)
    
    def _maybe_update_threshold(self, cluster_id: str):
        window = self.decision_windows[cluster_id]
        if len(window) < 10:
            return  # Δεν έχουμε αρκετά data ακόμα
        
        accept_rate = sum(1 for d, _ in window if d == "accept") / len(window)
        current = self.cluster_thresholds.get(cluster_id, self.auto_route_threshold)
        
        new_threshold = update_threshold(current, accept_rate)
        self.cluster_thresholds[cluster_id] = new_threshold
    
    def get_threshold(self, cluster_id: str) -> float:
        return self.cluster_thresholds.get(cluster_id, self.auto_route_threshold)
    
    def upgrade_auto_approve(self, cluster_id: str, confidence_floor: float):
        """
        Αναβάθμιση cluster σε auto-approve mode για confidence >= floor
        """
        self.cluster_thresholds[cluster_id] = confidence_floor - 0.05
        # Logging για transparency
        self.auto_approve_clusters[cluster_id] = {
            "upgraded_at": now(),
            "confidence_floor": confidence_floor
        }
```

### 3.3 Calibration of Confidence — Γιατί Έχει Σημασία

Από ConiferAI Glossary: confidence threshold calibration = systematic adjustment
των scoring thresholds ώστε να αντιστοιχούν στην πραγματική accuracy.

**Το πρόβλημα**: ένα μοντέλο που λέει "0.85 confidence" μπορεί να έχει ΠΡΑΓΜΑΤΙΚΗ
accuracy 70%. Δεν εμπιστευόμαστε τυφλά τα confidence scores.

**Λύση**: temperature scaling ή Platt scaling ως post-hoc calibration μετά από κάθε
batch retraining του projection layer.

```python
def calibrate_confidence(raw_score: float, temperature: float = 1.0) -> float:
    """
    Temperature scaling για calibration.
    temperature > 1.0 → soften probabilities (πιο conservative)
    temperature < 1.0 → sharpen probabilities (πιο aggressive)
    Το temperature ρυθμίζεται από held-out validation (approved decisions)
    """
    import math
    logit = math.log(raw_score / (1 - raw_score + 1e-10))
    calibrated_logit = logit / temperature
    return 1 / (1 + math.exp(-calibrated_logit))
```

---

## ΘΕΜΑ 4: Personal Use = Aggressive Automation

### 4.1 Τι Αλλάζει Conceptually

Σε enterprise ML:
- False positive cost HIGH (wrong routing → angry employee)
- Conservative thresholds necessary
- Privacy constraints limit signals
- Generalization required (10,000 users)

Στο Curator (single user):
- False positive cost = 2 seconds (undo)
- Aggressive thresholds = desired
- ALL signals available (είναι τα δικά του δεδομένα)
- Overfitting = FEATURE

### 4.2 Personal Pattern Registry

```python
# Hardcoded personal patterns + learned patterns hybrid
# Hardcoded: ξέρουμε a priori για τον συγκεκριμένο χρήστη
# Learned: το σύστημα ανακαλύπτει νέα patterns αυτόματα

PERSONAL_PATTERNS: Dict[str, PersonalPattern] = {
    "university": PersonalPattern(
        domains=["ucy.ac.cy", "cs.ucy.ac.cy", "students.ucy.ac.cy"],
        keywords=["EPL", "UCY", "assignment", "lecture", "lab", "exercise",
                  "midterm", "final", "semester", "ECTS"],
        filename_patterns=[r"EPL\d{3}", r"cs\d{3}", r"assignment_\d+"],
        sender_patterns=["@ucy.ac.cy"],
        confidence_boost=0.15,  # +15% αν match → πολύ πιο πιθανό university
        auto_approve_threshold=0.65  # Lower threshold for known domains
    ),
    
    "personal_mobile": PersonalPattern(
        airdrop_sources=["Haralampos's iPhone", "Haralampos's iPad"],
        # Φωτογραφίες/videos από personal devices → personal folder
        mime_prefixes=["image/", "video/"],
        confidence_boost=0.20,
        auto_approve_threshold=0.60
    ),
    
    "developer": PersonalPattern(
        domains=["github.com", "stackoverflow.com", "npmjs.com", "pypi.org",
                 "crates.io", "pkg.go.dev"],
        filename_patterns=[r"\.py$", r"\.rs$", r"\.ts$", r"\.go$", r"Cargo\.toml",
                          r"package\.json", r"requirements\.txt"],
        confidence_boost=0.12,
        auto_approve_threshold=0.68
    ),
    
    "financial": PersonalPattern(
        keywords=["invoice", "receipt", "bank", "statement", "tax", "ΑΦΜ",
                 "τιμολόγιο", "απόδειξη", "φορολογική"],
        filename_patterns=[r"invoice_\d+", r"receipt_\d+", r"\d{4}_tax"],
        confidence_boost=0.18,
        auto_approve_threshold=0.62
    )
}


class PersonalPatternEngine:
    """
    Εφαρμόζει personal patterns ως confidence boosts.
    Hybrid: static (hardcoded) + dynamic (learned από history).
    """
    
    def apply_patterns(self, file: FileRecord, base_confidence: float) -> float:
        boosted = base_confidence
        
        for pattern_name, pattern in PERSONAL_PATTERNS.items():
            if self._file_matches_pattern(file, pattern):
                boosted = min(1.0, boosted + pattern.confidence_boost)
        
        # Dynamic patterns: learned από approval history
        for learned_pattern in self.learned_patterns:
            if self._file_matches_learned(file, learned_pattern):
                boosted = min(1.0, boosted + learned_pattern.boost)
        
        return boosted
    
    def learn_new_pattern(self, file: FileRecord, cluster_id: str, 
                           n_confirmations: int = 5):
        """
        Μετά από N confirmations με κοινό attribute (π.χ. sender domain)
        → auto-discover νέο personal pattern
        """
        if n_confirmations >= 5:
            common_attributes = self._extract_common_attributes(
                self.confirmation_history[cluster_id]
            )
            if common_attributes:
                self.learned_patterns.append(
                    LearnedPattern(attributes=common_attributes, 
                                   cluster=cluster_id, boost=0.10)
                )
```

### 4.3 Signals χωρίς Privacy Concerns

Στο single-user context χρησιμοποιούμε **όλα** τα διαθέσιμα signals:

```python
AVAILABLE_SIGNALS = {
    # Spotlight metadata
    "kMDItemUsedDates":     weight=0.15,  # Πότε χρησιμοποιήθηκε
    "kMDItemUseCount":      weight=0.10,  # Πόσες φορές
    "kMDItemWhereFroms":    weight=0.20,  # Από πού κατέβηκε
    "kMDItemAuthors":       weight=0.08,  # Ποιος το έφτιαξε
    
    # QuarantineEventsV2.db
    "quarantine_agent":     weight=0.15,  # Ποια app το κατέβασε
    "quarantine_sender":    weight=0.18,  # Email sender
    "quarantine_origin_url": weight=0.12, # URL προέλευσης
    
    # FSEvents history
    "manual_moves_history": weight=0.25,  # Τι είχε κάνει ο user παλιά
    
    # Calendar correlation
    "calendar_proximity":   weight=0.08,  # Ανέβηκε κοντά σε event;
}
```

---

## ΘΕΜΑ 5: "Run 1 vs Run N" Evolution — State Machine

### 5.1 Πίνακας Εξέλιξης

| Run | Κατάσταση | Auto-approve rate | Clusters | Thresholds | Behavior |
|-----|-----------|------------------|----------|------------|----------|
| 1 | Cold Start | ~25-30% | HDBSCAN scratch | Conservative (0.80+) | Ρωτά συχνά |
| 3 | Early Learning | ~45-50% | Refined centroids | 0.75+ | Αρχίζει patterns |
| 5 | Calibrating | ~60-65% | Stable cores | 0.70+ | Fewer questions |
| 10 | Established | ~78-82% | Known + stable | 0.65+ per-cluster | Mostly autonomous |
| 20+ | Mature | ~88-92% | Expert-level | Dynamic per-cluster | Near-autonomous |

### 5.2 Concrete State Machine

```python
from enum import Enum

class CuratorLearningState(Enum):
    COLD_START = "cold_start"          # 0-2 runs, < 20 confirmations
    EARLY_LEARNING = "early_learning"  # 3-5 runs, 20-60 confirmations
    CALIBRATING = "calibrating"        # 6-10 runs, 60-150 confirmations
    ESTABLISHED = "established"        # 10+ runs, 150-400 confirmations
    MATURE = "mature"                  # 20+ runs, 400+ confirmations


class CuratorEvolutionEngine:
    """
    State machine που ελέγχει πόσο aggressive είναι το σύστημα.
    Transitions βασίζονται σε confirmations, not just run count.
    """
    
    STATE_TRANSITIONS = {
        CuratorLearningState.COLD_START: {
            "min_confirmations": 0,
            "auto_route_threshold": 0.85,  # Very conservative
            "suggest_threshold": 0.60,
            "ask_for_every_new_cluster": True,
            "enable_personal_patterns": False,  # Learn first, apply later
            "replay_buffer_weight": 0.0
        },
        CuratorLearningState.EARLY_LEARNING: {
            "min_confirmations": 20,
            "auto_route_threshold": 0.78,
            "suggest_threshold": 0.55,
            "ask_for_every_new_cluster": True,
            "enable_personal_patterns": True,
            "replay_buffer_weight": 0.3
        },
        CuratorLearningState.CALIBRATING: {
            "min_confirmations": 60,
            "auto_route_threshold": 0.72,
            "suggest_threshold": 0.50,
            "ask_for_every_new_cluster": False,  # Auto-cluster αν similarity > 0.65
            "enable_personal_patterns": True,
            "replay_buffer_weight": 0.5
        },
        CuratorLearningState.ESTABLISHED: {
            "min_confirmations": 150,
            "auto_route_threshold": 0.65,  # Per-cluster adaptive
            "suggest_threshold": 0.45,
            "ask_for_every_new_cluster": False,
            "enable_personal_patterns": True,
            "replay_buffer_weight": 0.7,
            "enable_auto_approve_upgrade": True  # Clusters μπορούν να αναβαθμιστούν
        },
        CuratorLearningState.MATURE: {
            "min_confirmations": 400,
            "auto_route_threshold": 0.60,  # Highly personalized
            "suggest_threshold": 0.40,
            "ask_for_every_new_cluster": False,
            "enable_personal_patterns": True,
            "replay_buffer_weight": 0.9,
            "enable_auto_approve_upgrade": True,
            "enable_learned_pattern_discovery": True  # Ανακαλύπτει νέα patterns μόνο του
        }
    }
    
    def get_current_state(self) -> CuratorLearningState:
        total_confirmations = self.db.count_all_confirmations()
        
        for state in reversed(list(CuratorLearningState)):
            if total_confirmations >= self.STATE_TRANSITIONS[state]["min_confirmations"]:
                return state
        
        return CuratorLearningState.COLD_START
    
    def get_config(self) -> Dict:
        return self.STATE_TRANSITIONS[self.get_current_state()]
```

### 5.3 Visualization της Εξέλιξης

```
Cold Start            Early Learning         Calibrating
────────────────────  ─────────────────────  ───────────────────
"Curator routing     "Curator βλέπει        "Αρχίζει να ξέρει:
 όλα → σε εσένα     patterns: EPL → uni,    - αυτό πάει εδώ
 confirm/reject"     AirDrop → personal"     - αυτό μάλλον εκεί
                                             - αυτό ρωτάω"
Threshold: 0.85      Threshold: 0.78         Threshold: 0.72
Auto: 25%            Auto: 45%               Auto: 60%

Established          Mature
────────────────────  ─────────────────────
"Ξέρει σχεδόν        "Σχεδόν self-driving:
 τα πάντα.           Ρωτά μόνο για truly
 Ρωτά για edge       ambiguous files.
 cases μόνο."        Discoveries μόνο του."
Threshold: 0.65      Threshold: 0.60
Auto: 80%            Auto: 90%
```

---

## Συμπεράσματα ΜΕΡΟΣ 35

1. **Implicit signals > explicit prompts** για personal use. Το macOS παρέχει
   FSEvents, kMDItemUsedDates, kMDItemUseCount — ό,τι χρειάζεται για silent learning
   χωρίς user burden. Manual move = best negative label. Use count growth = best positive label.

2. **SetFit-style contrastive fine-tuning** του projection layer με **5-10 confirmations**
   αρκεί για significant improvement. Ο Experience Replay buffer αποτρέπει catastrophic
   forgetting κρατώντας representative παλιά examples σε κάθε retrain.

3. **Adaptive thresholds per-cluster** είναι ανώτεροι από global thresholds. Ο αλγόριθμος
   sliding window (window=20, step=0.02) είναι simple αλλά effective: αν ο user
   αποδέχεται > 90% → κατέβα, αν < 70% → ανέβα.

4. **Personal Pattern Registry** (hardcoded + learned hybrid) δίνει immediate
   confidence boost για γνωστά patterns (UCY domains, AirDrop from iPhone, GitHub URLs)
   χωρίς να περιμένουμε να "μάθει" το σύστημα από scratch.

5. **State machine εξέλιξης** (Cold Start → Mature) επιτρέπει στο Curator να είναι
   conservative αρχικά και να γίνεται progressively more aggressive καθώς
   συσσωρεύει confirmations. Target: 80%+ auto-approve rate στα 150+ confirmations.

6. **Η κύρια insight**: σε single-user personal system, το distinction μεταξύ
   "training data" και "production data" εξαφανίζεται. Κάθε user interaction
   ΕΙΝΑΙκαι training και inference ταυτόχρονα.

---

*Πηγές: arXiv:2209.11055 (SetFit), arXiv:2306.01277 (Beyond Active Learning),
arXiv:2409.11329 (Catastrophic Forgetting Self-Distillation),
Forensic4cast kMDItemUsedDates, Apple Developer FSEvents Documentation,
github.com/iyaja/llama-fs (watch mode), github.com/hyperfield/ai-file-sorter (learned DB),
NCBI PMC ai-assistant correction detection, USPTO patent 8239203 (adaptive speech thresholds),
Springer AI Review human-in-the-loop 2022, DenStream/River online clustering.*