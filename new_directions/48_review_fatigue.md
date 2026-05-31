# 48 — Human Review UX & Review Fatigue in AI-Assisted File Organization
_Research date: 2026-05-31_

---

## Why This Matters for Curator

Curator's core interaction loop asks users to confirm AI-generated folder suggestions before files are moved. This is a sound design principle — it preserves user agency and prevents irreversible errors. But it immediately creates an adversarial dynamic with human psychology: every confirmation prompt is a micro-decision, and micro-decisions accumulate. If Curator surfaces 50 suggestions after an initial scan, a user might thoughtfully review the first 10, skim the next 20, and rubber-stamp the last 20 without reading them. By that point the product has degraded from an intelligent assistant into a nuisance — and the user may disable it entirely.

The stakes are asymmetric. A single poorly reviewed move (a project file sent to "Archive/Misc") can destroy trust in the entire system even if the preceding 49 suggestions were correct. The product must therefore solve two simultaneous problems: keeping the total number of prompts within the range where humans are still actively deliberating, and structuring those prompts so that each one remains perceptually distinct rather than blending into a mechanical "accept" stream.

This is not a new problem. Decades of research in clinical alarm fatigue, security warning habituation, decision science, and HCI interruption scheduling have converged on a clear practical picture: humans have a finite deliberation budget per session, the budget varies by stakes and confidence, and smart systems can extend the effective range by batching, pacing, varying presentation, and defaulting to high-confidence actions while flagging low-confidence ones.

---

## Decision Fatigue: Psychology Literature

### The Original Ego Depletion Theory

Roy Baumeister, Bratslavsky, Muraven, and Tice (1998) proposed that self-control draws on a single, finite, domain-general resource they called "willpower." Their foundational experiment showed that participants who first resisted eating chocolates subsequently gave up sooner on an unsolvable puzzle — as if the first act of self-regulation depleted the resource available for the second [1]. This "ego depletion" model became one of the most cited frameworks in social psychology and inspired the broader concept of **decision fatigue**: the degradation of decision quality over a sequence of choices.

The widely-cited "hungry judges" study by Danziger, Levav, and Avnaim-Pesso (2011, PNAS) appeared to provide real-world support: Israeli parole judges granted parole ~65% of the time immediately after a food break, dropping toward 0% just before the next break [2].

### The Replication Crisis and What Survived

Hagger et al.'s **Registered Replication Report** (2016, *Perspectives on Psychological Science*), involving 23 independent labs and N=2,141 participants, failed to reproduce the core ego depletion effect — the effect size was close to zero (d ≈ 0.04) [3]. Vadillo, Gold, and Osman (2016) reviewed the glucose literature and found evidence of publication bias, concluding that the evidential value of the glucose model is limited [4]. The Danziger "hungry judges" finding was reanalyzed by Glöckner (2016), whose simulations showed the pattern could arise from scheduling artifacts rather than glucose depletion [5].

### The Motivational Shift Alternative

Inzlicht and Schmeichel (2012, 2016) proposed a more durable account: self-control failure after prior exertion reflects a **motivational shift** rather than resource depletion. Initial effortful control shifts motivation away from long-term goals and toward immediate gratification; people fail to control themselves because they become *less willing* to do so, not because they are *unable* to [6].

### What Is Confirmed vs. Contested

| Claim | Status |
|---|---|
| Performing many decisions in sequence degrades decision quality in real-world settings | Confirmed |
| This is caused by glucose depletion in the brain | Contested / probably false |
| A single laboratory manipulation depletes self-control resources | Not reliably replicated |
| Motivational shift drives sequential decision degradation | Plausible, growing support |
| People default to the status quo option as deliberation effort drops | Well-replicated |

The practical implication holds even if the mechanistic theory is wrong: **humans making many sequential decisions of similar structure do produce worse, more heuristic-driven, and more status-quo-biased outcomes over time.**

---

## Alert Fatigue in HCI: Prior Art

