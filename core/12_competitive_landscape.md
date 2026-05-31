# 12 — Competitive Landscape: AI File Organizers (May 2026)
_Date: 2026-05-30_

## Top 3 Technical Competitors

| Tool | Clustering | CP/UQ | Per-file tracking | Local | Stars/Status |
|---|---|---|---|---|---|
| messy-folder-reorganizer-ai | Agglomerative (needs k) | No | No | Yes (Ollama+Qdrant) | ~16 stars, active |
| VaultSort | Batch embed→cluster | No | No | Yes (macOS) | Commercial, active |
| llama-fs | None (LLM tree) | No | No | Optional | ~5700 stars, inactive 2024 |

## Features Unique to Curator (zero competitors)
- Conformal prediction for routing
- Per-file Markov biography
- HDBSCAN without predefined categories
- DenStream online clustering
- FLACON distance metric
- H(location|intent) quality metric

## Key Finding: FileGram (arXiv:2604.04901, April 2026)
Research framework — NOT a product. Uses LLM episodic/procedural memory from file-system behavioral traces for agent personalization. Conceptually adjacent to File Biography but uses different mechanism (LLM memory channels vs. Markov chain). Must cite and differentiate.

Source: https://arxiv.org/abs/2604.04901

## Other Notable Entries
- **Sparkle (Every.to, Aug 2024)**: Cloud GPT-4, 10M files / 10k users. Mainstream traction but entirely cloud.
- **Files Magic AI (May 2025)**: macOS + Apple Intelligence, privacy-first, 168 PH upvotes.
- **AIOS-LSFS (ICLR 2025)**: Academic prototype, semantic file system via LLM, no clustering.
- **Sortio**: Cross-platform NLP + rules, offline mode, growing commercial traction.

## References

- Synvo-ai. (2026). "FileGram: LLM Episodic and Procedural Memory from Filesystem Behavioral Traces." arXiv:2604.04901. https://arxiv.org/abs/2604.04901.
- Campello, R. J. G. B., Moulavi, D., & Sander, J. (2013). "Density-Based Clustering Based on Hierarchical Density Estimates." *Pacific-Asia Conference on Knowledge Discovery and Data Mining (PAKDD)*, pp. 160–172. DOI: 10.1007/978-3-642-37456-2_14.
- Cao, F., Ester, M., Qian, W., & Zhou, A. (2006). "Density-Based Clustering over an Evolving Data Stream with Noise." *Proceedings of the 2006 SIAM International Conference on Data Mining (SDM)*. DOI: 10.1137/1.9781611972764.29.
- Yoon, S. (2025). "FLACON: An Information-Theoretic Approach to Flag-Aware Contextual Clustering for Large-Scale Document Organization." *Entropy*, 27(11), 1133. DOI: 10.3390/e27111133.
