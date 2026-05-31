# 56 — Calibration in Agentic Pipelines
_Research date: 2026-05-31_

---

## Why This Matters for Curator

Calibration research has, until recently, been almost exclusively a single-step problem: given a classifier's output confidence score, does that score accurately reflect empirical accuracy? Guo et al. (2017) made this tractable by introducing Expected Calibration Error (ECE) and showing that a single learned temperature parameter can restore calibration in over-confident deep networks. Conformal Prediction (Angelopoulos & Bates, 2021) went further by offering distribution-free coverage guarantees that hold without any assumptions about the model's internal probabilities. These are mature, well-understood tools.

What neither tradition addresses is what happens when you chain calibrated (or nearly calibrated) steps together. Curator is a four-step pipeline: file-type classification → intent inference → candidate folder selection → confirmation/action. Each step conditions on the output of the previous one. If Step 1 produces a slightly wrong categorization, Step 2 receives off-distribution input — and the calibration guarantee that Step 2 was trained to provide applies to in-distribution inputs. The calibration of Steps 3 and 4 degrades further. The pipeline's *joint* accuracy can be dramatically lower than any individual step's accuracy, yet no step's confidence score reports this degradation. The system can appear confidently calibrated at each node while being badly miscalibrated end-to-end.

This is not a solved problem. The literature on calibration error propagation in multi-step classification pipelines is sparse: NLP researchers documented it empirically in 2015, control-systems engineers quantify it analytically for physical simulations, and a handful of papers published in 2025–2026 have begun to address it for LLM agent trajectories and multi-stage NLP systems. For Curator, this gap represents both a research opportunity and a practical engineering obligation: without explicit pipeline-level calibration measurement, the system cannot know when to interrupt the user versus when to act autonomously.

---

## Calibration Fundamentals

**Guo et al., "On Calibration of Modern Neural Networks," ICML 2017 — arXiv:1706.04599**

The canonical reference. Guo, Pleiss, Sun, and Weinberger showed that modern deep networks are significantly more overconfident than networks from the early 2000s, attributing the degradation to depth, width, batch normalization, and the absence of weight decay. They introduced or popularized three key tools:

- **ECE (Expected Calibration Error):** Predictions are binned by confidence; within each bin, the gap between mean confidence and mean accuracy is measured; ECE is the weighted average of these gaps. A perfectly calibrated model has ECE = 0.
- **Reliability Diagrams:** A visual analog of ECE — a 45-degree diagonal means perfect calibration; bars above the diagonal indicate underconfidence, bars below indicate overconfidence.
- **Temperature Scaling:** A single scalar parameter T divides all logits before the softmax. T > 1 softens the distribution (reduces overconfidence). T is fit on a held-out validation set by minimizing negative log-likelihood. It does not change accuracy, only confidence scores.

**Key caveat:** Ovadia et al. (NeurIPS 2019, "Can You Trust Your Model's Uncertainty? Evaluating Predictive Uncertainty under Dataset Shift") showed that calibration trained in-distribution degrades substantially under covariate shift — exactly the problem Curator faces when encountering unusual file types or novel user workflows.

**LLM-specific calibration:** Kadavath et al. (2022, arXiv:2207.05221, "Language Models (Mostly) Know What They Know") showed that large language models can be surprisingly well self-calibrated when asked to produce P(True) estimates for proposed answers. Calibration improves with model scale, but degrades on open-ended generation tasks — relevant because Curator's intent-inference step is closer to generation than multiple-choice.

---

## Conformal Prediction as a Calibration Guarantee

**Angelopoulos & Bates, "A Gentle Introduction to Conformal Prediction and Distribution-Free Uncertainty Quantification," arXiv:2107.07511**

Split Conformal Prediction (SCP) protocol:

1. Train the model on a training set.
2. Compute nonconformity scores (e.g., 1 − softmax probability for the true class) on a held-out calibration set of size n.
3. Set the threshold q̂ at the ⌈(n+1)(1−α)⌉/n quantile of calibration scores.
4. At test time, output the prediction set: all classes whose nonconformity score is below q̂.

The coverage guarantee is marginal and finite-sample: P(Y ∈ Ĉ(X)) ≥ 1 − α, with the only assumption being exchangeability of calibration and test points. This is a **coverage guarantee, not a point-calibration guarantee** — it says the set will contain the true answer with probability 1 − α, but does not say anything about confidence score accuracy.

**The crucial distinction:** ECE-style calibration asks "is the confidence score numerically accurate?" SCP asks "does the prediction set contain the true label with at least 1 − α probability?" SCP is strictly weaker (it does not require accurate confidence scores), but also strictly more robust (it holds in finite samples under no model assumptions). For Curator's folder selection step, SCP is directly applicable: the correct destination folder will be in the prediction set at least 1 − α fraction of the time.

