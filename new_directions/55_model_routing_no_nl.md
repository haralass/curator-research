# 55 — Model Routing Without Natural Language Input
_Research date: 2026-05-31_

---

## Why This Matters for Curator

Curator is a local AI file organizer for macOS that must decide, for every file it encounters, whether to invoke a small fast model or a large accurate one. The decision must be made before any LLM call happens — at intake — and the only available signals are structural: file extension, MIME type, file size, creation timestamp, and (for some binary files) nothing else at all. This is a fundamentally different routing problem from everything published in the LLM routing literature.

Every published routing system — RouteLLM, FrugalGPT, Mixture-of-Agents, IRT-Router, ParetoBandit, and the dozens of papers from 2024–2025 surveyed below — is designed around one invariant: a natural language query exists and can be embedded. The routing signal is always some transformation of text: a semantic embedding, a perplexity score, a BERT classification of query difficulty, or a similarity to previous text queries. None of these signals exist when the input is a `.pkl` file or a `.heic` photo with no OCR text. The gap is not a matter of degree; it is a categorical mismatch between what the literature addresses and what Curator needs.

Three sub-problems compound the challenge. First (I1), Curator must route on metadata alone — no readable text content is guaranteed. Second (I2), as Curator's model pool grows (e.g., a specialist vision model for images, a code model for `.py` files), the router must absorb a new specialist without re-running preference-data collection campaigns. Third (I3), before relying on a new specialist, Curator needs to cheaply characterize where that model succeeds and fails — its capability boundary — using only a handful of test files rather than thousands.

---

## Existing LLM Routing Systems: RouteLLM, FrugalGPT, Mixture-of-Agents

### RouteLLM
**Authors:** Isaac Ong et al. (LMSYS, UC Berkeley)
**arXiv:** 2406.18665 (2024); GitHub: https://github.com/lm-sys/RouteLLM

**Routing mechanism:** RouteLLM trains one of four router architectures over human preference data drawn from the Chatbot Arena. The four architectures are:
1. **Similarity-Weighted (SW) Ranking** — cosine similarity between the incoming query embedding and all historical Arena queries, weighted by a Bradley–Terry win-probability model. Uses OpenAI `text-embedding-3-small`.
2. **Matrix Factorization** — learns a bilinear scoring function over query and model embeddings. Query embedding uses `text-embedding-3-small`.
3. **BERT Classifier** — BERT-base (768-dim [CLS] token) over the user query, followed by logistic regression.
4. **Causal LLM Classifier** — A fine-tuned small LLM reads the query as an instruction prompt and outputs a binary routing decision.

**Why it cannot handle metadata-only input:** All four architectures require text to exist. Feeding an empty string or a fabricated pseudo-query ("file: photo.heic, size: 4.2MB") would produce arbitrary results uncalibrated to routing quality. RouteLLM's training data is 100% human-written text conversations; there is no provision for non-linguistic inputs.

---

### FrugalGPT
**Authors:** Lingjiao Chen, Matei Zaharia, James Zou (Stanford)
**arXiv:** 2305.05176 (2023); TMLR 2024

**Routing mechanism:** LLM cascade with a learned scoring function g(q, a) ∈ [0,1]. At each stage, the cheapest model is queried first; the scoring function evaluates whether its answer is "good enough." The scoring function is implemented as a DistilBERT regression model trained with (query + response) as input.

**Why it cannot handle metadata-only input:** FrugalGPT routes on the pair (query, answer) — it requires both a natural language question and a generated answer. It is a post-response cascade, not a pre-routing system. Even re-interpreted as a pre-routing system, the scoring function is a language model over text tokens; structural metadata has no representation in its vocabulary.

---

### Mixture-of-Agents (MoA)
**Authors:** Junlin Wang et al. (Together AI)
**arXiv:** 2406.04692 (2024)

**Routing mechanism:** MoA is not a router in the traditional sense — it builds a layered pipeline where every model in each layer processes the query and all previous layer outputs. All agents are always called. There is no cost-saving dispatch.

