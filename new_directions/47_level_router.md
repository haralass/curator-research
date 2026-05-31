# 47 — Level Router: Adaptive Tiered Analysis for File Classification
_Research date: 2026-05-31_

---

## Why This Matters for Curator

Curator's Level Router is a six-tier adaptive analysis system that decides, for each file, the minimum computation needed to classify it confidently. Level 0 needs only a filename and extension; Level 5 calls a large language model. Between them lie progressively more expensive analyses — MIME/xattr metadata, lightweight semantic embeddings, full document embeddings, and a small LLM — with escalation triggered only when confidence at the current tier falls below a threshold. The router therefore trades compute against uncertainty rather than spending the same budget on every file regardless of complexity.

This design is novel in the Personal Information Management (PIM) literature. Existing PIM tools either apply a single analysis strategy to all files (Hazel uses rule-based patterns; most macOS file indexers rely on Spotlight's uniform metadata pass) or hand off to a single LLM unconditionally (e.g., Sortio's "content sorting toggle"). No published PIM system exposes an explicit confidence-gated escalation ladder. The Level Router fills that gap by borrowing and adapting three converging research traditions: LLM routing/cascading systems that dynamically select between strong and weak models, early-exit neural networks that halt inference at the layer where entropy is already low, and classical attentional cascade classifiers that reject easy negatives early and spend computation only on ambiguous candidates.

The key engineering bet is that the cost distribution of real file collections is extremely skewed: the vast majority of files (`.png`, `.mp4`, `README.md`) are unambiguously classifiable from filename + extension alone, a minority require a metadata pass, and only a small tail of heterogeneous or ambiguous documents need LLM inference. An adaptive tier router converts that skew into direct latency and cost savings without degrading classification accuracy on hard cases.

---

## Prior Art: LLM Routing Systems

### RouteLLM: Learning to Route LLMs with Preference Data
**Authors:** Isaac Ong, Amjad Almahairi, Vincent Wu, Wei-Lin Chiang, Tianhao Wu, Joseph E. Gonzalez, M. Waleed Kadous, Ion Stoica (LMSYS / UC Berkeley)
**arXiv:** 2406.18665 (2024)
**GitHub:** https://github.com/lm-sys/RouteLLM

RouteLLM trains lightweight router models on human preference data (from Chatbot Arena) to decide whether a given query should go to a strong model (GPT-4-class) or a weak model (Mixtral-8x7B-class). Four router architectures are provided: a BERT classifier, a matrix-factorisation model, a cosine-similarity model, and an LLM-judge model. The key result: >2× cost reduction with no measurable quality loss on MT-Bench and MMLU. Transfer learning is strong — routers trained on one model pair generalise to unseen pairs.

**Relevance to Curator:** The preference-data training paradigm is not directly applicable (Curator has no human-labelled "is level 2 sufficient?" corpus), but RouteLLM's confidence-score API and threshold-sweep evaluation methodology are directly reusable. The binary router can be generalised to a multi-class escalation decision if reformulated as a sequence of binary gates (escalate? yes/no at each tier).

---

### FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance
**Authors:** Lingjiao Chen, Matei Zaharia, James Zou (Stanford)
**arXiv:** 2305.05176 (2023; updated in TMLR 2024)

FrugalGPT introduces three complementary strategies — prompt adaptation, LLM approximation, and LLM cascade — and provides a formal instantiation of the cascade. The cascade learns, per query type, which sequence of LLMs to try and when to accept an answer. Key result: matches GPT-4 performance at up to 98% lower cost, or improves over GPT-4 by 4 percentage points at the same cost, on benchmarks including HellaSwag, BoolQ, and NaturalQuestions. The scoring function that decides whether to accept or escalate is a learned "judge" over LLM output quality.

**Relevance to Curator:** FrugalGPT is the closest academic ancestor to the Level Router. The cascade structure (cheapest model first, escalate on low-quality signal) maps directly onto Curator's tier structure. The trained judge maps onto Curator's confidence estimator. The 98% cost reduction result motivates the architecture: if LLM cascades save that much cost on NLP benchmarks, the savings on a file-classification workload — where most files need no LLM at all — should be even more pronounced.

