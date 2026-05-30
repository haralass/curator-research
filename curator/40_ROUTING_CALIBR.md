## ΜΕΡΟΣ 40: ROUTING CALIBRATION & COLD START OPTIMIZATION
*(Έρευνα 2026-05-23)*

---

### ΘΕΜΑ 1: Routing Weights Calibration — Πώς βρίσκεις τα σωστά weights χωρίς labeled data

#### Το πρόβλημα

Το τρέχον routing formula του Curator:

```
score = 0.35·semantic + 0.25·provenance + 0.15·extension + 0.10·behavioral + 0.10·falcon + 0.05·temporal
```

Αυτά τα weights είναι αυθαίρετα. Δεν υπάρχει εγγύηση ότι το 0.35 για semantic είναι σωστό — μπορεί η provenance να είναι πιο predictive για τον συγκεκριμένο χρήστη. Χωρίς ground truth labels, πώς το ξέρουμε;

#### Τρεις στρατηγικές από την έρευνα

**Α. Online Logistic Regression (Recommended)**

Κάθε φορά που ο χρήστης επιβεβαιώνει ή απορρίπτει μια πρόταση, έχουμε ένα labeled example:
- Features: `[semantic_score, provenance_score, extension_score, behavioral_score, falcon_score, temporal_score]`
- Label: `1` (accepted) ή `0` (rejected)

Από 20+ τέτοια examples, ένα simple logistic regression μαθαίνει ποια signals είναι πραγματικά predictive. Το "Efficient Improper Learning for Online Logistic Regression" (arXiv:2003.08109) αποδεικνύει near-optimal regret bounds για online logistic regression — κατάλληλο για streaming user feedback.

**Β. Multi-Armed Bandit (Thompson Sampling)**

Treat κάθε weight configuration ως "arm". Παρατήρησε reward (1/0). Update posteriors. Η έρευνα "Generalized Thompson Sampling for Contextual Bandits" (Microsoft Research) δείχνει ότι Thompson Sampling converges γρηγορότερα από UCB για small action spaces. Μειονέκτημα: ο weight space είναι continuous → χρειάζεται discretization.

**Γ. Ablation Approach (Πιο απλό, για early stage)**

Ψάξε ποιο signal μόνο του προβλέπει καλύτερα τα accepted/rejected decisions:
- Route με μόνο semantic → μέτρα accuracy
- Route με μόνο provenance → μέτρα accuracy
- Συγκρίνεσε

Η Capital One XAI research επιβεβαιώνει: ablation studies που αφαιρούν ένα feature τη φορά αποκαλύπτουν unique predictive value. Αν η αφαίρεση του semantic δεν αλλάζει accuracy → δεν αξίζει 0.35.

#### Υλοποίηση: Weight Self-Calibration Algorithm

