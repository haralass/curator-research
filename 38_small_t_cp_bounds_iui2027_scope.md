# Small-T Conformal Prediction Coverage Bounds and IUI 2027 Paper Scope

**Date:** 2026-05-31
**Project:** Curator — macOS AI File Organizer
**Topic:** CP coverage guarantees under small calibration sets, and strategic scoping of the IUI 2027 submission

---

## 1. The Small-T Problem in Conformal Prediction

Standard split conformal prediction (CP) is a distribution-free framework that, under exchangeability, produces prediction sets with exact finite-sample marginal coverage: if you draw `n` calibration points and a new test point from the same distribution, the probability that the test point is covered is at least `1 - α`. The formal guarantee is:

```
P(Y_test ∈ C(X_test)) ≥ 1 - α
```

where the coverage threshold is computed as the `⌈(1-α)(n+1)⌉/n`-th quantile of the calibration nonconformity scores. The `+1` inside the ceiling is the finite-sample correction — without it, empirical coverage is systematically slightly below nominal.

The critical observation for Curator: **this guarantee is marginal in expectation over the calibration draw**. For a single, fixed calibration set of size `n`, the realized coverage can deviate substantially from `1-α`. The smaller `n` is, the larger this deviation.

### 1.1 What Small n Actually Means

Let `Cov_n` be the empirical coverage achieved by split CP with `n` calibration points. Under exchangeability, the exact finite-sample distribution is:

```
K ~ Beta-Binomial(n_test, ⌈(1-α)(n+1)⌉, ⌊α(n+1)⌋)
```

where `K` is the number of covered test points. The variance of the empirical coverage rate is:

```
Var(Cov_n) ≈ α(1-α) / n
```

For concrete numbers:

| Calibration n | α = 0.10 | Std dev of coverage | 95% CI for realized coverage |
|--------------|----------|---------------------|------------------------------|
| n = 20       | 0.10     | ±0.067              | [0.77, 1.00]                 |
| n = 50       | 0.10     | ±0.042              | [0.82, 0.98]                 |
| n = 100      | 0.10     | ±0.030              | [0.84, 0.96]                 |
| n = 200      | 0.10     | ±0.021              | [0.86, 0.94]                 |
| n = 500      | 0.10     | ±0.013              | [0.87, 0.93]                 |

**Interpretation for Curator:** A user with a personal filesystem of 300 files, using 50% for calibration (`n = 150`), has a realized coverage that is likely within ±2.5 percentage points of the nominal level. This is tight enough to be useful. However, a user with only 80 files (`n = 40`) may experience coverage that swings up to ±7 points — meaning the "90% routing confidence" claim becomes a 83–97% interval around the real number. The system still works, but the uncertainty is higher than it looks.

### 1.2 The Small Sample Beta Correction (SSBC)

Recent work by Zwart and colleagues (arXiv:2509.15349) formalizes this problem and proposes the **Small Sample Beta Correction (SSBC)**: a plug-and-play adjustment to the significance level `α` that leverages the exact Beta-Binomial distribution of conformal coverage. Rather than targeting expected coverage of `1-α`, SSBC ensures that **with user-defined probability `δ` over the calibration draw**, the deployed predictor achieves at least `1-α` coverage.

This is a strictly stronger statement. It converts the "in expectation" guarantee into a probabilistic guarantee: "with probability at least `1-δ`, your deployed model achieves at least 90% coverage." For production deployment of Curator on real user devices, this is the right framing. SSBC raises the effective `α` used at calibration time to account for variance, producing slightly larger prediction sets but making the coverage claim honest.

**Practical implication:** For `n = 50` and nominal `α = 0.10`, SSBC increases the effective threshold to achieve `δ = 0.90` probabilistic coverage guarantee — meaning approximately 0.047 violation rate in experiments versus 0.10 without correction. This is a meaningful improvement for safety-critical routing decisions.

---

## 2. Jackknife+ and Cross-Conformal for Small Personal Filesystems

### 2.1 Jackknife+ (Barber et al., Annals of Statistics 2021)

Barber, Candès, Tibshirani and Venn introduced the **jackknife+** as an improvement over standard leave-one-out (LOO) jackknife. The key theoretical result is:

