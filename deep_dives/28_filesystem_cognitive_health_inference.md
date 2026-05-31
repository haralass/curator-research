# Inferring Cognitive and Mental Health States from Personal Filesystem Behavior Patterns

**Research document for the Curator project**
Date: 2026-05-31
Status: Exploratory — hypothesis formation and prior work survey

---

## Abstract

Personal file systems generate continuous, ambient behavioral traces: files created, modified, accessed, searched for and not found, clusters that grow or collapse, topics that go quiet. This document surveys the literature on passive sensing for mental health inference, synthesizes what is known about digital behavioral biomarkers, identifies what is unique about filesystem signals specifically, evaluates which signals available in Curator are most theoretically promising, and outlines both an ethical framework and a research validation agenda. The central argument is that filesystem behavior offers a qualitatively distinct class of ambient signal — one rooted in knowledge-seeking, sensemaking, and information structuring — that is largely absent from the smartphone/wearable literature and merits its own empirical program.

---

## 1. Framing the Hypothesis

The hypothesis has two parts:

**Part A (Signal existence).** Observable, non-self-reported filesystem behaviors — file creation rates, cluster entropy, FLRS decay trajectories, antilibrary growth, search failure rates — correlate reliably with internally experienced cognitive/affective states including burnout, depression, anxiety, cognitive overload, and life-stage transitions.

**Part B (Signal specificity).** These correlations are stable enough, and distinct enough from noise and individual style variation, to discriminate meaningfully between states across time within the same person (intra-individual design), even if cross-person generalization is weaker.

Part B is the more tractable scientific target. The literature surveyed below consistently shows that within-person longitudinal modeling outperforms population-level classifiers for digital phenotyping, and Curator's architecture of per-user FLRS models and evolving clusters is well-suited for this framing.

---

## 2. Prior Work Landscape

### 2.1 The Passive Sensing Paradigm for Mental Health

The foundational framework — using continuously collected device data as a proxy for mental state — was established around 2010–2014 and is now a mature subfield. The landmark demonstration is the **StudentLife study** (Wang et al., Dartmouth, UbiComp 2014; extended in MobileHealth 2017). Over a single academic term, 48 undergraduate students were monitored via smartphone sensors (microphone activity for conversation inference, accelerometer for activity, GPS for location, sleep via phone dark/unlock cycles). The study found:

- Significant negative correlation between sleep duration and PHQ-9 depression score
- Significant associations between conversation activity, physical movement, and flourishing/loneliness scales
- A clear term-lifecycle pattern: students start the semester high on positive affect and activity, then as workload mounts, stress rises and all positive behavioral indicators fall
- Linear regression with lasso regularization predicted cumulative GPA from sensor features with reasonable accuracy

**Citation:** Wang, R., et al. "StudentLife: Assessing mental health, academic performance and behavioral trends of college students using smartphones." ACM UbiComp 2014. DOI: 10.1145/2632048.2632054

**Signal used:** GPS (mobility), microphone (conversation), accelerometer (activity), phone usage patterns
**What was inferred:** Depression (PHQ-9), stress, flourishing, loneliness, GPA
**Validation:** Ground truth self-report scales; within-study correlation
**Relevance to Curator:** The term-lifecycle pattern is analogous to what we might expect in Curator's cluster evolution: semester start = new topic clusters forming rapidly; exam period = cluster entropy collapses into narrow high-churn areas; end of semester = broad re-engagement or total withdrawal

---

### 2.2 Large-Scale Validated Studies

**Mohr group longitudinal study (n=1013, npj Mental Health Research 2023).** Used the LifeSense app to passively collect GPS, app usage, and device interaction logs across 16 weeks. Key findings relevant here:
- Spending more time at home relative to personal baseline was an early predictor of PHQ-8 (depression) severity — not absolute time at home, but deviation from individual baseline
- Circadian regularity of movement was proximally associated with PHQ-8 but did not predict it, suggesting it is a concurrent rather than leading indicator
- Anxiety (GAD-7) and depression required different temporal feature windows

**Citation:** "Differential temporal utility of passively sensed smartphone features for depression and anxiety symptom prediction: a longitudinal cohort study." npj Mental Health Research, 2023. DOI: 10.1038/s44184-023-00041-y

**Signal used:** GPS mobility, app/device use, communication logs
**What was inferred:** PHQ-8 depression, GAD-7 anxiety
**Validation:** Longitudinal cohort; hierarchical linear regression
**Key methodological insight:** Deviations from individual baselines outperform population-level thresholds. This is the most important finding for Curator's design: we should model deviation from each user's own FLRS patterns, not compare across users.

---

### 2.3 Keystroke Dynamics as a Validated Desktop-Native Signal

The most directly relevant desktop-behavior validated signal in the literature is keystroke dynamics.

**BiAffect Study (University of Illinois Chicago).** Deployed a custom iOS keyboard (BiAffect) that passively collected keystroke metadata: timing between keystrokes (dwell time, flight time), backspace frequency, autocorrect usage, accelerometer tremor. Key findings:
- During depressive episodes: slower typing, longer inter-key intervals, shorter message length, less overall phone engagement
- During manic episodes: faster typing, more errors (higher backspace rate), more use of autocorrect, accelerometer-detected tremor
- Mood disturbance severity predicted from keystroke metadata alone with correlations r ≈ 0.4–0.6 against clinical mood ratings in initial 9-subject study
- A 2025 follow-up (Frontiers in Psychiatry) extended to cognitive function assessment in bipolar disorder

