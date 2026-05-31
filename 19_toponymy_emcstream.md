# 19 — Toponymy + EmCStream
_Date: 2026-05-30_

## Toponymy: ✅ USE IT

Real library, maintained by the UMAP/HDBSCAN authors (Leland McInnes, TutteInstitute).
GitHub: https://github.com/TutteInstitute/toponymy
Docs: https://toponymy.readthedocs.io/

**What it does:** Cluster LABELING (not clustering). Sits on top of HDBSCAN [Campello et al. 2013], uses LLM to generate human-readable names. Native multi-resolution hierarchy — produces `['University', 'Academic Research', 'ML Papers']` style 3-level paths. No predefined categories.

**BGE-M3 compatible:** Yes — SentenceTransformer API.
**Local LLM support:** Yes — LlamaCpp, HuggingFace backends.
**Replaces k-LLMmeans:** Yes, and it's better — HDBSCAN-native, hierarchy-aware, no fixed k.

`pip install toponymy` (from source) — ready to use.

## EmCStream (METU 2023, Wiley): ❌ DON'T SWITCH FROM DENSTREAM

Full paper: https://onlinelibrary.wiley.com/doi/full/10.1002/sam.11590
Code: https://gitlab.com/alaettinzubaroglu/emcstream

Novel idea: streaming UMAP (2D reduction) + k-means for evolving streams with drift detection. [Zubaroglu & Atalay 2023]

**Why it fails for Curator:**
1. Requires predefined k — can't know number of file categories in advance
2. 2D UMAP loses too much information from BGE-M3's 1024 dims
3. Designed for high-velocity numeric streams, not 5-20 files/day
4. No text/document benchmark — never tested on document streams
5. Sudden drift detection irrelevant for gradual personal file evolution

DenStream [Cao et al. 2006] wins on every axis for our use case. Keep it.

## References

- McInnes, L., Healy, J., & Astels, S. (2017). "hdbscan: Hierarchical density based clustering." *Journal of Open Source Software*, 2(11), 205. DOI: 10.21105/joss.00205.
- Campello, R. J. G. B., Moulavi, D., & Sander, J. (2013). "Density-Based Clustering Based on Hierarchical Density Estimates." *PAKDD 2013*, pp. 160–172. DOI: 10.1007/978-3-642-37456-2_14.
- McInnes, L., Healy, J., & Melville, J. (2020). "UMAP: Uniform Manifold Approximation and Projection for Dimension Reduction." arXiv:1802.03426.
- TutteInstitute. (2024). "Toponymy: Cluster Labeling with LLMs." GitHub. https://github.com/TutteInstitute/toponymy.
- Zubaroglu, A., & Atalay, V. (2023). "EmCStream: A Novel Evolving Micro-Cluster Stream Clustering Algorithm." *Statistical Analysis and Data Mining*, 16(3), 231–248. DOI: 10.1002/sam.11590.
- Cao, F., Ester, M., Qian, W., & Zhou, A. (2006). "Density-Based Clustering over an Evolving Data Stream with Noise." *Proceedings of the 2006 SIAM International Conference on Data Mining (SDM)*. DOI: 10.1137/1.9781611972764.29.
- Chen, J., Xiao, S., Zhang, P., Luo, K., Lian, D., & Liu, Z. (2024). "BGE M3-Embedding: Multi-Lingual, Multi-Functionality, Multi-Granularity Text Embeddings Through Self-Knowledge Distillation." arXiv:2402.03216.
