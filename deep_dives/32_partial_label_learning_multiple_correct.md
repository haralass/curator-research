# Partial Label Learning and Learning with Multiple Correct Answers: Deep Read of arXiv:2602.09402 and Relevance to Curator's Conformal Prediction Feedback Problem

**Date:** 2026-05-31  
**Status:** Literature survey, gap analysis  
**Tags:** partial-label learning, conformal prediction, semi-bandit, endogenous feedback, online learning, Curator

---

## 1. The Problem This Research Is Trying to Solve

Curator routes files to folders using a conformal prediction (CP) set. At coverage level 1−α, CP outputs a prediction set C(x) ⊆ F guaranteed (in the calibration-set sense) to contain the true destination folder with probability at least 1−α. The user then picks a folder from C(x). The feedback Curator receives—the user's click—is therefore doubly constrained:

1. **Set-constrained**: the user can only confirm a folder that appears in C(x). If the user's genuine preferred folder is not in C(x), that preference is invisible to the system.
2. **Endogenous**: the prediction set C(x) itself was chosen by the model. Future calibration data is therefore a function of the model's own past decisions, not an i.i.d. draw from nature.

These two constraints together create what economists call a **selection bias from endogenous choice sets**. It is a strictly harder problem than standard conformal calibration, standard bandit learning, and standard partial label learning. Understanding exactly how hard it is—and what existing theory covers—requires surveying three adjacent literatures: learning with multiple correct answers (arXiv:2602.09402), partial label learning, and semi-bandit conformal prediction.

---

## 2. Deep Read: arXiv:2602.09402 — "Learning with Multiple Correct Answers: A Trichotomy of Regret Bounds under Different Feedback Models"

**Authors:** Alireza F. Pour, Farnam Mansouri, Shai Ben-David  
**Submitted:** February 2026  

### 2.1 The Setting

The paper formalizes a multi-class online learning scenario where each instance x has a *set* of valid labels Y(x) ⊆ Y (not just one). At each round t, the learner observes x_t, outputs a label ŷ_t, and then receives feedback whose structure depends on the feedback model. A mistake is made when ŷ_t ∉ Y(x_t).

This captures language generation (multiple acceptable phrasings), recommendation (multiple acceptable items), and file routing (multiple plausible folders). The standard Halving/Littlestone framework applies when |Y(x)| = 1; this paper lifts that restriction.

### 2.2 The Three Feedback Models

The trichotomy result turns on which feedback is available after each round:

| Feedback Model | What the learner observes |
|---|---|
| **Full information** | The entire correct set Y(x_t) is revealed |
| **Partial feedback** | A subset S ⊆ Y(x_t) is disclosed (possibly just whether ŷ_t was correct) |
| **Bandit** | Only the correctness of ŷ_t ∈ Y(x_t) is revealed; no label is named |

### 2.3 The Trichotomy of Regret Bounds

The paper's central theorem is that these three feedback models are not merely quantitatively different—they induce **qualitatively different** learnability regimes in the agnostic (non-realizable) setting:

- **Full information regime**: Optimal regret is characterized by a combinatorial dimension analogous to the Littlestone dimension extended to set-valued labelings. Logarithmic or polynomial regret is achievable.
- **Partial feedback regime**: A strictly weaker guarantee holds. The complexity measure is a new partial-feedback shattering dimension. In certain hypothesis classes, regret that was logarithmic under full information becomes polynomial here.
- **Bandit regime**: For some classes, regret becomes **exponential** and no efficient algorithm can avoid this blow-up. This establishes a genuine separation—the first paper to prove that the bandit version of multi-correct-answer learning is exponentially harder than full-information in the worst case.

The paper additionally proves tight mistake bounds in the realizable setting (when a perfect consistent hypothesis exists) and derives batch sample complexity bounds from the online analysis via standard online-to-batch conversion.

### 2.4 Does the Paper Handle Endogenous Label Sets?

This is the critical question for Curator. The answer is **no, not explicitly**. In arXiv:2602.09402:

