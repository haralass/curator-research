# 24 — ProxAnn + ARISE + GhostUMAP2
_Date: 2026-05-30_

## ProxAnn (ACL 2025)
Paper: arXiv:2507.00828 — "ProxAnn: Use-Oriented Evaluations of Topic Models and Document Clustering" [Hoyle et al. 2025]
GitHub: https://github.com/ahoho/proxann

LLM-as-judge for cluster quality, "statistically indistinguishable from human annotators."
**Limitation for Curator:** Text-oriented — needs readable text content per cluster. Not designed for metadata-only or mixed-type clusters. Usable only if cluster members have meaningful textual content (filenames, document text).

**For us:** Viable for text-heavy clusters (PDFs, code files). Not viable for image/binary/metadata-only clusters without adaptation.

## ARISE (ICLR 2025) ✅ DIRECTLY APPLICABLE
Paper: arXiv:2601.01162 — "Bridging the Semantic Gap for Categorical Data Clustering via LLMs" [ARISE Authors 2025]
GitHub: https://github.com/develop-yang/ARISE

Solves EXACTLY our FLACON flag problem: categorical values (e.g., kMDItemContentType = "public.pdf", "com.apple.application") → LLM generates semantic description per unique value → embed → use as distance basis instead of one-hot encoding.

**For us:** Replace one-hot encoding in FLACON flag distance with ARISE-generated embeddings. One LLM query per unique flag value (done once, cached). This is the implementation of our FLACON β component.

## GhostUMAP2 (arXiv:2507.17174) ✅ INSTALLABLE, USEFUL
`pip install ghostumap2`
GitHub: https://github.com/jjmmwon/ghostumap

Quantifies UMAP point instability by placing M "ghost" duplicates near each point, running UMAP, measuring max displacement. (r,d)-stable if displacement d_i ≤ d. [Woo et al. 2025]

~1% of points are unstable under typical settings. Unstable points → HDBSCAN boundary members → flag as uncertain before clustering.

**For us:** Run as pre-clustering validation. Unstable file-points → route to human review instead of auto-assignment. ~5x overhead → run only during 6-month refit health checks (as already planned in 23_UMAP_STABILITY).

## References

- Hoyle, A., Calvo-Bartolomé, L., Boyd-Graber, J., & Resnik, P. (2025). "ProxAnn: Use-Oriented Evaluations of Topic Models and Document Clustering." *Proceedings of ACL 2025 (Main)*. arXiv:2507.00828. GitHub: https://github.com/ahoho/proxann.
- ARISE Authors. (2025). "Bridging the Semantic Gap for Categorical Data Clustering via Large Language Models." *Proceedings of ICLR 2025*. arXiv:2601.01162. GitHub: https://github.com/develop-yang/ARISE.
- Woo, J., et al. (2025). "GhostUMAP2: Measuring and Analyzing (r,d)-Stability of UMAP." arXiv:2507.17174. GitHub: https://github.com/jjmmwon/ghostumap. PyPI: ghostumap2.
- McInnes, L., Healy, J., & Melville, J. (2020). "UMAP: Uniform Manifold Approximation and Projection for Dimension Reduction." arXiv:1802.03426.
- Campello, R. J. G. B., Moulavi, D., & Sander, J. (2013). "Density-Based Clustering Based on Hierarchical Density Estimates." *PAKDD 2013*, pp. 160–172. DOI: 10.1007/978-3-642-37456-2_14.
- Yoon, S. (2025). "FLACON: An Information-Theoretic Approach to Flag-Aware Contextual Clustering for Large-Scale Document Organization." *Entropy*, 27(11), 1133. DOI: 10.3390/e27111133.
