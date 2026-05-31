# 34 — FSRS-4.5 as Replacement for SM-2 in the FLRS B(f,t) Component
_Date: 2026-05-31_

---

## 1. Context: Where B(f,t) Lives in FLRS

Curator's relevance signal is:

```
FLRS(f,t) = R(t,f) × A(f) × B(f,t)
```

- **R(t,f)**: Ebbinghaus retrievability — exponential decay since last access.
- **A(f)**: Activity weight — recency-weighted access frequency.
- **B(f,t)**: Retrieval-strengthening bonus — how much prior successful retrievals have consolidated the memory trace for file `f`.

Currently `B(f,t)` is implemented using SM-2 logic: each confirmed retrieval boosts an ease-factor-driven interval. This document evaluates replacing that with the FSRS-4.5 algorithm, which is demonstrably more accurate and theoretically better grounded in memory science.

---

## 2. What is FSRS?

**FSRS** (Free Spaced Repetition Scheduler) is an open-source spaced repetition algorithm developed by Jarrett Ye and collaborators, first published in 2022 and fitted on 500 million+ Anki review logs. It is the theoretical successor to Piotr Woźniak's SM-2 (1987) and supersedes it on every empirical benchmark.

The theoretical backbone is the **DSR model** — three coupled state variables:

| Variable | Symbol | Meaning |
|---|---|---|
| **Difficulty** | D | Intrinsic hardness of the item, 1–10 |
| **Stability** | S | Days until retrievability drops to 90%; measure of memory strength |
| **Retrievability** | R | Current probability of successful recall (0–1) |

These three variables replace SM-2's single scalar: the ease factor.

The foundational paper is: **Ye, Su & Cao (2022). "A Stochastic Shortest Path Algorithm for Optimizing Spaced Repetition Scheduling." Proceedings of the 28th ACM SIGKDD Conference on Knowledge Discovery and Data Mining, pp. 4381–4390.** The algorithm was shown to produce a 12.6% improvement over prior methods when validated on real Anki data.

---

## 3. FSRS Mathematical Formulation (FSRS-4.5 / v4)

### 3.1 Retrievability

```
R(t, S) = (1 + t / (9·S))^(-1)
```

This is a power-law forgetting curve. At `t = S`, `R ≈ 0.9` — that is, S is defined as the interval at which you have a 90% chance of remembering. An older form (FSRS v1/v3) used the exponential:

```
R(t, S) = 0.9^(t/S)
```

Both encode the same intuition: memory decays over time, stabilised by S. The power-law version fits empirical data slightly better.

### 3.2 Initial Stability S₀

For a new card rated with grade G ∈ {1=Again, 2=Hard, 3=Good, 4=Easy}:

```
S₀(G) = w_{G-1}
```

That is, the four initial weights directly encode expected stability per grade:

| Grade | Default S₀ (days) |
|---|---|
| Again (1) | w₀ = 0.4872 |
| Hard (2) | w₁ = 1.4003 |
| Good (3) | w₂ = 3.7145 |
| Easy (4) | w₃ = 13.8206 |

### 3.3 Difficulty D

**Initial difficulty** based on first grade G:

```
D₀(G) = w₄ − (G − 3) · w₅
```

Default values: w₄ = 5.1618, w₅ = 1.2298. So a "Good" first rating gives D₀ = 5.16, "Easy" gives ~3.9, "Again" gives ~7.6.

**Difficulty update** after each subsequent review:

```
D'(D, G) = w₇ · D₀(3) + (1 − w₇) · (D − w₆ · (G − 3))
```

Default: w₆ = 0.8975, w₇ = 0.031. This is a mean-reverting update: difficulty drifts back toward D₀(3) = w₄ over time, preventing permanent stigma from early hard reviews (unlike SM-2 where ease can drop to its floor and stay there).

### 3.4 Stability After Successful Recall (S'_r)

When a card is reviewed and recalled (grade ≥ 1), stability updates as:

```
S'_r(D, S, R, G) = S · [e^{w₈} · (11 − D) · S^{−w₉} · (e^{w₁₀·(1−R)} − 1) · hardBonus · easyBonus + 1]
```

Where:
- `hardBonus = w₁₅` if G = Hard (2), else 1
- `easyBonus = w₁₆` if G = Easy (4), else 1
- Default values: w₈ = 1.6474, w₉ = 0.1367, w₁₀ = 1.0461, w₁₅ = 0.2272, w₁₆ = 2.8755

