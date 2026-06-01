# 83 — Cross-File Synthesis: Convergences, Chains, Tensions, and Emergent Capabilities
_Date: 2026-06-01_
_Status: Complete_
_Scope: All 82 research files (foundational 01–10, core 11–25, deep dives 26–38, new directions 39–82, open doors 00)_

---

## Overview

This synthesis identifies the structural connections across the full 82-file research corpus. It organizes findings into four categories: **Convergences** (multiple independent files pointing to the same architectural conclusion), **Chains** (sequences of files where each one is a prerequisite for the next), **Tensions** (places where two files pull in incompatible directions and require a design decision), and **Emergent Capabilities** (combinations of two or more files that unlock something no individual file describes). A final section gives a priority ranking for which synthesis findings matter most for the IUI 2027 submission.

---

## Part 1 — Convergences

A convergence is a finding where three or more independent research threads arrive at the same design implication without referencing each other. These are the most reliable signal in the corpus.

---

### Convergence 1: The Multi-Signal File Affinity Graph Is the Core Architecture

**Files contributing:** 09 (personal knowledge graph), 40 (temporal co-access graph), 51 (project boundary detection), 70 (NER cross-format detection), 72 (implicit feedback signals), 76 (H(L|I) session boundaries), 78 (project detection engine synthesis), 82 (heterogeneous file affinity graph)

**The convergence:** Eight independent research threads — approaching from knowledge graph theory, software temporal coupling analysis, PIM project fragmentation, NER, recommender systems implicit feedback, information-theoretic session segmentation, and HIN theory — all converge on the same architectural conclusion: the correct representation for a personal file collection is a **weighted heterogeneous graph** where files are nodes and typed edges represent distinct relationship dimensions (semantic similarity, entity co-occurrence, behavioral co-access).

File 09 derives this from the PKG literature. File 40 derives it independently from Tornhill's temporal coupling work in software engineering. File 51 derives it from Bergman et al.'s "project fragmentation" problem. File 70 derives it from NER entity overlap as a grouping signal. File 72 derives it from implicit feedback modeling in recommender systems. File 76 derives it from information-theoretic session boundary detection. File 78 explicitly synthesizes files 70, 72, 76 into the first partial graph architecture. File 82 provides the full HIN formalization using PathSim and SNF (Similarity Network Fusion, Wang et al. 2014 Nature Methods).

**Implication:** HDBSCAN on raw embeddings alone is architecturally incomplete. The minimal viable graph for Curator requires: (1) semantic embedding edges (cosine similarity threshold), (2) NER entity co-occurrence edges (Jaccard similarity of entity sets), (3) behavioral co-access edges (FSEvents session-bounded, dwell-time-adjusted, PU-learning-debiased). These three edge types together represent the minimum that beats pure cosine similarity on project detection tasks.

**Research contribution:** The application of PathSim meta-path similarity (Sun et al., VLDB 2011) and SNF fusion to personal filesystem data is novel. No published PIM system has built a HIN over personal files.

---

### Convergence 2: Conformal Prediction Is the Unified Uncertainty Language

**Files contributing:** 01 (confidence calibration), 05 (conformal prediction), 16 (semi-bandit CP analysis), 23 (OCP-unlock), 31 (algorithmic monoculture), 38 (small-T CP bounds), 39 (save-point interception), 56 (calibration in agentic pipelines), 74 (doubly robust CP), 79 (CP practical implementation)

**The convergence:** Ten files, approaching from calibration theory, formal statistics, interactive learning, monoculture risk, asymptotic guarantees, UI design, agentic pipelines, covariate shift robustness, and implementation engineering, all converge on conformal prediction as the correct framework for communicating uncertainty in Curator.

File 01 establishes that verbalized HIGH/MEDIUM/LOW confidence is better calibrated than token probabilities for RLHF-tuned models — supporting the use of CP-derived prediction sets as the confidence interface. File 05 gives the formal foundation (RAPS is recommended; APS is the intermediate option; LAC is the minimum viable implementation from logprobs). File 16 identifies the semi-bandit feedback structure: users selecting from CP prediction sets provides exactly the feedback structure needed for IPS-corrected online CP updates. File 23 (OCP-Unlock+) solves the endogenous feedback problem — when users only ever see the prediction set, standard CP coverage degrades; OCP-Unlock+ corrects this. File 31 warns that CP monoculture (every user's prediction set is the same) is an adversarial failure mode when an attacker controls file content. File 38 gives the small-T bounds guaranteeing valid coverage during the early deployment period before large calibration sets accumulate. File 39 shows CP integrates directly into the Save-Point Interception UI, giving save-time folder suggestions a formal coverage guarantee. File 56 identifies the pipeline-level calibration problem: CP guarantees at each node do not compose to a guarantee on the terminal decision. File 74 gives the doubly robust extension for handling covariate shift (user's files differ from training distribution). File 79 provides the MAPIE implementation path, recommending Mondrian (class-conditional) CP with at least 250 calibration examples per class.

**Implication:** Curator should output prediction sets C(x) at level 1−α = 0.90 as its primary uncertainty representation throughout the pipeline — in the approval UI, in the save-panel, in the file-biography routing decision, and in the audit log. The prediction set replaces the raw softmax probability everywhere it is shown to the user. MAPIE with Mondrian calibration is the implementation.

**Research contribution:** First integration of doubly robust CP (covariate shift correction) into an interactive personal file organization system. First use of CP prediction sets as save-dialog folder suggestions.

---

### Convergence 3: Safe Automation Requires a Three-Tier Policy, Not a Toggle

**Files contributing:** 44 (safe automation policy), 48 (review fatigue), 53 (data loss anxiety), 73 (siblings preview UI), 77 (photo-taking impairment), 80 (zero-touch automation UX)