**Citation:** Zulueta, J., et al. "Predicting Mood Disturbance Severity with Mobile Phone Keystroke Metadata: A BiAffect Digital Phenotyping Study." JMIR 2018. DOI: 10.2196/jmir.9775; PMC6076371

**Signal used:** Keystroke timing, error rate, session length, accelerometer
**What was inferred:** Manic vs. depressive mood state severity (bipolar disorder)
**Validation:** Clinical mood ratings; n=9 initial, expanded subsequently
**Accuracy:** Correlation with clinical ratings r ≈ 0.4–0.6

**Nature Scientific Reports 2024 (Robust remote detection of depressive tendency via keystroke dynamics).** Larger study using desktop/smartphone keyboard interaction, finding that:
- Slower mean typing speed, increased inter-key latency variability, and higher backspace rate are reliable depressive indicators
- Feature engineering with machine learning achieved AUC ≈ 0.80 for depressive tendency classification

**Citation:** "Robust remote detection of depressive tendency based on keystroke dynamics and behavioural characteristics." Scientific Reports, 2024. DOI: 10.1038/s41598-024-78489-x

**Loneliness and social isolation study via keystroke dynamics (PMC 2024).** Found distinct daily keystroke patterns associated with loneliness ratings — reduced typing during typical social hours, more late-night typing sessions — suggesting isolation leaves temporal signatures in device behavior.

**Citation:** "Investigation of daily patterns for smartphone keystroke dynamics based on loneliness and social isolation." PMC 2024. PMID: PMC10874350

---

### 2.4 Multitasking and Context-Switching as Cognitive Load Signals

**Detecting Multitasking and Negative Routines from Computer Logs (Springer, 2016).** Proposed using window-switching frequency in computer logs as a multitasking rate indicator, combined with heart rate variability (HRV) as a relax rate indicator. Computer activity logging software (e.g., KidLogger, Manictime) generates timestamped records whenever a user opens or switches between windows, including application name, URL, and duration. Found a positive relationship between window-switching frequency and experienced stress.

**Citation:** "Detecting Multitasking Work and Negative Routines from Computer Logs." In: Proceedings of Intelligent Decision Technologies, Springer 2016. DOI: 10.1007/978-3-319-40397-7_52

**Signal used:** Window-switching frequency (from system logs), HRV
**What was inferred:** Multitasking rate; stress level
**Validation:** Case study (n=1 across 6 days); preliminary but methodologically instructive

**CHIWORK 2025 "Are Six Minutes of Focus Enough?"** Found in a workplace study that the median uninterrupted focus window is approximately 6 minutes, and that multitasking frequency significantly predicted self-reported cognitive overload.

**Citation:** "Are Six Minutes of Focus Enough? An Exploratory Study of Multitasking Patterns in Workplace Environments." CHIWork 2025. DOI: 10.1145/3729176.3729198

---

### 2.5 Burnout Detection from Digital Behavioral Patterns

**Burnout and the Quantified Workplace: Tensions around Personal Sensing Interventions for Stress in Resident Physicians (CSCW 2022).** Adler, Tseng, and Choudhury (Cornell Tech) conducted formative interviews and design provocations with resident physicians and their supervisors. Key findings:
- Personal sensing data (sleep, activity, work hours) could help detect burnout patterns before residents became acutely unwell
- Residents wanted *private* sensing with self-controlled visibility — they did not trust institutional access to their behavioral data
- A critical design tension emerged between the utility of burnout detection and the power asymmetry of who sees the data
- Autonomic-access data raised the most concern; aggregate/anonymized data less so

**Citation:** Adler, D., Tseng, E., et al. "Burnout and the Quantified Workplace: Tensions around Personal Sensing Interventions for Stress in Resident Physicians." Proceedings of the ACM on Human-Computer Interaction, Vol. 6, CSCW2, Article 430, 2022. DOI: 10.1145/3555531; PMC9879386

**Signal used:** Sleep duration, physical activity, work hours (from smartphone)
**What was inferred:** Burnout risk
**Key finding for Curator:** The privacy design principles that emerged here — user-owned data, self-controlled visibility, opt-in interpretation — should directly inform Curator's architecture

**Passive AI Detection of Stress and Burnout Among Frontline Workers (MDPI, 2025).** Reviewed passive detection approaches; found multimodal approaches (combining physiological, behavioral, and communication patterns) outperform unimodal ones. Behavioral digital traces (communication cadence, work-hour patterns) showed moderate predictive value for burnout independently.

**Citation:** "Passive AI Detection of Stress and Burnout Among Frontline Workers." MDPI Healthcare, 2025. DOI: 10.3390/healthcare15110373; PMC12655262

---

### 2.6 Digital Phenotyping: Systematic Reviews (2024)

**Digital Phenotyping for Stress, Anxiety, and Mild Depression: Systematic Literature Review (JMIR mHealth 2024).** Reviewed 40 studies (36 papers). Key consolidated findings:
- Travel behavior, physical activity patterns, sleep timing, social interaction frequency, and phone usage patterns were reliably associated with stress, anxiety, and mild depression
- However, a critical limitation is that these patterns are not exclusive to specific conditions: high mobility can indicate either healthy activity or anxiety-driven hyperactivity
- Context matters enormously for interpretation
- Studies were predominantly short-term (weeks to months), with poor replicability and few external validations

**Citation:** "Digital Phenotyping for Stress, Anxiety, and Mild Depression: Systematic Literature Review." JMIR mHealth and uHealth, 2024. DOI: 10.2196/40689; PMC11157179