---

### A Unified Approach to Routing and Cascading for LLMs
**Authors:** Jasper Dekoninck, Maximilian Baader, Martin Vechev (ETH Zürich SRI Lab)
**arXiv:** 2410.10347 (2024; accepted ICML 2025)

This paper unifies routing (pick one model) and cascading (try cheaper first, escalate on failure) under a single theoretical framework called "cascade routing." It derives a provably optimal strategy for cascading and proves the optimality of an existing routing strategy, then shows that combining both further improves the Pareto frontier of cost vs. accuracy. A formal decision-theoretic analysis identifies the conditions under which each strategy dominates.

**Relevance to Curator:** Directly applicable. The proof-of-optimality for the cascade strategy provides theoretical grounding for the Level Router's escalation rule. The unified framework shows that Curator could mix routing (e.g., always skip to Level 4 for `.eml` files regardless of confidence) with cascading (standard escalation for ambiguous files).

---

### CARGO: Confidence-Aware Routing with Gap-based Optimization
**Authors:** (ETH/CASCON 2025)
**arXiv:** 2509.14899

CARGO uses a single embedding-based regressor trained on LLM-judged pairwise comparisons to predict model performance, plus an optional binary classifier invoked when predictions are uncertain. This two-stage design avoids human-annotated supervision and handles mid-tier specialist models, not just "small vs large."

**Relevance to Curator:** The two-stage design (regressor for easy decisions, classifier for borderline cases) mirrors Level 0–1 (heuristic) + Level 2–3 (embedding) + Level 4–5 (LLM) tiers. The technique of using the LLMs themselves as quality judges to generate training signal is adoptable for Curator's confidence calibration.

---

### Dynamic Model Routing and Cascading for Efficient LLM Inference: A Survey
**Authors:** Yasmin Moslem, John D. Kelleher (ADAPT Centre / Trinity College Dublin)
**arXiv:** 2603.04445 (2026)

A comprehensive survey characterising routing systems along three axes: *when* decisions are made (pre-inference, mid-inference, or post-inference), *what* information informs decisions (query text, difficulty heuristics, model confidence, learned classifiers), and *how* computations are executed (parallel, sequential, hybrid). Covers RouteLLM, FrugalGPT, RouterDC, C2MAB-V, P2L, UniRoute, and others. Notes that well-designed routing outperforms even the strongest individual model by exploiting complementary specialisations.

**Relevance to Curator:** The when/what/how taxonomy is a useful design checklist for the Level Router. Curator's router is pre-inference (it selects the tier before running it), uses difficulty heuristics plus learned confidence, and executes sequentially — this maps cleanly onto the survey's vocabulary.

---

### Mixture-of-Agents Enhances Large Language Model Capabilities
**Authors:** Junlin Wang et al. (Together AI)
**arXiv:** 2406.04692 (2024; ICLR 2025)

Mixture-of-Agents (MoA) stacks multiple LLM agents in layers: each agent receives all outputs from the previous layer as context. Exploits "collaborativeness of LLMs" — even weak models improve when conditioned on peer outputs. Achieves 65.1% on AlpacaEval 2.0 (vs. 57.5% for GPT-4o) using only open-source models.

**Relevance to Curator:** MoA is an *ensemble* rather than a cascade, so its cost model is inverted (it spends more compute, not less). However, the collaborativeness insight is relevant if Curator's Level 4/5 inference is structured as: Level 4 generates a candidate label that Level 5 refines, rather than Level 5 running blind.

---

## Prior Art: Cascading Classifiers & Early-Exit Networks

### Viola & Jones: Rapid Object Detection Using a Boosted Cascade of Simple Features
**Authors:** Paul Viola, Michael Jones
**Venue:** IEEE CVPR 2001, proceedings pp. I-511–I-518

The foundational attentional cascade: AdaBoost selects Haar-like features; classifiers are arranged in a cascade where each stage quickly rejects negative windows and passes only candidates to the next (more expensive) stage. Background regions are discarded in 2–3 cheap features; face regions accumulate 30+ features of progressively refined analysis. Real-time face detection (15 fps) was the result.