**Why it cannot handle metadata-only input:** MoA is architecturally orthogonal to the routing problem. Every agent receives the full natural language query. Even adapting it for metadata inputs, MoA provides no mechanism to avoid calling expensive models.

---

## What Routing Features Are Used in Practice

A survey of routing papers from 2023–2026 (arXiv:2603.04445 survey; IRT-Router arXiv:2506.01048; ParetoBandit arXiv:2604.00136; kNN routing arXiv:2505.12601) reveals a consistent feature palette:

**Pre-generation features (routing before calling any model):**
- Dense text embeddings (BERT [CLS], sentence-transformers, OpenAI embeddings) — used by RouteLLM SW, matrix factorization, BaRP, ParetoBandit, kNN routers
- Query length in tokens — heuristic proxy for complexity
- Perplexity of the query under a small model — higher perplexity → harder query
- Binary attribute vectors: "does the query require multi-step reasoning?", "is this a factual lookup?"
- IRT-derived difficulty and discrimination parameters (IRT-Router, arXiv:2506.01048)

**Post-generation features (after at least one model call):**
- Answer consistency / self-consistency score of the weak model
- DistilBERT reliability score over (query, answer) pair — FrugalGPT
- Token log-probabilities from the first model

**Key observation:** Every pre-generation feature is a transformation of natural language text. No published routing paper uses structural file metadata (extension, MIME type, byte size, magic-byte signature, entropy) as a primary routing signal.

---

## Metadata-Only Routing: The Gap

For Curator, the input space at routing time is:

| Signal | Available? |
|---|---|
| File extension (.pdf, .py, .heic, .pkl) | Always |
| MIME type (from libmagic / file(1)) | Usually |
| File size in bytes | Always |
| Creation / modification timestamps | Always |
| File entropy (byte-distribution Shannon entropy) | Computable cheaply |
| Readable text content | Not guaranteed |
| Natural language query from user | Not available at triage |

The gap is categorical: all existing routing systems require text embeddings as the primary routing signal. File metadata has no representation in their feature spaces.

**Closest adjacent literature:**
- A 2021 digital forensics paper (DOI: 10.1016/j.fsidi.2021.301027) applied ML to smartphone file metadata (extension, MIME, size, directory depth, timestamp patterns) for file triage. This is the file-metadata-as-feature-space approach, but it routes files to categories, not to LLMs.
- US Patent 12,287,761 (2025) describes ML-based file classification using metadata cascaded through sequential classifiers with confidence thresholds — structurally identical to FrugalGPT's cascade but for files, not text queries.

**The gap precisely stated:** No paper has combined (a) the metadata-only feature space of file triage systems with (b) the LLM model selection problem of the routing literature.

---

## Online Routing Updates: Adding New Specialists

### Thread 1: Bandit-based routing with online updates

**ParetoBandit** (arXiv:2604.00136, Taberner-Miller, 2026) is the most directly relevant work. It formulates LLM routing as a contextual UCB bandit over a dynamic arm set. Key mechanisms for I2:

- **`add_arm()` / `delete_arm()` at runtime** — models can be added without restarting or retraining the router
- **Forced exploration burn-in** — 20 unconditional pulls are routed to the new arm regardless of UCB score, bootstrapping its posterior
- **UCB integration** — after burn-in, the new arm competes via standard UCB exploration bonus; the system reaches meaningful adoption (~3% share) in approximately 142 steps
- **Context encoding** — `all-MiniLM-L6-v2` embeddings projected to 25 PCA components

**Limitation for Curator:** ParetoBandit requires text embeddings as context. Substituting a file metadata vector would require re-fitting the PCA transform, but the bandit mechanics (add_arm, forced exploration, UCB) transfer directly.

### Thread 2: Time-increasing bandits

**TI-UCB** (arXiv:2403.07213, "Which LLM to Play?") handles non-stationary model performance (models being fine-tuned over time). Relevant if Curator's specialists are updated.