**Large-scale digital phenotyping identifying depression and anxiety (arXiv 2024).** Demonstrated that fine-grained behavioral patterns from passively collected digital data can distinguish depression and anxiety at population scale when sample sizes are large (n in the thousands), though individual-level accuracy remains considerably more challenging.

**Citation:** "Large-scale digital phenotyping: identifying depression and anxiety." arXiv:2409.16339, 2024.

---

### 2.7 Rare Life Event and Behavioral Drift Detection

**Rare Life Event Detection via Mobile Sensing Using Multi-Task Learning (CHIL 2023).** Used data from 126 information workers across 10,106 days with 198 rare life events (events occurring <2% of observation time). Proposed a multi-task framework combining unsupervised autoencoder (to capture irregular behavioral patterns) with an auxiliary sequence predictor (to identify transitions in workplace performance). This is directly analogous to the problem of detecting life-stage transitions in Curator (e.g., career change, relationship change, health crisis) from shifts in file cluster activity.

**Citation:** Pillai, A., et al. "Rare Life Event Detection via Mobile Sensing Using Multi-Task Learning." Proceedings of CHIL 2023 (PMLR Vol. 209). DOI: proceedings.mlr.press/v209/pillai23a.html

**Signal used:** Smartphone behavioral features across multiple modalities
**What was inferred:** Rare life events (job change, bereavement, health crisis, major travel)
**Validation:** 126-worker dataset over ~80 days each; multi-task learning
**Relevance:** The problem formulation maps almost exactly onto detecting cluster emergence/collapse events in Curator as potential life-stage transition signals

**Behavioral Drift Detection for Mental Health (MDPI Electronics 2025).** Proposed an adaptive individual-baseline framework for detecting behavioral drift from wearable/smartphone streams. Key methodological insight: "personalised baselines" defined from first 30 days of observation, with deviation scores calculated against this rolling window. Sustained deviations (>7 days) were better predictors than acute ones.

**Citation:** "From Patterns to Deviations: Detecting Behavioural Drift for Mental Health Monitoring Using Smartphone and Wearable Data." MDPI Electronics, 2025. DOI: 10.3390/electronics15040885

---

### 2.8 File Management Research as a Research Domain

**Dinneen & Julien (2020): The Ubiquitous Digital File: A Review of File Management Research.** The first dedicated systematic review of file management research, synthesizing 230+ publications. Key findings:
- Most file management research concerns storage, organization, retrieval, and sharing — it does not examine what file management behavior reveals *about the user*
- Three research motivations: understanding behavior patterns; understanding factors that determine behavior; improving UX
- Significant frustration, anxiety, and affective load are associated with file re-finding failures and misfiling — described as "a complex and intense affective experience characterized by anxiety and frustration"
- There is a near-complete gap in the literature on using file management behavior as an inferential signal for user cognitive/emotional state

**Citation:** Dinneen, J.D., & Julien, C.A. "The ubiquitous digital file: A review of file management research." Journal of the Association for Information Science and Technology, 71(E1-E32), 2020. DOI: 10.1002/asi.24222

**The gap this identifies:** As of 2020, no published work had systematically examined file behavior as a passive sensor for cognitive state. The field studied file behavior as an outcome (how do users manage files?), not as a predictor (what does file behavior predict about users?). This gap remains substantially open.

---

### 2.9 FileGram: File-System Behavioral Traces for Agent Personalization (arXiv 2026)

**FileGram (arXiv:2604.04901, April 2026).** A framework that grounds AI agent memory in file-system behavioral traces rather than conversation logs. Three components: FileGramEngine (simulates realistic file-system workflows to generate training data), FileGramBench (benchmark for evaluating memory systems on profile reconstruction, persona drift detection, and trace disentanglement), and FileGramOS (memory architecture building user profiles from atomic file operations and content deltas, organized into procedural, semantic, and episodic memory channels).

FileGram demonstrates that file-system behavioral traces are sufficiently rich and structured to support sophisticated user modeling — but focuses entirely on personalization for task performance, not cognitive/emotional state inference. It shows, however, that the fundamental premise is technically tractable: filesystem operations do encode persistent, interpretable behavioral patterns.

**Citation:** "FileGram: Grounding Agent Personalization in File-System Behavioral Traces." arXiv:2604.04901, 2026.

**Gap relative to Curator's hypothesis:** FileGram uses filesystem behavior to infer task preferences and work style. The Curator hypothesis extends this to inferring affective and cognitive states — a step that requires grounding filesystem signals in clinical/psychological theory and validating against ground-truth wellbeing measures.

---

### 2.10 Personal Information Management, Stress, and Cognitive Overload

The PIM literature (Jones 2008; Dinneen 2020) identifies information fragmentation, re-finding failures, and loss of organizing structure as primary stressors in knowledge work. Key relevant findings:

- 60–80% of web page visits are re-accesses (re-finding behavior dominates navigation)
- 40% of web searches are aimed at re-finding previously seen information
- Information fragmentation across platforms (email, local files, cloud, note apps) is a primary source of cognitive overhead and frustration
- Gaps between a user's ideal PIM state and their actual state produce "a complex and intense affective experience characterized by anxiety and frustration"

The Spotlight search rate (searches per unit time) and especially the *failed retrieval rate* (searches that end without file opening) are thus not merely usability metrics — they are potentially sensitive indicators of the degree to which a user is struggling to re-locate their own cognitive artifacts. A sustained increase in failed retrievals, particularly within a previously well-organized cluster, could signal a deterioration in that cluster's internal coherence (entropy increase) that parallels cognitive difficulty.