```python
import numpy as np
from collections import deque
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class RoutingDecision:
    """Ένα routing event με τα signals και το outcome."""
    signals: dict[str, float]   # {"semantic": 0.82, "provenance": 0.61, ...}
    accepted: bool
    file_path: str
    timestamp: float


class RoutingWeightCalibrator:
    """
    Παρακολουθεί ποια signals είναι predictive του user acceptance.
    Μετά από MIN_SAMPLES confirmations → update weights με online logistic regression.

    Αρχιτεκτονική:
    - Συλλέγει (signals, accepted) pairs
    - Κάθε UPDATE_INTERVAL decisions → fit logistic regression
    - Εξάγει normalized coefficients ως νέα weights
    - Κρατά exponential moving average για stability (αποφυγή overfit σε noise)
    """

    SIGNAL_NAMES = ["semantic", "provenance", "extension", "behavioral", "falcon", "temporal"]

    # Defaults από το current formula
    DEFAULT_WEIGHTS = {
        "semantic": 0.35,
        "provenance": 0.25,
        "extension": 0.15,
        "behavioral": 0.10,
        "falcon": 0.10,
        "temporal": 0.05,
    }

    def __init__(
        self,
        min_samples: int = 20,
        update_interval: int = 10,
        smoothing_alpha: float = 0.3,   # EMA: 0=never update, 1=always replace
        window_size: int = 200,          # Sliding window — ignore stale decisions
    ):
        self.min_samples = min_samples
        self.update_interval = update_interval
        self.smoothing_alpha = smoothing_alpha

        self._history: deque[RoutingDecision] = deque(maxlen=window_size)
        self._current_weights = dict(self.DEFAULT_WEIGHTS)
        self._decisions_since_update = 0
        self._total_decisions = 0
        self._calibration_log: list[dict] = []  # Για debugging

    def record_outcome(self, signals: dict[str, float], accepted: bool, file_path: str = "") -> None:
        """Καταγράφει ένα routing outcome. Trigger update αν χρειάζεται."""
        import time
        decision = RoutingDecision(
            signals=signals,
            accepted=accepted,
            file_path=file_path,
            timestamp=time.time(),
        )
        self._history.append(decision)
        self._total_decisions += 1
        self._decisions_since_update += 1

        # Update weights αν έχουμε αρκετά samples ΚΑΙ έχουν περάσει αρκετά νέα decisions
        if (
            len(self._history) >= self.min_samples
            and self._decisions_since_update >= self.update_interval
        ):
            self._update_weights()
            self._decisions_since_update = 0

    def get_current_weights(self) -> dict[str, float]:
        """Επιστρέφει τα τρέχοντα calibrated weights (normalized, sum=1)."""
        return dict(self._current_weights)

    def compute_score(self, signals: dict[str, float]) -> float:
        """Υπολογίζει routing score με τα calibrated weights."""
        w = self._current_weights
        return sum(w.get(k, 0.0) * v for k, v in signals.items())

    def calibration_status(self) -> dict:
        """Πόσο confident είμαστε στα weights."""
        n = len(self._history)
        accepted = sum(1 for d in self._history if d.accepted)
        return {
            "total_decisions": self._total_decisions,
            "window_decisions": n,
            "acceptance_rate": accepted / max(n, 1),
            "calibrated": n >= self.min_samples,
            "weights": self._current_weights,
            "updates_performed": len(self._calibration_log),
        }

    def _update_weights(self) -> None:
        """
        Fits online logistic regression στα (signals, accepted) pairs.
        Coefficients → normalize → EMA blend με current weights.
        """
        from sklearn.linear_model import LogisticRegression
        from sklearn.preprocessing import StandardScaler

        decisions = list(self._history)
        X = np.array([[d.signals.get(k, 0.0) for k in self.SIGNAL_NAMES] for d in decisions])
        y = np.array([int(d.accepted) for d in decisions])

        # Skip αν δεν έχουμε και τις δύο κλάσεις (όλα accepted ή όλα rejected)
        if len(set(y)) < 2:
            return

        # Standardize για να είναι συγκρίσιμα τα coefficients
        scaler = StandardScaler()
        X_scaled = scaler.fit_transform(X)

        clf = LogisticRegression(max_iter=200, C=1.0, solver="lbfgs")
        clf.fit(X_scaled, y)

        # Coefficients → absolute value (θέλουμε predictive power, όχι direction)
        # Direction: αν negative coefficient → signal αντι-correlated με acceptance
        coefs = clf.coef_[0]  # shape: (n_signals,)
        abs_coefs = np.abs(coefs)

        # Clip τα πολύ μικρά coefficients — prevent degeneracy
        abs_coefs = np.maximum(abs_coefs, 0.01)

        # Normalize → sum to 1
        new_weights_arr = abs_coefs / abs_coefs.sum()
        new_weights = {k: float(new_weights_arr[i]) for i, k in enumerate(self.SIGNAL_NAMES)}

        # EMA blending: μην πηδάμε άμεσα στα νέα weights
        for k in self.SIGNAL_NAMES:
            self._current_weights[k] = (
                self.smoothing_alpha * new_weights[k]
                + (1 - self.smoothing_alpha) * self._current_weights[k]
            )

        # Re-normalize μετά το blending
        total = sum(self._current_weights.values())
        for k in self._current_weights:
            self._current_weights[k] /= total

        self._calibration_log.append({
            "raw_weights": new_weights,
            "blended_weights": dict(self._current_weights),
            "n_samples": len(decisions),
        })

    def run_ablation_analysis(self) -> dict[str, float]:
        """
        Ablation: για κάθε signal, μετρά τι χάνεται αν το αφαιρέσουμε.
        Επιστρέφει {signal_name: accuracy_drop}.
        Υψηλό accuracy_drop → σημαντικό signal.
        """
        from sklearn.linear_model import LogisticRegression
        from sklearn.model_selection import cross_val_score

        if len(self._history) < self.min_samples:
            return {}

        decisions = list(self._history)
        X = np.array([[d.signals.get(k, 0.0) for k in self.SIGNAL_NAMES] for d in decisions])
        y = np.array([int(d.accepted) for d in decisions])

        if len(set(y)) < 2:
            return {}

        # Baseline accuracy με όλα τα features
        clf_full = LogisticRegression(max_iter=200, C=1.0)
        baseline_acc = cross_val_score(clf_full, X, y, cv=min(5, len(y) // 5)).mean()

        importance = {}
        for i, signal in enumerate(self.SIGNAL_NAMES):
            # Αφαίρεση του signal i
            X_ablated = np.delete(X, i, axis=1)
            clf_ablated = LogisticRegression(max_iter=200, C=1.0)
            ablated_acc = cross_val_score(clf_ablated, X_ablated, y, cv=min(5, len(y) // 5)).mean()
            importance[signal] = baseline_acc - ablated_acc  # Accuracy drop

        return importance
```

