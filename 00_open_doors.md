# 00 — Open Doors: Genuinely New Research Directions
_Date: 2026-05-31_
_Scope: Cross-file analysis of files 01–38. Only directions NOT already covered by another file in the set._

---

## Theme A — Filesystem as Cognitive & Mental Health Signal
**Source: File 28 (entirely new direction — not touched by any other file)**

This is the most significant open door in the entire set. No other file in 01–38 addresses it.

**A1. FLRS decay shape as a digital biomarker**
File 28 proposes that the shape of an individual's FLRS decay trajectory (rapid decay → burnout signal; abnormal flattening → disengagement) constitutes a novel class of behavioral biomarker. No analog exists in smartphone or wearable digital phenotyping literature (StudentLife, BiAffect, Mohr group). FLRS is unique because it captures *cognitive engagement with information*, not physical behavior.

**A2. Antilibrary growth rate as anxiety/overload indicator**
The rate at which a user accumulates files they intend to read but never access (antilibrary) is proposed as an anxiety/cognitive overload signal. No prior PIM or digital phenotyping work uses this construct.

**A3. Cluster entropy over time as cognitive scatter indicator**
Increasing spread of file creation across many clusters simultaneously (high cluster entropy) as a signal of cognitive scatter / attention fragmentation. Not proposed or measured in any existing study.

**A4. Work/personal file ratio shift as burnout signal**
A shift toward near-zero personal file creation (everything is work) as a passive burnout indicator. No smartphone behavioral study captures this dimension.

**A5. Failed Spotlight retrieval rate as organizational stress indicator**
Elevated Spotlight search rate with no subsequent file access (failed retrieval → re-search pattern) as a signal of organizational stress. This is a new signal not used in any digital phenotyping pipeline.

**A6. Inferential privacy problem: semantic content**
File system behavioral inference is qualitatively more sensitive than step-count or location data because the content of files reveals intellectual interests, medical concerns, relationship status, etc. The inferential privacy problem — where passive behavioral signals enable re-identification or sensitive attribute disclosure through semantic file content — is not addressed in existing privacy literature for PIM or digital phenotyping. Requires its own threat model.

**A7. Within-person longitudinal design requirement**
Cognitive state correlations in file behavior are plausible only with within-person longitudinal designs (each user as their own baseline). Between-person designs are confounded by personality and organizational style. This methodological constraint is novel in the digital phenotyping space, where most studies use between-person designs.

**Why novel:** Dinneen & Julien (JASIST 2020) — the most relevant PIM + wellbeing paper — explicitly confirms no published study uses file behavior as an inferential signal for cognitive state.

---

## Theme B — Life-Stage Detection from Latent File Patterns
**Source: File 17**

**B1. Structural life-stage transitions from cluster emergence/collapse**
Detecting transitions (student→professional, single→partnered, pre-parenthood→parenthood) from the emergence of new dominant clusters and collapse of old ones — without requiring explicit user input. All prior life-stage detection work uses social media text (Twitter/Instagram posting patterns) or wearable sensor data. No system infers life-stage from latent file organizational patterns. Not addressed in files 04, 22, 26, or 28.

---

## Theme C — CP Feedback Loop: IPS Correction, Endogenous Choice, Learning
**Source: Files 31, 32 — specific open problems not resolved by other files**

**C1. Adaptive exploration schedule for IPS-corrected CP**
The ε_t schedule (probability of showing a folder outside the CP prediction set, to gather propensity data) creates a bandit-within-CP problem: what ε_t minimizes long-run regret while maintaining coverage guarantees? No existing bandit or CP paper addresses this compound objective. Files 16, 21, 23 discuss monoculture and OCP-Unlock+ but not the ε_t optimization problem specifically.

**C2. Coverage-learning tradeoff with formal regret decomposition**
File 32 identifies the precise open problem: online CP that simultaneously maintains marginal coverage AND provides consistent label distribution estimation from endogenously censored user selections f_t ∈ C_t(x_t). A formal regret decomposition coupling both goals — coverage regret and estimation regret — does not exist in any CP, PLL, or online learning paper.