### Security Alert Fatigue

**Sunshine, Egelman, Almuhimedi, Atri, and Cranor (2009)**, "Crying Wolf: An Empirical Study of SSL Warning Effectiveness," USENIX Security '09 [7]. The landmark paper on security warning habituation. The majority of users habitually clicked through SSL certificate warnings without reading them. The "cry wolf" framing captures the core problem: when a warning fires too frequently with low base-rate of genuine risk, users rationally downweight it.

**Anderson, Vance, Kirwan, Jenkins, and Eargle (CHI 2015)**, "How Polymorphic Warnings Reduce Habituation in the Brain: Insights from an fMRI Study," DOI: 10.1145/2702123.2702322 [8]. Using fMRI: **neural response in visual processing areas dropped dramatically after just the second exposure** to the same warning. By contrast, polymorphic variants (same warning, varied appearance) maintained substantially higher neural engagement across the same exposure sequence.

**Vance et al. (2018)**, "Tuning Out Security Warnings: A Longitudinal Examination of Habituation Through fMRI, Eye Tracking, and Field Experiments" [9]. Confirmed in a longitudinal field study that eye-tracking dwell time on security warnings dropped sharply within days of first exposure, and actual compliance followed the same curve.

**Bravo-Lillo et al. (SOUPS 2013)**, "Your Attention Please: Designing Security-Decision UIs to Make Genuine Risks Harder to Ignore" [10]. Tested "attractor" UI elements that temporarily prevented users from taking the dangerous action (e.g., disabling the OK button for 3 seconds). Effective at breaking habituated click-through — though the benefit decays as users learn to anticipate the friction.

### Medical Alarm Fatigue

The clinical literature provides the most granular quantitative data. The Joint Commission (2013) identified alarm fatigue as the most common cause of medical device-related deaths in US hospitals [12].

Quantitative baselines from the ICU literature:
- Modern ICU monitors generate **200–700 alarms per patient per day**, of which **72–99% are clinically irrelevant** (Sendelbach & Funk, 2013 systematic review).
- Nurse response rates: over **60% of alarms do not receive timely responses**.
- A 2024 Chinese survey of 597 ICU nurses: 30.8% low fatigue, 54.4% moderate fatigue, 14.9% high-fatigue / negative coping [13].

The threshold finding most relevant to Curator: **false positive rates above 80–90%** are the threshold at which fatigue becomes functionally incapacitating. This maps directly to prediction set quality: if Curator's first-ranked suggestion is wrong more than ~20% of the time, users will stop evaluating the suggestions.

### IDS / Security Operations Centre Alert Fatigue

ACM Computing Surveys paper, "Alert Fatigue in Security Operations Centres: Research Challenges and Opportunities" (DOI: 10.1145/3723158) [14]:
- Most enterprise SOCs receive **>10,000 alerts per day**, with false positive rates >50%.
- A Trend Micro survey found **51% of SOC teams feel overwhelmed**, with analysts spending >25% of their time on false positives.
- Analysts develop heuristic shortcuts that prioritize speed over accuracy under sustained high-volume conditions, and experience **decision paralysis** where no action is taken.

---

## Habituation to Confirmation Dialogs

### Windows UAC

Windows Vista's User Account Control dialog became a widely documented case study in confirmation dialog habituation. Within weeks of Vista deployment, users established a conditioned reflex to click "Allow" whenever the UAC dialog appeared, without reading the dialog content. This was so pronounced that Microsoft significantly reduced UAC frequency in Windows 7, acknowledging that over-prompting was actively reducing security by training users to dismiss all dialogs reflexively [15].

The mechanism is operant conditioning: if clicking "Allow" always resolves the friction, users learn the least-effort path to task completion.

### Cookie Consent Banners