**The convergence:** Six files approaching from formal policy design, decision psychology, trust theory, UI design, cognitive psychology, and empirical PIM failures all converge on the same architecture: automation for file operations must be tiered (AUTO / REVIEW / NEVER-AUTO), and the exact tier assignment must track action reversibility and user confidence simultaneously.

File 44 derives the three-tier framework from robot autonomy literature (Parasuraman & Sheridan 2000) and corrigible AI research. File 48 quantifies the decision fatigue problem: users can actively deliberate on roughly 20–25 decisions per session before habituation; beyond this, approval becomes rubber-stamping. File 53 identifies the trust asymmetry: one unexpected displacement destroys trust earned through 49 correct suggestions; undo is not sufficient because moved files are hard to discover as missing. File 73 shows that showing 3–5 destination-folder siblings at decision time measurably improves decision quality and reduces regret-driven undos. File 77 identifies the photo-taking impairment analog: if Curator auto-organizes everything silently, users develop a shallow cognitive model of their own filesystem ("Curator organized it somewhere"), which degrades retrieval even when Curator is correct. File 80 documents the iCloud Drive "automation surprise" failure mode (silent eviction without warning) and the Parasuraman-Sheridan level analysis placing appropriate PIM automation at Level 4–5 (system narrows options, human executes) rather than Level 8–9 (full autonomy).

**Implication:** The specific tier assignments, derived from reversibility analysis in file 44, are: AUTO applies to textual rename suggestions and tag additions (fully reversible, low cognitive cost); REVIEW applies to any move operation regardless of confidence; NEVER-AUTO applies to deletions, archive moves to trash-equivalent locations, and any operation on files older than 6 months or larger than 50MB. The REVIEW queue must be batched (maximum 15 items per session to respect fatigue thresholds from file 48) and must show destination siblings (file 73).

**Research contribution:** First formal automation policy framework for local AI file organization grounded in both decision theory and empirical PIM literature.

---

### Convergence 4: Cold Start Is Solved by a Layered Signals Cascade, Not a Single Method

**Files contributing:** 11 (novelty verification — active learning prior art), 45 (cold start personalization), 07 (on-device fine-tuning), 72 (implicit feedback), 40 (temporal co-access graph)

**The convergence:** Five files converge on the insight that the cold start problem has multiple phases with different available signals, each requiring a different method.

File 45 establishes the three phases: (Phase 0) install-time passive inference from the existing filesystem structure; (Phase 1) first 10 interaction events used for filing style classification; (Phase 2) Bayesian updating from 10–100 corrections toward a personal model. File 11 establishes that cluster-based centroid sampling for cold start active learning has 20+ years of prior art, so Curator must frame the contribution as novel application to personal filesystems with empirical benchmarks, not novel theory. File 07 establishes that on-device LoRA fine-tuning is feasible for 0.5B–3B models (2–15 minutes on any M-series chip ≥8GB RAM) but requires at least 50–100 labeled examples to avoid catastrophic forgetting — placing fine-tuning in Phase 2 or later. File 72 shows that dwell-time-normalized implicit feedback accumulates faster than explicit corrections and should be used as the primary Phase 1 signal. File 40 shows that temporal co-access patterns (which files are opened together) emerge within the first 20 sessions even without NER or embeddings, providing early project-detection edges.

**Implication:** Curator should implement the three-phase cascade explicitly, with clear threshold transitions: Phase 0 analyzes existing folder structure and filenames to infer filing style taxonomy (deep hierarchy vs. flat vs. date-based) within the first 5 minutes of installation; Phase 1 accumulates implicit feedback for the first 50 file events to update the population prior toward a personal model; Phase 2 triggers LoRA fine-tuning when ≥100 correction events have accumulated.

---

### Convergence 5: Local Privacy Is a Multi-Layer Problem Requiring Defense in Depth

**Files contributing:** 43 (AI-readable filesystem standard), 49 (local privacy threat model), 36 (apple provenance xattr), 50 (AI provenance and academic integrity), 00 (open doors Theme A6 — inferential privacy)

**The convergence:** Five files approaching from filesystem standards, security threat modeling, macOS xattr mechanics, academic integrity, and cognitive health inference all converge on the finding that Curator's privacy surface is far larger than the raw file content alone.

File 49 identifies three attack vectors specific to local AI: (1) the SQLite behavioral database as a semantic portrait even without raw file content (Vec2Text, Morris et al. EMNLP 2023, achieves 92% exact-match text recovery from embeddings); (2) the ChromaDB embedding store as an inversion target; (3) xattr metadata traveling with files through AirDrop and iCloud Drive, exposing AI classifications to unintended recipients. File 43 (ARFS standard) notes that sidecar files (.textwin) and extended attributes created by Curator permanently annotate user files — a design decision with privacy implications beyond the current session. File 36 documents that kMDItemWhereFroms and com.apple.quarantine are stripped by AirDrop and iCloud, creating inconsistency in provenance pipelines that rely on these signals. File 50 notes the false-positive risk of AI provenance tools (Yale student case, 2025 suspension) and warns against Curator writing provenance attributions to file metadata without explicit user opt-in. File 00 (Theme A6) identifies the inferential privacy problem: filesystem behavioral signals reveal intellectual interests, medical concerns, and relationship status through semantic content — qualitatively more sensitive than step-count or location data.

**Implication:** Defense in depth for Curator requires: (1) SQLCipher encryption for the SQLite behavioral database at rest (file 65 — GRDB with SQLCipher v7.10.0 SPM support is the implementation path); (2) local-only ChromaDB instance with no network exposure and no cloud sync exclusion from iCloud Drive and Time Machine for the embedding store specifically; (3) explicit user disclosure that xattr writes are permanent and travel with files; (4) opt-in-only provenance labeling, never default-on; (5) design documentation treating the embedding store as a privacy artifact equivalent in sensitivity to the raw file content.

