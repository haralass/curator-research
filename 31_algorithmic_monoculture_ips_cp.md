# Algorithmic Monoculture in Conformal Prediction Feedback Loops and IPS Correction

**Date:** 2026-05-31  
**Context:** Curator file router — conformal prediction (CP) assigns each incoming file a prediction set of candidate folders; the user selects one, which becomes a positive training signal. Folders absent from the shown set receive no feedback ever, producing a feedback loop that collapses the folder utilisation distribution.

---

## 1. The Revealed Preference Problem: Formal Statement

Let F = {f₁, f₂, …, F_K} be the full folder vocabulary. At time t, CP produces a prediction set C_t ⊆ F such that |C_t| ≤ m for some maximum set size m ≪ K. The user observes only C_t and selects folder y_t ∈ C_t. We record (x_t, y_t) as a labelled training example.

Define the **policy propensity** of folder f at time t:

```
π_t(f | x_t)  =  P(f ∈ C_t | x_t)
```

Under the naive update rule, the empirical positive-feedback rate for folder f after T rounds is:

```
r_T(f) = (1/T) Σ_t  1[y_t = f]
```

For any folder f that is never included in any prediction set (π_t(f | x_t) ≈ 0 for all t), r_T(f) = 0 identically, even if f would be the ground-truth label for many files x. This is a **truncated feedback distribution**: labels are only observed conditional on the policy having exposed them. Naively treating (x_t, y_t) as i.i.d. supervised data yields **systematically biased nonconformity scores** for under-represented folders, which in turn suppresses those folders from future prediction sets — completing the feedback loop.

The problem is formally equivalent to the **choice-based sampling** problem in econometrics (McFadden, 1978): the researcher observes choices only from a restricted opportunity set, and must correct for the fact that the sampling probability of each alternative is policy-determined, not random.

---

## 2. Revealed Preference and Consideration Sets in Economics

### 2.1 McFadden (1978): Sampling Correction for Multinomial Logit

McFadden's correction addresses the case where a researcher cannot enumerate the full choice set C (here, all K folders) and instead works with a sampled subset D ⊆ C of size d. The multinomial logit (MNL) log-likelihood on the full set is:

```
ℓ(β) = Σ_i  [ V(y_i, x_i; β) − log Σ_{f ∈ C} exp V(f, x_i; β) ]
```

which is computationally intractable for large K. With sampled sets D_i drawn with known inclusion probability q(f | x_i, D_i), the corrected log-likelihood is:

```
ℓ_corr(β) = Σ_i  [ V(y_i, x_i; β) − log(q(y_i | x_i, D_i)) 
                   − log Σ_{f ∈ D_i} exp( V(f, x_i; β) − log q(f | x_i, D_i) ) ]
```

Under uniform conditioning (equal sampling probability per alternative), the log-correction cancels within the softmax and consistent estimation is restored. The key insight for our problem: **consistency requires knowing the sampling probability of every alternative in the shown set, including the chosen one.** In the Curator setting, this probability is exactly π_t(f | x_t), the CP set-inclusion probability.

### 2.2 Barseghyan et al. (AER 2021): Heterogeneous Choice Sets

Barseghyan, Molinari, and co-authors developed semi-nonparametric identification of discrete choice models where consideration sets are **endogenous** — they depend on both observed features and unobserved preference heterogeneity. The AER 2021 paper "Discrete Choice under Risk with Limited Consideration" (Vol. 111, No. 6, pp. 1972–2006) proves point identification under mild conditions without assuming that the researcher observes which consideration set was presented.

Their framework models the consideration set formation as a latent random set Γ(x) ⊆ F, and the choice as:

```
y = argmax_{f ∈ Γ(x)}  U(f, x, ε)
```

where ε is unobserved heterogeneity. The identification condition requires variation in x that induces variation in Γ(x) independently of ε. This is an exact analogue of our problem: in Curator, x is the file's feature vector, Γ(x) = C_t is the CP prediction set, and the user's selection is y. The difference is that in our setting, unlike in the economic application, we **know** Γ(x) — the prediction set is fully observed — which should in principle make the correction easier, but the feedback loop creates temporal dependence that the static identification framework does not address.