```
P(Y_test ∈ C_jackknife+(X_test)) ≥ 1 - 2α
```

This is a finite-sample guarantee. The cost of not having a separate calibration set (using full LOO instead) is a factor of `2` on `α`. For `α = 0.10`, jackknife+ guarantees at least 80% coverage rather than 90%.

The advantage for Curator: **jackknife+ uses all available labeled data for both training and calibration** via the LOO structure. There is no wasteful split. For a user with 100 files, split CP might use 70 for training and 30 for calibration; jackknife+ uses all 100 for training while still producing calibrated uncertainty.

The coverage cost (`2α` instead of `α`) is real and must be communicated honestly: jackknife+ is appropriate when data is very scarce and the `2α` factor is acceptable, or when a slightly inflated `α` is used to recover the desired coverage level.

### 2.2 K-Fold CV+ (Cross-Conformal)

An intermediate method, K-Fold CV+ (or cross-conformal), partitions data into `K` folds and achieves:

```
P(Y_test ∈ C_CV+(X_test)) ≥ 1 - 2α - √(2/n)
```

For `n = 100` and `K = 5`, this gives coverage ≥ `1 - 2(0.10) - 0.14 = 0.76`. Still conservative, but better than split CP on very small `n` because it uses more data for training.

**Recommendation for Curator:** Use jackknife+ as the baseline small-`n` method when `n < 100`. Transition to standard split CP (with SSBC correction) when `n ≥ 150`. This threshold should be surfaced to developers as a configurable runtime parameter.

---

## 3. Mondrian CP with Small Strata

Mondrian conformal prediction (MCP) provides **group-conditional** coverage: each file-type stratum (PDF, image, code file, etc.) has its own separate coverage guarantee. The method simply applies split CP independently within each stratum.

### 3.1 The Stratum Size Problem

If Curator stratifies by file type, and a user has:
- 200 PDFs (n_strata ≈ 100 calibration)
- 60 images (n_strata ≈ 30 calibration)
- 15 code files (n_strata ≈ 7 calibration)

The PDF stratum gets decent coverage guarantees (coverage variance ≈ ±3%). The code file stratum is nearly uncalibrated — with `n = 7`, the coverage can swing from 60% to 100%.

The fundamental Mondrian property is that group-conditional coverage is **exact** within each stratum under exchangeability. But exactness here means the distribution of coverage follows the same Beta-Binomial law — it does not mean coverage variance is small. The guarantee holds, but it is wide.

### 3.2 Practical Mitigations

Three practical approaches for Curator:

1. **Stratum merging with minimum size threshold:** If a file-type stratum has fewer than `n_min = 30` calibration examples, collapse it into a parent category (e.g., merge all "code" subtypes into a single stratum). This preserves meaningful coverage bounds.

2. **Hierarchical Mondrian:** Define a two-level taxonomy. Level 1: broad categories (documents, media, data, code). Level 2: fine-grained types. Calibrate at Level 2 when `n ≥ 50`, otherwise fall back to Level 1. This is analogous to hierarchical conformal classification (arXiv:2508.13288).

3. **SSBC per stratum:** Apply the SSBC correction within each stratum. For small strata, SSBC will inflate the prediction set (route to multiple candidate folders) — which is arguably the right behavior: express uncertainty honestly.

---

## 4. Adaptive CP for Cold Start

### 4.1 The Day-1 Problem

A new Curator user has zero calibration history. Standard CP requires calibration data. This is the classic cold-start problem, and it is acute for CP because there is no asymptotic workaround — CP simply cannot produce valid prediction sets with zero calibration points.

Three principled approaches:

**Approach A — Population Prior with Transfer Calibration:**
Use a held-out population-level calibration set collected from opt-in users during development. Calibrate on this population dataset and deploy population-level nonconformity thresholds as the initial model. As user-specific data accumulates, blend toward user-specific calibration using a weighted scheme. This is analogous to the Conf-OT transfer learning approach (arXiv:2502.17744), which uses optimal transport to bridge source and target domain calibration while maintaining coverage guarantees.

**Approach B — Online / Sequential CP:**
Online conformal prediction (OCP) provides **deterministic long-run miscoverage guarantees** without requiring a batch calibration set. The threshold is updated sequentially after each observed outcome. The guarantee is:

```
(1/T) Σ_{t=1}^{T} 1{Y_t ∉ C_t(X_t)} ≤ α + O(1/T)
```

For Curator, this means: on day 1, prediction sets may be very wide (high uncertainty); by day 30, if the user has routed 50 files, the system has accumulated enough signal to be nearly as tight as batch CP. The regret is sublinear — the system "catches up." Adaptive Conformal Inference (ACI) extends OCP to handle non-exchangeable, drifting distributions, which is important for growing personal filesystems (arXiv:2302.07869).

**Approach C — Hierarchical Shrinkage CP:**
For zero-shot users, route with maximum uncertainty (route to the most probable folder but report `α = 0.50` confidence). As files accumulate, progressively tighten confidence reporting. This is not a rigorous CP guarantee but a pragmatic UX decision — communicate to users that confidence bounds will tighten over time.

**Recommendation:** Implement Approach B (online CP) from day 1, initialized with population-level score distributions from Approach A. This gives coverage guarantees that are valid from the first file routed, with improving tightness over time. The mathematical story is clean and the engineering is tractable.

---

## 5. IUI 2027 Paper Scope Definition

### 5.1 Conference Facts

- **Full name:** 32nd Annual ACM Conference on Intelligent User Interfaces (ACM IUI 2027)
- **Abstract registration deadline:** August 13, 2026
- **Paper submission deadline:** August 20, 2026
- **Time from today (2026-05-31):** 11 weeks to abstract, 11.5 weeks to full paper
- **Format:** ACM SIGCHI (PCS submission portal)
- **Page limit:** No strict page cap, but **10,000 words strongly encouraged**; papers over 12,000 words require a justification note
- **Supplemental:** Optional but encouraged (study materials, demo videos, data sheets)
- **Scope:** Intelligent user interfaces, ML-powered interaction systems, uncertainty in AI-assisted tools, personalization, file/PIM systems all fit

### 5.2 What IUI Reviewers Value

Based on IUI 2024–2026 norms and call-for-papers language:

- **Dual contribution:** A paper must contribute something computational (system, algorithm, model) AND something human-centric (user study, behavioral analysis, formative study). Single-contribution papers (pure systems or pure user studies) are publishable but riskier.
- **Rigorous evidence:** Claims must be backed by evidence appropriate to the contribution type. For a system paper, this is typically a technical evaluation plus a user study. For a behavior paper, a controlled study.
- **Practical motivation:** IUI is not a pure ML venue. The HCI framing — why does this matter for users, what interaction problem does it solve? — must be explicit and compelling.
- **User study sample:** IUI has accepted lab studies with n = 12–20 participants for exploratory/formative work; n = 20–30 for controlled evaluations. Lab studies are credible if within-subject design is used and effect sizes are meaningful.

### 5.3 The Minimum Viable Paper (MVP)

Given 11 weeks, here is the realistic MVP for IUI 2027:

**Title (draft):** "Curator: Calibrated Routing for Personal File Organization with Conformal Prediction"

**Central claim (single sentence):** A conformal prediction routing layer provides statistically valid coverage guarantees for AI-assisted file organization, matching or exceeding user mental models of system reliability while reducing file retrieval time.

**Core contributions:**
1. FLRS (File Location Retrieval Score) as a behavioral signal that captures file location forgetting — motivated by memory and PIM literature
2. A CP-based folder routing system that produces honest, calibrated prediction sets (not point predictions)
3. A small lab user study (n = 20, within-subjects) demonstrating: (a) measured improvement in file retrieval time, and (b) that users' trust calibration matches the CP-reported confidence better than a baseline uncalibrated system

**What does NOT need to be in the paper:**
- Cold-start online CP (frame as future work)
- Full Mondrian stratification (mention as extension)
- SSBC correction (mention in limitations)
- The full Curator app ecosystem — focus on the routing component

### 5.4 What Can Be Automated vs. Requires a User Study

**Fully automated evaluation (can be done in 2–3 weeks):**
- Coverage validation: Run CP on a held-out personal filesystem dataset; confirm empirical coverage matches nominal α
- Prediction set size: Measure average set sizes as a function of calibration n (validate the theory from Sections 1–3)
- Routing accuracy: Compare CP routing decisions against ground-truth human-labeled folder placements
- Cold-start simulation: Simulate online CP on sequential file traces; plot coverage convergence curve