**C3. Private signal contamination**
Users select folders based on private signals z_t (memory of which project a file belongs to, deadline context) not present in model features x_t. This introduces a structural assumption problem: the label-generating process is f_t ~ P(F | F ∈ C_t, x_t, z_t), where z_t is latent. No existing partial label learning or CP paper models this structure.

**C4. Doubly robust CP bound under propensity misspecification**
If propensity scores π_t(f|x) are misspecified (wrong model for which folders appear in prediction sets), the IPS correction breaks. A doubly robust CP bound — valid when either the CP model or the propensity model is correctly specified — does not exist. Standard doubly robust estimation (Robins & Rotnitzky 1995) does not carry over to the CP coverage guarantee setting.

**C5. Adaptive adversary with shifting behavior distributions**
If user behavior shifts over time (new job → new filing habits), the propensity model π_t becomes stale. Combining CP guarantees with adaptive feedback distributions under an adversarial user model is not addressed by OCP-Unlock+ (which fixes the feedback structure) or ACI (which requires full feedback).

---

## Theme D — Cluster Quality Evaluation for Non-Text Files
**Source: File 24**

**D1. No cluster quality metric for metadata-only or mixed-type clusters**
ProxAnn (ACL 2025) requires readable text content per cluster member. For clusters containing images, binaries, audio files, or metadata-only records, no existing method evaluates cluster quality. HDBSCAN's internal metrics (DBCV, relative validity) measure density-based cohesion but not semantic coherence. This is an open evaluation problem with no solution in the literature.

---

## Theme E — FLRS Empirical Validation & FSRS Calibration
**Source: Files 25, 34**

**E1. No direct measurement of file-location memory decay**
S₀ ≈ 1.0 day is theoretically motivated (Ebbinghaus + binding fragility penalty) but has no direct empirical support in a naturalistic filesystem context. A simple within-subjects user study — measure when participants forget where they saved a test file under controlled conditions — would provide direct S₀ calibration data. File 25 flags this as a potential CHI/IUI contribution. No other file covers this specific study design.

**E2. FSRS burst access pattern overfitting**
Files accessed in concentrated bursts (dissertation writing week, tax preparation) accumulate high stability S values during the burst, then show rapid decay afterward. FSRS assumes access events are quasi-independent; burst clustering violates this assumption. No adaptation of FSRS for temporally clustered access patterns is published.

**E3. Grade imbalance in file access patterns**
Most file accesses are direct navigation (Grade 3/Good in FSRS). The lapse parameters (w₁₁–w₁₂ in FSRS-4.5) governing what happens when memory fails are fitted on flashcard data where lapses are common. In file access, lapses are rare and indirect (search-assisted). Transferring FSRS parameters without refitting on file-specific data may yield poorly calibrated lapse behavior.

**E4. Access sparsity for personalized FSRS fitting**
Most personal files never accumulate 10+ access events. Personalized FSRS parameter fitting requires statistically sufficient event histories. For the majority of files, only population-level priors apply. The precision-coverage tradeoff — how many files can be personally fitted vs. must use priors — is not quantified in the literature.

---

## Theme F — GhostUMAP2 × Markov Biography Cross-Validation
**Source: File 37**

**F1. GhostUMAP2 instability persistence across refits**
The cross-validation hypothesis (GhostUMAP2 instability ↔ Markov stationary entropy) assumes both signals measure stable underlying ambiguity. But a file that is stable at epoch t (low GhostUMAP2 displacement) may become unstable at t+1 as the corpus grows and embedding space shifts. Whether GhostUMAP2 instability is a stable property of a file or a snapshot of corpus state is not addressed by the GhostUMAP2 paper.

**F2. Markov chain order assumption**
First-order Markov assumption for cluster assignment history may be too simple if assignments show autocorrelation of order 2+. Higher-order HMMs are infeasible with 2 refits/year (insufficient state transition data). A principled method for selecting Markov order under sparse temporal observations is not available.

