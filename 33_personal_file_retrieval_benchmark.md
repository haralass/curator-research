# Personal File Retrieval Benchmarks: What Exists, What's Missing, and What Curator Should Create

*Research compiled: May 2026*

---

## 1. Overview and Motivation

Evaluating a personal file organizer is harder than evaluating a web search engine. There is no universal "correct" folder structure, no shared corpus that researchers can download and re-run experiments on, and no agreed-upon task set. This document surveys the landscape of benchmarks that touch personal file retrieval and personal information management (PIM), identifies the structural gaps in that landscape, and proposes what a principled benchmark for Curator — a macOS AI file organizer targeting IUI 2027 — should look like.

---

## 2. Existing Benchmarks and Evaluation Traditions

### 2.1 TREC and the Pseudo-Desktop Collection Approach

TREC (Text REtrieval Conference), run by NIST, has been the gold standard for IR evaluation since 1992. TREC has covered enterprise e-mail search, genomics, spam filtering, and e-Discovery, but it never produced a dedicated personal file retrieval track. The closest effort was academic work building "pseudo-desktop collections" — synthetic personal filesystems that mimic the statistical properties of real desktops (mix of Office documents, PDFs, email exports, presentations) without sharing actual user data. The core paper, "Retrieval Experiments Using Pseudo-Desktop Collections" (Elsweiler et al., JASIST), defined desktop search as a semi-structured known-item retrieval problem and showed that simulated query generation could produce repeatable experiments. This remains the most cited methodology for replicable desktop IR evaluation, but it evaluates full-text search ranking, not organizational quality or navigation behavior.

TREC's Interactive Knowledge Assistance Track (iKAT, launched 2023, ongoing through 2024) is the nearest current TREC effort to personal context — it evaluates conversational search agents that adapt to a user's interaction history and personal context — but it does not address file organization or local filesystem navigation.

### 2.2 NTCIR Lifelog

NTCIR (NII Testbed and Community for Information access Research), the Japanese sister conference to TREC, ran the Lifelog task from NTCIR-12 (2015) through NTCIR-14 (2019), with a revival at NTCIR-16 and again at NTCIR-19 (2026). The Lifelog task evaluates retrieval from multimodal personal data archives — wearable camera images, GPS logs, biometric streams, application usage logs — rather than file-system documents per se. Participants retrieve "moments" from a day given a natural-language description. The dataset is semi-anonymized and released to registered participants.

The Lifelog task is the most mature benchmark involving personal data retrieval at scale. Its lessons for Curator include: (a) multimodal evidence is essential for resolving ambiguous queries, (b) temporal context matters enormously (people remember roughly when something happened), and (c) inter-annotator agreement on "relevance" is moderate at best because personal data is inherently subjective.

Importantly, Lifelog does not evaluate file organization — it evaluates search recall given a fixed archive. There is no concept of a user having "correctly" or "incorrectly" structured their data beforehand.

### 2.3 Desktop Search Test-Beds and the PIRE Living-Lab Tool

"Building a Desktop Search Test-Bed" (Kelly et al., SIGIR workshop 2010) proposed a methodology for constructing test collections from real user machines using a privacy-preserving protocol: queries are formed from the user's own history, relevance judgments are collected in-situ, and no raw files leave the user's machine. The Personal Information Retrieval Evaluation (PIRE) tool implemented this as a "living laboratory" — a plugin installed on a participant's actual machine that logs queries, retrieves results from multiple retrieval engines simultaneously, and collects implicit/explicit relevance feedback. This allows cross-system comparison without sharing personal corpora.

PIRE is methodologically elegant but requires sustained participant engagement (participants must use the tool in their daily work for weeks) and produces non-transferable results (no shared corpus, so replication requires a new cohort of participants). No maintained public instance of PIRE exists as of 2026.

### 2.4 Beagle++ and Semantic Desktop Search Evaluation

The Beagle++ system (Minack, Nejdl et al., ESWC 2006, published in full in Web Semantics 2009) extended desktop search with RDF-annotated activity metadata and evaluated its personalized ranking against the baseline Beagle search engine. Evaluation used human judges rating results for personalized queries — a small-scale within-lab study. This work established that activity context (what applications accessed a file, what was co-open at the time) significantly improves retrieval precision over full-text alone. The evaluation methodology, however, was not standardized: queries were chosen by the researchers, judges were lab members, and the corpus was not shared.