**Βασικό insight:** Το logistic regression δεν χρειάζεται εξωτερικά labels — τα confirmed/rejected decisions ΤΟΥ ΙΔΙΟΥ ΤΟΥ ΧΡΗΣΤΗ είναι τα labels. 20 αποφάσεις αρκούν για πρώτη calibration. Η EMA με `smoothing_alpha=0.3` αποτρέπει abrupt changes όταν ο χρήστης κάνει μια series από atypical decisions.

---

### ΘΕΜΑ 2: Cold Start — Φτάνοντας "Mature" πιο γρήγορα

#### Το πρόβλημα με το τρέχον threshold

Το Curator απαιτεί 400+ confirmations για "Mature" κατάσταση. Για έναν casual χρήστη που επιβεβαιώνει 3-4 αρχεία/εβδομάδα → **2+ χρόνια** για να φτάσει Mature. Αυτό καθιστά το feature practically useless.

#### Τι λέει η έρευνα για cold start

Η έρευνα "A two-stage clustering-based cold-start method for active learning" (IOS Press, IDA journal) τεκμηριώνει ότι:
1. **Clustering-first**: Κάνε cluster τα unlabeled data
2. **Centroid selection**: Από κάθε cluster, επίλεξε το example πιο κοντά στο centroid ως "representative"
3. **Batch labeling**: Ζήτα label μόνο για τα representatives

Αυτή η προσέγγιση μειώνει τον αριθμό των απαιτούμενων labels κατά 60-80% σε σχέση με random sampling, ενώ επιτυγχάνει comparable performance.

Επιπλέον, η "Foundation Model Makes Clustering A Better Initialization For Cold-Start Active Learning" (arXiv:2402.02561) δείχνει ότι embedding από pre-trained models (π.χ. sentence-transformers) + clustering converges ακόμα πιο γρήγορα.

#### Τρία shortcuts για το Curator

**Shortcut A: Metadata-Seeded Clustering**

Αντί να ξεκινάς HDBSCAN από scratch, χρησιμοποίησε macOS Spotlight metadata ως weak initial labels:
- `kMDItemWhereFroms`: αρχεία από τον ίδιο domain (π.χ. `github.com`) → πιθανόν ίδια κατηγορία
- `kMDItemCreator`: αρχεία που δημιουργήθηκαν από το ίδιο app → strong signal
- Extension groups: `{.py, .js, .ts}` = code, `{.pdf, .docx}` = documents

Αυτό είναι weak supervision — δεν είναι ground truth, αλλά μειώνει τον space τον οποίον πρέπει να εξερευνήσεις. Το Snorkel framework (snorkel.ai) τεκμηριώνει: "multiple weak labeling sources → labeling pipeline → usable training signal".

**Shortcut B: Batch Confirmation Session**

Αντί 1 αρχείο τη φορά, παρουσίασε **ένα representative αρχείο ανά cluster** και ζήτα 1 confirmation. Αν έχεις 15 clusters → 15 questions → καλύπτεις ολόκληρο τον space.

Από την έρευνα: "Clustering can identify a representative instance that is generally closest to the centroid of each cluster." Centroid = average embedding vector. Representative = αρχείο με embedding πλησιέστερο στο centroid.

**Shortcut C: Existing Folders as Weak Labels**

