# 52 — Explainability of File Organization Decisions
_Research date: 2026-05-31_

---

## Why This Matters for Curator

When Curator suggests moving `PHYS301_midterm_notes.pdf` to `University/Physics/`, the system needs to tell the user *why* — not just *what*. Without an explanation, the suggestion is a black box: users must either trust it blindly or reject it reflexively. Both outcomes are bad. Blind trust leads to misorganized files when Curator is wrong; reflexive rejection means users abandon the feature entirely. The explanation is the bridge between correct prediction and correct user behavior.

The challenge is that file organization explanations face a uniquely tight set of constraints simultaneously. They must be compact enough to fit in a notification or tooltip — not a paragraph. They must reference concepts the user already understands — not similarity scores or embedding distances. They must be multi-signal, because no single feature (filename, content, metadata, recency) is reliable alone. And they must be calibrated enough that users can detect when the explanation is wrong, which is the condition for appropriate reliance rather than over-reliance.

Finally, the problem is unsolved in the literature. There is no published framework for explaining local, multi-signal, personal-data-grounded file classification decisions at interactive latency on device. The closest analogues are XAI for document classification (LIME/SHAP on text features), natural language explanations in recommender systems ("because you liked X"), and compact UI patterns from CHI. Curator has an opportunity to synthesize these three streams into something purpose-built.

---

## XAI Foundations: LIME, SHAP, and Feature Importance

### LIME — Ribeiro, Singh, Guestrin (2016)

"Why Should I Trust You?: Explaining the Predictions of Any Classifier" (arXiv:1602.04938) introduced LIME as a model-agnostic local explainer. For any single prediction, LIME perturbs the input — in text, by randomly removing words — runs the black-box model on hundreds of perturbations, and fits a sparse linear model on the results. The coefficients of that linear model become the explanation: words with large positive weights pushed the prediction toward the predicted class; large negative weights pushed it away.

For text classification, LIME outputs a ranked list of word-importance pairs. The standard visualization highlights words in the original document in colors corresponding to sign and magnitude. This is inherently compact: a user can read "words 'midterm', 'physics', 'chapter 4' pushed toward Physics Assignments" in a single glance.

**Scalability caveat**: LIME requires hundreds of forward passes per explanation. For a local Ollama-style model this can be 5–30 seconds — too slow for interactive use. Two mitigation approaches: (1) use a fast surrogate or lighter embedding model for the perturbation pass, not the full classifier; (2) cache explanations — if the file hasn't changed and the folder hierarchy hasn't changed, the prior explanation is still valid. Mardaoui & Garreau (AISTATS 2021, arXiv:2010.12487) showed theoretically that LIME provides meaningful explanations for linear and tree-based models; convergence guarantees for deep models are weaker.

### SHAP — Lundberg & Lee (2017)

"A Unified Approach to Interpreting Model Predictions" (arXiv:1705.07874, NeurIPS 2017) unified six prior feature-attribution methods under the SHAP (SHapley Additive exPlanations) framework. SHAP values are the unique additive feature attribution satisfying three axioms: local accuracy, missingness, and consistency (derived from Shapley values in cooperative game theory).

The practical difference from LIME: SHAP provides both local (per-prediction) and global (per-model) importance. Global importance across a user's history tells you *which features typically matter for each folder*, which can pre-weight explanations. For file organization where features are a 5–15 element set (filename tokens, content keywords, file extension, recency, folder co-occurrence), KernelSHAP is tractable.

**Hybrid feature vectors for file classification:** Run LIME/SHAP not on raw token embeddings but on a structured feature vector:
- Filename tokens (e.g., "midterm", "2024", "PHYS301")
- Content keywords (top TF-IDF terms)
- File extension / MIME type (categorical)
- Recency signal (days since last access, normalized)
- Co-occurrence history (whether user has placed similar files in a given folder before)
- Folder name similarity (cosine of filename embedding vs. candidate folder name embedding)

With ~15 features, SHAP is fast and exact. The explanation becomes: "Top factors: filename contains 'PHYS301' (+0.4), co-occurrence with Physics folder (+0.3), content keywords 'wave function' (+0.2)." This translates into: *"Matches your Physics files by name and content."*

---

## Natural Language Explanations for Recommendations