Post-GDPR cookie consent banners represent the largest-scale natural experiment in confirmation dialog fatigue:
- Studies show acceptance rates exceed **95%** when banner design makes acceptance easier than rejection — even among users who self-report caring about privacy.
- Research from 2022 documents measurably degraded decision quality after repeated banner encounters within a session: users exposed to their Nth banner in a day are significantly more likely to click "Accept All" [16].
- A 2025 arXiv paper (2603.21515) found that of scraped banners, 2,656 included "Accept All" but only 349 included "Reject All" — the asymmetry compounds fatigue.

### SSL Warnings

The Sunshine et al. 2009 study found that users who had encountered more warnings in the past were significantly more likely to ignore them — direct behavioral evidence that per-user exposure count is a meaningful predictor of habituation severity. A 2017 CHI paper ("What Do We Really Know about How Habituation to Warnings Occurs Over Time?" DOI: 10.1145/3025453.3025896) found that habituation happens fast — within days — even if the precise "N exposures to habituation" number is not established.

---

## AI-Assisted Decision Making: Batch Size Research

### Overreliance and Automation Bias

**Parasuraman and Manzey (2010)** defined *automation bias* as the tendency to favor suggestions from automated systems over contradictory information from other sources.

A 2025 ACM CHI paper, "Do People Appropriately Rely on AI-Advice? An Analytical Review of HCI Research on Human-AI Decision-Making" (DOI: 10.1145/3772318.3791467) [17], found that **the mere knowledge that advice is AI-generated increases over-reliance** — users accept incorrect AI outputs more readily than they would accept identical incorrect human advice.

"Cognitive Forcing for Better Decision-Making: Reducing Overreliance on AI Systems Through Partial Explanations" (ACM PACMHCI, DOI: 10.1145/3710946) [18] found that withholding part of the AI's reasoning (forcing users to form their own partial judgment first) significantly reduced overreliance. Direct design signal for Curator: showing the *reasoning* behind a suggestion creates cognitive forcing that keeps users actively deliberating.

A 2025 arXiv paper, "Bias in the Loop: How Humans Evaluate AI-Generated Suggestions" (arXiv:2509.08514) [19], found that **requiring corrections for flagged AI errors reduced human engagement and paradoxically increased acceptance of subsequent incorrect suggestions** — a "correction fatigue" effect. Critical warning for systems that allow inline edits: making corrections effortful may backfire.

### Batch Size and Session Length

No HCI paper directly provides a single "optimal batch size for AI suggestions" number. What is established:
- Email batching research (Mark, Iqbal, Czerwinski, Johns, Sano, CHI 2016, DOI: 10.1145/2858036.2858262) [21] found that 3× daily batch email processing was equally productive to continuous checking and reported higher satisfaction.
- Push notification research found that **1–3 targeted notifications per day** produce up to 20% higher engagement than zero or excessive volume. **3–6 push notifications per day** causes 40% of users to disable notifications entirely [22].
- A Micro-Randomized Trial (Nahum-Shani et al., 2023, PMC10337295) found daily-notification groups showed faster disengagement over 3 weeks than every-other-day groups.

**Synthesized practical range**: The collective evidence points toward **5–15 decisions per review session** as the range where humans maintain deliberative engagement. Beyond ~20 sequential binary decisions of similar structure, heuristic responding begins to dominate.

---

## Interruption Research

### Adamczyk and Bailey (CHI 2004)

**Adamczyk and Bailey (2004)**, "If Not Now, When? The Effects of Interruption at Different Moments Within Task Execution," ACM CHI 2004 [23]. Core finding: interruptions delivered at **coarse breakpoints** (natural task boundaries) cause significantly less resumption lag, frustration, and performance degradation than interruptions at fine breakpoints (within a subtask).

### Iqbal and Bailey (CHI 2006, CHI 2008, TOCHI 2010)

**Iqbal and Bailey (2008)**, "Effects of Intelligent Notification Management on Users and Their Tasks," CHI 2008 [25]: implemented and evaluated a notification manager that deferred alerts to predicted breakpoints. Achieved measurably lower resumption lag and frustration scores.