### 2.5 HippoCamp (2026)

HippoCamp ("Benchmarking Contextual Agents on Personal Computers", arXiv:2604.01221, April 2026) is the most directly relevant recent benchmark. It constructs 581 QA pairs over realistic simulated personal file systems containing over 2,000 real-world files spanning 42.4 GB and multiple modalities (documents, images, code, email exports). The benchmark evaluates multimodal LLM agents on three capabilities: (1) user profile construction from files, (2) known-item retrieval requiring multi-hop evidence across modalities, and (3) multi-step reasoning. Even state-of-the-art commercial models achieve only 48.3% accuracy on user profiling tasks, with multimodal perception and evidence grounding identified as the primary bottlenecks.

HippoCamp evaluates AI agents navigating personal file systems, not the quality of file organization itself. It assumes the file system is already populated and asks: can the agent find things? This makes it a retrieval benchmark, not an organization benchmark.

### 2.6 FileGram and FileGramBench (2026)

FileGram ("Grounding Agent Personalization in File-System Behavioral Traces", arXiv:2604.04901, April 2026) is a framework for personalizing AI agents through behavioral traces — the sequence of file create, move, open, rename, and delete events. It introduces FileGramBench, which evaluates memory systems on four tasks: user profile reconstruction, trace disentanglement (separating interleaved activities), persona drift detection, and multimodal grounding. The data engine (FileGramEngine) generates synthetic but persona-driven action sequences.

FileGram is the closest existing benchmark to what Curator needs: it treats file system behavior, not just file content, as the primary signal. However, FileGramBench evaluates agent memory systems (can the agent build an accurate model of the user?) rather than evaluating file organization quality as experienced by the user.

### 2.7 PIM User Studies: Teevan, Bergman, and the Re-Finding Tradition

Before the modern benchmark era, PIM was evaluated through diary studies and laboratory re-finding tasks.

**Teevan et al.** documented that approximately 40% of web queries are re-finding queries (queries for content the user has previously seen). Teevan's doctoral thesis, "Supporting Finding and Re-Finding Through Personalization" (MIT, 2006), analyzed re-finding behavior across web pages, local files, and email, establishing that people use very different strategies for re-finding vs. finding-new: re-finding tends to be context-anchored and associative rather than keyword-driven.

**Bergman et al.** conducted the most rigorous quantitative studies of personal folder use. In a large-scale elicited retrieval study with 275 users retrieving 860 of their own files, they found that user-created folder hierarchies had a 65% lower failure rate than default folders (e.g., My Documents), and more than five times lower failure rates than folders created by others. They also found a strong preference for folder hierarchy over tagging, and resistance to assigning more than one tag per item. These findings have important implications: any AI file organizer must ultimately be evaluated on whether users can successfully retrieve their own files from the AI-proposed organization — not whether the organization looks "clean" to an external observer.

---

## 3. What Is Missing

### 3.1 No Shared, Privacy-Safe Corpus for File Organization

Every approach reviewed above either sacrifices shareability (PIRE: real machines, no export), sacrifices realism (pseudo-desktop: synthetic content, no behavioral context), or evaluates retrieval rather than organization (HippoCamp, FileGram). There is no publicly available anonymized corpus of real personal file systems with associated access-pattern logs, retrieval success/failure labels, and user satisfaction ratings. The reason is straightforward: personal file systems are among the most privacy-sensitive data a person holds. They contain financial records, medical information, personal correspondence, and professional materials.

### 3.2 No Benchmark for Organizational Quality

Existing benchmarks measure retrieval success (can you find file X?) or content understanding (what does this file collection say about the user?). None measure whether one organizational structure is better than another by the only metric that matters: whether the user can navigate it successfully under realistic time pressure and cognitive load. What does "correct organization" even mean? It is user-specific, task-specific, and time-varying.

### 3.3 No Longitudinal Ground Truth

Files accumulate over years. A benchmark snapshot captures a single moment. But good organization must work over time: categories that made sense three years ago may be obsolete, and new categories may not yet exist. No existing benchmark models the longitudinal dimension of personal file organization.