**F3. Causal direction in the instability-entropy correlation**
High Markov stationary entropy could result from (a) genuine boundary ambiguity or (b) file content evolution (the file itself changed). Controlling for content evolution requires monitoring file modification timestamps and re-embedding on modification — an architectural complication not designed into the current FLRS + biography pipeline.

**F4. Corpus size threshold for statistical validation**
The cross-validation hypothesis requires n ≥ 150 files with ≥ 5 epoch histories. New Curator deployments cannot satisfy this threshold for months or years. The cold-start validity problem for Markov-based ambiguity detection is not addressed.

---

## Theme G — H(location|intent) Implementation Gaps
**Source: File 29**

**G1. Session boundary sensitivity**
The 15-minute inactivity threshold for session segmentation (standard in PIM literature) is arbitrary. H(location|intent) values are sensitive to this choice because different thresholds yield different co-occurrence patterns and hence different conditional entropy estimates. A sensitivity analysis quantifying how much H(L|I) varies across plausible threshold values (5–30 minutes) has not been done.

**G2. File aliasing and symlink resolution**
Hard links and symlinks mean a single file can appear at multiple filesystem locations. Without canonical path resolution before location discretization, H(L|I) is artificially inflated. No PIM metric paper addresses symlink normalization.

**G3. CP bound tightness in practice**
The formal connection E[|C(x)|] ≥ 2^{H(L|I)} (from Angelopoulos et al. NeurIPS 2024 arXiv:2405.02140) may be very loose in practice because CP prediction sets are constructed to be conservative. Empirical measurement of the gap between E[|C(x)|] and 2^{H(L|I)} on real file collections is needed to determine whether the CP bound is informative.

**G4. Intent proxy construct validity**
Spotlight query logs capture only 30–50% of file accesses (direct navigation, recent files list, and bookmark access leave no Spotlight trace). If intent proxies are systematically missing for certain access patterns, H(L|I) estimates are biased toward query-based access, which is qualitatively different from habitual direct navigation. This is a major construct validity risk not addressed elsewhere in the 38 files.

---

## Theme H — Energy Measurement Gaps
**Source: File 03**

**H1. No published Ollama-specific energy benchmarks**
zeus-apple-silicon and IOReport are the available tools, but no published benchmark measures Ollama inference energy on Apple Silicon across model sizes and prompt types. Per-decision energy for file classification tasks is entirely unmeasured.

**H2. IOReport accuracy unvalidated**
Apple has not published calibration data for IOReport power estimates. Whether IOReport accurately reflects actual chip-level power dissipation (vs. model-based estimates from CPU/GPU activity counters) is unknown. This is a fundamental measurement validity problem for energy-aware inference research on Apple Silicon.

**H3. Thermal throttling non-linearity**
Sustained inference (>5–15 minutes) triggers thermal throttling on MacBook hardware, reducing clock speeds and hence throughput. Energy per decision therefore increases non-linearly with sustained operation. Existing energy benchmarks measure short bursts; no study characterizes the throttling regime relevant to background file organization tasks.

**H4. Apple Neural Engine underutilization**
ANE power consumption is near-zero for Ollama/llama.cpp workloads because neither framework exposes ANE compute pathways for arbitrary models. The potential energy efficiency of ANE (designed for matrix operations at lower power) is entirely unexploited. Quantifying the ANE efficiency gap requires CoreML-based inference pathways not currently available for open-weight models.

---

## Theme I — Model Routing Without Natural Language Input
**Source: File 02**

**I1. Metadata-only routing**
All published model routing systems (RouteLLM, Mixture-of-Agents, FrugalGPT) require a natural language query as input. Routing decisions for file classification based only on file extension, MIME type, file size, and creation timestamp — without text content — is an unaddressed problem. No routing literature covers this input modality.

**I2. Adaptive routing without retraining when new specialist added**
When a new domain-specialist model is added to the routing pool (e.g., a new code model), existing routing systems require full retraining of the router. An online routing update that integrates a new specialist using only a small calibration set is not published.

**I3. Orchestrator cold-start profiling**
Characterizing a new specialist model's capability boundary (on which file types is it better than the generalist?) requires benchmarking. No method for efficient few-shot profiling of a specialist model across file domains is published.