**Citation:** Keeping Found Things Found: The Study and Practice of Personal Information Management. Jones, W. (University of Washington / Cambridge University Press, 2008).

---

## 3. What Is Unique About Filesystem Signals vs. Other Digital Biomarkers

All current digital biomarker research targets *behavioral rhythms* (sleep/wake, activity, mobility, communication timing) or *motor signatures* (keystroke dynamics, gait via accelerometer). Filesystem signals differ in three fundamental ways:

### 3.1 Content as Context

Filesystem signals are inherently semantic: the *topics* of files being created, accessed, or abandoned are visible (at least at the cluster level). No other passive sensing modality provides this. A user who begins creating many files tagged with medical terminology, legal documents, or financial planning material is potentially signaling a life event (illness, legal dispute, financial stress) that is invisible to accelerometer or GPS data. This contextual richness is double-edged: it is powerful for inference, but raises far more serious privacy concerns than step-count data.

### 3.2 Temporal Structure as Intentionality Signal

File creation and access reflects *purposive information behavior* — every file-system interaction involves some degree of intentional knowledge-seeking or knowledge-structuring. This is categorically different from passively recorded motion or communication logs. Patterns in purposive behavior encode something about the user's cognitive agenda: what they are trying to understand, produce, or remember. Disruption of that structure (cluster collapse, antilibrary growth without access, cessation of file creation in a previously active project cluster) may signal motivational or executive dysfunction with more direct face validity than, say, GPS circle of wandering.

### 3.3 The Forgetting Signal: FLRS Decay

No other digital phenotyping approach has a direct analog of a within-system *forgetting model*. Curator's FLRS (File-Level Relevance Score) scores encode how recently and frequently each file has been accessed relative to its expected decay curve. If a file's FLRS decays faster than expected (user stops accessing files they normally revisit), or slower (user keeps returning to items they would normally have internalized), this is a novel signal class. The *shape* of decay trajectories across a cluster — do they decay uniformly (healthy consolidation) or erratically (disrupted information routine)? — is not available from any other sensing modality. This is arguably Curator's most distinctive potential contribution to the biomarker landscape.

### 3.4 What Filesystem Signals Cannot Do

- They do not capture physiological arousal (stress without behavioral output)
- They are subject to long stretches of non-activity (weekends, vacations) that may be hard to distinguish from depressive withdrawal
- Confounders include job type (a researcher will always have high file creation; an artist may have low text-file creation), platform diversity (if users work across multiple devices, the signal from any one system is incomplete), and external constraints (organizational requirements may force file-system patterns independent of personal state)
- Individual style variation is high — some people are prolific filers; others are inbox-zero document hoarders. Cross-person normalization is very difficult; within-person modeling is far more tractable.

---

## 4. Curator-Specific Signal Analysis

The following evaluates each signal available in Curator for its plausibility as a cognitive/emotional state indicator, drawing on the prior work above.

### Signal 1: File Creation Rate Per Cluster
**Plausible state associations:**
- Rapid acceleration → anxiety-driven overproduction, manic episode (cf. BiAffect typing speed increase in mania), new project excitement
- Sustained deceleration followed by plateau → burnout, depressive withdrawal, project completion
- Complete cessation in a previously active cluster → possible abandonment (life event, prioritization shift, avoidance)

**Best evidence basis:** StudentLife term-lifecycle (behavioral activity declines with stress); BiAffect (production speed changes in mood episodes)
**Confounders:** External deadlines, collaboration (files created by others entering the system), switching to different tool (e.g., moving from local files to Notion)
**Feasibility:** High — this is directly observable from file metadata

### Signal 2: FLRS Decay Trajectories
**Plausible state associations:**
- Accelerated decay (user stops revisiting files earlier than expected) → loss of interest, depressive anhedonia, cognitive avoidance
- Flattened decay (user keeps returning to same files repeatedly) → anxiety-driven rumination, inability to move forward, cognitive perseveration
- Irregular/noisy decay patterns (erratic revisitation schedule) → disrupted cognitive routine, fragmented attention, high context-switching load

**Best evidence basis:** No direct prior work — this is Curator's most novel signal. The theoretical basis comes from spaced repetition / forgetting curve research: healthy knowledge consolidation follows a predictable decay pattern; disruptions to that pattern may indicate underlying cognitive disruption.
**Priority:** High — this is the most theoretically novel contribution

### Signal 3: Cluster Entropy Over Time
**Cluster entropy** = inverse of within-cluster cohesion; high entropy means files in a cluster span many disparate topics; low entropy means tight thematic focus.

**Plausible state associations:**
- Rising cluster entropy → cognitive scatter, inability to focus, overload (too many things being worked on simultaneously)
- Collapsing entropy → hypernarrowing of attention (burnout tunnel vision, anxiety fixation)
- Oscillating entropy → natural project rhythm vs. disrupted alternation

**Best evidence basis:** Window-switching frequency as cognitive load signal (Springer 2016); multitasking rate and stress (CHIWork 2025); entropy measures computed on digital behavioral features correlating with depression severity (PMC 2019)
**Feasibility:** Computable from cluster state; requires defining a good entropy metric for topic cohesion

### Signal 4: Work/Personal File Ratio
**Plausible state associations:**
- Ratio shifting toward work → increasing work intensity, approaching burnout, loss of work-life boundary (cf. after-hours work statistics showing stress correlation)
- Ratio shifting toward personal → recovery, life-event preoccupation (health, relationship, housing), disengagement from work
- Ratio collapse (both categories going quiet) → potential depressive withdrawal across domains