**Relevance to Curator:** The Viola-Jones cascade is the structural ancestor of the Level Router. Key design lesson: calibrate each tier so its false-negative rate (failing to escalate a hard file) is near zero, concentrating the computational budget on genuinely hard cases. The "attentional cascade" framing is directly citable in any Curator paper to establish classical prior art.

---

### BranchyNet: Fast Inference via Early Exiting from Deep Neural Networks
**Authors:** Surat Teerapittayanon, Bradley McDanel, H.T. Kung
**arXiv:** 1709.01686 (2017); also IEEE ICPR 2016

BranchyNet augments a deep network with side-branch classifiers at intermediate layers. At each branch, the entropy of the softmax distribution is compared against a learned threshold. If entropy is low (high confidence), the sample exits and produces a prediction without executing further layers. BranchyNet achieves up to 5× inference speedup with <1% accuracy drop on MNIST/CIFAR-10.

**Relevance to Curator:** The entropy-based exit criterion is directly applicable as Curator's per-tier confidence gate. The "side branch at layer k" metaphor maps exactly onto "run level k analysis, check entropy, exit or escalate."

---

### DeeBERT: Dynamic Early Exiting for Accelerating BERT Inference
**Authors:** Zheng Xin Yong, Yue Dong, Jackie Chi Kit Cheung
**arXiv:** 2004.12993 (2020); ACL 2020

DeeBERT applies BranchyNet's exit strategy to transformer encoders: intermediate [CLS] representations are passed to per-layer classifiers; if softmax entropy falls below a threshold the token exits early. Saves up to 40% inference time on GLUE benchmarks with minimal accuracy loss. Two variants: fixed threshold and per-layer adaptive threshold.

**Relevance to Curator:** If Curator uses a BERT/SentenceTransformer-style encoder at Level 2–3, DeeBERT shows how to make the embedding model itself adaptive — potentially collapsing Levels 2 and 3 into a single model with two exit points.

---

### BERT Loses Patience: Fast and Robust Inference with Early Exit
**Authors:** Wangchunshu Zhou, Canwen Xu, Tao Ge, Julian McAuley, Ke Xu, Furu Wei
**arXiv:** 2006.04152 (2020); NeurIPS 2020

Patience-based Early Exit (PABEE): instead of a single-layer entropy threshold, the model exits when the prediction has been consistent for *p* consecutive layers. More robust to calibration noise than entropy alone; p=3–6 provides a good tradeoff.

**Relevance to Curator:** The patience mechanism is a strong alternative to pure confidence thresholding for Curator's escalation decision. A "level stays at 2 for 3 attempts without new information" rule is the discrete analogue.

---

### Multi-Scale Dense Networks (MSDNet) for Resource-Efficient Image Classification
**Authors:** Gao Huang, Danlu Chen, Tianhao Li, Felix Wu, Laurens van der Maaten, Kilian Weinberger
**arXiv:** 1703.09844 (2017); ICLR 2018 Oral

MSDNet trains a single network with multiple classifiers of increasing depth/scale, connected by dense skip connections. Addresses two scenarios: *anytime classification* (produce the best prediction at whatever time budget is available) and *budgeted batch classification* (spend computation unevenly across easy vs hard inputs). Demonstrates that multi-scale features prevent coarse early classifiers from corrupting fine-grained later ones — a crucial training concern.

**Relevance to Curator:** The budgeted batch scenario exactly matches Curator's workload (classify a folder of N files with a total compute budget). MSDNet's training methodology — ensuring inter-tier independence — should inform how Curator trains level-specific classifiers so they are not biased toward features only available at higher tiers.

---

### EENet: Learning to Early Exit for Adaptive Inference
**Authors:** Fatih Ilhan et al.
**arXiv:** 2301.07099 (2023)

EENet learns an exit *policy* (modelled as a lightweight policy network) rather than applying a fixed entropy threshold. The policy is trained to minimise expected inference cost subject to accuracy constraints, evaluated on CIFAR-10/100, ImageNet, SST-2, AgNews.

**Relevance to Curator:** Replacing hard-coded confidence thresholds with a learned exit policy is a future upgrade path for the Level Router; the EENet policy can be retrained as user corpora evolve.

