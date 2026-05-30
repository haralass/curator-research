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

## 🚪 New Doors Opened
1. **FileGram benchmark (arXiv:2604.04901)** — they built FileGramEngine for generating synthetic file system workflows. Could we *use their benchmark* to evaluate Curator? Or contribute to it? This could be a collaboration opportunity.
2. **VaultSort is the closest commercial threat** — but no public API or open benchmarks. If they add HDBSCAN, the gap narrows fast. Priority: publish before they iterate.
3. **Sparkle's 10M files / 10k users** — they have real usage data. Their failure modes (cloud dependency, no uncertainty) are Curator's opportunity. A user survey comparing satisfaction between cloud-dependent vs local organizers could be a CHI contribution on its own.
4. **No tool uses macOS Spotlight metadata as retrieval signal** — this gap exists across ALL competitors. This is a specific, citable, exploitable gap.