### Thread 3: Bandit feedback routing

**BaRP** (arXiv:2510.07429) formulates routing as a multi-objective contextual bandit with bandit feedback (partial supervision). Its decision head requires retraining when a new model is added — the limitation that ParetoBandit's `add_arm()` mechanic solves.

Key insight: in production, Curator will only observe the quality of the model it actually routed to — not how the alternative would have performed. This partial-feedback structure is exactly the contextual bandit problem.

---

## Cold-Start Specialist Profiling

When Curator adds a new specialist (e.g., a vision model for `.heic` files), it needs to characterize the model's capability boundary with minimal evaluations.

### Approach 1: Benchmark-based capability profiling

**InferenceDynamics** (arXiv:2505.16303) profiles models across capability dimensions by running a structured set of benchmark queries per dimension. For Curator's analog: run a structured set of test files per category (20 scanned PDFs, 20 code files, 20 photos) and measure per-category accuracy.

**STEM** (arXiv:2508.12096) — "Efficient Relative Capability Evaluation of LLMs through Structured Transition Samples" — evaluates models using "transition samples" near capability boundaries rather than random benchmarks, achieving efficient relative ordering with far fewer evaluations.

### Approach 2: IRT-based difficulty modeling

**IRT-Router** (arXiv:2506.01048, ACL 2025) applies Item Response Theory to model both query difficulty and model ability jointly. New models can be profiled by running them on a small set of "anchor items" with known difficulty parameters, then fitting their ability parameter. For Curator: a set of anchor files with known classification difficulty could profile a new specialist's ability in O(50–100) evaluations rather than thousands.

### Approach 3: kNN locality for few-shot transfer

The kNN routing paper (arXiv:2505.12601) demonstrates that routing quality depends on "locality properties in embedding space." For cold-starting a new specialist: if the new specialist's performance on 50 test files can be embedded and compared to a database of known-performance files, the kNN regressor can extrapolate performance to unseen files without full benchmark coverage.

### Approach 4: Zero-shot model selection via Model Label Learning

**MLL** (arXiv:2408.11449) — "Enabling Small Models for Zero-Shot Selection and Reuse through Model Label Learning" — builds a semantic DAG of model capabilities. Annotate each specialist with capability labels (e.g., "handles binary images", "handles code with comments", "handles scanned PDFs with tables") and use label matching as a zero-shot cold-start routing prior before any evaluation data is collected.

---

## A Proposed Metadata Routing Architecture for Curator

### Feature Engineering Layer

Compute a fixed-dimensional feature vector from file metadata:

- **Extension embedding** — one-hot or learned embedding over ~200 common extensions, grouped into coarse categories (image, document, code, archive, binary, audio, video)
- **MIME type group** — categorical (16 top-level MIME categories: image/*, text/*, application/*, etc.)
- **File size** — log-normalized (files span bytes to gigabytes)
- **Byte entropy** — Shannon entropy of a 4KB random sample (cheap, highly informative: text ~4.5 bits/byte, encrypted/compressed ~7.9 bits/byte, images ~6–7 bits/byte)
- **Magic byte signature match** — binary flags for common signatures (PDF header, PNG magic, ZIP PK header, ELF, etc.) — 20–30 flags
- **Timestamp features** — hour of day, day of week of creation (proxy for workflow context)

This yields a ~60-dimensional feature vector, entirely computable without reading file content.

### Router Model

A **lightweight gradient-boosted classifier** (XGBoost or a 2-layer MLP) trained on labeled examples of (metadata vector, correct_model_choice). AutoML literature consistently shows gradient boosting dominates on tabular feature vectors.

Training label: binary "small model sufficient" vs "large model required". Labels collected by running both models on a calibration set of ~500 files and recording which model produced acceptable output.

### Online Learning Layer (ParetoBandit-style)

Wrap the base classifier in a contextual bandit layer (LinUCB or Thompson Sampling over the metadata feature vector):
- **I2 (new specialists):** ParetoBandit's `add_arm()` mechanic — 20-pull forced exploration burn-in, then UCB integration
- **Distribution shift:** Geometric forgetting on sufficient statistics to adapt as the user's file corpus evolves
- **Partial feedback:** Only observe quality of the chosen model; bandit correctly handles this counterfactual gap

### Cold-Start Profiling Protocol (I3)

When onboarding a new specialist model:
1. **Label-based prior:** Annotate the specialist with capability labels. Initialize the bandit arm with a strong prior favoring those MIME/extension categories.
2. **Anchor evaluation:** Run the specialist on 50 anchor files spanning the extension/entropy space. Fit an IRT ability parameter per category.
3. **kNN extrapolation:** Use the 50-point evaluation results to seed a kNN regressor in metadata-feature space.
4. **Live burn-in:** 20 unconditional forced-exploration pulls (ParetoBandit protocol).

This pipeline reaches reliable routing in ~70–150 files evaluated per new specialist.

### Summary Architecture

```
File arrives
    │
    ▼
Feature extraction (extension + MIME + size + entropy + magic bytes) → ~60-dim vector
    │
    ▼
Contextual bandit router (LinUCB over metadata features)
    │               │
    ▼               ▼
Small model      Large model    [← hot-swappable specialist pool via add_arm()]
    │               │
    ▼               ▼
Quality signal fed back → bandit update (partial feedback handled natively)
```

---

## Key References

1. Ong et al. (2024). RouteLLM: Learning to Route LLMs with Preference Data. arXiv:2406.18665. GitHub: https://github.com/lm-sys/RouteLLM.
2. Chen, Zaharia, Zou (2023/2024). FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance. arXiv:2305.05176. TMLR 2024.
3. Wang et al. (2024). Mixture-of-Agents Enhances Large Language Model Capabilities. arXiv:2406.04692.
4. Moslem, Kelleher (2026). Dynamic Model Routing and Cascading for Efficient LLM Inference: A Survey. arXiv:2603.04445.
5. Taberner-Miller (2026). ParetoBandit: Budget-Paced Adaptive Routing for Non-Stationary LLM Serving. arXiv:2604.00136. GitHub: https://github.com/ParetoBandit/ParetoBandit.
6. Liu et al. (2025). Rethinking Predictive Modeling for LLM Routing: When Simple kNN Beats Complex Learned Routers. arXiv:2505.12601.
7. (2025). IRT-Router: Effective and Interpretable Multi-LLM Routing via Item Response Theory. arXiv:2506.01048. ACL 2025.
8. (2025). BaRP: Learning to Route LLMs from Bandit Feedback. arXiv:2510.07429.
9. (2024). TI-UCB: Which LLM to Play? Convergence-Aware Online Model Selection with Time-Increasing Bandits. arXiv:2403.07213.
10. (2025). InferenceDynamics: Efficient Routing Across LLMs through Structured Capability and Knowledge Profiling. arXiv:2505.16303.
11. (2024). MLL: Enabling Small Models for Zero-Shot Selection and Reuse through Model Label Learning. arXiv:2408.11449.
12. (2025). STEM: Efficient Relative Capability Evaluation of LLMs through Structured Transition Samples. arXiv:2508.12096.
13. (2021). Machine learning based approach to analyze file meta data for smart phone file triage. Forensic Science International: Digital Investigation. DOI: 10.1016/j.fsidi.2021.301027.
14. US Patent 12,287,761 (2025). Systems and methods for machine learning-based classification of digital computer files using file metadata.
15. (2025). Routing, Cascades, and User Choice for LLMs. arXiv:2602.09902.
16. (2024). Large Language Model Cascades with Mixture of Thoughts Representations for Cost-efficient Reasoning. arXiv:2310.03094.
17. LMSYS Blog (2024). RouteLLM: An Open-Source Framework for Cost-Effective LLM Routing. https://www.lmsys.org/blog/2024-07-01-routellm/
18. (2026). Bayesian Online Model Selection. arXiv:2602.17958.