Αν `~/Documents/EPL326/` ήδη υπάρχει → αρχεία εκεί = ground truth για cluster "EPL326". Το Dropbox ML team ("Smart Move: ML-powered file organization") χρησιμοποιεί ακριβώς αυτή την προσέγγιση: existing organization = training signal, χωρίς explicit labeling.

Αυτό είναι το ισχυρότερο shortcut: αν ο χρήστης έχει ήδη organized 200 αρχεία σε folders → αυτά είναι δωρεάν 200 labeled examples.

#### Υλοποίηση: ColdStartAccelerator

```python
import os
import subprocess
from pathlib import Path
from dataclasses import dataclass
from typing import Optional
import numpy as np


@dataclass
class ClusterRepresentative:
    """Ένα representative αρχείο για κάθε cluster."""
    cluster_id: int
    file_path: str
    centroid_distance: float   # Πόσο κοντά στο centroid (χαμηλό = πιο representative)
    weak_label: Optional[str]  # Από metadata ή folder structure


class ColdStartAccelerator:
    """
    Επιταχύνει το cold start του Curator από ~400 confirmations σε ~30-80.

    Στρατηγική:
    1. Εξάγει weak labels από macOS metadata + existing folders
    2. Κάνει metadata-seeded HDBSCAN clustering
    3. Επιλέγει centroid representatives ανά cluster
    4. Παρουσιάζει batch confirmation session (1 per cluster)
    """

    def __init__(self, embedder, clusterer, files_dir: str = str(Path.home())):
        self.embedder = embedder        # Sentence transformer
        self.clusterer = clusterer      # HDBSCAN instance
        self.files_dir = files_dir
        self._weak_labels: dict[str, str] = {}  # file_path → weak_label

    # ── Shortcut A: macOS Metadata Weak Labels ──────────────────────────────

    def extract_metadata_weak_labels(self, file_paths: list[str]) -> dict[str, str]:
        """
        Εξάγει weak labels από macOS Spotlight metadata.
        Returns: {file_path: weak_label_hint}
        """
        weak_labels = {}
        for fp in file_paths:
            label = self._infer_label_from_metadata(fp)
            if label:
                weak_labels[fp] = label
        self._weak_labels.update(weak_labels)
        return weak_labels

    def _infer_label_from_metadata(self, file_path: str) -> Optional[str]:
        """Χρησιμοποιεί mdls για να πάρει Spotlight attributes."""
        try:
            result = subprocess.run(
                ["mdls", "-name", "kMDItemWhereFroms", "-name", "kMDItemCreator",
                 "-name", "kMDItemKind", file_path],
                capture_output=True, text=True, timeout=2
            )
            output = result.stdout

            # kMDItemCreator → app που δημιούργησε το αρχείο
            if "Xcode" in output:
                return "code"
            if "Safari" in output or "Chrome" in output:
                return "web_download"
            if "Microsoft Word" in output or "Pages" in output:
                return "document"
            if "Zoom" in output or "Teams" in output:
                return "meeting"

            # kMDItemWhereFroms → domain origin
            if "github.com" in output:
                return "code_repository"
            if "drive.google.com" in output or "dropbox.com" in output:
                return "cloud_sync"
            if "arxiv.org" in output or "scholar.google" in output:
                return "research_paper"

        except (subprocess.TimeoutExpired, FileNotFoundError):
            pass

        # Fallback: extension-based
        ext = Path(file_path).suffix.lower()
        extension_map = {
            ".py": "code", ".js": "code", ".ts": "code", ".swift": "code",
            ".pdf": "document", ".docx": "document", ".doc": "document",
            ".jpg": "media", ".png": "media", ".mp4": "media",
            ".zip": "archive", ".tar": "archive",
        }
        return extension_map.get(ext)

    # ── Shortcut B: Existing Folder Structure as Labels ─────────────────────

    def extract_folder_labels(self, search_root: Optional[str] = None) -> dict[str, str]:
        """
        Σκανάρει existing organized folders → ground truth weak labels.
        ~/Documents/EPL326/ → files there = label "EPL326"
        """
        root = Path(search_root or Path.home() / "Documents")
        folder_labels = {}

        for folder in root.iterdir():
            if not folder.is_dir() or folder.name.startswith("."):
                continue
            label = folder.name
            for file in folder.rglob("*"):
                if file.is_file():
                    folder_labels[str(file)] = label

        self._weak_labels.update(folder_labels)
        return folder_labels

    # ── Shortcut C: Batch Confirmation Session ───────────────────────────────

    def select_batch_representatives(
        self,
        file_paths: list[str],
        embeddings: np.ndarray,
        cluster_labels: np.ndarray,
        max_per_cluster: int = 1,
    ) -> list[ClusterRepresentative]:
        """
        Για κάθε cluster, επιλέγει το αρχείο πλησιέστερο στο centroid.
        Αυτά είναι τα αρχεία που πρέπει να δούμε στο batch confirmation session.

        Algorithm:
        - Υπολογίζει centroid = mean embedding του cluster
        - Επιλέγει αρχείο με min L2 distance από centroid
        - Returns: sorted by cluster_id, most representative first
        """
        unique_clusters = set(cluster_labels) - {-1}  # -1 = noise in HDBSCAN
        representatives = []

        for cluster_id in sorted(unique_clusters):
            cluster_mask = cluster_labels == cluster_id
            cluster_files = [fp for fp, m in zip(file_paths, cluster_mask) if m]
            cluster_embeddings = embeddings[cluster_mask]

            # Centroid = mean embedding
            centroid = cluster_embeddings.mean(axis=0)

            # L2 distances από centroid
            distances = np.linalg.norm(cluster_embeddings - centroid, axis=1)

            # Επίλεξε τα top-k αρχεία πλησιέστερα στο centroid
            sorted_indices = np.argsort(distances)[:max_per_cluster]

            for idx in sorted_indices:
                fp = cluster_files[idx]
                representatives.append(ClusterRepresentative(
                    cluster_id=cluster_id,
                    file_path=fp,
                    centroid_distance=float(distances[idx]),
                    weak_label=self._weak_labels.get(fp),
                ))

        return representatives

    def estimate_confirmations_needed(self, n_clusters: int) -> dict:
        """
        Εκτιμά πόσα confirmations χρειάζεται το batch session.
        vs original 400-confirmation requirement.
        """
        batch_needed = n_clusters * 1       # 1 representative per cluster
        setfit_equivalent = n_clusters * 8  # SetFit benchmark: 8 examples/class
        original = 400

        return {
            "batch_session_min": batch_needed,
            "setfit_equivalent": setfit_equivalent,
            "original_required": original,
            "speedup_vs_original": original / max(batch_needed, 1),
            "note": f"Batch session: {batch_needed} confirmations vs {original} original"
        }
```