---

### Dynamic Neural Networks: A Survey
**Authors:** Yizeng Han, Gao Huang, Shiji Song, Le Yang, Honghui Wang, Yulin Wang
**arXiv:** 2102.04906 (2021)

Comprehensive survey categorising dynamic networks into instance-wise (vary architecture per input), spatial-wise (vary computation per spatial location), and temporal-wise (vary computation over time). Covers early-exit, mixture-of-experts, dynamic convolutions, and more.

**Relevance to Curator:** Instance-wise dynamic models are the direct category for the Level Router. The survey's taxonomy and evaluation conventions (accuracy vs. FLOPs curves, budgeted inference plots) should be adopted for Curator's evaluation section.

---

### A Survey of Early Exit Deep Neural Networks in NLP
**Authors:** Divya Jyoti Bajpai, Manjesh Kumar Hanawal
**arXiv:** 2501.07670 (January 2025)

Covers NLP-specific early-exit methods, exit criteria (entropy, patience, confidence gap, learned policy), training strategies (joint training vs. post-hoc insertion), and evaluation across GLUE/SuperGLUE tasks. Discusses calibration failures and adversarial robustness of early-exit systems.

**Relevance to Curator:** The most up-to-date survey of early-exit design choices in the transformer era; directly applicable to Levels 2–4 of Curator if those tiers use transformer-based models.

---

## Decision-Theoretic Stopping Rules

### Wald's Sequential Probability Ratio Test (SPRT)
**Reference:** Abraham Wald, "Sequential Analysis," John Wiley & Sons, 1947.

The SPRT defines the optimal stopping rule for sequential hypothesis testing: after each observation, compute the likelihood ratio; stop and accept H₁ if it exceeds an upper threshold, stop and accept H₀ if it falls below a lower threshold, otherwise take another observation. Wald proved this minimises expected sample size for given error rates. The cost of information is proportional to the expected change in log-likelihood ratio (KL divergence).

**Relevance to Curator:** Each tier in the Level Router is an "observation." The SPRT provides the formal framework for the stopping decision: escalate only when the current evidence (confidence score) is in the "continue sampling" band. Setting tier thresholds according to SPRT theory converts the Level Router into a provably optimal sequential test.

---

### Anytime Algorithms and Pareto-Optimal Stopping
**Reference:** Shlomo Zilberstein, "Using Anytime Algorithms in Intelligent Systems," AI Magazine, 1996.
**Additional:** "Pareto-Optimal Anytime Algorithms via Bayesian Racing," arXiv 2603.08493 (2026)

Anytime algorithms produce increasingly refined solutions as computation time increases, with a monotone quality-time profile. Curator's Level Router is an *anytime classifier*: it can be interrupted after any tier and still return a best-effort label. The anytime literature provides evaluation methodology (quality-time performance profiles, area under the performance curve) and design principles (interruptibility, monotone quality improvement).

---

## Formal Evaluation Methodology

Evaluating adaptive tiered systems requires richer methodology than single-point accuracy, because the operating point (which threshold to set at each tier) is a free parameter. The following metrics and conventions are standard:

**1. Quality-Cost Curves (QCC) / Performance-FLOPs Curves**
Plot accuracy (or F1) on the y-axis against average per-sample FLOPs (or inference latency, or API cost) on the x-axis, sweeping threshold parameters. Introduced by MSDNet [arXiv:1703.09844] and adopted universally in early-exit literature. For Curator, cost can be in milliseconds or monetary units (LLM API call cost).

**2. Area Under the Quality-Cost Curve (AUQCC)**
Aggregate scalar metric for comparing two adaptive systems across all operating points. Used as primary metric in RouteLLM [arXiv:2406.18665].

**3. Pareto Frontier Analysis**
Identify operating points where no reallocation of compute improves quality without degrading it, or vice versa. Used in "A Unified Approach to Routing and Cascading" [arXiv:2410.10347] to prove cascade routing is Pareto-optimal. Curator should plot its Level Router's frontier against fixed-tier baselines.

**4. Budgeted Batch Evaluation**
Fix a total compute budget for a test corpus and measure accuracy when the router allocates it optimally. Introduced by MSDNet [arXiv:1703.09844]. Appropriate for Curator's folder-scan workload.