- The label set Y(x_t) is **exogenous**. It is determined by the instance x_t (by nature or by a data-generating process), not by the learner's choice of prediction set.
- In the partial feedback model, the subset S disclosed to the learner is also determined externally (e.g., a supervisor who knows Y(x_t) but reveals only part of it). It is not the same thing as the learner's own prediction set constraining which labels can be confirmed.
- The bandit model is the closest analog: the learner picks a single label and learns only whether it was correct. But even here, the "arm" choice does not restrict which ground-truth labels are in principle observable; it only restricts what feedback arrives this round.

Curator's situation is structurally different: the prediction set C(x) the model outputs IS the constraint on what the user can confirm. A user who would have chosen folder F ∉ C(x) provides no signal. arXiv:2602.09402 does not model this. The correct framing is closer to **endogenous choice sets in revealed preference theory**—a different branch of the literature entirely.

---

## 3. Partial Label Learning: State of the Art

### 3.1 Cour, Sapp, Taskar (JMLR 2011) — The Foundation

Cour et al. introduced the formal framework for **partial label learning (PLL)**: each training example arrives with a candidate label set S_i ⊇ {y_i^*}, where y_i^* is the (unique, unknown) ground truth. The goal is to learn a classifier from such ambiguous supervision. Their main contribution was a convex relaxation of the disambiguation problem and convergence guarantees for average-based methods that treat all candidate labels uniformly. The key assumption: candidate label sets are generated independently of the learner's decisions. This exogeneity assumption runs throughout all classical PLL.

### 3.2 Feng, Lv, Han et al. (NeurIPS 2020) — Provably Consistent PLL

Feng et al. proposed the first generation model for candidate label sets, enabling provably consistent PLL estimators. They showed that without such a generative model, consistency is unidentifiable: any candidate set could be consistent with many different ground-truth distributions. With the generation model (which specifies how non-ground-truth labels get included in S_i), they derived:
- A **risk-consistent** estimator: the empirical risk converges to the population risk under the true label.
- A **classifier-consistent** estimator: the argmax recovers the Bayes-optimal classifier.

Both estimators are agnostic to which element of S_i is the true label, yet converge to the right answer because the generation model identifies the structure. The deep-learning version of this work catalyzed an explosion of PLL research using neural networks.

### 3.3 Xu, Qiao, Geng, Zhang (NeurIPS 2021) — Instance-Dependent PLL

Earlier PLL work assumed candidate labels were added uniformly at random. Xu et al. broke this assumption: in realistic annotation settings, confusable or visually similar labels are more likely to appear together. They introduced a **latent label distribution** ρ(x) over Y for each instance x, where the probability of label c ∈ S_i scales with ρ_c(x) (how well c describes x). This makes disambiguation harder because incorrect labels are systematically informative—they cluster around the true label in feature space. Their disambiguation algorithm uses self-training with a confidence-weighted loss that gradually up-weights the most likely true label. This is the dominant paradigm for deep PLL as of 2025.

### 3.4 What PLL Misses for Curator

PLL literature uniformly assumes:
1. The candidate set S_i is provided by the annotation process, **not chosen by the learning algorithm itself**.
2. The true label is guaranteed to be in S_i (the "superset assumption").
3. The learning task is offline (batch) or at worst online with fixed generating process.

None of these hold for Curator: C(x) is chosen by the algorithm, the true folder may not be in C(x) (CP provides probabilistic not certain coverage), and the generating process shifts as user behavior changes. PLL methods applied naively to Curator's feedback would be misspecified.

---

## 4. Semi-Bandit Feedback and Conformal Prediction

### 4.1 Zimmert, Seldin (ICML 2019 / JMLR 2021) — The Tsallis-INF Algorithm

Zimmert and Seldin's Tsallis-INF algorithm is the baseline for understanding semi-bandit regret. In the **combinatorial semi-bandit** setting, the learner picks a subset A_t ⊆ [K] of arms, and receives reward r_{i,t} for each i ∈ A_t (partial feedback on the chosen subset, full ignorance on unchosen arms). Tsallis-INF achieves simultaneously:
- O(log T) regret in stochastic environments
- O(√(KT)) regret in adversarial environments

without knowing which regime applies. This "best of both worlds" guarantee is the gold standard for semi-bandit algorithms.

For Curator, the analogy would be: the prediction set C(x) plays the role of the chosen arm subset, and the user's click is partial reward feedback. But the analogy is imperfect because (a) the prediction set is not chosen from a fixed combinatorial structure of K arms—it is the output of a conformal quantile mechanism—and (b) the user's behavior is not a stochastic reward but a choice from a constrained menu, which has a different statistical structure.