### 3.4 No IUI/CHI Evaluation Standard for AI File Organizers

Neither IUI nor CHI has a standard evaluation protocol for AI-assisted file organization. Papers in this space use ad hoc user studies with small samples (typically N=10–20), subjective rating scales (Likert, NASA-TLX for cognitive load), and task completion time on artificial tasks. There is no agreed-upon task battery, no standard file corpus for controlled experiments, and no shared baseline system to compare against.

### 3.5 Ecological Validity vs. Replicability

Living-lab approaches (PIRE) have high ecological validity but cannot produce replicable results across labs. Synthetic benchmarks (pseudo-desktop, HippoCamp) are replicable but low in ecological validity. No approach successfully achieves both simultaneously.

---

## 4. What Makes Personal File Retrieval Hard to Benchmark

**Privacy.** Real personal file systems cannot be shared. Any shared corpus must be synthetic or heavily anonymized, and heavy anonymization destroys the semantic coherence that makes the benchmark ecologically valid.

**Subjectivity of ground truth.** There is no objectively "correct" folder structure for a given set of files. What constitutes good organization is user-specific. Benchmark designers must either (a) evaluate against a user's own stated preferences, which requires per-user ground truth collection, or (b) define proxy metrics (task completion rate, navigation depth, search fallback rate) that correlate with user satisfaction without requiring a gold-standard organization.

**The single-user nature of PIM.** Web IR benchmarks aggregate relevance judgments from many judges over a shared corpus, reducing annotation noise. PIM benchmarks cannot aggregate across users because each user's data is unique. This means every participant is both their own corpus generator and their own judge — a methodological challenge that inflates variance and limits statistical power.

**Ecological validity of tasks.** Laboratory "find this file" tasks do not capture the full complexity of real retrieval, which involves uncertain memory, partial cues, time pressure, and interleaved activity. Participants who created a task-specific file five minutes ago will find it trivially; participants trying to locate a file they haven't touched in 18 months face a genuinely hard problem.

**Tool contamination.** Once a participant starts using an AI organizer, their file system changes. Measuring retrieval before and after reorganization is confounded by learning effects, familiarity with the new structure, and the AI's ongoing assistance during the measurement period.

---

## 5. Synthetic Benchmark Approaches

The pseudo-desktop methodology offers a principled starting point for a synthetic benchmark. Key design choices include:

**Corpus construction.** A synthetic personal file system should reflect realistic statistical properties: file type distribution (documents, images, code, PDFs, spreadsheets, email exports), naming convention diversity (systematic naming vs. ad hoc), folder depth distribution (typically 3–5 levels), and temporal spread of modification dates. The "What's in People's Digital File Collections?" study (arXiv:2402.06421) provides empirical data on 348 real personal collections that can anchor these statistics.

**Query generation.** Known-item queries can be simulated by selecting a target file and generating a natural-language description of it (its approximate content, when it was created, what project it relates to) without using its exact filename. This simulates the experience of trying to remember where you put something.

**Ground truth for organization.** Ground truth for organizational quality requires an indirect definition. One approach: define a set of canonical tasks (find a document related to topic X from project Y created around month Z), then measure success rate and navigation depth across multiple organizational schemes applied to the same corpus. An organizational scheme is "better" if it supports higher task success at lower navigation depth.

---

## 6. CHI and IUI Evaluation Standards

Papers at CHI and IUI evaluating file management tools consistently use a combination of:

- **Task completion rate and time** as primary objective metrics. Tasks involve finding a specific file given a verbal or written description.
- **Navigation depth** (number of folder traversals before finding target or giving up).
- **Search fallback rate** (proportion of tasks resolved by keyword search rather than navigation, indicating organizational failure).
- **NASA-TLX** (six dimensions: mental demand, physical demand, temporal demand, performance, effort, frustration) for subjective workload assessment.
- **Likert-scale satisfaction** items about the organization's intuitiveness, trustworthiness, and match to the user's own mental model.
- **Think-aloud protocols** for qualitative insight into navigation strategies and confusion points.

The standard sample size in this literature is small (N=12–20 participants), which limits statistical power. IUI reviewers increasingly expect within-subjects designs with counterbalancing to control for learning effects, and effect sizes reported alongside p-values.

---

## 7. Proposed Novel Benchmark: FileNav