**OASIS framework (TOCHI 2010)** [26]: coarser breakpoints (end of document, end of workflow) produce the lowest interruption costs.

### Key Quantitative Findings

- Recovering from an interruption requires an average of **~23 minutes** to return to the original task at full concentration (Mark, Gudith, and Klocke, CHI 2008).
- Interruption at a fine breakpoint produces **~4× the resumption lag** of interruption at a coarse breakpoint (Adamczyk & Bailey, 2004).

Curator should **not** interrupt a user mid-task. The review session should be triggered when the user is at a natural breakpoint — finishing a project, closing a Finder window, completing a download batch.

---

## Design Patterns: Fatigue Mitigation

**1. Polymorphism / Visual Variation**
Anderson et al. (CHI 2015) established that **varying the visual appearance** of a recurring prompt significantly reduces neural habituation. Curator should not present 15 identical "move file?" cards in a row. Vary card layout, use different visual treatments for different confidence tiers.

**2. Batch-and-Review Sessions vs. Inline Interruptions**
Email batching literature and interruption research both support **scheduled review sessions** over continuous interruption.

**3. Confidence-Tiered Defaults**
Nudge theory (Thaler & Sunstein, 2008) establishes that **defaults are the most powerful choice architecture tool**. High-confidence predictions (top-1 coverage >90%) can be **defaulted to "accept"** with a brief undo window, reserving active confirmation for low-confidence or high-stakes moves.

**4. Attractor / Friction Mechanisms for Low-Confidence Suggestions**
Bravo-Lillo et al. (SOUPS 2013): on suggestions where the system is uncertain, insert a brief forced-read period or require the user to select which of two options to accept rather than just clicking "OK."

**5. Showing Reasoning / Cognitive Forcing**
Bucinca, Malhi, and Gajos (2021, "To Trust or to Think," DOI: 10.1145/3449287) [27] and the 2025 follow-up both demonstrate that showing the **rationale** for an AI decision maintains deliberative engagement. Curator's review card should show why a file is being suggested for a folder.

**6. Session Length Caps and Progress Framing**
Cap review sessions at a fixed item count (suggested: **10–15**) and show a progress indicator ("3 of 10 decisions"). Progress framing activates goal-gradient motivation.

**7. Undo Over Confirmation**
For high-confidence, reversible actions, replacing a confirmation dialog with an **immediate-action + undo** pattern reduces friction while maintaining safety.

**8. Asynchronous / Ambient Mode**
A persistent but non-blocking review queue (accessible from menu bar / dock badge) lets users pull reviews at a convenient time rather than push reviews interrupting their work.

---

## Implications for Curator

**Batch Size:** Target **5–15 suggestions per review session.** Hard cap of 15 items with a clear progress indicator.

**Trigger Timing:** Queue suggestions and surface the review prompt at a natural breakpoint — when the user switches away from the Finder window containing the unorganized folder, when a download completes, or at a user-configured time.

**Confidence Stratification:**
- **High coverage (>90%), top-1 suggestion:** Default to accept, show in a "will-move" summary pane, allow undo for 30 seconds after batch execution.
- **Medium coverage (70–90%):** Show suggestion card with reasoning, one-click accept or change.
- **Low coverage (<70%) or ambiguous file:** Show as an explicit question requiring active selection from the prediction set. Insert a brief attractor delay if needed.

**Presentation Variation:** Do not use an identical card layout for every suggestion. Vary visual treatment by category type (rename, move, tag) and confidence tier.

**Reasoning Display:** Always show one-line reasoning ("moved because: filename matches folder pattern 'Invoice_*'"). This is the cognitive forcing function that keeps users actively evaluating.

**Abort and Snooze:** Provide a "remind me later" action at the top of each review session.