### 4.2 Lattimore and Szepesvári — Bandit Algorithms (2020, Cambridge)

The definitive reference for bandit theory. Key results relevant here:
- In the **linear bandit** setting with action set A_t ⊆ R^d, regret bounds scale with the size of the action set and the ambient dimension, not just K. This matters for Curator because C(x) varies in size across rounds.
- The **eluder dimension** governs when function approximation in bandits is tractable. For conformal predictors parameterized by a scalar quantile threshold, the eluder dimension is 1, which is favorable—but the endogeneity of the feedback distribution makes this analysis inapplicable directly.

### 4.3 Stochastic Online Conformal Prediction with Semi-Bandit Feedback (arXiv:2405.13268, ICML 2025)

This paper is the most directly relevant prior work to Curator's problem. It addresses online CP where the true label y_t is revealed **only if** y_t ∈ C_t(x_t) (the prediction set output by the model). Key results:

- They show that the naive approach of updating the conformal quantile only when feedback arrives leads to systematic **under-coverage**: if the model is conservative, it sees few positives, updates rarely, and becomes even more conservative.
- Their algorithm achieves **sublinear regret** relative to an oracle that always observes y_t, by using importance weighting to correct for the selection bias in feedback.
- Empirical validation on retrieval, image classification, and auction pricing.

**What it does and does not address for Curator:**  
It correctly identifies and corrects for the semi-bandit selection bias in coverage. However, it does not address the question of **which element of C_t(x_t) the user picks**. The paper's implicit assumption is that the user's feedback is binary (was the target in the set or not?). In Curator's setting, the user actively selects one folder from C(x), and that selected folder becomes the "revealed preference" training signal. The model needs to update on which label the user chose—not just whether coverage occurred.

### 4.4 Adversarial Semi-Bandit Conformal Prediction (arXiv:2604.17984)

This April 2026 paper extends to an adversarial data sequence. The learner must maintain coverage guarantees even when an adaptive adversary chooses the data. The main technique is to treat each candidate conformal threshold (discretized) as a bandit arm, and to achieve coverage via sublinear regret relative to the best fixed threshold in hindsight.

Again, this handles the coverage maintenance problem but not the revealed preference problem: the adversary determines y_t externally, and the paper does not model users selecting labels from the prediction set. Coverage guarantee and label distribution learning are treated as separate, when for Curator they are coupled.

### 4.5 Efficient Online Set-valued Classification with Bandit Feedback (arXiv:2405.04393)

This paper (Bandit Class-specific Conformal Prediction, BCCP) addresses full bandit feedback: the learner picks a **single** label and only learns whether it was correct. This is strictly harder than Curator's setting (where the whole set is shown and the user selects from it). BCCP uses unbiased gradient estimation to train both the model and the conformal calibrator simultaneously, achieving class-conditional coverage guarantees. Its techniques for unbiased estimation under sparse feedback are potentially useful building blocks, but the single-arm bandit structure doesn't match Curator's menu selection structure.

---

## 5. Partial Feedback Online Learning (arXiv:2601.21462)

This January 2026 paper addresses a distinct but related setting: each instance has multiple acceptable labels, but the learner observes only one acceptable label per round (not necessarily the one it predicted). They introduce:

- **Partial-Feedback Littlestone dimension (PFLdim)**: governs deterministic learnability.
- **Partial-Feedback Measure Shattering dimension (PMSdim)**: governs randomized learnability.

A striking negative result: beyond set-realizability, regret can become linear even with only two hypotheses. This warns that partial feedback in multi-correct-answer settings is not merely a degraded version of full-information learning—it can be categorically harder.