Key properties encoded here:
- Higher difficulty (D) → smaller stability gain (numerator decreases with D approaching 11)
- Higher current S → smaller proportional gain (S^{-w₉} term)
- Lower current R → larger gain (the `e^{w₁₀·(1-R)} - 1` term rewards reviewing just as you're about to forget)

This last property — **reviewing at low R produces larger stability gains** — is the core scientific insight absent from SM-2.

### 3.5 Stability After Lapse (S'_f)

When a card is forgotten (grade = Again):

```
S'_f(D, S, R) = w₁₁ · D^{−w₁₂} · ((S + 1)^{w₁₃} − 1) · e^{w₁₄·(1−R)}
```

Default: w₁₁ = 2.1072, w₁₂ = 0.0793, w₁₃ = 0.3246, w₁₄ = 1.587. Critically, this does **not** reset S to zero — it reduces it proportionally, preserving partial memory credit. SM-2 effectively resets the interval to 1 day on any failure.

### 3.6 FSRS-4.5 Full Default Parameter Set (w₀ – w₁₆)

```python
DEFAULT_PARAMETERS = [
    0.4872,   # w0:  S0 for Again
    1.4003,   # w1:  S0 for Hard
    3.7145,   # w2:  S0 for Good
    13.8206,  # w3:  S0 for Easy
    5.1618,   # w4:  Initial difficulty base
    1.2298,   # w5:  Difficulty sensitivity to first grade
    0.8975,   # w6:  Difficulty decay per grade delta
    0.031,    # w7:  Mean-reversion rate for difficulty
    1.6474,   # w8:  Stability gain scaling (recall)
    0.1367,   # w9:  Stability dampening exponent (recall)
    1.0461,   # w10: Retrievability bonus exponent (recall)
    2.1072,   # w11: Post-lapse stability coefficient
    0.0793,   # w12: Difficulty penalty on lapse stability
    0.3246,   # w13: Prior stability preservation on lapse
    1.587,    # w14: Retrievability penalty on lapse stability
    0.2272,   # w15: Hard-grade penalty multiplier
    2.8755,   # w16: Easy-grade bonus multiplier
]
```

FSRS-6 (latest as of 2025) extended this to 21 parameters (w₀–w₂₀), adding same-day review handling and a refined power-law forgetting exponent. FSRS-4.5 remains the most widely validated version for integration purposes.

### 3.7 State Machine

```
New ──(first review)──► Learning
Learning ──(graduated)──► Review
Review ──(lapse)──► Relearning
Relearning ──(relearned)──► Review
```

In py-fsrs terms:
```python
State.Learning   == 1
State.Review     == 2
State.Relearning == 3
```

---

## 4. SM-2: What It Does and Why It Falls Short

### 4.1 SM-2 Mechanism

SM-2 schedules reviews using two scalars per card:

- **Ease Factor (EF)**: initialised at 2.5, bounded to [1.3, ∞).
- **Interval (I)**: days until next review.

Update rules:
```
I₁ = 1 day
I₂ = 6 days
I_n = I_{n-1} × EF   (for n ≥ 3)

EF' = EF + (0.1 − (5 − q) × (0.08 + (5 − q) × 0.02))
```

Where q ∈ {0,…,5} is the quality score. A "perfect" response increases EF by 0.1; an "Again" drops it by 0.20.

### 4.2 SM-2 Limitations

**Ease Hell**: Cards that are reviewed at a bad time (tired, interrupted) accumulate EF reductions that compound. EF has no recovery mechanism and no mean-reversion. A card that drops to EF = 1.3 will forever be reviewed at 30% of the pace of other cards regardless of subsequent excellent performance.

**No Retrievability Sensitivity**: SM-2 does not model the current probability of recall. It cannot reward a "Good" review that happens at the moment of near-forgetting (R ≈ 0.1) more than the same grade when R = 0.95. FSRS's `e^{w₁₀·(1-R)} − 1` term does exactly this.

**Binary Reset on Failure**: Any lapse resets the interval to 1 day and drops EF by 0.20, regardless of whether the failure was marginal (90% → 88% retrieval) or catastrophic (previously forgotten for months). FSRS's `S'_f` preserves partial stability.

**No Intrinsic Difficulty**: EF is updated reactively from grades but is not decomposed into an item-intrinsic difficulty and a memory-strength component. If you're having a bad day, all items suffer equally. FSRS separates D (item property) from S (memory state).

**Accuracy Benchmark**: Per open-spaced-repetition benchmarking on the 500M-review Anki dataset:
- FSRS with per-user optimisation achieves mean log-loss ≈ **0.344**
- SM-2 (Anki variant) achieves significantly higher log-loss
- FSRS predicts recall more accurately for **99.5% of users**
- Same retention rate achieved with **~20–30% fewer reviews** using FSRS-5

---

## 5. Mapping FSRS to the FLRS B(f,t) Component

### 5.1 The Conceptual Mapping

FSRS was designed for flashcard recall. Curator's file retrieval context is analogous:

| Flashcard Domain | Curator File Domain |
|---|---|
| Card | File `f` |
| Review session | File access event |
| Grade: Again (1) | Search-assisted access (user couldn't navigate directly) |
| Grade: Hard (2) | Navigation with wrong first path, backtracked |
| Grade: Good (3) | Direct navigation (opened from correct directory) |
| Grade: Easy (4) | Instant access from pinned/recent (not useful for B) |
| Stability S | Memory consolidation strength for `f`'s location |
| Difficulty D | Intrinsic locatability of `f` (nested deep, non-obvious name) |
| Retrievability R | Current probability user can navigate directly to `f` |

### 5.2 Inferring Grade from Access Patterns

Access events in Curator can be classified into grades without explicit user input:

```python
def infer_grade(access_event) -> Rating:
    if access_event.method == "direct_navigation":
        return Rating.Good     # 3
    elif access_event.method == "search":
        return Rating.Again    # 1  (had to search = couldn't retrieve location)
    elif access_event.method == "recent_list":
        return Rating.Hard     # 2  (cued recall, not free recall)
    elif access_event.method == "bookmark":
        return Rating.Easy     # 4
```

This is the key insight from document 25: direct navigation = strong retrieval, search-assisted = retrieval failure for location memory purposes.

### 5.3 B(f,t) as FSRS Stability

With FSRS, `B(f,t)` becomes a function of the FSRS stability S for file `f`:

```
B(f,t) = R_FSRS(t_since_last_access, S_f)
       = (1 + t / (9 · S_f))^(-1)
```

Where S_f is maintained by FSRS and updated on each access event using the grade-inferred review. This replaces the ad-hoc SM-2 bonus with a theoretically grounded, continuously updated probability.

The full FLRS formula becomes:

```
FLRS(f,t) = R_decay(t,f) × A(f) × B_FSRS(f,t)
```

Where:
- `R_decay(t,f) = exp(-t / S₀)`: raw Ebbinghaus decay since last touch (all files)
- `B_FSRS(f,t) = (1 + Δt / (9·S_f))^(-1)`: FSRS-modulated retrieval strength
- `A(f)`: activity weight (unchanged)

Note that R_decay and B_FSRS will partially overlap in meaning, and you may elect to collapse them: use FSRS's R directly as the combined decay+memory signal, dropping R_decay in favour of FSRS's integrated model.

### 5.4 Difficulty D as File Locatability

FSRS's difficulty D (1–10) can be pre-seeded from filesystem features before sufficient access history accumulates:

```python
def initial_difficulty(file_path) -> float:
    depth = len(file_path.parts)
    name_clarity = 1.0 if file_path.suffix in ['.pdf','.py','.md'] else 1.5
    nesting_penalty = min(depth / 3.0, 3.0)
    return min(10.0, 5.0 + nesting_penalty * name_clarity)
```

Files deeply nested (depth > 5), with ambiguous names, or in non-obvious directories start with higher D, meaning their stability grows more slowly per access — reflecting the real cognitive cost of locating them.

---

## 6. py-fsrs Integration

### 6.1 Installation and API

```bash
pip install fsrs
```

```python
from fsrs import Scheduler, Card, Rating, State
from datetime import datetime, timezone

scheduler = Scheduler(
    parameters=DEFAULT_PARAMETERS,   # tuple of 17 or 21 weights
    desired_retention=0.9,           # target R at review time
    maximum_interval=365,
)
```

### 6.2 Per-File Card Lifecycle

```python
import json
from pathlib import Path

# One FSRS Card object per tracked file
def load_file_card(file_id: str) -> Card:
    data = db.get(f"fsrs:{file_id}")
    return Card.from_json(data) if data else Card()

def save_file_card(file_id: str, card: Card):
    db.set(f"fsrs:{file_id}", card.to_json())

def on_file_accessed(file_id: str, access_method: str):
    card = load_file_card(file_id)
    grade = infer_grade(access_method)  # Rating.Again/Hard/Good/Easy
    card, review_log = scheduler.review_card(card, grade)
    save_file_card(file_id, card)
    return card

def get_b_factor(file_id: str) -> float:
    card = load_file_card(file_id)
    return scheduler.get_card_retrievability(card)  # returns float 0–1
```

### 6.3 Handling Cold Start (Sparse Data)

FSRS defaults degrade gracefully to heuristic scheduling when data is sparse. For files with fewer than ~5 access events, the default w₀–w₃ values are used directly as S₀. A reasonable cold-start policy:

```python
MIN_ACCESSES_FOR_OPTIMISATION = 10

def get_effective_parameters(file_id: str) -> tuple:
    n_accesses = db.count_accesses(file_id)
    if n_accesses < MIN_ACCESSES_FOR_OPTIMISATION:
        return DEFAULT_PARAMETERS   # fall back to population defaults
    else:
        return db.get_optimised_params(file_id)  # per-file fitted weights
```

---

## 7. Fitting FSRS to File Access Data

### 7.1 What Training Data Looks Like

FSRS's optimizer expects a sequence of (card_id, review_timestamp, grade, outcome) tuples. For Curator:

```
(file_id, accessed_at: datetime, grade: 1|2|3|4, success: bool)
```

Where `success = grade > 1` (i.e., the user found the file without requiring a full search). This maps to binary cross-entropy loss training, identical to flashcard recall.

### 7.2 Using fsrs-optimizer

```bash
pip install FSRS-Optimizer
```

```python
from fsrs_optimizer import Optimizer

optimizer = Optimizer()
optimizer.load_revlogs(review_logs)   # list of dicts with card_id, review_time, rating
optimised_params = optimizer.compute_optimal_parameters()
```

The optimizer uses gradient descent on binary cross-entropy to fit the 17 (or 21) weights to your review history. Training is stable with ~500+ access events across files. Below ~500 events, default parameters are recommended.

### 7.3 Challenges of Refitting on File Access Data

**Key differences from flashcard data:**

1. **Imbalanced grades**: Most file accesses will be Grade 3 (Good/direct). Grade 1 (Again) happens when search is used, which is rarer. The imbalance means w₁₁–w₁₄ (lapse parameters) will be poorly fitted from file data alone.

2. **No explicit failure signal without search interception**: Unlike flashcards where "Again" is explicit, file access doesn't record whether the user *tried* to navigate directly and failed. Curator must infer this from access method metadata.

3. **Access sparsity**: A flashcard might be reviewed 50–100 times per year. A typical file is accessed far less frequently. Many files will never accumulate enough events for personalised fitting.

4. **Temporal clustering**: Files are often accessed in bursts (project sprints) rather than distributed over time. FSRS's stability model may overfit these burst patterns as high-stability items that then rapidly decay.

**Recommended approach**: Use FSRS default parameters (population priors from 500M Anki reviews) without per-user refitting initially. The default curve is already scientifically valid. Personalised fitting becomes viable at the user level (not per-file) once ~1,000 total graded access events are logged.

---

## 8. Empirical Evidence Supporting FSRS Over SM-2

### 8.1 FSRS Benchmark (open-spaced-repetition/srs-benchmark)

The open-spaced-repetition benchmark evaluated multiple schedulers on 500M Anki reviews using **log-loss** as the primary metric (lower = more accurate recall prediction):

| Algorithm | Mean Log-Loss | Relative Improvement |
|---|---|---|
| FSRS-6 (per-user) | ~0.344 | baseline |
| FSRS-4.5 (per-user) | ~0.348 | −1.2% vs FSRS-6 |
| SM-2 (Anki) | significantly higher | ~20–30% worse |
| FSRS default (no fitting) | slightly above per-user | viable cold start |

FSRS outperforms SM-2 for **99.5% of tested users**.

### 8.2 Review Efficiency

For a target 90% retention:
- FSRS requires **~20–25% fewer reviews** than SM-2 (Anki's variant)
- The reduction comes from FSRS scheduling longer intervals when S is high and R is still adequate, rather than SM-2's conservative fixed-multiplier approach

### 8.3 Ease Hell Prevention

In SM-2, once a card enters "ease hell" (EF → 1.3), it stays there. FSRS's mean-reverting difficulty update (w₇ term) ensures D drifts back toward the population mean, so a file that was hard to locate early (because it had a confusing name) is not permanently penalised once the user learns its location.

### 8.4 Original References

- **Woźniak (1987)**: SM-2 algorithm. SuperMemo documentation. Unpublished algorithm, publicly documented at supermemo.com.
- **Ye, Su & Cao (2022)**: "A Stochastic Shortest Path Algorithm for Optimizing Spaced Repetition Scheduling." ACM KDD '22, pp. 4381–4390. DOI: 10.1145/3534678.3539081.
- **Open-spaced-repetition benchmark**: github.com/open-spaced-repetition/srs-benchmark

---

## 9. Concrete B(f,t) Implementation Sketch

```python
from fsrs import Scheduler, Card, Rating
from datetime import datetime, timezone

DEFAULT_W = (
    0.4872, 1.4003, 3.7145, 13.8206,  # S0 per grade
    5.1618, 1.2298, 0.8975, 0.031,     # difficulty
    1.6474, 0.1367, 1.0461,            # recall stability
    2.1072, 0.0793, 0.3246, 1.587,    # lapse stability
    0.2272, 2.8755,                    # hard/easy modifiers
)

class FSRSBonusComponent:
    def __init__(self):
        self.scheduler = Scheduler(parameters=DEFAULT_W, desired_retention=0.9)

    def on_access(self, file_state: dict, method: str) -> dict:
        card = Card.from_dict(file_state) if file_state else Card()
        grade = self._grade(method)
        card, _ = self.scheduler.review_card(card, grade)
        return card.to_dict()

    def get_b(self, file_state: dict) -> float:
        if not file_state:
            return 1.0   # unseen file: no bonus, no penalty
        card = Card.from_dict(file_state)
        return self.scheduler.get_card_retrievability(card)

    @staticmethod
    def _grade(method: str) -> Rating:
        return {
            "direct":   Rating.Good,   # 3
            "recent":   Rating.Hard,   # 2
            "search":   Rating.Again,  # 1
            "bookmark": Rating.Easy,   # 4
        }.get(method, Rating.Good)


# Integration into FLRS:
def flrs(f, t, file_state, fsrs_component, decay_fn, activity_fn) -> float:
    R  = decay_fn(f, t)                       # Ebbinghaus decay
    A  = activity_fn(f)                       # access frequency weight
    B  = fsrs_component.get_b(file_state[f])  # FSRS retrieval bonus
    return R * A * B
```

---

## 10. Summary and Recommendation

FSRS-4.5 is a drop-in scientific upgrade over SM-2 for Curator's B(f,t) component. The key advantages:

1. **Three-state model** (D, S, R) versus SM-2's two scalars (EF, I) — more expressive and separates item difficulty from memory strength.
2. **Retrievability-sensitive stability gains** — a "Good" review at R=0.1 (near-forgetting) gives more stability boost than the same grade at R=0.9, matching the psychological testing effect.
3. **Partial stability preservation on lapse** — forgetting a file location does not reset its memory trace to zero.
4. **Mean-reverting difficulty** — prevents permanent "ease hell" for initially hard-to-find files.
5. **Empirically validated**: 99.5% of users see better recall predictions, 20–30% fewer reviews needed for same retention.
6. **py-fsrs** provides a clean Python API with `Card`, `Scheduler`, and `Rating` that maps naturally onto Curator's per-file state.

**Immediate action**: Replace the SM-2 scalar update in B(f,t) with `FSRSBonusComponent.get_b()`. Store one FSRS `Card` dict per tracked file in Curator's database. Infer grade from access method. No per-file refitting is needed initially — the FSRS default parameters (fitted on 500M human memory events) are already better calibrated than SM-2.

---

_Sources: [awesome-fsrs wiki](https://github.com/open-spaced-repetition/awesome-fsrs/wiki/The-Algorithm) · [py-fsrs](https://github.com/open-spaced-repetition/py-fsrs) · [KDD '22 paper](https://dl.acm.org/doi/10.1145/3534678.3539081) · [srs-benchmark](https://github.com/open-spaced-repetition/srs-benchmark) · [fsrs-optimizer PyPI](https://pypi.org/project/FSRS-Optimizer/) · [FSRS vs SM-2 (memstride)](https://memstride.com/blog/fsrs-vs-sm2-algorithm-comparison/) · [FSRS vs SM-2 (diane.app)](https://www.diane.app/en/guides/fsrs-vs-sm2)_