**Sequential dependency problem for SCP:** When CP is applied at multiple stages of a pipeline, sequential dependency chains across calibration scores violate the independence/exchangeability premise. Stage k's nonconformity scores are not exchangeable with calibration scores if the input at stage k was filtered or transformed by stage k-1. Standard SCP guarantees do not compose across pipeline stages.

---

## Calibration Error Propagation in Sequential Systems

The section with the most significant gap in the literature — and the most important for Curator.

**Classical NLP pipeline research:**

Fonseca & Dang-Nguyen, "When It's All Piling Up: Investigating Error Propagation in an NLP Pipeline" (CEUR-WS Vol-1386, 2015) quantified how errors compound through a timeline extraction pipeline. Even when each individual module performed at 90%+ accuracy, end-to-end accuracy on complex sentences was below 60%, because errors are correlated — the same inputs that confuse stage k are the same inputs that confuse stage k+1.

The standard mitigation in NLP was k-best list passing: rather than passing the single best output of stage k to stage k+1, pass the k best outputs with their probabilities. This preserves uncertainty through the pipeline, but requires faithful probability estimates at each stage.

**Bayesian network approaches:**

Bayesian networks (Kennedy & O'Hagan 2001, calibration of multi-physics models arXiv:1206.5015) provide a principled treatment of uncertainty propagation through computational graphs. In a BN pipeline, each stage is a conditional probability distribution; calibration error at stage 1 means the conditional at stage 2 is mis-specified. The correct Bayesian treatment is to marginalize over the uncertainty in stage 1's output when computing stage 2's belief.

**HopCast (arXiv:2501.16587):** This 2025 paper explicitly addresses calibrated multi-step prediction in autoregressive dynamics models. Models trained to produce calibrated one-step predictions do not produce calibrated multi-step predictions via naive autoregression. The paper proposes uncertainty propagation methods that maintain calibration at K > 1 steps ahead. The key insight relevant for Curator: multi-step calibration requires explicit modeling of how uncertainty in step t feeds into uncertainty in step t+1.

**"Calibration Error for Decision Making" (arXiv:2404.13503):** Defines a decision-theoretic calibration metric called Calibration Decision Loss (CDL) — the maximum improvement in expected payoff obtainable by re-calibrating predictions. This is closer to what Curator needs than ECE: it measures calibration in terms of decision value.

**Honest assessment:** As of mid-2026, there is no standard framework for measuring or guaranteeing joint calibration across a multi-stage classification pipeline where each stage conditions on the previous stage's output. This is an open problem.

---

## Calibration in LLM Agents (2024–2026)

**"Agentic Confidence Calibration" — Zhang, Xiong, Wu (arXiv:2601.15778, January 2026, Salesforce Research)**

Identifies the foundational problem: existing calibration methods are built for static single-turn outputs and cannot address agentic systems. Introduces Holistic Trajectory Calibration (HTC), which analyzes entire agent trajectories rather than individual steps. The key diagnostic features span macro dynamics (did the trajectory converge or oscillate?) and micro stability (did confidence scores fluctuate between steps?). HTC trains a General Agent Calibrator (GAC) on trajectory-level features that generalizes across tasks without retraining. Achieves superior calibration on 8 benchmarks. This is the closest existing work to what Curator needs.

**"Agentic Uncertainty Quantification" (arXiv:2601.15703, January 2026)**

Proposes a System 1 / System 2 architecture: System 1 is an Uncertainty-Aware Memory that propagates verbalized confidence and semantic explanations through the context window; System 2 is an Uncertainty-Aware Reflection module that uses those explanations to trigger targeted re-inference when uncertainty is high. The key design principle — that uncertainty estimates must be *propagated through the context* in agentic systems, not computed independently at each step — is directly applicable to Curator.

**"The Confidence Dichotomy: Analyzing and Mitigating Miscalibration in Tool-Use Agents" — Xuan et al. (arXiv:2601.07264, January 2026)**

Identifies a tool-type-specific calibration split: evidence-retrieval tools (web search, document lookup) systematically cause overconfidence because retrieved information is noisy but the agent treats it as authoritative; verification tools (code interpreter, calculator) reduce miscalibration by providing deterministic feedback. For Curator: file metadata lookup (type, size, extension) is a verification-style tool and should anchor calibration; file content analysis is evidence-style and should be treated with skepticism.

**"Uncertainty Quantification in LLM Agents: Foundations, Emerging Challenges, and Opportunities" — Oh et al. (arXiv:2602.05073, accepted ACL 2026)**

The first unified formal framework for agent UQ. Identifies four open challenges: (1) selecting appropriate uncertainty estimators for LLMs, (2) handling uncertainty across heterogeneous entity types (structured metadata vs. unstructured content), (3) modeling how uncertainty *evolves* within interactive sequences, (4) evaluation benchmarks that match real multi-step scenarios. Introduces τ²-bench as an evaluation suite.

**"Confidence Calibration and Rationalization for LLMs via Multi-Agent Deliberation" (arXiv:2404.09127, 2024)**

Proposes using multiple LLM agents to discuss and reconcile confidence estimates before committing to a final answer. Deliberation among agents with diverse perspectives yields better-calibrated final confidence than any single agent. For Curator, this suggests that having the model independently re-estimate confidence using different reasoning paths (e.g., estimating from file metadata alone, then from file content, then combining) could improve pipeline-level calibration.

**Semantic Entropy (Kuhn et al., Nature 2024, arXiv:2302.09664):**

Generates multiple samples from the model, clusters them by semantic equivalence, then computes entropy over semantic clusters rather than raw tokens. Calibrated by construction under mild assumptions. Relevant for Curator's intent-inference step where paraphrases of the same intent should not be counted as separate uncertain hypotheses.

---

## Decision-Theoretic Threshold Selection

This is sub-problem J2 from Curator's open doors. The core question: given a calibrated probability P(correct destination | file), where should the threshold be set to separate HIGH (auto-move) / MEDIUM (ask) / LOW (skip) tiers?

**Chow's Reject Rule (1970):** The foundational result. For a binary classifier with misclassification cost c_e and rejection cost c_r, the optimal strategy is to classify when max posterior probability > 1 − c_r/c_e, and reject otherwise. This generalizes to K classes: reject when the top posterior probability falls below a threshold derived from the cost ratio. The threshold is not a hyperparameter — it is derived from the cost structure.

**Franc, Prusa, Voracek, "Optimal Strategies for Reject Option Classifiers" (arXiv:2101.12523, JMLR 2023):**

Shows that three ostensibly different rejection models (cost-based, bounded-improvement, bounded-coverage) all lead to the same optimal prediction strategy: a Bayes classifier with a randomized Bayes selection function. Any classifier that produces a proper uncertainty score can be used to build the optimal reject option without knowing the exact cost ratio at training time. The threshold can be adjusted post-hoc as the cost ratio changes.

**"The Foundations of Cost-Sensitive Learning" (Elkan, IJCAI 2001):**

Optimal cost-sensitive classification reduces to setting the probability threshold at c_FP / (c_FP + c_FN), where c_FP is the cost of a false positive and c_FN is the cost of a false negative. For Curator: if wrongly auto-moving a file costs 5× more (in user frustration, recovery effort) than unnecessarily asking, the threshold should be set at 1/(1+5) = 0.833, meaning auto-move only when confidence > 83.3%.

**"Calibration Error for Decision Making" (arXiv:2404.13503, 2024):**

Defines CDL as the max decision improvement obtainable from recalibration. Key result: if the model is already well-calibrated, the optimal threshold is exactly at the cost-ratio point above. If miscalibrated, the optimal threshold deviates, and the direction and magnitude depend on the direction and magnitude of miscalibration.

**For Curator specifically:** The cost asymmetry is not fixed — it depends on file category. Auto-moving a tax document to the wrong folder costs far more than auto-moving a meme. The pipeline should carry a per-file cost estimate and use it to select the threshold dynamically, not use a single global threshold for all file types.

---

## Cascading Errors in Classical ML Pipelines

Fonseca & Dang-Nguyen (CEUR-WS Vol-1386, 2015) established the empirical pattern: errors do not simply add — they multiply. A 5% error rate at stage 1 and 5% at stage 2 produces worse than 10% end-to-end error because errors are correlated.

**Mitigation strategies documented in literature:**

1. **K-best list passing** (Finkel et al. 2006): Pass the top-k outputs with probabilities rather than the 1-best. Preserves uncertainty but requires downstream models trained to consume distributions.
2. **Joint inference / end-to-end models**: Eliminate the pipeline entirely by training a single model. Reduces error propagation but sacrifices modularity and interpretability.
3. **Reinforcement learning for pipeline correction** (arXiv:1702.06794): Train stage k to be robust to the error distribution of stage k-1, rather than assuming clean inputs.
4. **Confidence-weighted features**: Pass not just the output of stage k but also its confidence score as a feature to stage k+1. Allows stage k+1 to discount unreliable upstream inputs.

---

## A Framework for Pipeline-Level Calibration in Curator

Based on the literature surveyed, a principled approach to measuring and improving calibration across Curator's four-step pipeline:

**Step 1: Instrument each stage independently using ECE + Reliability Diagrams.** Establish per-stage baselines. Maintain a rolling calibration set of (confidence, correct?) pairs for each of the four stages. Compute ECE weekly. Apply temperature scaling if ECE > 0.05.

**Step 2: Measure joint calibration via end-to-end ECE.** Rather than asking "was Step 3's folder selection calibrated?", ask "was the pipeline's final action correct, and what was the pipeline's stated confidence at each decision point?" Joint ECE is computed using the minimum confidence score across all four stages (the weakest link heuristic).

**Step 3: Adopt PASC for the folder-selection step.** PASC (Pipeline-Aware Split Conformal Prediction, arXiv:2605.18812, 2026) addresses multi-stage joint coverage directly. Its core insight: joint coverage of a K-stage pipeline is equivalent to standard coverage of the scalar joint maximum nonconformity score — the maximum nonconformity score across all K stages for a given input. This yields a finite-sample distribution-free guarantee that all K stages are simultaneously covered with probability ≥ 1 − α.

**Step 4: Propagate uncertainty explicitly.** At each pipeline stage, pass not only the top prediction but also the stage confidence and, where feasible, the top-k alternatives with their probabilities. Stage k+1 should be conditioned on this uncertainty distribution, not just the MAP estimate. This mirrors the HTC finding (arXiv:2601.15778) that trajectory-level calibration requires process-level features.

**Step 5: Set thresholds using cost-derived Chow rules, not grid search.** Estimate two costs for each file category: c_auto (cost of wrong autonomous action) and c_interrupt (cost of unnecessary user interruption). Optimal threshold: p* = 1 − c_interrupt / c_auto. For a low-stakes file: p* = 0.5. For a high-stakes file: p* = 0.9. The threshold is derived, not tuned.

**Step 6: Detect distribution shift as a calibration signal.** Following Ovadia et al. (NeurIPS 2019), maintain a reference distribution of calibration set inputs for each stage. When shift is detected, widen the prediction set (increase α) or fall back to user confirmation — the calibration guarantee no longer applies.

---

## Key References

1. Guo, C., Pleiss, G., Sun, Y., & Weinberger, K.Q. (2017). **On Calibration of Modern Neural Networks.** ICML 2017. arXiv:1706.04599.
2. Angelopoulos, A.N. & Bates, S. (2021). **A Gentle Introduction to Conformal Prediction and Distribution-Free Uncertainty Quantification.** arXiv:2107.07511.
3. Kotte, V. (2026). **PASC: Pipeline-Aware Conformal Prediction with Joint Coverage Guarantees for Multi-Stage NLP and LLM Pipelines.** arXiv:2605.18812.
4. Zhang, J., Xiong, C., & Wu, C.-S. (2026). **Agentic Confidence Calibration.** arXiv:2601.15778.
5. Oh, C. et al. (2026). **Uncertainty Quantification in LLM Agents: Foundations, Emerging Challenges, and Opportunities.** arXiv:2602.05073. ACL 2026.
6. Xuan, W. et al. (2026). **The Confidence Dichotomy: Analyzing and Mitigating Miscalibration in Tool-Use Agents.** arXiv:2601.07264.
7. (Zylos Research). (2026). **Agentic Uncertainty Quantification.** arXiv:2601.15703.
8. Kadavath, S. et al. (2022). **Language Models (Mostly) Know What They Know.** arXiv:2207.05221.
9. Ovadia, Y. et al. (2019). **Can You Trust Your Model's Uncertainty? Evaluating Predictive Uncertainty under Dataset Shift.** NeurIPS 32.
10. Franc, V., Prusa, D., & Voracek, V. (2021/2023). **Optimal Strategies for Reject Option Classifiers.** arXiv:2101.12523. JMLR 24.
11. Elkan, C. (2001). **The Foundations of Cost-Sensitive Learning.** IJCAI 2001.
12. Chow, C.K. (1970). **On Optimum Recognition Error and Reject Tradeoff.** IEEE Transactions on Information Theory, 16(1), 41–46.
13. Fonseca, E. & Dang-Nguyen, D.T. (2015). **When It's All Piling Up: Investigating Error Propagation in an NLP Pipeline.** CEUR-WS Vol-1386.
14. Corro, C. & Titov, I. (2017). **Tackling Error Propagation through Reinforcement Learning.** arXiv:1702.06794.
15. Parasuraman, R. & Manzey, D.H. (2010). **Complacency and Bias in Human Use of Automation.** Human Factors, 52(3), 381–410. DOI:10.1177/0018720810376055.
16. Kuhn, L., Gal, Y., & Farquhar, S. (2023/2024). **Semantic Uncertainty.** Nature. arXiv:2302.09664.
17. Amodei, A. et al. (2024). **Calibration Error for Decision Making.** arXiv:2404.13503.
18. Venn, A. et al. (2025). **HopCast: Calibration of Autoregressive Dynamics Models.** arXiv:2501.16587.
19. Liu, B. & Guo, C. (2024). **Confidence Calibration and Rationalization for LLMs via Multi-Agent Deliberation.** arXiv:2404.09127.
