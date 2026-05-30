# 25 — FLRS: Empirical Neuroscience Grounding
_Date: 2026-05-30_

## Key Papers for Formula Grounding

**Pertzov et al. 2012 (PLOS ONE)** — object-location binding fails faster than item memory. Working memory scale (seconds). Core premise.
**Talamini & Gorree 2012 (Learning & Memory)** — associative binding decays faster than item memory at 1 week and 1 month. **The long-term scale paper we needed.**
**Antony & Paller 2018 (Learning & Memory, PMC)** — retrieval of spatial associations strengthens location memory beyond restudying. Testing effect compounds over time. **Directly validates SM-2 strengthening for location memory.**
**Murre & Dros 2015** — Ebbinghaus forgetting curve replication, clean empirical data.
**2025 PLOS Computational Biology** — DMN connectivity predicts individual forgetting speed → 2-3× variance across users.

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

**Sleep consolidation:** One night of sleep slightly retards decay (Antony & Paller). Optional: small S boost after first 8-hour window post-access.

**Individual variation:** 2-3× variance in S₀ across users (DMN connectivity). Build user-calibration phase or infer from access history.

## Formula Validation Status
R(t,f) = e^(-t/S) — ✅ Ebbinghaus exponential decay replicated (Murre & Dros 2015)
Binding fragility premise — ✅ Robust (Pertzov 2012, Talamini & Gorree 2012)  
SM-2 retrieval strengthening — ✅ Validated for spatial material (Antony & Paller 2018)
S₀ ≈ 1.0 day — ⚠️ Theoretically motivated, NOT direct empirical measurement

**Research gap:** No study measures file-specific location memory in naturalistic filesystem context. A simple user study (measure when participants forget where they saved a test file) would give direct calibration data. Could be a small CHI/IUI study contribution.

## 🚪 New Doors
1. **Antony & Paller 2018** — they tested sleep vs. wake consolidation of spatial memories. We could measure: do users forget file locations faster after bad sleep? macOS Screen Time data + FLRS predictions. Very quirky, high-novelty.
2. **"Retrieval quality as signal"** — if Spotlight search = failed retrieval and direct navigation = successful retrieval, monitoring this ratio per-user gives us a FLRS calibration signal for free. No privacy violation (we already log file events). This is a novel behavioral proxy for memory health.
3. **FSRS over SM-2** — FSRS (modern Anki algorithm, fitted to 500M reviews) gives better stability parameters than SM-2. Consider implementing FSRS-4.5 for B(f,t) instead of SM-2. Open source: https://github.com/open-spaced-repetition/py-fsrs