**Best evidence basis:** Temporal patterns of after-hours work associated with stress (occupational health literature); work-life balance monitoring from digital traces
**Confounders:** Role changes (new job = legitimately higher work ratio), cultural context

### Signal 5: Spotlight Search Rate / Failed Retrieval Rate
**Plausible state associations:**
- Rising search rate → deteriorating spatial/categorical memory for file locations; increasing cognitive load; possible sign of organizational breakdown
- Rising *failed* retrieval rate (search executed, no file opened) → either the file doesn't exist (antilibrary thinking) or the user cannot locate a file they believe they have; both are signals of organizational stress

**Best evidence basis:** PIM literature: re-finding failures as primary source of frustration/anxiety; Dinneen (2020) on findability as affective experience. No direct clinical validation for this specific signal.
**Feasibility:** Requires instrumenting Spotlight interactions or building an in-app search log

### Signal 6: Antilibrary Growth Rate
**Plausible state associations:**
- Rapid antilibrary growth → aspirational/anxious information hoarding; a response to overwhelm ("I should read this") without the capacity to process; anxiety-driven information collection behavior
- Antilibrary growth without corresponding FLRS access events → aspirational collection that never becomes active knowledge; possible indicator of executive dysfunction (difficulty initiating) or information avoidance
- Antilibrary shrinkage (deletion or access of previously hoarded items) → cognitive recovery, active processing, increased capacity

**Best evidence basis:** Information anxiety literature; tsundoku (book hoarding) as cultural concept linked to anxiety; Taleb/Eco antilibrary concept. No direct clinical validation.
**Novelty:** High — this signal is unique to Curator-style PIM systems. The ratio of "collected but never accessed" files to "collected and accessed" files is a new behavioral indicator.

### Signal 7: Temporal Co-occurrence Patterns
**Plausible state associations:**
- Late-night file activity clustering → disrupted sleep/wake cycle (one of the most validated depression signals); anxiety-driven overwork; circadian rhythm disruption
- Compressed daily activity windows (all activity in narrow AM hours, nothing after) → withdrawal, depressive behavior, possible avoidance

**Best evidence basis:** Mohr group (2023): circadian movement regularity associated with depression; BiAffect (loneliness patterns in temporal keystroke distribution); StudentLife (sleep duration correlates with depression)
**Feasibility:** Directly inferrable from file modification timestamps

---

## 5. Composite Signal Architecture

Rather than treating each signal in isolation, the most defensible inferential approach combines signals into a *composite behavioral fingerprint* compared against an individual's rolling baseline. The following framework is proposed:

```
State signal = f(
  baseline_deviation(file_creation_rate),
  FLRS_decay_shape_change,
  cluster_entropy_trajectory,
  work_personal_ratio_shift,
  failed_retrieval_rate_change,
  antilibrary_growth_vs_access_ratio,
  temporal_distribution_shift
)
```

Key design principles from the literature:
1. **Within-person modeling only** (Mohr 2023): compare against individual's own baseline, not population norms
2. **Sustained deviations matter more than acute spikes** (Behavioral Drift 2025): require >7 days of consistent deviation before flagging
3. **Multi-signal confluence** (systematic review 2024): no single signal is specific enough; look for co-occurring shifts
4. **Temporal leading vs. lagging indicators**: FLRS decay shape and cluster entropy may lead (precede state change); work/personal ratio and search failure rate may be concurrent

---

## 6. Burnout as the Highest-Priority Target State

Of all candidate states, **burnout** is the most tractable initial target for Curator for the following reasons:

1. **Gradual onset**: Burnout develops over weeks to months, giving filesystem signals time to accumulate before the state becomes clinically significant
2. **Multi-domain behavioral signature**: Burnout affects work output (file creation), information seeking (search patterns), and personal life engagement (work/personal ratio) simultaneously — multiple Curator signals should move together
3. **High prevalence in Curator's likely user base**: Knowledge workers, students, and researchers (people who maintain complex personal file systems) are among the highest burnout-risk populations
4. **Partial clinical validation of analog signals**: The CSCW 2022 burnout/quantified-workplace study demonstrated both the feasibility and the user demand for passive burnout sensing
5. **Clear behavioral theory**: Maslach's burnout model predicts progressive disengagement (emotional exhaustion → cynicism/depersonalization → reduced efficacy) — each stage has a plausible filesystem analog:
   - Exhaustion phase → reduced file creation, flattened FLRS decay
   - Depersonalization phase → work/personal ratio collapse, antilibrary growth stalls
   - Reduced efficacy phase → rising search failure rate, cluster entropy increase (scattered, unfocused work)

---

## 7. Ethical Framework

### 7.1 The Inferential Privacy Problem

The core ethical challenge is not data security (encrypting files) but *inferential privacy*: the risk that non-sensitive behavioral traces (file access patterns) enable inference of highly sensitive attributes (mental health state). This is structurally identical to the problem identified in the broader AI ethics literature — race can be inferred from ZIP code; mental health can be inferred from keyboard timing — but is more acute here because:

- Mental health status is among the most sensitive personal attributes
- Inferences may be wrong (false positives can cause anxiety, stigma, inappropriate intervention)
- The user may not realize that their file behavior is being interpreted at this level
- Any institutional access to these inferences creates power asymmetries (cf. the CSCW 2022 resident physician study)

