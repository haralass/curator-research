# 21 — ACI Distribution Shift CP + DataMapPlot
_Date: 2026-05-30_

## Gibbs & Candes — ACI (Adaptive Conformal Inference)

**2021 NeurIPS:** arXiv:2106.00170 — Original ACI, gradient descent on coverage error
**2022/2024 JMLR:** arXiv:2208.08401 — Improved ACI, auto-tunes step-size

**Critical:** Both require full feedback — true label revealed at EVERY step. Cannot handle semi-bandit. Cannot be applied directly to Curator.

## 🔴 KEY PAPER: arXiv:2604.17984 (April 2026)
"Online Conformal Prediction with Adversarial Semi-Bandit Feedback via Regret Minimization"

**First paper combining adversarial (non-IID) + semi-bandit feedback.** Uses EXP3.P-style bandit algorithm (OCP-Unlock+), proves sublinear regret → long-run coverage under both IID and adversarial settings. Cites Gibbs & Candes extensively.

This is the state-of-the-art framework for our exact use case (personal file stream, non-IID, semi-bandit). We should:
1. Cite it as related work
2. Potentially adopt OCP-Unlock+ instead of SPS for the non-IID regime
3. The gap between SPS (stochastic) and 2604.17984 (adversarial) — combining them cleanly is still open

**Theoretical contribution opportunity:** Neither paper handles the "revealed preference" problem (user picks from constrained set ≠ true best label). This remains open.

## DataMapPlot (TutteInstitute)
GitHub: https://github.com/TutteInstitute/datamapplot
Same authors as UMAP + HDBSCAN (Leland McInnes).

**Static** (matplotlib PNG/SVG) **+ Interactive** (self-contained HTML via DeckGL).
**Tauri/React integration:** iframe of the interactive HTML. No native React component.
**Pipeline:** BGE-M3 embeddings → UMAP 2D → HDBSCAN → DataMapPlot → "visual map of your files"

**Limitation:** Python-only. Curator would need a Python sidecar for real-time generation.

## 🚪 New Doors
1. **arXiv:2604.17984 (April 2026)** — published ONE MONTH ago. This is the theoretical paper we need. Read it carefully — OCP-Unlock+ might be drop-in replacement for SPS with stronger non-IID guarantees. If we can show it converges faster on personal file streams, that's empirical contribution.
2. **DataMapPlot "visual file map"** — semantic map of the user's entire file system. Nobody has built this as a consumer feature. Could be Curator's killer visual differentiator. Feasibility: medium (Python sidecar for UMAP+DataMapPlot, webview in Tauri).
3. **"Revealed preference" problem is still open** — when user picks folder from constrained set, that's not ground truth. No paper addresses this. Formalizing it = novel theoretical contribution that no existing CP paper touches.
