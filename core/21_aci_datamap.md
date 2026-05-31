# 21 — ACI Distribution Shift CP + DataMapPlot
_Date: 2026-05-30_

## Gibbs & Candes — ACI (Adaptive Conformal Inference)

**2021 NeurIPS:** arXiv:2106.00170 — Original ACI, gradient descent on coverage error [Gibbs & Candès 2021]
**2022/2024 JMLR:** arXiv:2208.08401 — Improved ACI, auto-tunes step-size [Gibbs & Candès 2024]

**Critical:** Both require full feedback — true label revealed at EVERY step. Cannot handle semi-bandit. Cannot be applied directly to Curator.

## KEY PAPER: arXiv:2604.17984 (April 2026)
"Online Conformal Prediction with Adversarial Semi-Bandit Feedback via Regret Minimization"

**First paper combining adversarial (non-IID) + semi-bandit feedback.** Uses EXP3.P-style bandit algorithm (OCP-Unlock+), proves sublinear regret → long-run coverage under both IID and adversarial settings. Cites Gibbs & Candès extensively. [OCP-Unlock+ 2026]

This is the state-of-the-art framework for our exact use case (personal file stream, non-IID, semi-bandit). We should:
1. Cite it as related work
2. Potentially adopt OCP-Unlock+ instead of SPS for the non-IID regime
3. The gap between SPS [Angelopoulos et al. 2025] (stochastic) and 2604.17984 (adversarial) — combining them cleanly is still open

**Theoretical contribution opportunity:** Neither paper handles the "revealed preference" problem (user picks from constrained set ≠ true best label). This remains open.

## DataMapPlot (TutteInstitute)
GitHub: https://github.com/TutteInstitute/datamapplot
Same authors as UMAP [McInnes et al. 2020] + HDBSCAN [Campello et al. 2013] (Leland McInnes).

**Static** (matplotlib PNG/SVG) **+ Interactive** (self-contained HTML via DeckGL).
**Tauri/React integration:** iframe of the interactive HTML. No native React component.
**Pipeline:** BGE-M3 embeddings → UMAP 2D → HDBSCAN → DataMapPlot → "visual map of your files"

**Limitation:** Python-only. Curator would need a Python sidecar for real-time generation.

## References

- Gibbs, I., & Candès, E. J. (2021). "Adaptive Conformal Inference Under Distribution Shift." *Advances in Neural Information Processing Systems 34 (NeurIPS 2021)*. arXiv:2106.00170. https://proceedings.neurips.cc/paper_files/paper/2021/hash/0d441de75945e5acbc865406fc9a2559-Abstract.html.
- Gibbs, I., & Candès, E. J. (2024). "Conformal Inference for Online Prediction with Arbitrary Distribution Shifts." *Journal of Machine Learning Research*, 25(1), 1–36. arXiv:2208.08401.
- OCP-Unlock+. (2026). "Online Conformal Prediction with Adversarial Semi-Bandit Feedback via Regret Minimization." arXiv:2604.17984.
- Angelopoulos, A. N., et al. (2025). "Stochastic Online Conformal Prediction with Semi-Bandit Feedback." *Proceedings of ICML 2025*. arXiv:2405.13268.
- McInnes, L., Healy, J., & Melville, J. (2020). "UMAP: Uniform Manifold Approximation and Projection for Dimension Reduction." arXiv:1802.03426.
- Campello, R. J. G. B., Moulavi, D., & Sander, J. (2013). "Density-Based Clustering Based on Hierarchical Density Estimates." *PAKDD 2013*, pp. 160–172. DOI: 10.1007/978-3-642-37456-2_14.
- TutteInstitute. (2024). "DataMapPlot: Creating Beautiful and Informative Data Map Visualisations." GitHub. https://github.com/TutteInstitute/datamapplot.
- Auer, P., Cesa-Bianchi, N., Freund, Y., & Schapire, R. E. (2002). "The Nonstochastic Multiarmed Bandit Problem." *SIAM Journal on Computing*, 32(1), 48–77. DOI: 10.1137/S0097539701398375. (EXP3 algorithm family underlying OCP-Unlock+.)