**Κρίσιμο εύρημα:** Για 15 clusters × 8 examples = **120 confirmations** αντί για 400. Με batch session (1 per cluster) = **15 confirmations**. Αλλά προσοχή: 15 είναι πολύ λίγα για robust calibration. Το realistically achievable target είναι **30-80 confirmations** (2-5 per cluster), όχι 400.

---

### ΘΕΜΑ 3: Confidence Calibration — Είναι το 0.80 threshold σωστό;

#### Θεωρητικό υπόβαθρο

Το threshold 0.80 σημαίνει: "auto-route μόνο αν είμαι >80% confident". Αλλά τι σημαίνει "80% confident" αν το model δεν είναι calibrated;

Η paper "On Calibration of Modern Neural Networks" (Guo et al., arXiv:1706.04599) αποδεικνύει ότι τα neural networks είναι συχνά **overconfident** — λένε 90% ενώ η πραγματική accuracy είναι 70%. Αντίθετα, shallow classifiers (logistic regression) τείνουν να είναι underconfident.

**Temperature Scaling** (AWS Prescriptive Guidance): Διαιρεί τα logits με scalar T > 1 → softer probabilities → καλύτερη calibration. Minimal implementation:

```python
def apply_temperature_scaling(logit: float, temperature: float = 1.5) -> float:
    """Calibrate overconfident scores by dividing logits."""
    import math
    # Inverse sigmoid → scale → sigmoid
    # Assumes logit ∈ [0,1] is already a probability
    if logit <= 0 or logit >= 1:
        return logit
    raw_logit = math.log(logit / (1 - logit))
    scaled_logit = raw_logit / temperature
    return 1.0 / (1.0 + math.exp(-scaled_logit))
```

#### Cost Asymmetry: ποιο λάθος είναι χειρότερο;

Για personal file organization:
- **False Positive** (auto-route λάθος): αρχείο πήγε στο λάθος φάκελο — ο χρήστης δεν το βρίσκει
- **False Negative** (missed route → review queue): αρχείο καταλήγει στο "review" — ο χρήστης πρέπει να αποφασίσει μόνος

