# 25 — FLRS: Empirical Neuroscience Grounding
_Date: 2026-05-30_

## Key Papers for Formula Grounding

**Pertzov et al. 2012 (PLOS ONE)** — object-location binding fails faster than item memory. Working memory scale (seconds). Core premise. [doi:10.1371/journal.pone.0048214]

**Talamini & Gorree 2012 (Learning & Memory)** — associative binding decays faster than item memory at 1 week and 1 month. **The long-term scale paper we needed.** [Talamini & Gorree 2012]

**Antony & Paller 2018 (Learning & Memory, PMC)** — retrieval of spatial associations strengthens location memory beyond restudying. Testing effect compounds over time. **Directly validates SM-2 strengthening for location memory.** [doi:10.1101/lm.046268.117]

**Murre & Dros 2015** — Ebbinghaus forgetting curve replication, clean empirical data. [doi:10.1371/journal.pone.0120644]

**2025 PLOS Computational Biology** — DMN connectivity predicts individual forgetting speed → 2-3× variance across users. [Khosrowabadi et al. 2025]

## Recommended Default FLRS Parameters

| Parameter | Value | Basis |
|---|---|---|
| S₀ (initial stability) | **1.0 day** | Ebbinghaus + binding fragility penalty (0.5-0.7× item memory) |
| FLRS = 0.5 crossover | **~17 hours** after single access | R(0.69) = 0.5 |
| S after 1st retrieval | 3–5 days | SM-2 ease factor ~3-4; FSRS w₁₄ ≈ 5.1 |
| S after 2nd retrieval | 10–15 days | Continued compounding |
| S after 3rd retrieval | 30–50 days | "Feels permanent" |
| Binding fragility vs item memory | 0.5–0.7× | Talamini & Gorree 2012 |

## Critical Implementation Notes

**Retrieval quality matters:**
- Direct navigation (open file without search) = strong retrieval → large S increase
- Spotlight/search-assisted = weak retrieval → minimal S increase (treat as non-retrieval)

**Sleep consolidation:** One night of sleep slightly retards decay [Antony & Paller 2018]. Optional: small S boost after first 8-hour window post-access.

**Individual variation:** 2-3× variance in S₀ across users (DMN connectivity) [Khosrowabadi et al. 2025]. Build user-calibration phase or infer from access history.

## Formula Validation Status
R(t,f) = e^(-t/S) — ✅ Ebbinghaus exponential decay replicated [Murre & Dros 2015]
Binding fragility premise — ✅ Robust [Pertzov et al. 2012; Talamini & Gorree 2012]
SM-2 retrieval strengthening — ✅ Validated for spatial material [Antony & Paller 2018]
S₀ ≈ 1.0 day — ⚠️ Theoretically motivated, NOT direct empirical measurement

**Research gap:** No study measures file-specific location memory in naturalistic filesystem context. A simple user study (measure when participants forget where they saved a test file) would give direct calibration data. Could be a small CHI/IUI study contribution.

## References

- Pertzov, Y., Dong, M. Y., Peich, M.-C., & Husain, M. (2012). "Forgetting What Was Where: The Fragility of Object-Location Binding." *PLOS ONE*, 7(10), e48214. DOI: 10.1371/journal.pone.0048214.
- Talamini, L. M., & Gorree, E. (2012). "Aging memories: Differential decay of episodic memory components." *Learning & Memory*, 19(6), 239–246. https://learnmem.cshlp.org/content/19/6/239.full.html.
- Antony, J. W., & Paller, K. A. (2018). "Retrieval and sleep both counteract the forgetting of spatial information." *Learning & Memory*, 25(6), 258–263. DOI: 10.1101/lm.046268.117. PMC: PMC5959224.
- Murre, J. M. J., & Dros, J. (2015). "Replication and Analysis of Ebbinghaus' Forgetting Curve." *PLOS ONE*, 10(7), e0120644. DOI: 10.1371/journal.pone.0120644.
- Khosrowabadi, R., et al. (2025). "Default Mode Network Connectivity Predicts Individual Forgetting Speed." *PLOS Computational Biology*. (DMN and individual forgetting rate variance.)
- Ebbinghaus, H. (1885). *Über das Gedächtnis: Untersuchungen zur experimentellen Psychologie*. Duncker & Humblot. (Translated: Memory: A Contribution to Experimental Psychology, 1913.)
- Wozniak, P. A. (1990). "Optimization of Learning." Master's thesis, University of Technology, Poznan. (SM-2 spaced repetition algorithm.)
- Ye, T., Luo, S., Ma, R., et al. (2024). "FSRS-4.5." Open Spaced Repetition GitHub. https://github.com/open-spaced-repetition/py-fsrs. (FSRS algorithm fitted on 500M reviews.)
- Elsweiler, D., Ruthven, I., & Jones, C. (2007). "Towards Memory Supporting Personal Information Management Tools." *Journal of the American Society for Information Science and Technology*, 58(7), 924–946. DOI: 10.1002/asi.20570.
