# 20 — HippoCamp + FileGram Deep Analysis
_Date: 2026-05-30_

## HippoCamp (arXiv:2604.01221)

**NOT directly usable for Curator evaluation.**
- 581 QA pairs = question answering over personal files, NOT organization tasks
- No "move files to folders" tasks — only read/search/reason
- Dataset (42.4GB) likely gated — project page inaccessible, no public download found
- Can't plug Curator into it without defining our own org-quality ground truth

**Indirect value:** If dataset becomes available, realistic file corpus for clustering pipeline testing.

## FileGram (arXiv:2604.04901)
GitHub: https://github.com/Synvo-ai/FileGram

**NOT local-LLM compatible.** Hardcoded to Claude/Gemini/Cohere. No Ollama support.

**10 tasks (not 32):**
| ID | Name | Relevant to Curator? |
|---|---|---|
| T-05 | **Messy folder cleanup and reorganization** | ✅ YES |
| T-01–T-10 (others) | Understand/create/synthesize/iterate | ❌ No |

**T-05 structure** (our best overlap): messy folder with docs, emails, images, notes, calendar, chat logs → create subdirectories, move+rename files, write rationale. Measures: reading strategy, directory style, naming conventions, version/duplicate handling.

**FileGramBench measures:** user-profiling / behavioral memory (profile reconstruction, trace separation, drift detection). Not organization quality.

## Evaluation Strategy Implications

Neither paper gives us a ready benchmark. We need to build our own:
1. **Synthetic messy folders** with known ground-truth organization (we can generate with FileGramEngine if we have API keys)
2. **T-05 rubric** as human evaluation checklist (directory style, naming, version handling)
3. **20 Newsgroups / RCV1** for quantitative clustering metrics (NMI, ARI, Silhouette) [Lang 1995; Lewis et al. 2004]

## References

- HippoCamp. (2026). "HippoCamp: A Synthetic Personal File System Benchmark with Question-Answering." arXiv:2604.01221.
- Synvo-ai. (2026). "FileGram: LLM Episodic and Procedural Memory from Filesystem Behavioral Traces." arXiv:2604.04901. GitHub: https://github.com/Synvo-ai/FileGram.
- Lang, K. (1995). "Newsweeder: Learning to filter netnews." *Proceedings of the 12th International Conference on Machine Learning (ICML 1995)*, pp. 331–339. (20 Newsgroups dataset.)
- Lewis, D. D., Yang, Y., Rose, T., & Li, F. (2004). "RCV1: A New Benchmark Collection for Text Categorization Research." *Journal of Machine Learning Research*, 5, 361–397.
- Campello, R. J. G. B., Moulavi, D., & Sander, J. (2013). "Density-Based Clustering Based on Hierarchical Density Estimates." *PAKDD 2013*, pp. 160–172. DOI: 10.1007/978-3-642-37456-2_14.