**Requires user study (needs 4–6 weeks for design + recruitment + running + analysis):**
- File retrieval time: Does CP-assisted Curator reduce time to re-find files vs. baseline Finder search?
- Trust calibration: Do users' subjective confidence ratings match the CP-reported confidence levels?
- Qualitative experience: How do users interpret the prediction set (multiple candidate folders)?

**Recommended split:** 3 weeks technical evaluation, 5 weeks user study (design + IRB-equivalent + 20 participants + analysis), 3 weeks writing. This is tight but feasible if design begins immediately.

### 5.5 User Study Design Sketch (Lab Study, n = 20)

**Design:** Within-subjects, counterbalanced
**Conditions:** (A) Baseline — manual Finder search; (B) Curator with point prediction routing; (C) Curator with CP-calibrated prediction sets
**Tasks:** 20 file retrieval tasks per condition, files seeded into a realistic personal filesystem across 3 sessions
**Measures:**
- Primary: File retrieval time (seconds)
- Secondary: Trust calibration score (Brier score between user confidence ratings and system confidence)
- Tertiary: NASA-TLX cognitive load, qualitative post-session interview

**Statistical power:** With n = 20 within-subjects and expected medium effect size (d = 0.5) from CP's efficiency improvement, a paired t-test achieves >80% power at α = 0.05. This is publishable at IUI.

### 5.6 The Paper Narrative Arc

The narrative must flow from a human problem to a technical solution back to a human validation:

1. **The problem (Introduction):** People forget where they put their files. This is a documented, quantified problem (cite PIM literature). Existing AI solutions (Spotlight, smart folders) make confident predictions with no uncertainty quantification — users can't tell when to trust the system.

2. **The gap (Related Work):** CP has been used in robotics, medicine, NLP — but never for personal filesystem routing. PIM systems lack calibrated uncertainty. This is the gap.

3. **The system (Curator):** FLRS signal → file embedding → CP router → prediction set delivery. Describe the CP routing layer in detail. Include the small-n analysis to show the system is honest about its own limitations.

4. **Technical validation:** Automated experiments on held-out filesystem datasets. Coverage holds. Prediction sets are tight. Cold-start converges.

5. **User study:** Lab study n = 20. Retrieval time drops. Trust calibration improves vs. baseline. Users appreciate knowing when the system is uncertain.

6. **Discussion:** What does calibrated uncertainty mean for PIM systems broadly? Implications for AI-assisted organization tools.

7. **Limitations and future work:** Online CP for cold start, Mondrian stratification, longitudinal deployment.

---

## 6. Key Theoretical Results to Cite in the Paper

| Result | Citation | Key Claim |
|--------|----------|-----------|
| Exact finite-sample CP coverage | Vovk, Gammerman, Shafer (2005); Shafer & Vovk (2008) | Coverage ≥ 1-α under exchangeability, exact for finite n |
| Jackknife+ finite-sample guarantee | Barber, Candès, Tibshirani, Venn, Ann. Stat. 2021 | Coverage ≥ 1-2α without train/calibration split |
| Mondrian CP group-conditional coverage | Vovk et al.; Gibbs & Cherian (2023) arXiv:2305.12616 | Per-stratum exact coverage, independent guarantees |
| SSBC for small calibration sets | arXiv:2509.15349 | Probabilistic coverage control with user-defined δ |
| Online / Adaptive CP | Gibbs & Candès (ACI); arXiv:2302.07869 | Long-run coverage without batch calibration set |
| Hierarchical conformal | arXiv:2508.13288 | Class-hierarchy-aware prediction sets |

---

## 7. Strategic Risks and Mitigations

**Risk 1: User study logistics take longer than 5 weeks.**
Mitigation: Pre-register the study protocol. If data collection slips past the submission deadline, submit the paper with the automated evaluation as the primary result and the user study as an ongoing/pilot result. IUI accepts "system + technical eval" papers; this is riskier for acceptance but avoids missing the deadline.

**Risk 2: Reviewers expect a larger-scale deployment study.**
Mitigation: Frame the lab study as a controlled proof-of-concept, explicitly calling for longitudinal in-the-wild follow-up. IUI reviewers generally accept lab studies for novel system contributions if the technical evaluation is rigorous.

