# 13 — Evaluation Methodology for CHI 2027
_Date: 2026-05-30_

## Foundational Papers
- Bergman et al. 2010 (JASIST, N=296): Re-finding time + navigation steps — baseline metrics [Bergman et al. 2010]
- Elsweiler & Ruthven 2007 (SIGIR): Diary study methodology — log real failures, replay in lab [Elsweiler & Ruthven 2007]
- Haraty et al. 2017 (JASIST): Closest precedent — mixed-initiative system, in-situ, acceptance + correction rate [Haraty et al. 2017]
- Brackenbury et al. 2021 (SIGIR): Perceived co-grouping metric — pairwise file similarity agreement [Brackenbury et al. 2021]
- Zhang et al. CHI 2024 (N=600, Best Paper HM): CP sets improve accuracy vs single-label, effect medium [Zhang et al. 2024]
- Straitouri et al. ICML 2024 (arXiv:2401.13744): RCT — variable set size IS the mechanism [Cresswell et al. 2024]

## Automated Benchmarks (no users)
- **20 Newsgroups**: 20K docs, 20 categories, NMI/AMI/ARI against known labels
- **RCV1-v2**: Hierarchical Reuters taxonomy — best proxy for multi-level folder structure
- **CFCOS (Electronics 2026, doi:10.3390/electronics15071524)**: Direct LLM file classification benchmark with GPT-4.1 upper bound [CFCOS 2026]
- **HippoCamp (arXiv:2604.01221, April 2026)**: 42.4GB synthetic personal file system, 581 QA pairs — NEW, purpose-built for this problem [HippoCamp 2026]

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

## References

- Bergman, O., Whittaker, S., Sanderson, M., Nachmias, R., & Ramamoorthy, A. (2010). "The effect of folder structure on personal file navigation." *Journal of the American Society for Information Science and Technology*, 61(12), 2426–2441. DOI: 10.1002/asi.21415.
- Elsweiler, D., & Ruthven, I. (2007). "Towards task-based personal information management evaluations." *Proceedings of the 30th Annual International ACM SIGIR Conference*, pp. 23–30. DOI: 10.1145/1277741.1277748.
- Haraty, M., Tam, D., Haddad, S., McGrenere, J., & Tang, C. (2017). "Individual Differences in Personal File Organization and Its Effect on the Design of Mixed-Initiative Systems." *Journal of the American Society for Information Science and Technology*, 68(1), 159–180. DOI: 10.1002/asi.23620.
- Brackenbury, W., Harrison, G., Chard, K., Elmore, A., & Ur, B. (2021). "Files of a Feather Flock Together? Measuring and Modeling How Users Perceive File Similarity in Cloud Storage." *Proceedings of the 44th International ACM SIGIR Conference on Research and Development in Information Retrieval (SIGIR '21)*. https://www.blaseur.com/papers/sigir21.pdf.
- Zhang, D., Chatzimparmpas, A., Kamali, N., & Hullman, J. (2024). "Evaluating the Utility of Conformal Prediction Sets for AI-Advised Image Labeling." *Proceedings of the 2024 CHI Conference on Human Factors in Computing Systems*. DOI: 10.1145/3613904.3642446. arXiv:2401.08876.
- Cresswell, J. C., Sui, Y., Kumar, B., & Vouitsis, N. (2024). "Conformal Prediction Sets Improve Human Decision Making." *Proceedings of the 41st International Conference on Machine Learning (ICML 2024)*. arXiv:2401.13744.
- CFCOS. (2026). "Content-Based File Classification and Organization System Using LLMs." *Electronics*, 15(7), 1524. DOI: 10.3390/electronics15071524.
- HippoCamp. (2026). "HippoCamp: A Synthetic Personal File System Benchmark." arXiv:2604.01221.
- Lang, K. (1995). "Newsweeder: Learning to filter netnews." *Proceedings of the 12th International Conference on Machine Learning*. (20 Newsgroups dataset.)
- Lewis, D. D., Yang, Y., Rose, T., & Li, F. (2004). "RCV1: A New Benchmark Collection for Text Categorization Research." *Journal of Machine Learning Research*, 5, 361–397.