---

## 3. Inverse Propensity Scoring (IPS): The Causal Inference Toolkit

### 3.1 The Standard IPS Estimator

In counterfactual / off-policy learning, we observe a dataset D = {(x_t, a_t, r_t, π_0(a_t | x_t))}_{t=1}^T, where a_t is the action taken by the logging policy π_0, r_t is the observed reward, and π_0(a_t | x_t) is the logged propensity. The IPS estimator of the value of a target policy π is:

```
V̂_IPS(π) = (1/T) Σ_t  r_t · [π(a_t | x_t) / π_0(a_t | x_t)]
```

This is an unbiased estimator of V(π) = E_{x ~ ρ, a ~ π(·|x)}[r(x, a)] whenever support(π) ⊆ support(π_0) — i.e., whenever the logging policy assigns non-zero probability to every action the target policy might select.

**Unbiasedness proof sketch:** 

```
E[V̂_IPS(π)] = E_{x,a~π_0} [ r(x,a) · π(a|x) / π_0(a|x) ]
             = E_x Σ_a π_0(a|x) · r(x,a) · π(a|x) / π_0(a|x)
             = E_x Σ_a π(a|x) · r(x,a)
             = V(π)
```

The support condition is precisely where CP monoculture fails: if folder f has π_0(f | x_t) = 0 for all t (never shown), no IPS correction can recover its counterfactual value.

### 3.2 Swaminathan and Joachims (ICML 2015): Counterfactual Risk Minimization

Swaminathan and Joachims (arXiv:1502.02362, ICML 2015) formalize **Counterfactual Risk Minimization (CRM)**: learning a new policy π_θ from logged bandit feedback. They minimize:

```
R̂_IPS(π_θ) = − (1/T) Σ_t  r_t · [π_θ(a_t | x_t) / π_0(a_t | x_t)]
```

subject to a variance penalty. They prove a generalization error bound of the form:

```
R(π_θ) ≤ R̂_IPS(π_θ) + λ · √( Var[ŵ · r] / T )
```

where ŵ_t = π_θ(a_t|x_t) / π_0(a_t|x_t) are the importance weights. The variance penalty is critical: when π_0 assigns very small probabilities to some actions (as happens early in a CP feedback loop before those folders are shown), the weights explode, causing high-variance gradients. Their POEM algorithm (Policy Optimizer for Exponential Models) addresses this by clipping and normalising weights.

The **self-normalised IPS (SNIPS)** estimator from their companion NIPS 2015 paper is:

```
V̂_SNIPS(π) = [Σ_t r_t · π(a_t|x_t)/π_0(a_t|x_t)] / [Σ_t π(a_t|x_t)/π_0(a_t|x_t)]
```

SNIPS is biased but typically has 30–40% lower variance than vanilla IPS, which matters greatly when the denominator propensities are near zero — exactly the setting of under-explored folders in CP.

### 3.3 Joachims et al. (WSDM 2017): Unbiased Learning-to-Rank