### The Tintarev–Masthoff Framework (2007)

Tintarev and Masthoff, "A Survey of Explanations in Recommender Systems" (IEEE ICDE Workshops 2007, DOI:10.1109/ICDEW.2007.4401070). Seven aims: **effectiveness**, **satisfaction**, **transparency**, **scrutability**, **trust**, **persuasiveness**, **efficiency**. For Curator, *effectiveness*, *scrutability*, and *trust* are the priority aims. *Persuasiveness* is not a goal — we don't want to talk users into a wrong move.

The key implication: explanations for Curator should be *scrutinizable*. The user should be able to look at the explanation and immediately understand the basis well enough to say "no, that's wrong, this isn't a physics file." This argues against opaque single-score explanations ("confidence: 94%") and toward feature-grounded language ("because the filename says 'PHYS301'").

### Template vs. Free-Text NL Explanations

Most deployed recommendation explanations use templates: "Because you liked *X*, we recommend *Y*." Li et al., "Generate Natural Language Explanations for Recommendation" (arXiv:2101.03392), showed that free-text explanations referencing specific item features outperform generic templates on user satisfaction. For Curator: a template engine covers 90% of cases (*"Matches [N] other files in [folder] by [feature]."*); an LLM-based explanation generator handles edge cases.

### Explanation Bias and Persuasion Risk

"Measuring the Impact of Explanation Bias" (arXiv:2303.09498): the *framing* of NL explanations can shift user choices even when the underlying model didn't change. Evaluative language ("very similar to your important lecture notes") biases acceptance. Neutral feature language is safer: "content keywords match Lecture Notes folder (3 matching terms)."

### LLM-Generated Explanations as Rationalization

"Characterizing LLMs as Rationalizers" (ACL 2024 Findings): LLMs can generate plausible-sounding explanations that are not faithful to the underlying model — post-hoc rationalization rather than introspection. Safer pattern: run SHAP/LIME on the classifier to get actual feature importances, then use an LLM only to *verbalize* those importances. "From Feature Importance to Natural Language Explanations Using LLMs with RAG" (arXiv:2407.20990) implements exactly this architecture.

---

## Counterfactual Explanations

### DiCE — Mothilal, Sharma, Tan (2020)

"Explaining Machine Learning Classifiers through Diverse Counterfactual Explanations" (ACM FAccT 2020, DOI:10.1145/3351095.3372850). DiCE generates a *diverse* set of counterfactual examples: minimal perturbations to the input that flip the model's prediction.

For file organization, the counterfactual form is natural and intuitive: *"This file would be placed in 'Downloads/Misc' instead if its filename didn't contain 'PHYS'."* These are contrastive ("why here rather than there?") which aligns with Miller 2019's finding that humans naturally ask contrastive why-questions.

"A Comparative Analysis of Counterfactual Explanation Methods for Text Classifiers" (arXiv:2411.02643): methods that operate on semantically meaningful units (words, phrases) produce more user-intelligible counterfactuals than character-level perturbations.