For Curator: the setting is again exogenous. One acceptable label is revealed per round, but by whom and constrained how is left unspecified. The endogeneity of which acceptable label gets revealed (because it is the user's choice from a constrained menu) is not modeled.

---

## 6. Conformal Prediction and Human Decision Making (arXiv:2503.11709)

A 2025 paper that asks whether CP prediction sets actually help human decision-making. Key finding: when a decision-maker has **private information** (knowledge not in the model's features), even perfectly calibrated probabilities can mislead, and the prediction set degrades because its coverage guarantee is marginal (averaged over instances), not conditional on the user's private signal.

For Curator: this formalizes why treating user folder choices as i.i.d. feedback is wrong. The user's selection from C(x) is conditioned on private signals (file context, memory, project structure) that Curator's model does not observe. The user's choice from C(x) is therefore **not exchangeable** with any hypothetical choice from the full folder set F. This is exactly the endogeneity problem: P(user chooses F | F ∈ C(x)) ≠ P(user chooses F | F ∈ F).

---

## 7. Revealed Preference, Selection Bias, and the Endogeneity Gap

The economics and econometrics literature on **revealed preference** is illuminating here. When an agent chooses from a constrained menu, their choice reveals only that the chosen option is preferred among the *available* alternatives—not that it is globally preferred. This is the standard critique of discrete choice models with endogenous choice sets.

In machine learning, the analogous problem appears in:
- **Slate recommendation**: a ranker shows k items; user clicks reveal preference over the slate, not the full catalog. The **position bias** and **slate propensity** literature addresses this, but assumes the slate is chosen by the algorithm on behalf of the user (not by a conformal mechanism with coverage guarantees).
- **Online learning with defective feedback**: Cesa-Bianchi and Lugosi's work on partial monitoring games covers the general information structure, but does not couple the feedback structure to a coverage constraint.
- **Inverse propensity scoring**: the standard tool for debiasing click feedback in information retrieval. IPS requires knowing P(item shown | query), which in CP is exactly the conformal coverage probability P(F ∈ C(x)). CP provides this! This creates an opening: the conformal coverage probability could serve as the propensity score for IPS correction. But this connection has not been formalized in the CP literature.

---

## 8. What Gap Remains: The Genuine Open Problem

After surveying all of this literature, the following gap is clear and well-defined:

**The problem:** An online learner produces a conformal prediction set C_t(x_t) ⊆ F. A user with private signals selects one label f_t ∈ C_t(x_t). The learner observes (x_t, C_t(x_t), f_t). It must simultaneously:
1. Maintain calibrated marginal coverage: P(y_t^* ∈ C_t(x_t)) ≥ 1 − α.
2. Learn the true distribution over labels: improve the ranking/scoring of labels in future rounds.

**Why existing work does not solve this:**

| Paper | Coverage maintained? | Label learning corrected for endogeneity? |
|---|---|---|
| arXiv:2405.13268 (stochastic semi-bandit CP) | Yes | No (binary feedback only) |
| arXiv:2604.17984 (adversarial semi-bandit CP) | Yes | No (adversarial y_t, no user choice model) |
| arXiv:2602.09402 (multiple correct answers) | Not applicable | No (exogenous label sets) |
| arXiv:2601.21462 (partial feedback online) | Not applicable | No (exogenous revealed label) |
| PLL literature (Feng 2020, Xu 2021, etc.) | Not applicable | No (batch, exogenous candidates) |
| BCCP arXiv:2405.04393 | Yes (class-conditional) | No (single-arm bandit, not menu selection) |

**The open problem, stated precisely:**

> Design an online conformal predictor that maintains marginal (or ideally conditional) coverage P(y_t^* ∈ C_t(x_t)) ≥ 1 − α under semi-bandit feedback (y_t^* observed only when y_t^* ∈ C_t(x_t)), while simultaneously providing consistent estimation of the label distribution P(Y | X) from user selections f_t ∈ C_t(x_t), where f_t is drawn from P(F | F ∈ C_t(x_t), private signal z_t) rather than from P(F | x_t) directly.

**Decomposed into sub-problems:**

1. **Propensity correction**: Use the known conformal inclusion probability p_t(f) = P(f ∈ C_t(x_t)) as an IPS weight to debias the label learning signal. Requires proving that the conformal quantile mechanism produces well-calibrated inclusion probabilities per label.

2. **Coverage-learning tradeoff**: A smaller C_t(x_t) reduces user effort and improves usability but increases the probability that the true label is excluded and the label learning signal is censored. Formalizing this tradeoff as a regret objective that couples both goals is open.

3. **Private signal contamination**: The user's private signal z_t (knowledge of the file's content, project context) means f_t is not a sample from P(Y | x_t) even after IPS correction. This is the hardest sub-problem and likely requires a structural assumption on z_t (e.g., that it is a sufficient statistic for the residual confusion among labels in C_t(x_t)).

4. **Adaptive adversary extension**: If the user's behavior evolves (changes in workflow, new projects), the feedback distribution itself shifts. Combining conformal coverage guarantees with adaptive feedback distributions is an open problem even without the endogeneity issue.

**Why this is NeurIPS-worthy:**

The coupling between a coverage-guaranteeing prediction set and a label-learning objective is structurally novel. It is not a special case of semi-bandit conformal prediction (which ignores which label the user chose). It is not a special case of PLL (which ignores the algorithmic origin of the candidate set). It is not a special case of revealed preference / IPS (which ignores the coverage constraint). A clean formalization with provable guarantees would be a genuine theoretical contribution, and the file routing application is a concrete, motivated instance.

---

## 9. Concrete Implications for Curator

**Short term — what can be done now:**

1. **IPS weighting with conformal propensities.** For each round where the user selects folder f_t ∈ C_t(x_t), weight the training signal by 1 / P_CP(f_t ∈ C_t(x_t)). The conformal inclusion probability P_CP(f ∈ C(x)) can be estimated from the calibration set as the fraction of calibration instances where f is included in the CP set for similar queries. This is an approximation but corrects the dominant selection bias.

2. **Track coverage empirically.** Log, for each round, whether the user accepted a folder from C(x) or dismissed/manually navigated. The fraction of accepted assignments estimates realized coverage. If this falls below 1−α, the quantile threshold is being updated incorrectly—switch to the importance-weighted update from arXiv:2405.13268.

3. **Pessimistic label learning.** When the user does not select from C(x) (manual navigation), treat this as censored feedback: all folders in C(x) are negative signals, the true folder is unknown. This is recoverable only if a future interaction reveals the destination.

**Medium term — research agenda:**

- Formalize the joint objective as a regret decomposition: regret on coverage + regret on label ranking.
- Prove that IPS-corrected online gradient descent on the scoring model is consistent under the conformal feedback mechanism.
- Study whether conditional coverage (P(y* ∈ C(x) | x) ≥ 1−α) is achievable under endogenous feedback, or whether only marginal coverage can be maintained.

---

## 10. References

- Pour, Mansouri, Ben-David. "Learning with Multiple Correct Answers — A Trichotomy of Regret Bounds under Different Feedback Models." arXiv:2602.09402, February 2026.
- Cour, Sapp, Taskar. "Learning from Partial Labels." JMLR 12, 2011.
- Feng, Lv, Han, Xu, Niu, Geng, Sugiyama. "Provably Consistent Partial-Label Learning." NeurIPS 2020. [PDF](https://proceedings.neurips.cc/paper/2020/file/7bd28f15a49d5e5848d6ec70e584e625-Paper.pdf)
- Xu, Qiao, Geng, Zhang. "Instance-Dependent Partial Label Learning." NeurIPS 2021. [OpenReview](https://openreview.net/pdf?id=PHcLZ8Yh6h4)
- Zimmert, Seldin. "Beating Stochastic and Adversarial Semi-bandits Optimally and Simultaneously." ICML 2019. [Paper](https://proceedings.mlr.press/v97/zimmert19a.html)
- Zimmert, Seldin. "Tsallis-INF: An Optimal Algorithm for Stochastic and Adversarial Bandits." JMLR 2021.
- Lattimore, Szepesvári. *Bandit Algorithms.* Cambridge University Press, 2020.
- Stochastic Online Conformal Prediction with Semi-Bandit Feedback. arXiv:2405.13268 (ICML 2025). [arXiv](https://arxiv.org/abs/2405.13268)
- Online Conformal Prediction with Adversarial Semi-Bandit Feedback via Regret Minimization. arXiv:2604.17984. [arXiv](https://arxiv.org/abs/2604.17984)
- Efficient Online Set-valued Classification with Bandit Feedback (BCCP). arXiv:2405.04393. [arXiv](https://arxiv.org/abs/2405.04393)
- Partial Feedback Online Learning. arXiv:2601.21462. [arXiv](https://arxiv.org/abs/2601.21462)
- Conformal Prediction and Human Decision Making. arXiv:2503.11709. [arXiv](https://arxiv.org/html/2503.11709v2)