Η cost asymmetry είναι **ασύμμετρη**: ένα λάθος routing (false positive) είναι πολύ χειρότερο από ένα missed routing (false negative). Χαμένο αρχείο > επιπλέον manual review.

Συνέπεια: **Threshold πρέπει να είναι HIGH (0.85-0.90)** για personal use. Προτιμάμε false negatives (review queue) από false positives (wrong folder).

#### Simulation Function

```python
import numpy as np
from scipy import stats


def simulate_threshold_impact(
    similarity_distribution: list[float],
    threshold: float,
    false_positive_rate: float = 0.05,  # Εκτιμώμενο FP rate για scores > threshold
) -> dict:
    """
    Simulates routing behavior για 5000 αρχεία με δοσμένο threshold.

    Args:
        similarity_distribution: cosine similarity scores για κάθε αρχείο
        threshold: confidence threshold για auto-routing
        false_positive_rate: εκτιμώμενο ποσοστό λανθασμένων auto-routings

    Returns:
        dict με auto_routed, needs_review, estimated_errors, experience_quality
    """
    scores = np.array(similarity_distribution)
    n_total = len(scores)

    # Auto-route: scores >= threshold
    auto_routed_mask = scores >= threshold
    n_auto_routed = int(auto_routed_mask.sum())

    # Review queue: scores < threshold
    n_needs_review = n_total - n_auto_routed

    # Estimated errors (false positives in auto-routed)
    n_estimated_errors = int(n_auto_routed * false_positive_rate)

    # Experience quality metrics
    auto_route_pct = n_auto_routed / n_total
    review_burden = n_needs_review  # Files user must manually handle

    # User experience score (heuristic)
    # Balance: high auto-route is good, but errors are very bad (weight 3x)
    ux_score = auto_route_pct - 3 * (n_estimated_errors / n_total)

    return {
        "threshold": threshold,
        "total_files": n_total,
        "auto_routed": n_auto_routed,
        "auto_routed_pct": round(auto_route_pct * 100, 1),
        "needs_review": n_needs_review,
        "needs_review_pct": round((n_needs_review / n_total) * 100, 1),
        "estimated_errors": n_estimated_errors,
        "error_rate_pct": round((n_estimated_errors / n_total) * 100, 2),
        "ux_score": round(ux_score, 3),
        "recommendation": _threshold_recommendation(threshold, auto_route_pct, n_estimated_errors)
    }


def _threshold_recommendation(threshold: float, auto_pct: float, errors: int) -> str:
    if errors > 50:
        return "THRESHOLD_TOO_LOW: too many errors in auto-route"
    if auto_pct < 0.30:
        return "THRESHOLD_TOO_HIGH: most files go to review, defeating the purpose"
    if 0.50 <= auto_pct <= 0.75 and errors < 20:
        return "GOOD_BALANCE: acceptable auto-route rate with low errors"
    return "ACCEPTABLE"


def compare_thresholds(scores: list[float], thresholds: list[float] = None) -> None:
    """Τυπώνει σύγκριση για διάφορα thresholds."""
    if thresholds is None:
        thresholds = [0.60, 0.70, 0.75, 0.80, 0.85, 0.90]

    print(f"{'Threshold':>10} | {'Auto-Route':>10} | {'Review':>8} | {'Est. Errors':>12} | {'UX Score':>9}")
    print("-" * 65)
    for t in thresholds:
        r = simulate_threshold_impact(scores, t)
        print(f"{t:>10.2f} | {r['auto_routed_pct']:>9.1f}% | {r['needs_review_pct']:>7.1f}% | "
              f"{r['estimated_errors']:>12} | {r['ux_score']:>9.3f}")
```

#### Ποιο threshold για το Curator;

Βάσει της cost asymmetry ανάλυσης και του false positive cost:

| Threshold | Χαρακτήρας | Κατάλληλο για |
|-----------|------------|---------------|
| 0.70 | Aggressive | Demo/testing — αποδεκτές πολλές errors |
| 0.80 | Balanced | Default — αλλά **δεν αντισταθμίζει cost asymmetry** |
| 0.85 | Conservative | **Recommended για production** |
| 0.90 | Very conservative | Αν ο χρήστης είναι πολύ particular |