**5. Confidence Calibration (ECE/MCE)**
Expected Calibration Error measures whether a model's stated confidence matches its empirical accuracy. Tier-exit decisions are only valid if confidence scores are calibrated. See Guo et al., "On Calibration of Modern Neural Networks," ICML 2017.

**6. Per-Tier Escalation Rate**
Report the fraction of files that exit at each tier. If >90% of files exit at Level 0–1, the system is working as designed. Sharp mode collapses (e.g., 99% at Level 0) suggest the router is under-utilising higher tiers; flat distributions suggest thresholds are too loose.

**7. Decision Boundary Analysis**
For each tier, report the confusion matrix conditioned on the files that exited at that tier. This reveals whether escalation is correctly targeted at hard cases.

---

## Gap Analysis: What Doesn't Exist

1. **No adaptive tiered analysis system for local file organisation.** Every published PIM system and macOS AI file organiser (Sortio, Hazel, Sparkle, Finder Smart Folders) uses a single analysis strategy. Sortio's "content sorting toggle" is the closest — but it is binary (on/off), not confidence-gated, and does not route on a per-file basis. The Level Router is the first system to apply confidence-gated multi-tier escalation to file classification.

2. **No application of SPRT or Bayesian stopping rules to file/document classification.** The cascading classifier literature (Viola-Jones, AdaBoost) and the LLM cascade literature (FrugalGPT, RouteLLM) both use heuristic or learned confidence thresholds rather than decision-theoretically optimal stopping rules derived from SPRT. Curator could be the first to provide principled threshold derivation.

3. **No cascade that spans heterogeneous feature modalities (filename → MIME → embedding → LLM).** Existing cascades always route within a single modality (LLM to LLM in RouteLLM; layer to layer in early-exit nets; Haar-feature to Haar-feature in Viola-Jones). Curator's cascade crosses modality boundaries (string matching → MIME lookup → vector similarity → language inference), which introduces calibration challenges not studied in existing literature.

4. **No formal cost model for the metadata vs. content vs. LLM tradeoff.** FrugalGPT models LLM API cost in dollars. Early-exit models count FLOPs within a single neural network. Curator's cost spans three orders of magnitude (nanoseconds for filename match; tens of seconds for large LLM) and involves heterogeneous cost units (I/O, CPU, API tokens). No existing framework handles this mixed-unit cost model.

5. **No early-exit literature applied to classification tasks where the input representation itself changes between tiers.** In all existing early-exit networks, every exit point sees the same (or a subsumed) input representation. In Curator, Level 1 sees MIME/xattr data not visible to Level 0, and Level 3 sees full document content not visible to Level 2. This "expanding observation" structure is closer to SPRT than to neural early-exit, and is not studied as a unified system elsewhere.

6. **No ablation study comparing adaptive tiering vs. uniform LLM-for-all on a real file corpus.** Sortio and similar products have no published benchmarks. The literature gap means Curator's first evaluation will constitute the baseline for the entire sub-field.

---

## Implementation Pointers

**LLM Routing Infrastructure**
- **RouteLLM GitHub:** https://github.com/lm-sys/RouteLLM — open-source framework with 4 trained router models. The `RouteLLM` Python API supports threshold sweeping and quality-cost curve generation out of the box.
- **LiteLLM Router:** https://github.com/BerriAI/litellm — production-grade LLM proxy with built-in cost-based routing; relevant for Levels 4–5 provider switching.

**Embedding / Semantic Similarity (Levels 2–3)**
- **sentence-transformers:** https://github.com/UKPLab/sentence-transformers — lightweight SBERT models usable for Level 2 snippet embedding. `all-MiniLM-L6-v2` (22M params) runs at ~5ms on CPU per snippet.
- **FAISS / Hnswlib:** fast nearest-neighbour lookup for embedding-based classification without LLM inference.

**Early-Exit Confidence Estimation**
- **BranchyNet reference implementation:** https://github.com/kunglab/branchynet
- **DeeBERT implementation:** https://github.com/castorini/DeeBERT — shows how to instrument transformer layers with entropy-based exits. Adaptable for Levels 2–3.

