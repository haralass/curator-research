# R9 — Evaluation Metrics for Curator

**Document status:** Research complete  
**Date:** 2026-06-08  
**Scope:** How to measure whether Curator actually helped the user — beyond clustering accuracy, capturing real PIM value for neurodivergent users  

---

## Table of Contents

1. [Metrics Taxonomy](#1-metrics-taxonomy)
2. [Prior Art Review — PIM Evaluation Literature](#2-prior-art-review)
3. [New Metrics for Curator](#3-new-metrics-for-curator)
4. [Neurodivergent-Specific Evaluation](#4-neurodivergent-specific-evaluation)
5. [Evaluation Protocol — 4-Phase Plan](#5-evaluation-protocol)
6. [Benchmarks — Applicability Assessment](#6-benchmarks)
7. [LLM-as-Judge](#7-llm-as-judge)
8. [Papers and Citations](#8-papers-and-citations)
9. [Design Decisions](#9-design-decisions)
10. [Open Questions](#10-open-questions)

---

## 1. Metrics Taxonomy

### 1.1 Automated Metrics (no users required)

These run on Curator's output without any user involvement. They measure cluster quality, name quality, and duplicate detection completeness.

| Metric | Formula | Range | Better |
|---|---|---|---|
| DBCV | Density-Based Clustering Validation (Moulavi 2014) | [-1, +1] | Higher |
| Cohesion Ratio | `mean(intra_cluster_sim) / mean(global_sim)` | [0, ∞) | Higher |
| Name-Embedding Cosine | `cosine(embed(folder_name), mean(embeds(files)))` | [-1, +1] | Higher |
| Duplicate Recall | `detected_duplicates / total_actual_duplicates` | [0, 1] | Higher |
| DCR | `files_processed / group_decisions_required` | [1, ∞) | Higher |

**DBCV** (Density-Based Clustering Validation):
```
DBCV = (1/|C|) Σ_{c ∈ C} [  (min_density_separation(c) - max_density_within(c))  
                              / max(min_density_separation(c), max_density_within(c))  ]
```
Range: [-1, +1]. Specifically valid for HDBSCAN and other density-based clusterers. Silhouette score assumes convex clusters and is WRONG for Curator's non-convex context groups. Implementation: `github.com/FelSiq/DBCV` (fully compatible with original MATLAB implementation, supports parallel computation).

**Name-Embedding Cosine**:
```python
name_score(folder) = cosine(
    embed(folder.name),
    mean([embed(f.name + " " + f.summary) for f in folder.files])
)
```
Measures whether the LLM-generated folder name captures the semantic center of its contents. Target: > 0.65 (empirical threshold; calibrate on developer's own filesystem).

**Cohesion Ratio** (from arXiv:2511.19350):
```
CR = mean_{c ∈ C}[ mean_{i,j ∈ c, i≠j}(sim(i,j)) ] / mean_{i,j ∈ all, i≠j}(sim(i,j))
```
Intrinsic, reference-free, information-theoretically motivated. Correlates strongly with NMI and homogeneity. Designed specifically for short text embeddings — relevant because filenames are short texts.

---

### 1.2 Behavioral / Implicit Metrics (deployed app, passive logging)

These require the app to log user actions and filesystem events without interrupting the user.

| Metric | Signal source | Formula | Target |
|---|---|---|---|
| Undo Rate | Action log | `undo_actions / total_commits` | < 5% |
| Re-organization Rate | FSEvents | `manual_moves_of_curated_files / total_curated_files` | < 10% |
| Search-After-Organization Rate | NSWorkspace / Spotlight | `spotlight_searches_resolving_to_curated_files / total_curated_files` | < 15% |
| Re-find Time | FSEvents open events | `time(folder_open → file_open)` in seconds | Baseline: ~15 s |
| Abandonment Rate | Session log | `incomplete_review_sessions / total_review_sessions` | < 20% |

**Re-organization Rate** uses FSEvents (`FSEventStreamCreate`) to detect when a file Curator placed is subsequently moved by the user to a different location within 30 days. High rate = Curator made wrong placement decisions.

**Search-After-Organization Rate** approximates whether Curator's placement was predictable. If a user organized by Curator still uses Spotlight to find a file instead of navigating, the placement was unpredictable. Measured via `NSWorkspace.shared` notifications and correlation of Spotlight result paths against Curator's placement log.

**Re-find Time** is the gold standard from Bergman (2010). Approximate from FSEvents: measure elapsed time between `kFSEventStreamEventFlagItemIsDir` (folder open) and `kFSEventStreamEventFlagItemIsFile` (file open) within the same folder subtree.

---

### 1.3 User-Study Metrics (explicit measurement with participants)

| Metric | Instrument | When |
|---|---|---|
| Group Coherence Rate | Review session log | Phase 1+ |
| Review Burden Score | Composite (see §3) | Phase 1+ |
| Decision Compression Ratio | Action log | Phase 1+ |
| System Usability | SUS (10 items, 0-100) | Phase 1+ |
| Cognitive Load | NASA-TLX (6 subscales) | Phase 2+ |
| Re-finding success | Timed task: "find file X" | Phase 2+ |
| Context Boundary Accuracy | Post-review survey | Phase 2+ |
| Decision Fatigue Markers | Decision accuracy over time in session | Phase 2+ |
| OBI before/after | Filesystem scan | All phases |

---

## 2. Prior Art Review

### 2.1 Bergman et al. (2010) — Folder Structure and Navigation

**Citation:** Bergman, O., Whittaker, S., Sanderson, M., Nachmias, R., & Ramamoorthy, A. (2010). The effect of folder structure on personal file navigation. *JASIST*, 61(12), 2426–2441. https://doi.org/10.1002/asi.21415

**Study design:** N = 296 participants. 1,131 files retrieved total. 5,035 navigation steps analyzed.

**Key baseline values (gold standards for Curator):**

| Metric | Value |
|---|---|
| Overall success rate | 94% |
| Direct success (no backtracking) | 79% |
| Eventual success (with backtracking) | 15% |
| Mean folder depth reached | 2.86 |
| Mean files per folder | 11.82 |
| Mean time to retrieve (direct success) | ~15 seconds |

**Implication for Curator:** Any folder structure Curator creates should aim for depth ≤ 3 and 8–15 files per folder. These are empirically validated for fast re-finding. Curator's OBI (see §3.6) already uses Folder Depth Entropy — calibrate against these baselines.

**Curator decision:** Use Bergman's 94% success rate and ~15-second re-find time as the reference baseline. Phase 2 user study must measure whether post-Curator performance meets or exceeds these values.

---

### 2.2 Elsweiler, Ruthven & Jones (2007) — Memory and Re-finding

**Citation:** Elsweiler, D., Ruthven, I., & Jones, C. (2007). Towards memory supporting personal information management tools. *JASIST*, 58(7), 924–946. https://doi.org/10.1002/asi.20570

**Study design:** Diary study, 25 participants from diverse backgrounds. Participants recorded re-finding tasks encountered during normal daily activities.

**Key findings:** The study showed that memory lapses are the primary cause of re-finding failure. Participants frequently failed to remember (a) where they stored information, (b) what they named it, and (c) when they created it. The paper identifies three memory components that PIM tools should support: episodic memory (when/where created), semantic memory (content/meaning), and prospective memory (intended future use).

**Implication for Curator:** Group-level organization supports semantic memory ("this is my Cyber Security work") but Curator should also surface temporal cues ("last modified 3 weeks ago") and intent cues ("labeled TODO") during review. Metrics should include whether users can find files using folder navigation (memory-supported) vs. search (memory failure fallback).

**Curator decision:** Add a temporal cue surface to group display in Review Hub. Track the ratio of navigation vs. search as an implicit memory-support metric.

---

### 2.3 Brackenbury et al. (2021) — Perceived Co-grouping

**Citation:** Brackenbury, W., Harrison, G., Chard, K., Elmore, A., & Ur, B. (2021). Files of a feather flock together? Measuring and modeling how users perceive file similarity in cloud storage. *Proceedings of SIGIR 2021*. https://par.nsf.gov/biblio/10227474

**Study design:** User study examining how participants perceive file similarity along multiple dimensions (topical, temporal, access pattern, file type, project association) and whether "co-management desire" — wanting to manage two files together — predicts actual co-location in folder hierarchies.

**Perceived Co-grouping metric:** For each pair of files (i, j), a binary variable `cogroup(i,j) = 1` if the user states they want to manage i and j together. Agreement across annotators is computed as pairwise Cohen's κ. Cluster-level co-grouping rate = `|approved_pairs_in_cluster| / |total_pairs_in_cluster|`.

**Implication for Curator:** Curator's Group Coherence Rate (§3.3) approximates this without pairwise elicitation — instead, group approval without splitting serves as a proxy. For a more rigorous study, sample 50 random file pairs from each group and ask users "do you want these together?"

**Curator decision:** Use Group Coherence Rate as primary metric. For Phase 2 user study only, collect pairwise co-grouping data on a 10% sample of groups for validation.

---

### 2.4 Cushing (2022) — PIM Burden Framework

**Citation:** Cushing, A. L. (2022). Personal information management burden: A framework for describing nonwork personal information management in the context of inequality. *JASIST*, 73(11), 1543–1558. https://doi.org/10.1002/asi.24692

**Note:** The 2022 JASIST "PIM-B" framework was authored by Cushing, not Dinneen & Julien. Dinneen's 2022 contribution was a separate workshop paper on PIM and death (pimworkshop.org). This should not be conflated.

**PIM Burden components:** The framework identifies PIM burden as arising from: (1) additional PIM activities beyond normal management, (2) negative affect associated with managing information, (3) lack of identity self-extension to one's personal information collection, and (4) additional information seeking required due to poor organization.

**Implication for Curator:** The Review Burden Score (§3.2) operationalizes components (1) and (2). Component (3) suggests a subjective survey item: "I feel my files represent me / my work" (Likert 1–7). Component (4) maps to Search-After-Organization Rate.

**Curator decision:** Add a single-item "ownership" survey question to Phase 2: "After Curator organized your files, do you feel your folder structure represents how you think about your work?" (1=Not at all, 7=Completely).

---

### 2.5 Haraty (2017) — Mixed-Initiative File Organization

No confirmed JASIST 2017 paper by an author named Haraty on mixed-initiative file organization was located in literature databases. The reference as originally specified could not be verified. The closest confirmed work on mixed-initiative file organization evaluation is:

**Confirmed alternative:** Bergman et al. (2013) "Folder versus tag preference in personal information management," JASIST 64(10), which compared acceptance of automated tag suggestions vs. manual folder organization — finding ~70% tag acceptance rate when the system was confident, declining to ~40% for low-confidence suggestions.

**Curator decision:** Treat "70% acceptance rate = good, < 40% = requires redesign" as the working threshold for Group Coherence Rate, sourced from Bergman (2013). Do not cite a Haraty 2017 paper without a verified DOI.

---

## 3. New Metrics for Curator

### 3.1 Decision Compression Ratio (DCR)

**Definition:**
```
DCR = Files_processed / Group_decisions_required
```

Where `Group_decisions_required` = the number of approve / reject / split / merge / rename actions the user performs across one Curator session.

**Example:** 1,000 files processed → 20 group-level decisions → DCR = 50.

**Target:** DCR ≥ 30 for a well-structured filesystem. DCR ≥ 50 is excellent. DCR < 10 means groups are too fragmented (too many small groups requiring individual attention).

**Prior work:** No direct prior work uses the term "Decision Compression Ratio" in PIM. The closest analog is the concept of "initiative compression" in mixed-initiative systems (Horvitz 1999: "Principles of mixed-initiative user interfaces," CHI 1999), where the value of a proactive system is measured partly by how many user actions it replaces. DCR is a novel operationalization of this principle for file organization.

**Implementation:** Log every user action in the Review Hub (approve_group, reject_group, split_group, merge_groups, rename_group). DCR = `total_files_in_session / len(action_log)`. Report per-session and aggregate.

**Curator decision:** DCR is a primary headline metric. Report DCR in the abstract of any IUI 2027 paper submission. Target DCR ≥ 30 in Phase 1 (developer's own 30k filesystem).

---

### 3.2 Review Burden Score (RBS)

**Definition:** A composite measure of the cognitive load imposed by a single Review Hub session.

```
RBS = w1 * N_items + w2 * N_duplicate_families + w3 * N_failed_files 
      + w4 * (1 / mean_group_size)
```

Where:
- `N_items` = total number of groups shown in Review Hub
- `N_duplicate_families` = number of duplicate clusters requiring decision
- `N_failed_files` = files Curator could not classify (presented individually)
- `mean_group_size` = average files per group (larger groups = fewer decisions needed)
- Weights `w1..w4` to be calibrated empirically in Phase 1

**Motivation:** Larger Review Hub = more decisions = more cognitive load. Failed files are especially burdensome (user must handle individually). Duplicate families require binary decisions but are still distracting. Larger groups reduce burden.

**Normalized RBS:** Divide by `Files_processed` to get burden per file.

**Initial weights (to calibrate):** `w1=1.0, w2=2.0, w3=3.0, w4=5.0` (failed files and small groups penalized more heavily).

**Curator decision:** After Phase 1, run sensitivity analysis on weights. Track RBS per session over time — it should decrease as Curator learns the user's filesystem structure.

---

### 3.3 Group Coherence Rate (GCR)

**Definition:**
```
GCR = Groups_approved_without_modification / Total_groups_reviewed
```

Where "approved without modification" = user clicked Approve and did not subsequently split, rename, or partially reject the group.

**Interpretation:**
- GCR ≥ 0.80: Excellent. Curator's groups match user's mental model.
- GCR 0.60–0.79: Acceptable. Some mismatches; investigate which group types fail.
- GCR < 0.60: Groups are wrong. The context graph or clustering threshold needs adjustment.

**Stratified GCR:** Compute separately for:
- Large groups (≥ 20 files) vs. small groups (< 5 files)
- Cross-application groups vs. single-application groups
- Groups named by Curator vs. groups named by existing folder structure

**Curator decision:** GCR is a core metric for the group-first pipeline validity. Log it per session, per group-type. If GCR for small groups < 0.5, implement a merge-small-groups pass before surfacing to user.

---

### 3.4 Duplicate Recall

**Definition:**
```
Duplicate_Recall = TP_duplicates / (TP_duplicates + FN_duplicates)
```

Where:
- `TP_duplicates` = duplicate pairs Curator correctly identified
- `FN_duplicates` = duplicate pairs that exist in the filesystem but Curator missed

**Ground truth construction:** For evaluation purposes, randomly sample 200 files from the filesystem and manually audit for duplicates/near-duplicates. Use this audit as the ground truth set. This is the only metric that requires manual annotation.

**Duplicate Precision:**
```
Duplicate_Precision = TP_duplicates / (TP_duplicates + FP_duplicates)
```

High precision, lower recall is the correct tradeoff for Curator: it is worse to falsely flag originals as duplicates than to miss some actual duplicates.

**Target:** Precision ≥ 0.90, Recall ≥ 0.70 on the sampled set.

**Curator decision:** Prioritize precision over recall in the duplicate/version gate. Implement recall measurement on a 200-file random sample from Phase 1 filesystem.

---

### 3.5 Undo Rate

**Definition:**
```
Undo_Rate = Undo_actions / Total_commits
```

Where:
- `Undo_actions` = number of times user triggers "Restore" or "Undo" after a commit
- `Total_commits` = number of group commits executed

**Target:** Undo_Rate < 5%.

**Interpretation:** Each undo = Curator made a move the user actively disagrees with. This is the most direct measure of Curator making wrong decisions (as opposed to GCR, which measures wrong groupings before they are committed).

**Temporal decay:** Undo within 1 hour of commit = strong rejection signal. Undo within 1 week = mild rejection (user may have reconsidered). Track both.

**Curator decision:** Log all undos with timestamps. Alert the developer (Phase 1) if Undo_Rate exceeds 5% in any single session — this indicates a systematic error in the pipeline.

---

### 3.6 Organizational Burden Index (OBI) — Before/After

**Definition (from existing Curator research):**
```
OBI = α * H(location|intent) + β * FolderDepthEntropy + γ * OrphanRatio + δ * (1 - FLRS)
```

Where:
- `H(location|intent)` = entropy of file locations given intent labels — high entropy = files for a given purpose are scattered
- `FolderDepthEntropy` = Shannon entropy of the depth distribution across all files
- `OrphanRatio` = fraction of files in the root or in folders containing only 1 file
- `FLRS` = Folder-Level Relevance Score (fraction of files in a folder that belong together semantically)
- `α, β, γ, δ` are calibration weights

**Before/After measurement:** Run OBI on the full filesystem before Curator, then again after a complete Curator session (with user approvals committed). OBI improvement = `OBI_before - OBI_after`.

**Validation status:** OBI is currently theoretical/internal (Curator research). It has NOT been validated against a user study. Treat OBI changes as a direction indicator, not a ground truth until Phase 2 validation.

**Curator decision:** Compute OBI automatically at session start and end (Phase 1). Do not publish OBI as a validated metric until it correlates with user-reported satisfaction in Phase 2. This is a research contribution if the correlation is strong.

---

### 3.7 Context Boundary Accuracy (CBA)

**Definition:** When Curator creates separate groups for semantically related but contextually different content (e.g., "New Cyber Security 2026" vs "Old Security Notes 2023"), what fraction of those separations does the user agree with?

```
CBA = User_approved_separations / Total_separations_shown
```

Measure during review: if Curator creates two groups from what was originally one folder, present both groups simultaneously and ask "Should these be separate?" Log yes/no.

**This is distinct from GCR** — GCR measures within-group approval, CBA measures between-group boundary approval.

**Curator decision:** Surface CBA as a secondary metric in Phase 2. It is especially important for testing whether Curator's context graph correctly separates temporal phases of the same topic.

---

## 4. Neurodivergent-Specific Evaluation

### 4.1 Why Standard PIM Metrics Are Insufficient

Standard PIM evaluation (success rate, re-find time, navigation depth) treats users as homogeneous. For ADHD and executive dysfunction users:

1. **Decision fatigue is accelerated.** Neurotypical decision fatigue occurs after hundreds of decisions across hours (Israeli judges study, Danziger et al. 2011, PNAS). ADHD users may experience equivalent depletion after 20-30 decisions within a single session. The Review Hub must be designed with this threshold in mind.

2. **"Success" does not mean "low stress."** A neurotypical user may successfully find a file in 15 seconds but find the process moderately frustrating. An ADHD user may find the same file in 15 seconds but experience significant anxiety during the search. Standard PIM metrics don't capture this.

3. **Abandonment is a primary failure mode.** If an ADHD user abandons the Review Hub mid-session, the files remain in limbo — neither organized nor abandoned cleanly. Standard completion-rate metrics don't flag this as a special class of failure.

4. **Cognitive load instruments designed for aviation (NASA-TLX) may not be calibrated for desktop software tasks.** The original NASA-TLX was designed for Air Force pilots. Its Physical Demand subscale is irrelevant. Frustration and Mental Demand subscales are the most relevant for Curator.

---

### 4.2 Decision Fatigue — Operationalization for Curator

**The Danziger et al. (2011) Israeli judges finding:** Favorable rulings dropped from ~65% at session start to nearly 0% just before breaks, recovering to ~65% after rest. This is the most cited evidence that decision quality degrades within a session (PNAS doi: 10.1073/pnas.1018033108).

**Note:** Subsequent work (e.g., Glöckner et al., Judgment and Decision Making) has questioned the causal mechanism (hunger vs. cognitive fatigue vs. case ordering), but the basic phenomenon — quality degradation within decision sequences — is robust.

**Curator implementation of decision fatigue detection:**

```python
# For each review session, track:
session_decisions = [(timestamp, action, group_id, user_spent_time_seconds), ...]

# Decision quality proxy: time spent per decision
# Short time + high modification rate = lower engagement quality
decision_time_series = [d.user_spent_time for d in session_decisions]

# Compute linear trend: negative slope = degrading engagement
slope = linregress(range(len(decision_time_series)), decision_time_series).slope

# If slope < threshold AND session_length > 15 decisions: trigger "Take a break" prompt
```

**Overwhelm threshold:** Based on clinical ADHD literature (not yet empirically validated for Curator), aim for Review Hub sessions ≤ 15 items before a natural break or "pause and continue later" offer. This is a testable hypothesis for Phase 2.

**Curator decision:** Cap the Review Hub at 15 items per session by default (configurable). Offer "Continue later" prominently. Measure abandonment rate before and after this cap is implemented.

---

### 4.3 Cognitive Load Instruments

**NASA-TLX** (Hart & Staveland 1988): Validated, widely used, 6 subscales (Mental Demand, Physical Demand, Temporal Demand, Performance, Effort, Frustration). For Curator, Physical Demand can be dropped. Administer after each Phase 2 session (paper or digital form takes 2 minutes).

**Raw TLX vs. weighted TLX:** The simpler Raw TLX (unweighted average of subscales) performs equivalently to the weighted version for most software evaluation contexts. Use Raw TLX.

**Alternative: Single-Ease Question (SEQ):** A single 7-point scale "Overall, how would you rate the difficulty of this task?" Validated as a sensitive usability metric by Sauro & Dumas (2009). Administer after each group decision in Phase 2 to track intra-session fatigue.

**Pupillometry (objective):** Pupil dilation correlates reliably with cognitive load (validated across multiple studies, including desktop software). Requires an eye-tracker. The RIPA2 index (real-time pupillometric assessment) isolates cognitive-load-specific frequency bands from pupil data. Feasible for a university lab study but not for a home study. Mark this as optional for Phase 2.

**Curator decision:** Use NASA-TLX (5 subscales, drop Physical Demand) post-session. Use SEQ after each major decision in Phase 2. Reserve pupillometry for a Phase 2b lab component if university lab access is available.

---

### 4.4 Neurodivergent-Specific Metrics — Proposed

The following metrics are not standard in PIM evaluation and represent Curator's contribution to neurodivergent-centered evaluation methodology:

**Decision Quality Degradation Index (DQDI):**
```
DQDI = (GCR_first_third_of_session - GCR_last_third_of_session) / GCR_first_third_of_session
```
Measures whether group approval quality degrades as the session progresses. DQDI > 0.2 = significant fatigue effect; consider reducing session length.

**Overwhelm Abandonment Point (OAP):**
The number of items reviewed at the moment of abandonment. Collect across users, compute median OAP. Cluster by neurotype (ADHD vs. neurotypical). If ADHD median OAP < neurotypical median OAP, evidence that ADHD users hit cognitive limits sooner.

**Re-engagement Rate:**
```
Re-engagement_Rate = sessions_resumed_after_abandonment / total_abandonment_events
```
High re-engagement = the "Continue later" design works. Low re-engagement = abandoned = permanently unorganized. Target: > 70% of abandoners resume within 48 hours.

**Curator decision:** Track DQDI, OAP, and Re-engagement Rate from Phase 1 (developer's own data). These are novel metrics — if the ADHD/neurotypical gap in OAP is statistically significant in Phase 2, this is a publishable finding in its own right.

---

## 5. Evaluation Protocol

### Phase 0 — Automated Evaluation (no users, weeks 1-4)

**Goal:** Validate that Curator's pipeline produces objectively good clusters before exposing users.

**Dataset:** HippoCamp benchmark (arXiv:2604.01221) — 42.4 GB, 2K+ real-world files, 581 QA pairs. Use HippoCamp's file-retrieval QA pairs to test whether Curator's organization enables correct answers.

**Metrics to compute:**
- DBCV on all context groups (should be > 0.0; target > 0.2)
- Cohesion Ratio per group (target > 1.5)
- Name-Embedding Cosine per folder (target > 0.60)
- DCR on full HippoCamp dataset
- Duplicate Recall on HippoCamp's known duplicate pairs (if any are seeded)
- QA accuracy: after Curator organizes HippoCamp, can the LLM answer HippoCamp's 581 QA pairs by navigating the Curator-organized structure?

**Tools needed:** FelSiq/DBCV, sentence-transformers, HippoCamp benchmark (github.com/synvo-ai/HippoCamp).

**Cost:** 2–4 weeks of implementation. No IRB needed. No participants.

**Curator decision:** Phase 0 is a prerequisite for paper submission. QA accuracy on HippoCamp provides an objective, reproducible, referenceable benchmark result. This is the "automated evaluation" section of any IUI 2027 paper.

---

### Phase 1 — Single User Study (developer, weeks 5-10)

**Goal:** Validate metrics on real filesystem (30k files). Catch pipeline failures before exposing other users.

**Dataset:** Developer's own 30k-file filesystem (already ingested).

**Protocol:**
1. Compute OBI baseline before Curator session
2. Run full Curator pipeline (adaptive scan → duplicate gate → context graph → Review Hub)
3. Complete 3+ Review Hub sessions over 2 weeks (natural usage)
4. After each session, complete NASA-TLX (5 subscales) and SUS

**Metrics to collect:**
- DCR per session
- GCR per session (stratified by group type)
- RBS per session
- Undo Rate (aggregate)
- OBI delta (before vs. after each commit batch)
- Session abandonment (did the developer stop mid-session?)
- Time-to-Useful-Structure: time from first pipeline run to first committed batch

**SUS interpretation:** Score ≥ 68 = meets average usability baseline. Score ≥ 80 = good. Score ≥ 90 = excellent. Target for Phase 1: ≥ 70 (acceptable; neurodivergent-targeted software should aim higher in later phases).

**Curator decision:** Phase 1 is the developer validation phase. Results are not publishable as a user study but provide calibration data for Phase 2 recruitment and protocol design. Any systematic DCR < 10 or GCR < 0.50 must trigger pipeline fixes before Phase 2.

---

### Phase 2 — Small User Study (5-10 users, weeks 11-20)

**Goal:** Measure Curator's value for ADHD users vs. neurotypical users.

**Recruitment:** 3 ADHD users (self-reported diagnosis or screening-confirmed), 3 neurotypical users, 2+ additional mixed. Mixed filesystems (no HippoCamp — use real user filesystems).

**Design:** Within-subjects, counterbalanced. Each participant organizes two comparable folder collections:
- Condition A: with Curator (group-first Review Hub)
- Condition B: manual organization (drag-and-drop + folder creation)

**Primary measures:**
- Re-finding time (timed task: find file X that was organized 1 week ago)
- Re-finding success rate (binary: found within 5 minutes = success)
- DCR (Curator condition only)
- GCR (Curator condition only)
- NASA-TLX per condition per session
- SUS per condition
- Abandonment Rate (Curator condition)
- DQDI (Curator condition)

**Secondary measures:**
- OAP by neurotype
- Re-engagement Rate
- Pairwise co-grouping sample (10% of groups, Brackenbury-style)
- CBA (Context Boundary Accuracy)
- "Ownership" survey item (Cushing-inspired)

**Statistical analysis:**
- Mixed ANOVA: neurotype (ADHD vs. NT) × condition (Curator vs. manual) × time
- Primary hypothesis: Curator ADHD users achieve re-finding success rate ≥ Bergman baseline (94%) despite starting from higher organizational chaos
- Secondary hypothesis: ADHD OAP < NT OAP (p < 0.05)

**IRB:** Required. Protocol must cover: filesystem access consent, screen recording consent, neurotype disclosure. See Curator Research file 61 for existing IRB notes.

**Curator decision:** Phase 2 is the minimum required for an IUI 2027 paper. 6 participants (3 ADHD, 3 NT) is small but acceptable for a CHI/IUI paper if effect sizes are large and within-subjects design is used. Aim for 8+ participants to allow for attrition.

---

### Phase 3 — Longitudinal Deployment (4 weeks, concurrent with or after Phase 2)

**Goal:** Measure implicit metrics in real use, not lab-controlled tasks.

**Participants:** Same as Phase 2, or developer + 2-3 willing Phase 2 participants.

**Protocol:** Enable Autopilot Mode (daily background scan). No scripted tasks. Participants use their computer normally.

**Metrics collected passively:**
- Search-After-Organization Rate (weekly)
- Re-organization Rate (weekly)
- Undo Rate (per session)
- Session frequency (does the user keep engaging with Curator?)
- OBI delta (weekly filesystem scan)

**Analysis:** Compute metric trajectories over 4 weeks. Hypothesis: Re-organization Rate decreases over time as Curator learns the user's structure. Undo Rate should stabilize below 5% by week 3.

**Curator decision:** Phase 3 provides longitudinal evidence that is rare in PIM papers. Even a 4-week n=3 longitudinal study is valuable at IUI. Prioritize if Phase 2 runs on schedule.

---

## 6. Benchmarks — Applicability Assessment

### 6.1 HippoCamp (arXiv:2604.01221, April 2026)

**Description:** 42.4 GB synthetic personal file system based on real user profiles. 581 QA pairs covering search, evidence perception, and multi-step reasoning. 46.1K densely annotated trajectories for step-wise failure diagnosis. Best commercial models achieve only 48.3% accuracy on user profiling tasks.

**Evaluation protocol:** Agent navigates the file system and answers QA pairs. Metrics: accuracy, trajectory step count, modality-specific sub-scores.

**Applicability to Curator:** HIGHLY APPLICABLE. HippoCamp provides:
1. A reproducible filesystem that Curator can organize
2. 581 QA pairs to test whether Curator's organization improves retrieval
3. A published baseline (48.3%) to compare against

**How to use:** Run Curator on HippoCamp filesystem → measure QA accuracy of a downstream retrieval agent navigating the Curator-organized structure → compare against HippoCamp baseline.

**Curator decision:** Use HippoCamp as the primary Phase 0 benchmark. This is the strongest automated evaluation story for IUI 2027.

---

### 6.2 CFCOS — Content-Based File Classification and Organization System (Electronics 2026)

**Citation:** Content-Based File Classification and Organization System Using LLMs. *Electronics* 2026, 15(7), 1524. https://doi.org/10.3390/electronics15071524

**Description:** LLM-based file classification into semantic categories using content-derived summaries. Addresses limitations of metadata-only classification. Demo code: github.com/B23en/CFCOS_demo_simulation_for_reference.

**Evaluation metrics used:** Classification accuracy per category (file type and semantic category). The system generates semantic summaries and classifies into predefined categories.

**Applicability to Curator:** PARTIALLY APPLICABLE. CFCOS evaluates individual file classification, not group-level organization. Its accuracy scores (category-level) can serve as a baseline for Curator's file-level category assignment in the context graph step. The CFCOS approach (LLM content summarization → category label) is methodologically similar to Curator's context-tagging step.

**Curator decision:** Use CFCOS as a baseline for file-level classification accuracy in Phase 0. If Curator's context graph achieves better category-level accuracy than CFCOS on the same file set, this is a contribution to report.

---

### 6.3 20 Newsgroups and RCV1-v2

**20 Newsgroups:** 20,000 newsgroup posts across 20 categories. Standard NLP clustering benchmark.

**RCV1-v2:** Reuters Corpus Volume 1, 800,000+ newswire articles with hierarchical topic labels.

**Applicability to Curator:** NOT APPLICABLE for primary evaluation. These are text-only, topic-labeled corpora with clean domain separation. Personal filesystems have:
- Mixed modalities (PDFs, images, code, DOCX)
- No ground-truth topic labels
- Overlapping contexts (a file can belong to multiple projects)
- Strong temporal and relational structure that 20NG/RCV1 lacks

**Partial use case:** If Curator's context graph is tested purely on its text-clustering component (ignoring modality, temporal, and relational features), 20NG/RCV1 can validate that the underlying clustering algorithm is correct. But this is an internal validation step, not a PIM evaluation.

**Curator decision:** Do not use 20NG or RCV1 as primary benchmarks. Use them only for isolated algorithm validation if needed. Prefer HippoCamp for integrated system evaluation.

---

## 7. LLM-as-Judge

### 7.1 ProxAnn (ACL 2025, arXiv:2507.00828)

**Citation:** Hoyle, A., Calvo-Bartolomé, L., Boyd-Graber, J., & Resnik, P. (2025). ProxAnn: Use-oriented evaluations of topic models and document clustering. *ACL 2025 (Main)*. arXiv:2507.00828. GitHub: https://github.com/ahoho/proxann

**Method:**
1. Show the LLM a set of documents assigned to a cluster/topic
2. LLM infers a category label for the group (without being given the label)
3. LLM applies that inferred label to classify held-out documents from the same cluster and documents from other clusters
4. Metric: accuracy of held-out document classification

**Key finding:** The best LLM proxies are statistically indistinguishable from human crowd annotators. This means ProxAnn can replace expensive human annotation for cluster quality evaluation.

**How to apply to Curator:**

```python
# For each group G produced by Curator:
# 1. Sample 10 files from G (show filenames + summaries to LLM)
# 2. Ask LLM: "What is this group about? Give a one-sentence purpose."
# 3. Show LLM 5 held-out files from G + 5 files from other groups
# 4. Ask LLM: "Which of these files belong to the group with purpose X?"
# 5. Score: accuracy of LLM on this held-out classification task

proxann_score(G) = held_out_accuracy(G)
proxann_global = mean([proxann_score(G) for G in all_groups])
```

**ProxAnn score interpretation:** > 0.85 = excellent coherence. 0.70–0.85 = good. < 0.70 = groups are not coherent enough for an LLM to infer their purpose.

**Cost:** ~$0.10 per group at Claude claude-sonnet-4-6 API prices (10 files × ~200 tokens each). For 50 groups = ~$5. Cheap enough to run after every pipeline change.

**Curator decision:** Implement ProxAnn evaluation as part of the Phase 0 automated test suite. Run it on every HippoCamp evaluation and report ProxAnn score alongside DBCV and Cohesion Ratio.

---

### 7.2 Coherence Verification (arXiv:2604.07562)

**Citation:** Islam, T. (2026). Reasoning-based refinement of unsupervised text clusters with LLMs. arXiv:2604.07562.

**Method:** Three-stage LLM pipeline:
1. **Coherence verification:** LLM assesses whether cluster summaries are supported by member texts (boolean: coherent / incoherent)
2. **Redundancy adjudication:** LLM merges or rejects candidate clusters based on semantic overlap
3. **Label grounding:** LLM assigns interpretable labels in an unsupervised manner

**How to apply to Curator:**

Stage 1 (coherence verification) maps directly to Curator's group quality check:

```python
prompt = f"""
You are evaluating a file organization group.
Group name: {group.name}
Group summary: {group.summary}
Files in group: {[f.name + ": " + f.one_line_summary for f in group.files[:10]]}

Question: Is this group coherent — do all files clearly belong together based on the group name and summary?
Answer: YES or NO, followed by one sentence of reasoning.
"""
```

Stage 2 (redundancy adjudication) maps to Curator's merge-groups decision:

```python
# Run for all group pairs with cosine(group_embedding_i, group_embedding_j) > 0.75
# Ask LLM: "Should these two groups be merged or kept separate?"
```

**Curator decision:** Use Coherence Verification (Stage 1) as a pre-surface filter in the Review Hub pipeline. Groups that fail LLM coherence verification are auto-split or flagged for review before being shown to the user. This reduces GCR failures before they reach the user.

---

### 7.3 Combined LLM-Judge Pipeline for Curator

```
Curator groups (post-clustering)
  → Stage 1: Coherence Verification (arXiv:2604.07562) — filter incoherent groups
  → Stage 2: Redundancy Adjudication — auto-merge high-overlap groups  
  → Stage 3: ProxAnn scoring — score remaining groups for held-out accuracy
  → Stage 4: Name-Embedding Cosine — score folder names
  → Surface to user (Review Hub)
```

Groups that pass all 4 stages have high automated confidence. Groups that fail any stage are either auto-fixed (merge, split) or flagged with a confidence indicator in the Review Hub.

**Curator decision:** Implement this 4-stage pipeline in Phase 0. Make the confidence indicator visible to the user in the Review Hub ("Curator is confident about this group" / "Curator is less sure — please check").

---

## 8. Papers and Citations

### Full Citations

1. **Bergman, O., Whittaker, S., Sanderson, M., Nachmias, R., & Ramamoorthy, A. (2010).** The effect of folder structure on personal file navigation. *Journal of the American Society for Information Science and Technology*, 61(12), 2426–2441. https://doi.org/10.1002/asi.21415

2. **Elsweiler, D., Ruthven, I., & Jones, C. (2007).** Towards memory supporting personal information management tools. *Journal of the American Society for Information Science and Technology*, 58(7), 924–946. https://doi.org/10.1002/asi.20570

3. **Brackenbury, W., Harrison, G., Chard, K., Elmore, A., & Ur, B. (2021).** Files of a feather flock together? Measuring and modeling how users perceive file similarity in cloud storage. *Proceedings of the 44th International ACM SIGIR Conference*, 2111–2121. https://par.nsf.gov/biblio/10227474

4. **Cushing, A. L. (2022).** Personal information management burden: A framework for describing nonwork personal information management in the context of inequality. *Journal of the Association for Information Science and Technology*, 73(11), 1543–1558. https://doi.org/10.1002/asi.24692

5. **Moulavi, D., Jaskowiak, P. A., Campello, R. J. G. B., Zimek, A., & Sander, J. (2014).** Density-based clustering validation. *Proceedings of the 2014 SIAM International Conference on Data Mining (SDM)*, 839–847. https://doi.org/10.1137/1.9781611973440.96

6. **HippoCamp (2026).** HippoCamp: Benchmarking contextual agents on personal computers. arXiv:2604.01221. GitHub: https://github.com/synvo-ai/HippoCamp

7. **CFCOS (2026).** Content-based file classification and organization system using LLMs. *Electronics*, 15(7), 1524. https://doi.org/10.3390/electronics15071524. GitHub: https://github.com/B23en/CFCOS_demo_simulation_for_reference

8. **Hoyle, A., Calvo-Bartolomé, L., Boyd-Graber, J., & Resnik, P. (2025).** ProxAnn: Use-oriented evaluations of topic models and document clustering. *ACL 2025 (Main)*. arXiv:2507.00828. GitHub: https://github.com/ahoho/proxann

9. **Islam, T. (2026).** Reasoning-based refinement of unsupervised text clusters with LLMs. arXiv:2604.07562.

10. **Bautista, L. A., et al. (2025).** Ground truth clustering is not the optimum clustering. *Scientific Reports*. arXiv:2305.13218. https://doi.org/10.1038/s41598-025-90865-9

11. **Liu, S., Tian, S., Hu, K., et al. (2026).** FileGram: Grounding agent personalization in file-system behavioral traces. arXiv:2604.04901.

12. **Danziger, S., Levav, J., & Avnaim-Pesso, L. (2011).** Extraneous factors in judicial decisions. *Proceedings of the National Academy of Sciences*, 108(17), 6889–6892. https://doi.org/10.1073/pnas.1018033108

13. **(arXiv:2511.19350)** Scalable parameter-light spectral method for clustering short text embeddings with a cohesion-based evaluation metric. arXiv:2511.19350.

14. **Hart, S. G., & Staveland, L. E. (1988).** Development of NASA-TLX (Task Load Index): Results of empirical and theoretical research. *Human Mental Workload*, 1, 139–183.

15. **Brooke, J. (1996).** SUS: A quick and dirty usability scale. In P. W. Jordan et al. (Eds.), *Usability Evaluation in Industry*, 189–194. Taylor & Francis.

16. **Horvitz, E. (1999).** Principles of mixed-initiative user interfaces. *Proceedings of CHI 1999*, 159–166.

17. **Bergman, O., Whittaker, S., & Yanai, S. (2013).** Folder versus tag preference in personal information management. *JASIST*, 64(10). https://doi.org/10.1002/asi.22906

18. **Sauro, J., & Dumas, J. S. (2009).** Comparison of three one-question, post-task usability questionnaires. *Proceedings of CHI 2009*, 1599–1608.

---

## 9. Design Decisions

**DD-01: DBCV over Silhouette**  
Curator must use DBCV (FelSiq/DBCV implementation) and Cohesion Ratio for automated cluster quality, never Silhouette score. Silhouette assumes convex clusters; HDBSCAN/UMAP-based context groups are non-convex. Reporting Silhouette in a paper would be methodologically incorrect.

**DD-02: DCR is the headline metric**  
Decision Compression Ratio is Curator's most distinctive contribution to PIM evaluation methodology. It directly captures the "group-first" value proposition: fewer decisions per organized file. All paper abstracts and demo descriptions should lead with DCR.

**DD-03: GCR threshold is 0.70**  
Based on Bergman (2013)'s finding that ~70% acceptance rate for confident automated suggestions represents a working system, set GCR = 0.70 as the minimum acceptable threshold. Below 0.70 = pipeline needs fixing before user studies.

**DD-04: ProxAnn in the automated pipeline**  
ProxAnn (LLM held-out classification accuracy) is the primary automated quality gate. Groups with ProxAnn score < 0.70 should be auto-split or flagged before surfacing to the user. This prevents low-coherence groups from reaching the Review Hub.

**DD-05: Session cap at 15 items**  
Based on ADHD decision fatigue literature and Danziger et al. (2011), cap the default Review Hub session at 15 items. This is a testable hypothesis (Phase 2 can vary this parameter). Make it configurable.

**DD-06: HippoCamp as Phase 0 benchmark**  
HippoCamp (arXiv:2604.01221) is the correct automated benchmark for Curator. Use its QA accuracy protocol: organize HippoCamp filesystem with Curator, then test whether a retrieval agent can answer HippoCamp's 581 QA pairs better than the 48.3% baseline.

**DD-07: OBI is a direction indicator, not a validated metric**  
OBI should be computed before/after each session and reported internally, but not published as a validated user-study metric until Phase 2 produces a correlation with user satisfaction. The OBI formula needs empirical calibration of its weights.

**DD-08: Undo Rate < 5% is a hard quality gate**  
If Undo Rate exceeds 5% in Phase 1, halt Phase 2 recruitment. The pipeline is not ready. Undo is the strongest signal of active user rejection.

**DD-09: Separate ADHD and neurotypical cohorts in Phase 2**  
Statistical analysis must be stratified by neurotype. A mixed-ANOVA with neurotype as a between-subjects factor and condition (Curator vs. manual) as within-subjects is the correct design. This produces the key publishable finding: does Curator's benefit differ by neurotype?

**DD-10: Coherence Verification as a pre-filter**  
Apply Islam (2026)'s Stage 1 coherence verification to all groups before surfacing to the user. This is an automated pipeline improvement, not just an evaluation metric. Groups that fail coherence verification are auto-refined, reducing the user's correction burden.

**DD-11: Do not use 20NG or RCV1 as primary benchmarks**  
These datasets do not represent personal filesystem structure, mixed modalities, or overlapping contexts. Their use would misrepresent Curator's evaluation domain.

**DD-12: Report OAP by neurotype**  
The Overwhelm Abandonment Point (number of items reviewed at abandonment) is a novel metric. If ADHD users consistently abandon at lower OAP than neurotypical users, this is a publishable finding about ADHD-specific overwhelm thresholds in file organization software.

---

## 10. Open Questions

**OQ-01: How to weight OBI components?**  
The OBI formula has four components (H(location|intent), FolderDepthEntropy, OrphanRatio, FLRS) with weights α, β, γ, δ. What are the correct weights? Currently unknown. Phase 2 can provide empirical calibration if OBI correlates with user-reported satisfaction (r ≥ 0.5 would be publishable).

**OQ-02: What is the correct DCR target for a 30k-file filesystem?**  
DCR = 30 means 30 files per decision. Is this the right target for a chaotic filesystem vs. a well-organized one? The target likely depends on baseline organization level. Need to measure DCR across different starting conditions in Phase 1.

**OQ-03: Does Coherence Verification in the pipeline improve GCR enough to justify API cost?**  
Running LLM coherence verification before surfacing groups adds ~$2-5 per full session (API cost). Does this improve GCR by enough to be worth it? Measure GCR with and without this step in Phase 1.

**OQ-04: Is the 15-item session cap correct for ADHD users?**  
The 15-item cap is based on clinical literature, not Curator-specific data. Phase 2 should measure OAP across different session lengths and identify the empirically optimal cap. This may differ by user and context.

**OQ-05: Can FSEvents reliably detect Spotlight searches that resolve to Curator-organized files?**  
The Search-After-Organization Rate requires correlating Spotlight invocations with Curator's placement log. NSWorkspace provides `NSWorkspaceDidActivateApplicationNotification` but not query content. MDQuery APIs can intercept Spotlight queries but require elevated permissions. Is this feasible without a kernel extension? Open engineering question.

**OQ-06: How does HippoCamp's QA accuracy map to real-world re-finding success?**  
HippoCamp tests an agent answering questions about file content. Real users navigate folders with their own memory. The mapping between HippoCamp QA accuracy and human re-finding success is unknown. Need to validate this mapping in Phase 2 (measure both, compute correlation).

**OQ-07: Is a 6-participant Phase 2 study sufficient for IUI 2027?**  
IUI accepts papers with small user studies if the effect sizes are large and the design is within-subjects. However, reviewers may request larger N. A power analysis based on Phase 1 effect sizes should determine the minimum N before Phase 2 recruitment begins.

**OQ-08: How to handle files Curator cannot classify (failed files) in the Undo Rate?**  
If Curator surfaces a file individually (because it could not group it) and the user manually moves it, should this count as an Undo? Or does the undo metric apply only to committed group decisions? Define precisely before Phase 1.

**OQ-09: Does OBI correlate with subjective organizational stress?**  
OBI is a structural metric. It may or may not correlate with the user's subjective feeling of "my files are a mess." A single-item survey question ("On a scale of 1-10, how disorganized do you feel your filesystem is?") administered alongside OBI computation would test this. Add this to Phase 2 protocol.

**OQ-10: What is the minimum ProxAnn score for a group to be shown to the user without warning?**  
Currently proposed: 0.70. This threshold has no empirical basis for Curator specifically. Phase 0 should explore the distribution of ProxAnn scores across HippoCamp groups and use the natural breakpoint in the distribution as the threshold.

---

---

## 11. Operational Pipeline Metrics (R11–R14 + TECH Passports)

These metrics measure Curator's internal engine health — not user-facing value, but engineering correctness. They are monitored internally and surface only when they indicate a problem (R10 failure intelligence model: errors are internal, problems are user-facing).

### 11.1 Completeness Metrics (R13)

| Metric | Formula | Target | Alert threshold |
|---|---|---|---|
| **Inventory Coverage** | `files_with_Layer1 / total_eligible_files` | 1.0 (100%) | < 0.99 |
| **Minimum Understanding Rate** | `files_with_tier_reached ≥ 1 / total_eligible_files` | ≥ 0.95 within 24h of ingest | < 0.80 after 48h |
| **Deep Understanding Rate** | `files_with_tier_reached = 2 / files_with_target_tier_3` | ≥ 0.90 over lifetime | < 0.70 (stalled queue) |
| **Completion Debt Trend** | `Σ (target_tier - current_tier) × priority_weight` over time | Decreasing or 0 | Increasing for > 60 min during idle |
| **Thermal Pause Accumulation** | `COUNT(status = 'paused_thermal') / total_queue_size` | < 0.30 | > 0.70 (system too hot to make progress) |
| **Retry Accumulation Rate** | `COUNT(attempts ≥ 2) / total_queue_size` | < 0.05 | > 0.15 (systematic extraction failures) |

### 11.2 Thermal Governor Metrics (R11)

| Metric | Measurement | Target | Alert |
|---|---|---|---|
| **State Distribution** | % time in each state (Calm/Attentive/Reduced/Paused/Asleep) | Calm ≥ 60% during active use | Paused > 30% of active hours |
| **Transition Frequency** | State changes per hour | < 6/h (stable) | > 20/h (oscillating = threshold too tight) |
| **E-Core Utilization** | % of Idle Completion jobs confirmed on E-cores (via `psutil.cpu_affinity` or `powermetrics`) | > 95% | < 80% (workers leaking to P-cores) |
| **Thermal False Calm** | Jobs that triggered `thermalState > fair` mid-run despite `CALM` passport floor | 0 | Any occurrence → tighten CPU threshold |

### 11.3 PassportGate Metrics (TECH)

| Metric | Formula | Target | Alert |
|---|---|---|---|
| **Gate Allow Rate** | `ALLOW / (ALLOW + DEFER + REJECT)` | ≥ 0.70 during Calm state | < 0.30 (too many deferrals — Mac too stressed) |
| **Defer Reason Distribution** | % of DEFERs by reason (thermal / battery / idle / RAM) | No single reason > 60% | `thermal` > 80% of DEFERs → Mac consistently too hot |
| **DEFER Storm Detection** | ≥ N consecutive DEFERs with 0 ALLOWs | — | 0 completions in > 30 min during expected idle window |
| **Reject Rate** | `REJECT / total_gate_calls` | 0 in production | Any REJECT → passport validation bug in code |
| **Passport Cost Calibration Drift** | `actual_cpu_seconds / estimated_cpu_seconds` per operation type | 0.5–2.0x (within 2x) | > 3x or < 0.3x → update `estimated_cpu_seconds` in registry |

### 11.4 Candidate Graph Metrics (R14)

| Metric | Measurement | Target | Alert |
|---|---|---|---|
| **Graph Freshness** | % of edges with `computed_at > 30 days ago` | < 0.20 | > 0.50 (edge decay not running) |
| **Recall Proxy — Neighbor Stability** | Change in top-K neighbors between successive queries (no file change) | < 0.10 (< 10% churn) | > 0.30 (HNSW degradation, trigger index health check) |
| **Duplicate Consistency** | Files flagged as near-duplicate by R2 where both are in each other's top-K | ≥ 0.90 | < 0.75 (recall loss affecting duplicate detection) |
| **Graph Size** | `COUNT(*)` in `candidate_graph` | ≤ MVP target (300k at 30k files) | > 5M rows without expected scale increase (edge decay failure) |

### 11.5 How These Metrics Are Used

These metrics are **never shown to the user directly**. They feed:
- R10 failure intelligence — thresholds trigger problem cards ("X files need help") or self-healing actions
- Developer diagnostics — available via `GET /health/full` in debug builds
- Automated tests — the TECH_testing_safety_validation.md test specs should include metric regression tests (e.g., after a test run, Inventory Coverage must be 1.0, Gate Reject Rate must be 0)

*End of R9 — Evaluation Metrics*  
*Next: R10 — IRB and Ethics Protocol (see research file 61 for existing notes)*