Based on the gap analysis above, Curator should develop and publish a benchmark — provisionally called **FileNav** — that addresses the missing dimensions. The following describes its design.

### 7.1 Design Goals

1. **Replicable.** Other researchers can re-run the exact benchmark without access to participants' private data.
2. **Ecologically valid.** Tasks reflect real retrieval scenarios, not artificial "find this contrived file" tasks.
3. **Organization-sensitive.** The benchmark measures organizational quality, not just retrieval capability of a search engine applied post-hoc.
4. **Privacy-preserving by design.** No real personal files are collected or shared.

### 7.2 Corpus: Synthetic Persona File Systems

FileNav provides a set of synthetic personal file systems, each representing a realistic user persona (e.g., "graduate student in biology," "freelance graphic designer," "small business owner"). Each persona's file system is constructed to have:

- 1,500–3,000 files across 8–15 top-level categories
- Realistic naming (mix of systematic and ad hoc names)
- Temporal spread across 3–5 simulated years
- Cross-cutting projects that span multiple natural categories (the hardest organizational challenge)
- Deliberate "mess": orphaned files, duplicate versions, misplaced items

The personas and their file systems are publicly released. This allows any research group to apply their organizational algorithm to the same input and report results against a common baseline.

### 7.3 Task Battery

FileNav defines three task classes:

**Class A: Known-item retrieval.** "Find the presentation you used at the conference in March of last year." The file exists; the query provides temporal and semantic cues but not the exact filename. Success is binary (found / not found within 5 minutes or 10 navigation steps). Primary metrics: success rate, navigation depth, time-to-find.

**Class B: Category navigation.** "Find all files related to your tax documentation from last year." This requires locating a coherent category that may be spread across multiple folders under different AI-proposed structures. Metric: precision and recall of the retrieved set, navigation depth to achieve 80% recall.

**Class C: Unfamiliar retrieval.** Participants retrieve files from a persona that is not their own — they must use the organizational structure as a new user would. This tests the legibility of the organization to someone without prior knowledge. Metric: success rate and time-to-find for Class A tasks.

### 7.4 Evaluation Protocol

**Participants.** 20–30 participants recruited through university channels. Within-subjects design: each participant navigates multiple organizational schemes applied to the same corpus, with order counterbalanced using a Latin square. Washout period (one week) between conditions to reduce memory contamination.

**Conditions.** At minimum three conditions: (1) flat dump (no organization, all files in one folder — baseline floor), (2) a simple rule-based organizer (file type + year), (3) Curator's AI-proposed organization. Ideally a fourth condition: the user's own manual organization of the same synthetic corpus (ceiling estimate).

**Measures collected.** Task success rate; time-to-find; navigation depth; search fallback rate; NASA-TLX at end of each condition; Likert items on intuitiveness, trustworthiness, and sense of control; post-study semi-structured interview (15 minutes).

**Infrastructure.** Tasks are administered through a custom macOS application that presents the file system in a Finder-like interface with all metadata preserved but with system search disabled (to force navigation). Task descriptions are shown in a sidebar. The application logs all navigation events automatically. Participants interact with synthetic files (realistic names and content stubs) rather than their own data.

### 7.5 Metrics Summary

| Metric | Task Class | Type |
|--------|-----------|------|
| Success rate | A, B, C | Objective |
| Time-to-find (seconds) | A, C | Objective |
| Navigation depth (steps) | A, B, C | Objective |
| Search fallback rate | A, B | Objective |
| Recall@10steps | B | Objective |
| NASA-TLX composite | All | Subjective |
| Intuitiveness (Likert 1–7) | All | Subjective |
| Trustworthiness (Likert 1–7) | All | Subjective |
| Sense of control (Likert 1–7) | All | Subjective |

### 7.6 Novelty and Contribution

FileNav would be the first benchmark to: (a) use a shared, publicly released synthetic corpus that any group can apply their algorithm to; (b) measure organizational quality through user navigation tasks rather than search-engine retrieval; (c) include a "foreign user" condition (Class C) that tests legibility to someone other than the organizing agent; and (d) combine objective navigation telemetry with validated subjective workload measures in a within-subjects design over multiple organizational schemes. Publishing the benchmark corpus and evaluation software as open-source artifacts alongside an IUI 2027 paper would constitute a genuine infrastructure contribution to the PIM research community.