**Citation:** Mittelstadt, B., et al. "Data mining for health: staking out the ethical territory of digital phenotyping." npj Digital Medicine, 2019. DOI: 10.1038/s41746-019-0104-8; PMC6550156

**Citation:** "Predictive privacy: towards an applied ethics of data analytics." Ethics and Information Technology, 2021. DOI: 10.1007/s10676-021-09606-x

### 7.2 The Accuracy Gap Problem

From the 2024 systematic review and from the CSCW 2022 paper: most digital phenotyping studies are small, short-term, and fail to externally validate. AUCs of 0.75–0.85 are commonly reported, but these are typically within-sample results. Real-world deployment accuracy is substantially lower. A model that flags 20% false positives for "burnout risk" causes meaningful harm if that information is surfaced to the user in a way that induces anxiety or pathologizes normal variation.

Design implication: **Never show probabilistic state inferences as diagnoses.** If cognitive state signals are used, they should be surfaced as ambient reflection prompts ("You've been working a lot on your thesis this week — would a review of your Health cluster be useful?") rather than assessments ("Your behavioral patterns suggest you may be experiencing burnout").

### 7.3 Required Ethical Constraints for Curator

Drawing on the frameworks of Mittelstadt (2019), Nebeker et al. (2017 — ethical perspectives on digital mental health), and the CSCW 2022 findings, the following constraints are proposed:

**Consent architecture:**
- All cognitive state inference must be opt-in at a granular level, not bundled into general terms of service
- The user must be able to see exactly what signals are being computed and why
- Consent should be treated as an ongoing relationship: the user can withdraw specific inference modules at any time without losing access to core features
- No paternalistic "we've noticed you might be struggling" notifications without the user explicitly requesting this kind of reflection

**Data sovereignty:**
- All inference must be performed on-device; no behavioral fingerprint data should leave the device
- Inferred states must never be transmitted to third parties, employers, or even cloud backup systems without explicit per-inference consent
- FLRS data and cluster evolution data should be treated as clinically sensitive even if their surface form looks like file metadata

**Transparency and legibility:**
- The system must be able to explain, in plain language, what behavioral patterns triggered any inference or reflection prompt
- Users must be able to see their own signal trajectories (this doubles as a therapeutic reflection affordance)

**Epistemic humility in UI:**
- Inferences must be framed as patterns, not diagnoses
- The system should always offer an alternative mundane explanation for any pattern ("You've had many late-night file sessions this week — are you in a crunch period, or is something else going on?")
- Avoid the "uncanny valley of inference": if the system appears to know too much about the user's inner state, it will feel invasive regardless of accuracy

**No secondary use:**
- Behavioral fingerprint data must never be used for advertising targeting, insurance pricing, or employment screening

### 7.4 Consent as Ongoing Relationship

The most cited ethical principle in the digital phenotyping literature (Torous et al., 2019; Mittelstadt 2019) is that consent for passive monitoring should not be treated as a one-time transaction but as an ongoing relationship in which:
- The scope of inference can be renegotiated
- The user has access to their own data at all times
- The relationship can be ended without penalty (data deletion, not just deactivation)

---

## 8. Research Agenda: Validating the Approach

The following is a phased research program to move from hypothesis to validated method.

### Phase 1: Signal Instrumentation (Months 1–6)
**Objective:** Establish what can actually be measured from Curator's existing data model without new sensors.

Tasks:
- Compute FLRS decay shape metrics for each cluster: fit per-cluster power-law decay curves, compute residuals as "unexpected revisitation" signals
- Define and compute cluster entropy using TF-IDF or topic-model coherence scores across cluster contents
- Instrument Spotlight search events in Curator (can intercept via NSMetadataQuery / FSEvents)
- Log antilibrary growth: files that have not received a single access event since creation
- Extract temporal distribution of file modification events: by hour-of-day and day-of-week, per week

Deliverable: A signal database with one feature vector per week per user, covering all 7 signals above.

### Phase 2: Ground Truth Collection (Months 3–12, overlapping)
**Objective:** Collect validated ground truth on cognitive/emotional state from a volunteer cohort.

Approach: Ecological Momentary Assessment (EMA). Weekly administrations of:
- PHQ-9 (depression screening)
- GAD-7 (anxiety screening)
- Maslach Burnout Inventory – General Survey (MBI-GS), short form
- Perceived Stress Scale (PSS-4)
- Optional: Single-item energy/motivation rating (0-10)

Study design: Within-person longitudinal. Minimum 6 months per participant; target n=30–50 Curator users who opt in. At this scale, within-person models are statistically tractable.

**Ethical safeguards:** Any PHQ-9 score ≥15 (moderately severe depression) triggers a standardized resource referral message. The study is not a clinical intervention.

### Phase 3: Signal-State Correlation Analysis (Months 9–15)
**Objective:** Identify which signals, with which temporal lags, correlate with validated ground truth.

Statistical approach:
- Time-lagged cross-correlation: does signal at time t predict PHQ-9 at time t+1, t+2, t+4 weeks?
- Mixed-effects models with individual random slopes: allows each person's baseline to be estimated separately
- Change-point analysis: do signal shifts co-occur with or precede subjective state transitions?