For structured feature vectors (Curator's 15-feature case), finding the nearest counterfactual is a fast optimization problem. DiCE-Extended (arXiv:2504.19027) improves robustness for real-world deployments.

---

## Attention Mechanisms as Explanations

### The Jain & Wallace Claim (2019)

Jain and Wallace, "Attention is not Explanation" (NAACL 2019): attention weights do not correlate strongly with gradient-based feature importance measures, and adversarial attention distributions that achieve nearly identical predictions can exist. Conclusion: attention weights cannot be reliably used to explain model decisions.

### Wiegreffe & Pinter's Response (2019)

"Attention is not not Explanation" (arXiv:1908.04626, EMNLP 2019): challenged the methodology. Even when adversarial distributions technically *exist*, they are not found during normal training. Attention can serve as a plausible explanation for RNN-based models, qualified by the distinction between *faithfulness* (does attention reflect what the model actually used?) and *plausibility* (does it look reasonable to humans?).

### Current State of the Debate

"Is Attention Explanation? An Introduction to the Debate" (ACL 2022): attention is neither definitively explanation nor definitively not. For transformer self-attention in large models, attention alone is not a reliable faithful explainer. For shallow models with structured inputs — like Curator's file-feature attention — attention weights may carry more interpretive weight.

**Practical implication for Curator:** If using a transformer-based embedding model (e.g., fine-tuned MiniLM), attention-based explanations are unreliable without validation. The safer route is post-hoc LIME/SHAP attribution over the feature vector, which is architecture-agnostic.

---

## User Studies: Do Explanations Help?

### The Over-Reliance Problem

"Explanations Can Reduce Overreliance on AI Systems During Decision-Making" (arXiv:2212.06823, CSCW 2023): only explanations that help users *verify* the AI's reasoning — not just understand it — reduce over-reliance. Providing an explanation often triggers an *illusion of explanatory depth*: users feel they understand the model better than they do.

Wang et al., "Towards Human-centered Explainable AI: A Survey of User Studies for Model Explanations" (arXiv:2210.11584), aggregated findings from 86 papers. Key takeaways:
- Feature-based explanations (LIME/SHAP style) improve *subjective* understanding most consistently.
- Interactive, manipulatable explanations produce better trust calibration than static ones.
- Word highlighting in text classification was *harmful* to user trust in at least one study.
- The optimal explanation length is shorter than researchers typically assume.

### What Users Actually Want

"Help Me Help the AI" (CHI 2023, DOI:10.1145/3544548.3581001): users want *practically useful* information rather than technical system details. "Can I trust this specific decision?" not "How does the model work generally?" Strong argument for instance-level, grounded explanations over model-level descriptions.

### Lai & Tan (2019) Baseline

Lai and Tan, "On Human Predictions with Explanations and Predictions of Machine Learning Models" (arXiv:1811.07901, FAccT 2019): example-based explanations ("this is like file X that you put in folder Y") performed comparably to feature-based in user comprehension, and was preferred by non-technical users.

---

## Explanation Types for Curator's Context

### Taxonomy Applied to Curator's Level Router

**Level 1 — Rule-based (file extension, MIME type)**
Explanation type: Rule statement. Format: *"PDF file — matched to Documents folder."* Trivially explainable because the rule is transparent. No LIME/SHAP needed.

**Level 2 — Co-occurrence / collaborative filtering**
Explanation type: Example-based + frequency. Format: *"You've put 8 similar files here before"* or *"Matches your Physics folder (5 past placements)."* Fast to compute, intuitive to understand.

**Level 3 — Semantic embedding + cosine similarity**
Explanation type: Feature attribution (SHAP on structured features) + NL verbalization. Format: *"Content keywords match Lecture Notes (wave function, interference, lab)."*

**Counterfactual override layer (all levels):** *"If the filename didn't contain 'draft', this would go to Final Reports."* Surface only when predicted confidence is below threshold.

### Tintarev–Masthoff Goals Mapping for Curator

Goal priority order: **scrutability > effectiveness > trust > transparency**. Persuasiveness deprioritized — Curator should not try to talk the user into a bad move.

---

## Compact UI Patterns for Explanations

### The One-Line Rationale Pattern

The most relevant UI primitive is the **one-line rationale**: a single sentence (ideally under 60 characters) appended to the suggestion. Used in: email client priority flags ("Marked important because of frequent conversations with Alice"), news feed ranking, bank transaction categorization.

Structure of an effective one-liner: `[Action verb] + [specific grounding signal] + [optional count/strength]:`
- *"Matches your Physics files by name"*
- *"3 similar PDFs already in Lecture Notes"*
- *"Content contains 'lab report', 'yield', 'titration'"*
- *"Would go to Reports if name lacked 'draft'"* (counterfactual variant)

### Progressive Disclosure Pattern

CHI 2021 XAI interface design (ACM DOI:10.1145/3411763.3451759): show compact summary explanation by default, with a "why?" expansion affordance for users who want detail. For Curator's suggestion card:
- **Primary line:** File destination path
- **Secondary line** (always visible): One-line rationale
- **Expandable detail:** SHAP bar chart or ranked feature list

### Trust Calibration Signals

"Explainable recommendation: when design meets trust calibration" (PMC:8327305): showing confidence alongside explanation improves trust calibration. For Curator, a three-level confidence icon (high / medium / uncertain) paired with the one-liner gives users a quick signal about when to scrutinize more carefully.

### Anti-Patterns to Avoid

- **Numeric scores alone** ("similarity: 0.87") — unintelligible and not actionable.
- **Over-highlighting** — showing every contributing word buries the signal. Maximum 3 highlighted terms.
- **Explanation overload** — a full SHAP plot in a notification increases cognitive load without improving decisions.
- **Evaluative framing** — "important file", "highly relevant" — biases acceptance regardless of accuracy.

---

## Open Problems

1. **Latency at local inference.** LIME and SHAP both require multiple forward passes. No published work addresses explanation generation latency for local-first, privacy-preserving document classifiers at interactive speed.

2. **Multi-signal feature fusion explanation.** Existing XAI work treats single modality. File classification is multi-signal (name + content + metadata + temporal patterns + folder history). No framework handles explaining the *interaction* between these signals — especially when signals conflict (filename says "Physics" but content is about Chemistry).

3. **Personalized explanation vocabulary.** Users have their own taxonomy — folders named by project codes, abbreviations, dates meaningful only to them. Explanations in generic terms ("academic document") may not resonate. No published system grounds explanations in the user's *own* folder names for local file organization.

4. **Explanation stability over time.** As the user's file collection grows, SHAP values and folder co-occurrence signals shift. There is no published work on explanation drift for personal file organization systems.

5. **Non-text files.** For binary or media files (images, audio, video, executables), explanation signals collapse to filename and metadata only. Explainability for non-text personal files is an open research problem.

6. **Ground-truth for explanation evaluation.** No public dataset exists for evaluating whether an explanation of a file organization decision matches the user's actual intent.

---

## Key References

1. Ribeiro, M.T., Singh, S., Guestrin, C. (2016). "Why Should I Trust You?": Explaining the Predictions of Any Classifier. arXiv:1602.04938. KDD 2016.
2. Lundberg, S.M., Lee, S. (2017). A Unified Approach to Interpreting Model Predictions. arXiv:1705.07874. NeurIPS 2017.
3. Jain, S., Wallace, B.C. (2019). Attention is not Explanation. NAACL 2019.
4. Wiegreffe, S., Pinter, Y. (2019). Attention is not not Explanation. arXiv:1908.04626. EMNLP 2019.
5. Miller, T. (2019). Explanation in Artificial Intelligence: Insights from the Social Sciences. arXiv:1706.07269. *Artificial Intelligence*, 267, 1–38.
6. Adadi, A., Berrada, M. (2018). Peeking Inside the Black-Box: A Survey on XAI. *IEEE Access*, 6, 52138–52160. DOI:10.1109/ACCESS.2018.2870052.
7. Mothilal, R.K., Sharma, A., Tan, C. (2020). Explaining Machine Learning Classifiers through Diverse Counterfactual Explanations. FAccT 2020. DOI:10.1145/3351095.3372850.
8. Tintarev, N., Masthoff, J. (2007). A Survey of Explanations in Recommender Systems. IEEE ICDE Workshops 2007. DOI:10.1109/ICDEW.2007.4401070.
9. Li, P., et al. (2021). Generate Natural Language Explanations for Recommendation. arXiv:2101.03392.
10. Mardaoui, D., Garreau, D. (2021). An Analysis of LIME for Text Data. AISTATS 2021. arXiv:2010.12487.
11. Wang, X., et al. (2022). Towards Human-centered Explainable AI. arXiv:2210.11584.
12. "Help Me Help the AI." CHI 2023. DOI:10.1145/3544548.3581001.
13. Lai, V., Tan, C. (2019). On Human Predictions with Explanations. arXiv:1811.07901. FAccT 2019.
14. Samek, W., et al. (2023). Explainable Artificial Intelligence (XAI) 2.0. arXiv:2310.19775.
15. "From Feature Importance to Natural Language Explanations Using LLMs with RAG." arXiv:2407.20990.
16. "Measuring the Impact of Explanation Bias." arXiv:2303.09498. SIGIR 2023.
17. "A Comparative Analysis of Counterfactual Explanation Methods for Text Classifiers." arXiv:2411.02643.
18. "Is Attention Explanation? An Introduction to the Debate." ACL 2022.
19. "Explanations Can Reduce Overreliance on AI Systems." arXiv:2212.06823. CSCW 2023.
20. DiCE-Extended. arXiv:2504.19027.