---

## Theme J — Calibration in Agentic Pipelines
**Source: File 01**

**J1. Compounding calibration error in multi-step agents**
Calibration research targets single-step prediction. In multi-step agentic pipelines (classify file type → infer intent → select folder → confirm with user), calibration errors compound across steps. A pipeline-level calibration framework — ensuring the terminal decision is well-calibrated even when intermediate steps are not — does not exist.

**J2. Principled threshold selection for confidence tiers**
The partition of confidence scores into HIGH/MEDIUM/LOW tiers (which determines whether Curator acts autonomously, prompts for confirmation, or escalates) has no principled selection method in the literature. Threshold selection is treated as a hyperparameter; no decision-theoretic framework for setting thresholds based on task-specific costs (wrong auto-move vs. unnecessary confirmation) is published for PIM contexts.

---

## Theme K — RAG Novel Angles for Personal Files
**Source: File 14**

**K1. Spotlight metadata as first-class retrieval signal**
kMDItemLastUsedDate, kMDItemWhereFroms, kMDItemKeywords, kMDItemContentTypeTree are rich retrieval signals that encode recency, provenance, and semantic type. No published RAG pipeline uses macOS Spotlight metadata as a scored retrieval dimension alongside embedding similarity and BM25. The information-theoretic value of these signals for file retrieval has not been measured.

**K2. HDBSCAN soft-membership as query expansion prior**
Using HDBSCAN soft membership probabilities (probabilities_ array) as a continuous prior for query expansion — boosting retrieval scores for files in the same soft cluster as known-relevant documents — is not published. Standard query expansion uses discrete category labels.

**K3. Embedding inversion as local privacy risk**
Recovering approximate original document content from stored embeddings (embedding inversion attacks, e.g., Morris et al. 2023) is a local privacy risk when embeddings are stored in a local ChromaDB instance that could be exfiltrated. File 36 covers xattr provenance privacy; no file covers embedding inversion specifically.

---

## Theme L — Personal Knowledge Graph Evaluation & Schema Drift
**Source: File 09**

**L1. No benchmark for PKG quality from file collections**
File 33 covers PIM navigation benchmarks (FileNav); it does not cover knowledge graph quality benchmarks. No published benchmark evaluates the quality of a PKG constructed from a personal file collection: entity extraction precision/recall, relation accuracy, schema coherence. The PKG Ecosystem Survey (Skjæveland et al. 2023) explicitly calls this out as missing.

**L2. Schema drift without bound**
As an LLM extracts new entity types from heterogeneous files over time, the KG schema expands. Without schema anchoring or entity type consolidation, the schema grows without bound and relations between distantly named equivalent concepts are never merged. No incremental KG construction paper addresses this for personal, longitudinally-grown KGs.

---

## Theme M — Concept Drift & File Biography Tracking
**Source: File 06**

**M1. Per-item drift attribution**
ADWIN and STUDD detect that drift has occurred globally across the file corpus. They do not identify which specific files (or file clusters) are causing the drift signal. Per-item drift attribution — flagging the specific files responsible for triggering a corpus-level change — is unsolved in stream learning literature.

**M2. Temporal granularity for file biography**
Recording a file's cluster assignment per-access vs. per-day vs. per-week yields qualitatively different biography sequences. The Markov chain entropy analysis in file 37 assumes per-epoch (6-month) granularity; finer granularity enables richer models but requires more storage and computation. No principled method for selecting biography granularity based on expected drift rate and storage constraints is published.

**M3. Review fatigue threshold**
How many files should be flagged for user review per session without causing review fatigue and habituation? PIM diary studies suggest users tolerate 3–5 organizational decisions per session before ignoring prompts. A principled model of review fatigue — relating flag rate, false positive rate, and user habituation — is not published for file organization contexts.

---

## Theme N — On-Device Fine-tuning Without Explicit Labels
**Source: File 07**