**Συμπέρασμα:** Άλλαξε default από 0.80 σε **0.85**. Επίσης: εφάρμοσε temperature scaling (T=1.5) στα raw scores αν χρησιμοποιείς neural embeddings — πιθανότατα είναι overconfident.

---

### ΘΕΜΑ 4: Πόσα Confirmations Χρειάζονται ΠΡΑΓΜΑΤΙΚΑ

#### Τι λέει η έρευνα

**SetFit benchmark (HuggingFace):** 8 examples/class → 92.7% accuracy στο IMDB dataset — "remarkably close to the all-data upper bound trained with 25,000 samples". Αυτό είναι ο gold standard για few-shot text classification.

**Few-shot classification theory** (arXiv:1909.11722): Θεωρητικό lower bound εξαρτάται από:
1. Class separability (πόσο διαφορετικά είναι τα clusters)
2. Embedding quality (pre-trained models → καλύτερα embeddings → λιγότερα examples)
3. Target accuracy

Για high-quality pre-trained embeddings + well-separated classes → **5-10 examples/class αρκούν**.

**KNN perspective:** Για k=3 nearest neighbors, χρειάζεσαι τουλάχιστον k examples per class → 3 examples/class = absolute minimum. Αλλά 3 είναι πολύ ευάλωτο σε outliers. Practical minimum: **8-15 examples/class**.

#### Required Confirmations Calculator

```python
import math


def estimate_required_confirmations(
    num_clusters: int,
    target_auto_approve_rate: float = 0.80,
    cluster_similarity: float = 0.3,   # 0=very different clusters, 1=very similar
    embedding_quality: str = "high",   # "high" (pre-trained) or "low" (TF-IDF)
    use_batch_session: bool = True,
) -> dict:
    """
    Εκτιμά πόσα confirmations χρειάζονται για target auto-approve rate.

    Βασίζεται σε:
    - SetFit benchmark: 8 examples/class για ~90% accuracy
    - Cold-start active learning: 1 centroid representative per cluster για initialization
    - Cluster similarity: παρόμοια clusters → περισσότερα examples για disambiguation

    Args:
        num_clusters: αριθμός HDBSCAN clusters
        target_auto_approve_rate: target ποσοστό auto-routing (0.0-1.0)
        cluster_similarity: πόσο overlap έχουν τα clusters (0.0-1.0)
        embedding_quality: ποιότητα embeddings ("high"=pre-trained, "low"=TF-IDF)
        use_batch_session: αν χρησιμοποιείται batch confirmation session

    Returns:
        dict με estimated confirmations και breakdown
    """
    # Base examples per class βάσει SetFit research
    if embedding_quality == "high":
        base_examples_per_class = 8    # SetFit with sentence-transformers
    else:
        base_examples_per_class = 25   # TF-IDF needs more data

    # Adjust για target accuracy
    # 80% target → μπορεί να πάμε με λιγότερα
    # 95% target → χρειαζόμαστε περισσότερα
    accuracy_multiplier = 1.0
    if target_auto_approve_rate < 0.70:
        accuracy_multiplier = 0.6
    elif target_auto_approve_rate < 0.80:
        accuracy_multiplier = 0.8
    elif target_auto_approve_rate < 0.90:
        accuracy_multiplier = 1.0
    else:
        accuracy_multiplier = 1.8  # 90%+ requires significantly more data

    # Adjust για cluster similarity
    # Παρόμοια clusters → πιο δύσκολη disambiguation → περισσότερα examples
    similarity_multiplier = 1.0 + cluster_similarity  # 0.3 → 1.3x

    # Final examples per class
    examples_per_class = math.ceil(
        base_examples_per_class * accuracy_multiplier * similarity_multiplier
    )
    examples_per_class = max(examples_per_class, 3)  # Absolute minimum: 3

    # Total confirmations
    total_confirmations = num_clusters * examples_per_class

    # Batch session: πρώτα 1 per cluster, μετά επιπλέον rounds
    if use_batch_session:
        initial_batch = num_clusters  # Round 1: 1 per cluster
        followup_rounds = examples_per_class - 1
        followup_total = num_clusters * followup_rounds
    else:
        initial_batch = total_confirmations
        followup_rounds = 0
        followup_total = 0

    # Comparison με original 400 threshold
    speedup = 400 / max(total_confirmations, 1)

    # Time estimate (3 confirmations/day for casual user)
    days_casual = math.ceil(total_confirmations / 3)
    days_active = math.ceil(total_confirmations / 10)

    return {
        "num_clusters": num_clusters,
        "target_auto_approve_rate": target_auto_approve_rate,
        "examples_per_class": examples_per_class,
        "total_confirmations": total_confirmations,
        "initial_batch_session": initial_batch,
        "followup_confirmations": followup_total,
        "speedup_vs_400": round(speedup, 1),
        "days_to_mature_casual": days_casual,    # 3/day
        "days_to_mature_active": days_active,    # 10/day
        "vs_original_days_casual": math.ceil(400 / 3),
        "recommendation": _confirmation_recommendation(total_confirmations, speedup),
    }


def _confirmation_recommendation(total: int, speedup: float) -> str:
    if total <= 30:
        return f"FAST_COLD_START: {total} confirmations, {speedup:.1f}x faster than original"
    elif total <= 100:
        return f"REASONABLE: {total} confirmations achievable in 1-2 weeks"
    elif total <= 200:
        return f"ACCEPTABLE: {total} confirmations, consider batch session to accelerate"
    else:
        return f"SLOW: {total} confirmations — reconsider architecture or lower target"


# Παράδειγμα χρήσης για τυπικό Curator scenario
if __name__ == "__main__":
    # Τυπικός χρήστης: 15 clusters, 80% auto-approve target, high-quality embeddings
    result = estimate_required_confirmations(
        num_clusters=15,
        target_auto_approve_rate=0.80,
        cluster_similarity=0.3,
        embedding_quality="high",
        use_batch_session=True,
    )
    print("\n=== Required Confirmations for Curator ===")
    for k, v in result.items():
        print(f"  {k:35s}: {v}")
```