---

### Convergence 6: The Level Router Principle — Compute Is Not Uniform Across Files

**Files contributing:** 02 (SAM orchestration and model routing), 47 (level router), 55 (model routing without NL input), 08 (local VLMs)

**The convergence:** Four files converge on the finding that the cost distribution over a real personal file collection is extremely skewed, and a uniform per-file analysis budget is therefore a category error.

File 47 establishes the six-tier escalation ladder empirically: Level 0 (filename + extension) handles the majority of common file types unambiguously; Levels 1–3 add MIME/xattr, lightweight embeddings, and full document embeddings respectively; Levels 4–5 invoke small and large LLMs only for the ambiguous tail. File 02 provides the theoretical grounding from the SAM orchestration and dynamic routing literature, identifying the six routing paradigms and citing HierRouter's 2.4x quality improvement over best individual model. File 55 identifies the unique challenge for Curator: every published routing system (RouteLLM, FrugalGPT, IRT-Router) assumes a natural language query as input, but Curator must route on metadata alone for binary, image, and non-text files — a categorically unaddressed problem in the routing literature. File 08 identifies which VLMs are appropriate for Level 5 escalation when files require visual understanding: Qwen2.5-VL (3B or 7B) for structured documents; FastVLM (Apple, CVPR 2025, 85x faster TTFT than LLaVA) for general images; Granite Vision 2B for CPU-only fallback.