**N1. Implicit quality metric for on-device fine-tuning**
Correction rate (fraction of Curator's suggestions that users override) is a natural implicit quality metric for fine-tuning triggers. But correction rate is not a calibrated metric: some users correct frequently out of habit; others rarely correct even when unsatisfied. Formalizing correction rate as a quality signal with user-level normalization and distinguishing "preference correction" from "error correction" is not published.

**N2. Cold-start fine-tuning performance paradox**
Fine-tuning is most needed early in deployment (when the model is least personalized) but performs worst at that point (fewest training examples). The cold-start performance paradox for continual on-device learning — and principled methods to mitigate it (population priors, transfer from similar users) — is not addressed in MLX/LoRA literature.

---

## Theme O — AMEDIA Extension: Filing Behavior and Metacognition
**Source: File 26**

**O1. Photo-taking impairment analog for file saving**
The photo-taking impairment effect (Henkel 2014, Psychological Science) — where photographing objects reduces internal memory encoding of their details — has no analog in file organization research. Does the act of carefully organizing a file (giving it a descriptive name, placing it in a specific folder) reduce internal encoding of its location, because the user delegates location memory to the system? This is a testable cognitive psychology hypothesis with direct implications for FLRS personalization. Not addressed in any of the 38 files.

**O2. Metacognitive model of externalization decisions**
AMEDIA commentators (Hutmacher et al. 2024 discussion) call for a metacognitive model of when people decide to externalize information to digital artifacts vs. retain internally. For file organization, the equivalent question is: what signals trigger the decision to save (and where to save) vs. discard? No computational model of this metacognitive process exists. Such a model would directly inform FLRS weighting (files saved with high metacognitive deliberation should have higher initial S₀).

---

## Theme P — FileNav Benchmark: Foreign User Condition
**Source: File 33**

**P1. Legibility testing via foreign user condition**
The "foreign user condition (Class C)" — where participants navigate a persona's file system without prior knowledge of the organizing logic — tests whether a file organization is legible to someone other than its creator. This tests a qualitatively different property than usability for the organizing user (Classes A and B). No existing PIM benchmark includes this condition. It directly measures whether Curator produces *communicable* organization rather than idiosyncratic personal systems.

---

## Theme Q — macOS Provenance Edge Cases
**Source: File 36**

**Q1. xattr stripping by AirDrop and iCloud Drive**
kMDItemWhereFroms and com.apple.quarantine are stripped when files are transferred via AirDrop or synced through iCloud Drive. Files arriving via these mechanisms have no provenance metadata. No published method handles provenance imputation for files arriving without xattr — using filename heuristics, content fingerprinting, or transfer timestamp correlation as fallback signals.

**Q2. QuarantineEventsV2 unbounded growth**
There is no TTL or automatic pruning for QuarantineEventsV2.db entries. On a long-lived macOS installation, the database can contain millions of entries from years of downloads. Curator's reliance on this database requires either (a) a local index of relevant rowIDs or (b) handling the performance degradation of large database queries. App Store review may additionally restrict access to this database without entitlements. No published macOS application documents a solution to this tripartite problem.

---

## Cross-Cutting Priority Assessment

**Highest research priority (publication-ready with focused effort):**
- A1–A7: Filesystem as mental health signal — novel enough for CHI/UbiComp main track
- C2: Coverage-learning tradeoff — NeurIPS/ICML-level theoretical contribution
- E1: S₀ user study — small CHI/IUI short paper; directly calibrates a core FLRS parameter
- P1: Foreign user condition — novel benchmark design element for IUI 2027 FileNav paper

**Medium priority (strengthen existing Curator contributions):**
- D1: Non-text cluster quality evaluation — practical need for Curator evaluation methodology
- G4: Intent proxy construct validity — must be addressed for H(location|intent) validity claims
- K1: Spotlight metadata as RAG signal — differentiating engineering contribution

**Lower priority (engineering or foundational work for later phases):**
- H1–H4: Energy measurement — validates energy claims but not core to IUI 2027 submission
- Q1–Q2: Provenance edge cases — implementation detail, not research contribution
- L2: KG schema drift — relevant only after PKG component is built

---
_Generated by cross-file analysis of 01–38. Each door verified absent from all other files before inclusion._