**Τυπικό output για 15 clusters:**
```
  num_clusters                       : 15
  target_auto_approve_rate           : 0.8
  examples_per_class                 : 11
  total_confirmations                : 165
  initial_batch_session              : 15
  followup_confirmations             : 150
  speedup_vs_400                     : 2.4
  days_to_mature_casual              : 55     ← 3/day
  days_to_mature_active              : 17     ← 10/day
  vs_original_days_casual            : 134
```

#### Αναθεωρημένα maturity thresholds

Αντί για ένα monolithic "400 = Mature":

| Κατάσταση | Confirmations | Χαρακτήρας |
|-----------|---------------|------------|
| Initializing | 0–14 | Batch session: 1 per cluster |
| Learning | 15–(8×clusters) | 8 examples/class target |
| Proficient | 8×clusters–(15×clusters) | 15 examples/class |
| Mature | 15×clusters+ | Stable calibration |

Για 15 clusters: Mature = **225 confirmations** (αντί 400). Για 10 clusters: **150 confirmations**. Πολύ πιο ρεαλιστικό.

---

### ΣΥΝΟΨΗ: Τι αλλάζει στον Curator

| Στοιχείο | Τρέχουσα κατάσταση | Προτεινόμενο |
|----------|-------------------|--------------|
| Routing weights | Hardcoded (0.35, 0.25...) | `RoutingWeightCalibrator` — update μετά 20 decisions |
| Cold start | 400 confirmations για Mature | Batch session (1/cluster) + 8 examples/class target |
| Confidence threshold | 0.80 (flat) | 0.85 default + temperature scaling (T=1.5) |
| Maturity definition | Fixed 400 | `15 × num_clusters` (dynamic) |
| Folder labels | Ignored | `ColdStartAccelerator.extract_folder_labels()` |

**Το σημαντικότερο insight:** Το 400-confirmation threshold δεν βασίζεται σε καμία θεωρητική ή εμπειρική βάση. Η SetFit έρευνα δείχνει 8 examples/class ≈ 90% accuracy. Για 15 clusters, αυτό σημαίνει 120 confirmations — και με batch session μπορούμε να κάνουμε Initializing → Proficient με **30-50 confirmations** αν χρησιμοποιούμε centroid representatives.

---

*Πηγές: arXiv:2003.08109, arXiv:2402.02561, HuggingFace SetFit blog, Microsoft Research Thompson Sampling, AWS Temperature Scaling docs, Snorkel AI weak supervision, Dropbox ML blog, Capital One XAI ablation studies, IOS Press IDA journal cold-start active learning.*