**Threshold Calibration**
- **Netcal (calibration library):** https://github.com/EFS-OpenSource/calibration-framework — ECE/MCE computation and post-hoc calibration (temperature scaling, Platt scaling) for per-tier confidence scores.
- **scikit-learn `CalibratedClassifierCV`:** straightforward isotonic regression or sigmoid calibration applicable to Level 0–2 confidence scores.

**macOS Metadata Access (Levels 0–1)**
- **`xattr` Python package:** reads extended attributes (com.apple.quarantine, kMDItemContentType, etc.) — the primary signal for Level 1.
- **`file` command / libmagic:** MIME type detection without reading file content (Level 1). Python binding: `python-magic`.
- **`mdfind` / Spotlight API:** CoreSpotlight index gives pre-computed metadata including `kMDItemContentType`, `kMDItemKind`, last-used date — accessible at Level 1 with zero I/O cost.

**Document Classification via Filename Only (Level 0 reference)**
- arXiv:2410.01166 (Li, Larson, Leach 2024) shows TF-IDF + supervised learner on filename alone achieves 99.63% accuracy at 442× speedup vs. DiT. Directly applicable as Curator's Level 0 classifier.

---

## Key References

1. Ong, I. et al. "RouteLLM: Learning to Route LLMs with Preference Data." arXiv:2406.18665, 2024.
2. Chen, L., Zaharia, M., Zou, J. "FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance." arXiv:2305.05176, 2023 (TMLR 2024).
3. Dekoninck, J., Baader, M., Vechev, M. "A Unified Approach to Routing and Cascading for LLMs." arXiv:2410.10347, 2024 (ICML 2025).
4. Wang, J. et al. "Mixture-of-Agents Enhances Large Language Model Capabilities." arXiv:2406.04692, 2024 (ICLR 2025).
5. Moslem, Y., Kelleher, J.D. "Dynamic Model Routing and Cascading for Efficient LLM Inference: A Survey." arXiv:2603.04445, 2026.
6. "CARGO: Confidence-Aware Routing with Gap-based Optimization." arXiv:2509.14899, 2025 (CASCON 2025).
7. Viola, P., Jones, M. "Rapid Object Detection Using a Boosted Cascade of Simple Features." IEEE CVPR 2001.
8. Teerapittayanon, S., McDanel, B., Kung, H.T. "BranchyNet: Fast Inference via Early Exiting from Deep Neural Networks." arXiv:1709.01686, 2017.
9. Yong, Z.X., Dong, Y., Cheung, J.C.K. "DeeBERT: Dynamic Early Exiting for Accelerating BERT Inference." arXiv:2004.12993, 2020 (ACL 2020).
10. Zhou, W. et al. "BERT Loses Patience: Fast and Robust Inference with Early Exit." arXiv:2006.04152, 2020 (NeurIPS 2020).
11. Huang, G. et al. "Multi-Scale Dense Networks for Resource Efficient Image Classification." arXiv:1703.09844, 2017 (ICLR 2018 Oral).
12. Ilhan, F. et al. "EENet: Learning to Early Exit for Adaptive Inference." arXiv:2301.07099, 2023.
13. Han, Y. et al. "Dynamic Neural Networks: A Survey." arXiv:2102.04906, 2021.
14. Bajpai, D.J., Hanawal, M.K. "A Survey of Early Exit Deep Neural Networks in NLP." arXiv:2501.07670, January 2025.
15. Li, Z., Larson, S., Leach, K. "Document Classification using File Names." arXiv:2410.01166, 2024.
16. Wald, A. "Sequential Analysis." John Wiley & Sons, 1947.
17. Zilberstein, S. "Using Anytime Algorithms in Intelligent Systems." AI Magazine 17(3), 1996.
18. Guo, C. et al. "On Calibration of Modern Neural Networks." ICML 2017. arXiv:1706.04599.
19. "Pareto-Optimal Anytime Algorithms via Bayesian Racing." arXiv:2603.08493, 2026.
20. "Adaptive Inference through Early-Exit Networks: Design, Challenges and Directions." arXiv:2106.05022, 2021.
