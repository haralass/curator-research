# 13 — Evaluation Methodology for CHI 2027
_Date: 2026-05-30_

## Foundational Papers
- Bergman et al. 2010 (JASIST, N=296): Re-finding time + navigation steps — baseline metrics
- Elsweiler & Ruthven 2007 (SIGIR): Diary study methodology — log real failures, replay in lab
- Haraty et al. 2017 (JASIST): Closest precedent — mixed-initiative system, in-situ, acceptance + correction rate
- Brackenbury et al. 2021 (SIGIR): Perceived co-grouping metric — pairwise file similarity agreement
- Zhang et al. CHI 2024 (N=600, Best Paper HM): CP sets improve accuracy vs single-label, effect medium
- Straitouri et al. ICML 2024 (arXiv:2401.13744): RCT — variable set size IS the mechanism

## Automated Benchmarks (no users)
- **20 Newsgroups**: 20K docs, 20 categories, NMI/AMI/ARI against known labels
- **RCV1-v2**: Hierarchical Reuters taxonomy — best proxy for multi-level folder structure
- **CFCOS (Electronics 2026, doi:10.3390/electronics15071524)**: Direct LLM file classification benchmark with GPT-4.1 upper bound
- **HippoCamp (arXiv:2604.01221, April 2026)**: 42.4GB synthetic personal file system, 581 QA pairs — NEW, purpose-built for this problem

## 4-Phase Protocol for CHI 2027

| Phase | What | N | Duration |
|---|---|---|---|
| 0 | Automated benchmark (20NG, RCV1, CFCOS, HippoCamp) | — | 2-4 weeks |
| 1 | Diary study — real re-finding failures | 14-16 | 2 weeks |
| 2 | Controlled lab — within-subjects, re-finding tasks | 36-40 | 2 sessions |
| 3 | In-the-wild longitudinal | 20 | 4 weeks |

**Primary DVs:** re-finding time (log-transformed), success rate, acceptance rate, correction/undo rate, search rate pre/post.

**Conformal sub-condition in Phase 2:** 20 participants see single-label, 20 see prediction sets (between-subjects nested within within-subjects design).

## Behavioral Logging Schema (privacy-preserving)
```
event_type ∈ {suggestion_shown, accepted, rejected, modified, undone, manual_move}
prediction_set_size (N labels shown)
time_to_decision_ms
session_id
spotlight_search_invoked (binary, no query content)
```
All file identifiers hashed. No content logged.

## 🚪 New Doors Opened
1. **HippoCamp (arXiv:2604.01221)** — 42.4GB synthetic personal file benchmark with 581 QA pairs, released April 2026. We should evaluate Curator against it NOW, before the CHI deadline. This positions us as the first real system evaluated on this benchmark.
2. **Zhang et al. CHI 2024 (N=600 CP study)** — They did image labeling. We can do the exact same study design but for *file routing*. This would be the first replication of CP-in-HCI in a file management context — a clean, publishable angle.
3. **"Search rate post-organization" as a metric** — Whittaker et al. showed search rate correlates with organizational failure. We could measure this passively via macOS accessibility API (no query content). If Curator reduces Spotlight search rate, that's a strong behavioral signal.
4. **Haraty et al. 2017 mixed-initiative design** — They evaluated a system that suggested file organizations. Our setup is nearly identical. Could we contact them for methodological guidance or even collaboration?