---

## 8. Privacy-Preserving Public Datasets

No publicly available dataset of anonymized real personal file system access patterns currently exists. The closest resources are:

- **NTCIR Lifelog datasets** (released to registered participants under data use agreement): multimodal lifelogs from a small number of participants, not file-system specific.
- **The 348-collection study** (arXiv:2402.06421): statistical summaries of real personal collections (file type distributions, folder depth distributions) without file content — useful for calibrating synthetic corpus generation but not usable as a retrieval benchmark.
- **Pseudo-desktop methodology papers**: describe how to generate synthetic corpora statistically matched to real desktops, but do not release the real data that served as the statistical target.

The absence of a real-world dataset is not a solvable problem in the short term given privacy law constraints (GDPR, CCPA). The synthetic-corpus approach is therefore not merely a convenience — it is the only viable path to a replicable, shareable benchmark.

---

## 9. Positioning FileNav for IUI 2027

IUI 2027 will expect a rigorous empirical evaluation. The following positioning is recommended:

- Frame FileNav as both a **methodology paper** and a **tool paper** (open-source benchmark suite).
- Compare Curator against at least one commercial baseline (e.g., Hazel rule-based organization, or Sortio AI organizer) to establish competitive context.
- Report effect sizes (Cohen's d) and confidence intervals, not just p-values.
- Include a qualitative analysis of failure cases: when does Curator's organization confuse users, and why?
- Acknowledge the ecological validity limitation explicitly: synthetic corpora match real file systems statistically but cannot capture personal semantic meaning. Frame this as a known limitation and propose future work using the PIRE living-lab approach for longitudinal ecological validity studies.

The benchmark corpus, evaluation application, and analysis scripts should be released on GitHub under a permissive license at paper submission time, with the data hosted on a persistent repository (Zenodo or OSF) to ensure long-term availability.

---

## Sources

- [HippoCamp: Benchmarking Contextual Agents on Personal Computers](https://arxiv.org/abs/2604.01221)
- [FileGram: Grounding Agent Personalization in File-System Behavioral Traces](https://arxiv.org/abs/2604.04901)
- [Retrieval Experiments Using Pseudo-Desktop Collections](https://www.researchgate.net/publication/220269855_Retrieval_experiments_using_pseudo-desktop_collections)
- [Building a Desktop Search Test-Bed](https://www.researchgate.net/publication/221397707_Building_a_Desktop_Search_Test-Bed)
- [Evaluating Personal Information Retrieval (PIRE)](https://link.springer.com/chapter/10.1007/978-3-642-28997-2_60)
- [NTCIR Lifelog: The First Test Collection for Lifelog Research](https://dl.acm.org/doi/10.1145/2911451.2914680)
- [NTCIR19-Lifelog Task](https://lifelog-ntcir-project-bd9eae.gitlab.io/)
- [What's in People's Digital File Collections?](https://arxiv.org/pdf/2402.06421)
- [Bergman: The User-Subjective Approach to Personal Information Management](https://asistdl.onlinelibrary.wiley.com/doi/abs/10.1002/asi.10283)
- [Bergman: Shared Files — The Retrieval Perspective](https://asistdl.onlinelibrary.wiley.com/doi/abs/10.1002/asi.23147)
- [Teevan: Supporting Finding and Re-Finding Through Personalization (doctoral thesis)](http://teevan.org/publications/theses/doctoralthesis.pdf)
- [Beagle++: Leveraging Personal Metadata for Desktop Search](https://www.sciencedirect.com/science/article/abs/pii/S1570826809000754)
- [PersonaBench: Evaluating AI Models on Understanding Personal Information](https://arxiv.org/html/2502.20616)
- [TREC iKAT 2024 Overview](https://trec.nist.gov/pubs/trec33/papers/Overview_ikat.pdf)
- [NASA-TLX](https://en.wikipedia.org/wiki/NASA-TLX)
- [Living Labs for Information Retrieval Evaluation (LL4IR) CLEF 2015](https://www.researchgate.net/publication/279206579_Overview_of_the_Living_Labs_for_Information_Retrieval_Evaluation_LL4IR_CLEF_Lab_2015)