Joachims et al. (2017) apply IPS to position bias in information retrieval. A user clicking on a result at rank k is more likely to have examined that rank than rank k+10. The propensity is the examination probability e_k (probability the user's eye reaches rank k). The IPS-corrected relevance estimate for document d shown at rank k with click c_{dk} ∈ {0,1} is:

```
r̂(d) = c_{dk} / e_k
```

This directly parallels the CP problem: position in the prediction set corresponds to rank, the CP set-inclusion probability corresponds to the examination probability, and the user's folder selection corresponds to the click. The structural analogy is exact for ordered prediction sets; for unordered sets it simplifies further since all shown folders have equal (unordered) salience.

---

## 4. The Heckman Selection Model

Heckman (1979) introduced a two-stage econometric correction for sample selection bias. The setup: the researcher observes an outcome y only when a selection variable s = 1. If (y, s) are jointly determined by shared latent factors, OLS on the selected subsample is biased.

**Stage 1:** Model the selection probability as a probit:
```
P(s = 1 | x, z) = Φ(z'γ)
```
where z contains instruments (variables affecting selection but not the outcome directly).

**Stage 2:** Add the inverse Mills ratio λ(z'γ̂) = φ(z'γ̂)/Φ(z'γ̂) to the outcome regression as a control variable:
```
E[y | x, s=1] = x'β + σ_{u1} · λ(z'γ)
```

Correcting for selection recovers consistent estimates of β. In Curator's case, the selection mechanism is the CP policy itself — whether a folder is included in the prediction set. The instrument would be features that affect CP inclusion but are independent of the folder's true relevance (e.g., a folder's historical frequency in past prediction sets, independent of the current file's semantic content). If such instruments exist, a Heckman-style correction is applicable. However, a key limitation is that the Heckman model assumes a parametric (Gaussian/probit) selection process, whereas CP set construction is nonparametric and set-valued. The instrument validity assumption is also hard to satisfy in a closed-loop system where everything is correlated.

A more practically relevant result is from Ovaisi et al. (SIGIR 2021), "Correcting for Selection Bias in Learning-to-Rank Systems" (arXiv:2001.11358), which adapts the Heckman two-stage approach to ranking, showing it is more robust to noise than pure IPS approaches when there is moderate to no position bias — a potentially useful hybrid for early-training CP where propensity estimates are unreliable.

---

## 5. Partial Feedback in Online Learning

### 5.1 Shao et al. (arXiv:2601.21462, 2026): Partial-Feedback Online Learning

This paper formalises a protocol where each instance x admits a **set of acceptable labels** Y(x) ⊆ Y, but the learner observes only one acceptable label per round (drawn from Y(x), not necessarily the one it predicted). Two key dimensions are introduced:

- **PFLdim** (Partial-Feedback Littlestone dimension): characterises minimax regret for deterministic learners.
- **PMSdim** (Partial-Feedback Measure Shattering dimension): characterises minimax regret for randomised learners.

The paper proves a barrier result: beyond set-realizability, even with just two hypotheses, regret can be linear in T. This is structurally relevant to CP monoculture: the "partial feedback" the CP system receives (a user pick from the shown set) is not the full label Y(x) (the true ideal folder). The learner never observes which unshown folders would have been acceptable, making the regret structure in the barrier regime a plausible characterisation of the monoculture failure mode.

**Key gap relative to Curator's problem:** In Shao et al., the set Y(x) of acceptable labels is exogenous — it is determined by the problem, not by the learner's own past policy. In Curator, C_t (the prediction set) is endogenous: it is produced by the current CP model trained on past biased feedback. This endogeneity is the core structural difference that makes the Curator problem harder.

### 5.2 Pour, Mansouri, and Ben-David (arXiv:2602.09402, 2026): Trichotomy of Regret Bounds

This paper studies online learning where each instance has multiple correct labels and establishes a trichotomy under three feedback models:

1. **Mistake-unknown feedback** (learner sees one correct label, no indicator of its own correctness): linear regret Ω(T) is unavoidable even for simple hypothesis classes.
2. **Mistake-known feedback** (learner sees one correct label plus a binary correct/incorrect signal): sublinear regret O(√(|Y| · T · LD_known · log T)).
3. **Set-valued feedback** (learner sees all correct labels): constant or logarithmic regret.

The CP feedback loop places Curator in regime 1 — closest to "mistake-unknown" — because:
- The user selects from C_t but does not signal whether other folders in C_t were also acceptable.
- Folders outside C_t are unobserved entirely.
- There is no external oracle providing the correct label.

This trichotomy implies that, without propensity correction, learning in Curator's exact feedback regime is subject to the worst-case linear regret result. Achieving sublinear regret requires either moving to mistake-known feedback (eliciting "was this the best folder?" from the user) or set-valued feedback (showing all acceptable folders) — both of which are impractical UI choices. IPS correction is the third option: it does not change the feedback model but re-weights the observed signals to obtain an unbiased gradient, effectively simulating access to counterfactual outcomes.

**Crucially, neither paper addresses endogenous selection**: the case where the set of observed labels depends on the learner's own current policy. This is the open problem specific to our setting.

---

## 6. Conformal Prediction and Selection Bias: The Open Problem

### 6.1 Standard CP Calibration

In standard split conformal prediction, we hold out a calibration set D_cal = {(x_i, y_i)}_{i=1}^n where (x_i, y_i) are i.i.d. from P_{XY}. The nonconformity score α_i = s(x_i, y_i) is computed for each calibration point (e.g., one minus the softmax probability of y_i). The prediction set at level 1 − δ is:

```
C(x) = { y : s(x, y) ≤ Q_{1-δ}(α_1, …, α_n, +∞) }
```

where Q_{1-δ} is the (1-δ)-quantile. This guarantees P(Y_{n+1} ∈ C(X_{n+1})) ≥ 1 − δ under exchangeability of the calibration and test points.

### 6.2 Feedback Covariate Shift

Fannjiang et al. (PNAS 2022, arXiv:2202.03613) study "Conformal Prediction Under Feedback Covariate Shift": the setting where the training and test data are statistically dependent because the learned model chooses the test-time input distribution. They construct confidence sets with finite-sample guarantees that hold for any prediction algorithm, even when a trained model determines the test distribution. This is structurally related to our problem — the CP model determines which files are routed where, which affects which files are labelled, which shapes the next calibration set.

However, Fannjiang et al. address covariate shift in X (the input distribution shifts because the model selects which x to query), not **label shift in Y conditioned on X** (the feedback distribution over y is truncated to C_t ⊆ F). Our problem is therefore a **label-censoring** variant of their setting, not a covariate shift variant.

### 6.3 Weighted Conformal Prediction

Tibshirani, Barber, Candès, and Ramdas (NeurIPS 2019, arXiv:1904.06019) extend CP to covariate shift by introducing likelihood-ratio weights. When the test covariate distribution Q differs from the training distribution P by a known density ratio w(x) = dQ/dP(x), the weighted prediction set uses a weighted empirical quantile:

```
C_w(x) = { y : s(x, y) ≤ Q_{1-δ}^w(α_1, …, α_n) }
```

where Q_{1-δ}^w is the (1-δ)-quantile of the weighted empirical distribution placing weight w(x_i)/Σ_j w(x_j) on each calibration score α_i. Marginal coverage P(Y ∈ C_w(X)) ≥ 1 − δ is maintained under this weighting.

This is the closest existing CP result to our setting, but it handles X-space shift only. The missing piece is: **what is the analogue when the calibration labels y_i are themselves selected by the policy (i.e., y_i ∈ C_{t_i} for some past prediction set C_{t_i}), making the calibration set endogenously censored in label space?**

---

## 7. Algorithmic Monoculture: Formal Results

### 7.1 Kleinberg and Raghavan (PNAS 2021)

Kleinberg and Raghavan model a one-sided matching market where k firms each rank n applicants using either (a) an algorithm with accuracy p (probability of correctly identifying the best applicant) or (b) an independent noisy human evaluator with accuracy q < p. When all firms use the same algorithm (monoculture), the top-ranked applicants are the same for every firm; since only one firm can hire each applicant, firms end up hiring sequentially down a shared ranking. Their main result is a **Braess's paradox for algorithmic decision-making**:

> *There exist accuracy parameters (p, q) such that: (1) each individual firm strictly prefers the algorithm over an independent evaluator, yet (2) social welfare — the expected quality of hires summed across all firms — is strictly lower under monoculture than under independent evaluation.*

The formal condition involves the correlation structure of the algorithm's errors across firms. When all firms use the same algorithm, errors are perfectly correlated: if the algorithm ranks applicant a above a better applicant b, every firm makes the same mistake, and the globally better applicant b is never hired. With independent evaluators (errors uncorrelated), at least some firms correctly rank b above a, and b is eventually hired.

The **Price of Monoculture** (formalised in subsequent work, arXiv:2604.00444, 2025) quantifies this as the ratio of social welfare under diversity to monoculture. For the hiring model, the price can be Θ(k) in the worst case — monoculture's social welfare is a factor of k worse than independent evaluation, where k is the number of firms (folders in our analogy).

### 7.2 The CP-Specific Monoculture Mechanism

In the Curator setting, the monoculture mechanism differs from Kleinberg-Raghavan in a crucial way: the feedback loop is **temporal** and **self-referential**. Folders that were popular at time t₀ appear in prediction sets, receive positive feedback, have their nonconformity scores improved, appear in even more prediction sets — a reinforcement dynamic. Folders that were absent at t₀ remain absent. This is closer to a **rich-get-richer** (preferential attachment) dynamic than to the simultaneous-firm monoculture of Kleinberg-Raghavan.

Formally, let μ_t(f) = P(f ∈ C_t) be the marginal inclusion probability of folder f at time t. Under naive (uncorrected) updates:

```
μ_{t+1}(f) ≈ μ_t(f) + η · [1[f ∈ C_t] · 1[y_t = f] − μ_t(f)]
```

For folders with μ_t(f) ≈ 0, the update signal is zero almost surely. The fixed point μ_∞(f) = 0 is absorbing for any f not in the initial prediction sets. This is not a convergence to a good equilibrium but to a degenerate one: the CP model's knowledge of many folders is frozen at initialisation.

---

## 8. The Proposed Correction: IPS-Adjusted CP

### 8.1 IPS-Corrected Nonconformity Scores

The core proposal is to reweight the contribution of each observation (x_t, y_t) to the calibration set by the inverse of the propensity that y_t was shown and selected:

```
w_t  =  π_ref(y_t | x_t) / π_t(y_t | x_t)
```

where π_t(y_t | x_t) = P(y_t ∈ C_t | x_t) · P(user selects y_t | C_t, x_t) is the joint probability of showing and the user selecting y_t at time t, and π_ref is a target reference distribution (e.g., uniform over F, or the prior folder frequency distribution).

The IPS-corrected empirical nonconformity score distribution for folder f is:

```
F̂_f^IPS(τ) = [Σ_{t: y_t = f} w_t · 1[s(x_t, f) ≤ τ]] / [Σ_{t: y_t = f} w_t]
```

This is a SNIPS estimator of the true nonconformity distribution for folder f under the reference distribution π_ref. If π_ref is uniform, it estimates what the nonconformity score distribution would look like if folders had been shown uniformly — correcting for the fact that some folders were shown more frequently than others.

**Coverage guarantee (informal):** If propensities π_t(·|·) are estimated consistently, then the IPS-corrected quantile threshold Q̂_f^IPS used in set construction satisfies:

```
P_{(X,Y) ~ P_ref}(Y ∈ C^IPS(X)) → 1 − δ
```

under standard IPS consistency conditions. The critical requirement is that π_t(f | x_t) > 0 for all f and t — the **positivity (overlap) condition**. Without forced exploration (ε-greedy or similar), this is violated for never-shown folders.

### 8.2 Modified Calibration Set Construction

The practical procedure has three components:

**1. Forced exploration:** At every round t, with probability ε_t (decaying exploration rate, e.g., ε_t = 1/√t), sample the prediction set C_t from a randomized policy rather than the pure CP model. This guarantees positivity: every folder f has π_t(f | x_t) ≥ ε_t · π_explore(f | x_t) > 0 for a well-designed exploration distribution π_explore.

**2. Propensity logging:** Record the propensity π_t(f | x_t) for the selected folder f = y_t alongside each labelled example. This requires logging the CP model's output probabilities at prediction time, not just the set membership.

**3. IPS-weighted calibration:** When recalibrating the CP model (periodically or online), use the IPS-weighted empirical distribution of nonconformity scores rather than the raw empirical distribution. The weighted quantile threshold for folder f is:

```
Q̂_f^IPS = inf{ τ : F̂_f^IPS(τ) ≥ 1 − δ }
```

This threshold is then used to define the folder-conditional prediction set:

```
C^IPS(x) = { f ∈ F : s(x, f) ≤ Q̂_f^IPS }
```

### 8.3 Variance Control and Doubly Robust Extension

The main practical risk of IPS is high variance when some propensities are near zero. Two mitigations:

**Propensity clipping:** Replace π_t(f|x) with max(π_t(f|x), τ_clip) for a small clipping threshold τ_clip. This introduces a small bias but substantially reduces variance. The clipped estimator is still consistent as τ_clip → 0.

**Doubly robust (DR) correction:** Combine IPS with a model-based imputation. Let r̂(x, f) be a predicted "relevance score" for folder f given file x (e.g., from the CP model's softmax output). The DR estimator is:

```
V̂_DR(f) = r̂(x, f) + [1[y=f] − r̂(x, f)] · π_ref(f|x) / π_t(f|x)
```

This is consistent if either the propensity model or the outcome model is correctly specified (the "doubly robust" property). In early training when propensities are unreliable, the outcome model term dominates; as propensities stabilise, the IPS correction term dominates. AIPW (Augmented IPW) achieves the semiparametric efficiency bound and is the natural target for this correction in our setting.

---

## 9. Theoretical Positioning: NeurIPS / ICML Contribution

### 9.1 What Is New

The combination of the following three ingredients appears to be novel in the literature:

1. **Conformal prediction as the policy** — not a classifier or ranker, but a set-valued prediction mechanism with coverage guarantees.
2. **Endogenous label censoring** — the calibration set labels are themselves censored by the CP policy's prediction sets, creating a circular dependency not present in covariate-shift CP (Tibshirani et al. 2019, Fannjiang et al. 2022) or partial feedback online learning (Shao et al. 2026, Pour et al. 2026).
3. **IPS-corrected nonconformity score quantiles** — using logged propensities to debias the calibration-set quantile estimation, preserving marginal coverage guarantees under the reference distribution.

A search across arXiv for combinations of {conformal prediction, propensity score, selection bias, feedback loop} does not surface any paper that addresses this triad. The closest are:
- Tibshirani et al. (2019): CP + covariate shift in X (not label censoring in Y).
- Fannjiang et al. (2022): CP + feedback covariate shift in X (active learning setting).
- Joachims et al. (2017): IPS + learning-to-rank (no CP, no coverage guarantees).
- Swaminathan & Joachims (2015): CRM + bandit feedback (no CP).
- Shao et al. (2026): partial feedback online learning (no CP, no propensity correction).

### 9.2 Theoretical Contribution Sketch

**Theorem (Informal):** Let {C_t}_{t=1}^T be prediction sets generated by a CP model with logged propensities π_t(·|·) satisfying π_t(f|x) ≥ π_min > 0 for all f, x, t (enforced via exploration). Let Q̂_f^IPS(δ) be the IPS-corrected (1−δ)-quantile of nonconformity scores for folder f, computed from n_f IPS-weighted calibration points for folder f. Then for any test point (X, Y) drawn from the reference distribution P_ref:

```
P( Y ∈ C^IPS(X) ) ≥ 1 − δ − O( √(log(K/δ) / (π_min² · n_min)) )
```

where n_min = min_f n_f is the minimum (IPS-effective) sample size across folders, and K = |F| is the folder count. The correction vanishes as n_min → ∞, recovering exact (1−δ) coverage.

**Corollary:** Without IPS correction (π_min = 0 for some f), the coverage guarantee for under-explored folders degenerates: folders f with π_t(f|x) = 0 for all t have Q̂_f^IPS = +∞ (never calibrated), resulting in either never being included in prediction sets (if a finite threshold is used) or always being included (trivial coverage, no informativeness).

### 9.3 Open Subproblems

Several clean theoretical questions remain open:

- **Adaptive exploration:** What is the optimal exploration schedule ε_t that minimises the regret from forced randomisation while achieving O(1/√T) IPS estimation error? This is a bandit-within-CP problem.
- **Coverage under distribution shift:** If the true label distribution P(Y | X) shifts over time (files from new domains), does the IPS-corrected CP maintain coverage, or does a doubly robust + conformal correction (Kuchibhotla et al. style) become necessary?
- **Set size / efficiency tradeoff:** IPS correction may increase prediction set sizes (more conservative thresholds for rare folders). Characterising the IPS-efficiency tradeoff — coverage guarantee vs. expected |C(x)| — is a clean minimax problem.
- **Propensity estimation from set membership:** In practice, π_t(f|x) must be estimated from the CP model's softmax scores. Does using estimated rather than true propensities affect coverage? A doubly robust CP bound that is valid under propensity misspecification is needed.

---

## 10. Connection to the Monoculture Literature: A Synthesis

The Kleinberg-Raghavan monoculture result is about **simultaneous agents** sharing a ranking and competing for scarce resources (candidates). Curator's monoculture is **sequential self-reference**: a single agent whose outputs become its own training signal. The two mechanisms differ but converge on the same pathology — diversity collapse.

A unifying perspective: both are instances of **correlated failure under positive feedback**. In Kleinberg-Raghavan, correlation is across firms (same algorithm, same errors). In Curator, correlation is across time (same model, same biased training signal). The Price of Monoculture (Θ(k) in the K-R model) translates in our setting to a prediction-set efficiency loss: if m folders are perpetually excluded from consideration sets, their files are always routed to the wrong folder, contributing 0% correct routing for approximately m/K of future files — a loss proportional to the fraction of the vocabulary that is frozen out.

The IPS correction breaks the correlation by reweighting observations to simulate what would have been seen under a uniform or diversity-encouraging policy. It is, in a sense, the temporal analogue of K-R's recommendation to use **diverse (independent) evaluators** rather than a shared algorithm.

---

## References

- Barseghyan, L., Coughlin, M., Molinari, F., & Teitelbaum, J. C. (2021). Discrete choice under risk with limited consideration. *American Economic Review*, 111(6), 1972–2006.
- Barseghyan, L., & Molinari, F. (2023). Heterogeneous choice sets and preferences. *Econometrica*, 91(6).
- Fannjiang, C., Bates, S., Angelopoulos, A. N., Listgarten, J., & Jordan, M. I. (2022). Conformal prediction under feedback covariate shift for biomolecular design. *PNAS*, 119(43).
- Heckman, J. J. (1979). Sample selection bias as a specification error. *Econometrica*, 47(1), 153–161.
- Joachims, T., Swaminathan, A., & Schnabel, T. (2017). Unbiased learning-to-rank with biased feedback. *WSDM 2017*. [PDF](https://www.cs.cornell.edu/~tj/publications/joachims_etal_17a.pdf)
- Kleinberg, J., & Raghavan, M. (2021). Algorithmic monoculture and social welfare. *PNAS*, 118(22), e2018340118. [Link](https://www.pnas.org/doi/10.1073/pnas.2018340118)
- McFadden, D. (1978). Modelling the choice of residential location. *Transportation Research Record*, 673.
- Ovaisi, Z., Bhargava, R., Zhang, Y., Vasilescu, Y., & Zheleva, E. (2020). Correcting for selection bias in learning-to-rank systems. arXiv:2001.11358.
- Pour, A. F., Mansouri, F., & Ben-David, S. (2026). Learning with multiple correct answers — a trichotomy of regret bounds under different feedback models. arXiv:2602.09402.
- Shao, S., Fang, C., Lin, Z., & Tao, D. (2026). Partial feedback online learning. arXiv:2601.21462.
- Swaminathan, A., & Joachims, T. (2015). Counterfactual risk minimization: Learning from logged bandit feedback. *ICML 2015*. arXiv:1502.02362.
- Swaminathan, A., & Joachims, T. (2015). The self-normalized estimator for counterfactual learning. *NIPS 2015*.
- Tibshirani, R. J., Barber, R. F., Candès, E. J., & Ramdas, A. (2019). Conformal prediction under covariate shift. *NeurIPS 2019*. arXiv:1904.06019.