Primary hypotheses to test (ranked by theoretical priority):
1. FLRS decay shape abnormality predicts PHQ-9 and MBI-GS scores at 2-week lag
2. Cluster entropy increase predicts PSS-4 stress score at 1-week lag
3. Antilibrary growth-to-access ratio predicts MBI-GS at 4-week lag (burnout's slow onset)
4. Failed Spotlight retrieval rate predicts GAD-7 at 1-week lag (anxiety-driven disorganization)
5. Temporal distribution shift (late-night clustering) predicts PHQ-9 at 1-week lag

### Phase 4: Model Development and Evaluation (Months 15–24)
**Objective:** Build a within-person model that generates interpretable risk signals.

Approach:
- LSTM or state-space model trained on each user's own signal-state time series
- Evaluation: leave-future-out cross-validation (never use future data to predict past)
- Primary metric: AUC for detecting "elevated" vs. "baseline" states; Spearman correlation with continuous scale scores
- Interpretability requirement: for any prediction, the model must identify the top-2 contributing signals

**Critical evaluation criterion:** The model should have lower false positive rate than random at the individual level. A 20% false positive rate may be acceptable in a research context but is harmful in a user-facing product.

### Phase 5: UX Validation and Ethical Audit (Months 20–30)
**Objective:** Test whether surfacing these signals to users is helpful or harmful.

Approach: A/B design within participant pool. Half receive ambient reflection prompts based on signal trajectories; half do not. Measure:
- PHQ-9/GAD-7 trajectory: does prompt exposure improve outcomes?
- User-reported sense of agency and control: does the system feel helpful or surveillance-like?
- Opt-out rates: if many users disable the feature, that itself is a finding

**Ethical audit:** External review of all inference logic, surfacing UI, and consent flows by a mental health professional and a data ethics researcher before any public release.

---

## 9. Positioning in the Research Landscape

The Table below summarizes how Curator's proposed approach differs from existing digital phenotyping work:

| Dimension | Smartphone/wearable phenotyping | Keyboard dynamics (BiAffect) | Curator filesystem approach |
|---|---|---|---|
| **Signal type** | Physiological + motor rhythm | Fine-grained motor | Semantic-behavioral |
| **Intentionality of signal** | Passive/unintentional | Semi-intentional | Intentional (purposive information behavior) |
| **Topic/content awareness** | None | None | Yes (at cluster level) |
| **Forgetting model** | None | None | FLRS decay — novel |
| **Temporal grain** | Seconds–hours | Milliseconds | Hours–days |
| **Validated for** | Depression, anxiety, bipolar | Bipolar, depression | Not yet validated |
| **Individual baseline modeling** | Partial (Mohr 2023) | No | Required by design |
| **Inference privacy risk** | High (GPS, microphone) | Medium (typing content) | High (file content topics) |
| **Deployment context** | Mobile-first | Mobile-first | Desktop/laptop-first |

The *semantic-behavioral* dimension is Curator's core differentiator. The information-theoretic properties of file behavior — particularly FLRS decay shape and antilibrary dynamics — have no precedent in the existing digital biomarker literature. This is a genuine contribution opportunity.

---

## 10. Summary of Key Validated Findings from Literature

| Study | Signal | Target | Method | Key result |
|---|---|---|---|---|
| StudentLife (Dartmouth, UbiComp 2014) | GPS, microphone, accelerometer | Depression, stress, GPA | Longitudinal, n=48 | Significant correlations; term-lifecycle pattern |
| Mohr group (npj 2023) | GPS mobility deviation | PHQ-8 depression | n=1013, 16 weeks | Baseline deviation > absolute values |
| BiAffect (UIC, JMIR 2018) | Keystroke timing, backspace rate | Bipolar mood state | n=9 | r ≈ 0.4–0.6 with clinical ratings |
| Nature Sci Reports 2024 | Keystroke dynamics | Depressive tendency | Larger sample | AUC ≈ 0.80 |
| Digital phenotyping SLR (JMIR 2024) | Multi-sensor smartphone | Stress, anxiety, mild depression | 40 studies | Patterns not exclusive to specific conditions |
| Rare Life Event Detection (CHIL 2023) | Multi-modal smartphone | Life events | n=126, ~80 days | Autoencoder + sequence model works |
| Behavioral Drift (MDPI 2025) | Wearable + smartphone | Mental health change | Longitudinal | 30-day personalized baseline; 7-day sustained deviation |
| Burnout/Quantified Workplace (CSCW 2022) | Sleep, activity, work hours | Burnout (resident MDs) | Formative + design provocation | Privacy/power tensions are primary barrier |
| Multitasking from computer logs (Springer 2016) | Window-switching frequency | Stress, multitasking | Case study, n=1 | Positive relation: switching rate ↔ stress |
| FileGram (arXiv 2026) | File-system behavioral traces | User profile/preferences | Synthetic + benchmark | Filesystem operations encode persistent behavioral patterns |
| Dinneen & Julien (JASIST 2020) | File management behavior review | N/A (literature review) | Systematic review, 230+ papers | Gap identified: file behavior as inferential signal not studied |

---

## 11. Conclusions

1. **The gap is real.** No published study has used personal filesystem behavior (file creation rates, access patterns, cluster evolution, forgetting dynamics) as a signal for cognitive or mental health state inference. The nearest work is FileGram (2026), which uses filesystem behavior for personalization, not clinical inference. The Dinneen & Julien review (2020) confirms this gap persists in the PIM literature.

2. **The theoretical grounding is strong.** Analogy to validated signals is close: cognitive withdrawal (depression) should manifest in file creation and access cessation; cognitive scatter (anxiety/overload) should manifest in cluster entropy; motivational dysfunction (burnout) should manifest in FLRS decay shape changes. These mappings are theoretically coherent and testable.

3. **FLRS decay shape is the most novel and theoretically motivated signal.** It has no analog in the smartphone/wearable literature and draws on well-established memory science (Ebbinghaus forgetting curves; spaced repetition research). If validated, it could constitute a genuinely new class of digital biomarker.

4. **Within-person longitudinal design is the only defensible approach.** Cross-person classification is too confounded by individual file-system style. The Mohr group (2023) finding that baseline deviations outperform absolute measures is the critical methodological prior.

5. **Privacy and consent design must be primary, not secondary.** The CSCW 2022 burnout study found that even medically beneficial sensing is rejected when users perceive surveillance or lack control. Mental health inference from file behavior is substantially more sensitive than sleep monitoring from a fitness tracker. The ethical architecture must be built into Curator from the ground floor: on-device inference, user-visible signals, no institutional access, opt-in framing.

6. **The immediate practical contribution is ambient reflection, not diagnosis.** The near-term value of these signals is not automated clinical detection but surfacing patterns to users for their own reflection ("You haven't touched your Personal Finance cluster in 6 weeks — is everything okay there?"). This is lower-risk, higher-acceptability, and more immediately achievable.

---

## Sources

- [StudentLife: Assessing Mental Health, Academic Performance and Behavioral Trends of College Students Using Smartphones (ACM UbiComp 2014)](https://dl.acm.org/doi/10.1145/2632048.2632054)
- [StudentLife Study – Dartmouth CS](https://studentlife.cs.dartmouth.edu/)
- [Differential temporal utility of passively sensed smartphone features for depression and anxiety prediction (npj Mental Health Research 2023)](https://www.nature.com/articles/s44184-023-00041-y)
- [Predicting Mood Disturbance Severity with Mobile Phone Keystroke Metadata: BiAffect (JMIR 2018)](https://www.jmir.org/2018/7/e241/)
- [BiAffect Study – University of Illinois Chicago](https://cs.uic.edu/news-stories/biaffet-app-can-typos-give-insight-into-your-mental-health/)
- [Robust remote detection of depressive tendency based on keystroke dynamics (Scientific Reports 2024)](https://www.nature.com/articles/s41598-024-78489-x)
- [Investigation of daily patterns for smartphone keystroke dynamics based on loneliness and social isolation (PMC 2024)](https://pmc.ncbi.nlm.nih.gov/articles/PMC10874350/)
- [Detecting Multitasking Work and Negative Routines from Computer Logs (Springer 2016)](https://link.springer.com/chapter/10.1007/978-3-319-40397-7_52)
- [Are Six Minutes of Focus Enough? Multitasking Patterns in Workplace Environments (CHIWork 2025)](https://dl.acm.org/doi/10.1145/3729176.3729198)
- [Burnout and the Quantified Workplace: Tensions around Personal Sensing Interventions for Stress in Resident Physicians (CSCW 2022)](https://dl.acm.org/doi/10.1145/3555531)
- [Passive AI Detection of Stress and Burnout Among Frontline Workers (MDPI 2025)](https://www.mdpi.com/2039-4403/15/11/373)
- [Digital Phenotyping for Stress, Anxiety, and Mild Depression: Systematic Literature Review (JMIR mHealth 2024)](https://mhealth.jmir.org/2024/1/e40689)
- [Large-scale digital phenotyping: identifying depression and anxiety (arXiv 2024)](https://arxiv.org/pdf/2409.16339)
- [Rare Life Event Detection via Mobile Sensing Using Multi-Task Learning (CHIL 2023)](https://proceedings.mlr.press/v209/pillai23a.html)
- [From Patterns to Deviations: Detecting Behavioural Drift (MDPI Electronics 2025)](https://www.mdpi.com/2079-9292/15/4/885)
- [The ubiquitous digital file: A review of file management research – Dinneen & Julien (JASIST 2020)](https://asistdl.onlinelibrary.wiley.com/doi/abs/10.1002/asi.24222)
- [FileGram: Grounding Agent Personalization in File-System Behavioral Traces (arXiv 2026)](https://arxiv.org/abs/2604.04901)
- [Sensing Wellbeing in the Workplace, Why and For Whom? (ACM CHI 2023)](https://dl.acm.org/doi/10.1145/3610207)
- [Data mining for health: staking out the ethical territory of digital phenotyping (npj Digital Medicine 2019)](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6550156/)
- [Predictive privacy: towards an applied ethics of data analytics (Ethics and Information Technology 2021)](https://link.springer.com/article/10.1007/s10676-021-09606-x)
- [A call for open data to develop mental health digital biomarkers (PMC 2022)](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8935940/)
- [The State of Digital Biomarkers in Mental Health (PMC 2024)](https://pmc.ncbi.nlm.nih.gov/articles/PMC11584197/)
- [Personal Sensing: Understanding Mental Health Using Ubiquitous Sensors and Machine Learning (PMC 2019)](https://pmc.ncbi.nlm.nih.gov/articles/PMC6902121/)
- [Systematic review of smartphone-based passive sensing for health and wellbeing (PMC 2018)](https://pmc.ncbi.nlm.nih.gov/articles/PMC5793918/)
- [Anomaly detection to predict relapse risk in schizophrenia (PMC 2021)](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7798381/)
- [Heart Rate Variability as a Biomarker of Burnout (medRxiv 2025)](https://www.medrxiv.org/content/10.1101/2025.09.06.25335221.full.pdf)
- [Inferring Mental States from Brain Data: Ethico-legal Questions (PMC 2025)](https://pmc.ncbi.nlm.nih.gov/articles/PMC11882133/)
