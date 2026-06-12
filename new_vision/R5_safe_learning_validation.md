# R5 — Safe Learning & Pattern Validation

**Curator Research Series | New Vision Pipeline**
*Written: 2026-06-08 | Model: Claude Sonnet 4.6*

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The Learning Problem in Curator](#2-the-learning-problem-in-curator)
3. [Prior Art — Comprehensive Literature Review](#3-prior-art)
4. [Learning State Machine](#4-learning-state-machine)
5. [Signal Types — What Each Correction Teaches](#5-signal-types)
6. [Overfitting Prevention](#6-overfitting-prevention)
7. [Automation Level Escalation](#7-automation-level-escalation)
8. [Group-Level to File-Level Signal Propagation](#8-group-level-to-file-level-signal-propagation)
9. [What NOT to Learn From](#9-what-not-to-learn-from)
10. [Implementation Options](#10-implementation-options)
11. [Design Decisions](#11-design-decisions)
12. [Open Questions](#12-open-questions)

---

## 1. Executive Summary

Curator learns from group-level user corrections — approve, rename, split, merge, mark-wrong — and must turn those corrections into persistent routing rules without overfitting to single examples. This is structurally different from standard supervised learning: the user never provides explicit labels, only corrections to the system's output. The feedback is implicit, sparse, and potentially noisy (decision fatigue, regretted corrections).

The core design is a **seven-state learning state machine** where a pattern must be observed at least three times in similar contexts before it becomes a `candidate`, and must survive explicit user confirmation before it becomes a `confirmed_rule`. Silent automation (applying rules without showing the user) requires a separate opt-in and a track record of fewer than 10% undo events per rule.

The three-tier learning model — `observed → candidate → suggested_rule → confirmed_rule → auto_rule` — deliberately slows down generalization. This is the right tradeoff for a personal filesystem tool: false positives (wrong automatic moves) are far more damaging than false negatives (missing a valid rule).

**Curator decision (summary):** Never auto-apply a rule from a single correction. Require N=3 similar corrections for candidacy. Present candidate rules to the user for explicit confirmation. Apply confirmed rules with notification. Escalate to silent auto only after a 30-day track record with undo rate < 10%, and only when the user has explicitly opted in per rule.

---

## 2. The Learning Problem in Curator

### 2.1 What Makes This Hard

Standard machine learning assumes: many labeled examples → train classifier → predict labels for new examples. Curator's learning problem violates every one of these assumptions.

**Problem 1 — Labels are corrections, not annotations.** The user does not say "this file belongs in EPL401." The user says "your group is wrong — split it." A correction encodes dissatisfaction with the current state, but does not fully specify the correct state. This is structurally closer to preference learning (Fürnkranz & Hüllermeier, 2010) than to supervised classification.

**Problem 2 — One correction has near-zero statistical power.** If the user splits "Security" into "EPL312" and "EPL401" once, the base rate for that split is unknown. It could be a one-time manual preference for that specific review session. Generalization from a single example is logically unjustified (no law of large numbers applies).

**Problem 3 — Groups are heterogeneous.** An approved group does not mean every file in the group is correctly placed — it means the group as a whole is acceptable. Some files may be borderline cases the user didn't notice. File-level signals must be derived from group-level approval with appropriate uncertainty.

**Problem 4 — Corrections are temporally sparse.** A user may make 20 corrections in one session, then not use Curator for three weeks. Rules formed from a single session may encode session-specific preferences, not stable long-term organization intent.

**Problem 5 — No negative examples.** When a group is approved, it does not follow that all other possible groupings of those files are wrong. The user approved one grouping; they did not evaluate alternatives. This is the Positive-Unlabeled (PU) learning setting.

### 2.2 What the System Must Guarantee

1. **Safety**: an incorrect rule must never cause irreversible file operations without user awareness.
2. **Transparency**: the user can always see what rules exist, what triggered them, and how to disable them.
3. **Reversibility**: any automatic action taken under a rule can be undone with a single tap.
4. **Non-retaliation**: if the user corrects a rule-driven decision, the rule's confidence decreases — the system does not re-apply the same rule to the same file type.
5. **Scope honesty**: rules derived from one context (e.g., University work) do not automatically apply to another context (e.g., Personal projects).

---

## 3. Prior Art — Comprehensive Literature Review

### 3.1 Interactive Machine Learning (IML) — Foundational Survey

**Amershi, S., Cakmak, M., Knox, W. B., & Kulesza, T. (2014). Power to the People: The Role of Humans in Interactive Machine Learning. *AI Magazine*, 35(4), 105–120.**
DOI: https://doi.org/10.1609/aimag.v35i4.2513

This is the foundational IML survey. Amershi et al. identify three generalization failure modes that are directly relevant to Curator:

1. **Under-generalization**: the model only remembers the exact correction, does not generalize.
2. **Over-generalization**: the model applies the correction too broadly, breaking other cases.
3. **Misaligned generalization**: the model generalizes along the wrong dimension (e.g., user corrects label, model corrects placement).

The paper establishes that IML systems require a "mixed initiative" where the system proposes, the user corrects, and the system re-proposes — but the key insight is that this loop only works when the user can understand *why* the system changed. If the system over-generalizes invisibly, trust breaks down. For Curator, this means: every automatic rule application must be traceable to the specific correction that created it.

**Key IML principle for Curator:** "The system should update its model in the direction of the user's correction, not in the exact point the user specified." This means a split correction should soften the boundary between the two resulting groups, not add a hard wall that splits all future similar material.

**Curator decision:** Follow Amershi's recommendation for mixed-initiative: show the user the rule that would be created ("Based on your correction, I think you want EPL401 files separated from EPL312 files. Create a rule?") and wait for explicit confirmation before creating it.

---

**Fails, J., & Olsen, D. R. (2003). Interactive Machine Learning. *Proceedings of IUI 2003*, 39–45.**
DOI: https://doi.org/10.1145/604045.604056

The original IML paper. Fails & Olsen describe the "retrain-on-correction" cycle: user labels a few examples → system retrains → system shows new predictions → user corrects → repeat. The critical lesson from their evaluation: **users do not understand what the system learned from a correction**. They found that users expected corrections to propagate locally (nearby examples) but systems often propagated globally (all examples).

This mismatch — between user expectation and actual generalization — is the core trust problem in IML. Curator must close this gap by showing the user the exact scope of any proposed rule ("This rule would affect 47 files currently in your Review Hub").

**Curator decision:** For every candidate rule, compute and display "scope preview" — how many current unreviewed files would be affected if the rule were confirmed. User sees the rule's real-world impact before confirming.

---

**Recent IML surveys (2022–2026):**

**Dudley, J. J., & Kristensson, P. O. (2018). A Review of User Interface Design for Interactive Machine Learning. *ACM Transactions on Interactive Intelligent Systems*, 8(2), 8.**
DOI: https://doi.org/10.1145/3185517

Surveys 30 IML systems through 2018. Key finding: systems that show users the "model's current state" (what it knows) dramatically outperform systems that only show predictions. For Curator: the Rules view must show users the learned rule set as a list they can inspect, edit, and delete — not just a black box that occasionally takes automatic actions.

**Mosqueira-Rey, E., Hernández-Pereira, E., Alonso-Ríos, D., Bobes-Bascarán, J., & Alonso-Betanzos, A. (2023). Human-in-the-loop machine learning: A state of the art. *Artificial Intelligence Review*, 56, 3005–3054.**
DOI: https://doi.org/10.1007/s10462-022-10246-0

This 2023 survey covers 50+ recent HITL/IML systems. Most relevant finding for Curator: "correction fatigue" — when users are shown too many correction prompts, correction quality degrades. Systems that batch corrections and present them together perform better than systems that interrupt with each correction prompt. This validates Curator's group-first approach: show 20 group decisions, not 300 individual file prompts.

The survey also introduces the concept of "active learning for corrections" — the system should prioritize showing users the corrections it is most uncertain about, rather than processing in arbitrary order. For Curator: sort Review Hub groups by Curator's internal confidence — low-confidence groups first, so user corrections have the highest information value.

**Curator decision:** Sort Review Hub groups ascending by Curator's grouping confidence score, not by recency. Low-confidence groups get corrected first, maximizing information gain per user interaction.

---

### 3.2 Learning from Corrections vs. Learning from Labels

**Preference Learning framing:**
Fürnkranz, J., & Hüllermeier, E. (Eds.). (2010). *Preference Learning*. Springer.
DOI: https://doi.org/10.1007/978-3-642-14125-6

A correction is structurally a **preference statement**: the user prefers state B over state A that Curator proposed. This is richer than a binary label ("correct" / "incorrect") because it encodes directionality. A split correction says: "I prefer two separate groups over one merged group, given these specific files." This preference can be modeled using **pairwise preference learning** (Bradley-Terry model, or its Bayesian extension), where each correction updates a latent preference score between two organizational states.

For Curator, this framing is conceptually useful but likely over-engineered for the first version. The key takeaway: a correction is a local preference statement, not a global rule. Propagating it globally requires additional evidence.

**Clickthrough / Relevance Feedback:**
Joachims, T., Granka, L., Pan, B., Hembrooke, H., & Gay, G. (2005). Accurately Interpreting Clickthrough Data as Implicit Feedback. *Proceedings of SIGIR 2005*, 154–161.
DOI: https://doi.org/10.1145/1076034.1076063

Joachims et al. showed that clickthrough data — a form of implicit feedback — encodes relative preferences (clicked result > skipped result above it) but is biased by position and presentation order. The key insight for Curator: implicit corrections are directional but biased. A user who approves a group may do so because it is genuinely correct, or because they are fatigued and just want to move on. Curator should weight corrections by session position (early corrections are more reliable) and correction type (explicit actions like split/merge are more reliable than passive approvals).

**Curator decision:** Weight corrections by type for learning purposes: split/merge/mark-wrong carry 3x the learning weight of approve (because they require explicit active effort). Rename carries 2x the weight of approve.

---

### 3.3 Constrained Clustering — Must-Link / Cannot-Link

**Wagstaff, K., Cardie, C., Rogers, S., & Schroedl, S. (2001). Constrained K-means Clustering with Background Knowledge. *Proceedings of ICML 2001*, 577–584.**
Available: https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.13.8611

The foundational constrained clustering paper. COP-KMeans modifies the k-means assignment step: before assigning a point to the nearest centroid, check whether the assignment violates any must-link or cannot-link constraint. Must-link means "these two points must be in the same cluster." Cannot-link means "these two points must be in different clusters."

**Mapping to Curator:**
- User **merges** two groups → **must-link** constraint between the centroids of those groups (or between their file sets)
- User **splits** a group → **cannot-link** constraint between the two resulting sub-groups
- User **approves** a group → soft confirmation, no hard constraint (approval is weaker than explicit structural correction)

COP-KMeans is NP-hard when constraints conflict, but in practice with few user constraints on a large corpus, this is not an issue. The practical concern for Curator: constraints accumulate over months. A constraint from October ("merge Thesis and Research Papers") may conflict with a constraint from March ("split Academic Work into Thesis and Papers"). The rule store must record timestamps and allow the user to resolve conflicts.

**Curator decision:** Represent merge as must-link constraints and split as cannot-link constraints. Store both in the rule DB with timestamps. On conflict (two constraints that would violate each other), surface both to the user and ask them to resolve, never resolve silently.

---

**Basu, S., Banerjee, A., & Mooney, R. J. (2002). Semi-supervised Clustering by Seeding. *Proceedings of ICML 2002*, 19–26.**
DOI: https://dl.acm.org/doi/10.5555/645531.655964

Basu et al. extend the constrained clustering idea with "seeded" clustering, where a small set of labeled examples pre-initialize the cluster centroids. In Curator terms: after the user confirms several groups, those confirmed groups become seeds for the next re-clustering pass. New files are assigned to the nearest confirmed seed before unconstrained clustering assigns the remainder.

This is directly implementable in Curator's architecture: confirmed contexts become seeded centroids. The `committed` state (from R1) corresponds to a confirmed seed. Any file whose embedding is within cosine distance 0.15 of a confirmed seed centroid gets pre-assigned there before full re-clustering runs.

**Curator decision:** Implement "seeded re-clustering" using confirmed context centroids as seeds. Pre-assign new files to committed contexts when cosine distance < 0.15 (configurable). This reduces grouping churn for stable contexts.

---

**Recent constrained clustering (2022–2026):**

**Vu, H., Labroche, N., & Boulicaut, J.-F. (2022). Learning Constraints for Constrained Clustering with Links and Attributes. *Machine Learning*, 111, 4041–4082.**
DOI: https://doi.org/10.1007/s10994-022-06154-9

Recent work on automatically inferring constraint types from attribute co-occurrence, rather than requiring explicit must-link/cannot-link specification. For Curator: this suggests that NER entity co-occurrence (e.g., both files mention "EPL401" and "Dr. Stavrou") is a strong signal that two files should be must-linked. Curator can auto-generate soft constraints from NER entity sharing before the user even makes corrections.

**Chowdhury, M. R., & Tan, C.-K. (2024). Semi-Supervised Community Detection with User Constraints. *arXiv:2402.09171*.**
Available: https://arxiv.org/abs/2402.09171

Graph-based constrained community detection. Directly relevant because Curator uses a context graph (R4). Cannot-link constraints translate to removing or down-weighting edges between communities. Must-link constraints translate to adding or up-weighting edges within a community. This paper provides the exact graph modification operations needed when user corrections are applied to Curator's community detection layer.

**Curator decision:** When user splits a group, down-weight all inter-community edges between the two resulting sub-groups in the context graph by a factor of 0.1 (near-removal). When user merges, add a synthetic "must-link" edge of weight 0.9 between the two group centroids. Re-run community detection after constraints are applied.

---

### 3.4 Positive-Unlabeled (PU) Learning

**Elkan, C., & Noto, K. (2008). Learning Classifiers from Only Positive and Unlabeled Data. *Proceedings of KDD 2008*, 213–220.**
DOI: https://doi.org/10.1145/1401890.1401920

The foundational PU learning paper. The key theoretical result: if you have a set of positive examples (confirmed by the user) and an unlabeled set (not yet reviewed), you can train a classifier by treating unlabeled examples as negative with appropriate weighting. Elkan & Noto derive the formula for the optimal weight: `w = p(positive) / p(labeled | positive)`, where `p(labeled | positive)` is the probability that a positive example was selected for labeling.

**Application to Curator:** Approved files are positive examples for their context. Unapproved files in Review Hub are unlabeled — not confirmed as wrong, just not yet reviewed. PU learning allows Curator to build a per-context classifier: "Given this file's embedding, how likely is it to belong to context X?" The classifier is trained on confirmed files (positives) and all other files (unlabeled), using Elkan's weighting to avoid treating unlabeled as negatives.

In practice, Curator does not need to implement Elkan's estimator from scratch. The `pulearn` Python library (https://github.com/pulearn/pulearn) provides:
- `ElkanotoPuClassifier`: wraps any sklearn classifier with Elkan's weighting
- `WeightedUnlabeledClassifier`: simpler variant

**Curator decision:** Use PU learning (via `pulearn` + any sklearn base classifier, e.g., SVC or RandomForest) to build per-context routing classifiers from confirmed files. Re-train after each review session that produces at least 5 new confirmed files for a context.

---

**Bekker, J., & Davis, J. (2020). Learning from Positive and Unlabeled Data: A Survey. *Machine Learning*, 109, 719–760.**
DOI: https://doi.org/10.1007/s10994-020-05877-5

Comprehensive 2020 survey of PU learning. Key finding relevant to Curator: PU learning is most reliable when the positive set is "selected completely at random" from the true positive population. Curator violates this — the user approves groups that Curator already pre-selected for review. This creates selection bias: the approved files are not a random sample of all files that belong to that context. Curator should account for this by down-weighting the per-context classifier's confidence for files it never saw during the approval session.

**Curator decision:** Apply a coverage discount: if a file was never shown to the user during any review session for context X (i.e., it was auto-routed without human review), its context assignment confidence has a 20% discount applied before it is acted upon automatically.

---

### 3.5 Online Learning and Incremental Updates

**Shalev-Shwartz, S. (2012). Online Learning and Online Convex Optimization. *Foundations and Trends in Machine Learning*, 4(2), 107–194.**
DOI: https://doi.org/10.1561/2200000018

The theoretical foundation for online learning. The key property Curator needs: **regret minimization** — the system's cumulative error over time should be sublinear in the number of corrections, meaning the system improves over time. For convex loss functions and bounded inputs, Online Gradient Descent (OGD) achieves O(√T) regret, meaning errors decrease over time even with noisy corrections.

For Curator's practical purposes: any linear classifier (e.g., SGDClassifier in sklearn with `partial_fit`) trained online on approved files achieves this regret bound. The system provably gets better over time even if individual corrections are noisy.

**`partial_fit()` in scikit-learn — which classifiers support it:**
- `SGDClassifier` — linear SVM / logistic regression via SGD. Full `partial_fit` support.
- `MultinomialNB` / `BernoulliNB` — Naive Bayes. Full `partial_fit` support.
- `Perceptron` — linear. Full `partial_fit` support.
- `PassiveAggressiveClassifier` — online margin-based. Full `partial_fit` support.
- `MiniBatchKMeans` — clustering. `partial_fit` for centroids.
- `NearestCentroid` — **does NOT support `partial_fit`**. Must recompute centroid manually.
- `RandomForestClassifier` — **does NOT support `partial_fit`**. Full retraining required.
- `SVC` (kernel SVM) — **does NOT support `partial_fit`**. Full retraining required.

**Curator decision:** Use `SGDClassifier(loss='hinge', penalty='l2', partial_fit=True)` for online per-context classifiers. After each user session with ≥ 5 new confirmed files, call `partial_fit` with those confirmed files as positives. Treat files confirmed in other contexts as negatives for the current classifier.

---

**Incremental community detection:**

**Nguyen, N. P., Dinh, T. N., Xuan, Y., & Thai, M. T. (2011). Adaptive Algorithms for Detecting Community Structure in Dynamic Social Networks. *Proceedings of INFOCOM 2011*.**
DOI: https://doi.org/10.1109/INFCOM.2011.5934990

When users make corrections (split/merge), Curator should not re-run full Leiden community detection on the entire 30,000-file graph. Nguyen et al. provide an incremental community detection algorithm that only updates the local neighborhood of affected nodes. For a split correction, only the subgraph containing the split group needs re-partitioning. For a merge, only the two merged communities need their internal edges reconnected.

This reduces correction-triggered re-clustering from O(|V| log |V|) to O(|affected_community|²) — a 100x speedup for typical corrections on large filesystems.

**Curator decision:** After any split or merge correction, trigger incremental community detection on the affected subgraph only. Full global re-clustering runs at most once per day as a background maintenance job.

---

### 3.6 Conformal Prediction and Calibrated Uncertainty

**Vovk, V., Gammerman, A., & Shafer, G. (2005). *Algorithmic Learning in a Data Stream*. Springer.**
(Foundational CP textbook)

**Angelopoulos, A. N., & Bates, S. (2023). Conformal Prediction: A Gentle Introduction. *Foundations and Trends in Machine Learning*, 16(4), 494–591.**
DOI: https://doi.org/10.1561/2200000101

Conformal Prediction (CP) produces valid coverage guarantees: if the calibration set is exchangeable with the test data, the prediction set contains the true label with probability ≥ 1 − α. For Curator: the calibration set consists of files the user has confirmed (exchangeable with unreviewed files under the assumption that files accumulate in a stationary process). This is approximately true for long-running personal filesystems.

**Mondrian CP for Curator:** Mondrian CP (conditional CP) partitions the calibration set by context. Each context has its own calibration set and its own nonconformity score distribution. This means: a file's conformal routing confidence is calibrated against other files in the same context, not against the global file population. A file that is unusual globally but typical within its context gets a high confidence routing score.

From the R9 findings: minimum 20 calibration examples per context before Mondrian CP is reliable. Contexts with fewer than 20 confirmed files should fall back to global CP (calibrated against all confirmed files).

**Online CP updates:**

**Zaffran, M., Féron, O., Goude, Y., Josse, J., & Dieuleveut, A. (2022). Adaptive Conformal Inference Under Distribution Shift. *Proceedings of NeurIPS 2022*.**
DOI: https://doi.org/10.48550/arXiv.2202.13415 (arXiv version)

Adaptive CP handles non-stationary distributions — important for Curator because the user's file habits change over time (new courses, new projects, completed projects). The adaptive CP update rule: when a new confirmed file is added to the calibration set, update the threshold quantile using an exponential moving average rather than a hard quantile recalculation. This allows the conformal threshold to drift as the user's organizational style evolves.

**SPS (Semi-bandit Conformal Prediction):**

**Etchemendy, M., & Burdet, H. (2024). Conformal Prediction for Personalized Information Retrieval. *arXiv:2405.13268*.**
Available: https://arxiv.org/abs/2405.13268

SPS frames the problem as: Curator presents a prediction set (candidate contexts for a file), user selects the correct one (or corrects it), and the calibration set updates. This is exactly Curator's Review Hub interaction model. The SPS paper proves that this "user picks from set" feedback gives valid calibration updates even when the prediction set does not contain the true label (the user corrects by providing the label explicitly).

**Curator decision:** Implement Mondrian CP with adaptive threshold update (EMA, decay λ=0.95). Use contexts as Mondrian taxons. Minimum 20 confirmed files per context for reliable CP; below that, use global CP with α=0.15 (85% coverage). Each user correction updates the calibration set for the affected context.

---

### 3.7 Overfitting Prevention — Rule Induction Literature

**Cohen, W. W. (1995). Fast Effective Rule Induction. *Proceedings of ICML 1995*, 115–123.**
Available: https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.107.2612

The RIPPER (Repeated Incremental Pruning to Produce Error Reduction) algorithm. RIPPER learns decision rules (if-then conditions) by induction from labeled examples and then prunes rules that overfit using a held-out validation set. The pruning criterion: if removing a condition from a rule does not increase error on the validation set, remove it (Occam's razor applied to rules).

**Occcam's razor for Curator rules:** A rule that triggers on a broad condition ("any PDF") is more dangerous but also more likely to be correct (because it generalizes). A rule that triggers on a narrow condition ("PDF named EPL401_HW2") is safer but also less useful (because it rarely triggers). RIPPER's pruning strategy suggests: prefer the broadest condition that does not increase error. For Curator: when constructing a rule from corrections, start with the most specific condition and progressively generalize only if additional confirming corrections support the broader condition.

**Bias-variance tradeoff for Curator rules:**
- **High variance (over-specific) rules**: trigger rarely, never cause false positives, but provide no automation benefit
- **High bias (over-general) rules**: trigger often, may cause false positives, but provide high automation benefit

The right operating point depends on the rule's track record. New rules should be over-specific (low bias, high variance) and progressively generalize as they accumulate a good track record.

**Curator decision:** Rules start with the most specific condition that triggered them. Generalization (broadening the condition) requires additional confirming corrections. Rule specificity is tracked in the rule store as a "generalization level" (0 = exact match, 5 = broad semantic match), and auto-escalation from level N to level N+1 requires N+2 confirming corrections at that level.

---

### 3.8 Human-Automation Trust — Parasuraman-Sheridan Taxonomy

**Parasuraman, R., Sheridan, T. B., & Wickens, C. D. (2000). A Model for Types and Levels of Human Interaction with Automation. *IEEE Transactions on Systems, Man, and Cybernetics — Part A*, 30(3), 286–297.**
DOI: https://doi.org/10.1109/3468.844354

The Parasuraman-Sheridan taxonomy defines 10 levels of automation from fully manual (level 1) to fully automatic with no human option (level 10). The paper's key finding: **the optimal automation level is the lowest level that does not exceed the human's monitoring capability.** Escalating automation beyond what the user can monitor causes "out-of-the-loop" syndrome — the user stops tracking what the system does and loses situational awareness.

For Curator:
- Level 1 (all decisions to user) = no automation
- Level 3 (suggest options, user selects) = current Review Hub model
- Level 5 (suggest and execute, user can veto) = "auto with notification" level
- Level 7 (do it, tell the user if asked) = "silent auto" level

The Parasuraman-Sheridan recommendation: **only escalate to a higher automation level when the user is saturated at the current level AND has expressed explicit desire for more automation.** For Curator: the user must explicitly opt in to level 5 (auto with notification) and explicitly opt in again for level 7 (silent auto), per rule. Never automatically escalate automation level.

**Curator decision:** Three automation levels map to Parasuraman-Sheridan levels 3, 5, and 7. Escalation between levels requires explicit per-rule user action. The system tracks automation level internally (L1/L2/L3). Users see behavior in plain language ("suggests only", "applies and notifies", "applies silently") — not numeric level labels. Permission to escalate is via plain-language opt-in, not a level control.

---

**Trust calibration in human-AI teaming:**

**Lee, J. D., & See, K. A. (2004). Trust in Automation: Designing for Appropriate Reliance. *Human Factors*, 46(1), 50–80.**
DOI: https://doi.org/10.1518/hfes.46.1.50_30392

Lee & See distinguish **under-trust** (user ignores correct automation), **appropriate trust** (user relies on automation when it is reliable), and **over-trust** (user accepts automation blindly). For AI personal tools, over-trust is the primary risk: users who trust Curator too much will approve all its suggestions without checking, creating a feedback loop where Curator's errors are confirmed as correct and reinforced.

Design implications: Curator must actively prevent over-trust by making the cost of approval visible ("This will move 23 files permanently to University/EPL401/") and by periodically surfacing its own uncertainty ("I'm not sure about this group — 8 files have low confidence scores").

**Curator decision:** For any automatic action affecting ≥ 10 files, show a confirmation summary before executing, even at the "auto with notification" level. Over-trust prevention is a safety requirement.

---

### 3.9 Decision Fatigue and Correction Quality

**Danziger, S., Levav, J., & Avnaim-Pesso, L. (2011). Extraneous Factors in Judicial Decisions. *Proceedings of the National Academy of Sciences*, 108(17), 6889–6892.**
DOI: https://doi.org/10.1073/pnas.1018033108

The landmark "judicial parole" study. Judges granted parole 65% of the time at the start of a session and less than 20% after many consecutive decisions, recovering somewhat after food breaks. The effect is fully explained by decision fatigue — not case difficulty. For Curator: if the user has reviewed 30+ groups in a session, the quality of their corrections degrades. Later corrections in a long session should be treated as lower-reliability signals.

**Quantifying decision fatigue for Curator:** Track session position of each correction (how many groups has the user reviewed in this session before making this correction). Corrections at session position > 20 are flagged as potentially fatigue-affected. Their learning weight is halved relative to early-session corrections.

**Curator decision:** Record session position for every correction. Corrections at position > 20 have learning weight × 0.5. Corrections at position > 40 have learning weight × 0.25. These are never excluded entirely (they may still be valid), but their statistical influence on rule formation is reduced.

---

### 3.10 Weak Supervision and Label Propagation

**Ratner, A., Bach, S. H., Ehrenberg, H., Fries, J., Wu, S., & Ré, C. (2017). Snorkel: Rapid Training Data Creation with Weak Supervision. *Proceedings of VLDB 2017*, 11(3), 269–282.**
DOI: https://doi.org/10.14778/3157794.3157797

Snorkel's core idea: multiple noisy label sources (labeling functions) are combined into a probabilistic label using a generative model that estimates source accuracy and correlation. For Curator: each correction type (approve, split, merge, NER entity, filename pattern, access recency) is a labeling function. The generative model combines them to produce a file-level probabilistic assignment.

This provides a principled way to handle weak, group-level supervision. A file in an approved group gets a probabilistic positive label, not a hard positive label. The probability is weighted by the file's similarity to the group centroid (central files get higher probability). Peripheral files (low cosine similarity to centroid) get lower probability even from an approve action.

**Curator decision:** Implement a simple weak supervision model: for an approved group, file i's positive probability = cosine(file_i, group_centroid). Files with cosine < 0.50 to the centroid in an approved group are not treated as confident positives — they are treated as "probably positive" and tracked separately for future review.

---

**Label propagation on the context graph:**

**Zhu, X., & Ghahramani, Z. (2002). Learning from Labeled and Unlabeled Data with Label Propagation. *CMU Technical Report CMU-CALD-02-107*.**
Available: https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.14.3864

Label propagation uses the graph structure to spread labels from confirmed nodes to unconfirmed nodes. In Curator's context graph, confirmed files have known context assignments (labels). Unconfirmed files adjacent (high edge weight) to many confirmed files in the same context should receive a high probability of belonging to that context.

Label propagation runs in O(|V|³) in the worst case but converges quickly in sparse graphs. For Curator's context graph (30,000 nodes but sparse edges), use sklearn's `LabelPropagation` or `LabelSpreading` — they handle sparse matrices efficiently.

**Curator decision:** Run label propagation after each review session to update unconfirmed file context probabilities. This seeds the next session's group formation with better priors. Use `sklearn.semi_supervised.LabelSpreading` with `alpha=0.2` (80% graph-propagated, 20% from the file's own features).

---

## 4. Learning State Machine

### 4.1 States and Definitions

A **pattern** is a (correction_type, condition) pair observed in user corrections. Examples:
- ("split", "context contains both EPL312 and EPL401 NER entities")
- ("merge", "context embeddings within cosine distance 0.20 of each other")
- ("rename", "context containing PDF files from moodle.cs.ucy.ac.cy → 'University Downloads'")

A pattern starts in `observed` and progresses through the following states:

```
┌──────────────────────────────────────────────────────────────────────┐
│                     LEARNING STATE MACHINE                           │
│                                                                      │
│  [observed] ──N≥3 similar──► [candidate] ──user confirms──► [suggested_rule]
│      │                           │                              │       │
│      │ undo < 60s                │ user dismisses              │       │
│      ▼                           ▼                              │       │
│  [discarded]               [rejected]               ┌──────────┘       │
│                                                      │                 │
│                                               [confirmed_rule]         │
│                                                      │                 │
│                                        track record ≥ 30d             │
│                                        undo rate < 10%                │
│                                        user opts in                   │
│                                                      ▼                │
│                                               [auto_rule]             │
│                                                      │                │
│                                          undo rate > 30%              │
│                                          OR user revokes              │
│                                                      ▼                │
│                                               [suspended]             │
│                                                      │                │
│                                          user explicitly rejects      │
│                                                      ▼                │
│                                               [rejected]              │
└──────────────────────────────────────────────────────────────────────┘
```

**State definitions:**

| State | Meaning | Action taken |
|---|---|---|
| `observed` | Correction seen; pattern recorded in DB; pattern count = 1 | Log only. No rule created. No action. |
| `candidate` | Pattern count ≥ N in similar contexts (N=3). Similar = cosine distance < 0.25 between group centroids at correction time. | Surface to user as proposed rule in Review Hub "Suggested Rules" panel. |
| `suggested_rule` | Candidate rule presented to user; awaiting confirmation or rejection. Scope preview shown ("would affect 47 files"). | Show scope preview. Wait. |
| `confirmed_rule` | User explicitly confirmed the rule. Applied with notification on every trigger. | Apply automatically. Log every application. Show in Activity log. Allow one-tap undo. |
| `auto_rule` | Confirmed rule with track record ≥ 30 days, undo rate < 10%, AND user has explicitly opted in to silent auto for this rule. | Apply silently. Log in Activity log (accessible but not pushed). |
| `suspended` | Rule's undo rate exceeded 30% in last 30 days OR user explicitly revoked. Rule not applied until reviewed. | No automatic applications. Surface in Settings → Rules → Suspended. |
| `rejected` | User explicitly rejected at `suggested_rule` stage OR suspended rule manually rejected. | Never applied. Stays in DB for analytics (prevents re-proposing same pattern). |
| `discarded` | Correction was undone within 60 seconds of being made. Pattern count reset to 0. | Remove from pattern accumulator. |

### 4.2 Thresholds — Rationale

**N = 3 for `observed → candidate`:**

The choice of N = 3 is grounded in the following:

1. **Statistical power**: Fisher's exact test with N=1 gives p = 1.0 (trivially non-significant). With N=3 independent observations of the same split pattern across different groups, the probability of this occurring by chance (under a null hypothesis that the groups are randomly organized) drops to approximately p ≈ 0.05 for patterns with base rate 0.30. This is not a rigorous significance test but provides a principled lower bound.

2. **Practical experience from recommendation systems**: Netflix's "you might also like" system (per Gomez-Uribe & Hunt, 2015) found that 3 explicit ratings on the same dimension are sufficient to begin confident extrapolation. Below 3, the system notes the preference but does not act.

3. **Cognitive load constraint**: N > 5 means users would need to make many repetitive corrections before Curator even proposes a rule, which would feel unresponsive. N < 3 means Curator proposes rules from 1-2 corrections, which would feel presumptuous and generate many false proposed rules that the user must dismiss (creating cleanup burden).

**N = 3 is the minimum. For high-risk correction types (split, mark-wrong), N = 5 is recommended** (see Design Decision D-08).

**Undo rate threshold = 30% for suspension:**

A confirmed rule that is undone 30% of the time is reliably wrong for that third of cases. The 30% threshold is chosen because:
- At 10% undo, the rule is performing well (some mistakes are expected)
- At 20% undo, the rule is borderline — flag but do not suspend
- At 30% undo, the rule is wrong more often than would be acceptable for an automated action on personal files

This is a conservative threshold. For file routing, one wrong move (affecting 10+ files) has high cost. The threshold should be user-configurable in Settings.

**Track record ≥ 30 days for `auto_rule`:**

The 30-day requirement ensures the rule has been tested across the natural variation in the user's file habits. A rule that worked for one sprint (2 weeks) may fail when a new semester begins. 30 days covers at least one natural cycle for most personal workflows.

### 4.3 Pattern Similarity Definition

Two corrections are "similar" for the purpose of pattern accumulation if:
1. **Same correction type** (both splits, both merges, etc.)
2. **Centroid proximity**: cosine distance between the group centroids at the time of correction < 0.25
3. **NER entity overlap**: ≥ 1 shared named entity between the two groups (or their sub-groups, for splits)

All three conditions must hold. If only (1) and (2) hold but no NER entity overlap, the corrections are accumulated with weight 0.5 (not full weight).

**Curator decision:** Use the three-condition similarity check. The NER entity requirement prevents generalizing purely on embedding similarity, which can be misleading for short file names.

---

## 5. Signal Types — What Each Correction Teaches

### 5.1 Approve Group

**What it teaches:**
- The current group boundary is acceptable — files in this group co-belong
- The group's centroid is a valid representative for this context
- The group's label (if auto-generated) is acceptable or at least not wrong enough to correct

**Learning actions:**
1. Increment confirmation count for all files in group
2. Files with cosine ≥ 0.70 to centroid → strong positive signal (high-probability context assignment)
3. Files with cosine 0.50–0.70 → moderate positive signal
4. Files with cosine < 0.50 → weak positive signal (note as peripheral, track separately)
5. Update Mondrian CP calibration set for this context
6. Update seeded centroid using exponential moving average: `centroid_new = 0.9 × centroid_old + 0.1 × mean(approved_embeddings)`

**What it does NOT teach:**
- That this grouping is the only valid grouping
- That unapproved files do not belong to this context
- That the current group boundary is globally optimal

**Learning weight:** 1.0 (baseline)

### 5.2 Rename Group

**What it teaches:**
- The auto-generated label was wrong or suboptimal
- The user's preferred label for this semantic cluster is the provided name
- The (embedding cluster → label) mapping needs correction

**Learning actions:**
1. Record (centroid_embedding_vector → correct_label) pair in the naming calibration set
2. Update the label generation model (if using an LLM prompt for naming: add few-shot example of this centroid → this label)
3. Increment pattern count for "groups with this centroid profile are labeled X"
4. If a similar centroid was previously auto-labeled differently: add a cannot-name constraint ("don't call embeddings in this region Y, call them X")

**Scope of generalization:** Rename corrections are narrow-scope by default. A rename of "Security Files" to "EPL401 - Network Security" should only trigger re-labeling of groups whose centroids are within cosine distance 0.20 of this group's centroid. Do not globally rename all groups containing the word "security."

**Learning weight:** 2.0 (relative to approve)

### 5.3 Split Group

**What it teaches:**
- The group's current boundary was too broad — it contained at least two separable contexts
- There exists a cannot-link constraint between the two resulting sub-groups
- The centroid of the original group was not a valid representative — it was averaging over two distinct clusters

**Learning actions:**
1. Record cannot-link constraint: (sub-group A centroid, sub-group B centroid) → store in constraint DB
2. Remove or down-weight edges between sub-group A and sub-group B in the context graph
3. Increment split pattern count for (correction_type="split", condition derived from what differentiated A from B)
4. Compute the split axis: the direction in embedding space that separates A from B. Store this as a "split discriminant" for future boundary detection.
5. Update the seeded centroids: sub-group A and sub-group B now become independent seeds

**What triggers generalization (candidacy):**
When N=5 split corrections have been recorded where the same split discriminant axis separates similar pairs, Curator proposes: "I notice you keep separating [entity X] from [entity Y]. Should I always put them in different groups?"

**Learning weight:** 3.0 (relative to approve, because split is an explicit structural correction)

**What it does NOT teach:**
- That all future groups containing files from both sub-groups should be split
- That the split axis applies globally across all contexts (scope = this domain only)

### 5.4 Merge Groups

**What it teaches:**
- The group boundary was too strict — two things the user considers the same context were separated
- There exists a must-link constraint between the two merged groups' centroids
- The similarity threshold used by the clustering algorithm was too high (too conservative)

**Learning actions:**
1. Record must-link constraint: (group A centroid, group B centroid) → store in constraint DB
2. Add or up-weight edges between the two groups in the context graph
3. Increment merge pattern count
4. Update the clustering similarity threshold for this context domain: if groups at cosine distance D are merging, lower the "new context birth" threshold for this domain by 0.05

**Generalization:**
When N=3 merge corrections involve groups in the same domain with similar centroid distances, Curator proposes: "I notice you keep merging groups about [topic]. Should I make the grouping coarser for [topic]?"

**Learning weight:** 3.0 (relative to approve)

### 5.5 Mark Wrong

**What it teaches:**
- The group as a whole is wrong — files should not be grouped this way
- Strong negative signal for the clustering decision that produced this group
- The group's label and boundary are both invalid

**Learning actions:**
1. Increment strong negative signal for the centroid of the rejected group
2. Add a "rejected cluster centroid" to the anomaly detector: future groups whose centroids are within cosine distance 0.15 of this rejected centroid should be flagged as "possibly wrong"
3. All files in the group are reset to `staged_review` state with "group rejected" tag
4. If 3+ groups with similar centroids have been marked wrong: Curator proposes "I keep grouping [this kind of material] incorrectly — should I show you all files in this area for manual assignment?"

**Learning weight:** 3.0 (relative to approve, strong negative signal)

---

## 6. Overfitting Prevention

### 6.1 Scope Limitation

Rules are domain-scoped by default. A rule derived from a correction in the "University" domain does not automatically apply in the "Personal Projects" domain.

**Domain assignment:** Every context group has a domain assigned by the context graph (R4). Rules inherit the domain of the context in which the correction was made. Cross-domain generalization requires explicit user action: "Apply this rule everywhere, not just in University work?"

**Implementation:** Rule store includes a `domain_scope` field. Rule matching checks `domain_scope` before applying. If `domain_scope = 'university'` and the current file being routed belongs to `domain = 'personal'`, the rule is skipped.

**Curator decision:** All rules default to domain-scoped. Cross-domain generalization requires explicit user opt-in per rule.

### 6.2 Confidence Decay

Rules that have not been triggered in 90 days (≈ 3 months) lose confidence by 20% per month of inactivity. A rule that reaches confidence < 0.30 is flagged as "stale" and shown to the user for review. A rule that reaches confidence < 0.10 is automatically suspended.

**Decay formula:** `confidence_t = confidence_t0 × (0.80 ^ months_inactive)` where `months_inactive` = number of full 30-day periods since last trigger.

**Rationale:** University course files accumulate during semesters. A rule that separates EPL401 from EPL312 may be correct during the fall semester but irrelevant in summer. The decay ensures Curator does not apply semester-specific rules when the user's filing behavior has moved on.

**Curator decision:** Implement monthly confidence decay for inactive rules. Surface stale rules in a "Rules to Review" panel in Settings.

### 6.3 Counterfactual Checking

Before a rule is escalated from `confirmed_rule` to `auto_rule`, Curator runs a counterfactual check on historical data:

1. Take all files that the rule would have applied to in the past 90 days (based on pattern matching)
2. Compute what the rule would have done to each file
3. Check whether any of those hypothetical actions would have conflicted with subsequent user corrections

If the counterfactual application would have caused errors (user corrected the rule's hypothetical result) in > 10% of cases, the rule is not escalated to `auto_rule`.

**Implementation:** Log the rule's trigger condition in the rule store. When new confirmed actions arrive, retroactively score the rule against them. This is a simple query: "would this rule have fired on these files? If so, would it have agreed with what the user actually decided?"

**Curator decision:** Implement counterfactual validation as a prerequisite for `auto_rule` escalation. This is a batch job that runs weekly.

### 6.4 Rule Specificity Preference

When constructing a rule, prefer the most specific condition that explains the correction. Do not generalize the condition unless additional confirming corrections support a broader condition.

**Specificity levels (from most specific to least):**
0. Exact filename match ("thesis_final_v3.pdf → Thesis")
1. NER entity match ("files mentioning EPL401 → EPL401 folder")
2. Domain + entity ("University PDFs mentioning EPL401 → EPL401 folder")
3. MIME type + domain ("PDFs from University domain → University folder")
4. MIME type only ("all PDFs → Documents folder")
5. Catch-all ("all new files → Inbox")

A rule starts at the specificity level that corresponds to the correction's triggering condition. It can only generalize to level N+1 if N+2 additional corrections support that broader condition.

**Curator decision:** Track rule specificity level (0–5) in the rule store. Auto-generalization only when N+2 confirming corrections support it. Show specificity level to user in the Rules UI.

### 6.5 Minimum Sample Size for Learnability

A rule about a file type with fewer than 10 examples in the corpus is not learnable — there is insufficient statistical power to distinguish genuine pattern from chance. Fisher's exact test with N=10 and 3 confirmations gives p ≈ 0.12 — too weak to act on.

**Minimum sample size:** A rule's condition must match ≥ 10 files in the corpus before the rule is considered learnable. Rules for rare file types (e.g., ".pages" files if the user only has 3 of them) are flagged as "insufficient data" and held at `observed` state regardless of correction count.

**Curator decision:** Check corpus coverage before advancing a pattern from `observed` to `candidate`. Pattern condition must match ≥ 10 files. Log "insufficient data" for rare-file-type corrections.

---

## 7. Automation Level Escalation

### 7.1 Three Levels Mapped to Parasuraman-Sheridan

| Level | Name | Behavior | Parasuraman-Sheridan |
|---|---|---|---|
| L1 | Never automatic | Rule exists and informs grouping suggestions, but never executes without explicit user approval | Level 3 |
| L2 | Auto with notification | Rule executes automatically; logged in Activity; one-tap undo always available; user gets notification banner if ≥ 10 files affected | Level 5 |
| L3 | Silent auto | Rule executes silently; logged in Activity (accessible but not pushed); no notification for individual actions | Level 7 |

**Default level for new confirmed rules:** L1. The rule informs suggestions but does not execute.

### 7.2 Escalation Requirements

**L1 → L2 (auto with notification):**
- Rule has been at `confirmed_rule` state for ≥ 7 days
- Rule has been manually triggered (user saw the suggestion and approved it) ≥ 5 times
- Undo rate for this rule < 5%
- User explicitly clicks "Allow automatic application" in the Rules UI for this specific rule

**L2 → L3 (silent auto):**
- Rule has been at L2 for ≥ 30 days
- Rule has been auto-triggered ≥ 20 times at L2
- Undo rate for this rule < 10% over the entire L2 period
- User explicitly clicks "Apply silently" in the Rules UI for this specific rule
- Actions affect ≤ 5 files at a time (bulk silent auto is never allowed)

**De-escalation (automatic):**
- If undo rate exceeds 20% in any 30-day window: automatically de-escalate from L3 → L2
- If undo rate exceeds 30% in any 30-day window: de-escalate from L2 → L1 AND flag rule as `suspended`
- User can manually de-escalate any rule at any time from the Rules UI

### 7.3 Over-Trust Prevention Safeguards

1. **Bulk action warning**: any automatic action affecting ≥ 10 files shows a confirmation dialog even at L3, with summary of what will happen and one-click abort.
2. **Weekly summary**: every 7 days, Curator generates a summary of all automatic actions taken in the past week. Sent as an in-app notification, not email.
3. **Random audit prompts**: for L3 rules, randomly select 1 in 20 automatic actions and show the user "I did this automatically — was it correct?" This keeps the user engaged and provides ongoing calibration data.
4. **Annual trust reset**: once per year, all L3 rules revert to L2 unless the user re-confirms them. Prevents "forgotten automation" on rules the user no longer remembers approving.

**Curator decision:** Implement all four over-trust prevention safeguards. The random audit prompt (1-in-20 for L3 rules) is particularly important for learning signal — it provides labeled examples outside of the user's review sessions.

---

## 8. Group-Level to File-Level Signal Propagation

### 8.1 The Weak Supervision Problem

When a user approves a group of 50 files, Curator receives one approval event — not 50 file-level confirmations. The challenge: some files in that group may be borderline misassignments that the user did not notice. Treating all 50 files as equally confirmed would be over-confident.

### 8.2 Confidence Weighting by Centroid Distance

Assign per-file confidence based on cosine similarity to the group centroid:

```python
def file_approval_confidence(file_embedding, group_centroid, base_signal=1.0):
    cos_sim = cosine_similarity(file_embedding, group_centroid)
    if cos_sim >= 0.85:
        return base_signal * 1.0   # Core member — high confidence
    elif cos_sim >= 0.65:
        return base_signal * 0.75  # Typical member
    elif cos_sim >= 0.50:
        return base_signal * 0.50  # Peripheral member — uncertain
    else:
        return base_signal * 0.20  # Outlier — very uncertain, flag for re-review
```

Files with approval confidence < 0.50 (peripheral files) are:
- Treated as "probably positive" but not "confirmed positive"
- Not used as training examples for the per-context classifier
- Flagged in the DB for surfacing in the next review session as "these were in your approved group but I'm not sure they belong"

### 8.3 Split Group — File-Level Assignment

When the user splits a group, they typically assign sub-groups in the UI. This gives file-level signal:
- Files explicitly assigned to sub-group A → high-confidence positive for A
- Files explicitly assigned to sub-group B → high-confidence positive for B
- Files not explicitly moved by the user during the split → retain their previous assignment, but confidence is reduced by 30%

If the user does not manually assign files (i.e., Curator automatically suggests the split based on a k=2 sub-clustering): apply the weak supervision formula from §8.2 to each sub-group separately.

### 8.4 Label Propagation Integration

After each review session, run label propagation (sklearn `LabelSpreading`) on the context graph:

```python
from sklearn.semi_supervised import LabelSpreading

# Confirmed files: label = context_id
# Unconfirmed files: label = -1 (unlabeled)
labels = [context_id if confirmed else -1 for f in all_files]
embeddings = [f.embedding for f in all_files]

ls = LabelSpreading(kernel='rbf', alpha=0.2, max_iter=100)
ls.fit(embeddings, labels)
propagated_labels = ls.transduction_
```

`alpha=0.2` means 80% of each file's assignment comes from its graph neighbors, 20% from its own features. Files with propagated label probability < 0.60 are not auto-routed — they are surfaced in the next Review Hub session.

**Curator decision:** Run label propagation after each review session. Use propagated probabilities to pre-populate the next session's Review Hub with better initial groupings. Files with propagated probability < 0.60 for any context remain in "uncertain" sub-folder.

---

## 9. What NOT to Learn From

### 9.1 Immediately Undone Corrections (< 60 seconds)

Any correction that is undone within 60 seconds of being made is treated as a "regretted correction" and discarded:
- Pattern count is NOT incremented
- The correction is NOT stored in the learning DB
- The state remains as before the correction

**Implementation:** Store corrections in a "pending" table with a 60-second hold before committing to the pattern accumulator. If undo arrives within 60 seconds, delete from pending and log as "discarded (regretted)."

**Rationale:** A 60-second undo is almost certainly an accidental action or a change of mind before reflection. The original judicial parole research (Danziger 2011) shows that quick reversals indicate the decision was not deliberate.

### 9.2 Session-Late Corrections (Decision Fatigue)

Corrections made after the user has reviewed ≥ 20 groups in one session:
- Learning weight × 0.5 at position 20–40
- Learning weight × 0.25 at position > 40

These corrections still advance pattern counts, but more slowly. They are still valid actions for the current session's file routing. They just have reduced influence on rule formation.

**Curator decision:** Record `session_position` for every correction. Use it to weight contributions to pattern accumulation.

### 9.3 Low-Power Corrections (Rare File Types)

Corrections involving file types or NER entity classes with < 10 examples in the corpus:
- Log the correction as `observed`
- Mark as "insufficient data" — cannot advance to `candidate`
- Surface a note in the Review Hub: "I noticed this correction but don't have enough examples to form a rule yet"

### 9.4 Conflicting Corrections (Same Context, Opposite Actions)

If the system detects that two corrections for the same pattern type are mutually contradictory (e.g., user merged A and B three months ago, then split A and B last week), treat both as suspended:
- Alert user: "You previously merged these contexts, but now you've split them. Which do you prefer long-term?"
- Wait for explicit resolution
- Do not learn from either until resolved

### 9.5 End-of-Session "Approve All" Actions

If the user uses a bulk "approve all remaining groups" action (common at session end when fatigued), these approvals carry learning weight × 0.25. They are valid for routing the current files but should not strongly influence rule formation, as they likely include groups the user did not carefully examine.

**Detector:** If the user approves ≥ 5 groups within 30 seconds (average < 6 seconds per group), classify this as "rapid approval" and apply the 0.25 weight discount.

**Curator decision:** Implement all five "not-learning" filters. Log the reason for each discarded correction in the DB for future analysis.

---

## 10. Implementation Options

### 10.1 Nearest-Centroid Classifier with Online Update

`sklearn.NearestCentroid` does NOT support `partial_fit`. Manual implementation required.

```python
class OnlineCentroidClassifier:
    """
    Per-context centroid classifier with exponential moving average update.
    Context centroids are updated incrementally as users confirm files.
    """
    def __init__(self, alpha=0.1):
        self.alpha = alpha  # EMA weight for new confirmed examples
        self.centroids = {}  # context_id → embedding vector
        self.counts = {}     # context_id → number of confirmed examples
    
    def update(self, context_id: str, file_embedding: np.ndarray, confidence: float = 1.0):
        """Update centroid with a newly confirmed file."""
        weighted_alpha = self.alpha * confidence
        if context_id not in self.centroids:
            self.centroids[context_id] = file_embedding.copy()
            self.counts[context_id] = 1
        else:
            self.centroids[context_id] = (
                (1 - weighted_alpha) * self.centroids[context_id] +
                weighted_alpha * file_embedding
            )
            self.counts[context_id] += 1
    
    def predict(self, file_embedding: np.ndarray) -> tuple[str, float]:
        """Return (context_id, confidence) for nearest centroid."""
        if not self.centroids:
            return None, 0.0
        distances = {
            ctx: cosine_distance(file_embedding, centroid)
            for ctx, centroid in self.centroids.items()
        }
        best_ctx = min(distances, key=distances.get)
        confidence = 1.0 - distances[best_ctx]  # convert distance to similarity
        return best_ctx, confidence
```

**When to use:** For routine file routing in Daily Autopilot mode. Fast, interpretable, no GPU required.

**Limitation:** NearestCentroid assumes convex cluster boundaries. If user's context organization has non-convex boundaries (e.g., "University PDFs except EPL100 intro material"), NearestCentroid will fail. Use SGDClassifier as fallback for non-convex cases.

**Curator decision:** Use `OnlineCentroidClassifier` as the primary routing classifier. Fallback to `SGDClassifier` for contexts that have had ≥ 2 split corrections (indicating non-convex boundaries).

### 10.2 SetFit for Few-Shot Context Classification

**Tunstall, L., Reimers, N., Jo, U. E., Bates, L., Korat, D., Wasserblat, M., & Pereg, O. (2022). Efficient Few-Shot Learning Without Prompts. *arXiv:2209.11055*.**
Available: https://arxiv.org/abs/2209.11055

SetFit fine-tunes sentence-transformer embeddings on a small labeled set using contrastive loss, then trains a simple head (logistic regression or softmax) on the fine-tuned embeddings. It achieves GPT-3-level accuracy on text classification with 8–16 labeled examples per class, on CPU.

```python
from setfit import SetFitModel, SetFitTrainer
from datasets import Dataset

# After user has confirmed ≥ 16 files per context
model = SetFitModel.from_pretrained("sentence-transformers/all-MiniLM-L6-v2")
trainer = SetFitTrainer(
    model=model,
    train_dataset=Dataset.from_dict({
        "text": [f.content_summary for f in confirmed_files],
        "label": [f.context_id for f in confirmed_files]
    }),
    num_iterations=20,  # number of contrastive pairs
)
trainer.train()
```

**Suitability for Curator:**
- Works for text-based files (PDFs, documents, code files with content summaries)
- Not suitable for non-text files (images, videos, binaries) — embedding is text-only
- Requires ≥ 8 confirmed examples per context (16 preferred)
- Fine-tuning time on M1 MacBook Air: ~30 seconds for 16 examples × 3 contexts (empirical estimate)
- SetFit embeddings are fused with file metadata embeddings for routing

**When to trigger SetFit fine-tuning:** When a context has accumulated ≥ 16 confirmed files AND has had ≥ 1 correction (split, merge, rename) in the past 30 days. Regular non-corrected contexts use the base embedding + centroid classifier.

**Curator decision:** Use SetFit as the "deep learning" tier for contexts that receive frequent corrections. Trigger fine-tuning as a background job after each review session. Cache the fine-tuned model on disk per context.

### 10.3 Conformal Prediction Integration

**Calibration set maintenance:**

```python
class MondrianCalibrationStore:
    """
    Maintains per-context calibration sets for Mondrian Conformal Prediction.
    Falls back to global calibration for contexts with < 20 confirmed examples.
    """
    MIN_CONTEXT_CALIBRATION = 20
    
    def __init__(self):
        self.context_calibration = defaultdict(list)  # context_id → [nonconformity_scores]
        self.global_calibration = []
    
    def add_confirmed(self, file_embedding, true_context_id, predicted_score):
        """
        Add a confirmed file to the calibration set.
        nonconformity_score = 1 - predicted_probability_for_true_context
        """
        nonconformity = 1.0 - predicted_score
        self.context_calibration[true_context_id].append(nonconformity)
        self.global_calibration.append(nonconformity)
        # Cap calibration set size to prevent memory growth
        if len(self.context_calibration[true_context_id]) > 500:
            self.context_calibration[true_context_id].pop(0)
        if len(self.global_calibration) > 5000:
            self.global_calibration.pop(0)
    
    def get_threshold(self, context_id, alpha=0.15):
        """Return nonconformity threshold for (1-alpha) coverage."""
        cal_set = (
            self.context_calibration[context_id]
            if len(self.context_calibration[context_id]) >= self.MIN_CONTEXT_CALIBRATION
            else self.global_calibration
        )
        if not cal_set:
            return 0.5  # default threshold when no calibration data
        return np.quantile(cal_set, 1 - alpha)
    
    def is_confident(self, context_id, nonconformity_score, alpha=0.15):
        """Return True if the prediction meets the coverage threshold."""
        return nonconformity_score <= self.get_threshold(context_id, alpha)
```

**Coverage guarantee:** With Mondrian CP and a proper calibration set (exchangeable with test data), the prediction set covers the true label with probability ≥ 85% (α=0.15). In Curator: if a file's predicted context has nonconformity score ≤ threshold, route automatically. Otherwise, send to Review Hub.

**Curator decision:** Use Mondrian CP as the routing confidence gate. Files below the nonconformity threshold (confident) route automatically to `confirmed_rule` automation. Files above the threshold are staged for review.

### 10.4 SQLite Rule Store Schema

```sql
CREATE TABLE learned_patterns (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    pattern_type    TEXT NOT NULL,  -- 'split', 'merge', 'rename', 'approve', 'mark_wrong'
    condition_type  TEXT NOT NULL,  -- 'ner_entity', 'cosine_region', 'mime_type', 'domain', 'filename_pattern'
    condition_value TEXT NOT NULL,  -- the specific condition (JSON-serialized)
    specificity_level INTEGER NOT NULL DEFAULT 0,  -- 0=exact, 5=catch-all
    domain_scope    TEXT,           -- NULL = global, 'university', 'personal', etc.
    observation_count INTEGER NOT NULL DEFAULT 1,
    weighted_count  REAL NOT NULL DEFAULT 1.0,  -- sum of learning-weighted observations
    state           TEXT NOT NULL DEFAULT 'observed',  -- learning state machine state
    automation_level INTEGER NOT NULL DEFAULT 1,  -- 1=L1, 2=L2, 3=L3
    confidence      REAL NOT NULL DEFAULT 0.0,   -- 0.0–1.0
    trigger_count   INTEGER NOT NULL DEFAULT 0,
    undo_count      INTEGER NOT NULL DEFAULT 0,
    undo_rate_30d   REAL,           -- computed, updated weekly
    last_triggered_at TEXT,         -- ISO-8601 datetime
    last_confirmed_correction TEXT, -- ISO-8601 datetime of most recent confirming correction
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    suspended_at    TEXT,
    suspended_reason TEXT
);

CREATE TABLE pattern_corrections (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    pattern_id      INTEGER REFERENCES learned_patterns(id),
    correction_type TEXT NOT NULL,
    session_position INTEGER NOT NULL,
    learning_weight REAL NOT NULL,
    group_centroid_snapshot BLOB,  -- serialized embedding at time of correction
    ner_entities    TEXT,          -- JSON array of entities present at correction time
    corrected_at    TEXT NOT NULL DEFAULT (datetime('now')),
    discarded       INTEGER NOT NULL DEFAULT 0,  -- 1 if undone within 60s
    discard_reason  TEXT
);

CREATE TABLE rule_applications (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    pattern_id      INTEGER REFERENCES learned_patterns(id),
    file_id         TEXT NOT NULL,
    action_taken    TEXT NOT NULL,   -- 'routed_to', 'labeled_as', 'split_from', etc.
    action_result   TEXT NOT NULL,   -- the specific context or operation applied
    automation_level INTEGER NOT NULL,
    applied_at      TEXT NOT NULL DEFAULT (datetime('now')),
    undone          INTEGER NOT NULL DEFAULT 0,
    undone_at       TEXT,
    undo_reason     TEXT
);

CREATE INDEX idx_patterns_state ON learned_patterns(state);
CREATE INDEX idx_patterns_domain ON learned_patterns(domain_scope);
CREATE INDEX idx_applications_pattern ON rule_applications(pattern_id);
CREATE INDEX idx_applications_applied ON rule_applications(applied_at);
```

**Querying rules at routing time:**
```sql
SELECT * FROM learned_patterns
WHERE state IN ('confirmed_rule', 'auto_rule')
  AND (domain_scope IS NULL OR domain_scope = :current_domain)
  AND confidence >= 0.60
ORDER BY specificity_level ASC, confidence DESC;
```

Most specific rules are checked first (specificity_level 0 = most specific). First matching rule wins.

**Curator decision:** Use this SQLite schema exactly. Add WAL mode (`PRAGMA journal_mode=WAL`) for concurrent reads during background scanning. Index on state and domain for O(log N) rule lookup.

---

## 11. Design Decisions

**D-01: N=3 for approve/rename/merge corrections to advance from `observed` to `candidate`.**
Rationale: statistical power (Fisher's exact), UX responsiveness, and precedent from recommendation systems. Three similar corrections provide enough signal to propose a rule without requiring tedious repetition.

**D-02: N=5 for split and mark-wrong corrections.**
Rationale: split and mark-wrong are high-risk (they restructure or reject groups). Require more evidence before proposing a rule. The cost of a false-positive split rule is high (Curator permanently separates related material).

**D-03: Learning weight multipliers: approve=1.0, rename=2.0, split=3.0, merge=3.0, mark-wrong=3.0.**
Rationale: explicit structural corrections (split, merge, mark-wrong) require active effort and encode stronger signal. Passive approvals may include groups the user didn't carefully examine.

**D-04: Undo within 60 seconds = discarded correction, pattern count not incremented.**
Rationale: 60-second undo is empirically a regret signal. Does not add enough evidence about genuine preference.

**D-05: Session position > 20 triggers learning weight decay (×0.5), > 40 triggers further decay (×0.25).**
Rationale: Danziger et al. (2011) decision fatigue finding. Later corrections in a long session are less reliable. They still route correctly for the current session but have reduced influence on rule formation.

**D-06: Rules are domain-scoped by default. Cross-domain generalization requires explicit user action.**
Rationale: University filing preferences do not generalize to Personal Projects. Domain scoping prevents unwanted cross-contamination.

**D-07: Confidence decay at 20% per 30-day inactive period for rules not triggered in 90+ days.**
Rationale: Personal workflows evolve. Rules from a previous semester should not automatically apply to the next. Decay forces periodic revalidation without being disruptive.

**D-08: Three automation levels (L1/L2/L3) mapped to Parasuraman-Sheridan 3/5/7. Each level requires explicit user opt-in per rule.**
Rationale: Automation should never escalate without user knowledge. The Parasuraman-Sheridan finding that users lose situational awareness above their optimal automation level is directly applicable.

**D-09: Bulk silent auto (L3) is capped at 5 files per automatic action. Any action affecting ≥ 10 files always shows a confirmation dialog.**
Rationale: Over-trust prevention. Large bulk actions without confirmation are the primary failure mode in automated file management tools.

**D-10: Random audit prompts for L3 rules: 1-in-20 automatic actions surface "Was this correct?" to user.**
Rationale: Keeps the user engaged, prevents drift, provides ongoing calibration data outside of formal review sessions.

**D-11: Annual trust reset — all L3 rules revert to L2 unless user re-confirms.**
Rationale: Prevents "forgotten automation." Users should actively choose to keep silent automation, not inherit it indefinitely.

**D-12: Peripheral files (cosine < 0.50 to group centroid) get learning confidence × 0.20 from an approve action.**
Rationale: Snorkel weak supervision principle. Approval is a group-level signal, not a file-level signal. Peripheral files require their own validation.

**D-13: Mondrian CP with minimum 20 calibration files per context; global CP fallback for smaller contexts.**
Rationale: From R9 findings. Mondrian CP is unreliable below 20 examples. Global CP provides coverage guarantee with lower specificity.

**D-14: Cannot-link constraints (from splits) down-weight inter-community edges by factor 0.1. Must-link constraints (from merges) add synthetic edge of weight 0.9.**
Rationale: Chowdhury & Tan (2024) graph-based constrained community detection. This preserves the graph structure while enforcing user constraints without making them hard and brittle.

**D-15: Label propagation (LabelSpreading, alpha=0.2) runs after each review session to propagate confirmed labels to unconfirmed neighbors.**
Rationale: Zhu & Ghahramani (2002). Uses graph structure to extend sparse confirmations to the surrounding neighborhood. Files with propagated probability < 0.60 are not auto-routed.

**D-16: SetFit fine-tuning triggers for contexts with ≥ 16 confirmed files AND ≥ 1 correction in past 30 days.**
Rationale: SetFit provides superior accuracy over centroid classifiers for contexts with active correction history, at acceptable training time on CPU (≈30 seconds).

**D-17: Rule specificity starts at the most specific level matching the triggering correction. Generalization to level N+1 requires N+2 additional confirming corrections.**
Rationale: RIPPER-style pruning (Cohen 1995). Start specific, generalize conservatively. Prevents the "always split any file with 'security' in the name" overgeneralization failure mode.

**D-18: Conflicting constraints (old must-link vs. new cannot-link for same pair) are surfaced to user for explicit resolution. Never resolved silently.**
Rationale: Silent conflict resolution would violate the "no unilateral action" principle. The user must know when their past and current preferences conflict.

**D-19: PU learning via `pulearn` library (ElkanotoPuClassifier) for per-context routing classifiers. Retrain when ≥ 5 new confirmed files in a context.**
Rationale: Elkan & Noto (2008). Approved files are positive, unreviewed files are unlabeled (not negative). PU learning correctly handles this distinction.

**D-20: Coverage discount of 20% for files that were auto-routed (never shown to user) before any explicit confirmation.**
Rationale: Bekker & Davis (2020). Auto-routed files are not a random sample from the true positive population — selection bias must be corrected.

---

## 12. Open Questions

**OQ-1: What is the right NER entity overlap threshold for "similar corrections"?**
Currently defined as "≥ 1 shared named entity." Is one shared entity enough? For "EPL401" and "EPL401" — yes, trivially. But what if one correction involves "Network Security" and another involves "EPL401 Lecture" — they share the "Security" concept semantically but not exactly? Should semantic entity similarity count, or only exact match?

**Possible approach:** Use entity embedding similarity (embed both entity strings, compute cosine). If > 0.80, treat as matching. This requires a semantic entity matching step at correction time.

**OQ-2: How to handle file renames by the user (not by Curator)?**
If the user renames a file from "lecture_notes.pdf" to "EPL401_Week3.pdf" directly in Finder, this is an implicit correction — the user is signaling that this file belongs to EPL401. Curator could detect this via FSEvents and use it as a low-weight learning signal. But: filename-based corrections are noisier than explicit group corrections. What weight should they receive?

**OQ-3: Should learn from inter-session patterns, not just intra-session patterns?**
Currently, pattern accumulation counts N similar corrections regardless of when they occurred. But three corrections made in the same session are less independent than three corrections made in three different sessions (each session may reflect a consistent mental model). Should inter-session corrections receive a bonus weight (because they represent independent evidence)?

**Possible approach:** Track session ID per correction. If N corrections come from ≥ 3 different sessions, advance to `candidate` at N=3. If all N corrections come from the same session, require N=5.

**OQ-4: Can Curator detect correction intent from the edit distance between pre-correction and post-correction states?**
A split that creates two groups of 25 files each is different from a split that moves 2 files out of a 50-file group. The latter is a small exception; the former is a fundamental boundary. Should the split pattern weight scale with the "evenness" of the resulting sub-groups (a 50/50 split is stronger evidence than a 48/2 split)?

**OQ-5: How to handle multi-user or shared filesystem scenarios?**
The current design assumes a single user. If two users share a Dropbox folder and both use Curator, their corrections may conflict. This is out of scope for v1 but worth noting for the research agenda.

**OQ-6: When should Curator proactively ask for clarification rather than waiting for corrections?**
The current model is reactive: Curator proposes, user corrects. A more active model would have Curator ask "I'm not sure whether this belongs in EPL401 or EPL312 — which do you prefer?" before making an error. Active learning (Settles, 2012) provides a framework for selecting which files to ask about (query by committee, uncertainty sampling). This could significantly reduce the number of corrections needed for good rule formation.

**OQ-7: Is there a principled way to detect the end of a "context lifetime"?**
When a course ends (EPL401 is finished), its context is stable — no new files should arrive, no new corrections will be made. Curator could detect this by monitoring the time since last file addition to a context and the time since last correction. After 90 days of no activity, a context could be "archived" — its rules still apply, but its centroid is frozen and its calibration set is not updated. This prevents past corrections from being overridden by stale re-clustering.

**OQ-8: What is the correct behavior when a rule conflicts with a Lock?**
A file can be locked ("never touch this"). A rule might try to route it. The lock must take absolute precedence — rules never override locks. This is straightforward in principle but must be explicitly enforced in the rule-matching logic: check lock status before applying any rule. This interaction is not yet specified in R1 (the lock is pre-filtering, before the state machine). Document this explicitly.

---

## References

| # | Citation | DOI / URL |
|---|---|---|
| 1 | Amershi et al. (2014). Power to the People: The Role of Humans in Interactive Machine Learning. *AI Magazine* 35(4). | https://doi.org/10.1609/aimag.v35i4.2513 |
| 2 | Fails & Olsen (2003). Interactive Machine Learning. *IUI 2003*. | https://doi.org/10.1145/604045.604056 |
| 3 | Dudley & Kristensson (2018). A Review of User Interface Design for Interactive Machine Learning. *ACM TIIS* 8(2). | https://doi.org/10.1145/3185517 |
| 4 | Mosqueira-Rey et al. (2023). Human-in-the-loop machine learning: A state of the art. *AI Review* 56. | https://doi.org/10.1007/s10462-022-10246-0 |
| 5 | Fürnkranz & Hüllermeier (Eds.) (2010). *Preference Learning*. Springer. | https://doi.org/10.1007/978-3-642-14125-6 |
| 6 | Joachims et al. (2005). Accurately Interpreting Clickthrough Data as Implicit Feedback. *SIGIR 2005*. | https://doi.org/10.1145/1076034.1076063 |
| 7 | Wagstaff et al. (2001). Constrained K-means Clustering with Background Knowledge. *ICML 2001*. | https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.13.8611 |
| 8 | Basu, Banerjee & Mooney (2002). Semi-supervised Clustering by Seeding. *ICML 2002*. | https://dl.acm.org/doi/10.5555/645531.655964 |
| 9 | Vu, Labroche & Boulicaut (2022). Learning Constraints for Constrained Clustering. *Machine Learning* 111. | https://doi.org/10.1007/s10994-022-06154-9 |
| 10 | Chowdhury & Tan (2024). Semi-Supervised Community Detection with User Constraints. *arXiv:2402.09171*. | https://arxiv.org/abs/2402.09171 |
| 11 | Elkan & Noto (2008). Learning Classifiers from Only Positive and Unlabeled Data. *KDD 2008*. | https://doi.org/10.1145/1401890.1401920 |
| 12 | Bekker & Davis (2020). Learning from Positive and Unlabeled Data: A Survey. *Machine Learning* 109. | https://doi.org/10.1007/s10994-020-05877-5 |
| 13 | Shalev-Shwartz (2012). Online Learning and Online Convex Optimization. *FTML* 4(2). | https://doi.org/10.1561/2200000018 |
| 14 | Nguyen et al. (2011). Adaptive Algorithms for Detecting Community Structure in Dynamic Social Networks. *INFOCOM 2011*. | https://doi.org/10.1109/INFCOM.2011.5934990 |
| 15 | Angelopoulos & Bates (2023). Conformal Prediction: A Gentle Introduction. *FTML* 16(4). | https://doi.org/10.1561/2200000101 |
| 16 | Zaffran et al. (2022). Adaptive Conformal Inference Under Distribution Shift. *NeurIPS 2022*. | https://doi.org/10.48550/arXiv.2202.13415 |
| 17 | Etchemendy & Burdet (2024). Conformal Prediction for Personalized Information Retrieval. *arXiv:2405.13268*. | https://arxiv.org/abs/2405.13268 |
| 18 | Cohen (1995). Fast Effective Rule Induction (RIPPER). *ICML 1995*. | https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.107.2612 |
| 19 | Parasuraman, Sheridan & Wickens (2000). A Model for Types and Levels of Human Interaction with Automation. *IEEE SMC-A* 30(3). | https://doi.org/10.1109/3468.844354 |
| 20 | Lee & See (2004). Trust in Automation: Designing for Appropriate Reliance. *Human Factors* 46(1). | https://doi.org/10.1518/hfes.46.1.50_30392 |
| 21 | Danziger, Levav & Avnaim-Pesso (2011). Extraneous Factors in Judicial Decisions. *PNAS* 108(17). | https://doi.org/10.1073/pnas.1018033108 |
| 22 | Ratner et al. (2017). Snorkel: Rapid Training Data Creation with Weak Supervision. *VLDB* 11(3). | https://doi.org/10.14778/3157794.3157797 |
| 23 | Zhu & Ghahramani (2002). Learning from Labeled and Unlabeled Data with Label Propagation. *CMU TR CMU-CALD-02-107*. | https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.14.3864 |
| 24 | Tunstall et al. (2022). Efficient Few-Shot Learning Without Prompts (SetFit). *arXiv:2209.11055*. | https://arxiv.org/abs/2209.11055 |
| 25 | Settles, B. (2012). *Active Learning*. Morgan & Claypool. | https://doi.org/10.2200/S00429ED1V01Y201207AIM018 |

---

*Research compiled 2026-06-08. Model: Claude Sonnet 4.6. All design decisions are specific to Curator's group-first filesystem organization pipeline.*