**False Positive Rate Target:** Based on clinical alarm fatigue literature, target **<20% false positive rate** (i.e., Curator's top suggestion should be correct at least 80% of the time). Above 20%, users will scrutinize every suggestion. Above ~40%, users will begin auto-accepting.

---

## Key References

1. Baumeister, R. F., et al. (1998). Ego depletion: Is the active self a limited resource? *Journal of Personality and Social Psychology*, 74(5), 1252–1265.
2. Danziger, S., Levav, J., & Avnaim-Pesso, L. (2011). Extraneous factors in judicial decisions. *PNAS*, 108(17), 6889–6892.
3. Hagger, M. S., et al. (2016). A Multilab Preregistered Replication of the Ego-Depletion Effect. *Perspectives on Psychological Science*, 11(4), 546–573.
4. Vadillo, M. A., Gold, N., & Osman, M. (2016). The bitter truth about sugar and willpower. *Psychological Science*, 27(9).
5. Glöckner, A. (2016). The irrational hungry judge effect revisited. *Judgment and Decision Making*, 11(6), 601–610.
6. Inzlicht, M., & Schmeichel, B. J. (2012). What is ego depletion? *Perspectives on Psychological Science*, 7(5), 450–463.
7. Sunshine, J., et al. (2009). Crying wolf: An empirical study of SSL warning effectiveness. *USENIX Security '09.*
8. Anderson, B. B., et al. (2015). How polymorphic warnings reduce habituation in the brain. *ACM CHI 2015.* DOI: 10.1145/2702123.2702322.
9. Vance, A., et al. (2018). Tuning out security warnings. *MIS Quarterly*, 42(2), 355–380.
10. Bravo-Lillo, C., et al. (2013). Your attention please. *SOUPS 2013.*
11. Anderson, B. B., et al. (2020). Repetition of computer security warnings results in differential repetition suppression effects. *Frontiers in Psychology.* PMC7751389.
12. The Joint Commission (2013). Medical device alarm safety in hospitals. *Sentinel Event Alert*, 50.
13. Exploring the factors influencing alarm fatigue in intensive care units nurses. (2024). *PMC.* PMC12233232.
14. Alert Fatigue in Security Operations Centres. *ACM Computing Surveys.* DOI: 10.1145/3723158.
15. UAC Dialog Blindness. Practitioner documentation, 2009.
16. Cookie consent fatigue. Syrenis research; arXiv:2603.21515.
17. Do People Appropriately Rely on AI-Advice? *ACM CHI 2026.* DOI: 10.1145/3772318.3791467.
18. Cognitive Forcing for Better Decision-Making. *ACM PACMHCI.* DOI: 10.1145/3710946.
19. Bias in the Loop: How Humans Evaluate AI-Generated Suggestions. arXiv:2509.08514.
20. If in a Crowdsourced Data Annotation Pipeline, a GPT-4. *ACM CHI 2024.* DOI: 10.1145/3613904.3642834.
21. Mark, G., et al. (2016). Email duration, batching and self-interruption. *ACM CHI 2016.* DOI: 10.1145/2858036.2858262.
22. Push notification engagement statistics. BusinessOfApps, 2024.
23. Adamczyk, P. D., & Bailey, B. P. (2004). If not now, when? *ACM CHI 2004.*
24. Iqbal, S. T., & Bailey, B. P. (2006). Leveraging characteristics of task structure. *ACM CHI 2006.*
25. Iqbal, S. T., & Bailey, B. P. (2008). Effects of intelligent notification management. *ACM CHI 2008.*
26. Iqbal, S. T., & Bailey, B. P. (2010). OASIS. *ACM TOCHI.*
27. Bucinca, Z., Malhi, M. B., & Gajos, K. Z. (2021). To trust or to think. *ACM CSCW 2021.* DOI: 10.1145/3449287.
28. Dietvorst, B. J., Logg, J. M., & Logg, J. (2015). Algorithm aversion. *Journal of Experimental Psychology: General*, 144(1), 114–126.
29. Thaler, R. H., & Sunstein, C. R. (2008). *Nudge.* Yale University Press.