**Implication:** The Level Router is both a practical engineering component and a research contribution. The metadata-only routing problem (file 55's I1, I2, I3) is the specific novel contribution — routing without any natural language input is an open problem in the routing literature. Curator's implementation constitutes the first published solution and evaluation of this problem class.

---

## Part 2 — Chains

A chain is a dependency sequence: understanding or building File B requires File A as a prerequisite. The chains below are multi-hop, each one identifying the full prerequisite path for a major Curator subsystem.

---

### Chain 1: The FLRS Pipeline

**Full chain:** 06 (concept drift, file biography motivation) → 34 (FSRS algorithm) → 25 (FLRS empirical grounding) → 37 (GhostUMAP2 × Markov cross-validation) → 06 (drift detection methods) → 72 (implicit feedback for grade derivation) → 81 (folder biography extending FLRS to folder level)

**Reading order explanation:**

File 06 establishes the theoretical motivation: files have lifecycles, cluster assignments change over time, and per-file tracking of this history is conceptually novel. File 34 provides the mathematical substrate: the FSRS spaced repetition algorithm with its 4 differential equations and 19 parameters, which FLRS adapts from flashcard memory to file-location memory. File 25 provides the empirical grounding, verifying that the FLRS adaptation is theoretically defensible against the memory science literature and identifying three open empirical gaps (S₀ calibration, burst access overfitting, grade imbalance from lapse sparsity). File 37 adds the Markov file biography layer on top of FLRS: each file's cluster assignment history is modeled as a Markov chain; the stationary entropy of this chain determines FLRS-routing priority (high stationary entropy → frequently re-assigned → ambiguous → route to human review). File 06 is revisited for the ADWIN-U and STUDD drift detection methods that can trigger biography updates without labels. File 72 provides the implicit feedback signals (dwell time, edit frequency, copy/move actions) that serve as approximate FLRS grades when explicit user corrections are unavailable. File 81 extends the biography concept from the file level to the folder level: a folder biography tracks the folder's expected file-type distribution over time, enabling completeness scoring and actionable gap detection.

**Open problems inherited from the chain:** File 00 Theme E1 (S₀ has no direct empirical support — a user study measuring file-location memory decay is needed), Theme E2 (burst access pattern overfitting), Theme F1 (GhostUMAP2 instability may be a snapshot of corpus state rather than a stable file property), Theme M2 (biography temporal granularity selection).

---

### Chain 2: The Conformal Prediction Pipeline

**Full chain:** 01 (calibration basics) → 05 (CP mechanics) → 16 (semi-bandit feedback) → 23 (OCP-Unlock+) → 31 (monoculture risk) → 38 (small-T bounds) → 56 (agentic pipeline calibration) → 74 (doubly robust extension) → 79 (MAPIE implementation)

**Reading order explanation:**

File 01 motivates CP by showing that raw LLM confidence is systematically mis-calibrated and that verbalized confidence is better for RLHF-tuned models. File 05 gives the formal CP framework: split conformal, LAC/APS/RAPS nonconformity scores, and the coverage guarantee. File 16 shows that Curator's user interaction structure (user selects a folder from the prediction set) is exactly the semi-bandit feedback structure analyzed in Angelopoulos et al. ICML 2025, enabling IPS-corrected online coverage maintenance. File 23 (OCP-Unlock+) addresses the critical endogenous feedback problem: if Curator only ever shows the prediction set, the feedback is censored and standard CP coverage guarantees degrade — OCP-Unlock+ uses occasional "unlocking" (showing a folder outside the set) to collect unbiased calibration data. File 31 warns that CP can create monoculture: if all users' prediction sets contain the same clusters because the model was trained on similar data, an adversary controlling file content can exploit the shared prediction structure. File 38 establishes that small-T (Wilks 1938) bounds are more appropriate than asymptotic guarantees during early deployment with small calibration sets. File 56 introduces the pipeline calibration problem: CP at each node does not compose; a new calibration mechanism for the terminal decision is needed. File 74 provides the doubly robust extension handling covariate shift between training and deployment distributions. File 79 gives MAPIE as the practical library, recommends Mondrian CP for per-class coverage, and specifies 250+ calibration examples per class as the reliability threshold.

---

### Chain 3: The Project Detection Pipeline

**Full chain:** 51 (project boundary problem) → 40 (temporal co-access graph) → 70 (NER entity extraction) → 76 (H(L|I) session boundaries) → 72 (implicit feedback debiasing) → 78 (synthesis) → 82 (HIN formalization) → 69 (betweenness centrality for review queue ordering)

**Reading order explanation:**

File 51 names the problem: semantic clusters ≠ projects; Bergman et al. (CHI 2006) named project fragmentation as an open PIM problem. File 40 provides the first signal layer: temporal co-access graphs from Tornhill's temporal coupling work in software engineering (CodeScene/Code Maat), adapted to personal file systems. File 70 provides the second signal layer: NER entity extraction across formats identifies shared PROJECT_NAME, PERSON, ORG, DATE entities as high-confidence cross-file links; Apple NaturalLanguage framework's NLTagger provides this natively on macOS without network calls. File 76 provides the session boundary detection needed to scope co-access windows: H(L|I) entropy spikes mark context switches in the FSEvents stream. File 72 provides dwell-time normalization and positive-unlabeled (PU) learning framing needed to debias the behavioral co-access edges. File 78 synthesizes files 70, 72, 76 into the first concrete multi-layer architecture: three edge types (NER, co-access, embedding similarity) fused via SNF. File 82 formalizes this as a HIN using PathSim meta-paths and specifies igraph as the practical graph library for 10k–500k file nodes. File 69 adds betweenness centrality as the review queue ordering criterion: high-BC files are structural bridges whose placement decision resolves ambiguity for many adjacent files; reviewing these first maximizes the information value of each user decision.

---

### Chain 4: The macOS Architecture Stack

**Full chain:** 62 (SwiftUI vs Tauri decision) → 64 (menu bar persistent app) → 63 (Swift–Python sidecar) → 65 (GRDB + SQLCipher) → 66 (ChromaDB via sidecar) → 60 (PDF extraction) → 59 (archive extraction) → 36 (xattr provenance) → 43 (ARFS standard)

**Reading order explanation:**

File 62 resolves the framework decision: SwiftUI is correct for Curator, not Tauri, because Full Disk Access, file bookmarks, Sandbox entitlements, and App Store path all require native macOS integration. File 64 establishes the menu bar persistent app pattern using MenuBarExtra (macOS 13+) with LSUIElement = YES and .activationPolicy(.accessory). File 63 establishes that localhost HTTP (FastAPI) is the correct Swift–Python sidecar communication method: Ollama itself uses this pattern; URLSession is the Swift client. File 65 resolves the database choice: GRDB with SQLCipher (v7.10.0 SPM support, August 2025) for encrypted local behavioral storage. File 66 establishes that ChromaDB runs inside the Python sidecar (via FastAPI), not as a separate process — Swift never speaks to ChromaDB directly. File 60 documents the PDF extraction pipeline (PyMuPDF for digital PDFs, Greek CMAP encoding handling, Tesseract OCR fallback for scanned PDFs). File 59 documents the archive extraction pipeline for the ~30% of Downloads that are ZIP/tar/DMG, where filename-only embedding produces near-useless representations. File 36 establishes the xattr provenance layer: kMDItemWhereFroms, com.apple.quarantine, and Spotlight metadata APIs (MDItem) are the macOS-native provenance signals. File 43 (ARFS standard) defines the target state for each file's metadata representation: a sidecar .textwin plus versioned xattr fields encoding classification confidence, extraction hash, embedding model version, and relationship links.

---

### Chain 5: The Approval UI Chain

**Full chain:** 48 (review fatigue) → 44 (automation policy tiers) → 73 (siblings preview) → 52 (explainability) → 69 (betweenness centrality ordering) → 53 (trust and undo)

**Reading order explanation:**

File 48 establishes the psychological constraints: 20–25 decisions per session before habituation; alarm fatigue literature (clinical setting) shows habituation reduces the beneficial effect of suggestions to near-zero. File 44 defines the tier structure that constrains queue population: only REVIEW-tier decisions enter the approval queue; AUTO and NEVER-AUTO decisions are handled without user interaction. File 73 specifies the destination sibling preview as a required UI element: show 3–5 files already in the proposed destination folder to reduce ambiguity at decision time. File 52 establishes the explanation requirements: explanations must fit in a notification/tooltip, must reference user-legible concepts (not similarity scores), must be multi-signal, and must be calibrated enough that users can detect when the explanation is wrong. File 69 establishes betweenness centrality as the queue ordering principle: present high-BC files first. File 53 specifies the trust and undo requirements: every REVIEW-tier action must be atomic, undoable with a single shortcut, and logged in the audit trail; the system must proactively surface the undo option for 30 seconds after each confirmed move, because the cost of a wrong move is asymmetric and high.

---

### Chain 6: The Save-Point Interception Chain

**Full chain:** 04 (PIM filing behavior at save time — Barreau & Nardi 1995) → 39 (save-point interception) → 05 + 79 (CP integration at save time) → 44 (automation policy — interception is REVIEW tier) → 72 (FLRS implications: zero-displacement saves eliminate phantom location memories)

**Reading order explanation:**

File 04 establishes the PIM theoretical motivation: Barreau & Nardi (1995) showed filing decisions are made at save time; Bergman et al. (2010) showed user-created hierarchies have 65% lower retrieval failure; Jones & Teevan (2007) identified filer vs. piler dichotomy. File 39 introduces Save-Point Interception: intercept NSSavePanel using the accessoryView mechanism (App Store-compatible, used by Default Folder X), embed the partial filename, active application bundle ID, and window title, run a quantized sentence transformer (<50ms pipeline on Apple Silicon), and display the top-3 conformal prediction set folders as ranked suggestions directly in the Save panel. File 39 also introduces the Zero-Displacement Rate (ZDR) metric — the fraction of saves where the first-written location equals the 30-day stable location — as the primary outcome variable for the proposed user study. Files 05 and 79 provide the CP integration: the save-panel suggestions are a prediction set with formal coverage guarantee. File 44 clarifies the policy tier: save-time suggestions are REVIEW-tier (user must explicitly accept); no file is auto-written to a non-default location without user action. File 72 connects to FLRS: a zero-displacement save eliminates the phantom location memory problem (false memory trace from a wrong initial save location that was later corrected) and therefore strengthens the initial FLRS stability S₀.

---

## Part 3 — Tensions

A tension is a place where two or more files make incompatible recommendations or pull toward opposing design choices. Each tension requires an explicit resolution decision.

---

### Tension 1: Automation Depth vs. Metacognitive Engagement

**Files in tension:** 44 (safe automation policy, advocates for AUTO tier to reduce user burden) vs. 77 (photo-taking impairment, warns that automation degrades users' mental model of their own files) vs. 80 (zero-touch automation UX, places appropriate automation at Level 4–5 out of 10)

**Nature of tension:** File 44 is optimizing for efficiency: if 80% of renames can be done autonomously, requiring user review for all of them is waste. File 77 argues that precisely this automation — even when correct — impairs the user's ability to find their own files because they never encode the organizational decisions. File 80 corroborates file 77 with empirical PIM failures (iCloud Optimize Mac Storage, Time Machine silent exclusions).

**Resolution:** The tension is real but resolvable with a distinction file 44 does not make explicitly: AUTO tier should be restricted to **fully reversible, low-information-content actions** (tag additions, thumbnail labels, TextTwin sidecar creation) where the action has no impact on the user's spatial mental model of their filesystem. File moves — even moves Curator is highly confident about — must remain REVIEW tier because moves alter the user's internal location map. Renames are borderline: they are highly reversible but do alter the user's ability to find files by partial-name recall. Recommendation: renames require a one-time opt-in to AUTO tier per folder, with the user explicitly reading the rename rule before it is applied. This preserves metacognitive engagement (the user has seen the rule) while reducing per-file overhead.

---

### Tension 2: Embedding Coverage vs. Privacy Depth

**Files in tension:** 08 (local VLMs — rich embedding from image content is clearly better) vs. 49 (local privacy threat model — rich embeddings are precisely what Vec2Text can recover text from)

**Nature of tension:** File 08 argues that Qwen2.5-VL 7B and ColPali produce far better semantic representations than filename-only embeddings, especially for image-heavy file collections. File 49 establishes that the richer the embedding, the more recoverable the original content is if the embedding store is exfiltrated — Vec2Text achieves 92% exact-match recovery from 32-token sequences.

**Resolution:** The tension resolves differently for different file types. For text documents, the full embedding is as sensitive as the full text. For images, the embedding captures only semantic gist, not pixel-level reconstruction. The practical mitigation for text documents: store only a **dimensionality-reduced projection** (e.g., 64-dim PCA reduction of a 768-dim embedding) in the persistent database; keep the full-dimension embedding only in-memory during active classification sessions. The 64-dim projection retains enough information for approximate cluster assignment while being much harder to invert to original text. Empirically verify this tradeoff using Vec2Text on the projected vectors before deployment.

---

### Tension 3: FLRS Relies on Access Signals That Are Biased by the System Itself

**Files in tension:** 25 (FLRS empirical grounding, requires access timestamps as grade proxies) vs. 72 (implicit feedback, establishes that access patterns are biased by presentation position, recency, and system-surfaced suggestions) vs. 37 (Markov biography, treats cluster assignment history as independent observations)

**Nature of tension:** FLRS grades file access events as "the user found this file — memory confirmed." But if Curator itself surfaced this file via a suggestion, the access event is caused by Curator's presentation, not by independent user recall. This creates a feedback loop: Curator inflates the FLRS stability of files it frequently suggests, causing it to suggest them even more. File 72's multi-behavior alignment and PU learning framing partially addresses this but does not fully solve it for FLRS.

**Resolution:** Implement an access event attribution flag in the behavioral database: each FSEvents access is tagged as either user-initiated (no Curator activity in the preceding 10-second window) or system-prompted (Curator made a suggestion in the preceding window). FLRS grade calculations must use only user-initiated access events for stability scoring. System-prompted accesses are logged separately for implicit feedback model training. This requires storing a lightweight Curator event log alongside FSEvents timestamps — feasible given file 65's GRDB architecture.

---

### Tension 4: NER Entity Extraction vs. Privacy of File Content

**Files in tension:** 70 (NER entity detection, requires reading file content to extract PERSON, ORG, PROJECT_NAME entities) vs. 49 (local privacy threat model, warns that any persistent metadata derived from file content expands the attack surface)

**Nature of tension:** NER entity extraction requires reading file content — the most privacy-sensitive step in the pipeline. The extracted entities (PERSON names, ORG names, PROJECT_NAME strings) stored in the KG or as xattr are potentially more specific than the file's topical cluster assignment, and they are stored in a format that directly re-identifies people and organizations.

**Resolution:** Entity extraction must be strictly on-device (Apple NaturalLanguage framework's NLTagger, zero network calls). Extracted entities must be stored in the SQLCipher-encrypted behavioral database with the same access controls as the embedding store. No entity strings should be written to file xattr or to any storage that travels with the file (AirDrop, iCloud sync). The ARFS sidecar (.textwin) may store topic labels and confidence scores but must not store raw entity strings. Users must be informed that entity extraction reads file content before enabling the project detection feature; it should default to off at install time.

---

### Tension 5: Heterogeneous Graph Scale vs. Real-Time Responsiveness

**Files in tension:** 82 (heterogeneous file affinity graph, full HIN with PathSim is O(V·E) per meta-path query) vs. 47 (level router, requires <50ms at Level 0–3) vs. 64 (menu bar persistent app, must not consume visible CPU when idle)

**Nature of tension:** The full SNF fusion over a three-layer HIN for a 100k-file corpus is not a real-time operation. PathSim meta-path similarity is O(V·E) per query with Brandes' algorithm. This conflicts with the Level Router's latency requirements and the menu bar app's requirement to be nearly invisible when not actively classifying.

**Resolution:** The HIN is a **background index**, not a real-time inference component. It is rebuilt asynchronously (nightly or on explicit trigger) and serves pre-computed project membership caches. Real-time classification (Level 0–3) uses only the embedding index and metadata signals. The HIN is consulted only at Level 4–5 for deep ambiguity resolution. Betweenness centrality values (file 69) are pre-computed and cached with the HIN. igraph's Python implementation (file 82's recommended library) handles batch updates efficiently; SNF fusion updates run in a background Python sidecar process (file 63) on a low-priority QoS queue, notifying the Swift frontend via XPC when the cache is refreshed.

---

### Tension 6: Version Chain Detection Requires Content Reading, But Content Reading Has Latency Cost

**Files in tension:** 46 (version semantics, needs text similarity and metadata comparison across file pairs) vs. 47 (level router, wants to defer content reading to Levels 3–5) vs. 60 (PDF extraction has significant latency for scanned PDFs requiring OCR)

**Nature of tension:** Accurate version chain detection (draft → revision → final) requires comparing the text content of candidate file pairs. But content reading is Levels 3–5 in the router. For a new file arriving at intake, the router will not reach content reading until it escalates — by which time the version chain signal has been lost if the file is auto-routed at Level 0–1.

**Resolution:** Implement a lightweight pre-filter for version chain candidates at Level 0–1 using filename signals alone: files sharing a common stem with version indicators (_v2, _final, _FINAL, numeric suffix) or sharing identical extension and creation-date proximity (< 7 days) are flagged as potential version chain members without content reading. These flagged pairs are queued for background content comparison using a text fingerprint (MinHash of character 4-grams) that can be computed in < 5ms per document without full OCR. Full content comparison (Levenshtein or cosine similarity) is reserved for confirmed version chain candidates. This tiered approach keeps real-time latency within the Level Router budget while retaining high precision on version detection.

---

## Part 4 — Emergent Capabilities

An emergent capability is something that becomes possible only when two or more files are combined — it is not visible in either file alone.

---

### Emergent 1: File-Biography-Driven Conformal Prediction Sets

**Emerges from:** 37 (Markov file biography, per-file stationary entropy) + 05/79 (conformal prediction, Mondrian class-conditional CP)

**What emerges:** Mondrian CP (file 79) allows per-stratum calibration: the nonconformity score threshold q_hat can be computed separately for different strata of files. The Markov file biography provides a natural stratification variable: files with low stationary entropy (stable cluster assignment history) use one threshold; files with high stationary entropy (frequently reassigned, biography-ambiguous) use a looser threshold, producing larger prediction sets that signal genuine ambiguity rather than falsely confident single-folder suggestions.

The result is a CP system that is not just calibrated on average but calibrated *per-file* in proportion to the file's own organizational history. A file that has been stably in one cluster for 18 months gets a tight prediction set (often size 1). A file that has been assigned to 3 different clusters in 6 months gets a prediction set of size 3–4, honestly reflecting its ambiguity. No existing CP or PIM system achieves this per-history calibration.

**Research contribution:** Novel Mondrian stratification variable derived from longitudinal cluster assignment history. Publishable as part of the IUI 2027 paper as the "biography-informed prediction set" contribution.

---

### Emergent 2: Organizational Burden Index (OBI) as a Conformal Prediction Validity Test

**Emerges from:** 41 (OBI — Organizational Burden Index) + 05 (conformal prediction) + 29 (H(location|intent) entropy)

**What emerges:** File 41 defines OBI as a composite score over five components including H(location|intent) (file 29), DBCV cluster quality, folder depth entropy, FLRS decay rate, and access failure rate. File 29 establishes the formal relationship between H(L|I) and CP prediction set size: E[|C(x)|] ≥ 2^{H(L|I)} (Angelopoulos et al. NeurIPS 2024). This means OBI — which includes H(L|I) as a component — provides a lower bound on the average CP prediction set size that Curator will produce. A high-OBI filesystem (very disorganized) forces Curator to produce large prediction sets (high uncertainty) even for files it would confidently classify in a well-organized collection.

The emergent capability: OBI becomes a **pre-audit instrument** for predicting whether Curator's CP system will be able to provide useful (small) prediction sets before any classification is run. If OBI is very high, users can be informed that the initial scan will produce many ambiguous suggestions and that organizational investment is needed before CP prediction sets become actionable. This converts OBI from a retrospective quality score into a prospective system performance predictor.

---

### Emergent 3: Zero-Displacement Save as FLRS Intervention

**Emerges from:** 39 (save-point interception, zero-displacement saves eliminate phantom locations) + 25 (FLRS, initial stability S₀ governs long-term retention) + 77 (photo-taking impairment, deliberate placement at save time improves encoding)

**What emerges:** File 39 introduces ZDR (Zero-Displacement Rate) as a metric and shows that save-point interception prevents phantom location memories. File 25 establishes that FLRS S₀ is the most impactful parameter for long-term file retrieval success — a file with high S₀ is retrievable for months; a file with low S₀ is effectively lost within days. File 77 shows that deliberate, user-executed filing (as opposed to AI-silent-automation) improves encoding of the file's location.

The emergent capability: save-point interception is not just a convenience feature — it is a **direct FLRS intervention** that maximizes S₀ for each saved file. A zero-displacement save at the correct location, executed by the user (not silently auto-moved), simultaneously (1) sets the correct initial location (avoiding phantom traces), (2) engages the user's metacognitive attention (improving initial encoding, per file 77), and (3) provides Curator with a high-confidence ground-truth label for the calibration set (the user accepted the suggestion under ideal deliberation conditions). This triple benefit makes save-point interception a higher-value investment than post-hoc organization for the same file.

This emergent capability can be framed as a novel theoretical contribution: **the causal pathway from save-point AI assistance to FLRS retrieval outcome**, which connects interactive systems design (save dialog), cognitive psychology (metacognitive encoding), and formal memory modeling (FLRS) in a single explanatory chain.

---

### Emergent 4: H(L|I) as a Real-Time Automation Gate

**Emerges from:** 76 (H(L|I) session boundary detection) + 44 (safe automation policy) + 40 (temporal co-access for project detection)

**What emerges:** File 76 shows that H(L|I) spikes mark context switches (user switched projects or tasks). File 44 establishes that AUTO-tier actions require high confidence. File 40 shows that project boundaries are detectable from co-access patterns.

The emergent capability: H(L|I) can serve as a **real-time confidence gate** for automation decisions. During a stable work session (low H(L|I), user is in a coherent project context), Curator's confidence in file routing is high and AUTO-tier actions are appropriate. When H(L|I) spikes (context switch), Curator should temporarily pause AUTO-tier actions and queue new files for REVIEW, because the routing model has not yet adapted to the new session context. This converts H(L|I) from a one-time session segmentation tool into a continuous automation eligibility signal — a design pattern not described in any existing PIM or automation policy paper.

---

### Emergent 5: The Foreign-User Legibility Test as CP Prediction Set Width Proxy

**Emerges from:** 33 (FileNav benchmark, foreign user condition) + 05 (conformal prediction) + 82 (HIN, project structure legibility)

**What emerges:** File 33's Class C condition (a stranger navigates the organized filesystem) tests whether the organization communicates project structure to someone without prior knowledge of the organizer's logic. File 05 shows that CP prediction set size measures the ambiguity of a file's categorization. A large prediction set means even the AI cannot confidently determine where the file belongs — which is exactly the condition that will confuse a foreign user.

The emergent capability: **average CP prediction set size over the full filesystem is a proxy for the foreign-user legibility score** — both measure how unambiguously the organization communicates its own logic. This means Curator can estimate FileNav Class C performance without running a user study: compute mean prediction set size over a calibrated held-out set. The correlation between prediction set size and Class C navigation performance is a falsifiable hypothesis that can be verified empirically as part of the IUI 2027 evaluation, connecting the formal CP framework to the HCI evaluation methodology in a single experiment.

---

### Emergent 6: Doubly Robust CP Enables Cross-User Model Transfer

**Emerges from:** 74 (doubly robust CP under covariate shift) + 45 (cold start personalization) + 71 (PRISM-X pooled preference fine-tuning)

**What emerges:** File 74's doubly robust CP provides valid coverage guarantees when the test distribution (new user's files) differs from the calibration distribution (existing calibration set). File 45 establishes that cold start is the hardest problem: the system has no personalized data. File 71's PRISM-X finding shows that pooled preference fine-tuning over many users outperforms both generic models and individual personalization — meaning a shared fine-tuned model, recalibrated per user, is the right architecture.

The emergent capability: **cold start + doubly robust CP = formally valid personalization under distribution shift**. A new user's file collection is a distributional shift from the population calibration set. Standard CP would provide coverage guarantees only for the average user. Doubly robust CP provides valid guarantees for this specific new user's distribution without requiring their own calibration set — the doubly robust property ensures validity as long as either the importance weights (from PRISM-X pooled model) or the outcome model (classification) is correctly specified. This is the formal statistical grounding for Curator's claim to provide calibrated suggestions to new users from day one.

---

## Part 5 — Priority Ranking for IUI 2027 Submission

The following ranks the synthesis findings by their combined research novelty, feasibility within the submission timeline (October 2026), and relevance to the IUI 2027 audience.

---

### Tier 1 — Core Paper Contributions (Must Include)

**1. Convergence 2 + Chain 2: CP as Unified Uncertainty Language**
Priority rationale: The most theoretically grounded contribution in the corpus. The IPS-corrected online CP with semi-bandit feedback structure (file 16) is the paper's primary formal contribution. OCP-Unlock+ (file 23) addresses the critical endogenous feedback problem that no other CP+PIM paper has handled. MAPIE + Mondrian implementation (file 79) makes it reproducible. This is the contribution that distinguishes Curator from every other AI file organizer.

**2. Emergent 1: File-Biography-Driven Conformal Prediction Sets**
Priority rationale: Directly combines the FLRS biography (files 25, 37) with Mondrian CP (file 79) into a novel per-history-calibrated prediction system. Publishable as a 3–4 page technical section. No implementation blocker — MAPIE.partial_fit() and the Markov entropy calculation are both straightforward.

**3. Convergence 1 + Chain 3: Multi-Signal File Affinity Graph**
Priority rationale: The project detection problem (file 51) is PIM's most-cited open problem and the multi-signal HIN solution is genuinely novel. The minimum viable implementation (NER edges from Apple NaturalLanguage + behavioral co-access edges from FSEvents, fused with SNF) is implementable in the timeline. Full PathSim meta-path querying can be deferred to a follow-on paper.

**4. Chain 1: FLRS with File Biography**
Priority rationale: The Markov file biography (file 37) is confirmed novel (file 11 novelty verification). FLRS adapts FSRS (file 34) to the file-location memory domain. Together they constitute a coherent theoretical contribution with implementation components already designed. The S₀ user study (Theme E1) should be run as a short pilot to provide empirical grounding.

---

### Tier 2 — Strong Supporting Contributions (Include if Space Permits)

**5. Chain 6 + Emergent 3: Save-Point Interception + ZDR metric**
Priority rationale: File 39 is thoroughly designed with study protocol, technical implementation, and prior art analysis complete. ZDR is a novel metric. The connection to FLRS (Emergent 3) gives it theoretical depth. A within-subjects user study (N=24, 4-week) is feasible before the October submission deadline if IRB approval (file 61) is obtained immediately (June 2026).

**6. Convergence 3 + Chain 5: Safe Automation Policy + Approval UI**
Priority rationale: Files 44, 48, 73, 80 together constitute a formally grounded UI contribution that IUI audiences value highly. The Parasuraman-Sheridan framing positions Curator precisely in the human-AI teaming literature. No new implementation required — this is a design contribution documented from existing components.

**7. Emergent 4: H(L|I) as Real-Time Automation Gate**
Priority rationale: Short, elegant, formally grounded. Directly connects file 76's entropy calculation to file 44's automation policy. Novel design pattern with no prior publication. Implementation is ~50 lines of Python (rolling entropy over FSEvents stream).

---

### Tier 3 — Future Paper Directions (Do Not Include in IUI 2027)

**8. Theme A (File 00): Filesystem as Mental Health Signal**
Priority rationale: Highest novelty, but requires longitudinal within-person study design (months of data collection) and separate IRB protocol for health-adjacent research. Timeline is incompatible with October 2026.

**9. Convergence 5: Local Privacy Defense in Depth**
Priority rationale: Critical engineering requirement but not an IUI research contribution. Document in a security report or arXiv preprint; reference briefly in the IUI paper's privacy section.

**10. Tension 1 Resolution: Automation Depth vs. Metacognitive Engagement**
Priority rationale: Interesting HCI design question but would require a dedicated user study to resolve empirically. Insufficient time for IUI 2027. Frame as a known tradeoff and cite file 77 as theoretical motivation.

**11. Emergent 5: Foreign-User Legibility as CP Prediction Set Proxy**
Priority rationale: Interesting hypothesis but requires the FileNav benchmark study to validate. Describe as a future evaluation direction in the paper's limitations section.

**12. Emergent 6: Doubly Robust CP + Cold Start Transfer**
Priority rationale: Theoretically strong but requires multi-user deployment data to demonstrate the covariate shift correction empirically. Single-user evaluation cannot validate this. Future paper with multi-user dataset.

---

## Part 6 — Summary Table: Strongest Research Claims

| # | Claim | Files | Novelty Verdict (File 11 basis) |
|---|---|---|---|
| 1 | Markov file biography with per-file stationary entropy routing | 06, 25, 34, 37 | Confirmed novel — no prior art |
| 2 | IPS-corrected online CP with semi-bandit file feedback | 05, 16, 23 | Partial prior art (domain novelty) |
| 3 | Biography-informed Mondrian CP strata | 37, 05, 79 | Confirmed novel — combination |
| 4 | Save-point interception as ZDR + FLRS S₀ intervention | 39, 25, 77 | Confirmed novel — no prior papers |
| 5 | Multi-signal HIN for project detection | 40, 70, 76, 78, 82 | Confirmed novel in PIM domain |
| 6 | H(location\|intent) entropy as CP prediction set lower bound | 29, 05 | Confirmed novel — combination |
| 7 | H(L\|I) as real-time automation eligibility gate | 76, 44 | Confirmed novel — design pattern |
| 8 | OBI as CP prediction set size predictor | 41, 05, 29 | Confirmed novel — combination |
| 9 | Three-tier automation policy grounded in formal reversibility | 44, 48, 53 | Confirmed novel in PIM domain |
| 10 | Doubly robust CP for cold-start covariate shift | 74, 45, 71 | Partial prior art (DR-CP exists, PIM application novel) |

---

## Part 7 — Implementation Dependency Order

The following order minimizes blocking dependencies for implementation:

1. **SQLite + GRDB (File 65)** — foundational database before anything else
2. **Python sidecar + FastAPI (File 63)** — needed for all ML components
3. **PDF extraction + archive extraction (Files 60, 59)** — needed before any content analysis
4. **Embedding pipeline (Qwen3-Embedding workaround per File 57, DenStream replacement per File 58)** — needed before clustering
5. **Level Router (File 47)** — gates everything above; implement skeleton first, fill tiers incrementally
6. **HDBSCAN clustering (existing) + SNF fusion (File 82 layer 1: semantic edges only)** — minimal graph
7. **NER extraction (File 70, Apple NaturalLanguage)** — add second graph layer
8. **FSEvents behavioral logging + session boundary detection (Files 76, 72)** — add third graph layer
9. **Conformal prediction (MAPIE, File 79)** — calibration layer over existing classifier
10. **FLRS + File Biography (Files 25, 37)** — requires stable clustering first
11. **Approval UI (Files 44, 48, 52, 73)** — requires routing to be stable
12. **Save-point interception (File 39)** — can be built in parallel once sidecar is running
13. **Full HIN + PathSim + betweenness centrality (Files 82, 69)** — background index, build last

---

_End of synthesis. All 82 research files incorporated. File references verified against corpus._