**Risk 3: CP coverage bounds with small n look unimpressive.**
Mitigation: Lean into honesty as a feature. The paper's angle is not "CP makes routing perfect" but "CP makes routing *honest* — users know when to trust it." The small-n analysis is not a weakness; it is the paper's motivation for why SSBC and online CP matter.

**Risk 4: The submission feels too "systems" for IUI.**
Mitigation: Lead with the HCI problem (file forgetting, trust in AI assistants, calibration mismatch). Put the user study results in the abstract. IUI rewards work that bridges the computational and human aspects — ensure the human aspect is visible from the first paragraph.

---

## 8. Summary and Immediate Next Steps

**What the theory says:** CP for Curator is mathematically sound even with small personal filesystems, provided the right method is chosen for the available n. For `n ≥ 150`, standard split CP with SSBC correction is appropriate. For `n < 100`, jackknife+ is preferred. Mondrian stratification requires careful stratum-size management. Cold-start is best handled with online CP initialized from a population prior.

**What the IUI 2027 paper should be:** A focused, dual-contribution paper presenting the FLRS signal, the CP routing layer, and a lab user study (n = 20) demonstrating retrieval time improvement and trust calibration improvement. Target 10,000 words. Submit by August 20, 2026.

**Immediate actions:**
1. Begin lab study IRB/ethics review process this week (6–8 week lead time)
2. Implement automated coverage validation experiment on synthetic filesystem dataset
3. Draft the paper introduction and system description (Sections 1–3 of the paper) in the next 2 weeks
4. Finalize FLRS definition and make it citable (preprint or technical report) before IUI submission

---

## Sources

- [Probabilistic Conformal Coverage Guarantees in Small-Data Settings (arXiv:2509.15349)](https://arxiv.org/abs/2509.15349)
- [Probabilistic Conformal Coverage Guarantees — HTML version](https://arxiv.org/html/2509.15349v1)
- [Finite-Sample Guarantees in Conformal Prediction: From DKW to Beta-Binomial (Medium)](https://medium.com/@peterzwart_44168/finite-sample-guarantees-in-conformal-prediction-from-dkw-to-beta-binomial-577228b313fe)
- [Predictive Inference with the Jackknife+ (arXiv:1905.02928)](https://arxiv.org/abs/1905.02928)
- [Predictive Inference with the Jackknife+ (Annals of Statistics, 2021)](https://projecteuclid.org/journals/annals-of-statistics/volume-49/issue-1/Predictive-inference-with-the-jackknife/10.1214/20-AOS1965.pdf)
- [Mondrian CP — MAPIE Documentation](https://mapie.readthedocs.io/en/latest/theoretical_description_mondrian.html)
- [Conformal Prediction with Conditional Guarantees — Gibbs & Cherian (arXiv:2305.12616)](https://arxiv.org/pdf/2305.12616)
- [Hierarchical Conformal Classification (arXiv:2508.13288)](https://arxiv.org/abs/2508.13288)
- [Improved Online Conformal Prediction via Strongly Adaptive Online Learning (arXiv:2302.07869)](https://arxiv.org/pdf/2302.07869)
- [Conformal Prediction Under Covariate Shift with Posterior Drift (arXiv:2502.17744)](https://arxiv.org/pdf/2502.17744)
- [IUI 2027 Call for Papers](https://iui.acm.org/2027/call-for-papers/)
- [IUI 2027 Conference Main Page](https://iui.acm.org/2027/)
- [Shafer & Vovk — A Tutorial on Conformal Prediction](https://www.researchgate.net/publication/1895464_A_tutorial_on_conformal_prediction)
- [Vovk, Gammerman, Shafer — Algorithmic Learning in a Random World](https://www.alrw.net/)
- [Universal Distribution of Empirical Coverage in Split Conformal Prediction (arXiv:2303.02770)](https://arxiv.org/pdf/2303.02770)
- [Conformal Prediction Beyond Exchangeability (arXiv:2202.13415)](https://arxiv.org/pdf/2202.13415)
- [IUI 2026 Call for Papers (ACM)](https://iui.acm.org/2026/call-for-papers/)
